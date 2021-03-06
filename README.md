# 🔮 clairvoyance
![Builds](https://github.com/0vercl0k/clairvoyance/workflows/Builds/badge.svg)

Clairvoyance (/**klɛərˈvɔɪəns**/; from French clair meaning *clear* and voyance meaning *vision*) from [Wikipedia](https://en.wikipedia.org/wiki/Clairvoyance).

<p align='center'>
<img src='pics/ida64_dmp-ph.annotated.png' width=60% alt='clairvoyance'>
</p>

## Overview

**clairvoyance** creates a colorful visualization of the page protection of an entire 64-bit process address space (user and kernel) running on a Windows 64-bit kernel.

To transform the 1 dimension space, that is the address space, into a 2 dimensions visualization, the [hilbert space-filling curve](https://en.wikipedia.org/wiki/Hilbert_curve) is used. Each colored pixel on the above picture represents the page protection (*UserRead*, *UserReadWrite*, etc.) of a 4KB page in virtual memory.

The address space is directly calculated by manually parsing the [four-level](https://en.wikipedia.org/wiki/X86-64#Virtual_address_space_details) page tables hierarchy associated with a process from a kernel crash-dump that has been generated using [WindDbg](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-download-tools).

Finally, the program program outputs a file with the metadata required to have it displayed on a two dimensional canvas as well as being able to calculate the virtual address corresponding to a specific highlighted pixel.

Compiled binaries are available in the [releases](https://github.com/0vercl0k/clairvoyance/releases) section. An online viewer is also hosted at [0vercl0k.github.io/clairvoyance](https://0vercl0k.github.io/clairvoyance).

Shouts out to:
- [Alexandru Radocea](https://twitter.com/defendtheworld) and [Georg Wicherski](https://twitter.com/ochsff) for the inspiration (see their BlackHat USA 2013 research: *[Visualizing Page Tables for Exploitation](https://media.blackhat.com/us-13/US-13-Wicherski-Hacking-like-in-the-Movies-Visualizing-Page-Tables-WP.pdf)*),
- [The Hacker's delight second edition](https://www.amazon.com/Hackers-Delight-2nd-Henry-Warren/dp/0321842685)'s chapter 16 *Hilbert's curve* for providing the algorithms used.

## Usage

To generate the kernel crash dump it is recommended to use [WinDbg](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-download-tools), [KDNet](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/setting-up-a-network-debugging-connection-automatically) with the [.dump /f](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-dump--create-dump-file-) command.

Once the dump has been acquired you can pass its path to clairvoyance as well as the physical address of the page directory you are interested in:

```
./clairvoyance <dump path> [<page dir pa>]
```

This generates a file with the *clairvoyance* extension that you then can visualize in your browser at [0vercl0k.github.io/clairvoyance](https://0vercl0k.github.io/clairvoyance) or by checking out the [gh-pages](https://github.com/0vercl0k/clairvoyance/tree/gh-pages) branch which is where the viewer is hosted at.


<p align='center'>
<img src='pics/clairvoyance.gif' width=100% alt='clairvoyance'>
</p>

## Build

The [CI](https://github.com/0vercl0k/clairvoyance/blob/main/.github/workflows/clairvoyance.yml) builds clairvoyance on Linux using [clang++-11](https://clang.llvm.org/) and on Windows using Microsoft's [Visual studio 2019](https://visualstudio.microsoft.com/vs/community/).

To build it yourself you can use the scripts in [build/](https://github.com/0vercl0k/clairvoyance/blob/main/build):

```
(base) clairvoyance\build>build-msvc.bat
(base) clairvoyance\build>cmake ..
-- Selecting Windows SDK version 10.0.19041.0 to target Windows 10.0.19042.
-- Configuring done
-- Generating done
-- Build files have been written to: clairvoyance/build

(base) clairvoyance\build>cmake --build . --config RelWithDebInfo
Microsoft (R) Build Engine version 16.8.2+25e4d540b for .NET Framework
Copyright (C) Microsoft Corporation. All rights reserved.

  clairvoyance.vcxproj -> clairvoyance\build\RelWithDebInfo\clairvoyance.exe
  Building Custom Rule clairvoyance/CMakeLists.txt
```

## Various findings

The below are things I've noticed on a kernel crash-dump generated from an Hyper-V VM of Windows:

```
kd> vertarget
Windows 10 Kernel Version 18362 UP Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 18362.1.amd64fre.19h1_release.190318-1202
Machine Name:
Kernel base = 0xfffff805`36800000 PsLoadedModuleList = 0xfffff805`36c432f0
Debug session time: Sat Jul 25 10:00:19.637 2020 (UTC - 8:00)
System Uptime: 0 days 0:18:53.609
```

### Type of pages

Windows doesn't seem to be using huge pages (1GB) or at least I have not seen one being used in any of the dumps I collected.

Large pages are used in abundance to map some kernel executables like the Windows kernel *nt* for example:

```
kd> ? nt
Evaluate expression: -8773703827456 = fffff805`36800000
```

```
VA:0xfffff80536800000, PA:0x2400000 (KernelReadWriteExec, Large, PML4E:0xd5745f80, PDPTE:0x42080a0, PDE:0x4209da0, PTE:0x0)
```

There are also a bunch of kernel read, write, executable pages that are not large pages, which was somewhat a surprise. I was aware that the kernel / hal
could be mapped using large pages and that those were *krwx*. The reason for that is that 2MB is so large that it spans both executable and data sections; meaning the page has to be writeable and executable.

The only public mention of this I could find is in this [blogpost](https://nadav.amit.zone/windows/2018/09/15/windows-pti.html) (thx [`Ivan](https://twitter.com/ivanlef0u)):


> I contacted Microsoft which claimed that this is intended since “in some cases the kernel is mapped with large pages” and that this can be prevented by enabling virtualization based protection (VBS).

### Virtual address sinks

A bunch of large kernel memory sections are mapped against the same physical page (filled with zero):

```
VA:0xffffc27ef4401000, PA:0x4200000 (KernelRead, Normal, ...)
VA:0xffffc27ef4402000, PA:0x4200000 (KernelRead, Normal, ...)
VA:0xffffc27ef4403000, PA:0x4200000 (KernelRead, Normal, ...)
...
VA:0xffffc27ef63fb000, PA:0x4200000 (KernelRead, Normal, ...)
VA:0xffffc27ef63fc000, PA:0x4200000 (KernelRead, Normal, ...)
VA:0xffffc27ef63fd000, PA:0x4200000 (KernelRead, Normal, ...)
VA:0xffffc27ef63fe000, PA:0x4200000 (KernelRead, Normal, ...)
VA:0xffffc27ef63ff000, PA:0x4200000 (KernelRead, Normal, ...)
```

Here is smaller one (the region is not completely contiguous, there are a few holes):

```
VA:0xffffc27ed2201000, PA:0x4300000 (KernelRead, Normal, ...)
VA:0xffffc27ed2202000, PA:0x4300000 (KernelRead, Normal, ...)
VA:0xffffc27ed2203000, PA:0x4300000 (KernelRead, Normal, ...)
...
VA:0xffffc27ed25fc000, PA:0x4300000 (KernelRead, Normal, ...)
VA:0xffffc27ed25fd000, PA:0x4300000 (KernelRead, Normal, ...)
VA:0xffffc27ed25fe000, PA:0x4300000 (KernelRead, Normal, ...)
VA:0xffffc27ed25ff000, PA:0x4300000 (KernelRead, Normal, ...)
```

### Gallery of patterns

This is just a section showing off some of the cool patterns you can see in some regions of an address space.

#### Page heap

Page heap allocations and their guard pages are pretty cool looking and easy to spot:

<p align='center'>
<img src='pics/ph.png' width=80% alt='ph'>
</p>

#### Kernel stacks

Kernel stacks also have a nice recognizable shape because of their size and guard pages:

<p align='center'>
<img src='pics/kstack.png' width=85% alt='kstack'>
</p>

### System cache

The system cache region in the kernel seems to be looking like a nebula in the dumps I have seen:

<p align='center'>
<img src='pics/systemcache.png' width=90% alt='systemcache'>
</p>

## Authors

Axel '[0vercl0k](https://twitter.com/0vercl0k)' Souchet
