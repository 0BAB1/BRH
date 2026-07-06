---
title: "FPGA HFT Order Book: Part 7, Lowering Hash Table Collisions... Again"
date: 2026-06-30 16:00:00 +0800
categories: [Projects]
tags: [finance, fpga]
pin: false
math: false
mermaid: false
---

<!-- 2026-06-30-Trademaxxer_handling_collisions_2.md -->

## Introduction & Context

In the previous posts, we designed an entire custom ITCH parser, going all the way from raw ethernet parsing to having a market state.

We also, in the [part6 post](https://hugobrh.dev/posts/Trademaxxer_handling_collisions/), started to handle the references using a hash function, drastically lowering the collision rate.

My objective now is to make a real FPGA demo, with a sort of trading frontend or something like that. But before doing that, we need to spend a little bit more time optimizing our memory management method and make the system a bit more robust.

For an **extremely high stakes** system such as the **TRADEMAXXER**, we need to completely avoid collisions. This objective is theoretically impossible, but practically doable if we play our cards right.

## Adding a FIFO / Buffer

First of all, the trademaxxer was, until now, a 100% parsing system, that only parsed information on the fly without much processing.

Now that we are going to introduce a more advanced pipeline to handle collisions better, some stages may need to stall the data stream for X number of clock cycles (variable delay if we want to add cache and DDR access for larger memory availability), **during which ITCH data keeps coming in over ethernet!**

This is why I designed a simple FIFO able to serve as a buffer between the input FSM (responsible for committing micro operations to the pipeline) and the `order_book` pipeline:

![new pipeline with FIFO buffer](../assets/img/maxxer_fifo.png).

## Cuckoo (Double) Hashing

I'm not gonna pretend to be a nerd who knows how to optimize memory solutions and data structures, because leetcode always pissed me off severely (personally). BUT sometimes we do need to use some computer science nerdy principles, and this is one of those times.

Until now, we had many collisions, and no real way to handle them.

A first solution to make that better: we can store the reference alongside order metadata to check whether the hashed address is occupied or not, and if yes, with which reference (*will be useful later, a simple flag won't cut it...*).

That is a good first step; once you know what lives inside a slot you try to write to, you can start probing for a new available space, but this is **super inefficient (around O(logn) or slightly lower AT BEST)** IMO for hardware applications, this is the kind of stuff software does because they have no other solutions.

What we can do instead is use **2 hash functions per reference**. This way, if a slot is not available for a given ref at a given hash, we can use a fixed alternative! Which drastically decreases the chances of a collision occurring.

This method is great for quick fixed duration lookup but is not gonna be 100% efficient (i.e. collisions WILL occur before 100% of our memory space is used), but we'll have more tricks up our sleeves for that later!

Also, it means we need to add some complexity to check what slot is free to write to, where our ref is stored etc, which is a big downside, but not as big as [hardware probing solutions](https://en.wikipedia.org/wiki/Cuckoo_hashing) in my opinion.

So we can now imagine a pipeline that would look more like this:

![Trademaxxer with cuckoo hashing and control unit](../assets/img/maxxer_control.png)

Now stage 0 gets a lot of added complexity:

- It generates 2 different fixed hashes per ref.
- Looks up both addresses, one at a time, and checks which is free / or contains the requested ref, depending on op type.
- Chooses a definitive address for the given ref.
- The rest of the pipeline does not change much, it writes to the chosen address and passes the result to the ladder.

A `done` handshake signal allows the first stage to signal the buffer *when* it's done with the lookup, 

After a couple of days of experimenting, we get the following ladder result:

![Cuckoo hashing ladder](../assets/img/cuckoo_ladder.png)

## Adding a Stash for Remaining Collisions

It's very good! But high load stress testing easily reveals that collisions are still an issue

> even though they are not visible in the ladder, as the stress in the real data test is not high/long enough to create visible chaos.

![order book stress testbench showing a collision](../assets/img/maxxer_cuckoo_collision.png)

In this testbench, you can see in the last `replace` micro operation that the `no_space` exception signal gets asserted, showing that despite our cuckoo solution, collisions still occur.

That is because 2 hashes lower collisions between 2 references, making them much less likely, but still, as you fill up the small memory space, probabilities of collisions increase dramatically!

This signal can be used by a CPU to handle the exception, but we can still add some hardware to handle these edge cases: add a stash!

We don't need unlimited memory (because orders come and go), and to handle the little amount of collisions that **may and will** happen using our double hashing technique, we can simply add a stash that will be implemented as a simple collection of registers ready to be accessed at any given time.