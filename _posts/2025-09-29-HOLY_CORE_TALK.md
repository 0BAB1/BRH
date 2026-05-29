---
title: "HOLY CORE : My biggest project"
date: 2025-09-28 15:00:00 +0800
categories: [Projects]
tags: [fpga, thoughts]
pin: false
math: false
mermaid: false
---

> This post is not technical but rather a way for me to share thoughts on the HOLY CORE project. For context, HOLY CORE is an *open source* RV32I **compliant** CPU that runs on FPGA and the entire development process is document for everyone to learn through an easy to understand course. I made some docs (yes, I did lol) here if you'd like to read more on the technical side of things. GLHF : https://0bab1.github.io/HOLY_CORE_COURSE/

## HOLY CORE : Presentation

For those of you who do not know holy core, here is the [youtube video](https://www.youtube.com/watch?v=ix8vlIM7Iv8) I made on the project :
{% include embed/youtube.html id='ix8vlIM7Iv8' %}

## Introduction

So here’s the deal.

I’m a normal guy.
A normal former student that learned a few things in college, has a few projects in my head, and wants to get them done.

But in reality: **school doesn’t teach you shit.**

I’ve always been into computer stuff, a “nerd” if you will, but I never went into computer science because the tech market was looking bad. Still, I was always curious about how computers *really* work.

And one day I though I could make my own CPU core and SoC... And teach people how to do it too by docomenting the whole learning process !

## The feeling you get when building a CPU CORE.

At its core, a CPU is a machine that does math.
Data is represented as **1s and 0s** — switches that are either ON or OFF. That’s it.
Using these switches, we build **logic gates**, and with enough of them, we can do math.

A CPU basically runs a loop:

* **FETCH** → get the instruction
* **DECODE** → understand it
* **EXECUTE** → do the math
* **MEMORY** → read/write if needed
* **WRITE BACK** → save results into registers

Registers are just super-fast local memory taht captre a signal state. Instruction after instruction, **YOUR** computing logic is executed. The whole thing feels like clockwork, kinda like building Big ben, and debugging the CORE is a special feeling as there is no abstraction level whatsoever, you either know how it works exectly and can use your brian to imagine where is went wrong, or you don't know how it works, and you wil **never** find the bug, as not error code dump in chat GPT will ever help you.

## Designing the CPU

So, how do you go from “that sounds cool” to an actual working CPU?

1. **Choose an architecture** → I picked **RISC-V** because it’s free.
2. **Describe the design** → I used **SystemVerilog**, a hardware description language.
3. **Test it** → Hardware description code is buggy by default, so I used **cocotb** + **verilator** (both free tools) to simulate the design with testbenches.

After setup, it was time to actually build the damn thing.

Starting from **nothing**, I had to add:

* Program counter
* Instruction memory
* Control unit
* Arithmetic Logic Unit (ALU)
* Registers
* Data memory
* Write-back logic

I started small: implementing just **one instruction** (`lw` = load word). From there, I added more instructions one by one, debugging with testbenches and waveforms until…

My CPU ran my test program correctly **in simulation**.

## From Simulation to Reality

But running in simulation is not the same as running in the real world.

To make it real, I used an **FPGA** (a chip with configurable logic blocks). The idea: program the FPGA to behave like my CPU.

Easier said than done.

The CPU needed to talk to the outside world to fetch data, instruction but also interact with MMIOs, so I had to implement an **AXI interface**. Painful. Then, the flaws in my design started showing:

* Clock domain crossing issues (AXI vs CPU running on different clocks)
* Cache system bugs (so broken I wondered how it ever passed simulation)
* Endless battles with **Vivado** (the FPGA tool I now both love and hate)

After *a lot* of suffering… I finally got **signs of life**:

* First LED blink
* Infinite idle loop
* Actual program running

It was messy, but it worked.

## Building a Tiny SoC

With a stable CPU core on the FPGA, the next step was building a mini **System-on-Chip** (SoC).

That meant:

* Hooking the CPU up to the outside world (LEDs, memory, etc.) via AXI
* Writing little software hacks to drive the cache properly
* Adding reset signals and glue logic

In the end, I had a chaotic terrbly no determenistic core. To this day, the core has evolved **A LOT** and is now compliant and usable. More on this in the next videos ;)

## HOLY CORE : *Exclusive* Debrief and Future Work.

### Youtube exposure

A year ago now, I started a youtube channel documenting my learning journey in the digital design. I always were fascinated by this field and I grabbed the opportunity to hop on it before graduating.

Long story short, I ended up thinking that learning this field was **hard** (maybe in purpose to gatekeep the knowledge, to protect the ield from AI, not enouh demand ?) so I buit a Risc-V core, documented the whole thing and made some nice tutorials.

The video got tons of exposure, which I'm very happy about, now programming FPGAs is literally my job. It feels weird to say that but here I am.

The thing is this youtube exposure makes me want to continue making content, which is what I do, but it also makes me question the future. I have a job now, what should I be focusing on in life ..? What is the end *purpose* of my job and of what I do on youtube ? Money cannot be the only answer, even though it is an important factor.

I want to make the *HOLY CORE* a reference project when it come to lerning, that is my ambition. I named it "HOLY CORE" as a christian pun (translated as "*coeur saint*" in French) as the faith was something that helped me carry on with this project, knowing that despite its simplicity compared to big industrial project, it would shine because it brings something more to people that just a cool open source project that *nobody* really understand, an opportunity to learn and better themselves.

Anyway, more is comming on the *HOLY CORE*, this is for sure, but open sourcing the tutorials themselves is becomming less of an option as I advance in more complex topics... For multiple reasons but I think I will naturally come to talk about it later in my "career".

### Some very personnal thoughs on the project

Once I got the core to blink LEDs, I remembered how much fun it was. This project took a lot of work to come to this simple milestone but I remember how enjoyable the learning process was :

- No chat GPT to help you
- Pure low level learning
- Felt like doing clockwork or tinkering on a complex machine.

I kinda wanted to share that with people. I don't know *where* this will go. I do have my desires: of course I would like the HOLY CORE project to blow up and become *THE* reference Risc-V digital design learning path for **self learniners** but nothing ever goes as what you would like them to go so the only thing I can do is make tis project as nice as I can make it.

But I also have to think, **again**, about the purpose of all of this. I like to think that, when I'll be extremely sick and about to die, I will think about what I could've done differently and I gotta admit that "*programming more FPGAs*" or "*working harder on shareholder value maximisation*" will not be on my bucket list. I do like this as a job, but I do not aspire to this spirtually, which is why the *HOLY CORE* is a special project for me, I would like it to be a lasting project, helping out a community to build and understand a little more the tech that surronds them, something more real than just a cold chip on an FPGA, that serve no "real" purpose (for me, a real purpose is adding beauty and meaning to the world).

ASometime, finding a purpose is hard, but I'm sure we'll figure it out !

Thank you for reading to this point. You can write a comment below if you have any question.

*Godspeed*

-BRH