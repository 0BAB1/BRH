---
title: "Trademaxxing part4 : Orders Bookkeeping"
date: 2026-04-08 16:00:00 +0800
categories: [Projects]
tags: [finance, fpga]
pin: false
math: false
mermaid: false
---

## Introduction & Context

In the previous posts, we designed an ITCH parser. `ITCH 5.0` is the Nasdaq protocol that streams live market events to whoever has the licence to listen.
To build our parser logic, we previously used a testbench that uses real ITCH market data given by Nasdaq in order to support the following events:

* R (Stock directory to know what locate each stock uses on that specific trading day)
* A, F (add orders)
* E, C, X (execute or cancel orders)
* D (delete orders)
* U (replace order)

This demo parser tracks a single stock (AAPL). And our goal is to do some bookkeeping, i.e. track ongoing orders and build a "price ladder".
We arrived at a point where the parser is able to figure out what state changes are needed to update the books and reflect the market events.

Today, we'll focus on the logic that will grab the parser's commit signal and update the books' states accordingly, but also generate the ladder state updates as well.

This "double role" will make things a bit difficult, but nothing impossible, so let's get started.

## State Updates Operations Thoughts

### Macro Operations List

For our order book (I'll also refer to it as "Order List"), there aren't many ways the state can change. In fact, here are the different "OP_TYPES" the parser generates from market events:

```verilog
typedef enum logic [7:0] {
    ADD_ORDER,
    EXEC_ORDER,
    DELETE_ORDER,
    REPLACE_ORDER
} order_op_t;
```

{: .prompt-info }

> we'll refer to these as "macro OPs"

Because yeah, even though we support different types of market events (R, A, F, E, C, X, D, U), most of them do the exact same thing when it comes to an order list perspective.

**Example**: E means an order has been executed (at a quantity amount) and X means it has been canceled (same thing, we just remove a quantity from an order in the books).

From a trading strategy perspective, these are important distinctions but here we kinda don't care so the OPERATION is the exact same, and many other refactorizations can be done.

### Micro Operations Thoughts

Another point I'd like to talk about is `REPLACE`: it can be broken down into `DELETE` + `ADD`. Seeing it like so makes things way easier to design a datapath later on.

### Operations Refs

Orders are referenced like this: `new quantity = old quantity - executed quantity`. Each order has a unique ref for each trading day, even after being deleted.

BUT we have a **HUGE** problem, the reference of an order is 8 bytes wide, or 64 bits. If you have any experience working with memory, you know there is no way we have 2^64 slots of memory lying around, especially since we'll want to store a structure like that:

```verilog
// pseudo code

order[ref] = {price, quantity, is_buy}; // 65bits
```

So 2^65 * 64 bits = **TOO MUCH** and I'm not even gonna bother calculating it lol.

Let's try to quantify what kind of actual data volume we need to deal with using a python script on the ITCH50 binary data provided by NASDAQ:

```python
Peak simultaneous live orders  : 921
Total unique refs seen         : 12769
Max ref value                  : 287214607 # so it's pretty much any growing number...
Min ref value                  : 19575     
```

<!-- Collisions with 16 bits: 9881 -->

I chose the `ZYNE` stock here as it has a fair amount of orders per day, but not too much (unlike AAPL that has way too much liquidity, i.e. live orders).

Takeaways from that ?

* Orders refs can be pretty much anything and have a huge range possibility in practice
* 948 live orders max

I'll be implementing the design on a **KC705** FPGA and I don't plan on using the onboard DRAM, I'll instead use BRAM within the FPGA as `~ 1000 entries => 1000*65 = 65kbits of RAM` which largely fits in BRAM. Let's say we'll use a 16-bit address just to be sure we have plenty of space.

We can even reduce this amount by only storing lower 24 bits of price and quantities.

Now, I don't wanna make a complex hash/mapping system, so we'll brute-force the issue. Also I don't mind a small collision amount for my demo. Long story short, I got to a solution where I could use the 64-bit refs and map them to a 17-bit address and got these stats:

```python
Peak simultaneous live orders : 921
Final live orders             : 3
BRAM slots used               : 3
Total collisions              : 37
Total unique refs seen        : 12769
```

Not too bad ! that's like a 0.2% collision rate. Not too bad.

## Order Book Datapath

Okay I think the problematics are now pretty clear, enough with the technical yapping and here is the final datapath after lots of consideration. Again, a big challenge here is to also generate state update signals in the end for the price ladder, which definitely adds a bit of complexity but nothing fancy really.

![Order book block diagram](/assets/img/order_book_scheme2.png)

## Testbench

To test this out, we are first gonna make a separate testbench, because the RAM interfaces use AXI, which makes the code itself a bit more complex so I use cocotbext.axi to make sure both RAM masters are AXI compliant and we can leverage python to run automated assertions on random datasets to stress test this order book.

![Order book tb image](/assets/img/order_book_tb.png)

What you are seeing above is a completely random TB simulating parsed orders and comparing BRAM states to a golden reference to make sure state is properly saved. Price ladder signals are also tested in simpler use cases at the beginning to validate their behavior for each type of macroOP/microOP.

## Integration

And now, all that remains is to plug this subdesign into the global trademaxxer design DUT and observe ! It's just a matter of plugging in the cables at this point.. so that's exactly what we do !

And there we go ! In the following testbench, the order book input/output in the context of real ITCH market data coming in !

![order book overview on tb](/assets/img/overview_order_book_tb.png)

*Impressive. Very nice.*

You can see the orders come in and get executed as time goes by. Very cool.

{: .prompt-info }

> Of course, given how scarce `ZYNE` orders are, the testbench does a big filtering of orders and only passes ZYNE-related orders to this TB. In reality, valid orders will be more scarce. (*because yes I'm tracking ZYNE here and not AAPL, remember, too much volume may create collisions in our BRAM and we did not really mitigate that lol*).

Here is a focus on a single order being saved (state update to BRAM internally) and then transmitting the adequate state changes to the future price ladder:

![order book zoom on tb](/assets/img/zoom_orderbook_integration_tb.png)

Thank you for reading to this point. You can write a comment below if you have any questions.

*Godspeed*

-BRH
