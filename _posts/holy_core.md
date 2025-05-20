---
title: "HOLY CORE : Debrief"
date: 2025-05-20 15:00:00 +0800
categories: [Projects]
tags: [fpga, cpu]
pin: false
math: false
mermaid: false
image:
  path: https://raw.githubusercontent.com/0BAB1/HOLY_CORE_COURSE/refs/heads/master/banner.png
  alt: Cocotb waveforms example on my HOLY CORE course
---

> This post is not technical but rather a way for me to share thoughts on the HOLY CORE project. For context, HOLY CORE is an *open source* RV32I compliant CPU that runs on FPGA and the entire development process is document for everyone to learn through an easy to understand course.

## HOLY CORE : Presentation

For those of you who do not know holy core, here is the [youtube video](https://www.youtube.com/watch?v=ix8vlIM7Iv8) I made on the project :
{% include embed/youtube.html id='ix8vlIM7Iv8' %}

## HOLY CORE : *Exclusive* Debrief and Future Work.

### Coming back into the project

Recently, I've been doing a lot a finance stuff for a big project that I decided to put on hold. I'll talk about this in a future post. Because I want to keep my Youtube channel updated with good projects I decided to hop back on the Holy Core after a couple of month of not touching it.

Doing so, I though to myself :

> What the hell went through my mind for me to make all of this ?

The project was not working anymore, I had to re-do he whole setup and I put myself in the shoes of those who just want to get the core running to tinker around : **ITS SIMPLY NOT DOABLE**. So I decided to add some "quickstart" guides to make things easier for newcomers who don't want to go through the whole course.

### Why I came back

I came back to Holy Core because I thrive on side projects (they got me a job so I figured it's also a pretty important thing to keep doing !).

**But I don't thrive on any type of project**, I like hard and "*dumb*" projects. They type of projects that reinvents the wheel just for the fun of making things by myself and learning stuff even though it sound very not doable.

Now that I built a litteral CPU, I though "why not use it ? It would also make some great content for the youtube channel !" and so I decided to order components online and while they ship, I have a week or so to add a few things :

- Raw AXI LITE for MMIO
- Some software basics like :
  - Utilities for the cache nightmare
  - try to make UART work
  - ...

More on what is next in the following sections, but first some thoughs on how the projects turned out...

### Some personnal thoughs on the project

Once I got the core to blink LEDs, I remembered how much fun it was. This project took a lot of work to come to an end but I remember how enjoyable the learning process was :

- No chat GPT to help you (its not python or js so most of the bugs you have to solve yourself)
- Pure low level learning
- Total control (except on the damn toolchain but that's another subject)

More on the personnal side, my studies are not really 100% in this very low level field and I have a particular college cursus where I do 50% studying and 50% work in a genuine business that manufactures auto parts, which has ABSOLUTELY NOTHING to do with FPGAs or even electronics.

So working on this was a bittersweet feeling. When I worked in the open space and had time to spare (as well as nobody looking ;) ), I did as much research as I could, I took notes, I red documentation etc... And when I was back at my place, I immediatly got to work to implement what I researched until I went to bed.

I described this particular situation as "bittersweet" because this project was nothing. Building a low performance softcore is useless, not in sync with my **current** (bad) career choices and I worked on this instead of doing other and maybe better things like touching grass.

Yet I pushed through, got it done, bundled it all together in a nice repo and video and finally posted it on youtube. And it got more attention than I expected, which made me very happy, but I had to move on with other projects.

