---
title: "N64 TAS (Tool-Assisted) Speedrun Research"
date: 2021-02-21T23:28:30-04:00
header:
  - image: /assets/images/n64/21-02-21 23-31-33 7169-min.jpg
  - caption: "My N64 with test points."
categories:
  - Blog
tags:
  - Projects
  - Consoles
---

For several years I've watched the speedrunning community participate in a bi-yearly event called [Games Done Quick](https://gamesdonequick.com/). This showcase/telethon works to collect as many donations as possible during a week-long videogame speedrunning marathon. One segment that caught my eye was the [Tool-Assisted Speedruns (TAS)](http://tasvideos.org/). These speedruns focus on "scripting" a series of button presses in an emulated environment (an emulator) to beat (and sometimes break) video games. A further extension of this subset of the speedrunning community involves replaying" pre-programmed scripts (thus the "tool" portion) on real hardware. I've been putting off reaching out to the community for a few years, but I finally decided to reach out this year. What follows are my observations, learnings, and other notes from working to improve TAS reliability on the N64 console.

## 03/10/2021

Over the weekend I reached out to several N64 modding communities in hopes that someone had previously attempted to get JTAG working on a console. One of the members pointed out that the NEC processor pinout was slightly customized in the final (production) N64 CPU design, which led me to discover that JTCK and INT1 pins were swapped compared to the commercial VR4300 CPU pinout. After lifting the necessary pin and moving over the jumper wire, I fired up OpenOCD and probed the JTAG port. 

Once I tweaked the OpenOCD configuration a bit, I successfully read the expected `ircapture` response from the JTAG port! Here's what OpenOCD and GDB look like after successfully connecting to the N64.

<figure>
  <a href="/assets/images/n64/openocd_gdb_ubuntu.png"><img src="/assets/images/n64/openocd_gdb_ubuntu.png" alt="OpenOCD Successfully Connecting to the N64 CPU!"></a>
</figure>

