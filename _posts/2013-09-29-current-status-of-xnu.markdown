---
layout: post
title:  "Current status of XNU/ARM and beyond"
date:   2013-09-29 20:00:00
categories: xnu projects research
---

As you are all probably aware of (I hope), I maintain an open-source port of the XNU (iPhone OS/Mac OS X) kernel
to ARMv7-A (soon AArch64 and ARMv6/v5!) platforms. The kernel is very bare in its implementation and needs a lot of
work. Work seems to also not be put into the proper areas, so I made this article to rectify that issue. Simple enough,
I hope.

The code for my fork of the XNU kernel is located on my [GitHub repository](https://github.com/winocm/xnu). The entire thing is
a miasma of licensing (between APSL and BSD). 

# Current State

The kernel does boot past IOKit initialization and can start a usermode process. Validation of a simple binary that calls `write()` 
using standard UNIX syscalls did succeed. It also can boot from a multitude of booters including Apple's own iBoot (albeit a very
old revision of it). It also can additionally start `launchd` and actually execute `dyld` proper now. `launchd` now dies after
forking, but hey, it's a start.

# Platforms

This fork of XNU supports the following hardware configurations (as of right now):

* ARM RealView Emulation Baseboard with Cortex-A8 CoreTile
* ARM RealView Platform Baseboard for Cortex-A8
* Samsung S5L8930X (Apple A4 -- iPhone 4 GSM/CDMA/RevA, iPad 1, iPod touch 4G, Apple TV 2G)
* Samsung S5L8922X (iPod touch 3G)
* Samsung S5L8920X (iPhone 3GS)
* Texas Instruments OMAP3530/3730 (BeagleBoard xM)

It's not a lot of hardware, but they all can boot to mounting root (albeit, some platforms have some more breakage than others). For example,
the timer implementation in the S5L devices is completely wrong (thanks to the lack of documentation), event timer deadline resynchronization
is only active in ARM RealView, and so on.

# Future Plans

The current target for this kernel is to allow it to run `launchd` proper and spawn a shell. After the system can achieve a single/multiuser
boot, the kernel is considered feature complete.

However, I do plan for certain things to be done, namely:

* Rewriting the 'pmap' subsystem for ARMv7-A.
* Adding support for ARMv6K and ARMv5T based platforms.
* Adding support for AArch64 systems.
* Refactoring the SoC dispatch architecture into something far more modular.
* Kernel extension support.

# Conclusion

This kernel does currently reach a very late portion of system startup, and it appears to work somewhat on various hardware combinations. With
that, I leave you with this picture:

<figure>
<img src="/images/xnun18.png" alt="XNU booting (somewhat) on an iPod touch 3G">
</figure>

Hope you do have fun. There's an IRC channel called `##darwin-on-arm` on Freenode if you'd like to join/help. Feel free to ask questions.

