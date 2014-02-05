---
layout: post
title:  "The Phantasy Zone"
date:   2014-02-04 22:38:00
categories: research
---

[TrustZone](http://arm.com/products/processors/technologies/trustzone/index.php) is a feature present in many modern ARM processors
such as the Cortex-A8 or A9. Essentially, it provides an additional 'world' for execution, partitioning the processor into two effective
virtual ones. The 'secure' state has banked copies of most of the system registers (for example, `TTBR0`/translation-table base register 0)
and implements the monitor mode versus the 'non-secure' mode that does not.

TrustZone isn't simply a processor feature, it's a complete collection of technologies that also include the [BP147 AMBA Protection Controller](http://infocenter.arm.com/help/topic/com.arm.doc.dto0015a/DTO0015_primecell_infrastructure_amba3_tzpc_bp147_to.pdf)
and the [TZASC](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0431b/). Using these technologies properly allows you to implement a 
'secure' world, seperated from the normal address bus and 'non-secure' applications.

This post will mainly be concerned about how one can implement TrustZone on a machine with security extensions. The example used here
will be an iPhone 4, however, any device that boots into a 'secure' state can be switched over to a 'non-secure' state very easily. 

# Worlds Collide

The 'secure' world has its own set of banked registers, including its own `sp`, `lr` and `spsr`, much like other ARM modes. However, the biggest
difference between the two worlds lie in peripheral accesses. A peripheral can check for the `AxPROT[1]` signal and yield an error/return 0 if
the processor is not in a secure state. System page tables can also mark a region of memory as 'secure' or 'non-secure' as necessary, however, this
relies on the trusted OS only being able to manage the system page tables. Additionally, certain system coprocessors are unavailable in non-secure 
mode (`SCR` for example), writing/reading from them will result in an undefined instruction error. 

The processor always will boot in the most secure state (`SCR` = 0), along with all of its peripherals on reset. Returning to the 'non-secure' world
can be done by writing the `SCR.NS` bit. A switch between worlds is effective immediately. Returning back to the 'secure' world is done using the `SMC`
instruction.

If you're making a 'trusted OS', please make sure to properly harden your `SMC` handlers and prevent register/information leaks as much as you can. Don't
be a Motorola.

# Setting up TrustZone

The `SMC` instruction depends on the MVBAR being set to a valid MVA, if there are no instructions present at the specified address, `pc` will be redirected
there anyhow. The MVBAR format is very similar to the standard ARM vector table, however, the `SVC` handler is replaced with the `SMC` handler. It's pretty
simple. 

{% highlight c %}
EnterARM(start)
    b       _reset_secure_tramp                    /* Reset vector. */
    nop                                            /* Unused, undefined. */
    b       _reset_secure_sp_tramp                 /* SMC instruction handler. */
    nop                                            /* Unused, prefetch abort. */
    nop                                            /* Unused, data abort. */
    nop                                            /* Unused, address line exception. */
    nop                                            /* Unused, IRQ. */
    nop                                            /* Unused, FIQ. */
{% endhighlight %}

`SMC` can then be used in a privileged context to enter monitor mode. It forms the communication link between both worlds. The `SMC` exception handler 
works very much like a standard ARM exception handler. You must use `movs pc,lr`, `sub pc,lr,#imm`, `rfe` or somesuch to return from the exception state.

(Please do note that the TTBR used for address translation in secure mode will be the banked copy of the register.)

# 'Trusted' iPhone

Since the iPhone 4 is one of those platforms that implement TrustZone, but also reset in secure mode, we can use it as a very 'cheap' development
platform to create a simple TrustZone bootloader.

A terribad sample implementation of a stubloader is available [here](https://github.com/winocm/tzboot).

iBoot seems to run fine with `SCR.NS=1` based on what I've tested. Your mileage may vary.

(There's also a QEMU fork with TrustZone support available [here](https://github.com/jowinter/qemu-trustzone), however, it is not perfect.)
