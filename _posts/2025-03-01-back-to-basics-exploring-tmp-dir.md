---
title: "Back to Basics: Exploring the tmp Directory"
description: Diving into the tmp directory in Yocto
date: 2025-03-01 00:00:00 -0500
categories: [Programming, Shell Scripting, Linux, Embedded Linux, Yocto]
tags: [linux, embedded linux, scripting]
---

[Last blog](https://tomice.github.io/posts/back-to-basics-folder-structure-deep-dive/)
was looking into the overall folder structure of a Yocto-based Linux build.
Towards the end, we got to the tmp directory and stopped to give people a break
from the Yocto information firehose.

This time, we'll jump straight into the tmp directory and check out exactly
what is going on inside. This is the folder where you will be spending most of
your time debugging, so it's important we take a look at every folder, as well
as different files that get placed in here. Understanding this will vastly
increase productivity and reduce headaches while working with Yocto.

## tmp at High Level

Just like before, we should level-set what the tmp directory structure looks
like. Because this folder is so large, we'll take it one directory at a time
and continue to drill deeper in each section. First, a high level look:

```sh
tmp/
├── abi_version
├── buildstats/
├── cache/
├── deploy/
├── hosttools/
├── log/
├── pkgdata/
├── saved_tmpdir
├── sstate-control/
├── stamps/
├── sysroots/
├── sysroots-components/
├── sysroots-uninative/
├── work/
└── work-shared/
```

## abi_version

This is a simple human-readable plain text file that verifies the ABI
compatibility of libraries and executables. BitBake automatically generates
this and references it as necessary. Generally, you can ignore this file unless
you are having some odd binary compatibility issues or major toolchain changes.

## buildstats

```sh
buildstats/
├── 20241210005044
│   ├── acl-2.3.1-r0
│   │   ├── do_compile
│   │   ├── do_compile_ptest_base
│   │   ├── do_configure
│   │   ├── do_configure_ptest_base
│   │   ├── do_deploy_source_date_epoch
│   │   ├── do_fetch
│   │   ├── do_install
│   │   ├── do_install_ptest_base
│   │   ├── do_package
│   │   ├── do_package_qa
│   │   ├── do_package_write_rpm
│   │   ├── do_packagedata
│   │   ├── do_patch
│   │   ├── do_populate_lic
│   │   ├── do_populate_sysroot
│   │   ├── do_prepare_recipe_sysroot
│   │   └── do_unpack
│   ├── acl-native-2.3.1-r0
│   │   ├── do_compile
│   │   ├── do_configure
│   │   ├── do_deploy_source_date_epoch
│   │   ├── do_fetch
│   │   ├── do_install
│   │   ├── do_patch
│   │   ├── do_populate_lic
│   │   ├── do_populate_sysroot
│   │   ├── do_prepare_recipe_sysroot
│   │   └── do_unpack
└── 20241215050356
    └── build_stats
```

The buildstats folder in BitBake contains detailed information about the build
process itself, and it can help identify any bottlenecks that might be
occurring.

### Date/Time Folder Structure

Each folder within the buildstats directory represents a specific build run and
is named based on the timestamp of the build. The timestamp is usually in a
format like `YYYYMMDDHHMMSS` or a similar date/time format.

For example, you might see folders like:

```sh
20250223143005/ # 2025-02-23 14:30:05
20250224101230/ # 2025-02-24 10:12:30
```

Each of these folders contains subdirectories that represent the package that
was being built, as well as statistics related to each task that was executed
at the corresponding date and time. This makes it easy to track changes in
build performance across different runs.

### Package Contents

Within each package directory (like `acl-2.3.1-r0`), there will be files named
after the tasks that were executed as part of the build for that package.

These tasks include:

- `do_compile`
- `do_fetch`
- `do_install`
- `do_package`
- `do_configure`
- etc.

Each of these files represents a log files that tracks the performance and
resource usage of the respective task during execution:

```sh
Event: TaskStarted
Started: 1733792620.61
acl-2.3.1-r0: do_compile
Elapsed time: 1.22 seconds
utime: 6
stime: 1
cutime: 346
cstime: 73
(...snipped for brevity...)
Status: PASSED
Ended: 1733792621.83
```

If you ever need to build in hooks regarding performance and identifying
bottlenecks, this is the main directory you want to look inside of. Otherwise,
you most likely won't interact with this folder too much.

## cache

```sh
cache/
└── default-glibc
    └── beaglebone-yocto
        └── x86_64
            ├── bb_cache.dat -> bb_cache.dat.12b2f8286d76abf22e7ebff253c10085840d82736babd3d163b37fe5a46877d5
            ├── bb_cache.dat.12b2f8286d76abf22e7ebff253c10085840d82736babd3d163b37fe5a46877d5
            └── bb_cache.dat.fde49510fbe509875f543f39627b08b24a11dc3d758ea2f273613824feca26a7
```

The cache directory in a BitBake build environment is primarily used to store
cached data that can help speed up subsequent builds. This cache is critical
for optimizing build times and ensuring that tasks don’t need to be re-executed
unnecessarily, especially when the inputs haven’t changed.

In this, you'll find information about your host, toolchain, and the target
machine you are attempting to build. There exist `.dat` files that BitBake
first creates after a build. It then references these to verify if any task
that is about to be executed has already been previously run and stored.

Again, you most likely won't need to reference this folder too much in your
Yocto journey, but it is nice to know what is going on and how the advanced
caching mechanisms work in BitBake.

## deploy

```sh
deploy/
├── images
│   └── beaglebone-yocto
│       ├── MLO -> MLO-beaglebone-yocto-2022.01-r0
│       ├── MLO-beaglebone-yocto -> MLO-beaglebone-yocto-2022.01-r0
│       ├── MLO-beaglebone-yocto-2022.01-r0
│       ├── am335x-bone--5.15.150+git0+567f0adb9d_4fca0c4373-r0-beaglebone-yocto-20241210021028.dtb
│       ├── am335x-bone-beaglebone-yocto.dtb -> am335x-bone--5.15.150+git0+567f0adb9d_4fca0c4373-r0-beaglebone-yocto-20241210021028.dtb
│       ├── am335x-bone.dtb -> am335x-bone--5.15.150+git0+567f0adb9d_4fca0c4373-r0-beaglebone-yocto-20241210021028.dtb
│       ├── am335x-boneblack--5.15.150+git0+567f0adb9d_4fca0c4373-r0-beaglebone-yocto-20241210021028.dtb
│       ├── am335x-boneblack-beaglebone-yocto.dtb -> am335x-boneblack--5.15.150+git0+567f0adb9d_4fca0c4373-r0-beaglebone-yocto-20241210021028.dtb
│       ├── am335x-boneblack.dtb -> am335x-boneblack--5.15.150+git0+567f0adb9d_4fca0c4373-r0-beaglebone-yocto-20241210021028.dtb
│       ├── am335x-bonegreen--5.15.150+git0+567f0adb9d_4fca0c4373-r0-beaglebone-yocto-20241210021028.dtb
│       ├── am335x-bonegreen-beaglebone-yocto.dtb -> am335x-bonegreen--5.15.150+git0+567f0adb9d_4fca0c4373-r0-beaglebone-yocto-20241210021028.dtb
│       ├── am335x-bonegreen.dtb -> am335x-bonegreen--5.15.150+git0+567f0adb9d_4fca0c4373-r0-beaglebone-yocto-20241210021028.dtb
│       ├── modules--5.15.150+git0+567f0adb9d_4fca0c4373-r0-beaglebone-yocto-20241210021028.tgz
│       ├── modules-beaglebone-yocto.tgz -> modules--5.15.150+git0+567f0adb9d_4fca0c4373-r0-beaglebone-yocto-20241210021028.tgz
│       ├── mycustom-image-beaglebone-yocto-20241215050252.qemuboot.conf
│       ├── mycustom-image-beaglebone-yocto-20241215050252.rootfs.jffs2
│       ├── mycustom-image-beaglebone-yocto-20241215050252.rootfs.manifest
│       ├── mycustom-image-beaglebone-yocto-20241215050252.rootfs.tar.bz2
│       ├── mycustom-image-beaglebone-yocto-20241215050252.rootfs.wic
│       ├── mycustom-image-beaglebone-yocto-20241215050252.rootfs.wic.bmap
│       ├── mycustom-image-beaglebone-yocto-20241215050252.testdata.json
│       ├── mycustom-image-beaglebone-yocto.jffs2 -> mycustom-image-beaglebone-yocto-20241215050252.rootfs.jffs2
│       ├── mycustom-image-beaglebone-yocto.manifest -> mycustom-image-beaglebone-yocto-20241215050252.rootfs.manifest
│       ├── mycustom-image-beaglebone-yocto.qemuboot.conf -> mycustom-image-beaglebone-yocto-20241215050252.qemuboot.conf
│       ├── mycustom-image-beaglebone-yocto.tar.bz2 -> mycustom-image-beaglebone-yocto-20241215050252.rootfs.tar.bz2
│       ├── mycustom-image-beaglebone-yocto.testdata.json -> mycustom-image-beaglebone-yocto-20241215050252.testdata.json
│       ├── mycustom-image-beaglebone-yocto.wic -> mycustom-image-beaglebone-yocto-20241215050252.rootfs.wic
│       ├── mycustom-image-beaglebone-yocto.wic.bmap -> mycustom-image-beaglebone-yocto-20241215050252.rootfs.wic.bmap
│       ├── mycustom-image.env
│       ├── rootfs
│       ├── u-boot-beaglebone-yocto-2022.01-r0.img
│       ├── u-boot-beaglebone-yocto.img -> u-boot-beaglebone-yocto-2022.01-r0.img
│       ├── u-boot-initial-env -> u-boot-initial-env-beaglebone-yocto-2022.01-r0
│       ├── u-boot-initial-env-beaglebone-yocto -> u-boot-initial-env-beaglebone-yocto-2022.01-r0
│       ├── u-boot-initial-env-beaglebone-yocto-2022.01-r0
│       ├── u-boot.img -> u-boot-beaglebone-yocto-2022.01-r0.img
│       ├── zImage -> zImage--5.15.150+git0+567f0adb9d_4fca0c4373-r0-beaglebone-yocto-20241210021028.bin
│       ├── zImage--5.15.150+git0+567f0adb9d_4fca0c4373-r0-beaglebone-yocto-20241210021028.bin
│       └── zImage-beaglebone-yocto.bin -> zImage--5.15.150+git0+567f0adb9d_4fca0c4373-r0-beaglebone-yocto-20241210021028.bin
├── licenses
│   ├── acl
│   │   ├── COPYING
│   │   ├── COPYING.LGPL
│   │   ├── generic_GPL-2.0-or-later
│   │   ├── generic_LGPL-2.1-or-later
│   │   └── recipeinfo
│   └── acl-native
│       ├── COPYING
│       ├── COPYING.LGPL
│       ├── generic_GPL-2.0-or-later
│       ├── generic_LGPL-2.1-or-later
│       └── recipeinfo
└── rpm
    ├── beaglebone_yocto
    │   ├── base-files-3.0.14-r89.beaglebone_yocto.rpm
    │   ├── base-files-dbg-3.0.14-r89.beaglebone_yocto.rpm
    │   ├── base-files-dev-3.0.14-r89.beaglebone_yocto.rpm
    │   ├── base-files-doc-3.0.14-r89.beaglebone_yocto.rpm
    │   ├── example-char-driver-1.0-r0.beaglebone_yocto.rpm
    │   ├── example-char-driver-dbg-1.0-r0.beaglebone_yocto.rpm
    │   ├── sysvinit-inittab-dbg-2.88dsf-r10.beaglebone_yocto.rpm
    │   └── sysvinit-inittab-dev-2.88dsf-r10.beaglebone_yocto.rpm
    ├── cortexa8hf_neon
    │   ├── acl-2.3.1-r0.cortexa8hf_neon.rpm
    │   ├── gawk-locale-it-5.1.1-r0.cortexa8hf_neon.rpm
    │   ├── gawk-locale-ja-5.1.1-r0.cortexa8hf_neon.rpm
    │   ├── gawk-locale-ko-5.1.1-r0.cortexa8hf_neon.rpm
    │   ├── libfontconfig-dev-2.13.1-r0.cortexa8hf_neon.rpm
    │   └── zstd-staticdev-1.5.2-r0.cortexa8hf_neon.rpm
    └── noarch
        ├── alsa-topology-conf-1.2.5.1-r0.noarch.rpm
        ├── alsa-ucm-conf-1.2.6.3-r0.noarch.rpm
        ├── autoconf-archive-2022.02.11-r0.noarch.rpm
        ├── autoconf-archive-doc-2022.02.11-r0.noarch.rpm
        └── wireless-regdb-static-2024.10.07-r0.noarch.rpm
```

The deploy directory is the central location where BitBake gathers and
organizes all of the build artifacts it generates during the image creation
process. This is one of the places you will be spending most of your time
during your Yocto journey.

### images

This directory contains bootable images and files for supporting the boot
process. Your bootloader, kernel image, root file system, device tree blobs,
and more will all be here. These are the files you grab to provision your
target hardware.

### licenses

Licenses contains all of the licenses used for all components built into
your system. If you are commercializing or otherwise publishing your custom OS,
you can go here to handle all license-related manners. For hobbyists, this
folder probably won't get used much, but it is extremely important in the
commercial size of the industry.

### rpm (or deb, ipk, etc)

This folder functions much like the repository in a typical RPM-based build
system. It serves as the central location for all your packaged binaries.
When BitBake constructs the final image, it pulls the prebuilt packages from
this directory, almost like how a traditional package manager like `yum` or
`dnf` reference their own repositories to install them. You'll frequently refer
to this directory to verify that your packages are built correctly. Tools such
as `rpm2cpio <packagename> | cpio -idmv` (or analogous for other package types)
are essential for debugging package-level issues.

## hosttools

```sh
ice@wsl2:hosttools(kirkstone)$ ll
total 0
lrwxrwxrwx 1 ice ice 10 Dec  9 19:50 '[' -> '/usr/bin/['
lrwxrwxrwx 1 ice ice 11 Dec  9 19:50  ar -> /usr/bin/ar
lrwxrwxrwx 1 ice ice 11 Dec  9 19:50  as -> /usr/bin/as
lrwxrwxrwx 1 ice ice 12 Dec  9 19:50  awk -> /usr/bin/awk
lrwxrwxrwx 1 ice ice 17 Dec  9 19:50  basename -> /usr/bin/basename
lrwxrwxrwx 1 ice ice 13 Dec  9 19:50  bash -> /usr/bin/bash
lrwxrwxrwx 1 ice ice 14 Dec  9 19:50  bzip2 -> /usr/bin/bzip2
lrwxrwxrwx 1 ice ice 12 Dec  9 19:50  cat -> /usr/bin/cat
```

The hosttools folder is a curated collection of host-side utilities that
BitBake uses during the build process. Rather than searching through the entire
system each time for a utility to use, or relying solely on the user's path,
BitBake looks in well-known paths (like `/usr/bin`) for the required tools and
creates symlinks in the hosttools directory that point to those binaries.
BitBake then references this folder any time it needs to do host-related
actions during the build.

This folder is created automatically, so you rarely need to interact with it.
However, if a required host utility moves or if BitBake behaves unexpectedly
during host-related operations, examining this folder might help diagnose the
issue.

## log

```sh
log/
├── cleanlogs
│   ├── dnsmasq
│   │   ├── log.do_clean -> log.do_clean.10821
│   │   ├── log.do_clean.10821
│   │   ├── log.task_order
│   │   ├── run.do_clean -> run.do_clean.10821
│   │   ├── run.do_clean.10821
│   │   └── run.sstate_cleanall.10821
│   ├── linux-yocto
│   │   ├── log.do_clean -> log.do_clean.29168
│   │   ├── log.do_clean.29168
│   │   ├── log.task_order
│   │   ├── run.do_clean -> run.do_clean.29168
│   │   ├── run.do_clean.29168
│   │   └── run.sstate_cleanall.29168
│   └── make-mod-scripts
│       ├── log.do_clean -> log.do_clean.29165
│       ├── log.do_clean.29165
│       ├── log.task_order
│       ├── run.do_clean -> run.do_clean.29165
│       ├── run.do_clean.29165
│       └── run.sstate_cleanall.29165
└── cooker
    └── beaglebone-yocto
        ├── 20241210005040.log
        ├── 20241210014422.log
        ├── 20241210014432.log
        ├── 20241215050333.log
        ├── 20241215050337.log
        ├── 20241215050355.log
        └── console-latest.log -> 20241215050355.log
```

The log folder is a critical component of BitBake's tmp area that serves as a
comprehensive record of every action taken during the build process.
Its primary purpose is to document the workflow, providing detailed insights
that help developers understand, debug, and optimize the build.

### cleanlogs

These logs are specifically related to cleaning operations for individual
recipes. They document the execution of tasks like `do_clean` and
`sstate_cleanall`, showing the order of operations and any issues encountered
during the cleaning phase. As you can see, I ran some of these commands against
the `dnsmasq` utility and Linux kernel. A symlink is created so that you can
simply call the log of the task that was performed, and you will get the very
latest version of this log.

The files are simple text files that detail the cleaning process steps, and it
may be beneficial when sanity checking stale references that might not have
been cleaned during the cleaning process.

### cooker

Under the cooker subdirectory, logs from the BitBake execution process (often
referred to as the cooker process) are stored. These logs are timestamped,
allowing you to track the build's progress and diagnose any errors that occur
during the build process. Like with the cleanlogs directory, the symlink, such
as `console-latest.log`, points to the most recent log file, offering a quick
way to review the latest build events.

Example of what one looks like during a successful build:

```sh
NOTE: Resolving any missing task queue dependencies

Build Configuration:
BB_VERSION           = "2.0.0"
BUILD_SYS            = "x86_64-linux"
NATIVELSBSTRING      = "universal"
TARGET_SYS           = "arm-poky-linux-gnueabi"
MACHINE              = "beaglebone-yocto"
DISTRO               = "poky"
DISTRO_VERSION       = "4.0.23"
TUNE_FEATURES        = "arm vfp cortexa8 neon callconvention-hard"
TARGET_FPU           = "hard"
meta
meta-poky
meta-yocto-bsp
meta-bbb             = "kirkstone:0bffb5eed1e8c9469b9c6e0d77f959dc9ade9c6a"
meta-oe
meta-python
meta-networking      = "kirkstone:7b3fdcdfaab2fc964bbf9eec2cce4e03001fa8cf"

Sstate summary: Wanted 0 Local 0 Mirrors 0 Missed 0 Current 1755 (0% match, 100% complete)
NOTE: Executing Tasks
NOTE: Setscene tasks completed
NOTE: Tasks Summary: Attempted 4443 tasks of which 4443 didn't need to be rerun and all succeeded.
```

## pkgdata

```sh
pkgdata
└── beaglebone-yocto
    ├── acl
    ├── alsa-lib
    ├── alsa-state
    ├── alsa-topology-conf
    ├── alsa-ucm-conf
    ├── alsa-utils
    ├── attr
    ├── autoconf-archive
    ├── avahi
    ├── base-files
    ├── base-passwd
    ├── bash
    ├── bash-completion
    <and so on>
```

The pkgdata folder is a repository of metadata for every package that BitBake
builds. It serves as a record of key package information generated during the
build process. Each file in this directory contains details about a specific
package, such as the list of package variants and associated components.

Example of what the `acl` pkgdata file looks like:

```sh
PACKAGES: acl-src acl-dbg libacl acl-ptest acl-staticdev acl-dev acl-doc acl acl-locale-de acl-locale-en+boldquot acl-locale-en+quot acl-locale-es`
```

This is a good location to debug packaging issues related to package metadata
or dependency resolution. In practice, you will rarely need to spend too much
time in this folder, however.

## saved_tmpdir

This is not a directory but rather a file. Simply put, it stores the location
of where the tmp directory exists:

```sh
/home/ice/bbb-example/poky/build/tmp
```

This file gets automatically created by BitBake, and it can help diagnose
potential issues you might face when moving workspaces and folders around.

## sstate-control

```sh
ice@wsl2:sstate-control(kirkstone)$ ll
total 19204
-rw-r--r-- 1 ice ice     216 Dec  9 19:52 index-all
-rw-r--r-- 1 ice ice    2476 Dec 15 00:03 index-allarch
-rw-r--r-- 1 ice ice   52831 Dec 15 00:03 index-beaglebone_yocto
-rw-r--r-- 1 ice ice   47299 Dec 15 00:03 index-cortexa8hf-neon
-rw-r--r-- 1 ice ice  364391 Dec 15 00:03 index-machine-beaglebone-yocto
-rw-r--r-- 1 ice ice   42917 Dec 15 00:03 index-x86_64
-rw-r--r-- 1 ice ice       0 Dec  9 19:52 manifest-all-gcc-source-11.5.0.deploy_source_date_epoch
-rw-r--r-- 1 ice ice       0 Dec  9 19:52 manifest-allarch-alsa-topology-conf.deploy_source_date_epoch
-rw-r--r-- 1 ice ice       0 Dec  9 20:02 manifest-allarch-alsa-topology-conf.package
-rw-r--r-- 1 ice ice       0 Dec  9 20:02 manifest-allarch-alsa-topology-conf.package_qa
-rw-r--r-- 1 ice ice     152 Dec  9 20:02 manifest-allarch-alsa-topology-conf.package_write_rpm
-rw-r--r-- 1 ice ice     329 Dec  9 19:53 manifest-allarch-alsa-topology-conf.populate_lic
<snipped>
```

The sstate-control directory helps in managing BitBake's shared sstate cache by
housing manifest files that record the files installed by each sstate task.
These manifests enable BitBake to clean up installed files when a recipe is
cleaned or when a newer version is about to be installed, ensuring that
outdated artifacts are removed efficiently. Additionally, the manifests allow
the build system to detect and warn about file conflicts, such as when one
task's output might overwrite files from another.

Generally, you won't be interacting with this folder too much, but it is an
extremely important folder for BitBake execution optimization.

## stamps

```sh
ice@wsl2:stamps(kirkstone)$ tree
.
├── all-poky-linux
│   ├── alsa-topology-conf
│   │   ├── 1.2.5.1-r0.do_compile.47537615809f5a079f71b20d7dce9023e7473a36825bd2bd041cb00dc0ed2216
│   │   ├── 1.2.5.1-r0.do_compile.sigdata.47537615809f5a079f71b20d7dce9023e7473a36825bd2bd041cb00dc0ed2216
│   │   ├── 1.2.5.1-r0.do_configure.423646b55ce81e1e2e1a338f550cb55453c41af25d9222f5661d5e81aa086c87
│   │   ├── 1.2.5.1-r0.do_configure.sigdata.423646b55ce81e1e2e1a338f550cb55453c41af25d9222f5661d5e81aa086c87
│   │   ├── 1.2.5.1-r0.do_deploy_source_date_epoch.38712e2a02848a19fdec77fb7607a52773f96c3d2eeb8e77abd8c9dbf97eb49c
│   │   ├── 1.2.5.1-r0.do_deploy_source_date_epoch.sigdata.38712e2a02848a19fdec77fb7607a52773f96c3d2eeb8e77abd8c9dbf97eb49c
│   │   ├── 1.2.5.1-r0.do_fetch.8cd1f65aad743ae35d59d60591a1c9c1a625b15a0d1adf742bec0399d8b2578f
│   │   ├── 1.2.5.1-r0.do_fetch.sigdata.8cd1f65aad743ae35d59d60591a1c9c1a625b15a0d1adf742bec0399d8b2578f
│   │   ├── 1.2.5.1-r0.do_unpack.cb2e174b7d4cc1558e0a05ffb141d404b409c2f543e335102a44b25e3444cf55
│   │   └── 1.2.5.1-r0.do_unpack.sigdata.cb2e174b7d4cc1558e0a05ffb141d404b409c2f543e335102a44b25e3444cf55
│   ├── alsa-ucm-conf
│   │   ├── 1.2.6.3-r0.do_compile.426c6df14d47066ebe23bc4460776c61124f46038b911f58732f2765e5a2e99b
│   │   ├── 1.2.6.3-r0.do_compile.sigdata.426c6df14d47066ebe23bc4460776c61124f46038b911f58732f2765e5a2e99b
│   │   ├── 1.2.6.3-r0.do_configure.7834045ec5b2ffea09679fd1dda8b0117d4e0bf10f0bb1dcfe8a889d105a9679
│   │   ├── 1.2.6.3-r0.do_configure.sigdata.7834045ec5b2ffea09679fd1dda8b0117d4e0bf10f0bb1dcfe8a889d105a9679
│   │   ├── 1.2.6.3-r0.do_deploy_source_date_epoch.7e8af75afbb0c50631eefe5342cd935f7d71e800038c9965505c677ff3344d0c
│   │   ├── 1.2.6.3-r0.do_deploy_source_date_epoch.sigdata.7e8af75afbb0c50631eefe5342cd935f7d71e800038c9965505c677ff3344d0c
│   │   ├── 1.2.6.3-r0.do_fetch.ccdc5c5c97e58d52dd0a3684133d2b5064edb30cb3c960d7659160e456c61dd8
│   │   ├── 1.2.6.3-r0.do_fetch.sigdata.ccdc5c5c97e58d52dd0a3684133d2b5064edb30cb3c960d7659160e456c61dd8
│   │   ├── 1.2.6.3-r0.do_install.53b78f2fcca2714bb184c26974a12231fe3002731095ffb51b75169f87f6957a
```

This is another directory that BitBake uses to track and manage what tasks have
run and when they were last ran. It's broken down by arch, package name, and
version, and binary files are written here by BitBake so it can keep track of
whether or not the task needs to be rerun.

Interestingly enough, all of these files are actually empty. BitBake is simply
using the name and time stamps to manage tasks.

In practice, this is an area you will almost never need to interact with.

## sysroots

```sh
sysroots
└── beaglebone-yocto
    └── imgdata
        └── mycustom-image.env
```

The sysroots folder is an interesting one. It is actually a legacy folder back
when BitBake used to create a global shared sysroot per machine, along with an
accompanying native one. This has since changed, and it is generally not
recommended to populate this directory. At most, you will find a general `.env`
file these days which configures some general variables regarding the build.

In my experience, this folder is an area you will rarely, if ever, interact
with.

## sysroots-components

```sh
sysroots-components
├── all
│   ├── autoconf-archive
│   │   ├── sysroot-providers
│   │   │   └── autoconf-archive
│   │   └── usr
│   │       └── share
│   │           └── aclocal
│   │               ├── ax_absolute_header.m4
│   │               ├── ax_ac_append_to_file.m4
│   │               ├── ax_ac_print_to_file.m4
│   │               └── ax_zoneinfo.m4
│   ├── update-rc.d
│   │   └── sysroot-providers
│   │       └── update-rc.d
│   └── wayland-protocols
│       ├── sysroot-providers
│       │   └── wayland-protocols
│       └── usr
│           └── share
│               ├── pkgconfig
│               │   └── wayland-protocols.pc
│               └── wayland-protocols
│                   ├── stable
│                   │   ├── presentation-time
│                   │   │   └── presentation-time.xml
│                   │   ├── viewporter
│                   │   │   └── viewporter.xml
│                   │   └── xdg-shell
│                   │       └── xdg-shell.xml
│                   ├── staging
│                   │   ├── drm-lease
│                   │   │   └── drm-lease-v1.xml
│                   │   ├── ext-session-lock
│                   │   │   └── ext-session-lock-v1.xml
│                   │   └── xdg-activation
│                   │       └── xdg-activation-v1.xml
│                   └── unstable
│                           └── xwayland-keyboard-grab-unstable-v1.xml
├── beaglebone_yocto
│   ├── base-files
│   │   ├── etc
│   │   │   └── skel
│   │   ├── lib
│   │   ├── sysroot-providers
│   │   │   └── base-files
│   │   └── usr
│   │       ├── include
│   │       ├── lib
│   │       └── share
│   │           ├── common-licenses
│   │           ├── dict
│   │           ├── doc
│   │           └── misc
│   ├── depmodwrapper-cross
│   │   ├── fixmepath
│   │   ├── fixmepath.cmd
│   │   ├── sysroot-providers
│   │   │   └── depmodwrapper-cross
│   │   └── usr
│   │       └── bin
```

Whenever BitBake runs the `do_prepare_recipe_sysroot` task, it links or
otherwise copies those contents into this folder for each recipe listed in
the `DEPENDS` section of a recipe. The sysroots-components directory is
essentially where these prepared sysroot components reside. Its contents are
automatically populated via shared state, as dictated by the `COMPONENTS_DIR`
variable. This process ensures that every recipe has the necessary files
and libraries available during the build without manual intervention.

Rarely, if ever, will you need to interact with this, but it might come in
handy if you need to troubleshoot issues related to dependency resolution.

## sysroots-uninative

```sh
sysroots-uninative
├── buildinfo
├── relocate_sdk.py
└── x86_64-linux
    ├── etc
    │   ├── bindresvport.blacklist
    │   ├── ld.so.cache -> /etc/ld.so.cache
    │   ├── ld.so.conf
    │   └── netconfig
    ├── lib
    │   ├── ld-linux-x86-64.so.2
    │   ├── ld-linux-x86-64.so.2.chksum
    │   ├── libBrokenLocale.so.1
    │   ├── libanl.so.1
    │   ├── libc.so.6
    │   ├── libdl.so.2
    │   ├── libgcc_s.so.1
    │   ├── libm.so.6
    │   ├── libresolv.so.2
    │   ├── librt.so.1
    │   └── libutil.so.1
    ├── sbin
    ├── usr
    │   ├── bin
    │   │   └── patchelf-uninative
    │   └── lib
    │       ├── audit
    │       ├── gconv
    │       │   ├── ANSI_X3.110.so
    │       │   ├── ARMSCII-8.so
```

The sysroots-uninative directory provides a precompiled, minimal native
environment that BitBake uses to run native tools during the build process.
This directory mirrors a standard Linux filesystem, containing directories such
as `etc`, `lib`, `sbin`, and so on, along with essential libraries and
binaries. It is mainly to ensure that native build tasks run reliably and
consistently across different host systems, even when the host's native
libraries might be outdated or incompatible.

That said, this is another series of folder that you will rarely need to
interact with during your day-to-day Yocto development.

## work

Finally, we get to the heart of the BitBake process and where you will be
spending most of your time when building and debugging. The work directory is
where BitBake carries out the build process for every package in their
respective architecture-specific workspaces.

For every recipe, a directory is created based on the recipe name, the version,
and target arch. The source is unpacked here, patched, and compiled.

### tunearch/recipename/version

The above is how packages get split out. Let's take a look at the following
area: `beaglebone_yocto-poky-linux-gnueabi/u-boot/1_2022.01-r0/`

```sh
ice@wsl2:1_2022.01-r0(kirkstone)$ ll
total 84
-rw-r--r--  1 ice ice  2382 Dec  7 17:37 0001-fs-squashfs-Use-kcalloc-when-relevant.patch
-rw-r--r--  1 ice ice  2951 Dec  7 17:37 0001-fs-squashfs-sqfs_read-Prevent-arbitrary-code-executi.patch
-rw-r--r--  1 ice ice  3691 Dec  7 17:37 0001-i2c-fix-stack-buffer-overflow-vulnerability-in-i2c-m.patch
-rw-r--r--  1 ice ice  7942 Dec  7 17:37 0001-net-Check-for-the-minimum-IP-fragmented-datagram-siz.patch
-rw-r--r--  1 ice ice  1605 Dec  7 17:37 0001-riscv-fix-build-with-binutils-2.38.patch
-rw-r--r--  1 ice ice  1135 Dec  7 17:37 0001-riscv32-Use-double-float-ABI-for-rv32.patch
drwxr-xr-x 19 ice ice  4096 Dec  9 20:02 build
drwxr-xr-x  2 ice ice  4096 Dec  9 19:53 deploy-source-date-epoch
drwxr-xr-x  2 ice ice  4096 Dec  9 20:02 deploy-u-boot
drwxr-xr-x 28 ice ice  4096 Dec  9 20:01 git
drwxr-xr-x  4 ice ice  4096 Dec  9 20:02 image
drwxr-xr-x  3 ice ice  4096 Dec  9 19:53 license-destdir
drwxr-xr-x  2 ice ice  4096 Dec  9 20:02 pseudo
drwxr-xr-x  5 ice ice  4096 Dec  9 20:02 recipe-sysroot
drwxr-xr-x  7 ice ice  4096 Dec  9 19:53 recipe-sysroot-native
drwxr-xr-x  2 ice ice  4096 Dec  9 19:51 source-date-epoch
drwxr-xr-x  3 ice ice  4096 Dec  9 20:02 sysroot-destdir
drwxr-xr-x  2 ice ice 12288 Dec  9 20:02 temp
```

- **Patch Files (e.g., `0001-*.patch`):**  
  These files contain patches that are applied to the source code during the
  build process. They modify or fix the upstream code before compilation.

- **build:**  
  This directory is used for out-of-tree builds. It contains temporary build
  artifacts generated during compilation and linking. If the recipe supports
  separate source and build directories, this is where the build is performed.

- **deploy-source-date-epoch:**  
  Holds files related to source date epoch data. This helps ensure reproducible
  builds by recording a consistent timestamp for the source files.

- **deploy-u-boot:**  
  Contains deployment artifacts specific to U-Boot. After the build, final
  bootloader images or related files are placed here, ready to be integrated
  into the final image.

- **git:**  
  This directory contains the source code repository or a copy of the source
  code fetched via git. It stores the unmodified upstream source before any
  patches are applied. This is where `${S}` resides.

- **image:**  
  The output of the `do_install` task is placed in this folder. It represents
  the file system hierarchy as installed by the recipe, which will later be
  used to create packages or merged into the final image.

- **license-destdir:**  
  Contains the license files and related documentation installed as part of the
  recipe. This ensures that proper licensing information is packaged with the
  software.

- **pseudo:**  
  Holds the pseudo database and logs for tasks executed under pseudo. Pseudo
  is used to emulate root user behavior during the build process, which is
  necessary for managing file ownership and permissions.

- **recipe-sysroot:**  
  Contains target-specific libraries and headers needed during the build. This
  sysroot mimics the target system's filesystem, ensuring that dependencies
  are correctly resolved.

- **recipe-sysroot-native:**  
  Similar to `recipe-sysroot`, but it holds native tools and dependencies
  required during the build process (e.g., compilers, scripts, and utilities)
  that run on the host system.

- **source-date-epoch:**  
  Stores files that record the source date epoch, contributing to reproducible
  builds by providing a consistent reference time for the source code.

- **sysroot-destdir:**  
  Contains the output of the `do_populate_sysroot` task. This directory
  structure is merged into the final sysroot for the recipe, ensuring all
  necessary dependencies are present.

- **temp:**  
  This folder stores logs and runtime scripts for each task executed for the
  recipe. It includes files like `log.do_*` for task logs, `run.do_*` scripts
  that contain the commands run, and a `log.task_order` file detailing the
  sequence of task execution.

Understanding the work directory is absolutely essential for any Yocto
developer aiming to master the system. This area is where you will spend most
of your time debugging builds, tracing where files are placed in the
filesystem, and unraveling how source code is compiled. It provides insight
into how packages are split and assembled, as well as how every component is
customized before becoming part of your custom distribution. Mastering this
space is key to troubleshooting and optimizing your Yocto builds.

## work-shared

```sh
work-shared
├── beaglebone-yocto
│   ├── kernel-build-artifacts
│   └── kernel-source
└── gcc-11.5.0-r0
    ├── 0001-CVE-2021-42574.patch
    ├── 0001-CVE-2021-46195.patch
    ├── 0001-aarch64-Update-Neoverse-N2-core-definition.patch
    ├── 0001-gcc-4.3.1-ARCH_FLAGS_FOR_TARGET.patch
    ├── 0002-CVE-2021-42574.patch
    ├── 0002-aarch64-add-armv9-a-to-march.patch
    ├── 0002-gcc-poison-system-directories.patch
    ├── 0003-64-bit-multilib-hack.patch
    ├── 0003-CVE-2021-42574.patch
    ├── 0003-aarch64-Enable-FP16-feature-by-default-for-Armv9.patch
    ├── 0004-CVE-2021-42574.patch
    ├── 0004-Use-the-defaults.h-in-B-instead-of-S-and-t-oe-in-B.patch
    ├── 0004-arm-add-armv9-a-architecture-to-march.patch
    ├── 0005-cpp-honor-sysroot.patch
    ├── 0006-Define-GLIBC_DYNAMIC_LINKER-and-UCLIBC_DYNAMIC_LINKE.patch
    ├── 0007-gcc-Fix-argument-list-too-long-error.patch
    ├── 0008-libtool.patch
    ├── 0009-gcc-armv4-pass-fix-v4bx-to-linker-to-support-EABI.patch
    ├── deploy-source-date-epoch
    ├── gcc-11.5.0
    ├── recipe-sysroot
    ├── recipe-sysroot-native
    ├── source-date-epoch
    └── temp
```

The work-shared directory is for any recipes that share a work directory with
other recipes. In practice, this is where you will go to work with, or
otherwise debug, large shared recipes such as `gcc` and the Linux kernel.

Very few actual recipes use this outside of the compiler and the Linux kernel,
but the moment you start customizing the kernel is the moment this directory
becomes extremely important. Generally, this is left for those who are starting
to read the
[kernel development manual](https://docs.yoctoproject.org/kernel-dev/index.html).

## Wrapping Up

Understanding what happens during build time is crucial for anyone working with
Yocto and BitBake. Hopefully this deepdive helped clarify some of the reasons
why folders exist and what their respective roles are. As you start to get more
familiar with the internal structure of Yocto-built Linux distros, you will be
able to more quickly diagnose and resolve build issues, making the development
process smoother and more predictable.

As stated in the tmp section, the next post will focus on the inner workings of
the tmp folder given its importance. I'll try to break down real examples and
explain what each folder, and even some files, are doing.

## Further Reading

Some useful links that helped me when I first started out:

- [Source Directory Structure](https://docs.yoctoproject.org/ref-manual/structure.html#)
- [Yocto Project Linux Kernel Development Manual](https://docs.yoctoproject.org/kernel-dev/index.html)
- [Fakeroot and Pseudo](https://docs.yoctoproject.org/overview-manual/concepts.html#fakeroot-and-pseudo)

Happy building!
