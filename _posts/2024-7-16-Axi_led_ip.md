---
title: Build you own 1st AXI LED IP
date: 2024-7-16 15:00:00 +0800
categories: [Tutorials, Zynq]
tags: [zynq, fpga, xilinx]
pin: false
math: false
mermaid: false
image:
  path: https://shop.trenz-electronic.de/media/image/07/1d/e2/TE0715-04-51I33-A_1.jpg
  alt: Zynq SoC
---

# Building your own AXI LED IP

Based on this [video](https://youtu.be/zJJTxOT37K4).
{% include embed/youtube.html id='zJJTxOT37K4' %}

> Level : Beginner in fpga (See prerequisites below).
{: .prompt-info }

## [The repo containing HW & SW code](https://github.com/0BAB1/BRH_Tutorials/tree/main/2%20AXI%20IP%20Hello%20world%20custom%20LED%20driver)

## Intro

So you decided you want to learn more about how to actually use your ZynQ Board and get your hands dirty ? Well, let me tell you are at the right place.

FROM designing a simple hello world design to understand the workflow
TO building your own IP over the AXI protocol,

This small post & video will make you go from newbie to (*almost*) proficient in engineering.


## Prerequisites

- Know the **basics** of C (Not much, just a simple pointer, literally 2 lines of code)
- Know the **basics** of HDL (Not much needed)
- Have Vivado & Vitis installed
- Have a very simple FPGA to test it out with an LED hooked up to it.

## Layout

> This tutorial is full-video, here is a layout of what you'll learn and some points of information.
{: .prompt-tip }

### Part 1 : Hello world

1. Implement the Zynq Processing Sytem (PS) (Embedded ARM CPU) in vivado
2. Add xilinx's base I/O IP (we'll see how to make ours in a minute)
3. Run sofware on it and learn how to interface with the FPGA using software running on the PS (Using Vitis) 

### Part 2 : Make our own LED Driver IP

1. Create a new AXI-LITE based IP project
2. Write some HDL logic that routes AXI signals to an output
3. Assign the out pin in the block design to an LED using constraint files
4. Run software on the PS to turn the LED on.

There may be some complications when it comes to routing the LED Signal all the way up to outside of the IP. If you take you time and ignore all the fancy useless signals xilinx leaves there by default, and you only focus on the user logic, it shall be good.

You also have schemes in the videos that should help understanding the routing matter.

If you have any complications, leave a comment under the video so I can troubleshoot you (I always read them all).

## Going further

Reads also work ! Don't hesitate to play around with the code to read value from a button, do some logic in the FPGA or in the code (both can be good too) and mess around with the LED like that.

## Ressources

Here are some resources to debug AXI interfaces if you'd like to try more advanced stuff :

- [Xilinx AXI VIP as a master to verify your IP](https://support.xilinx.com/s/article/1058302?language=en_US)
- [Xilinx AXI VIP, use it for your projects & access testbenches](https://www.xilinx.com/video/hardware/how-to-use-axi-verification-ip-to-verify-debug-design-using-simulation.html)