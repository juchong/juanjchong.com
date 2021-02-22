---
title: "N64 TAS (Tool-Assisted) Speedrun Research"
date: 2021-02-21T23:28:30-04:00
header:
  - image: /assets/images/laser-cutter-tweaks-review/IMG_4744.jpg
  - caption: "My N64 with test points."
categories:
  - Blog
tags:
  - Projects
  - Consoles
---

For several years I've watched the speedrunning community participate in a bi-yearly event called [Games Done Quick](https://gamesdonequick.com/). This showcase/telethon works to collect as many donations as possible during a week-long videogame speedrunning marathon. One segment that caught my eye was the [Tool-Assisted Speedruns (TAS)](http://tasvideos.org/). These speedruns focus on "scripting" a series of button presses in an emulated environment (an emulator) to beat (and sometimes break) video games. A further extension of this subset of the speedrunning community involves replaying" pre-programmed scripts (thus the "tool" portion) on real hardware. I've been putting off reaching out to the community for a few years, but I finally decided to reach out this year. What follows are my observations, learnings, and other notes from working to improve TAS reliability on the N64 console.

## 02/22/2021

Today was spent trying to wrap my head around the many aspects, headaches, and limitations of replaying TASs on the N64 console. N64 console emulation has not matured to a point where the emulator reliably mimics the Reality Signal Processor (RSP), leading to many "de-syncs." Very few games can be replayed reliably (Super Mario 64, MarioKart 64) on actual N64 hardware. 
I spent most of today chasing down "red herrings" and probing around the digital video signals, exposed the JTAG lines onto pins, and probed a few unused pins available on the N64 CPU. Additional information on the digital signals and protocol used for generating analog RGB signals can be found [here](http://members.optusnet.com.au/eviltim/n64rgb/n64rgb.html). Information about the N64 CPU can be found [here](http://en64.shoutwiki.com/wiki/N64_CPU). The table below seemed very interesting when paired with the N64 schematic (shown below). I specifically wanted to know whether the "unused" signals worked on the customized NEC CPU. 

![N64 CPU Table](/assets/images/n64/nes_cpu_table.png)

![N64 Unused Signals](/assets/images/n64/unused_n64_signals.PNG)

The !PMaster signal on the CPU appears to correlate nicely with when the controller inputs are polled. It's interesting to point out that when the N64 is taxed, there's a possibility that the RSP and RDP (Reality Display Processor) may "drop frames." When this behavior occurs, player inputs are ignored, and the RDP re-transmits the last video signal instead of a newly rendered frame. I believe that the key to managing the de-sync events on the N64 is to monitor for this behavior and use that information to determine whether joystick inputs should be repeated. 

This image shows the controller and CPU signals recorded in Donkey Kong 64 while standing still. 

![DK64 Sitting Idle](/assets/images/n64/sitting_idle.png)

This image shows how much the controller polling time changes depending on CPU load.

![DK64 Messy Timing](/assets/images/n64/messy_timing.png)

Finally, this image shows the same signals recorded in Donkey Kong 64 while rotating the player camera. While moving around, I noticed that the N64 video "skipped" (dropped frames). This logic analyzer capture also clearly shows that the N64 RSP requested the controller inputs during a window where the CPU was not ready. Maybe I can use this information to determine whether the next frame will be dropped? Regardless, this image shows how unstable the controller data acquisition process becomes when the CPU/RDP becomes taxed. 

![DK64 Walking Around](/assets/images/n64/cpu_busy_snip.png)

Here are a few pictures of the N64's current state now that I've added test points. 

![N64 Motherboard With Test Points 1](/assets/images/n64/21-02-21 23-31-33 7169-min.jpg)

![N64 Motherboard With Test Points 2](/assets/images/n64/21-02-21 23-31-39 7170-min.jpg)

![N64 Motherboard With Test Points 3](/assets/images/n64/21-02-21 23-31-46 7171-min.jpg)