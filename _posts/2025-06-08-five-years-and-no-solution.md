---
title: "Five Years and No Solution"
description: Revisiting an old forum post
date: 2025-06-08 00:00:00 -0400
categories: [Programming, Virtualization, ARM, Xilinx, AMD, AI]
tags: [programming, virtualization, compilers, ai, forums]
---

While logging into work early in the morning last week, I came across an
interesting email that hit my inbox from the AMD SoC/FPGA support forum.
In the email was someone asking if I had ever solved an issue that I ran into
about five years ago. Quick spoiler alert - I did not. After resetting my
password after long forgetting it, I jumped in to reply stating that I,
unfortunately, did not ever figure out a true solution to the problem, but I
did extend some advice I learned as I was debugging the issue.

For anyone curious, you can see the issue
[here](https://adaptivesupport.amd.com/s/question/0D52E00006hpX90SAE/mouse-and-keyboard-inputs-in-qemu-with-zynqmpsoc-zcu106?language=en_US).

In today's post, we'll be diving into this issue a bit more, setting up the
problem space (as best as I can talk about it given that it is a work-related
topic, after all), and peering into what the future might look like as the
world rapidly shifts thanks to our friendly new Clippy-esque companion - AI.

## The Task and Background

The company I work for specializes in building out software solutions for
embedded platforms, mostly focusing on the defense side of the house. Our
main specialty is building custom Linux-based operating systems for embedded
devices. This is probably of no surprise if you have read my previous blog
posts where we dive into the internals of Yocto and how it works. Essentially,
we do the same thing for major corporations that want to fully control their
operating system and the experience the end-user receives.

While working with a customer, we were tasked to build out a custom variant of
AMD's
[PetaLinux](https://www.amd.com/en/products/software/adaptive-socs-and-fpgas/embedded-software/petalinux-sdk.html).
This is a front end that AMD (previously Xilinx) created to help make working
with Yocto-based systems easier on the developer who might not be as
well-versed in the Yocto ecosystem as a normal Linux BSP developer would.
PetaLinux is nothing more than a simple front end to managing a Yocto-built
operating system for whatever AMD/Xilinx BSP you happen to be targeting.

Not all of our developers have access to secure labs where we have target
hardware readily available for testing. Those of us who are working remote also
need to be able to write software against this, so we needed to come up with a
solution for those devs (aka me) so we can all work on this task of building up
a custom OS for the customer together. The solution comes in the form of our
emulation friend [QEMU](https://www.qemu.org/).

### QEMU Intro

QEMU (Quick EMUlator) is an open-source machine emulator and virtualizer that
has become the go to standard of the virtualization and embedded development
world. It was originally written by the famous French programmer
[Fabrice Bellard](https://bellard.org/) in 2003. If you haven't heard of him, I
highly recommend reading what he has worked on because there's a good chance
you have either used his tech, or used a program that leveraged software he
invented. Thanks to its flexibility, speed, and ease of use, QEMU quickly
gained popularity and started obtaining broad hardware support, making it the
de facto standard for virtualization.

At its core, QEMU allows users to emulate various hardware platforms. This is
everything that a board manufacturer might happen to support, from x86 and ARM,
to PowerPC, MIPS, and more. This makes it possible to run operating systems and
applications designed for one architecture on a completely different one.

You will find QEMU used all over the place, too. For example, it is used as the
underlying engine for numerous higher-level virtualization tools, including
[KVM](https://www.redhat.com/en/topics/virtualization/what-is-KVM),
[libvirt](https://libvirt.org/), and a plethora of cloud infrastructure
platforms. It's also the go-to tool for embedded system developers, CI/CD
testing pipelines, operating system maintainers, and anyone needing to simulate
hardware they don't physically own.

As you probably guessed, this means QEMU is the perfect solution to our problem
that we were facing at our company. So what went wrong? Before getting into that,
I first should showcase common uses and the problem I was running into. So
first, let's take a mini lesson on how we would use QEMU in a practical manner.

### QEMU Mini Lesson

QEMU has two main modes:

* Full system emulation: QEMU can simulate a complete machine, including CPU,
                         memory, disk, and network interfaces. This enables
                         running entire guest operating systems in a sandboxed
                         environment.

* User-mode emulation: QEMU can execute single programs compiled for one
                       architecture on another, translating system calls and
                       CPU instructions as needed.

For our use case, we will be focusing on full system emulation because we want
to be able to emulate a piece of hardware that not all of us have access to.
We are, after all, wanting to be able to load our custom Linux on it and take
control of everything.

First, you want to specify what kind of hardware you want to emulate. This is
more than just the CPU architecture or a simple distro select menu that you
tend to see on most virtualization platforms. We need to be explicit about RAM,
storage, USB devices, and so on. Each one of these is defined by passing
options to QEMU via the command line.

Basic Command Structure:

```sh
qemu-system-ARCH [machine/board options] \
                 [CPU/ram/dtb options] \
                 [kernel/u-boot/image options] \
                 [device options] \
                 [drive/storage/network options] \
                 [other flags]
```

* `qemu-system-ARCH` - Specifies the architecture to emulate
                       (e.g., `qemu-system-x86_64` for a 64-bit PC,
                       `qemu-system-aarch64` for ARM64, etc.)

* Machine/board options - Tell QEMU what kind of board or machine to emulate

* CPU/ram/dtb options - Lets you choose CPU type, RAM amount, and provide a
                        device tree, if applicable

* Kernel/U-Boot/image options - Point QEMU at the software images you want to
                                run (e.g., kernel, device loader, file, etc.)

* Device options - Add devices like USB, network cards, or display hardware

* Drive/storage/network options - Control how disks and network interfaces
                                  should be configured for the board

* Other flags options - Things like display settings, GDB debugging, etc.

### ARM Mini Lesson

ARM (Advanced RISC Machine) processors are at the heart of most modern embedded
systems and SoCs. Unlike x86 processors found in most desktops and laptops, ARM
CPUs are designed with efficiency and low power consumption in mind, making them
ideal for everything from smartphones to FPGAs and development boards. Recently,
they have gotten so powerful that major manufacturers like Apple are even using
them for their desktops and laptops.

In the Xilinx world, the Zynq UltraScale+ MPSoC (Multi-Processor
System-on-Chip) family combines programmable logic with powerful ARM Cortex-A53
application processors. Developing for these chips often means targeting
unique hardware setups, custom peripherals, and deeply tuned Linux environments
to propagate all these features to userland.

Now that we know roughly how QEMU works, let's walk through a stripped-down
version of the kind of command you might use to bring up an ARM-based board:

```sh
qemu-system-aarch64 \
    -M virt \
    -cpu cortex-a53 \
    -m 2048 \
    -kernel Image \
    -dtb board.dtb \
    -drive file=rootfs.img,if=none,format=raw,id=hd0 \
    -device virtio-blk-device,drive=hd0 \
    -append "root=/dev/vda console=ttyAMA0" \
    -serial mon:stdio \
    -nographic
```

Explanation of commands:

* `-M virt` - Tells QEMU to use a generic ARM "virt" board

* `-cpu cortex-a53` - Sets the CPU model, matching real-world ARM equivalent

* `-m 2048` - Gives the guest 2GB of RAM to work with

* `-kernel Image` - Loads the Linux kernel image

* `-dtb board.dtb` - Loads a device tree describing the virtual hardware

* `-drive file=rootfs.img,if=none,format=raw,id=hd0` - Connect a raw root
                                                       filesystem image to the
                                                       VM at hd0

* `-append "root=/dev/vda console=ttyAMA0"` - Passes kernel command-line
                                              args telling Linux where its
                                              root filesystem is and which
                                              console to use

* `-serial mon:stdio` - Ties the guest's serial console to your terminal

* `-nographic` - Disables the graphical window so everything is in the terminal

## The Problem

At my company, we wanted to run and interact with our custom Linux images for
the ZynqMP ZCU106 board using QEMU, allowing multiple developers to test and
validate their work without requiring access to hardware. The idea was simple:
boot our custom Linux in QEMU, and use it just like the real thing, including
interacting with the VM using a mouse and keyboard, just as we would with a
real development board that had a keyboard, monitor, mouse, etc.

However, it wasn't quite that simple...

Despite QEMU's reputation for broad hardware support, I hit a wall when it came
to accessing peripherals in the QEMU environment. No matter what options or
device flags I used, QEMU refused to recognize any mouse or keyboard input.
I tried a variety of USB controller emulations (`qemu-xhci`, `usb-ehci`,
`usb-uhci`, and more), as well as different HID device options, cycling through
every reasonable combination the documentation recommended. I swapped out real
hardware, thinking maybe it might have been my devices themselves. Different
mice, keyboards, USB ports, laptops were all tested. Still, QEMU's guest OS sat
there never registering any of my USB devices.

For my specific use case, it wasn't a true showstopper. I could offload any of
that testing to my wonderful and very patient on-site tester. Still, it was
frustrating having to essentially throw test code over the wall to someone else
when QEMU should be able to handle such a simple use case.

One subtle wrinkle in embedded development is that companies like Xilinx often
maintain their own vendor forks of tools like QEMU. These forks include custom
patches, board support, and device models that are not yet available (or
sometimes never make it) in the official, "upstream" QEMU project.

In my case, the official upstream QEMU had very limited support for the Zynq
UltraScale+ MPSoC family that I was targeting. To emulate the board's unique
hardware and boot flows, you were required to use the version of QEMU provided
by Xilinx—often distributed as part of the PetaLinux SDK.

Here's the catch:

The Xilinx fork of QEMU lagged behind the upstream project in features, device
emulation, and bug fixes. Some features that worked in mainstream QEMU, such as
certain USB controllers or advanced input options, simply weren't present
enabled, or just didn't work as expected in the vendor fork.

Documentation from Xilinx sometimes referenced features that didn't actually
exist in the shipped binaries, or the functionality wasn't working as expected.

On top of that, if you strayed from the "golden path" like we were, you were
pretty much on your own.

## The Solution

The truth is, I never fully solved the problem; we only had a workaround. After
years of silence, when someone finally reached out to resurrect my old support
thread, I realized just how few details I had remembered about the scenario,
as well as what exactly went down to workaround this. Five years is a long time
in tech, and in the case of Xilinx's QEMU fork, things may have shifted under
the hood more than once since then. Hell, Xilinx was bought out by AMD after I
reached out for support in the forums.

To help out the fellow developer also struggling with the same problem, I threw
in my two cents, hoping it would lead to some sort of avenue to help them solve
their problem. It was generic advice, but it boiled down to:

* Double-check your device tree: Any USB controllers or input devices you
  configure in QEMU must actually exist in your device tree. If you're using a
  `.dtb` file, convert it to a `.dts` with `dtc` and look for xhci or usb nodes.

* Be explicit with device mappings: If you're getting errors about missing USB
  buses, try explicitly assigning the device to a known bus. For example:
  `-device usb-mouse,bus=xhci.0`

* Use the QEMU monitor: The built-in QEMU monitor can help you inspect what
  devices are visible to the emulated guest which is useful for debugging why
  input isn't working. Attach it with `-serial mon:stdio` (as shown above) and
  issue monitor commands.

* Verify kernel configuration: Sometimes it's not QEMU at all. Make sure your
  guest kernel is configured with support for the right USB controllers and HID
  devices.

## The Future

This got me thinking about today's world where everything is shifting to AI.
When I wrote this originally, there were no public AI models for anybody to
use. It was pre-AI revolution. Stack Overflow was still alive and well.
Google still worked (mostly) as expected. You could find solutions on the
Internet via forums, mailing lists, and more.

This is rapidly changing.

Today, people have AI to turn to. It's a void you type into, attempt to solve
problems with, and hope whatever it returns back to you works. It's an endless
feedback loop where you never know if you're going to get a correct answer.
Sure, you'll get an answer, but it'll be difficult to verify if it's correct
or not.

However, there's an even more interesting aspect about this that doesn't seem
to be discussed: the community banding together to solve issues. If
questions are never posted publicly, people might not know others are going
through similar issues, causing them to pursue a path that will surely lead
them to a roadblock. Manufacturers might not know there are issues with
their custom implementations. After all, many bug reports out there come about
from users attempting to utilize the components and them not behaving as
expected. In short, people can't learn from other real people.

Forums are already in decline thanks to closed gardens like Discord, Facebook,
and more. It makes me wonder what the future will look like when everyone has
their own little AI version of Clippy, and nobody is reaching out to others to
see if a solution is readily in front of them just a simple search away...

I'm sure someone could come up with scenarios where such support groups are no
longer needed, as well as counter arguments for everything stated above, but
the old grey beard in me remembers what life was like in the past and worries
what life might be like in the future...

## Further Reading

Some useful links on virtualization and emulation:

* [QEMU Master Documentation](https://www.qemu.org/docs/master/)
* [Configuring and Managing Virtualization](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_and_managing_virtualization/index)
* [libvirt - Ubuntu Server Documentation](https://documentation.ubuntu.com/server/how-to/virtualisation/libvirt/index.html)

Happy virtualizing!
