---
layout: post
title:  "Evading iOS Security"
date:   2014-01-12 01:30:00
categories: projects research
---

Here's some code:

{% highlight c %}
main() {
	syscall(0, 0x41414141, -1);
}
{% endhighlight %}

Here's what happens when you run it on a device using evasi0n7:
{% highlight bash %}
panic(cpu 0 caller 0x9dc204d7): sleh_abort: prefetch abort in kernel mode: fault_addr=0x41414140
r0:   0xffffffff  r1: 0x27dffdec  r2: 0x2bf0e9c4  r3: 0x2bf0e95c
r4:   0x41414141  r5: 0xa48782d4  r6: 0x81b6f594  r7: 0x8f5d3fa8
r8:   0x9df1d614  r9: 0x81b6f330 r10: 0x9df1db00 r11: 0x00000006
r12:  0x00000000  sp: 0x8f5d3f60  lr: 0x93dd7048  pc: 0x41414140
cpsr: 0x20000033 fsr: 0x00000005 far: 0x41414140
{% endhighlight %}

And here's what happens on an ARM64 device that also uses evasi0n7:
{% highlight bash %}
panic(cpu 0 caller 0xffffff801522194c): PC alignment exception from kernel. (saved state: 0xffffff800dbc4640)
	  x0: 0x2bea99c400401e08  x1:  0x00402fe92bea995c  x2:  0xbdf4cd1500403008  x3:  0x0040156400000000
	  x4: 0x0040156400000000  x5:  0x000000002bea995c  x6:  0x00402fe92bea995c  x7:  0xffffff80975df438
	  x8: 0xffffffff41414141  x9:  0x000000000000000e  x10: 0xffffff8096f9b100  x11: 0x0000000000000000
	  x12: 0x0000000003000004 x13: 0x0000000000401420  x14: 0x000000002be925a1  x15: 0x0000000000402ffe
	  x16: 0x0000000000000000 x17: 0x0000000000000000  x18: 0x0000000000000000  x19: 0xffffff8096f9b410
	  x20: 0xffffff80977203f0 x21: 0xffffff80975df438  x22: 0xffffff80155ea3c8  x23: 0x0000000000000018
	  x24: 0xffffff8096f9b418 x25: 0x0000000000000000  x26: 0x0000000000000006  x27: 0x0000000000000000
	  x28: 0xffffff80155ea3c8 fp:  0xffffff800dbc4a60  lr:  0xffffff8015362434  sp:  0xffffff800dbc4990
	  pc:  0xffffffff41414141 cpsr: 0x60000304         esr: 0x8a000000          far: 0xffffffff41414141
{% endhighlight %}

..And here's the system call handler for system call 0... (ARM32 of course!)
{% highlight bash %}
__text:00000000                 CODE32
__text:00000000                 STMFD           SP!, {R4-R7,LR}
__text:00000004                 MOV             R5, R2
__text:00000008                 MOV             R6, R1
__text:0000000C                 LDR             R0, =0x9E415A34
__text:00000010                 BLX             R0
__text:00000014                 LDR             R0, =0x9E41E160
__text:00000018                 BLX             R0
__text:0000001C                 LDR             R0, =0x9E415958
__text:00000020                 BLX             R0
__text:00000024                 LDR             R0, [R6]
__text:00000028                 CMP             R0, #0
__text:0000002C                 BEQ             locret_50
__text:00000030                 MOV             R4, R0
__text:00000034                 LDR             R0, [R6,#4]
__text:00000038                 LDR             R1, [R6,#8]
__text:0000003C                 LDR             R2, [R6,#0xC]
__text:00000040                 LDR             R3, [R6,#0x10]
__text:00000044                 BLX             R4
__text:00000048                 STR             R0, [R5]
__text:0000004C                 MOV             R0, #0
__text:00000050
__text:00000050 locret_50                               ; CODE XREF: __text:0000002Cj
__text:00000050                 LDMFD           SP!, {R4-R7,PC}
__text:00000050 ; ---------------------------------------------------------------------------
__text:00000054 off_54          DCD 0x9E415A34          ; DATA XREF: __text:0000000Cr
__text:00000058 off_58          DCD 0x9E41E160          ; DATA XREF: __text:00000014r
__text:0000005C off_5C          DCD 0x9E415958          ; DATA XREF: __text:0000001Cr
{% endhighlight %}

Jailbreaking ruins security and integrity. Enough said. Have a good day. 

(Oh, it also can be used by any user, including `mobile`. One can wonder if TaiG or such is using this as a
back door...)