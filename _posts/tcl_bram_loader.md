---
title: "Vivado : JTAG to AXI Master tcl tips"
date: 2024-11-21 15:00:00 +0800
categories: [Tutorials]
tags: [fpga, vivado, tcl, axi]
pin: false
math: false
mermaid: false
image:
  path: https://adaptivesupport.amd.com/sfc/servlet.shepherd/version/renditionDownload?rendition=THUMB720BY480&versionId=0684U00000HqipE&operationContext=CHATTER&contentId=05T4U00000wVTiJ&page=0
  alt: Cocotb waveforms example on my HOLY CORE course
---

<!-- todo -->
<!-- Also make a post on : HOLY CORE, and building a plane-->

## JTAG To AXI Master in vivado : Intro

Tha JTAG to AXI Master block is great to tinker around an SoC.

It's very versatile as the nature of JTAG is pretty much "live chip debugging" and the AXI_M interface allows a connection to almost anything.

But most important of all, it allows us to use `tcl` scripts to send AXI transcactions in the system !

## In this post

In this blog post, I will provide some tips to

- Make a very basic system in xhixh we can use JTAG <> AXIM to blink an LED
- Perform Reads for debugging
- Use JTAG <> AXIM to instantiate a BRAM memory inside a more complex system

## Prerequestites

- Vivado
- Basic knowledge of `TCL`
- Basic knowledge of the `AXI` protocol

## Basic LED Blink

For starters, as always, let's start by blinking an LED.

I'll use a Zybo z7-20 board for this post's exmaples but you can use whatever you want.

![zybo z7-20 board](https://raw.githubusercontent.com/0BAB1/HOLY_CORE_COURSE/refs/heads/master/images/zybo.jpg)

So we'll start by opening vivado and starting a new project. Then open your block design and add the following :

- AXI GPIO
- JTAG <> AXIM

> Take a second to write the GPIO base address down for later when we'll send the AXI transactions, in my case : `0x40000000`
{: .prompt-warning }

Then run  all the connection automations (selects LEDs for GPIO config)

![SoC](https://image.noelshack.com/fichiers/2025/01/3/1735724681-jtag.png)

Once this is done, you can create a couple of constraints, an HDL wrapper and generate the bistream. Then program the device and open the hardware manager.

Once in the hardware manager we can start working on the TCL script :

```tcl
# led_blink.tcl

reset_hw_axi [get_hw_axis hw_axi_1]

set gpio_address 0x40000000
set wt axi_gpio_wt
create_hw_axi_txn $wt [get_hw_axis hw_axi_1] -type write -address $gpio_address -len 1 -data {00000001}
run_hw_axi [get_hw_axi_txns $wt]

```

As you can see, the script is pretty self explainatory :

- `reset_hw_axi [get_hw_axis hw_axi_1]` resets the AXI ang uses `get_hw_axis` to make sure we have a working JTAG <> AXIM that vivado recognizes
- We set the base address and declare `wt` wich stands for "*write transaction*"
- We create a transaction taht sends a `1` at the GPIO base address, effectively turning an LED on.
- We execute the transaction

And then you can go to TOOLS > Run TCL Script.. and select your script. The led should on.

## Reading

