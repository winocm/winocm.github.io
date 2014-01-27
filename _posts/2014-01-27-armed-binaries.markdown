---
layout: post
title:  "ARMed binaries"
date:   2014-01-27 11:38:00
categories: boredom
---

Recently, I saw [this](http://blog.softboysxp.com/post/7888230192/a-minimal-168-byte-mach-o-arm-executable-for-ios) article on creating an 
incredibly small 168-byte Mach-O image. I thought I would also take up the same challenge, but instead, I'm not going to use a hex editor.
I used a standard ARM toolchain (`arm-none-eabi-`) to build this binary. Let's walk through how one can craft an incredibly small
Mach-O binary for ARM-styled OS X (not iOS!).

The technical reason as to why this will not work in modern iOS versions and `CONFIG_EMBEDDED` enabled kernels is due to the kernel
wanting to enforce a hard `__PAGEZERO` segment as mapped from the executable load segment commands. A binary that does not have a proper `__PAGEZERO` 
segment will be killed and unloaded immediately. (See [bsd/kern/mach_loader.c](http://opensource.apple.com/source/xnu/xnu-2050.18.24/bsd/kern/mach_loader.c)
for more details.)

Here's the source for said tiny binary:

{% highlight bash %}
.text
.globl _start
_start:
.Lhdrbegin:
    .long    0xfeedface        /* magic */
    .long    12                /* cputype */
    .long    0                 /* cpusubtype */
    .long    2                 /* filetype */
    .long    2                 /* ncmds */
    .long    (.Lend - .Lcmds)  /* sizeofcmds */
    .long    1                 /* flags */
.Lcmds:
.Lcmdseg:
    /* LC_SEGMENT */
    .long 1                    /* LC_SEGMENT */
    .long 56                   /* size */
.Lstr:
    .asciz "Hello, world\n\n\n"    /* segname */
.Lcmdsegcont:
    .long 0x1000                   /* vmaddr */
    .long 0x1000                   /* vmsize */
    .long 0                        /* fileoff */
    .long (.Lend - .Lhdrbegin)     /* filesize */
    .long 7                        /* maxprot */
    .long 5                        /* curprot */
    .long 0                        /* nsects */
    .long 0                        /* flags */
.Lcmdth:
    /* LC_UNIXTHREAD */
    .long 5                        /* LC_UNIXTHREAD */
    .long 84                       /* cmdsize */
    .long 1                        /* flavor, default to 1 */
    .long 17                       /* count, ARM_THREAD_STATE_COUNT */
.Lcodestart:
.code 32
    adr    r1, .Lstr               /* r0 */
    mov    r2, #13                 /* r1 */
    mov    r12, #4                 /* r2 */
    swi    #0x80                   /* r3 */
    mov    r12, #1                 /* r4 */
    swi    #0x80                   /* r5 */
    .long 0                        /* r6 */
    .long 0                        /* r7 */
    .long 0                        /* r8 */
    .long 0                        /* r9 */
    .long 0                        /* r10 */
    .long 0                        /* r11 */
    .long 0                        /* r12 */
    .long 0                        /* sp */
    .long 0                        /* lr */
    .long .Lcodestart              /* pc */
    .long 0                        /* cpsr */
.Lend:
{% endhighlight %}

Let's break it down and see why it works.

# Mach-O Header

{% highlight bash %}
.Lhdrbegin:
    .long    0xfeedface        /* magic */
    .long    12                /* cputype */
    .long    0                 /* cpusubtype */
    .long    2                 /* filetype */
    .long    2                 /* ncmds */
    .long    (.Lend - .Lcmds)  /* sizeofcmds */
    .long    1                 /* flags */
{% endhighlight %}

Every Mach-O binary starts with `0xfeedface` (for 32-bit) and `0xfeedfacf` (for 64-bit). `12` refers to the ARM CPU type, only binaries
targeted for a certain CPU type can execute on said CPU. The CPU subtype limits the binary to a specified CPU. iOS/OS X on ARM executes binaries
with any subtype tat matches the current processor or lower. The following 'chart' demonstrates the compatibility for different processor combinations:

{% highlight bash %}
          armv4t   xscale   armv5   armv6   armv7    cpu_subtype_all (armv4t)
armv4t:   Yes                                        Yes
xscale:   Yes      Yes                               Yes
armv5:    Yes               Yes                      Yes
armv6:    Yes               Yes     Yes              Yes
armv7:    Yes               Yes     Yes     Yes      Yes
{% endhighlight %}

As this is an executable binary, MH_EXECUTE was specified for filetype. No undefined symbols need to also be fixed up,
hance the flag of 1.

# Load Commands

{% highlight bash %}
.Lcmdseg:
    /* LC_SEGMENT */
    .long 1                    /* LC_SEGMENT */
    .long 56                   /* size */
.Lstr:
    .asciz "Hello, world\n\n\n"    /* segname */
.Lcmdsegcont:
    .long 0x1000                   /* vmaddr */
    .long 0x1000                   /* vmsize */
    .long 0                        /* fileoff */
    .long (.Lend - .Lhdrbegin)     /* filesize */
    .long 7                        /* maxprot */
    .long 5                        /* curprot */
    .long 0                        /* nsects */
    .long 0                        /* flags */
{% endhighlight %}

The segment name contains the 'Hello World' string to save space for more code, otherwise this is pretty much a standard
LC_SEGMENT command. Every Mach-O file is mapped at 'fileoff' at 'vmaddr' for 'vmsize'. 

# Crazy UNIX Thread

{% highlight bash %}
.Lcmdth:
    /* LC_UNIXTHREAD */
    .long 5                        /* LC_UNIXTHREAD */
    .long 84                       /* cmdsize */
    .long 1                        /* flavor, default to 1 */
    .long 17                       /* count, ARM_THREAD_STATE_COUNT */
.Lcodestart:
.code 32
    adr    r1, .Lstr               /* r0 */
    mov    r2, #13                 /* r1 */
    mov    r12, #4                 /* r2 */
    swi    #0x80                   /* r3 */
    mov    r12, #1                 /* r4 */
    swi    #0x80                   /* r5 */
    .long 0                        /* r6 */
    .long 0                        /* r7 */
    .long 0                        /* r8 */
    .long 0                        /* r9 */
    .long 0                        /* r10 */
    .long 0                        /* r11 */
    .long 0                        /* r12 */
    .long 0                        /* sp */
    .long 0                        /* lr */
    .long .Lcodestart              /* pc */
    .long 0                        /* cpsr */
{% endhighlight %}

This is a standard ARM `LC_UNIXTHREAD`, however, there are some obvious differences. The code for the executable starts at the thread
state register settings. That doesn't really matter much, however, the program will get those registers set initially during execution.
`pc` is set to the code start dynamically by the linker (which usually ends up being 0x1064 anyhow). 

As `r0` is set to 0 by the successful call to `write(2)`, there's no need to even reset it after the first supervisor call.

However, `adr` is incredibly volatile, remember that.

# Compiling:

{% highlight bash %}
$ arm-none-eabi-gcc ~/tiny.s -o tiny.o -nostdlib -Ttext=0x1000
$ arm-none-eabi-objcopy -O binary test.o test.macho
$ file test.macho
test.raw: Mach-O executable arm
$ $ otool -fahl test.raw 
test.raw:
Mach header
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
 0xfeedface      12          0  0x00          2     2        140 0x00000001
Load command 0
      cmd LC_SEGMENT
  cmdsize 56
  segname Hello, world



   vmaddr 0x00001000
   vmsize 0x00001000
  fileoff 0
 filesize 168
  maxprot 0x00000007
 initprot 0x00000005
   nsects 0
    flags 0x0
Load command 1
        cmd LC_UNIXTHREAD
    cmdsize 84
     flavor ARM_THREAD_STATE
      count ARM_THREAD_STATE_COUNT
	    r0  0xe24f1048 r1     0xe3a0200d r2  0xe3a0c004 r3  0xef000080
	    r4  0xe3a0c001 r5     0xef000080 r6  0x00000000 r7  0x00000000
	    r8  0x00000000 r9     0x00000000 r10 0x00000000 r11 0x00000000
	    r12 0x00000000 sp     0x00000000 lr  0x00000000 pc  0x00001064
	   cpsr 0x00000000
{% endhighlight %}

(Yes, I really had nothing better to do. You can probably make a tiny Mach-O that works with iOS if you add a __PAGEZERO and getting
rid of LC_UNIXTHREAD by replacing it with LC_LOAD_DYLINKER/LC_MAIN. The size should still be sufficiently small enough.)
