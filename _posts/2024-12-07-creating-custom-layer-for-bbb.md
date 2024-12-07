---
title: Creating a Custom Layer for the BeagleBone Black in Yocto
description: Learn to create and integrate a custom Yocto meta layer for the BeagleBone Black
date: 2024-12-07 00:00:00 -0500
categories: [Programming, Shell Scripting, Linux, Embedded Linux, Yocto]
tags: [linux, embedded linux, scripting]
---

In the [previous tutorial](https://tomice.github.io/posts/beginning-yocto-for-bbb/),
we built a custom embedded Linux distribution for the BeagleBone Black using
the `meta-yocto-bsp` layer provided by Poky and OpenEmbedded. This setup
offered a great starting point for most general-purpose tasks and
demonstrated the power of Yocto for embedded development. Towards the end of
the tutorial, we added some features to the base image such as `vim` and `git`.

In this tutorial, we'll go a step further and create our own custom meta layer.
This will allow us to centralize our customizations to the image and give us
a more organized approach to future modifications of the system. By the end of
this tutorial, you'll have created your own meta layer that centralizes
modifications, making it easier to maintain and extend your build. As a
practical example, we'll show how to host a simple web page on a `lighttpd`
server running on the BeagleBone Black.

## Custom Layers

We briefly touched upon layers in the last article, but we'll go through them
in greater detail here so you can get a better understanding of their purpose,
how they work, and why you would want to use them.

As stated in the previous article, layers are a collection of recipes,
configuration files, and other components organized in a way to be shared
across projects - or even just in a way that allows you to centralize the
majority of your changes.

Board vendors tend to provide their own layers where they essentially unlock
most, if not all, of the features of the board so you can utilize the hardware
to its fullest.

There are two ways to create layers:

1. The manual way
2. The automated way via BitBake

For more complex or heavily customized environments, the manual method provides
granular control. For straightforward cases, the automated way offers a faster,
more error-free setup. We'll use the automated way given how simple our project
is, but we'll run through everything it does in case you ever need to do this
manually on a project.

## Prerequisites

See the previous blog for the necessary prerequisites as these haven't changed.
This tutorial assumes you have read the previous post and have a general idea
of what is going on within Yocto. We'll still touch on concepts as we come
across them, though. To keep with the same theme, we will continue to use
Kirkstone as our main version.

To quickly catch up, open up a terminal and perform the following:

```sh
# Create the directory to build the image and enter it
mkdir ~/bbb-example ; cd $_
# Clone Poky and enter it
git clone git://git.yoctoproject.org/poky -b kirkstone ; cd poky
# Setup the environment
source oe-init-build-env
# Change MACHINE to be beaglebone-yocto
sed -i 's/^MACHINE ??= "qemux86-64"/MACHINE ??= "beaglebone-yocto"/' conf/local.conf
```

> **Note:** In the previous tutorial, I showed how to add `vim` and `git` to
> your image via the `conf/local.conf`. Since we are going to pivot
> to our new image, those won't be used and can be removed.

## Creating the Meta Layer

We will pick up from where we left off. I am assuming you have
`oe-init-build-env` currently sourced and your `MACHINE` variable in the
`build/conf/local.conf` set to `beaglebone-yocto` as shown above.

### 1. Navigate to the Poky Directory

We want to be in the `poky` directory so our meta layer is co-located with
the other meta layers. I am mostly doing this to ensure the new meta layer is
easily accessible alongside the Poky distribution and other layers, making it
simpler to reference in the future. You are free to change this, but always
note the location as it will matter in the next couple steps:

```sh
cd ~/bbb-example/poky
```

### 2. Create the Meta Layer

Creating a custom layer using BitBake's built-in
[`create-layer`](https://docs.yoctoproject.org/dev/dev-manual/layers.html#creating-a-general-layer-using-the-bitbake-layers-script)
script. It works by using the `bitbake-layers` tool and providing it with a
name you would like to call the new meta layer.

```sh
bitbake-layers create-layer meta-bbb
```

BitBake will output:

```sh
NOTE: Starting bitbake server...
Add your new layer with 'bitbake-layers add-layer meta-bbb'
```

It will automatically spin up the BitBake server and create a directory
for you in your `poky` directory. It should look like this:

```sh
ice@wsl2:meta-bbb(kirkstone)$ tree
.
├── COPYING.MIT
├── README
├── conf
│   └── layer.conf
└── recipes-example
    └── example
        └── example_0.1.bb
```

These files form the basic structure of a meta layer. Let's review what each
file does to better understand what BitBake is doing in case we ever need to
manually do this process in a more complex project:

#### COPYING.MIT

The `COPYING.MIT` file contains the text of the
[MIT license](https://opensource.org/license/mit). It's included by default
because many layers are shared or distributed, and licensing ensures proper
attribution and compliance with open-source practices.

This file is not mandatory, but it is considered best practice to include a
license for your meta layer.

#### README

The `README` is a pre-generated text file to help kickstart your documentation
for your custom layer. By default, it looks like the following:

```sh
This README file contains information on the contents of the meta-bbb layer.

Please see the corresponding sections below for details.

Dependencies
============

  URI: <first dependency>
  branch: <branch name>

  URI: <second dependency>
  branch: <branch name>

  .
  .
  .

Patches
=======

Please submit any patches against the meta-bbb layer to the xxxx mailing list (xxxx@zzzz.org)
and cc: the maintainer:

Maintainer: XXX YYYYYY <xxx.yyyyyy@zzzzz.com>

Table of Contents
=================

  I. Adding the meta-bbb layer to your build
 II. Misc


I. Adding the meta-bbb layer to your build
=================================================

Run 'bitbake-layers add-layer meta-bbb'

II. Misc
========

--- replace with specific information about the meta-bbb layer ---
```

You will want to go in here and edit all the necessary information before
pushing this to your source control of choice.

#### conf/layers.conf

This is the configuration file for your layer. It defines metadata about the
layer itself and provides BitBake with instructions on how to use it.
It looks like the following:

```sh
# We have a conf and classes directory, add to BBPATH
BBPATH .= ":${LAYERDIR}"

# We have recipes-* directories, add to BBFILES
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
            ${LAYERDIR}/recipes-*/*/*.bbappend"

BBFILE_COLLECTIONS += "meta-bbb"
BBFILE_PATTERN_meta-bbb = "^${LAYERDIR}/"
BBFILE_PRIORITY_meta-bbb = "6"

LAYERDEPENDS_meta-bbb = "core"
LAYERSERIES_COMPAT_meta-bbb = "kirkstone"
```

1. `BBPATH`: Extends the BitBake search path to include this layer directory.
2. `BBFILES`: Tells BitBake to look for any `.bb` or `.bbappend` files that
              may exist in any folders prefaced with `recipes-`.
3. `BBFILE_COLLECTIONS`: Registers this layer under the name `meta-bbb` so it
                         can be referenced elsewhere in the build system.
4. `BBFILE_PATTERN_meta-bbb`: Defines the file path pattern for this layer.
                              BitBake uses this to confirm that recipes belong
                              to the `meta-bbb` layer.
5. `BBFILE_PRIORITY_meta-bbb`: Sets the layer's priority relative to other
                               layers. Higher numbers take precedence when
                               multiple layers provide the same recipe or config.
6. `LAYERDEPENDS_meta-bbb`: Declares dependencies for this layer. In this case,
                            it depends on the core layer, which is a required
                            base layer for all Yocto builds.
7. `LAYERSERIES_COMPAT_meta-bbb`: Specifies the Yocto release(s) this layer is
                                  compatible with. Here, it's set to kirkstone,
                                  meaning this layer was designed and tested
                                  for the Kirkstone release.

This is probably the most critical file in your meta layer. It is the entry
point for integrating your layer with BitBake, and it is imperative that this
file is in good working order before anything else can proceed.

#### recipes-example/example/example_0.1.bb

This is a sample BitBake recipe included to demonstrate the directory structure
and recipe syntax. It's not functional by default but serves as a starting
point for creating your own recipes. It looks as follows:

```sh
SUMMARY = "bitbake-layers recipe"
DESCRIPTION = "Recipe created by bitbake-layers"
LICENSE = "MIT"

python do_display_banner() {
    bb.plain("***********************************************");
    bb.plain("*                                             *");
    bb.plain("*  Example recipe created by bitbake-layers   *");
    bb.plain("*                                             *");
    bb.plain("***********************************************");
}

addtask display_banner before do_build
```

This recipe simply defines a custom task that outputs a text banner during the
BitBake build process. Nothing gets added to the actual rootfs with this.

### 3. Add the Layer to the Build

Now that we created a new meta layer, we need to add it to our build. Again,
there are two ways to do this. We'll do it the automatic way and explain what
is going on underneath after

> **Note:** This script requires you to be in the `build` directory, but if you
> noticed where we created the meta layer in the previous step, we
> created it in the `poky` directory with the other meta layers.
> If you attempt to run this as shown in most tutorials, the script
> will complain that it cannot find the meta layer. You always want
> to provide the path of the meta layer to the `add-layers` script
> and not simply the name of the meta layer.

```sh
cd ~/bbb-example/poky/build
bitbake-layers add-layer ../meta-bbb
```

BitBake will say the following: `NOTE: Starting bitbake server...`.

Under the hood, BitBake is simply adding your new meta layer to the `BBLAYERS`
variable in the `conf/bblayers.conf` file:

```sh
ice@wsl2:build(kirkstone)$ cat conf/bblayers.conf
# POKY_BBLAYERS_CONF_VERSION is increased each time build/conf/bblayers.conf
# changes incompatibly
POKY_BBLAYERS_CONF_VERSION = "2"

BBPATH = "${TOPDIR}"
BBFILES ?= ""

BBLAYERS ?= " \
  /home/ice/bbb-example/poky/meta \
  /home/ice/bbb-example/poky/meta-poky \
  /home/ice/bbb-example/poky/meta-yocto-bsp \
  /home/ice/bbb-example/poky/meta-bbb \ # Added this line here
  "
```

Now that we have our own custom layer integrated into our build system, we can
start to build a proper custom image with our own packages and configurations.

## Layer Customization

The Yocto Project has tons of possible pre-written and compatible recipes for
components we can pull into our image. Any file you see with a `.bb` is
essentially fair game to add to our system. You can take a look at the possible
options [here](https://git.yoctoproject.org/poky/tree/meta?h=kirkstone).

Let's do something relatively simple, but a little more interactive than the
last tutorial. We'll add [`lighttpd`](https://git.yoctoproject.org/poky/tree/meta/recipes-extended/lighttpd?h=kirkstone)
and a custom `index.html` that can be seen when you visit the server's IP.

### 1. Create a Custom Image

We'll contain everything in a custom image that we'll name `mycustom-image`.
From now on, instead of calling `core-image-minimal` or similar, we can call
our custom version of the image we are creating. It will allow us to break away
from the predefined images Yocto provides and move towards a truly custom image.

```sh
cd ~/bbb-example/poky
mkdir -p meta-bbb/recipes-images/images
touch meta-bbb/recipes-images/images/mycustom-image.bb
```

Open up the newly created `mycustom-image.bb` and add the following:

```sh
# A short summary of the recipe
SUMMARY = "My Custom BBB Image with Lighttpd"
# Our license we are applying. Can be MIT, BSD, CLOSED, etc.
# NOTE: Because we are inheriting from core-image down below,
#       this will automatically inherit the core-image's license.
LICENSE = "MIT"
# Use core-image as our framework to simply the image process
inherit core-image
# Packages we are going to install on top of core-image:
#   lighttpd is the web server
#   lighttpd-module-access gives us some additional controls over lighttpd
#   logrotate prevents logs from consuming too much disk space
#   lighttpd-custom-files is a package we will create to store our splash screen
# NOTE: You can add vim, git, etc here, as well!
IMAGE_INSTALL += "lighttpd lighttpd-module-access logrotate lighttpd-custom-files"
```

### 2. Tweak Lighttpd

Before we begin, we need to find out where to place our custom `index.html`.
Looking at [`the documentation`](https://redmine.lighttpd.net/projects/lighttpd/wiki/Docs_ConfigurationOptions)
that `lighttpd` provides, `server.document-root` is the root of the filesystem
path which to serve requests. So this is our entry point where the `index.html`
should reside. Let's see what's in the `lighttpd` recipe:

```sh
ice@wsl2:lighttpd(kirkstone)$ tree
.
├── lighttpd
│   ├── index.html.lighttpd
│   ├── lighttpd
│   └── lighttpd.conf
└── lighttpd_1.4.67.bb
```

We can see they provide their own `lighttpd.conf` file, their own `index.html`,
and a shell script to handle kicking off `lighttpd`. Let's leave their
`index.html` page alone and create our own. But first, we need to check the
location of where `server.document-root` is pointing to:

```sh
ice@wsl2:lighttpd(kirkstone)$ cat lighttpd/lighttpd.conf | grep -i server.document-root
server.document-root        = "/www/pages/"
```

This is pointing to `/www/pages/`. To more closely match the tutorial that
`lighttpd` provides, let's change this to point to `/var/www/` instead. Since
the goal is to do all the customization possible in our own layer, let's head
back to our layer and make some changes to this `lighttpd` recipe. In order
to do that, we will use a `.bbappend` that will do a text replacement of this
line. We need to mirror `lighttpd`'s location, so let's go back to our layer
and create an analogous space and `.bbappend` to tweak this:

```sh
cd ~/bbb-example/poky/meta-bbb
mkdir -p recipes-extended/lighttpd
touch recipes-extended/lighttpd/lighttpd_%.bbappend
```

We are using the `%` wildcard so that this is applicable to any version of
`lighttpd` that might come after. This allows us to not care about the exact
version of `lighttpd` when applying this change. As always, this can be a
double-edged sword, so be careful when creating `.bbappend` files like this.

Add the following to the `lighttpd_%.bbappend`:

```sh
do_install:append () {
    # Adjust the document root in lighttpd.conf from /www/pages to /var/www
    sed -i 's|"/www/pages/"|"/var/www"|' ${D}${sysconfdir}/lighttpd/lighttpd.conf
}
```

Here, we are adding another statement after the `do_install()` function that
performs this `sed` command at the very end. This effectively changes the
location the `server.document-root` points to so we can more closely follow
along with the `lighttpd` tutorial.

### 3. Create a Custom Package

With our meta layer created and `lighttpd`'s path tweaked, we can finally
create our own custom recipe that will place an `index.html` file in the
`/var/www/` path that `lighttpd` will serve.

```sh
cd ~/bbb-example/poky/meta-bbb
mkdir -p recipes-example/lighttpd-custom-files/files
touch recipes-example/lighttpd-custom-files/lighttpd-custom-files.bb
touch recipes-example/lighttpd-custom-files/files/index.html
```

Edit the newly created `lighttpd-custom-files.bb` and add the following:

```sh
# Summary of the recipe
SUMMARY = "Test webpage files for Lighttpd"
# Our license we will use for this
LICENSE = "MIT"
# The location and hash of our MIT license
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"
# The location of where the index.html file was placed
# NOTE: BitBake will automatically look for a "files" directory in an attempt
#       to locate the files listed in the SRC_URI path
SRC_URI = "file://index.html"
# Our source directory
S = "${WORKDIR}"
# The install action. This mimics RPM install syntax
do_install() {
    install -d ${D}/var/www # Installing the directory /var/www
    install -m 0644 ${WORKDIR}/index.html ${D}/var/www/index.html # Installing the file index.html in /var/www
}
# The file(s) we want to add to the recipe to be installed
FILES_${PN} = "/var/www/*"
```

#### Licensing

Licensing is actually a big enough topic that it needed its own section.
BitBake requires all source have an appropriate license. In fact, if it cannot
find one, it will error out and you will not be able to complete the build.
There is an entire writeup on licenses [here](https://docs.yoctoproject.org/dev-manual/licenses.html).

However, I find it more practical to learn by example.

##### Using Predefined Licenses

The Yocto Project provides a large collection of predefined licenses in the
`meta/files/common-licenses` directory, referenced by the variable `${COMMON_LICENSE_DIR}`.
According to the [reference manual](https://docs.yoctoproject.org/ref-manual/variables.html#term-COMMON_LICENSE_DIR),
this directory contains many commonly used open-source licenses.

Let's inspect the directory:

```sh
ice@wsl2:common-licenses(kirkstone)$ ls -l
total 5096
-rw-r--r--   1 ice ice   643 Dec  7 17:37 0BSD
-rw-r--r--   1 ice ice  2526 Dec  7 17:37 AAL
-rw-r--r--   1 ice ice   488 Dec  7 17:37 ADSL
-rw-r--r--   1 ice ice  4676 Dec  7 17:37 AFL-1.1
-rw-r--r--   1 ice ice  6718 Dec  7 17:37 AFL-1.2
-rw-r--r--   1 ice ice  9014 Dec  7 17:37 AFL-2.0
-rw-r--r--   3 ice ice  8934 Dec  7 17:37 AFL-2.1
-rw-r--r--   1 ice ice 10327 Dec  7 17:37 AFL-3.0
<snipped>
```

You'll see a vast collection of licenses available for use, including `MIT`,
`GPL-2.0`, `GPL-3.0`, and many others.

##### Setting `LIC_FILES_CHKSUM`

To use one of these licenses in your recipe, you need to provide three
pieces of information for the `LIC_FILES_CHKSUM` variable:

1. The **file path** to the license file (e.g., `${COMMON_LICENSE_DIR}/MIT`).
2. The **hash algorithm** (Yocto uses `md5` by default).
3. The **checksum** of the license file (calculated using `md5sum`).

Let's calculate the MD5 checksum for the `MIT` license:

```sh
ice@wsl2:common-licenses(kirkstone)$ md5sum MIT
0835ade698e0bcf8506ecda2f7b4f302  MIT
```

Now that we have the path (`${COMMON_LICENSE_DIR}/MIT`), the algorithm (`md5`),
and the checksum (`0835ade698e0bcf8506ecda2f7b4f302`), we can use them in the
`LIC_FILES_CHKSUM` field in our recipe:

```sh
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"
```

This now ensures BitBake can verify the license for compliance during the build.

##### Closed Source Licenses

If you're working with closed-source software, set the `LICENSE` field to
`"CLOSED"` and omit `LIC_FILES_CHKSUM`. This tells BitBake that the recipe
uses proprietary code and skips the license checksum verification.

For example:

```sh
LICENSE = "CLOSED"
```

And that's it! It's a little more involved than simply adding a typical
`LICENSE` to the root of a source directory, but it isn't terrible once you
get the hang of it.

### 4. Customize the HTML

Next, edit the `index.html` and populate it with some basic HTML.
We'll go with a basic one for now:

```html
<!DOCTYPE html>
<html>
<head><title>BBB Lighttpd Test</title></head>
<body><h1>Hello from the BeagleBone Black! Built with Yocto!</h1></body>
</html>
```

### 5. Building

Your final directory structure should look like this in your meta layer:

```sh
ice@wsl2:meta-bbb(kirkstone)$ tree
.
├── COPYING.MIT
├── README
├── conf
│   └── layer.conf
├── recipes-example
│   ├── example
│   │   └── example_0.1.bb
│   └── lighttpd-custom-files
│       ├── files
│       │   └── index.html
│       └── lighttpd-custom-files.bb
├── recipes-extended
│   └── lighttpd
│       └── lighttpd_%.bbappend
└── recipes-images
    └── images
        └── mycustom-image.bb
```

If everything looks good, we can tell BitBake to build our new custom image.
Since we did a few things and moved around directories a bit, we'll do some
quick sanity checks before building:

```sh
cd ~/bbb-example/poky/
source oe-init-build-env
# We should now be back in the `build/` directory
bitbake mycustom-image
```

BitBake should now successfully build your newly created image! This image will
contain the `lighttpd` server, a couple extra nice-isms, and a customized
configuration file for `lighttpd` that will point to our newly created
`index.html` file. The server should start on boot automatically, too.

### 6. Deploying

Write your newly created `.wic` file, located in `tmp/deploy/images/beaglebone-yocto`,
to a MicroSD card as described in the previous blog.

Now, grab your BeagleBone Black (BBB), and follow these steps:

1. Connect an Ethernet cable to the RJ45 port on the BBB.
2. Plug the other end of the Ethernet cable into your home network (via a router or switch).
3. Insert the MicroSD card with your new image into the BBB's card slot.
4. Attach a keyboard and monitor to the BBB for direct interaction.
5. Hold the **S2** button on the BBB.
6. While holding the S2 button, apply power to the board.
7. After 5 seconds, release the S2 button and watch the board boot from the MicroSD card.

The login credentials are the same as before:

- **Username:** `root`
- **Password:** *(leave blank)*

On the BBB, find its IP address by running the following command in the
terminal:

```sh
ip addr show | grep eth0`
```

Look for the `inet` entry corresponding to the Ethernet interface. This will
typically be an IP address in the `192.x.x.x`, `172.x.x.x`, or `10.x.x.x` range.

From another machine on the same network as the BBB, open a web browser and
navigate to the BBB's IP address. You should see your custom `index.html`
page being served from the BeagleBone Black.

## Wrapping Up

You've successfully created and deployed a custom Yocto-based Linux image for
the BeagleBone Black, complete with a working `lighttpd` web server and your
custom web page. By organizing your customizations into a dedicated meta layer,
you’ve laid the groundwork for a scalable and maintainable development process.

Here are a couple possible projects you can work on to expand on this:

- Expand the functionality of your server by adding more `lighttpd` modules
  or other services.
- Experiment with new packages or custom scripts to tailor the image further
  to your needs.
- Exploring Yocto's advanced features, such as overlays and multiconfig.

We may pick up on this in the next blog post. Or we may pivot, who knows!

## Further Reading

Some useful links that helped me when I first started out:

- [Yocto Project Quick Build](https://docs.yoctoproject.org/brief-yoctoprojectqs/index.html)
- [Yocto Mega Manual](https://docs.yoctoproject.org/singleindex.html)
- [BeagleBone Black Docs](https://docs.beagle.cc/boards/beaglebone/black/index.html)
- [Lighttpd Tutorial](https://redmine.lighttpd.net/projects/lighttpd/wiki/TutorialConfiguration)

Happy building!
