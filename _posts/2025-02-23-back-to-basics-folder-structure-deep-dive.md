---
title: "Back to Basics: Folder Structure Deep Dive"
description: A look at the folder structure in Yocto
date: 2025-02-23 00:00:00 -0500
categories: [Programming, Shell Scripting, Linux, Embedded Linux, Yocto]
tags: [linux, embedded linux, scripting]
---

In the [last blog](https://tomice.github.io/posts/back-to-basics-with-yocto/),
we explored the history of Yocto, dove into what Poky is about and
how it works, and pulled back the curtain on BitBake's inner workings—from its
layering and recipe syntax to its server, cooker, and task execution engine.

This time, we'll take another deep dive, but from the perspective of the
directory structure that Yocto generates when executing a build, as well as
some common debugging tips as you continue your journey throughout embedded
Linux development.

## Example Folder Structure

Before we begin, we should level-set what a typical Yocto-based directory
structure looks like. In most modern tutorials you see on Yocto, the guide will
have you clone Poky. As we learned in the last blog, it's because Poky is the
reference distribution for Yocto. It provides a great starting point for devs
to start their embedded Linux journey.

In the past, you used to have to clone BitBake itself, and as such, the
actual folder structure would be slightly different compared to newer tutorials
you see online. This doesn't impact how BitBake works; as long as the `.conf`
files are configured correctly, folder structure doesn't matter too much. It is
just a little note to keep in the back of your mind when comparing older
workspaces to newer ones.

Let's choose a general workspace where you first clone Poky, and everything
resides in that folder:

```sh
poky/
├── bitbake/
│   ├── bin/
│   ├── contrib/
│   ├── doc/
│   └── lib/
├── build/
│   ├── cache/
│   ├── conf/
│   ├── downloads/
│   ├── sstate-cache/
│   └── tmp/
├── contrib/
├── documentation/
├── meta/
│   ├── classes/
│   ├── conf/
│   ├── recipes-bsp/
│   └── <additional extra folders in a typical meta layer>/
└── scripts/
```

## Peering into BitBake

The BitBake folder contains the heart of Yocto's build system. It contains
the core build engine which is responsible for parsing recipes, scheduling
tasks, and handling the overall build process for Yocto.

### bin

The bin folder contains the executable scripts used to run BitBake.
The scripts include the main BitBake command (e.g., `bitbake`), along with
utilities such as `bitbake-server`, `bitbake-layers`, and various worker
utilities. They serve as the primary interfaces and helpers to launch and
manage builds in Yocto.

Included in this is a utility called
[Toaster](https://wiki.yoctoproject.org/wiki/Toaster) which is the web-based
GUI to OpenEmbedded and BitBake. Toaster is a means of providing a more
user-friendly interface to the Yocto project. Definitely check this out if you
are new to Yocto as it makes managing, monitoring, and analyzing builds much
easier if you aren't used to how everything is laid out in Yocto yet.

### contrib

The contrib folder holds community-contributed tools and scripts that extend
BitBake's functionality or help in debugging and testing. For instance, there
exists a torture test utility called `bbparse-torture.py` that stress tests
BitBake's parser by interrupting it randomly during execution. There is a
helper script called `bbdev.sh` to set up and export env vars necessary for
working with BitBake, and more.

### doc

This directory is dedicated to BitBake's documentation. It includes source
files (mostly written in reStructuredText and other formats) that can be
processed to generate user manuals and reference guides. The important thing to
realize about this is that the documents are BitBake specific. They are not
necessarily Yocto-related. Remember, BitBake is the parsing engine, so if you
are looking for general information about Yocto, you will want to look at the
actual documentation folder which we'll get to later on.

### lib

The lib folder is where the core libraries and modules of BitBake reside.
It contains Python packages and modules (for example, the bb and bblayers
directories) that implement BitBake's main logic - such as parsing recipes,
managing tasks, and handling dependency resolution. Other items,
like `codegen.py`, progressbar, etc. provide supporting functionality that
BitBake relies on during the build process. The directory also includes some
third-party libraries (e.g., bs4 for HTML processing) and utilities for
notifications, diffing, and more. Most likely, you will not need to interact
with this folder too much, but it is nice to know what is inside here.

## Contrib

Like in the previous contrib folder inside of the BitBake folder, there also
exists a contrib folder in Poky itself. It contains some artwork, but most
important in here is the git hook you can add to send patches to upstream.
If you're not attempting to send patches to upstream, you can probably skip
looking at this folder entirely.

## Documentation

The documentation folder is probably the most overlooked folder in Yocto,
and yet it is easily the most useful folder out of them all. It contains a
plethora of information on the Yocto project in general, ranging from general
overviews, developer manuals, BSP guides, kernel documentation, SDK docs, and
even lessons learned from developers who have been programming in Yocto-based
projects.

Inside the folder is a `Makefile` that will generate the Sphinx-based docs.
The `.rst` files can all still be read with a standard text editor, but the
`Makefile` will generate `.html` that is a bit easier to read. This folder
should be the first thing you check out as you start to begin your journey
with Yocto.

## Meta

The meta folder is the Poky reference architecture. In this, you will see the
classes, conf files, and recipes for a basic embedded Linux image built with
Yocto. This should be the de facto place you look at for examples on how to
create recipes, how to lay out your own meta layer, how to write classes, and
how to develop using Yocto. Leverage the code and examples in here as much as
possible to ensure your build remains maintainable and consistent with how the
community expects your distro to be built.

## Scripts

The scripts directory contains a variety of utility scripts to help with tasks
related to the build process, configuration, testing, performance monitoring,
and more. These scripts enhance the development and build experience within the
Yocto ecosystem. I'll categorize some of these and highlight a few of them, but
it's definitely worth checking out as there are many more than the ones I list:

### Automation and Build Scripts

- **`buildall-qemu`**: Automates the process of building and testing recipes
                       across multiple QEMU machine configurations. See the
                       QEMU section below for more info.
- **`oe-selftest`**: Runs internal self-tests for BitBake tools.
- **`oe-run-native`**: Runs native tools within the build environment.
- **`devtool`**: Provides tools for developing and modifying recipes.
- **`resulttool`**: Analyzes test results and build artifacts for validation.
- **`wic`**: Creates bootable images for target devices using kickstart files.  
             Supports various image formats and partitioning schemes, essential  
             for customizing and deploying embedded Linux images.

### Performance and Build History Tools

- **`buildhistory-diff`**: Reports differences in build history between revisions.
- **`buildstats-diff`**: Compares build statistics between builds.
- **`task-time`**: Reports the time consumed by tasks during builds.
- **`oe-build-perf-report`**: Generates performance reports for builds.
- **`oe-build-perf-test`**: Tests the performance of builds via benchmarks.

### Repository and Patch Management Utilities

- **`combo-layer`**: Manages and merges multiple repositories into a combo-layer repo.
- **`create-pull-request`**: Automates the creation of pull requests in Git repositories.
- **`gen-lockedsig-cache`**: Generates signature caches for reproducible builds.
- **`bitbake-prserv-tool`**: Provides utilities for managing the BitBake PR service.

### Testing and Validation Tools

- **`oe-test`**: Executes tests related to OpenEmbedded builds.
- **`test-reexec`**: Tests the re-execution of processes for validation purposes.
- **`test-remote-image`**: Tests remote images by deploying and verifying them.
- **`verify-bashisms`**: Checks shell scripts for compatibility issues related
                         to non-POSIX features.
- **`yocto-check-layer`**: Verifies the correctness of Yocto Project layers.

### Package and SDK Management Tools

- **`oe-pkgdata-browser`**: GUI tool for browsing pkgdata information.
- **`oe-publish-sdk`**: Publishes generated SDKs to remote locations.
- **`install-buildtools`**: Installs required build tools for the environment.
- **`nativesdk-intercept`**: Handles SDK-specific task interceptions.
- **`native-intercept`**: Manages intercepts for native builds.

### QEMU Utilities

- **`runqemu`**: Runs OpenEmbedded images with QEMU for testing.
- **`runqemu-export-rootfs`**: Exports root filesystem images for QEMU.
- **`runqemu-extract-sdk`**: Extracts SDKs for QEMU testing.
- **`runqemu-addptable2image`**: Adds partition tables to disk images for QEMU.
- **`runqemu-gen-tapdevs`**: Generates tap devices for QEMU networking.
- **`runqemu-ifup`**: Handles network interface setup for QEMU.
- **`runqemu-ifdown`**: Handles network interface teardown for QEMU.

### Utility Scripts

- **`cp-noerror`**: Copies files while ignoring errors.
- **`oe-time-dd-test.sh`**: Tests disk I/O performance using `dd`.
- **`oe-depends-dot`**: Generates dependency graphs for visualization.
- **`pybootchartgui`**: Generates graphical representations of build performance.
- **`opkg-query-helper.py`**: Assists with querying package data for `opkg`.
- **`relocate_sdk.py`**: Adjusts SDK paths for relocatable SDKs.
- **`sysroot-relativelinks.py`**: Converts absolute links in sysroots to relative links.

Again, there is much more in the scripts folder. Once you get familiar with
BitBake, I highly recommend taking a look at all of the scripts in here. There
is a good chance a dev wrote a script that can boost productivity as you work
in Yocto.

## Inside the Build Directory

You might have noticed we skipped over the build directory. That was on purpose
as the build directory might be the number one place you will spend your time
as a Yocto developer. As the name implies, it contains everything related to
the BitBake build, and as such, it tends to be the most complex folder of
them all.

### Cache

The cache folder is a simple folder for BitBake to store some metadata in.
This helps speed up subsequent builds by avoiding redundant parsing. Various
`.dat` files exist here which contain cached parsing data, hash values
associated with tasks, checksums of local files, and more. A SQLite database
resides in here that is used by BitBake to store persistent data across builds,
too.

You normally won't need to interact with this folder much, if at all, during
your typical development process, but it is nice to know why the folder exists
and what is going on in here.

### Conf

The conf folder is where you will spend most of your time configuring and
customizing your distro as you begin your Yocto journey.

#### `bblayers.conf`

This file specifies the paths to the meta layers that should be included in
the build. If you remember from previous blogs where we created a `layers.conf`
file, then this will look extremely familiar. The main difference between the
two is that `bblayers.conf` is meant to define which layers are included in the
build, along with their paths, whereas `layers.conf` defines configuration
settings for a specific layer.

#### `local.conf`

The `local.conf` file is where you will spend a majority of your time
customizing your distro. It allows developers to specify options such
as the target machine, image types, and more. As described in the large
comment at the beginning of the file, it is, "your local configuration file
and where all local user settings are placed." The file is heavily commented
to help guide you.

Want to add `nano` or `htop` to your distro? You would place it in the image
output configuration section (e.g., `IMAGE_INSTALL:append = " nano htop"`).
Want to choose what kind of package manager you want to use? You'd define it
in `PACKAGE_CLASSES`. Locale, timezones, debug settings, SDK configurations
and more all reside in this one file. Take the time to really read up on every
option in here, as it will be extremely important as you build your distro.

#### `templateconf.cfg`

This simple file is nothing but a reference that specifies the path to the
template configuration used when initializing the build directory. It is a
plaintext file and human readable. You most likely won't have to interact with
this unless you are working in multi-project environments.

> *Note:* There exists some more advanced features that can get added to the
> build directory when you enable
> [multiconfig](https://docs.yoctoproject.org/bitbake/2.10/bitbake-user-manual/bitbake-user-manual-intro.html#executing-a-multiple-configuration-build).
> This is a much more advanced feature for executing multiple configuration
> builds, and you probably won't encounter it when you first start your Yocto
> journey, but it is worth mentioning as it is used heavily in professional
> environments.

### Downloads

The easiest and most straight-forward directory to explain is probably the
downloads directory, but it actually has a few tricks up its sleeves. This
folder contains all of the source code downloaded from each URL set in the
`SRC_URI` variable of every recipe. Let's first take a look at a real example
from my previous BeagleBone Black build. Note that I'll be shortening some of
the output for brevity:

```sh
ice@wsl2:downloads(kirkstone)$ ll
total 1443676
-rw-r--r--  1 ice ice    518292 Mar 16  2021 acl-2.3.1.tar.gz
-rw-r--r--  1 ice ice       463 Dec  9 19:50 acl-2.3.1.tar.gz.done
-rw-r--r--  1 ice ice   1079670 Dec  9  2021 alsa-lib-1.2.6.1.tar.bz2
-rw-r--r--  1 ice ice       463 Dec  9 19:51 alsa-lib-1.2.6.1.tar.bz2.done
```

In this folder, you can see the tarball from the source, as well as a special
file with a `.done` extension. Before diving into the `.done` file, let's take
a look at the `acl-2.3.1.tar.gz` file:

```sh
ice@wsl2:downloads(kirkstone)$ tar xzf acl-2.3.1.tar.gz
ice@wsl2:downloads(kirkstone)$ cd acl-2.3.1/
ice@wsl2:acl-2.3.1(kirkstone)$ ll
total 868
-rw-r--r-- 1 ice ice  93787 Oct 15  2018 ABOUT-NLS
-rw-r--r-- 1 ice ice    663 Mar  2  2021 Makefile.am
-rw-r--r-- 1 ice ice 147906 Mar 16  2021 Makefile.in
-rw-r--r-- 1 ice ice    410 Feb 28  2018 README
-rw-r--r-- 1 ice ice     58 Mar  1  2021 TODO
-rw-r--r-- 1 ice ice  44348 Mar 16  2021 aclocal.m4
drwxr-xr-x 2 ice ice   4096 Mar 16  2021 build-aux
```

As you can see, this is the same source that we are grabbing from the
[acl](https://layers.openembedded.org/layerindex/recipe/335205/) recipe in the
[sources section](https://download.savannah.gnu.org/releases/acl/acl-2.3.1.tar.gz).

Simple and straight forward - it is the tarball from the `SRC_URI`. Next, if
we look a little deeper in the `downloads` folder, we will see a `git2` folder.
This is where some repositories get stored when they get fetched.

```sh
ice@wsl2:downloads(kirkstone)$ cd git2/
ice@wsl2:git2(kirkstone)$ ll
total 428
drwxr-xr-x 7 ice ice 4096 Dec  7 17:43 git.kernel.org.pub.scm.linux.kernel.git.xiang.erofs-utils.git
-rw-r--r-- 1 ice ice    6 Dec  9 19:51 git.kernel.org.pub.scm.linux.kernel.git.xiang.erofs-utils.git.done
drwxr-xr-x 7 ice ice 4096 Dec  7 17:43 git.kernel.org.pub.scm.network.nfc.neard.git
-rw-r--r-- 1 ice ice    6 Dec  9 19:50 git.kernel.org.pub.scm.network.nfc.neard.git.done
```

Given that they have the `.git` extension, this should be a hint that they are
bare/mirrored repos. Let's do a quick test to see what it is:

```sh
ice@wsl2:git2(kirkstone)$ cd git.yoctoproject.org.opkg-utils/
ice@wsl2:git.yoctoproject.org.opkg-utils(master)$ ll
total 36
-rw-r--r-- 1 ice ice   23 Dec  7 17:42 HEAD
drwxr-xr-x 2 ice ice 4096 Dec  7 17:42 branches
-rw-r--r-- 1 ice ice  170 Dec  7 17:42 config
-rw-r--r-- 1 ice ice   73 Dec  7 17:42 description
drwxr-xr-x 2 ice ice 4096 Dec  7 17:42 hooks
drwxr-xr-x 2 ice ice 4096 Dec  7 17:42 info
drwxr-xr-x 4 ice ice 4096 Dec  7 17:42 objects
-rw-r--r-- 1 ice ice 1280 Dec  7 17:42 packed-refs
drwxr-xr-x 4 ice ice 4096 Dec  7 17:42 refs
ice@wsl2:git.yoctoproject.org.opkg-utils(master)$ git rev-parse --is-bare-repository
true
ice@wsl2:git.yoctoproject.org.opkg-utils(master)$ git config --get remote.origin.mirror
true
```

So these repositories are bare and
[mirrored repos](https://docs.gitlab.com/user/project/repository/mirror/).
Again, simple and straight-forward. Finally, we'll circle back to those `.done`
files and what they are.

When BitBake grabs a source from the internet, whether it be a tarball or a git
repo, it will generate a `.done` file with the same base name upon a successful
fetch. This is actually a binary file and not a human-readable file. It serves
to prevent redundant downloads if the file is already present, as well as
confirm to BitBake that the file was indeed downloaded correctly.

This should be the first place you check to verify that a download was fetched
correctly. If the `.done` file exists, then the fetch *should* be sound as it
means the source was fetched and the checksum was verified. You can also get
around network access issues if you `touch` a `.done` file, but be careful with
this as it will break reproducibility.

### sstate-cache

The sstate-cache directory is probably one of the most important directories
that helps optimize BitBake, and yet it's probably the one where you will spend
the least amount of your time in. It acts similar to how git hashes objects if
you have ever looked into git's `.git/objects` directory:

```sh
sstate-cache/
├── 00/
│   ├── 05/
│   │   ├── sstate:lighttpd:...:package_write_rpm.tar.zst
│   │   └── sstate:lighttpd:...:package_write_rpm.tar.zst.siginfo
│   ├── 2d/
│   │   └── sstate:openssl:...:compile.tar.zst.siginfo
│   └── ...
├── 01/
│   ├── 00/
│   │   ├── sstate:libcheck:...:configure.tar.zst.siginfo
│   ├── c4/
│   │   ├── sstate:dnsmasq-config:...:packagedata.tar.zst
│   │   └── sstate:dnsmasq-config:...:packagedata.tar.zst.siginfo
│   └── ...
```

Every time a task gets executed, that task gets hashed and its status is placed
in this folder. This is how BitBake is able to know whether or not to perform
certain tasks upon a rebuild.

One neat little trick you can do in a professional environment with many devs
working in Yocto is to generate a prebuilt sstate-cache that gets shared with
everyone. Assuming your OS is stable enough, and your build environment remains
consistent, this will vastly speed up build times for everyone.

On the other hand, if you ever run into a weird state where tasks are not
executing correctly due to a major change you made, it might be worth deleting
this folder and re-running the build from scratch. It will take longer, but it
ensures no stale cache issues and helps to sanity check the build, especially
when debugging task execution order problems.

### tmp

This is the primary output directory where all intermediate and final build
products are stored. Its structure is critical for managing, cachcing, and
deploying build outputs. In fact, this directory is so large that I figured
it would be best to have its own dedicated blog post, so stay tuned for the
next post on the nitty gritty details of the tmp folder.

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

- [Architecture of Open Source: Yocto](https://aosabook.org/en/v2/yocto.html)
- [Yocto Project Quick Build](https://docs.yoctoproject.org/brief-yoctoprojectqs/index.html)
- [Yocto Reference Manual](https://docs.yoctoproject.org/ref-manual/index.html)
- [Variable names and their paths](https://git.yoctoproject.org/poky/tree/meta/conf/bitbake.conf?h=kirkstone)

Happy building!
