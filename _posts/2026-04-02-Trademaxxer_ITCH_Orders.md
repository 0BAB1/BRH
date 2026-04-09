---
title: "Trademaxxing part4 : Orders Bookkeeping"
date: 2026-04-04 16:00:00 +0800
categories: [Projects]
tags: [finance, fpga]
pin: false
math: false
mermaid: false
---

## Introduction & Context

In the previous posts, we designed an ITCH parser. `ITCH 5.0` is the Nasdaq protocol that stream live market event to whoever has the licence to listen.
To build our parser logic, we previusly used a testbench that uses real ITCH market data given by Nasdaq in order to support the following events:

- R (Stock directory to know what locate each stock uses on that specific trading day)
- A,F (Add orders)
- E,C,X (Execute or cancel oerders)
- D (delete orders)
- U (replace order)

This demo parser tracks a single stock (AAPL). And out goal is to do some bookkeeping, i.e. tracks ongoing orders and build a "price ladder".
We arrived to a point where the parser is able to figure out what state changes are needed to update the books and reflect the market event.

Today, we'll focus on the logic that will grab the parsers commit signal and update the books' states accordingly, but also generate the ladders state update as well.

This "double role" will make thing a bit difficult, but nothing impossible, so let's get started.

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

Because yeah, even though e support different types of markets event (R,A,F,E,C,X,D,U), most of them do the exact same thing when it come to an order list perspective.

**Exmaple**: E means an order has been executed (at a quantity amount) and X means it has been canceled (Same thing, we just remove a quantity from an order in the books).

From a trading strategy perspective, these are important distinctions but here we kinda don't care so the OPERATION is the exact same, an many other refactorization can be done.

### Micro Operations Thoughts

Another poitn i'd like to talk about is `REPLACE`: it can be broken down `DELETE` + `ADD`. Seeing it like so makes things way easier to design a datapath later on.

### Operations Refs

Orders are reference like this: `new quantity = old quantity - executed quantity`. Each order has a unique ref at each trading day, even after being deleted.

BUT we have a **HUGE** problem, the reference of an order is 8 bytes wide, or 64 bits. If you have any experience working with memory, you know ther is no way we have 2^64 slots of memory lying around, especially since we'ell want to store a structure like that:

```verilog
// pseudo code

order[ref] = {price, quantuty, is_buy}; // 65bits
```

So 2^65 * 64 bits = **TOO MUCH** and i'm not even gonna bother calculating it lol.

Let's try to quantify what kind of actual data volume we need to deal with using a python script on the ITCH50 binary data provided by NASDAQ:

```python
Peak simultaneous live orders  : 921
Total unique refs seen         : 12769
Max ref value                  : 287214607 # so it's pretty much any growing number...
Min ref value                  : 19575     
```

<!-- Collisions with 16 bits: 9881 -->

I chose the `ZYNE` stock here as it has a fair amount of orders per day, but not too much (unlike APPL that has way too much liquity, i.e. live orders).

Takeaways from that ?

- Orders refs can be pretty much anything and has a huge range possibility in practice
- 948 live orders max

I'll be implemeting the design on a **KC705** FPGA and I don't plan on using the onboard DRAM, I'll instead use BRAM whithing the FPGA as `~ 1000entries => 1000*65 = 65kbits of RAM` which largely fits in BRAM. Let's say we'll use a 16 bits address just to be sure we have plety of space.

We can even reduce this amount by only storing lower 24bits of price and quantities.

Now, I don't wanna make a complex hash/mapping system, so we'll bruteforce the issue. Also I don't mind small collision amount for my demo. Long story short, I got to a sulution where I could use the 64 bits refs and map it to a 17bits address and got these stats:

```python
Peak simultaneous live orders : 921
Final live orders             : 3
BRAM slots used               : 3
Total collisions              : 37
Total unique refs seen        : 12769
```

not too bad ! that like a 0,2% collision reate. Not too bad.

## Order Book Datapath

Okay I think the problematics are now pretty clear, off with the technical yapping and here is the final datapath after lots of consideration. Againg, a big challenge here is to also generate state update signals in the end for the price ladder, which definitly add a bit of complexity but nothing fancy really.

![Order book block diagram](/assets/img/order_book_scheme2.png)

## Testbench

To test this out, we are first gonna make a separate testbench, because the RAM interfaces uses AXI, which makes the code itself a bit more complex so I use cocotbext.Axi to make sure both RAM masters are AXI compliant and we can leverage python to run automated assertions on random dataset to stress test this orders book.

![Order book tb image](/assets/img/order_book_tb.png)

What you are seeing above is a completely random tb simulating parsed orders and comparing BRAM states to a golden reference to make sure state is properly saved. Price ladder signals are also tested in simpler used cases at the beginning to validate their behavior for each type of macroOP/microOP.

## Integration

And now, all that remains is to plug this subdesign in the global trademaxxer design DUT and obseerve ! It's jsut a matter on plugging in the cble at this point.. so that's exacltly what we do !

And there we go ! In the following testbench, the order book input output in the context of real ITCH market data comming in !

![order book overview on tb](/assets/img/overview_order_book_tb.png)

*Impressive. Very nice.*

You can see the order come in and get executed as time goes by. Very cool.

{: .prompt-info }
> Of course, given of scrsae `ZYNE` orders are, the tesbench does a big filtering of orders and only passes ZYNE related orders to this tb. In reallity, valid orders will be more scarse. (*because yes i'm tracking ZYNE here and not appl, remember, too much volume may create collisions in ou BRAM and we did not rally mitigate that lol*).

Here is a focus on a single order being saved (state update to BRAM internally) and then transmitting to adequate state changes to the future price ladder:

![order book zoom on tb](/assets/img/zoom_orderbook_integration_tb.png)
