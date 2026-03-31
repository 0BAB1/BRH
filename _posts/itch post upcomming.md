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
- R : Stock directory

{: .prompt-info }
> I will let you check the specialities of each order type, you'll need dig into it anyway to design the following FSMs.

The FSM that it revolves around figues out on-the-go what we are dealing with and starts collecting the information we'll need to generate the `Orders` state commit signals:

![itch FSM hand sketch](/assets/img/ITCH_FSM.png)

So that's an FSM I drew on paper (*which is the best way to get things right btw*). Making yet another FSM here will help this module really figure out whats going on and allows us to store the future state changes commit information as the message comes in without adding any unecessary buffering.

That means we have to get the timings right as backpressures handling is not an option... Except if... `f_fabric > f_rx`.

This would be a **garantee** that the RX FIFO will be empty **most of the time** and will be able to buffer a little bit.

Now we can't really add some gigalong state to **commit the state changes** to the **orders list** and hop for the best, but we do have some wiggle room if we manage `f_fabric > f_rx` later on when implementing the design. 
MHz is a reasonable target so aiming for let's say 150Mhz later for the fabric is not insane, in fct, the simplicity of the parsing pipeline will allows to add some registers on the AXIS interfaces, making timing closures ammters easier later.

Anyway, in a nutshell, we'll assume `f_fabric > f_rx`. and add a small `COMMIT STATE` that wil de-assert `tready` as we won't be processing any message while commiting, this would last 1 cycle and then go back to `IDLE` to process another message.

**Okay that enough yapping for a non significant technical problem, let's move on.**

So we put hat state machine into SystemVerilog, and code each state role : storing the adequate information so that the parser knows what tjust happenned in the market and so that we can commit **correct state updates** when arriving at `COMMIT` stage.

