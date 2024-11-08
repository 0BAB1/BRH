---
title: "Direct Memory Access (DMA) : A hands on Zynq example"
date: 2024-7-21 15:00:00 +0800
categories: [Tutorials, Zynq]
tags: [fpga, beginner, C]
pin: false
math: false
mermaid: false
image:
  path: https://media.geeksforgeeks.org/wp-content/uploads/20240109112312/screen.png
  alt: DMA Overview
---

# DMA Tutorial : A hands-on example using Fast Fourier Transform (FFT) IP

Based on this [video](https://youtu.be/aySO9jCKj9g).
{% include embed/youtube.html id='aySO9jCKj9g' %}

> Level : Beginner +. If you come from the "begginer" section tag, please make sur you did the "implementing your own IP" tutorial (or that you have similar knowledge).
{: .prompt-info }

## [Find the example FFT DMA bare-metal firmware here](https://github.com/0BAB1/BRH_Tutorials/tree/main/4%20DMA%20tutorial)

## Intro

DMA is a way to access memory without using loads instructions from you CPU.

The CPU asks the DMA to handle memory transactions for him, allowing the cpu to do more valuable tasks.

> You can see DMA as a "memory co-processor"
{: .prompt-tip }

In this video, I walk you through a hands-on example to implement you first DMA on Zynq (or any FPGA if you know how to implement a softcore CPU) and how to interface it with an FFT IP as an exmaple device to leverage you IP designs without stalling to CPU to move data around.

## Prerequisites

- Know the basic of Vivado & vitis (see "implementing your own AXI LED IP" tutorial)
- You **don't** need to understand FFT deeply to complete this tutorial, the FFT is just an exmaple.
- Have vivado & vitis installed.

## Video layout

### 1 : What is DMA

This first part is all about explaining what is DMA. It's pretty straight-forward.

### 2 : Implementing a FFT IP (Hardware)

This part is about putting all the hardware toghther. We don not implement a custom IP but a pre-made FFT IP from xinlinx to compute fourier transforms FAST !

Given that we are not designing any IP this tutorial does not include any HDL.

### 3 : Implementing a FFT IP (Software)

In this part, We break down the firmware code logic and make our own firmaware so you can go on a make your own for your projects.

> IMPORTANT NOTE if you plan on using UART like in the tutorial : do not use ```xil_printf()``` but rather ```printf()``` to handle the display of floats correctly.
{: .prompt-warning }