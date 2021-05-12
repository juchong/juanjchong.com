---
title: "Making Ubuntu Usable on a Dell Latitude 7280"
date: 2021-05-12T21:34:31-04:00
header:
  - image: 
  - caption: ""
categories:
  - Blog
tags:
  - Projects
  - Linux
  - Guides
---

I've been using a Dell Latitude 7280 as my x86-based daily-driver for about a year. It's an excellent laptop for software development and hardware debugging in Windows, but requires some special attention when setting up Linux. 

Even when installing Ubuntu 20.04 LTS, the laptop struggles with lock-ups when it enters stand-by mode or the screen turns off due to inactivity. 

After a bit of debugging and research, I discovered that the root-cause of these issues is the lack of proprietary drivers. The stock Ubuntu image does **not** include any specialized Intel drivers, meaning that the kernel has no idea how to handle the chipset's low power modes. The easiest way to install these drivers is to add the Open Graphics Drivers ppa to your Ubuntu installation using the following commands:

```
sudo add-apt-repository ppa:oibaf/graphics-drivers
sudo apt update
sudo apt dist-upgrade
```

These commands should take effect immediately, but I recommend rebooting your PC. 

I also upgraded the WiFi adapter in my laptop by replacing the stock Intel 8260NGW adapter with a Killer Wireless N1535. Even though the adapter will seemingly work out-of-the-box, the laptop will refuse to successfully exit stand-by. The driver baked into the Linux kernel is designed to optionally load a precompiled firmware into the WiFi adapter's RAM when initialized. The most up-to-date firmware available for the adapter can be installed by issuing the following commands:

```
sudo apt update
sudo apt install linux-firmware
```

Finally, the driver can be re-loaded by issuing the following command:

```
sudo modprobe -r ath10k_pci && sudo modprobe ath10k_pci
```

The updated firmware should automatically load every time the kernel calls the N1535 driver. 