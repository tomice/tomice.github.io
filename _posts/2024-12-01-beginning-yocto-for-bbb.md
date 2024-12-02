---
title: Building a BeagleBone Black Image with Yocto
description: A gentle introduction to Yocto using the BeagleBone Black
date: 2024-12-01 00:00:00 -0500
categories: [Programming, Shell Scripting, Linux, Embedded Linux, Yocto]
tags: [linux, embedded linux, scripting]
---

It can be frustrating to search for [Yocto](https://www.yoctoproject.org/)
tutorials when you're just getting started in the embedded Linux field.
Many tutorials take you through steps that aren't fully explained,
are overly complicated, follow custom workflows, or, in some cases,
are simply incorrect.

Even the tutorial on the
[official BeagleBoard website](https://www.beagleboard.org/projects/yocto-on-beaglebone-black)
for building an image with Yocto is severely outdated and can set developers
up for failure when they are just trying to build a basic image.

In this tutorial, we'll show you how to build a simple image with Yocto,
without relying on external dependencies. The resulting image will boot to a
basic command-line interface that you can explore and use as a foundation to
further develop your project.

## What is Yocto?

Simply put, Yocto is an open-source toolset and framework for creating custom
Linux distributions. It was created back in 2010 by the Linux Foundation as a
way to standardize the development process across different hardware platforms.

Yocto allows developers to tailor a Linux distribution to their specific needs,
whether it's for embedded devices, IoT projects, or other specialized use cases.
Admittedly, if you've reached this blog post, you probably already have a decent
idea of what Yocto is, so we'll focus on highlighting the main parts.

### Components

#### Poky

Poky can be thought of as the distro. It is the general reference distribution,
which includes the OpenEmbedded build system, metadata for the core packages,
and recipes that contain instructions on where to grab source from and how to
build it.

Poky serves as a reference implementation and provides a base layer, but it is
not typically used directly as a distribution. Instead, it's a starting point
from which you can customize and build your own Linux distribution.

#### OpenEmbedded

The OpenEmbedded build system is at the heart of Yocto and is responsible for
compiling and building the complete system. It's designed to make the process
of cross-compiling software for different target devices easier.
This build system is highly customizable and allows you to create tailored
Linux distributions for a wide variety of embedded systems.

#### BitBake

BitBake is the task executor and scheduler for building. It reads the metadata
and executes tasks to build a specified target(s), from kernel modules to
user-space applications. When you're interacting with the build system,
you're generally interacting with BitBake.

#### Layers

Layers are a collection of recipes, configuration files, and other components,
organized in a way that allows them to be shared and reused across projects.
A Yocto build system is typically composed of multiple layers,
including core layers, board-specific layers, and custom layers.

These layers allow for easy customization of software packages and configuration
options for different hardware platforms. Board vendors often create layers for
your Yocto-based project to consume, and you can reference those layers for any
board-specific customizations. Layers typically have `meta-` prefaced to their
names, such as the [`meta-raspberrypi`](https://layers.openembedded.org/layerindex/branch/master/layer/meta-raspberrypi/)
layer.

## Why Yocto?

There are many ways to build Linux for embedded platforms.
Buildroot, Linux From Scratch, Yocto, and more are just a few options available.
However, Yocto stands out as the de facto standard, with nearly all hardware
vendors supporting it for their custom boards.

At the same time, Yocto can be one of the most difficult systems to get started
with, due to its evolving build system that continues to change over time.
This makes it less ideal for hobbyists but more suitable for those looking to
enter the professional embedded Linux world.

Another major advantage of using Yocto is the level of control it offers over
exactly what gets placed onto the system. This is particularly useful for
developers who need tight control over every aspect of their device.

As we explore more use cases for the BeagleBone Black in future blogs,
the benefits of this control might become even more apparent. At the same time,
it might also showcase why you might not want to use Yocto for your own home
project.

## Why the BeagleBone Black?

The BeagleBone Black is a remarkable project that is 100% open source. Unlike
other embedded development boards, it provides full access to the hardware
design and software, allowing developers to customize and modify it to their
specific needs. This openness makes it an ideal choice for anyone looking to
gain a deeper understanding of embedded systems.

The BeagleBone Black is also supported by a strong and active community,
which means there's a wealth of resources, tutorials, and forums available to
help troubleshoot and collaborate.

Most importantly, compared to other embedded development boards out there,
the BeagleBone Black is well-supported in Yocto, making it an excellent choice
for those wanting to create custom Linux images with this toolset.

## Prerequisites

Before we begin, we need to make sure we have a few things ready.
Some of these may be obvious, while others might not be so much.

- **BeagleBone Black Rev C**
- **MicroSD Card**: Minimum 256 MB
- **MicroSD Card Reader**
- **A decently powerful computer to build**:
  - A Linux machine running **Ubuntu 20.04** or higher is preferred.
    Alternatively, **WSL2** (Windows Subsystem for Linux) can be used on Windows.
    A virtual machine (VM) can also work, but expect significantly longer build times.
  - **8 GB of RAM** minimum
  - **100 GB of free space** minimum
- **A means to view the booted BeagleBone Black**:
  - A keyboard, monitor, and power source (either 5V power supply or Mini USB).
    Alternatively, you can use a **serial to USB cable**. Follow this tutorial for setup:
    [How to connect the BeagleBone Black via serial over USB](https://www.dummies.com/article/technology/computers/hardware/beaglebone/how-to-connect-the-beaglebone-black-via-serial-over-usb-144981/)

## Installing Dependencies

Before we can begin building, we need to install the necessary dependencies.
Thankfully, the [documentation](https://docs.yoctoproject.org/brief-yoctoprojectqs/index.html#build-host-packages)
provides the necessary steps for Ubuntu.
However, if any dependencies are missing, BitBake will notify you and provide
the name of the missing packages.

### Ubuntu/Debian

```sh
sudo apt install gawk wget git diffstat unzip texinfo gcc build-essential \
                 chrpath socat cpio python3 python3-pip python3-pexpect \
                 xz-utils debianutils iputils-ping python3-git \
                 python3-jinja2 python3-subunit zstd liblz4-tool file \
                 locales libacl1
sudo locale-gen en_US.UTF-8
```

### Fedora/CentOS/RedHat

```sh
sudo dnf install gawk wget git diffstat unzip texinfo gcc make chrpath socat \
                 cpio python3 python3-pip python3-pexpect xz python3-git \
                 python3-jinja2 python3-subunit zstd lz4 file glibc-common
sudo localectl set-locale LANG=en_US.UTF-8
```

### Other Distributions

Refer to your distro's package manager and install the equivalent packages.

## Building

Building a reference image for a BeagleBone Black is extremely straightforward.
However, knowing which [version of Yocto](https://wiki.yoctoproject.org/wiki/Releases)
to choose can be challenging. It has been possible to build a reference image
for the BBB since at least [Sumo](https://github.com/yoctoproject/poky/blob/sumo/meta-yocto-bsp/conf/machine/beaglebone-yocto.conf),
but in the interest in keeping with a Long Term Stable reference image that has
lots of support across different projects and layers, we will use Kirkstone as
a base. Feel free to adjust this as necessary. All future versions should
generally follow this same tutorial.

Open a terminal, and type the following:

### 1. Set Up the Working Directory

First, create and navigate to a directory for your project:

```sh
mkdir ~/bbb-example
cd ~/bbb-example
```

### 2. Clone the Poky Repository

Clone the Yocto Poky repository and check out the `kirkstone` branch:

```sh
git clone git://git.yoctoproject.org/poky -b kirkstone
```

### 3. Initialize the Build Environment

Yocto requires specific environment variables to be configured.
Source the provided initialization script to set them up:

```sh
cd poky
source oe-init-build-env
```

#### What Happens When You Source `oe-init-build-env`?

- **Environment Variables:** Sets variables like `OE_COREBASE`, `BUILD_DIR`, `DL_DIR`, and `TMPDIR` to manage files and temporary data.
- **Build Directory Setup:** Creates subdirectories (e.g., `tmp/`, `conf/`) if they don't already exist.
- **Toolchain Configuration:** Prepares the environment with the necessary toolchain.
- **Layer Configuration:** Loads configurations for meta-layers containing recipes and settings.

> **Note:** If you open a new terminal, you must re-source this script to continue using Yocto.

### 4. Configure `local.conf` for BeagleBone Black

Modify the `local.conf` file to target the BeagleBone Black:

```sh
vim conf/local.conf  # Use any editor (nano, gedit, etc.)
```

#### Update the `MACHINE` Variable

Look for the `MACHINE` variable (default is `qemux86-64`) and set it to `beaglebone-yocto`:

```sh
MACHINE ?= "beaglebone-yocto"
```

This configuration ensures that Yocto:

- Selects the appropriate toolchain for the BBB.
- Applies the correct kernel and device tree configurations.
- Creates an image only for the BBB.

If the `MACHINE` variable is missing, add it manually near the end of the file.

### 5. Build the Image

Run the following command to build a reference image:

```sh
bitbake core-image-full-cmdline
```

BitBake will then kick off the build process. It will parse your recipes,
config files, and anything else in your layers.

#### Build Details

- The build process may take ~30 minutes on a native system.
  Virtual machines can take significantly longer. Don't expect to use your
  computer much during this time as it will leverage all possible resources
  in order to parallelize the build as much as possible.
- Upon completion, the `.wic` file (bootable image) will be available in:

```sh
~/bbb-example/poky/build/tmp/deploy/images/beaglebone-yocto/
```

You will see a generalized symlink to the `.wic` named:
`core-image-full-cmdline-beaglebone-yocto.wic`

The actual `.wic` will be something like:
`core-image-full-cmdline-beaglebone-yocto-20241202012729.rootfs.wic`

## Writing the WIC to a MicroSD Card

Writing the `.wic` image to a MicroSD card is relatively straightforward.

### Linux

On Linux, you can use the `dd` command to write the `.wic` image to a micro
SD card. Be **very careful** with the `dd` command, as selecting the wrong
device (e.g., your main hard drive) can result in data loss. When in doubt,
use `lsblk` to find your device.

```sh
sudo dd if=~/bbb-example/poky/build/tmp/deploy/images/beaglebone-yocto/core-image-full-cmdline-beaglebone-yocto.wic of=/dev/sdX bs=4M status=progress conv=fsync
sudo sync
sudo eject /dev/sdX
```

### Windows

On Windows, you can use [Win32 Disk Imager](https://win32diskimager.org/).

1. Download the Win32 Disk Imager and install it
2. Move your `.wic` file from the `tmp/deploy/images/beaglebone-yocto` folder
   to somewhere on your Windows machine. WSL2 should provide mount points in
   `/mnt` that are mapped to your letter drives for convinence.
   > **Note**: There is a small quirk about Win32 Disk Imager
   [failing to launch sometimes](https://stackoverflow.com/questions/63928864/win32diskimager-fails-to-launch).
3. Open Win32 Disk Imager and write the image. Make sure to choose the correct
   drive letter for your MicroSD card

## Booting

Booting the BBB with your new Yocto image is simple:

1. Insert the MicroSD card that you just wrote into the BBB.
2. Hold down the S2 button (located near the MicroSD card slot)
   to force the BBB to boot from the MicroSD card.
3. Apply power (connect the power supply or USB cable).
4. Release the S2 button after about 5 seconds.

You should see activity on the LEDs indicating that the BBB is booting.

## Logging In

Depending on the version you use, it may either log you in automatically,
or require you to login with credentials.

If prompted, use:

Username: root

Password: (leave blank)

It should then look like this:

```sh
Poky (Yocto Project Reference Distro) 4.0.23 beaglebone-yocto tty1

beaglebone-yocto login: root
root@beaglebone-yocto:~#
```

## Exploring Your Custom Distro

You can now explore your own custom Linux distribution, compare it to the
pre-built one that's on the BBB's eMMC, and start exploring the world of Yocto.

From here, you can begin customizing your image by adding packages,
modifying configurations, and more. Future blog posts will focus more on this,
but for now, you can try adding in some extra packages to this image.

Go back to the `local.conf` and add this line at the end to add `vim` and `git`:

```sh
IMAGE_INSTALL:append = " vim git" # Note the space before vim! It's important!
```

> **Note**: Some older versions may require the legacy syntax of:
> `IMAGE_INSTALL_append = " vim git"`

Now rebuild using the same `bitbake core-image-full-cmdline` command.
BitBake will be smart enough to only build what's necessary to add the
dependencies to the image, so the build will be much shorter than before.

Write the image to the MicroSD card again, and you should see these utilities
available on the new system!

## Further Reading

Some useful links that helped me when I first started out:

- [Yocto Project Quick Build](https://docs.yoctoproject.org/brief-yoctoprojectqs/index.html)
- [Yocto Mega Manual](https://docs.yoctoproject.org/singleindex.html)
- [Yocto Reference Manual](https://docs.yoctoproject.org/ref-manual/index.html)
- [Variable names and their paths](https://git.yoctoproject.org/poky/tree/meta/conf/bitbake.conf?h=kirkstone)
- [BeagleBone Black Docs](https://docs.beagle.cc/boards/beaglebone/black/index.html)

Happy building!
