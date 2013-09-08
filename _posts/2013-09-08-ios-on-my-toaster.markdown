---
layout: post
title:  "iOS on my toaster?"
date:   2013-09-08 10:49:00
categories: research
---

Again, I do a lot of work relating to the XNU kernel. You should very well know what that is, I don't want to 
explain it another time. My work relates to bringing up the kernel on ARM platforms.. and seeing it break and work.

{% highlight bash %}
CODE SIGNING: proc 1(launchd) loaded detached signatures for file (libSystem.B.dylib) range 0x0:0x48000 flags 0x3
CODE SIGNING: cs_validate_page: mobj 0xa0ab8e8c off 0x0 size 0x1000: SHA1 OK
dyld: Library not loaded: /usr/lib/system/libcache.dylib
  Referenced from: /usr/lib/libSystem.B.dylib
  Reason: no suitable image found.  Did find:
	/usr/lib/system/libcache.dylib: unknown file type, first eight bytes: 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
	/usr/lib/system/libcache.dylib: stat() failed with errno=0
panic(cpu 0 caller 0x801065ed): sleh_undef: undefined user instruction
{% endhighlight %}

Fun stuff, right?

This post explains how to bring up a new hardware platform, at least on my XNU version, it should be relatively the
same for Apple's. Though, I must ask, why are you here if you have Apple's trunk, you should know how to add a pexpert. ;P

Platform Expert
===============

There are two types of platform experts, the core platform expert and the IOKit platform expert. The core platform expert
handles very low level hardware initialization such as GIC init, framebuffer init, timer and so on. The IOKit platform 
expert is responsible for providing `IORTC`/`IONVRAM` registry nodes and connecting the high level IOKit subsystems to
the hardware.

We'll be talking about the core platform expert in this article, and one of my favourite platforms, Samsung S5L8930X.

This platform doesn't have too much documentation, but it is relatively versatile. It is used in the iPhone 4, iPod touch 4G,
iPad 1 and so on. You may already know it as 'A4'.

SoC Dispatch Subsystem
======================

Instead of having one massive `ifdef` for generic things such as `__arm_timer_init` for example, I preferred to use a 
SoC dispatch table. The dispatch table contains function pointers to platform initialization functions. This includes things such
as initializing the framebuffer, GIC, timer and so on.  The structure is defined as follows:

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

This will eventually be refactored as to provide serial-exclusive output for systems that do not have a framebuffer, and vice-versa.
Currently, it just prints out the same information to both serial and framebuffer. If you want, you can implement it. 

