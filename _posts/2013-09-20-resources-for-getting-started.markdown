---
layout: post
title:  "Resources for getting started with 'iOS Hacking'"
date:   2013-09-20 21:35:15
categories: research
---

iOS is a very very large operating system. Which a codebase so large, there has to be at least one bug in
the entire system stack. There really isn't any resource available for 'new' people to easily join, or
to for them to learn what to even do. 

If you're well versed in ...everything here, feel free to ignore this post entirely. It's not technical. :(

# ARMing the Demon

I'd at least recommend reading the following technical resources:

* [ARM Architecture Reference Manual for ARMv7-A and ARMv7-R](http://www.cs.utsa.edu/~whaley/teach/FHPO_F11/DDI0406B_arm_architecture_reference_manual_errata_markup_9_0.pdf)
* [ARM Architecture Reference Manual for ARMv8-A](https://silver.arm.com/download/download.tm?pv=1448511)

Comprehending the ARM/ARM64 architecture (at least the Programmer's Model and the system control registers)
helps you to understand how the processor works, from a systems programmer point of view. You should know
how various concepts of the processor relate with each other.

Older processors have their own quirks, you can still find their stuff on the ARM developer website. 
(such as: ARM926EJ-S, ARM1176JF-S, etc).

# OS Design and Architecture

To understand low level parts of the OS, you should study source code from other operating systems to gain
knowledge of various system concepts, such as:

* Spinlocks/mutexes
* Virtual memory management
* Threads/context switching
* Platform initialization
* User/kernel system call layer
* And so on.

A good resource to start with is the [FreeBSD/Linux Cross Reference](http://fxr.watson.org/).

# iOS/OS X Source Code

A lot of OS X is open source! You can look at various projects on [Apple's open source software portal](http://opensource.apple.com).

For example:

* [CF-744.19 - CoreFoundation](http://opensource.apple.com/source/CF/CF-744.19/)
* [dyld-210.2.3 - The dynamic linker](http://opensource.apple.com/source/dyld/dyld-210.2.3/)
* [objc4-532.2 - The Objective-C runtime](http://opensource.apple.com/source/objc4/objc4-532.2/)
* [IOSerialFamily-60 - IOKit drivers for serial devices](http://opensource.apple.com/source/IOSerialFamily/IOSerialFamily-60/)
* [xnu-2050.24.15 - The Mac OS X kernel](http://opensource.apple.com/source/xnu/xnu-2050.24.15/)

There's an incredible amount of source code, and it's all good for reference. 

# Programming/Reverse Engineering

Obviously, you should be well versed in C/C++/ObjC/ObjC++ and should know how a processor works. If you
can't meet this requirement, go ahead and start doing something! 

You should also look at the `/usr/include` folder for interesting headers.

# Security/Fuzzing

This comes naturally after you understand how the entire system works. After you know what it does, see if
you can break it by doing things. Usually, the best part is when it generates a panic log with a register
context. ;)

# Jailbreaking

[The iPhone Wiki](http://theiphonewiki.com) is a decently good resource, but only go to it when you can't
find anything more technical related to the subject. That's not to say it's not a good resource, it really
is a brilliant thing.

Look at the jailbreak patches for the kernel and analyze how they work in context, and the same for iBoot. 
It also helps to analyze the kernel for additional bugs and interesting behavior indirectly. After all, you
might notice something no one else has really seen before.

# Don't be overentitled.

This is important, remain at least, ethical and humble about what you do. It really helps. (Though, not
everyone seems to care about that.). Realize what the community is, and how it should be a community.

We need more smart people, and less script kiddies. Will this ever happen?