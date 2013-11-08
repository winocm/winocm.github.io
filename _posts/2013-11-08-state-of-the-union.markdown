---
layout: post
title:  "State of the Union"
date:   2013-11-08 08:00:00
categories: projects research
---

(This part was copied verbatim from the previous post.)

As you are all probably aware of (I hope), I maintain an open-source port of the XNU (iPhone OS/Mac OS X) kernel
to ARMv7-A (soon AArch64 and ARMv6/v5!) platforms. The kernel is very bare in its implementation and needs a lot of
work. Work seems to also not be put into the proper areas, so I made this article to rectify that issue. Simple enough,
I hope.

The code for my fork of the XNU kernel is located on my [GitHub repository](https://github.com/winocm/xnu). The entire thing is
a miasma of licensing (between APSL and BSD). 

# Current State

The kernel can now run simple user binaries such as:

* bash
* ls
* grep
* uname
* init

`launchd` does not work entirely properly, that issue must be fixed. In addition, UNIX signal handling 
must also be implemented for job control to work. Sending a `^C` to bash causes the system to panic. :)

{% highlight bash %}
-sh-4.0# ^C
panic(cpu 0 caller 0x8047dce3): "sendsig"@/SourceCache/xnu/xnu-2050.22.13/bsd/dev/arm/unix_signals.c:107
Debugger message: panic
OS version: Not yet set
Kernel version: Darwin Kernel Version 12.5.0: Fri Nov  8 08:00:39 CST 2013; rms:xnu-2050.22.13/BUILD/obj//DEBUG_ARM_ARMPBA8
iBoot version: GenericBooter-100.3.1
secure boot?: NO
Paniclog version: 1
Kernel slide:     0x0000000000000000
Kernel text base: 0x80001000
Epoch Time:        sec       usec
  Boot    : 0x00000000 0x00000000
  Sleep   : 0x00000000 0x00000000
  Wake    : 0x00000000 0x00000000
  Calendar: 0x000d7a17 0x00000000
{% endhighlight %}

# Platforms

The previous platforms were maintained, but support was also added for a new board, Texas Instruments
AM335x. This adds support for BeagleBone and BeagleBone Black.

So, the currently supported platforms are:

* ARM RealView Emulation Baseboard with Cortex-A8 CoreTile
* ARM RealView Platform Baseboard for Cortex-A8
* Samsung S5L8930X (Apple A4 -- iPhone 4 GSM/CDMA/RevA, iPad 1, iPod touch 4G, Apple TV 2G)
* Samsung S5L8922X (iPod touch 3G)
* Samsung S5L8920X (iPhone 3GS)
* Texas Instruments OMAP3530/3730 (BeagleBoard xM)
* Texas Instruments AM335x (BeagleBone/BeagleBone Black)

These platforms should be able to all run a single user shell mostly properly. There are still some
various issues with memory management that need to be fixed.

Kernel extensions do work at this time. 

# Future Plans

After full support for UNIX signals is implemented, along with fixing issues with latency, user input
and `launchd`, this port will be considered fully complete.

Full kernel ASLR still also needs to be implemented, slides can be done with 1MB granularity or page
granularity.

That's pretty much it. It's definitely cool enough to see a bash instance on this operating system. :)

# Conclusion

The kernel now can run simple user binaries, such as `bash`, `ls` or such. It reaches a single user
shell and can execute things properly on all platforms.

<figure>
<img src="/images/xnushell.jpg" alt="XNU booting on OMAP3">
</figure>

Hope you do have fun. There's an IRC channel called `##darwin-on-arm` on Freenode if you'd like to join/help. Feel free to ask questions.