My OpenOCD configuration is hosted [here](https://github.com/juchong/openocd_n64).


While playing with the JTAG port and configuration a bit, I realized that the N64 would sometimes refuse to start even though seemingly nothing had changed. After a bit of probing with my oscilloscope, I discovered that the JTAG lines would start floating when my debugger released (high-z) the pins. I quickly fixed this issue by adding 10k pull-down resistors to each of the JTAG pins. 


I've put together a diagram that should help anyone set up JTAG on their retail N64. It's worth pointing out that once I selected the correct JTCK pin, I discovered that the OpAmp was no longer necessary! 

<figure>
  <a href="/assets/images/n64/N64_JTAG_JuanC_RevA.JPG"><img src="/assets/images/n64/N64_JTAG_JuanC_RevA.JPG" alt="Retail N64 JTAG Interface Schematic"></a>
</figure>

## 03/07/2021

I decided to broaden my "research" scope after speaking with a few folks on the TASbot Discord [link](https://discord.com/invite/CwnDTug). Instead of focusing on finding a hardware indicator to help identify when a new video frame is produced, I'm working to improve the hardware debugging capabilities of the N64 community. A few members mentioned that getting the JTAG port on the custom NEC CPU to function would be valuable, so I focused on getting that working this week. 

My first move was to figure out whether the JTAG port worked out of the box. Last week I added test points and lifted pins as necessary to allow the JTAG pins to be driven as necessary. I then used the resources listed on Wrongbaud's JTAG hacking guide [link](https://wrongbaud.github.io/posts/jtag-hdd/) to try and get the N64's CPU to respond to JTAG commands. I discovered that the CPU appears to be halting once the JTAG probing process starts, indicating that the pins appear not to be disabled! After carefully probing each of the pins with an oscilloscope, I discovered that the TDO pin appears not to be outputting any meaningful data, even when valid JTAG commands are transmitted. 

After staring at the N64 schematic [link](https://console5.com/techwiki/images/a/a2/N64_NUS-CPU-03.pdf) for a bit, I realized that the ColdReset pin on the CPU (Pin 110) is also connected to the Reality Coprocessor (RCP), presumably to allow the coprocessor to trigger a soft reset of the CPU. I theorized that the RCP might be using that pin to detect when the CPU is reset (think of it as if it were acting as a watchdog), so I decided to sever the connection after the N64 booted using a simple switch and a pull-up resistor. 

<figure>
  <a href="/assets/images/n64/21-03-07 00-53-16 7292.jpg"><img src="/assets/images/n64/21-03-07 00-53-16 7292.jpg" alt="Switch to Disable ColdReset"></a>
</figure>


This test produced interesting results since it proved that the RCP was happy to continue operating (drawing the same frame) even though the CPU was no longer sending it any data. 

Next, I tried watching TDO on my oscilloscope to see whether the pin ever showed any meaningful signal. My concern, at this point, was that Nintendo might have chosen to permanently disable the pin in their production (consumer) units. The pin appeared never to be toggled during regular operation. However, I was able to observe *something* happening on the pin during boot. 

<figure>
  <a href="/assets/images/n64/image_pre_opamp.png"><img src="/assets/images/n64/image_pre_opamp.png" alt="TDO Signal Before Buffering"></a>
</figure>

The capture shown above prompted me to measure the resistance to VDD and GND on several pins I knew to be operating as "outputs." After doing so, I discovered that the TDO pin had a similar resistance to VDD as other output pins (~52kohm), but it had a higher resistance to GND (1.92Mohm vs. 58kohm). While these measurements don't mean much due to how typical CMOS output drivers are designed, they were enough to prove a few things:

1. The pin was bonded out (a bond wire was placed between the ASIC and the package pin) at the factory.
2. This pin was designed slightly differently than the rest of the output pins. 

After taking a few days to reflect on what I had learned, I still couldn't shake the idea that the pin *was doing something at boot.* That's when I had this idea: "what if the output transistors were purposely handicapped to discourage people from probing them?"

Using the image above as a starting point, I quickly calculated that a simple non-inverting OpAmp circuit with a gain of 15 should be enough to boost the weakened signal to something usable by external hardware. I set to work dead-bugging the circuit, and after some soldering, ended up with the circuit shown in the images below. 

<figure>
  <a href="/assets/images/n64/21-03-07 00-53-08 7291.jpg"><img src="/assets/images/n64/21-03-07 00-53-08 7291.jpg" alt="Dead-Bugged OpAmp Circuit"></a>
</figure>

Once I verified that things looked good with a test signal, I turned on the N64 and discovered this:

<figure>
  <a href="/assets/images/n64/image_post_opamp.png"><img src="/assets/images/n64/image_post_opamp.png" alt="TDO Signal After Buffering"></a>
</figure>

It looks like I'm now able to discern usable signals from the handicapped TDO port! Unfortunately, I couldn't get the JTAG port, but regardless, this is excellent progress! 

Another member of the TASbot community pointed me to a [website](https://ultra64.ca/gallery/kyoto-microcomputer-co-ltd-k%c2%b5c/) listing several N64 developer consoles and hardware, so my next move is to try implementing some of those modifications on my N64.

## 02/22/2021

Today was spent trying to wrap my head around the many aspects, headaches, and limitations of replaying TASs on the N64 console. N64 console emulation has not matured to a point where the emulator reliably mimics the Reality Signal Processor (RSP), leading to many "de-syncs." Very few games can be replayed reliably (Super Mario 64, MarioKart 64) on actual N64 hardware. 
I spent most of today chasing down "red herrings" and probing around the digital video signals, exposed the JTAG lines onto pins, and probed a few unused pins available on the N64 CPU. Additional information on the digital signals and protocol used for generating analog RGB signals can be found [here](http://members.optusnet.com.au/eviltim/n64rgb/n64rgb.html). Information about the N64 CPU can be found [here](http://en64.shoutwiki.com/wiki/N64_CPU). The table below seemed very interesting when paired with the N64 schematic (shown below). I specifically wanted to know whether the "unused" signals worked on the customized NEC CPU. 

<figure>
  <a href="/assets/images/n64/nes_cpu_table.png"><img src="/assets/images/n64/nes_cpu_table.png" alt="N64 CPU Table"></a>
</figure>

<figure>
  <a href="/assets/images/n64/unused_n64_signals.PNG"><img src="/assets/images/n64/unused_n64_signals.PNG" alt="N64 CPU Pins"></a>
</figure>

The !PMaster signal on the CPU appears to correlate nicely with when the controller inputs are polled. It's interesting to point out that when the N64 is taxed, there's a possibility that the RSP and RDP (Reality Display Processor) may "drop frames." When this behavior occurs, player inputs are ignored, and the RDP re-transmits the last video signal instead of a newly rendered frame. I believe that the key to managing the de-sync events on the N64 is to monitor for this behavior and use that information to determine whether joystick inputs should be repeated. 

This image shows the controller and CPU signals recorded in Donkey Kong 64 while standing still. 

<figure>
  <a href="/assets/images/n64/sitting_idle.png"><img src="/assets/images/n64/sitting_idle.png" alt="DK64 Sitting Idle"></a>
</figure>

This image shows how much the controller polling time changes depending on CPU load.

<figure>
  <a href="/assets/images/n64/messy_timing.png"><img src="/assets/images/n64/messy_timing.png" alt="DK64 Messy Timing"></a>
</figure>

Finally, this image shows the same signals recorded in Donkey Kong 64 while rotating the player camera. While moving around, I noticed that the N64 video "skipped" (dropped frames). This logic analyzer capture also clearly shows that the N64 RSP requested the controller inputs during a window where the CPU was not ready. Maybe I can use this information to determine whether the next frame will be dropped? Regardless, this image shows how unstable the controller data acquisition process becomes when the CPU/RDP becomes taxed. 

<figure>
  <a href="/assets/images/n64/cpu_busy_snip.png"><img src="/assets/images/n64/cpu_busy_snip.png" alt="N64 CPU When Busy"></a>
</figure>

Here are a few pictures of the N64's current state now that I've added test points. 

<figure>
  <a href="/assets/images/n64/21-02-21 23-31-33 7169-min.jpg"><img src="/assets/images/n64/21-02-21 23-31-33 7169-min.jpg" alt="N64 Motherboard"></a>
</figure>

<figure>
  <a href="/assets/images/n64/21-02-21 23-31-39 7170-min.jpg"><img src="/assets/images/n64/21-02-21 23-31-39 7170-min.jpg" alt="N64 Video Processing Test Points"></a>
</figure>
<figure>
  <a href="/assets/images/n64/21-02-21 23-31-46 7171-min.jpg"><img src="/assets/images/n64/21-02-21 23-31-46 7171-min.jpg" alt="N64 JTAG and Other Test Points"></a>
</figure>

