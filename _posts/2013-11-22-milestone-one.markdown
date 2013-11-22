---
layout: post
title:  "Milestone One"
date:   2013-11-22 08:00:00
categories: projects research
---

Reecently, I achieved one of the core milestones of my personal project, porting the Darwin
kernel to the ARM architecture. This specified milestone was booting to a multiuser system.

Darwin is the core operating system that lies under both Mac OS X and iPhone OS. It is the
true core foundation that bridges the kernel to the actual UI above. (SpringBoard/loginwindow/etc).

With the help of @plus_chan and his Nokia N900, I present to you, Darwin/ARM running on
a Nokia N900. (Though, this is @stroughtonsmith's N900. Whatever.)

<figure>
<img src="/images/xnu-n900.png" alt="XNU booting on OMAP3430_RX51">
</figure>

Know that it doesn't just run on that, it also runs on:

* ARM RealView Emulation Baseboard (ARMPBA8_ALT)
* ARM RealView Platform Baseboard for Cortex-A8 (ARMPBA8)
* Texas Instruments OMAP3530 (BeagleBoard/BeagleBoard xM) (OMAP3530)
* Texas Instruments OMAP3430 (Nokia N900) (OMAP3430_RX51)
* Texas Instruments AM335x (BeagleBone/BeagleBone Black) (OMAP335X)
* Apple A4 (iPhone 4, iPod touch 4G, iPhone 4 CDMA, iPhone 4 GSM revA, iPad 1, Apple TV 2) (S5L8930X)
* iPhone 3GS (S5L8920X)
* iPod touch 3G (S5L8922X)

Board ports are only limited to the ARM hardware you need to port the kernel to. :)

(The system root filesystem is based on iPhone OS 4.3.5. It runs everything flawlessly for the
most part. However, there are a ton of kernel bugs to fix, including power management.)

Source is available on [GitHub](http://github.com/darwin-on-arm). 

Hope you do have fun. There's an IRC channel called `##darwin-on-arm` on Freenode if you'd like to join/help. Feel free to ask questions.

(And no, I do not plan for graphical UI support at this time, only the Core OS. After all, that's
what's truly important, right? :P)