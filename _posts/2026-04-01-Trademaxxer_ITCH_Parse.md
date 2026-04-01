---
title: "Trademaxxing part2 : ITCH Parsing FSM"
date: 2026-03-31 16:00:00 +0800
categories: [Projects]
tags: [finance, fpga]
pin: false
math: false
mermaid: false
---

## Introduction

In [the last post](https://0bab1.github.io/BRH/posts/Trademaxxer_MoldUDP64-copy/) I presented the ITCH protocol which is Nasdaq's way to broadcast live market data to trading systems.

We also designed a simple `MoldUDP64` parser, `MoldUDP64` being the the protocol that encapsulate ITCH messagesand that also adds somr etracking metadata.

Today, we are going to have a look at how to make sense of these parsed messages and how to do some "bookkeeping" on the markets states, as remember, ITCH is all about events, meanning you have to vuild the market state "as you go".

## Objective

For now, the objective will not to implement a strategy acting directly upon incomming messages, but rather use these maessage to do some bookkeeping.

So first thing first, let's define **what** we want to keep track of.

Well I sugeest we keep track of:

1. The **List** of orders
2. The bid/asks ladder, also called "*order book*" but this is gonna be confusing so I'll refer to it as "**the ladder**".

{: .prompt-info }
> The **Order lists** is pretty explicit, but in case you don't know, here is what I refer to as the bid/ask ladder :

>![ladder image](/assets/img/ladder.png)

With that in mind, let's outline what the design shall look like:

![FPGA overview](/assets/img/bookkeeping_fpga.png)

With the two `LADDER` and `TABLE` tables kept as BRAMS, more on that in future posts as here we'll really focus on the **ITCH PARSER**.
A smarter way to go about this would be to add an hashmap system and collison handler. Or a cache system. But I'm too lazy for that right now.

{: .prompt-info }
> Side note, if possible, we'll upgrade the FPGA fabric speed to ensure data is always consumed faster that it arrives. This is essential as if frames start ariving faster than we consume it in the FPGA, data will start to become corrupted as the fifo will wrap aoround and start erasing old data that we did not consume yet to make space for the new. I fact I use an open source FIFO, and I don't know what it does when this happens, but the best is just not to find out by making sure `f_fabric > f_rx`.

## Designing an ITCH parser and Figuring Out "State Changes" Commints

From there, we have to create the ITCH parser logic. Again, it will revolve around an FSM that will collect all the informations it needs at the incoming message unrols live, and that will simply drop progress by going back to IDLE if an error (eraly `tlast` signaling is detected).

First, we need to dtermine what messages we want to support for our basic needs:

- A, F : Adds
- E, C : Executions 
- X : Cancel (partial qty)
- D : Delete
- U : atomic replace

{: .prompt-info }
> I will let you check the specialities of each order type, you'll need dig into it anyway to design the following FSMs.

The FSM that it revolves around figues out on-the-go what we are dealing with and starts collecting the information we'll need to generate the `Orders` state commit signals:

![itch FSM hand sketch](/assets/img/ITCH_FSM.png)

So that's an FSM I drew on paper (*which is the best way to get things right btw*). Making yet another FSM here will help this module really figure out whats going on and allow us to store the future state changes commit information as the message comes in without adding any unecessary buffering.

{: .prompt-info }
> Most of  these staes do the same thing and the final FSM will be refactored, don't worry. You may also notice the "R" event in the FSM, I decided not to support it as its full of bloat informations, adding tons of specific states. Initially, I planned on using this event message to set the LOCATE property for stock I wanted to track, but in a context where I only wanna make a 1 stock demo tracker, and where the whole design does not bufferrize anything and just "follows the stream", "early filtering" is useles anyway.

Once the parser have gone through all its states, The ideal and clean way to **commit** state changes would be to add a `COMMIT` state to the FSM after each message end. During this state, for simplicity, we won't process incomming data and simply de-assert `tready`. (we could but I like to avoid non-necessary complexity).

**BUT** the ethernet RX MAC I designed does not handle frontpressure and **always assumes `tready == 1`**... So deasserting `tready` is not really an option... Except if... `f_fabric > f_rx`.

Why ? Because this would be a **garantee** that the RX FIFO will be empty **most of the time** and will be able to buffer a little bit.

Now we can't really add some gigalong `COMMIT` state to **commit the state changes** to the **orders list** and hope for the best, but we do have some wiggle room if we manage `f_fabric > f_rx` later on when implementing the design. 
130-150MHz is a reasonable target so aiming for let's say 150Mhz later for the fabric is not insane, in fact, the simplicity of the parsing pipeline will allows to add some registers on the AXIS interfaces, making timing closures ammters easier later.

Anyway, in a nutshell, we'll assume `f_fabric > f_rx`. and add a small `COMMIT STATE` that wil de-assert `tready` as we won't be processing any message while commiting, this would last 1 cycle and then go back to `IDLE` to process another message.

{: .prompt-info }
> I'll also add a `DRAIN` state that will just ignore the message. This will serve as primitive error handling and to ignore unsuported message we obviously don't care about.

**Okay that's enough yapping for FSMs technical problem, let's move on shall we ?**

So we put hat state machine into SystemVerilog, and code each state role : storing the adequate information so that the parser knows what tjust happenned in the market and so that we can commit **correct state updates** when arriving at `COMMIT` state.

Once at the the `COMMIT` state (i.e passed ALL filters,everything went wel etc..), we can pass update data, an opcode and a valid flag to the next componenet which will be the oerders table.

## First Simulation

{: .prompt-info }
> Won't get to much in the technical RTL here cuz its not the subject. The SYSTEM & the PROJECT are.

Just like I mentionned in the first post, I use a cocotb + verilator stack with cocotb extensions to simulate already verified standard RGMII signal embedding a fixed frame that I (& claude) put together using the specs. It's easy and fast to do and allow for quick verification and tweaks.

Here is a simulation of the itch parser running, gathering & filtering the incomming data on the fly (no buffering delays) and then passing on only the relevant state update info the the following orders table (we still didn't design it yet)

![basic itch testbech on "A" event message](/assets/img/basic_itch_tb.png)

Now some stuff remains to do on my side : Support all the event type message necessary to do some bookkeeping AND make it so the tesbench uses actual NASDAQ ITCH 5.0 data frames so we can really verify our design against more real data and tweak the necessary thing to make it work

But that will be another post.

Thank you for reading to this point. You can write a comment below if you have any question.

*Godspeed*

-BRH