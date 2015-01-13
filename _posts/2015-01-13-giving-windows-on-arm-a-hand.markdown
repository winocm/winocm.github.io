---
layout: post
title: "Giving Windows on ARM a hand"
date: 2015-01-13 01:02:03
categories: projects bringup osports
---

Windows on ARM, also known as Windows RT (commercially) is a 'mobile' operating system
that isn't necessarily a mobile operating system. Technically, it comes in two commercial
flavors, both Windows Phone and Windows RT. Both of these 'operating systems' use the NT kernel
and underlying foundation underneath for platform initialization and startup.

This blog post goes in depth on how I managed to port Windows RT 8.1 to run on a generic
ARM Cortex-A15/Cortex-A9 based system. (This procedure will work for Windows Phone 8.x too, but
I never tested that configuration.)

# Introduction

Windows RT has several core system requirements, namely (these don't necessarily follow
the official guidelines, but are instead things I've found out through manual research):

{% highlight bash %}
System processor: ARM Cortex-A9/Cortex-A15 compatible (Must be ARMv7-A&R, support VFPv3+VFP HalfPrec)
Memory: At least 512MB of RAM (can probably do less)
Peripherals:
    - System timer
    - ARM Generic Interrupt Controller (GIC), both CPU and Distributor Interface
    - Framebuffer
    - NS16650 based UART for System Kernel Debug
Firmware: UEFI 2.3/2.4
{% endhighlight %}

Anything else is really considered superflous and is not entirely necessary to boot Windows RT 8.1/8.0
successfully to the user interface. 

The firmware must implement the following ACPI tables:
{% highlight bash %}
- DSDT - Differentiated System Descriptor Table
- DBG2 - Debug Port 2
- FACP - Fixed ACPI Description Table
- FACS - Firmware ACPI Control Structure
- MADT - Machine APIC Descriptor Table
- CSRT - Core System Resource Table
{% endhighlight %}

Without these ports (and without a HAL extension), the system will fail to boot successfully.

(Unfortunately, the `MADT` and `DBG2` tables used for a 'successful' Windows RT boot deviate from ACPI 5.0
specification. The structure for mine closely followed those of the Surface RT, which would mean that
its device firmware violates ACPI spec...?)

# System Timer + Interrupt Controller

Common ARM operating systems need a system timer to drive them along with an interrupt controller
to dispatch interrupts across SMP/UP platforms in an orderly manner. There was really no standard way
to drive all interrupt controllers on ARM until the GIC came around and was put into use with the Cortex-A9
processor.

Cortex-A15 also adds architected timer, removing one other potentially proprietary component from the
system that Cortex-A9 and other processors still rely on for core system functionality.

Initial Windows RT devices did ship with Cortex-A9 processors, so how did they solve the issue? Microsoft
implemented a feature called "HAL Extensions". The Tegra3 family of devices, for example, ships with a library known
as `HalExtTegra2.dll`. This dynamic library controls arming and syncing of the system timer and plugs in to
the rest of the system through a function dispatch table.

For reference, the header `nthalext.h` in the Windows Driver Kit 8.1 can be referenced if you wish to make
your own HAL extension. The following structure comes directly from the WDK:

{% highlight c %}
typedef struct _TIMER_FUNCTION_TABLE {
    PTIMER_INITIALIZE Initialize;
    PTIMER_QUERY_COUNTER QueryCounter;
    PTIMER_ACKNOWLEDGE_INTERRUPT AcknowledgeInterrupt;
    PTIMER_ARM_TIMER ArmTimer;
    PTIMER_STOP Stop;
    PTIMER_SET_DIVISOR SetDivisor;
    PTIMER_SET_MESSAGE_INTERRUPT_ROUTING SetMessageInterruptRouting;
    PTIMER_CHANGE_LOSSLESS_RATE LosslessRateChange;
    PTIMER_SET_INTERRUPT_VECTOR SetInterruptVector;
    PTIMER_FIXED_STALL FixedStall;
} TIMER_FUNCTION_TABLE, *PTIMER_FUNCTION_TABLE;
{% endhighlight %}

Cortex-A15 systems and potentially others can use the system timer defined in the ARMv7 instruction set
architecture (64-bit cp15 register c14, etc), however the issue of finding out which IRQ the timer is tied
to is still a problem.

This issue is solved by adding a `GTDT` ACPI table to the system, which details the specific IRQ number 
the timer lies on.

{% highlight bash %}
[0004]               Secure EL1 Interrupt : 0000001D
[0004]          EL1 Flags (decoded below) : 00000000
                             Trigger Mode : 0
                                 Polarity : 0
                                Always On : 0

[0004]           Non-Secure EL1 Interrupt : 0000001E
[0004]         NEL1 Flags (decoded below) : 00000000
                             Trigger Mode : 0
                                 Polarity : 0
                                Always On : 0
{% endhighlight %}

(Please note: the names of the fields here follow the ARM64/ACPI 5.1 specification, I just used a newer
version of the `iasl` compiler from the [ACPICA](https://www.acpica.org).)

# System Debug

An ARM system can have multiple debug ports, but UART locations aren't really standardized across all ARM
systems. Enter the `DBG2` ACPI table.

The `DBG2` ACPI table defines the name path, port type and port address of a `DBGP` compatible system UART
or USB peripheral that can be attached to the WinDbg system debugger. Windows RT systems may not have a `DBGP`
table, only a `DBG2` table.

The namepath for the system UART must also match the one in the `DSDT`.

# Device Descriptor Information

Instead of using flattened device trees like most ARM based operating systems nowaday, Microsoft chose to support
only ACPI for their systems. This creates a barrier that separates Windows RT systems from standard ARM Linux/BSD 
machines, even if they both use UEFI.

The HAL uses the ACPI tables installed by the UEFI firmware (retrieved from the EFI system table) in order
to determine which addresses are which and where to map system peripherals. In addition, the `ACPI.SYS` driver is also
used in parallel with the plug and play system for device enumeration.

While a FDT device may have a path such as `/uart0@0x10006000` on a Linux system, an ACPI device on Windows RT has
the following path convention: `ACPI\VEND000A`, and so on. (Additional path schemes are available, but are not
documented here for simplicity.)

# Bringup on qemu-system-arm -M virt-rt

'Virt-rt' is a system based on qemu-system-arm's -M 'virt', with some simple modifications, namely the addition of a PL310
L2 cache controller, a NS16550 serial UART, 1GB of RAM PL111 LCD controller, extra storage and some modifications to the processor's
MIDR in order to boot Windows without complaint (Cortex-A15 => Cortex-A9). The system can boot without the PL310 controller
being on the SoC and in the DSDT, however one needs to use the `/DISABLEEXTCACHE` flag in the system load options to avoid
an unnecessary bugcheck.

This system has little in common with the Tegra chipsets that most Windows RT devices have, except that it also implements
an ARM Generic Interrupt Controller.

The system timer is architected, there is no direct timer peripheral like there is on Tegra or other platforms, instead, a 
GTDT table is present in the system firmware. The only ACPI tables are the ones needed, otherwise, it is a barren
platform with barely anything installed.

Once the system boots to EFI, we can start Windows RT from the EFI shell:
{% highlight bash %}
UEFI Interactive Shell v2.087477C2-69C7-11D2-8E39-00A0C969723B 7BC1FF14
EDK IIlProtocolInterface: 752F3136-4E16-4FDC-A22A-E5F46812F4CA 7BC1EC90
UEFI v2.40 (QEMU EFI Jan 11 2015 22:23:51, 0x00000000)FEF5DA4E 76FF52C8
Mapping table
      FS0: Alias(s):HD6b:;BLK1:
          VenHw(B615F1F5-5088-43CD-809C-A16E52487D00)/HD(1,GPT,B68E8885-DF4A-4F15-B04E-40E7903876F1,0x800,0xFF000)
     BLK0: Alias(s):
          VenHw(B615F1F5-5088-43CD-809C-A16E52487D00)

Press ESC in 5 seconds to skip startup.nsh or any other key to continue.
Shell> 
Shell> fs0:\efi\boot\bootarm
InstallProtocolInterface: 5B1B31A1-9562-11D2-8E3F-00A0C969723B 7BC437A8
ConvertPages: failed to find range 10000000 - 100F0FFF
add-symbol-file bootarm.pdb 0x76E4D400
Loading driver at 0x00076E4D000 EntryPoint=0x00076E4F8F1 bootarm.efi
InstallProtocolInterface: BC62157E-3E33-4FEC-9920-2D3B36D750DF 7BC1FD10
InstallProtocolInterface: 752F3136-4E16-4FDC-A22A-E5F46812F4CA 7F802BC8
{% endhighlight %}

And once we have access to the Windows Boot Manager and have the kernel running, we can attach 
WinDbg to the host and break into the system's kernel debugger:

{% highlight bash %}

Microsoft (R) Windows Debugger Version 6.3.9600.17298 X86
Copyright (c) Microsoft Corporation. All rights reserved.

Opened \\.\com1
Waiting to reconnect...
Connected to Windows 8 9600 ARM (NT) Thumb-2 target at (Tue Jan 13 06:48:08.845 2015 (UTC + 0:00)), ptr64 FALSE
Kernel Debugger connection established.
...
*** ERROR: Symbol file could not be found.  Defaulted to export symbols for ntkrnlmp.exe - 
Windows 8 Kernel Version 9600 MP (1 procs) Free ARM (NT) Thumb-2
Built by: 9600.16384.armfre.winblue_rtm.130821-1623
Machine Name:
Kernel base = 0x8105a000 PsLoadedModuleList = 0x81231100
System Uptime: 0 days 0:00:00.013
Break instruction exception - code 80000003 (first chance)
*******************************************************************************
*                                                                             *
*   You are seeing this message because you pressed either                    *
*       CTRL+C (if you run console kernel debugger) or,                       *
*       CTRL+BREAK (if you run GUI kernel debugger),                          *
*   on your debugger machine's keyboard.                                      *
*                                                                             *
*                   THIS IS NOT A BUG OR A SYSTEM CRASH                       *
*                                                                             *
* If you did not intend to break into the debugger, press the "g" key, then   *
* press the "Enter" key now.  This message might immediately reappear.  If it *
* does, press "g" and "Enter" again.                                          *
*                                                                             *
*******************************************************************************
*** ERROR: Symbol file could not be found.  Defaulted to export symbols for ntkrnlmp.exe - 
nt!DbgBreakPointWithStatus:
8107b3fc defe     __debugbreak
>
{% endhighlight %}

Some minor instruction patching (mainly to disable PatchGuard, nt!KiVerifyScopesExecute), and we're
able to get far into system initialization, far enough to get to a graphical user
interface:

<figure>
<img src="/images/rtboot.png" alt="RT booting to UI (Windows PE)">
</figure>

(Currently the system has very little functionality in terms of usage, there is no proper input driver
nor storage. The framebuffer provided to the system is the exact same one EFI sets up. Once additional
functionality is implemented, the system can become usable as a desktop Windows instance.)

Oh yeah, speed isn't too great either without KVM, but hey, it works for what it's worth.

# But what about Secure Boot and TPMs?

That's really optional. (And no, I don't think you'll ever get past Windows Logo testing either.)

# Does this mean I can break Secure Boot on my Surface RT?

No.

# tl;dr what do I need to do???

Make a new UEFI target, add ACPI support to it, add a BlockIo driver, add EFI GOP support,
ARM support, write ACPI descriptor tables, debug the system, ensure that the proper
peripherals are in place, get a Windows image, master it properly for your system, test, debug,
test, debug, test, debug, lose a couple nights of sleep and then finally marvel at your creation.

It's a lot of work.

