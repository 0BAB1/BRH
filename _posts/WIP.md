---
title: "Trademaxxing part7 : Getting Rid of Order Ref Collisions"
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

We also, in the [part6 post](https://hugobrh.dev/posts/Trademaxxer_handling_collisions/), started to handle the reference using an hash function, drastically lowering the collision rate.

Of course, for a high stake system such as the TRADEMAXXER, we need to completely avoid colisions. We want a 0% collision rate ! To do so, we'll use a very clever design strategy...