Serial KDP needs to be fixed too. :(

Grand Central
=============

The global interrupt controller handles all device interrupts and dispatches them to the processor. Interrupts are essentially 
asynchronous notifications to the processor from a hardware peripheral. XNU starts up without interrupts, mainly to prevent early system
crashes when core subsystems are being initialized. It's 'unthreaded', if you will.

The purpose of the GIC in my version is to provide IRQs for the system timer. Other hardware peripherals can have their own IRQ handles
and then be dispatched from the SoC interrupt handler. 

This provides a relatively small platform-dependent way to dispatch interrupts.

On S5L8930X, we have 4 vectored interrupt controllers. Each of these handles a set of interrupts and will dispatch them to the processor.
The initialization code goes like this:

{% highlight c %}
void S5L8930X_interrupt_init(void)
{
    /* Disable interrupts */
    ml_set_interrupts_enabled(FALSE);

    /* Disable all interrupts. */
    HwReg(gS5L8930XVic0Base + VICINTENCLEAR) = 0xFFFFFFFF;
    HwReg(gS5L8930XVic1Base + VICINTENCLEAR) = 0xFFFFFFFF;
    HwReg(gS5L8930XVic2Base + VICINTENCLEAR) = 0xFFFFFFFF;
    HwReg(gS5L8930XVic3Base + VICINTENCLEAR) = 0xFFFFFFFF;

    HwReg(gS5L8930XVic0Base + VICINTENABLE) = 0;
    HwReg(gS5L8930XVic1Base + VICINTENABLE) = 0;
    HwReg(gS5L8930XVic2Base + VICINTENABLE) = 0;
    HwReg(gS5L8930XVic3Base + VICINTENABLE) = 0;

    /* Please use IRQs. I don't want to implement a FIQ based timer decrementer handler. */
    HwReg(gS5L8930XVic0Base + VICINTSELECT) = 0;
    HwReg(gS5L8930XVic1Base + VICINTSELECT) = 0;
    HwReg(gS5L8930XVic2Base + VICINTSELECT) = 0;
    HwReg(gS5L8930XVic3Base + VICINTSELECT) = 0;

    /* Unmask all interrupt levels. */
    HwReg(gS5L8930XVic0Base + VICSWPRIORITYMASK) = 0xFFFF;
    HwReg(gS5L8930XVic1Base + VICSWPRIORITYMASK) = 0xFFFF;
    HwReg(gS5L8930XVic2Base + VICSWPRIORITYMASK) = 0xFFFF;
    HwReg(gS5L8930XVic3Base + VICSWPRIORITYMASK) = 0xFFFF;

    /* Set vector addresses to interrupt numbers. */
    int i;
    for(i = 0; i < 0x20; i++) {
        HwReg(gS5L8930XVic0Base + VICVECTADDRS + (i * 4)) = (0x20 * 0) + i;
        HwReg(gS5L8930XVic1Base + VICVECTADDRS + (i * 4)) = (0x20 * 1) + i;
        HwReg(gS5L8930XVic2Base + VICVECTADDRS + (i * 4)) = (0x20 * 2) + i;
        HwReg(gS5L8930XVic3Base + VICVECTADDRS + (i * 4)) = (0x20 * 3) + i;
    }

    return;
}
{% endhighlight %}

This code initializes the interrupt controller and disables all interrupts.

The End of Time
===============

The timer is a core part of the system. XNU relies on it to perform maintenance of the system clock. Without the timer, the kernel
would be broken in all sorts of strange ways. 

The kernel by default is tickless, however, my design makes it rely on timer interrupts to keep time maintained. The event timer (etimer)
keeps track of the events to pop when a certain deadline is passed. Older xnu revisions used `hertz_tick`, that is outside the context
of my kernel and this article.

During platform initialization, the IRQ line for the timer should be unmasked and the timer should be initialized. (I couldn't find any 
good documentation on the system timer, as it is a proprietary part, so the implementation may be completely wrong.)

{% highlight c %}
void S5L8930X_timebase_init(void)
{
    assert(gS5L8930XTimerBase);

    /* Set rtclock stuff */
    timer_configure();

    /* Disable the timer. */
    S5L8930X_timer_enabled(FALSE);

    /* Enable the interrupt. */
    HwReg(gS5L8930XVic0Base + VICINTENABLE) = 
        HwReg(gS5L8930XVic0Base + VICINTENABLE) | (1 << 5) | (1 << 6);

    /* Enable interrupts. */
    ml_set_interrupts_enabled(TRUE);

    /* Wait for it. */
    kprintf(KPRINTF_PREFIX "waiting for system timer to come up...\n");
    S5L8930X_timer_enabled(TRUE);

    clock_initialized = TRUE;
    
    while(!clock_had_irq)
        barrier();

    return;
}

void S5L8930X_handle_interrupt(void* context)
{
    /* Timer IRQs are handled by us. */
    if(HwReg(gS5L8930XVic0Base + VICADDRESS) == 6) {
        /* Disable timer */
        S5L8930X_timer_enabled(FALSE);

        /* rtclock_intr */
        rtclock_intr((arm_saved_state_t*) context);

        /* Update absolute time */
        clock_absolute_time += (clock_decrementer - S5L8930X_timer_value());
    
        /* EOI. */
        HwReg(gS5L8930XVic0Base + VICADDRESS) = 0;

        /* Enable timer. */
        S5L8930X_timer_enabled(TRUE);

        /* We had an IRQ. */
        clock_had_irq = TRUE;
    }
    return;
}
{% endhighlight %}

In the IRQ handler, make sure to reinitialize and reset the timer.

Framebuffer
===========

Framebuffer is very easy to initialize. On iBoot based platforms, this is done by iBoot itself, hence, very simple
framebuffer initialization code:

{% highlight c %}
void S5L8930X_framebuffer_init(void)
{
    PE_state.video.v_depth = 4 * (8);   // 32bpp
    
    kprintf(KPRINTF_PREFIX "framebuffer initialized\n");

    initialize_screen((void*)&PE_state.video, kPEEnableScreen);
    return;
}
{% endhighlight %}

However, for platforms that do not use iBoot, it is not as simple...
{% highlight c %}

void Omap3_framebuffer_init(void)
{
    /* This *must* be page aligned. */
    struct dispc_regs* OmapDispc = (struct dispc_regs*)(gOmapDisplayControllerBase + 0x440);

    /* Set defaults */
    OmapDispc->timing_h = 0x1a4024c9;
    OmapDispc->timing_v = 0x02c00509;
    OmapDispc->pol_freq = 0x00007028;
    OmapDispc->divisor = 0x00010001;
    OmapDispc->size_lcd = 0x02ff03ff;
    OmapDispc->config = (2 << 1);
    OmapDispc->control = ((1 << 3) | (3 << 8));
    OmapDispc->default_color0 = 0xffff0000;
    
    /* Initialize display control */
    OmapDispc->control |= DISPC_ENABLE;
    OmapDispc->default_color0 = 0xffff0000;
    
    /* initialize lcd defaults */
    uint32_t vs = OmapDispc->size_lcd;
    uint32_t lcd_width, lcd_height;
    
    lcd_height = (vs >> 16) + 1;
    lcd_width = (vs & 0xffff) + 1;
    kprintf(KPRINTF_PREFIX "lcd size is %u x %u\n", lcd_width, lcd_height);
    
    /* Allocate framebuffer */
    void* framebuffer = pmap_steal_memory(lcd_width * lcd_width * 4);
    void* framebuffer_phys = pmap_get_phys(kernel_pmap, framebuffer);
    bzero(framebuffer, lcd_width * lcd_height * 4);
    kprintf(KPRINTF_PREFIX "software framebuffer at %p\n", framebuffer);
    
    /* Set attributes in display controller */
	OmapDispc->gfx_ba0 = framebuffer_phys;
	OmapDispc->gfx_ba1 = 0;
	OmapDispc->gfx_position = 0;
	OmapDispc->gfx_row_inc = 1;
	OmapDispc->gfx_pixel_inc = 1;
	OmapDispc->gfx_window_skip = 0;
	OmapDispc->gfx_size = vs;
	OmapDispc->gfx_attributes = 0x91;
    
    /* Enable the display */
    *((volatile unsigned long*)(&OmapDispc->control)) |= 1 | (1 << 1) | (1 << 5) | (1 << 6) | (1 << 15) | (1 << 16);

    /* Hook to our framebuffer */
    PE_state.video.v_baseAddr = (unsigned long)framebuffer_phys;
    PE_state.video.v_rowBytes = lcd_width * 4;
    PE_state.video.v_width = lcd_width;
    PE_state.video.v_height = lcd_height;
    PE_state.video.v_depth = 4 * (8);   // 32bpp
    
    kprintf(KPRINTF_PREFIX "framebuffer initialized\n");
    
    initialize_screen((void*)&PE_state.video, kPEEnableScreen);
    return;
}
{% endhighlight %}

Always follow the SoC TRM for initializing displays and clocks. You may wreck something on accident. :)

Serial Initialization
=====================

Another core part which I did not mention earlier is Serial UART. UART initialization is done as early as possible to 
allow debugging of the kernel. Usually, the bootloader configures the baud rate, but you can always reinitialize it
to whatever you want.

Serial_putc and Serial_getc functions are also available. 

On S5L8930X, the serial interface is provided by standard S3C241x UARTs. It's always very simple to send bits to the uart.

{% highlight c %}
void S5L8930X_putc(int c)
{
    /* Wait for FIFO queue to empty. */
    while(HwReg(gS5L8930XUartBase + UFSTAT) & UART_UFSTAT_TXFIFO_FULL)
        barrier();

    HwReg(gS5L8930XUartBase + UTXH) = c;    
    return;
}
{% endhighlight %}
