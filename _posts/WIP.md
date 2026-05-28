---
title: "Matlab & Vivado Cosimulation" // 2026-05-28-Matlab-Vivado-Cosimulation
date: 2026-05-28 16:00:00 +0800
categories: [Tutorials]
tags: [fpga, software]
pin: false
math: false
mermaid: false
---

## Introduction

At my job, we're working on an optical PLL system. ([context paper link for those interrested](https://zenodo.org/records/19882710))

The physics are simulated in MATLAB. Part of this system is the phase recovery algorithm, which is implemented on FPGA.

The MATLAB Simulink model currently implements the phase recovery algorithm using Simulink blocks. The thing is, given how sensitive a closed-loop system can be to delays and noise, this representation is decent but **not ideal** as we lack the actual hardware behavior (quantizations, real delays, real sampling etc...).

So I recently had what I think is a pretty good idea to tackle more advanced demos with more confidence in our setup:

*Instead of modeling our HDL design using Simulink blocks, what if we just used the actual HDL in the simulation?*

Which sounds great on paper. In fact, MathWorks had the same idea a long time ago. HDL/Simulink co-simulation is a supported thing since before I was born (I'm exagerating here but yeah it's not exactly new).

**Except it's a massive pain to set up.** Like a real massive pain. And given the time vivado can take to install (2+hours on a good setup), tinkering around to set things up is not only painful, but also **very time consuming**.

So today, I'll walk you through my experience getting this co-simulation running, hoping to save you the days of tinkering with Vivado and MATLAB I had to go through to make everything work.

## Software Versions

The first thing that took me days to figure out, is the exact version combo that makes things works "flawlessly".

If you don't get these versions right, you **WILL** get segfaults crashes that are non worth trying to fix (I tried but failed miserably and resorted to try and find the optimal verion combo).

So here is the combo that worked for me:

- Vivado 2024.1
- Matlab 2024b

{: .prompt-info }
> Note I use Ubuntu 24.04.

And that is pretty much it. But it takes so long to install these softwares that the time spent on this part can quickly add up.

{: .prompt-info }
> You will also need to get all the Matlab HDL Verifier add ons.

You can find the official supported version combos here: https://fr.mathworks.com/matlabcentral/answers/518421-which-versions-of-vivado-are-supported-with-which-release-of-matlab. But this did not work for me so... Yeah.

## Drivers Et al. 

Before moving on to the Matlab self guided setup, you need to install a whole bunch of drivers (which is a step that isn't specified anywhere and that you get down by spending hours debuging stuff).

### Installing Vivado Cable Drivers.

That should not be a problem as this step is often completed whenever you install vivado on a new linux machine, but just in case, I add it here.

You simply have to execute the **very well hidden** driver installation script in you vivado installation folder, for me it was:

```sh
cd /tools/Xilinx/Vivado/2024.1/data/xicom/cable_drivers/lin64/install_script/install_drivers
```

And then:

```sh
sudo ./install_drivers
```

### Installing FTD2XX Drivers.

These drivers are supposedly downloaded by matlab when installing the Malab self guided tutorial add ons (we'll get to that later) But they apparently don't work, meanning you have to do it yourself.

To do so, go to this link: https://ftdichip.com/drivers/d2xx-drivers/

And download the FTD2XX drivers, a readme.pdf will guide you through the manual installation process.

### Installing Digilent Adept Drivers.

You also have to install the Digilent Adept Drivers. It's specified **nowhere** except in burried blog posts on Mathlab forums.

Anyway, go to this link : https://digilent.com/shop/software/digilent-adept/

Create an account and download / install the "Adept Runtime".

### Matlab Launch Script & Setup

This setup part sill solve 2 problems:

1. Ubuntu somehow refuses to make user input work on matlab in some ditros, making us unable to use the software properly (including mine, of course), these is a workaround for that we can add to a launch script. [link](https://fr.mathworks.com/matlabcentral/answers/1456459-can-t-enter-text-when-installing-r2021a-on-ubuntu-20-04)
2. Matlab cannot find essential vivado libraries, we need to add them to `$LD_LIBRARY_PATH`.

Here is a script that works for me to fix both these bugs:

```sh
#!/bin/bash

# Linux issues with user inputs workaround https://fr.mathworks.com/support/bugreports/1797911
export ENABLE_QWEBWINDOW=true;

# If repeated segfault, add some swap :
# sudo fallocate -l 32G /swapfile
# sudo chmod 600 /swapfile
# sudo mkswap /swapfile
# sudo swapon /swapfile
# make it perm : 
# echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
# https://adaptivesupport.amd.com/s/question/0D52E00006vFXkwSAG/is-there-a-workaround-for-vivado-rdiprog-crash-during-synth-while-running-20211-on-ubuntu-18046-lts?language=en_US

# Launch Matlab
export LD_LIBRARY_PATH=/tools/Xilinx/Vivado/2024.1/lib/lnx64.o:$LD_LIBRARY_PATH
matlab
```

> Note the swap thing, that was one fix I tried when dealing with segfaults because of version incompatibility. I leave it there just in case you still have segfault issues with the same setup as me because this is a potential fix.

## HDL Verifier Self Guided

Okay, now that we have a sane setup, we can finally start *trying* to work on our co-simulation scenario.

Here I'll focus on one goal : Simulate one existing hand-written systemVerilog submodule in simulink.

That's it, but it's pretty tedious...

Fortunatly for us, Mathworks got us covered with a self guided tutorial we can follow to setup our environement (modulo everything we had to do ourselves before lol).

You can get this tutorial here : https://fr.mathworks.com/matlabcentral/fileexchange/181146-hdl-verifier-self-guided-tutorial.

Go to Files > Download and open the PDF to start follwing the tutorial.

To achieve our previousl stated goal, we need to focus on two parts of this tutorial:

1. Hardware Support Package Installation
2. Demo 2: *HDL Cosimulation with Handwritten HDL Code*

{: .prompt-warning }
> You need the proper vivado licences for Part 1. depending on the board you chose to run the setup. Matlab does gives you choice for the board, maybe you can use a lower end board but I dont really know, I used a KC705, which requires a licence. Watch out for that !

### 1: Hardware Support Package Installation Problems Troubleshoot.

The "*Hardware Support Package Installation*" step is *theorically* optional, as our goal is not to use an actual FPGA in-the-loop simulation, but running this correctly proves that all our drivers are well installed and that our setup is tight, which is pretty much mandatory.

The self guided tutorial is pretty straight forward, but in the "*Hardware Support Package Installation*" step, you may face big challenges to install the hardware support packages.

I personally got an error when trying to install the FTD2XX drivers.

{: .prompt-info }
> By the way, I tried some versions where I did not get this error for FTD2XX, but still had to install them myself, lol. Great work right there. Anyway, let's continue...

After some research, in a burried post, we can find a workaround: https://fr.mathworks.com/matlabcentral/answers/2017321-the-404-error-when-installing-hdl-verifier-support-package-for-xilinx-fpga-boards#answer_1575225

The workaround also works like a charm for Linux.

### 2. HDL Cosimulation errors

#### Timing related

#### No .so found related

If your matlab launch script is well setup, libraries should be found, meaning the errors comes from vivado failling to build simulation files for your design, the main matlab console dumps the vivado outputs, you can use this to investigete.

If vivado successfully built the simulation files, that means you still have some library tinkering to do. refer to the "Installing FTD2XX Drivers." chapter.

## The Actual Cosimulation

Okay, after all of this, you *may* finally be granted the right 