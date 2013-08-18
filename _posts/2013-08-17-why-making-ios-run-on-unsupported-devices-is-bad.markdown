---
layout: post
title:  "Why making iOS run on unsupported devices is bad"
date:   2013-08-17 19:38:00
categories: rants
---

iOS, or iPhone OS, is designed to run only with the target hardware its designed for in mind. This can be seen at
many levels, from iBoot, all the way to the kernel and many userland components. It's not too good of an idea
to mess around with the internals of iOS at all (unless you have source code for everything, then feel free to
do whatever you want).

The XNU kernel is used widely on many Apple devices, ranging from the iMac, to the iPhone. It is a kernel based on Mach 
4.3 but also uses a lot of BSD code. Contrary to Mach's original design, XNU is not a microkernel, but rather a very 
large monolithic one. The userland consists of many other components such as `launchd`, `dyld`, `libSystem` and so on.
Of course, the actual UI stuff such as `UIKit` and `SpringBoard` is still there too.

This post is mainly concerned with low-level components of iOS, so be warned. This article is also relatively technical,
but that goes without saying.

What this post is not
=====================

I'm not trying to be discouraging in any form or fashion, but all of the problems presented are very real issues.
You *will* encounter these issues at some point if you use the iOS 7 bootchain or whatever on an unsupported 
board configuration. That isn't to say this can't be done; if you can do it, then go do it! It'll just be
annoying. 

Fun with iBoot
==============

iBoot is the choice of bootloader on embedded platforms by Apple. It's a very versatile bootloader, featuring threading,
USB DFU emulation, USB serial terminal, and so much more. Of course, much of this has now been stripped out of RELEASE
iBoot configurations. One can assume it's still all there for DEVELOPMENT or DEBUG style iBoots. (Commands such as
`tftp`, `iic`, `mwh`, and so on were removed from RELEASE iBoots after iOS 3.1/iBoot-636.26 and later.)

Of course, for each board configuration, iBoot has to initialize hardware. This is done to bring up core peripherals such
as NAND, H2FMI, DisplayPipe and so on. Each device has to have its own configuration tables for the hardware to initialize
properly. RELEASE style iBoots strip a lot of the information out and leave what's needed for only one device to boot properly.
Put simply, if you remove the board ID check, iBoot may work, but you probably won't have working peripherals. One of these may
include the NAND interface. Don't ask me where your serial number and such went on iBoot-1940 on N81 if you do that.

For devices that are not supported, you will get a message very much like this over serial UART. iBoot will then proceed to hang
and you will have to reset your ARM board.

{% highlight bash %}
cdma_init()
spi_init()
SysCfg: version 0x00020001 with 10 entries using 224 of 8192 bytes
BDEV: protecting 0x2000-0x8000
image 0x5ff32900: bdev 0x5ff32580 type illb offset 0x8000 len 0x10178
image 0x5ff32980: bdev 0x5ff32580 type ibot offset 0x189c0 len 0x29178
image 0x5ff32a00: bdev 0x5ff32580 type dtre offset 0x42380 len 0xeaf8
image 0x5ff32a80: bdev 0x5ff32580 type logo offset 0x516c0 len 0x1538
image 0x5ff32b00: bdev 0x5ff32580 type recm offset 0x53440 len 0x10078
image 0x5ff32b80: bdev 0x5ff32580 type bat0 offset 0x63d00 len 0xb178
image 0x5ff32c00: bdev 0x5ff32580 type bat1 offset 0x6f6c0 len 0x3cb8
image 0x5ff32c80: bdev 0x5ff32580 type glyC offset 0x73bc0 len 0xff8
image 0x5ff32d00: bdev 0x5ff32580 type glyP offset 0x75400 len 0xbb8
image 0x5ff32d80: bdev 0x5ff32580 type chg0 offset 0x76800 len 0xbb8
image 0x5ff32e00: bdev 0x5ff32580 type chg1 offset 0x77c00 len 0x3638
image 0x5ff32e80: bdev 0x5ff32580 type batF offset 0x7ba80 len 0x3c7f8

