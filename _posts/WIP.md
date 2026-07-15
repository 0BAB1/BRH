---
title: "FPGA HFT Order Book: Part 8, FPGA Synth & Impl"
date: 2026-07-17 16:00:00 +0800
categories: [Projects]
tags: [finance, fpga]
pin: false
math: true
mermaid: false
---

<!-- 2026-07-17-Trademaxxer_FPGA_1.md -->

## Introduction & Context

In the previous posts, we designed an entire custom ITCH parser, going all the way from raw ethernet parsing to having a market state.

**The overall functional block diagram:**

![Scheme 1: overall design](/assets/img/itch_soc_datapath.png)

We also; in the [part7 post](https://hugobrh.dev/posts/Trademaxxer_handling_collisions_2.md/); made it so the order book becomes collision free in simulation. (even though it's not perfect, it was a start).

Now that we have a solid HDL base, we can start working our way towards a real FPGA, by implementing the trademaxxer in FPGA fabric !

## First Synth

The first step we have to take, is to make sure our design can synth. I.e. is Vivado happy about our syntax and is the design using too much resources ?

So I create a new empty vivado project to strart testing things out. I will target the KC705 developpement board, which embeds a **XC7K325T** FPGA, which is a pretty *beefy* FPGA. It also has a PCIe connection, which can help later to build a frontend/monitoring application on a host PC.

![KC705 image](/assets/img/kc705.png)

{: .prompt-info }
> Real implemenation in finance will target more advaced boards, with >10G ethernet and HUGE fpgas ([example](https://www.mouser.fr/fr/ProductDetail/ReFLEX-CES/XpressVUP-LP5PT2)).

In vivado, I import my `trademaxxer.sv` top wrapper, which contains all the logic from ethernet parsing to price ladder RTL, and declare it as a top module.

![trademaxxer VIVADO basic hierarchy](../assets/img/basic_synth_hier.png)

The first raw synthesis gives us the following:

![1st synth results](../assets/img/itch_first_synth_results.png)

Unsuprisingly, the usage is pretty high, yet the FPGA is pretty big so we can bruteofrce our way into having something that fits whithin available fabric.

{: .prompt-info }
> Most of the FFs are used in the stash and most of the BRAM in the order book memory, as expected.

![utilization report](../assets/img/first_utilization_report.png)

We also see a large amount of DSP, most likely infered by the hash functions and the update quantity math (simple substractions).

## First Implementation

For our first implementation tests, we'll use a block deisng. Our goal is simple to instantiate the trademaxxer module, add constraints, make it run @ 125MHz (or a bit higher to avoid input ethernet async FIFO overflow) and add some AXI master to monitor the state of the price ladder during the first test runs.

The 1st implementation took ages (2hours) adn came up with shitty results timing-wise:

| **Setup**                    |                     | **Hold**                     |              | **Pulse Width**                          |              |
| :--------------------------- | :-----------------: | :--------------------------- | :----------: | :--------------------------------------- | :----------: |
| Worst Negative Slack (WNS):  |    **-57.045 ns**   | Worst Hold Slack (WHS):      | **0.064 ns** | Worst Pulse Width Slack (WPWS):          | **0.264 ns** |
| Total Negative Slack (TNS):  | **-9858536.534 ns** | Total Hold Slack (THS):      | **0.000 ns** | Total Pulse Width Negative Slack (TPWS): | **0.000 ns** |
| Number of Failing Endpoints: |      **270368**     | Number of Failing Endpoints: |     **0**    | Number of Failing Endpoints:             |     **0**    |
| Total Number of Endpoints:   |      **283335**     | Total Number of Endpoints:   |  **283271**  | Total Number of Endpoints:               |  **138276**  |

> bruh.

That was of course expected as our logic is currently extremely **UNOPTIMIZED**. After some investigation, turns out the critical path is indeed the stash lookup, as we expected in the [part7 post](https://hugobrh.dev/posts/Trademaxxer_handling_collisions_2.md/). The stash is comparing a 64bits reference against 1024 entries, which is brutally bad for timing (*duh*).

So multiple options become available:

1. Pipeline the lookup
2. IDK

Option 1 looks like it's the easiest to implement. To be honest, I'm too tired to think about other solutions right now.

What we can look into is register banks.

This means we separate our register "array" insto smaller chunks (banks). This allows for the "cheking logic" (i.e. is the ref we are looking for present here ?) to run in parllel for each of these chunks.

Here is a little diagram of how we'll implement it :

