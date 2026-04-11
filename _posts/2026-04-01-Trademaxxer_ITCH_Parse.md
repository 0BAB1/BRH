---
title: "Trademaxxing part2 : ITCH Parsing FSM"
date: 2026-04-01 16:00:00 +0800
categories: [Projects]
tags: [finance, fpga]
pin: false
math: false
mermaid: false
---

## Introduction

In [the last post](https://0bab1.github.io/BRH/posts/Trademaxxer_MoldUDP64-copy/) I presented the ITCH protocol which is Nasdaq's way to broadcast live market data to trading systems.

We also designed a simple `MoldUDP64` parser, `MoldUDP64` being the protocol that encapsulates ITCH messages and that also adds some tracking metadata.

Today, we are going to have a look at how to make sense of these parsed messages and how to do some "bookkeeping" on the market states, as remember, ITCH is all about events, meaning you have to build the market state "as you go".

## Objective

For now, the objective will not be to implement a strategy acting directly upon incoming messages, but rather use these messages to do some bookkeeping.

So first thing first, let's define **what** we want to keep track of.

Well I suggest we keep track of:

1. The **List** of orders
2. The bid/asks ladder, also called "*order book*" but this is gonna be confusing so I'll refer to it as "**the ladder**".

Just a side note on what these thing are: the **Order lists** is pretty explicit (keeps track of the orders state), but in case you don't know, here is what I refer to as the bid/ask "**ladder**" :
![ladder image](/assets/img/ladder.png)

With that in mind, let's outline what the design shall look like:

![FPGA overview](/assets/img/bookkeeping_fpga.png)

With the two `LADDER` and `TABLE` tables kept as BRAMs, more on that in future posts as here we'll really focus on the **ITCH PARSER**.
A smarter way to go about this would be to add a hashmap system and collision handler. Or a cache system. But I'm too lazy for that right now.

{: .prompt-info }
> Side note, if possible, we'll upgrade the FPGA fabric speed to ensure data is always consumed faster than it arrives. This is essential as if frames start arriving faster than we consume them in the FPGA, data will start to become corrupted as the FIFO will wrap around and start erasing old data that we did not consume yet to make space for the new. In fact I use an open source FIFO, and I don't know what it does when this happens, but the best is just not to find out by making sure `f_fabric > f_rx`.

## Designing an ITCH parser and Figuring Out "State Changes" Commits

From there, we have to create the ITCH parser logic. Again, it will revolve around an FSM that will collect all the information it needs as the incoming message unrolls live, and that will simply drop progress by going back to IDLE if an error (early `tlast` signaling) is detected).

First, we need to determine what messages we want to support for our basic needs:

* A, F : Adds
* E, C : Executions
* X : Cancel (partial qty)
* D : Delete
* U : atomic replace

{: .prompt-info }

> I will let you check the specificities of each order type, you'll need to dig into it anyway to design the following FSMs.

The FSM that it revolves around figures out on-the-go what we are dealing with and starts collecting the information we'll need to generate the `Orders` state commit signals:

![itch FSM hand sketch](/assets/img/ITCH_FSM.png)

So that's an FSM I drew on paper (*which is the best way to get things right btw*). Making yet another FSM here will help this module really figure out what's going on and allow us to store the future state changes commit information as the message comes in without adding any unnecessary buffering.

{: .prompt-info }

> Most of these states do the same thing and the final FSM will be refactored, don't worry. You may also notice the "R" event in the FSM, I decided not to support it as it's full of bloated information, adding tons of specific states. Initially, I planned on using this event message to set the LOCATE property for the stock I wanted to track, but in a context where I only wanna make a 1 stock demo tracker, and where the whole design does not buffer anything and just "follows the stream", "early filtering" is useless anyway.

Once the parser has gone through all its states, the ideal and clean way to **commit** state changes would be to add a `COMMIT` state to the FSM after each message end. During this state, for simplicity, we won't process incoming data and simply de-assert `tready`. (we could but I like to avoid unnecessary complexity).

**BUT** the ethernet RX MAC I designed does not handle backpressure and **always assumes `tready == 1`**... So deasserting `tready` is not really an option... Except if... `f_fabric > f_rx`.

Why ? Because this would be a **guarantee** that the RX FIFO will be empty **most of the time** and will be able to buffer a little bit.

Now we can't really add some gigalong `COMMIT` state to **commit the state changes** to the **orders list** and hope for the best, but we do have some wiggle room if we manage `f_fabric > f_rx` later on when implementing the design.
130-150MHz is a reasonable target so aiming for let's say 150MHz later for the fabric is not insane, in fact, the simplicity of the parsing pipeline will allow adding some registers on the AXIS interfaces, making timing closure matters easier later.

Anyway, in a nutshell, we'll assume `f_fabric > f_rx` and add a small `COMMIT STATE` that will de-assert `tready` as we won't be processing any message while committing, this would last 1 cycle and then go back to `IDLE` to process another message.

{: .prompt-info }

> I'll also add a `DRAIN` state that will just ignore the message. This will serve as primitive error handling and to ignore unsupported messages we obviously don't care about.

**Okay that's enough yapping for FSMs technical problem, let's move on shall we ?**

So we put that state machine into SystemVerilog, and code each state role: storing the adequate information so that the parser knows what just happened in the market and so that we can commit **correct state updates** when arriving at the `COMMIT` state.

Once at the `COMMIT` state (i.e passed ALL filters, everything went well etc..), we can pass update data, an opcode and a valid flag to the next component which will be the orders table.

## First Simulation

{: .prompt-info }

> Won't get too much into the technical RTL here cuz it's not the subject. The SYSTEM & the PROJECT are.

Just like I mentioned in the first post, I use a cocotb + verilator stack with cocotb extensions to simulate already verified standard RGMII signals embedding a fixed frame that I (& claude) put together using the specs. It's easy and fast to do and allows for quick verification and tweaks.

Here is a simulation of the ITCH parser running, gathering & filtering the incoming data on the fly (no buffering delays) and then passing on only the relevant state update info to the following orders table (we still didn't design it yet)

![basic itch testbench on "A" event message](/assets/img/basic_itch_tb.png)

Now some stuff remains to do on my side: support all the event type messages necessary to do some bookkeeping AND make it so the testbench uses actual NASDAQ ITCH 5.0 data frames so we can really verify our design against more real data and tweak the necessary things to make it work.

But that will be another post.

Thank you for reading to this point. You can write a comment below if you have any questions.

*Godspeed*

-BRH