**************************************
*                                    *
*     This unit is not supported     *
*                                    *
**************************************
{% endhighlight %}

Additionally, iBoot will perform signature validation on images. This means that only trusted Apple
images may run on the device proper. To get around this, you would need a device with different security fusings, or
you would need to run a patched iBoot. It is much easier to perform the latter, however, your mileage may vary
depending on the device. For example, it is not feasible to have an N81 constantly in DFU because LLB and iBoot 
are invalid all the time. 

As of iOS 5, Apple moved away from checking certificates and personalization blobs, but relied on various SHA-sum
chunks in the APTicket. Images don't need signatures anymore, they just need valid chunks in the apticket, however,
the apticket must be signed properly for iBoot to load it properly.

Device Tree
===========

All systems that boot Darwin have a device-tree of some sort. The device tree contains information about the hardware,
where peripherals are, their registers, interrupt controllers, processor descriptions and so on. This allows the kernel
to know where various things in the system are.

On iOS, the device tree would look something like this:
{% highlight bash %}
[virtual bool ARMPlatformExpert::start(IOService *)]: Dumping current service tree
Count 8
    iPhone1,1 <class IOPlatformExpertDevice, busy 4>
      ARMPlatformExpert <class ARMPlatformExpert, busy 5>
        IOPMrootDomain <class IOPMrootDomain, busy 1>
          IORootParent <class IORootParent, busy 0>
        cpu0@0 <class IOPlatformDevice, busy 1>
        cpus <class IOPlatformDevice, busy 1>
        arm-io <class IOPlatformDevice, busy 1>
      IOResources <class IOResources, busy 1>
[virtual bool ARMPlatformExpert::start(IOService *)]: Registered device with IOKit
{%endhighlight%}

Now of course, this is from my version of the XNU kernel using my own device-tree and such, but it would be incredibly
similar to that of iOS. The ARM platform expert is attached to the root node, which calls itself "iPhone1,1". The 
name of the root node comes from the `compatible` member in the root of the device-tree. SpringBoard and many other programs
use the name of the IOPlatformExpertDevice/rootnode to perform many tasks such as identifying the device model and type.

iOS 7 additionally puts many of the device capabilities into `/product` in the device-tree. These members include things such
as 3D-maps usage, Siri, FaceTime over 3G and so on. This was probably done to discourage editing of SpringBoard capability
property lists.

Kernelcache
===========

The kernelcache is the heart of the system, it performs everything the OS needs to run. iOS also implements many
proprietary drivers for platform management. The purpose of these drivers are to connect a specific board configuration's 
hardware to the software in a common way. The actual thing that bridges hardware and user-mode software is called IOKit.
You can search Apple's developer documentation for more information on IOKit and how to use it.

Now, the reason why you can't just copy over the kernel from another device and expect it to work is that there are 
very minute hardware differences between hardware variants. Much of the same can also be said for iBoot. One big example
is how Wi-Fi is implemented slightly differently in the N92 and N90 kernels. Sure you can use the N92 kernel on an N90, but
tons of things, such as Wi-Fi, will have broken.

iOS 7 deprecated N81 and the N81 hardware is very different compared to the N90/N92. For example, various peripherals use
different identifiers and have different chip ID versions. You can't just simply fix everything without having source code
to back it all. You will eventually hit a brick wall. 

You'll enjoy using KDB over serial for hours on end. (I wish there was a user accessible JTAG port...)

Conclusion
==========

Sure, you can run unsupported iBoot versions on various board configurations, and sure you can run the kernel on unsupported
hardware, but is it all worth it? Lots of things will have broken in the process and you will feel very happy after debugging
iBoot for hours on end wondering "why does it hang here" or debugging the kernel for hours on end wondering "why does it
panic here?".

Is it worth your time?
