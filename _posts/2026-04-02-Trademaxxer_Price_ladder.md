---
title: "Trademaxxing part5 : Ladder Bookkeeping"
date: 2026-04-11 16:00:00 +0800
categories: [Projects]
tags: [finance, fpga]
pin: false
math: false
mermaid: false
---

## Introduction & Context

In the previous posts, we designed an ITCH parser. `ITCH 5.0` is the Nasdaq protocol that streams live market events to whoever has the licence to listen.

We were able to parse many message types, and then design an orders tracker (*or "bookkeeper" as I like to call it*) to track the market state. This bookkeeper had some tradeoffs that we'll surely have to fix in the future (very bad address mapping and non-existent collision detection/handling). Yet it worked and after some pretty extensive testing, I figured it was good enough.

So today I'd like to move forward a bit (*and decide that the problems of the bookkeeper are for the later me to solve*) by designing the final piece of the puzzle: **the price ladder**.

For those wondering, it's a pretty visual tool that looks like that:

![price ladder image](/assets/img/ladder.png)

It's like THE tool to quickly visualize positions on a given market, it shows, for a given price, the amount of actors **asking** for that price (*selling*) or **bidding** for that price (*buying*).

The job of today's design will simply be to keep track of that using the state updates coming from the bookkeeper.

I also want to make it an **AXI4 LITE slave** so that a CPU can probe the market state easily, making the final demo a bit more visual.

## Design Tradeoffs (again ??)

As usual, you know the drill, because it's a simple demo, I'll make some tradeoffs.

Unlike the orders bookkeeper, where **I KNOW FOR SURE** the tradeoffs I made **WILL** bite me in the ass later, here the tradeoffs are not as critical.

We'll work with BRAM, limited space and we need to store that kind of data:

```verilog
// pseudocode
mem[price] = 32{is_buy, quantity[30:0]};
```

{: .prompt-info }

> Yes, I truncated the quantity by 1 bit in order to have nice words of 4 bytes, easier to deal with in AXI LITE. Quantities (almost) never reach 2^32 anyway.

So there's that, and given a stock doesn't really move that much in price within a day, we'll simply define a range, let's say 1K entries, quantize prices so each entry can hold multiple price values (with a 5-cent step for example), leading to a total of 4kB BRAM usage for this.

{: .prompt-info }

> Side note : we need to multiply this usage by 2 given I'll use a 2R/1W BRAM implementation, not ideal but I don't mind.

Of course that's just a simple example and I'll fine-tune these numbers later. We can just keep these things in mind to lay down a block design of our objective datapath.

## Datapath

Okay, enough yapping, with all of this in mind, we can lay down a simple datapath for the **price ladder**:

![price ladder datapath](/assets/img/price_ladder_datapath.png)

The scheme is very self-explanatory here, a simple 2-stage pipeline, first we read from the BRAM using our quantized address (derived from price) and then we compute the new quantity for that given price before storing it back.

## Basic Testbench

Testing the price ladder as a single unit is the same principle as the order book, except here it's quite a bit simpler.

Again, we generate signals that mimic what the order bookkeeper will generate and in the testbench, we keep track of a golden reference we compare the BRAM state to.

We of course make it fully random so it kinda stress-tests the design in the screenshot below, I took it for a spin of 20 iterations but in reality I like to run these tests over hundreds of thousands, though this design is simple, it pretty much catches all the obvious bugs.

We also add a second test to test edge cases, in the screenshot below, you'll see at the very end, we test a completely out-of-range price, which is flagged and prevents `write_enable` from going high, effectively dropping the state update from a price ladder perspective and preserving the BRAM from weird updates.

![price ladder basic testbench](/assets/img/price_ladder_basic_tb.png)

## Testing the AXI-LITE Slave

Okay, you remember how I said I'd like this price ladder to also be an AXI-LITE slave ?

