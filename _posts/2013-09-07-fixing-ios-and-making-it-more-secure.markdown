---
layout: post
title:  "Fixing iOS and making it more secure"
date:   2013-09-07 20:42:00
categories: research rants
---

The iOS kernel is incredibly vast, with lots of code and security features. iOS 6 was a pain to exploit because 
of many reasons, including some strange design decisions. Take this blog post as a way to improve iOS's security
and reliability. (This also includes some information about mitigating dyld/launchd abuse.)

iOS has many architectural holes that lead to various issues, mainly problems from Mac OS X that migrated
over to iOS.

Several parts were removed from this article.

The Kernel
==========

The kernel has several issues. Mainly those relating to `phys_to_virt` and `virt_to_phys`. I understand that fixing pmap
to not use these macros may be an issue, however, it does pose many security risks.

If a user knows the virtual base of the kernel, then they know the physical base of the kernel. A user can then walk 
the kernel translation tables and modify TTEs on the fly. (I personally use this in my 6.1.3 jailbreak to enable RW on 
specified kernel segments.). To mitigate this, randomizing the physical base of the kernel would be preferred. 

This works because the kernel is mapped linearly in SDRAM, mainly the beginning of SDRAM + 4096 bytes.

Apple, you also need to fix IOKit. It's the buggiest thing ever. Your kernel slide is dangling in front of everyone.

Stupidity in iBoot
==================

iBoot is an embedded bootloader used on Apple devices. However, it tends to do stupid things. For example,
load address and iBoot is mirrored every 512MB. This poses a problem as it allows a malicious user to overwrite
iBoot by copying data over it. They can also patch iBoot in realtime and run code from the load address.

There is no NX, no heap protection. No anything.

You should look at that.

Hopefully, this'll all make iOS far more secure as a mobile platform.

