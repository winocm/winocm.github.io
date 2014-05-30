---
layout: post
title:  "DisARMing the iOS kernel"
date:   2014-05-30 11:50:00
categories: technical
---

Disclaimer: The information this post pertains to iOS 6.x for the most part, some of it still applies to iOS 7.0, but not really for
7.1.1 and beyond. You are free to use the concepts presented for your own nefarious/'research' purposes.

# CVE-2014-1320

This bug relates to `IOPlatformArgs`, a member that's been published in the IOKit registry for over 13 years. It's been present
since the original OS X releases and was very useful for leaking the kernel base address.

In the case of iPhone OS, two booter arguments are leaked, boot-args and the head of the device tree. This is useful as
the kernel bootstrap and system page tables are located at a fixed offset from the beginning of boot-args. (This offset is about 8
pages after the end of boot-args, at least in my tests). As this location is easily modifiable by a kernel write-what-where style
vulnerability, one can simply write their own L1 page directory mappings into the translation table. By writing the entries, one
can guarantee that the area will be translated successfully (provided the address is out of the reach of the user map) and that
the system data/prefetch abort handlers will not be called as the translated entries do not exist in the processor's TLB beforehand.

Remapping the kernel into userspace also means that user programs can directly modify kernel memory, allowing for
easy patching of the code and also for shellcode insertion. This also removes the need for read, write and execute primitives
(at least, pre-iOS 7.1.1). Please do note that as of the current release, the kernel page table base remains static.

I personally remap the kernel using 1MB section maps with attributes: Shareable, Non-cacheable (strongly ordered), User + 
Supervisor Read/Write.

The booter memory layout looks very much like this:

{% highlight c %}
[Zero page (low resume exception vectors)]
[Kernel + prelinked kernel extensions]
[Device tree]
[RAM disk (if present)]
[Boot-args]
[Kernel bootstrap L1 table]
[Kernel bootstrap L2 tables (for extra memory, post iOS 4)]
[Kernel system L1 table]
[.....]
{% endhighlight %}

# Easy Patching

Once the kernel is remapped into user memory, one can simply modify `sysent[0]` to point to a user-supplied payload. However,
common jailbreaks such as `p0sixspwn` and `evasi0n(7)` insist on modifying kernel page tables. This is not actually needed, as
one can simply use the `DACR` (domain access control register) instead. Utilizing the `DACR` in manager mode for all domains allows
one to bypass page permissions, including NX, and also results in far less kernel reads/writes (for page table walking) and TLB flushes.

Please note that interrupts (both FIQ and IRQ) must be disabled in order to suspend context-switching, as running iOS with `DACR`
set to manager mode ends up making it do incredibly Weird Things(tm).

If one is also using the backdoor mapped kernel technique in order to read kernel memory, please take care in order to zero out the 
mappings, as the kernel map is globally visible across all applications.

Additionally, the kernel ASLR virtual base can also be found by simply reading through boot-args in remapped memory (the address of the
boot-args in physical memory space is stored in the zero page, to convert from physical to virtual, simply subtract the PA base and add
the remap base.)

Modifying the kernel from userspace by directly copying patches in is viable, however, you may experience issues related
to writeback caching. Patching the kernel directly from supervisor to the kernel virtual addresses is preferred for that reason.

# Example shellcode

{% highlight bash %}
#include <mach/arm/asm.h>

    .data
    .code 16
    .thumb_func
    .align 2
    .globl EXT(shellcode_begin)
    .globl EXT(shellcode_end)
LEXT(shellcode_begin)
    stmfd   sp!, {r4-r7, lr}

    // Disable interrupts, we don't want any timer/HW events during this critical section.
    cpsid   if

    // Get DACR's original value
    mrc     p15, 0, r4, c3, c0, 0

    // Switch over to manager mode
    mov     r5, #-1
    mcr     p15, 0, r5, c3, c0, 0

    ...

    // Zero out the mappings
    ldr     r0, EXT(u_ttbBase)
    ldr     r1, EXT(u_ttbSize)
    mov     r2, #0
_ClearMap:
    str     r2, [r0], #4
    subs    r1, r1, #4
    bgt     _ClearMap  

    ...

    // Clear unified TLB and restore DACR and interrupts
    mov     r0, #0
    mcr     p15, 0, r0, c8, c7, 0
    mcr     p15, 0, r4, c3, c0, 0
    isb     sy

    cpsie   if

    movs    r0, #0
    ldmfd   sp!, {r4-r7, pc}

    .align 2

    ...

    .globl EXT(u_ttbSize)
    .globl EXT(u_ttbBase)
LEXT(u_ttbBase)
    .long 0
LEXT(u_ttbSize)
    .long 0

    ...

LEXT(shellcode_end)
    nop
{% endhighlight%}

# Afterlife

That's pretty much it, by utilizing the power of CVE-2014-1320, the ARM architecture and also a simple kernel write-what-where
style vulnerability, one can own the iOS kernel and make a simple kernel exploitation binary for the sake of untethering or whatever.

These techniques should work on any iOS version, as the kernel L1 page directory base is always located at that same static
offset from the beginning of physical RAM. 

(It is easy to even calcaulate without knowing the contents of IOPlatformArgs too...)

The ARM architecture provides incredibly powerful, powerful components that make exploitation a lot easier. (If you know how
they all work!) 

:)