Well we'll also test that. I really like cocotb for a lot of reasons but extensions have to be one of the best. It allows for quick verification and extensive testing of standard communication interfaces like ethernet or AXI.

So we'll use the `cocotbext.axi` extension.

{: .prompt-info }

> if you need more infos on the code and people behind it, I have a dedicated blog post / tutorial on that [(Link)](https://0bab1.github.io/BRH/posts/TIPS_FOR_COCOTB/)

To also make things a bit tight regarding the interface, I like to use [pulp platform AXI interface declarations](https://github.com/pulp-platform/axi/blob/master/src/axi_intf.sv) and import them into the project, this way, if I use one of their AXI IPs in the future, I can just drop in their code, but this is just syntax so I'll stop yapping and start presenting some results.

The TB is pretty straightforward, we add a testcase just like the previous one, we send a fair amount of state updates in order to fill both the BRAM and our golden reference with meaningful data. Then we start initiating, from the TB, AXI LITE transactions over some prices to probe what's going on from a SoC master perspective.

We then compare the returned results with the golden reference. This test passing shows that our AXI LITE slave FSM is well implemented and does not miserably fail, which is kinda important, but also that data is correct. There is no *price <=> address* translation here, it is assumed the master knows that address X corresponds to price Y (which is the case given it's my demo and I do whatever I want).

## Implementation Within the **TRADEMAXXER** Top Module

So now "all that's left to do is to plug the price ladder into the main **TRADEMAXXER** top module and see how it goes.

{: .prompt-info }

> Reminder: the top level trademaxxer contains the whole ethernet MAC, the MoldUDP and ITCH parser + order book and now the price ladder as well. The input of this simulation are "real" (standard) ethernet frames built from the ITCH data provided by the NASDAQ itself, so the result below ran on a real trading day that happened back in 2019 or something, not sure of the exact date tbh)

To make this simulation a bit more understandable, at the end of it, we'll simulate (using [a cocotb extension](https://0bab1.github.io/BRH/posts/TIPS_FOR_COCOTB/)) an AXI LITE master that will recover the entire price ladder and we'll dump the result in a nice text file we can observe to find the obvious issues. (spoiler, there will be some, nothing ever works first try anyway).

And after some debugging, here is the result on a simulation that sent to our DUT the first 5000 messages of the trading day that are related to the AAPL stock, under the form of ethernet frames of course (yes I switched back to AAPL to get some volume, otherwise the parser takes ages (2 minutes) to find relevant messages)

![Price ladder result img](/assets/img/price_ladder_dump.png)

{: .prompt-info }

> Reminder : I prefilter the ITCH messages before sending them to the DUT so the waveforms are easier to debug and so the dump file doesn't take 10GB of space.

**And it's looking pretty good !** On the left are the **bid** quantities, on the right the **ask** quantities and in the middle the associated price bin (adjustable via params in the HDL) and the real BRAM address that stores this value.

**But of course there are some problems...** As you can see, we do have some obvious problems (highlighted in red) where we have a lone order crossing the spread, which is impossible in normal circumstances.

Again, the reason may be an unexpected bug but, as expected, it's about a collision from further upstream. In the screenshot below, on the far left hand side is a marker of the price ladder being written at the exact price where the order crossed the spread and on the right the last write that happened there. We have an add first on a specific ref and an executed message last.

![testbench showing collisions](/assets/img/tb_collision_demo.png)

In between, the orders book received the same ref twice, but wrote to the associated address multiple times (dashed blue lines) due to bad address mapping causing expected collisions. Anyway, this corrupts data and causes some problems like this. So I guess the next big fix is to make sure the address mapping is right, but that will be for a future post, as I also have a real job and limited time during my weekends during which I want to conduct other projects.

## What is next ?

Well as we just discussed: fix the order book address mapping to completely mitigate collision risks.

And.. well... test it on a real FPGA !

Thank you for reading to this point. You can write a comment below if you have any questions.

*Godspeed*

-BRH
