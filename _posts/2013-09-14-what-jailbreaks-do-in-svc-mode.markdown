---
layout: post
title:  "What jailbreaks do (in SVC mode)"
date:   2013-09-14 12:09:00
categories: research
---

God, it's so late at night. This post explains how the kernel exploits work and the
general code flow for patching an iOS 4.x/5.x and an 6.x kernel. (Mainly the interesting
bits, and how you can potentially patch these kernels on your own.)

iOS 4.x/5.x: True Simplicity
============================

Jailbreaks in this iOS version relied on a certain characteristic of the XNU `pmap` (physical
mapping memory manager.). This characteristic was that kernel and user memory were shared in one 
address space. The address space for user processes only switched in `machine_switch_context` and 
a few other functions (see `osfmk/arm/thredinit.c` and others). This meant that a user could redirect
a function pointer `sysent` to user executable code. This code could do many things such as hook various
kernel functions or patch signature checking routines, and so on. 

The only limitation is that the code page must be mapped as `r-x`, which is incredibly simple. Map the 
page using `vm_allocate()` or `mmap()`, change the protection flags and then invalidate/clean the caches.

iOS 6.x: Level Up
=================

iOS 6.x doesn't make things very easy. The kernel space executes in its own distinct address space,
without any user mapped processes. This makes it impossible for the user to change `sysent` to a user
mapped address, however, there are still flaws in this design.

In addition, the kernel text is mapped as read-only. Any writes will cause a data abort with DFSR bits
equivalent to permission fault on page/section. 

This is incredibly easy to circumvent, mainly because of the deisgn of the pmap subsystem. Apple uses
the `phys_to_virt` and `virt_to_phys` (or equivalent) macros to convert virtual addresses to physical
addresses. If you know the kernel slide, you can convert from a physical address to a virtual one by
subtracting the physical base and adding the virtual base, and vice-versa.

By walking through the kernel_pmap and through the ARM translation table entries, it's possible to
change the `APX` and `AP` bits in the pages to reflect supervisor read+write. After the entries have
been modified, the TLB must be flushed. When the TLB is flushed successfully, the new modified
mappings will be used.

To simplify the jailbreak process, it's much simpler to `bcopy()` the shellode over to a nop region
present in the sleep token or to `__start` in the kernel. There's not a lot of space, so be warned.
This way, you don't have to modify the page table entry for a page allocated by `kalloc()` or whatever.

You can also very much use the `sysent` technique on kernel mapped shellcode, but that's your choice.

Kernel Exploits: A Basic Primer
===============================

All of the kernel exploits always control PC in some way to cause maximum profit or mayhem. The PC or
program counter always points to the currently executing instruction.

When controlled, the PC can point to arbitrary code gadgets such as:

{% highlight c %}
    ldr    r0, [r1]
    bx     lr
{% endhighlight %}
(This specific gadget reads the address in r1 and returns the loaded 32-bit value to the callee.)

By combining the use of PC control and register control, you can effectively run code very easily.

Jailbreak Patches
=================

There are a few jailbreak patches, mainly:

* vm_map_enter
* vm_map_protect
* Sandbox.kext path evaluate
* AppleMobileFileIntegrity.kext codesign flags disable
* proc_enforce sysctl setting
* task_for_pid_0
* boot-args (iOS 6.x+)
* kernel codesign enforcement disable
* PE_I_can_has_debugger _debug_enabled value

These values are patched using a simple ldr/str routine in the kernel shellcode. Nothing too fancy.

When the shellcode is done, it is always best to invalidate the instruction cache to PoU (and probably
data).

Quick explanation of what each patch does
=========================================

<b>vm_map_enter/vm_map_protect</b>: Used in MobileSubstrate, allows pages to be mapped as `rwx`, not just `rw-` or `r-x`.

<b>Sandbox.kext path evaluate</b>: Used for allowing applications to execute properly outside of the specified sandbox domain (ie: MobileSafari in a stashed Applications directory).

<b>AppleMobileFileIntegrity.kext codesign flags disable</b>: Used for allowing non-codesigned applications.

<b>proc_enforce sysctl setting</b>: Disable all MAC policy hooks for processes.

<b>task_for_pid_0</b>: Allow applications with the `task-for-pid_allow` entitlement to use `task_for_pid` on the kernel task.

<b>boot-args</b>: Allows launchctl to load unsigned LaunchDaemon property lists.

<b>kernel codesign enforcement disable</b>: Used for allowing non-codesigned pages.

<b>PE_I_can_has_debugger</b>: Allow basic debugging functionality and much more relaxed sandbox.

Have fun!