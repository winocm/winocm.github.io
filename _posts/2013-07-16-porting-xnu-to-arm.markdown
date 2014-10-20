---
layout: post
title:  "Porting the XNU (Mac OS X/iOS) kernel to ARM"
date:   2013-07-16 20:07:00
categories: xnu projects
---

The XNU kernel is used widely on many Apple devices, ranging from the iMac, to the iPhone. It is a kernel based on Mach 
3.0 but also uses a lot of BSD code. Contrary to Mach's original design, XNU is not a microkernel, but rather a very 
large monolithic one. This article isn't really for debate against which kernel design is better, that's for somewhere 
else.

Apple does maintain a version of XNU for ARM devices, but this version is proprietary, and was never released on
[Apple's open source software portal](http://opensource.apple.com). Only the i386/x86_64 version and earlier, the 
PowerPC version, is/was open source. (If you care about PowerPC, the last version of it you'll find is 
[xnu-1504.9.37](http://opensource.apple.com/source/xnu/xnu-1504.9.37), or Darwin 10.7/Snow Leopard).

Bootloader Fun
==============

On ARM/embedded platforms, Apple uses a bootloader called `iBoot`. This bootloader initializes the hardware
and brings it up to a usable state. Then, it can load a `kernelcache` over a USB connection or over TFTP. A 
`kernelcache` is a LZSS compressed container that has all of the kernel extensions needed for boot built-in. Since I am 
working on a BeagleBoard xM, I obviously did not have iBoot on my platform.

To solve this problem, I made two bootloaders. One that acts as a shim between u-boot (the native Linux bootloader), and 
the XNU kernel, and an additional one that uses UEFI to bootstrap the kernel directly. Both contain code to flatten the 
plist (yes, real Property Lists) based device-tree and to add additional memory nodes if necessary. The Linux 
shim-bootloader also will pass along any initrd or commandline arguments to the XNU kernel if necessary, which makes 
development a lot easier. Moving a micro-SD card over and over from machine to machine gets very tiring.

I boot my kernel/bootloader combination over TFTP using the following configuration in u-boot. The device-tree and 
kernel image are attached to the end of the shim bootloader, sort of analogous to a `dtbImage` in the world of Linux.

{% highlight bash %}
setenv ipaddr 192.168.1.200
setenv serverip 192.168.1.15
setenv usbethaddr de:ad:be:ef:c0:fe
usb start
tftpboot 0x84000000 /mach_kernel
bootm 0x84000000
{% endhighlight %}

I also have a configuration very much like this for booting UEFI, but I placed that under the `user.txt` file for 
booting.

Core Bringup
============

Initially, I had to write a lot of the platform code. This included things such as spinlocks, thread setup, exception
handlers, physical memory mapper and so on. For functions I did not implement, I simply stubbed them out by using the C 
preprocessor and GNU assembler.

{% highlight c %}
#define UNIMPLEMENTED_STUB(Function)            \
    .align 4;                                   \
    .globl Function                         ;   \
    Function:                               ;   \
        nop                                 ;   \
        nop                                 ;   \
        nop                                 ;   \
        nop                                 ;   \
        nop                                 ;   \
        nop                                 ;   \
        nop                                 ;   \
        nop                                 ;   \
        nop                                 ;   \
        nop                                 ;   \
        nop                                 ;   \
        nop                                 ;   \
        ldr     r0, ps_ptr_ ##Function      ;   \
        blx     _Debugger                   ;   \
    ps_ptr_ ##Function:                     ;   \
        .long   panicString_ ##Function     ;   \
    panicString_ ##Function:                ;   \
        .asciz  genString(Function)         ;

/* ... */
UNIMPLEMENTED_STUB(_hibernate_restore_phys_page)
/* ... */
{% endhighlight %}

When starting, the kernel just worked, except for one thing. On TI OMAP3530, an external abort is asserted whenever
an exclusive instruction (i.e. `ldrex`, `strex`, `clrex`, etc) is used. This caused the kernel to hang before it could
print anything to the serial console. However, for platforms that do not support semihosting, all console output is sent
to an internal buffer. Said buffer can then be dumped from a JTAG board.

To work around this issue, and to make my life easier when I port the kernel to ARMv6 or ARMv5, I removed instances of
the exclusive stores/loads. This wouldn't really matter on this platform as it is uniprocessor anyway. To prevent
these routines from being interrupted during context switches, for example, I added interrupt barriers.  This solved
the issue and allowed me to boot.

Platform Expert
===============

The platform expert is a core component of XNU. It contains all of the hardware specific subroutines for any
specified machine configuration. On ARM systems, this includes setting up the interrupt controller, timers, 
framebuffer, serial UARTs and other core peripherals. My version of the XNU kernel does this by making SoC
plugins for each board configuration.

{% highlight c %}
typedef struct SocDeviceDispatch {
    SocDevice_Uart_Getc             uart_getc;
    SocDevice_Uart_Putc             uart_putc;
    SocDevice_Uart_Initialize       uart_init;
    SocDevice_InitializeInterrupts  interrupt_init;
    SocDevice_InitializeTimebase    timebase_init;
    SocDevice_HandleInterrupt       handle_interrupt;
    SocDevice_GetTimer0_Value       timer_value;
    SocDevice_SetTimer0_Enabled     timer_enabled;
    SocDevice_PrepareFramebuffer    framebuffer_init;
    SocDevice_GetTimebase           get_timebase;
} SocDeviceDispatch;
extern SocDeviceDispatch    gPESocDispatch;
{% endhighlight %}

This allows me to have one SoC dispatch table per board configuration and one standard API to use when communicating 
with basic hardware peripherals.

System Initialization
=====================

With all of the necessary pieces in place, I was able to boot the kernel to a semi-usable state, at least to the point
where the root file system could at least be mounted. 

<figure>
<img src="/images/xnuboot.png" alt="XNU booting and panicking">
</figure>

Getting the kernel up to userland is now the next step, not very much remains other than fixing everything. It needs a 
lot of work to get there. But hey, at least it works, and I'm happy that it got as far as it did.


