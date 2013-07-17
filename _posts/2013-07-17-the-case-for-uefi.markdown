---
layout: post
title:  "The case for an EFI based bootloader on ARM"
date:   2013-07-17 11:07:00
categories: xnu projects
---

Let's start off initially with what EFI or UEFI even is. EFI provides a 
firmware environment with core services and API programs can use for 
booting and additional services the operating system can use for 
maintaining platform data such as NVRAM variable storage. EFI's boot 
services give the user ways to perform things that are normally done by 
hand in the bootloader stage, such as printing text to a console, or 
loading a file off of a volume. You can find the EFI Development Kit II 
source over here on [Tianocore's GitHub](https://github.com/tianocore/edk2).

U-boot on ARM is very similar, but it doesn't seem to provide a rich 
execution environment for programs. The data has to all be loaded in 
memory by the bootloader during core startup. This isn't a very flexible 
design.

# But what about XNU?

I made an EFI booter for ARM XNU because I want flexibility. Having a 
kernel simply loading off TFTP and using U-Boot's/Linux's ATAG 
functionality is nice for creating a core memory map, parsing command 
lines and also for sending ramdisks to the kernel, but it isn't enough. 
The bootloader needs to be recompiled for each board to provide debug 
output over serial UART.

This gets annoying very quickly, as a new UART driver would need to be 
made every time a new board configuration is added.

# Enter UEFI

UEFI provides core services such as `Print`, `ZeroMem` and so on. But, 
those are only the beginning. Enter the EFI boot services table.

{% highlight c %}
typedef struct {
  EFI_TABLE_HEADER                           Hdr;
  EFI_RAISE_TPL                              RaiseTPL;
  EFI_RESTORE_TPL                            RestoreTPL; 
  EFI_ALLOCATE_PAGES                         AllocatePages; 
  EFI_FREE_PAGES                             FreePages; 
  EFI_GET_MEMORY_MAP                         GetMemoryMap; 
  EFI_ALLOCATE_POOL                          AllocatePool; 
  EFI_FREE_POOL                              FreePool; 
  EFI_CREATE_EVENT                           CreateEvent; 
  EFI_SET_TIMER                              SetTimer; 
  EFI_WAIT_FOR_EVENT                         WaitForEvent; 
  EFI_SIGNAL_EVENT                           SignalEvent; 
  EFI_CLOSE_EVENT                            CloseEvent; 
  EFI_CHECK_EVENT                            CheckEvent; 
  EFI_INSTALL_PROTOCOL_INTERFACE             InstallProtocolInterface; 
  EFI_REINSTALL_PROTOCOL_INTERFACE           ReinstallProtocolInterface; 
  EFI_UNINSTALL_PROTOCOL_INTERFACE           UninstallProtocolInterface; 
  EFI_HANDLE_PROTOCOL                        HandleProtocol; 
  VOID*                                      Reserved; 
  EFI_REGISTER_PROTOCOL_NOTIFY               RegisterProtocolNotify; 
  EFI_LOCATE_HANDLE                          LocateHandle; 
  EFI_LOCATE_DEVICE_PATH                     LocateDevicePath; 
  EFI_INSTALL_CONFIGURATION_TABLE            InstallConfigurationTable; 
  EFI_IMAGE_LOAD                             LoadImage; 
  EFI_IMAGE_START                            StartImage; 
  EFI_EXIT                                   Exit; 
  EFI_IMAGE_UNLOAD                           UnloadImage; 
  EFI_EXIT_BOOT_SERVICES                     ExitBootServices; 
  EFI_GET_NEXT_MONOTONIC_COUNT               GetNextMonotonicCount; 
  EFI_STALL                                  Stall; 
  EFI_SET_WATCHDOG_TIMER                     SetWatchdogTimer; 
  EFI_CONNECT_CONTROLLER                     ConnectController; 
  EFI_DISCONNECT_CONTROLLER                  DisconnectController;
  EFI_OPEN_PROTOCOL                          OpenProtocol; 
  EFI_CLOSE_PROTOCOL                         CloseProtocol; 
  EFI_OPEN_PROTOCOL_INFORMATION              OpenProtocolInformation; 
  EFI_PROTOCOLS_PER_HANDLE                   ProtocolsPerHandle; 
  EFI_LOCATE_HANDLE_BUFFER                   LocateHandleBuffer; 
  EFI_LOCATE_PROTOCOL                        LocateProtocol; 
  EFI_INSTALL_MULTIPLE_PROTOCOL_INTERFACES   InstallMultipleProtocolInterfaces; 
  EFI_UNINSTALL_MULTIPLE_PROTOCOL_INTERFACES UninstallMultipleProtocolInterfaces; 
  EFI_CALCULATE_CRC32                        CalculateCrc32; 
  EFI_COPY_MEM                               CopyMem; 
  EFI_SET_MEM                                SetMem;
  EFI_CREATE_EVENT_EX                        CreateEventEx;
} EFI_BOOT_SERVICES;
{% endhighlight %}

That's a lot of boot services. In EFI, everything is handled by specific 
protocols, which are installed by various drivers. These provide rich 
functionality for the user programs. For example, the 
`SimpleFileSystem` protocol allows the user to open volumes without 
having to rewrite platform code to open eMMC, then parse the partition 
tables, and finally then parsing the filesystem.

# Loading the mach_kernel.

Although EFI provides a rich firmware environment, its programming paradigm
can be awfully over the top at times.

{% highlight c %}
    /* Load the kernel. */
    Print(L" * Loading \"%s\"...", kMachKernelName);

    Status = gBS->LocateProtocol (&gEfiSimpleFileSystemProtocolGuid, NULL, (VOID **)&FsProtocol);
    if(Status != EFI_SUCCESS)
        return EFI_LOAD_ERROR;

    /* Try to open the volume and get the root directory. */
    Status = FsProtocol->OpenVolume (FsProtocol, &Fs);
    if(Status != EFI_SUCCESS)
        return EFI_LOAD_ERROR;

    File = NULL;
    Status = Fs->Open (Fs, &File, kMachKernelName, EFI_FILE_MODE_READ, 0);
    if(Status != EFI_SUCCESS)
        return EFI_LOAD_ERROR;

    Size = 0;
    File->GetInfo(File, &gEfiFileInfoGuid, &Size, NULL);
    FileInfo = AllocatePool (Size);
    Status = File->GetInfo(File, &gEfiFileInfoGuid, &Size, FileInfo);
    if(Status != EFI_SUCCESS)
        return EFI_LOAD_ERROR;

    /* Get the file size. */
    Size = FileInfo->FileSize;
    FreePool(FileInfo);

    Status = gBS->AllocatePages(Type, EfiBootServicesCode, EFI_SIZE_TO_PAGES(Size), Image);
    if ((Status == EFI_OUT_OF_RESOURCES) && (Type != AllocateAnyPages)) {
        Status = gBS->AllocatePages(AllocateAnyPages, EfiBootServicesCode, EFI_SIZE_TO_PAGES(Size), Image);
    }
    if (!EFI_ERROR (Status)) {
        Status = File->Read (File, &Size, (VOID*)(UINTN)(*Image));
    }
{% endhighlight %}

This is overly complex, but still, not as complex as having to do it all
from a core loader. People may ask, "why not just integrate everything into
u-boot"? The answer to this is to be board agnostic. If I can just place my
`boot.efi` file on a SD card for one board and I don't have to recompile the
firmware entirely, it makes my day go better.

# Loading Other Files

Darwin on ARM, at least my version, is intended to act a lot like desktop Mac OS X,
that is, it will have a proper HFS root file system with a `/System/Library/Extensions`
folder. The kernel will not be prelinked as per iOS, but will instead be a standalone
binary in the root of the filesystem. On u-boot, this isn't really possible without
extending the functionality built in to the bootloader, and I don't really want to
recompile u-boot for every single platform either.

EFI, as I stated before, provides a way to load files off block devices with
recognized file systems, albeit a very cumbersome one. Loading kexts and parsing them
is a simple task to do in this environment.

# Enter the Matrix

Now, passing control from EFI to the actual OS? That's simple. By calling `GetMemoryMap`
and then `ExitBootServices`, the core of EFI is effectively jettisoned out, and now the
OS is free to control mainly all of memory. 

I pass control to Darwin by copying the kernel to where it should be, then doing core
ARM initialization, and then finally using a `bx lr` into the kernel entrypoint.

After that, the operating system has to do everything.
