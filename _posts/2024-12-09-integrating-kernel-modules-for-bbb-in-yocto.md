---
title: Integrating Kernel Modules for the BeagleBone Black in Yocto
description: Learn to create and integrate drivers in a Yocto-based system
date: 2024-12-09 00:00:00 -0500
categories: [Programming, C, Device Drivers, Kernel Modules, Linux, Embedded Linux, Yocto]
tags: [linux, embedded linux, scripting]
---

In the [last tutorial](https://tomice.github.io/posts/creating-custom-layer-for-bbb/),
we created a custom layer called `meta-bbb` for our BeagleBone Black complete
with a `lighttpd` web server serving our own custom HTML page. This approach
gave us more control over the image and what gets placed on the rootfs instead
of shoehorning all of our changes into the `conf/local.conf`. It allows for our
code to be more portable, easier to share, and easier to work with.

Let's extend this concept and start working with some core embedded Linux
fundamentals. In this article, we'll extend our Yocto project by adding a
custom kernel module and a userspace application to interact with it.
We will build the code from scratch and write all the recipes ourselves.

## Device Driver Categories

Before we begin programming, we should set the stage on the two different
categories of kernel modules for Linux. There are static kernel modules and
dynamically loadable kernel modules. Each has its place depending on what
the designer is looking to do with the system, but some may make more sense
than others depending on the use case.

### Static Kernel Modules

Static kernel modules are built directly into the Linux kernel image.
This means the module is compiled and linked as part of the kernel itself and
cannot be dynamically removed or added during runtime. Static modules are
always present and initialized when the system boots.

**Advantages:**

- **Reliability:** Because they are part of the kernel image, static modules
                   are loaded and initialized as the kernel boots, reducing
                   potential runtime errors or missing dependencies.
- **Performance:** Static modules eliminate the slight overhead of dynamically
                   loading a module at runtime, albeit at the possible expense
                   of boot times.
- **Simplified Deployment:** No need to manage the loading or unloading of
                             modules as they are always available.

**Disadvantages:**

- **Increased Kernel Size:** Embedding modules directly into the kernel
                             increases the size of the kernel image, which may
                             be an issue for systems with limited memory.
- **Reduced Flexibility:** Static modules cannot be unloaded or updated without
                           recompiling and rebooting the kernel.

**When to Use:**

- When the module is critical for system operation and must always be available
- For systems with stringent performance requirements
- In environments where dynamic module management is not desirable,
  such as tightly controlled embedded systems

### Loadable Kernel Modules

Loadable kernel modules (LKMs) are compiled separately from the kernel and can
be dynamically loaded and unloaded at runtime without rebooting the system.
LKMs are the most commonly implemented approach for developing and deploying
kernel modules to modern desktop-based Linux systems.

**Advantages:**

- **Flexibility:** LKMs can be added or removed as needed, making them ideal
                   for testing, debugging, or adding features to a running
                   kernel.
- **Smaller Kernel Image:** The kernel image remains smaller since modules are
                            not embedded.
- **Modular Design:** Developers can isolate specific functionality into
                      separate modules, which simplifies development and
                      maintenance.

**Disadvantages:**

- **Runtime Overhead:** Loading modules at runtime incurs a slight overhead
                        compared to static modules.
- **Dependency Management:** LKMs may depend on other modules or specific
                             kernel features, which need to be loaded and
                             configured correctly.

**When to Use:**

- For optional features that aren't always required
- During development, when frequent testing and debugging are necessary
- In general-purpose systems where different modules may be required for
  different tasks

In our case, it is probably more reasonable to create a static kernel module
instead of a dynamic one. However, I will show how to use the same driver code
and be able to build both with some minor tweaks in Yocto.

## Classes of Linux Device Drivers

For this tutorial, we'll touch on the three main classes of Linux device
drivers that serve as the fundamental building blocks for most of
the features the kernel provides.

### Character Device Driver

Character device drivers are responsible for managing devices that handle data
sequentially, byte by byte. These devices typically do not allow random access
to data but instead process data in a continuous stream. When you interact with
a character device, the data is read or written one byte at a time, similar to
reading or writing text on a screen or sending data through a communication
channel.

Character devices are typically accessed via special files in `/dev`.
For example, `/dev/tty` represents terminal devices, and `/dev/null` is a
special character device that discards all data written to it.

Some examples of character devices include:

- Terminals (TTYs)
- Serial ports (e.g., RS232)
- Memory devices
- Sound cards
- USB devices

These devices interact with the kernel through character-based I/O operations.

### Block Device Driver

Block device drivers manage hardware that stores data in fixed-size blocks,
which can be accessed in any order. Unlike character devices, block devices are
designed for random access, meaning data can be read or written from any block
without processing it sequentially. This makes block devices ideal for data
storage applications, as they allow fast and efficient access to large volumes
of data.

Some examples of block devices include:

- Hard drives (HDDs/SSDs)
- Optical drives (e.g., CD/DVD/Blu-ray)
- Flash drives
- Memory cards

Block devices also have special files in `/dev`, such as `/dev/sda` for disk
drives, enabling the kernel and applications to interact with the underlying
hardware.

### Network Device Driver

Network device drivers enable communication between the system and the network.
These drivers manage the hardware responsible for sending and receiving data
over various types of networks, such as wired Ethernet connections, wireless
networks, or virtual network interfaces. Network drivers translate high-level
networking protocols into commands the hardware can understand.

Examples of network devices include:

- Network Interface Cards (NICs),
- Wi-Fi adapters,
- Bluetooth devices,
- Virtual network interfaces (e.g., `tun` or `tap` devices).

Network drivers are critical for facilitating communication across physical and
virtual networks and are a cornerstone of modern computing systems.

In this tutorial, we'll focus on creating a **character device driver** because
it is one of the easiest types to understand and implement within a typical
Linux system. It provides a straightforward introduction to kernel programming
while illustrating key concepts such as interaction between user space and
kernel space.

## Creating the Character Device Driver

Let's create a kernel module that behaves as a simple character device.
It will have the following functionality:

- **Writes:** Accepts user-provided messages, updates an internal message
              buffer, and increments a counter.
- **Reads:** Returns the current counter value and the stored message.

### 1. Setup Our Environment

Make sure we have a proper environment and starting point:

```sh
cd ~/bbb-example/poky/
source oe-init-build-env
cd ../meta-bbb
```

We should now have the BitBake environment sourced and be in the `meta-bbb`
directory that we created in the previous tutorial.

### 2. Create the Driver

Create a new directory in your meta layer for kernel recipes:

```sh
mkdir -p recipes-kernel/example-char-driver/files
touch recipes-kernel/example-char-driver/files/example_char_dev.c
```

Open up the `example_char_dev.c` file and populate it with the following:

```c
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/uaccess.h>
#include <linux/slab.h>
#include <linux/mutex.h>  // for mutex_lock and mutex_unlock

#define DEVICE_NAME "example_char"
#define BUFFER_SIZE 1024

static char *message;
static int counter;
static dev_t dev_num;
static struct cdev example_cdev;
static DEFINE_MUTEX(example_mutex);  // Mutex for protecting shared resources

static ssize_t example_read(struct file *file, char __user *buf, size_t count, loff_t *ppos) {
    char output[BUFFER_SIZE];
    int len;
    
    // Locking shared resources
    mutex_lock(&example_mutex);

    len = snprintf(output, BUFFER_SIZE, "Counter: %d, Message: %s\n", counter, message);

    // Unlocking after use
    mutex_unlock(&example_mutex);

    // Make sure we don't exceed the user buffer size
    return simple_read_from_buffer(buf, count, ppos, output, len);
}

static ssize_t example_write(struct file *file, const char __user *buf, size_t count, loff_t *ppos) {
    if (count > BUFFER_SIZE - 1) return -EINVAL;

    // Locking shared resources
    mutex_lock(&example_mutex);

    if (copy_from_user(message, buf, count)) {
        mutex_unlock(&example_mutex);
        return -EFAULT;
    }

    message[count] = '\0';
    counter++;

    // Unlocking after use
    mutex_unlock(&example_mutex);
    return count;
}

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .read = example_read,
    .write = example_write,
};

static int __init example_init(void) {
    int ret;

    message = kzalloc(BUFFER_SIZE, GFP_KERNEL);
    if (!message) return -ENOMEM;

    ret = alloc_chrdev_region(&dev_num, 0, 1, DEVICE_NAME);
    if (ret < 0) {
        kfree(message);
        return -EBUSY;
    }

    cdev_init(&example_cdev, &fops);
    ret = cdev_add(&example_cdev, dev_num, 1);
    if (ret < 0) {
        unregister_chrdev_region(dev_num, 1);
        kfree(message);
        return -EBUSY;
    }

    printk(KERN_INFO "example_char: loaded with major %d\n", MAJOR(dev_num));
    return 0;
}

static void __exit example_exit(void) {
    cdev_del(&example_cdev);
    unregister_chrdev_region(dev_num, 1);
    kfree(message);
    printk(KERN_INFO "example_char: unloaded\n");
}

module_init(example_init);
module_exit(example_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Tom Ice"); // Feel free to add your name here
MODULE_DESCRIPTION("Example Character Device Driver");
```

Great! We have a basic character device driver that we can now integrate into
our system. Next up is the userspace application to interact with
`/dev/example_char`.

## Creating the Userspace Application

We created a device driver, but it currently lives in kernel space. In order
for users to interact with it, we need an application in userspace that
understands how the device driver works.

### 1. Setup Area

Let's go back to where we started and create the work area:

```sh
cd ~/bbb-example/poky/meta-bbb
mkdir -p recipes-example/example-char-user/files
touch recipes-example/example-char-user/files/example_char_user.c
```

### 2. Populate the Userspace Application Source

Open up `example_char_user.c` and populate it with the following:

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>

#define DEVICE "/dev/example_char"

int main(int argc, char **argv) {
    char buffer[1024];
    int fd;
    ssize_t bytes;
    size_t write_length;
    ssize_t bytes_written;

    // Open the device file
    fd = open(DEVICE, O_RDWR);
    if (fd < 0) {
        perror("Failed to open device");
        return EXIT_FAILURE;
    }

    // If no argument is given, print a usage hint
    if (argc < 2) {
        fprintf(stderr, "No input provided. You can provide a string argument to write.\n");
        fprintf(stderr, "Usage: %s [data_to_write]\n", argv[0]);
    } else {
        // Ensure the input isn't too large for the device buffer
        write_length = strlen(argv[1]);
        if (write_length > sizeof(buffer) - 1) {
            fprintf(stderr, "Input too long, maximum length is %zu characters.\n", sizeof(buffer) - 1);
            close(fd);
            return EXIT_FAILURE;
        }

        // Write the user argument to the device
        bytes_written = write(fd, argv[1], write_length);
        if (bytes_written < 0) {
            perror("Failed to write to device");
            close(fd);
            return EXIT_FAILURE;
        }
    }

    // Read the output from the device
    bytes = read(fd, buffer, sizeof(buffer) - 1);
    if (bytes < 0) {
        perror("Failed to read from device");
        close(fd);
        return EXIT_FAILURE;
    }

    // Null-terminate the buffer and print the result
    buffer[bytes] = '\0';
    printf("Device Output: %s\n", buffer);

    // Close the device file
    close(fd);
    return EXIT_SUCCESS;
}
```

With our userspace application written, we can now plumb both the userspace
application and kernel module into the Yocto build system so they get compiled
into our rootfs. Let's start with the userspace application first since that
remains the same regardless of whether it's a static module or LKM.

### 3. Creating the Userspace Recipe

While in the `meta-bbb` repository, create the following file:

```sh
touch recipes-example/example-char-user/example-char-user.bb
```

Open `example-char-user.bb` and populate it with the following:

```sh
SUMMARY = "Example Character Device User Utility"
LICENSE = "GPL-2.0-only"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/GPL-2.0-only;md5=801f80980d171dd6425610833a22dbe6"

SRC_URI = "file://example_char_user.c"

S = "${WORKDIR}"
DEPENDS = "virtual/libc"

do_compile() {
    ${CC} ${CFLAGS} ${LDFLAGS} example_char_user.c -o example_char_user
}

do_install() {
    install -d ${D}${bindir}
    install -m 0755 example_char_user ${D}${bindir}/example_char_user
}
```

Great! That's all that's needed to make the simple application we have.
Next, we will show how to integrate a loadable kernel module (LKM).

## Adding a Loadable Kernel Module to Yocto

### 1. Create the Makefile

While in the `meta-bbb` repository, create the following file:

```sh
touch recipes-kernel/example-char-driver/files/Makefile
```

Open up the `Makefile` and populate it with the following:

```makefile
obj-m := example_char_dev.o

# NOTE: The spacing must be tabs!
all:
	$(MAKE) -C $(KERNEL_SRC) M=$(PWD) modules

clean:
	$(MAKE) -C $(KERNEL_SRC) M=$(PWD) clean
```

### 2. Create the LKM Recipe

While still in the `meta-bbb` repository, create the following:

```sh
touch recipes-kernel/example-char-driver/example-char-driver.bb
```

Next, open up the `example-char-driver.bb` and populate it with the following:

```sh
SUMMARY = "Example Character Device Driver"
DESCRIPTION = "A simple example character device kernel module"
LICENSE = "GPL-2.0-only"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/GPL-2.0-only;md5=801f80980d171dd6425610833a22dbe6"

inherit module

SRC_URI = "file://example_char_dev.c \
           file://Makefile"

S = "${WORKDIR}"
MODULE_NAME = "example_char_dev"

do_compile() {
    oe_runmake KERNEL_SRC=${STAGING_KERNEL_BUILDDIR} ARCH=${ARCH}
}

do_install() {
    install -d ${D}/lib/modules/${KERNEL_VERSION}
    install -m 0644 ${B}/${MODULE_NAME}.ko ${D}/lib/modules/${KERNEL_VERSION}/
}
```

### 3. Add the Image Hooks

Next, we want to integrate them into our image.
Open up `recipes-images/images/mycustom-image.bb` and add these two new
components:

```sh
IMAGE_INSTALL += "example-char-driver example-char-user"
```

> **Note:** These two components can be added to the same line as before, or
> you can simply place them after the `IMAGE_INSTALL` line from the previous
> tutorial. BitBake will concatenate them all together regardless.

### 4. Build

Now, we build:

```sh
cd ~/bbb-example/poky/
source oe-init-build-env
bitbake mycustom-image
```

If everything went correctly, it should build successfully, and you should have
your new `.wic` in the `tmp/deploy/images/beaglebone-black` folder. Write this
to a MicroSD card and boot like we have been doing throughout this tutorial.

## Testing the LKM

Check if the kernel module is on the system:

```sh
ls /lib/modules/$(uname -r)/example_char_dev*
```

You should see the `.ko` show up here. If so, great! It means it got added to
the rootfs! If you don't see it, double-check to make sure you added it to
the `IMAGE_INSTALL` variable correctly.

Now let's load it up and test it:

```sh
insmod /lib/modules/$(uname -r)/example_char_dev.ko
dmesg | grep "example_char: loaded with major"
# Look for something like example_char: loaded with major 240
mknod /dev/example_char c <major number> 0
example_char_user "Hello, Kernel!"
```

If all went well, you should see something like the following:

```sh
Device Output: Counter: 1, Message: Hello, Kernel!
```

## Creating the Static Kernel Module

There are a handful of ways to get static kernel modules built into the kernel
with Yocto. We'll look into utilizing the patching method as it most closely
resembles how you would apply patches to the Linux kernel source itself.

In order to do this, we will need to rearrange and just how some of these
the components fit together in our `meta-bbb` layer. Let's get started!

### 1. Preparation

In case anyone jumps to this section immediately or didn't follow along with
the LKM implementation was done, we want to make sure we have a sane
environment first. So let's do some quick prep:

```sh
cd ~/bbb-example/poky/
source oe-init-build-env
bitbake linux-yocto -c unpack # Grabs the source
bitbake linux-yocto -c patch # Patches source so it's up-to-date
```

### 2. Rearranging our Workspace

We want to rearrange some of our code now. Because our kernel module will
be built in the source tree of the Linux kernel itself, we want to remove some
references to our old kernel module.

Remove the `example-char-driver` from
`~/bbb-example/poky/meta-bbb/recipes-images/images/mycustom-image.bb` if it exists
in the `IMAGE_INSTALL` section, but leave the `example-char-user` as our
application to interface with the driver will remain the same.

An example of what it should look like is as follows. Note this contains
some functions from the previous tutorial:

```sh
# A short summary of the recipe
SUMMARY = "My Custom BBB Image with Lighttpd"
# Our license we are applying. Can be MIT, BSD, CLOSED, etc.
LICENSE = "MIT"
# Use core-image as our framework to simply the image process
inherit core-image
# Packages we are going to install on top of core-image:
#   lighttpd is the web server
#   lighttpd-module-access gives us some additional controls over lighttpd
#   logrotate prevents logs from consuming too much disk space
#   lighttpd-custom-files is a package we will create to store our splash screen
IMAGE_INSTALL += "lighttpd lighttpd-module-access logrotate lighttpd-custom-files example-char-user"
```

> **Note:** Don't worry about the `example-char-driver.bb` file in `recipes-kernel`.
> You can remove it if you'd like, but because we aren't adding it to
> `IMAGE_INSTALL`, it will not get installed. If you skipped over the LKM
> section, you won't need to worry about this to begin with.

Next, let's move the driver to the checked out source code:

```sh
cd ~/bbb-example/poky/build/tmp/work-shared/beaglebone-yocto/kernel-source/drivers/char/
mv ~/bbb-example/poky/meta-bbb/recipes-kernel/example-char-driver/files/example_char_dev.c . 
```

Our `example_char_dev.c` should now be in the `drivers/char` folder in the
Linux kernel source tree.

```sh
ice@wsl2:char(v5.15/standard/beaglebone)$ ll
total 596
-rw-r--r-- 1 ice ice 16672 Dec  8 23:41 Kconfig
-rw-r--r-- 1 ice ice  1455 Dec  8 23:41 Makefile
-rw-r--r-- 1 ice ice  4564 Dec  8 23:40 adi.c
drwxr-xr-x 2 ice ice  4096 Dec  8 23:40 agp
-rw-r--r-- 1 ice ice 17689 Dec  8 23:40 apm-emulation.c
-rw-r--r-- 1 ice ice 24724 Dec  8 23:40 applicom.c
-rw-r--r-- 1 ice ice  2597 Dec  8 23:40 applicom.h
-rw-r--r-- 1 ice ice  8427 Dec  8 23:40 bsr.c
-rw-r--r-- 1 ice ice  8602 Dec  8 23:40 ds1620.c
-rw-r--r-- 1 ice ice 12496 Dec  8 23:40 dsp56k.c
-rw-r--r-- 1 ice ice 16685 Dec  8 23:40 dtlk.c
-rw-r--r-- 1 ice ice  2398 Dec  8 23:41 example_char_dev.c
<snipped>
```

### 3. Adding Supporting files

We have the driver in the `drivers/char` folder, but we need to make sure the
kernel source understands it, so we'll need to create a few more files to tell
the source tree this new driver exists. This should follow standard Linux
kernel dev processes, so anybody familiar with kernel development should be
familiar with this, but if you're new, we'll outline all the steps necessary.

While inside the `drivers/char` folder, edit the `Kconfig` file:

```sh
vim Kconfig
```

Add the following to the end right before the `endmenu` statement:

```sh
config EXAMPLE_CHAR
    bool "Example Character Driver"
    default y
    help
      A simple example character driver that is built into the kernel.
```

It will look like this:

```sh
<snipped>
      Say Y here unless you have reason to mistrust your bootloader or
      believe its RNG facilities may be faulty. This may also be configured
      at boot time with "random.trust_bootloader=on/off".

config EXAMPLE_CHAR
    bool "Example Character Driver"
    default y
    help
      A simple example character driver that is built into the kernel.

endmenu
```

Our default is `y` to make sure we build this into the kernel as a static
module instead of a loadable one. Next, open up the `Makefile` in
`drivers/char` and add the following as the very last line:

```sh
obj-$(CONFIG_EXAMPLE_CHAR) += example_char_dev.o
```

This will tell the kernel's build system to compile `example_char_dev.c` into
the kernel when `CONFIG_EXAMPLE_CHAR` is enabled.

### 4. Creating the Patch

We now have our driver in the `drivers/char` folder with adjustments to the
`Kconfig` and `Makefile` so the kernel can recognize our example driver. Next
is to create our patch. We'll do this with the power of `git`.

To make pathing easier to understand, let's do this at the top level of the
kernel source before we begin:

```sh
cd ../.. # We should be in kernel-source now
```

Check our repo and add our files:

```sh
# Verify that only our files have been touched:
git status
On branch v5.15/standard/beaglebone
Your branch is behind 'origin/v5.15/standard/beaglebone' by 4584 commits, and can be fast-forwarded.
  (use "git pull" to update your local branch)

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   drivers/char/Kconfig
        modified:   drivers/char/Makefile

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        drivers/char/example_char_dev.c

no changes added to commit (use "git add" and/or "git commit -a")

# Add our files:
git add .
```

Verify the files got added to the staging area and commit them:

```sh
# Sanity check our staged changes:
git status
On branch v5.15/standard/beaglebone
Your branch is behind 'origin/v5.15/standard/beaglebone' by 4584 commits, and can be fast-forwarded.
  (use "git pull" to update your local branch)

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        modified:   drivers/char/Kconfig
        modified:   drivers/char/Makefile
        new file:   drivers/char/example_char_dev.c

# Commit our files:
git commit -m "Add example char driver"
[v5.15/standard/beaglebone be34b88336ee] Add example char driver
 3 files changed, 103 insertions(+)
 create mode 100644 drivers/char/example_char_dev.c
```

Now, we want to create a patch that can be applied to the kernel source.
There are various ways to do this, but probably the easiest way is done
via `git`.

```sh
git format-patch -1 HEAD
```

You should now have a `0001-Add-example-char-driver.patch` file in `kernel-source`:

```sh
ice@wsl2:kernel-source(v5.15/standard/beaglebone)$ ll
total 896
-rw-r--r--   1 ice ice   4033 Dec  9 18:30 0001-Add-example-char-driver.patch
-rw-r--r--   3 ice ice    496 Dec  8 23:40 COPYING
-rw-r--r--   1 ice ice 100996 Dec  8 23:40 CREDITS
drwxr-xr-x  82 ice ice   4096 Dec  8 23:40 Documentation
<snipped>
```

### 5. Adding the Patch to Yocto

We have our patch we created residing in the `kernel-source` directory,
but we need this in our meta layer. Let's relocate this to our `meta-bbb` layer:

```sh
mkdir -p ~/bbb-example/poky/meta-bbb/recipes-kernel/linux/files
mv 0001-Add-example-char-driver.patch ~/bbb-example/poky/meta-bbb/recipes-kernel/linux/files
```

Next, let's create a `.bbappend` to pull in the patch:

```sh
touch ~/bbb-example/poky/meta-bbb/recipes-kernel/linux/linux-yocto_%.bbappend
```

Edit the file to look like this:

```sh
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

SRC_URI += "file://0001-Add-example-char-driver.patch"
```

As you can see, it's a super simple `.bbappend` file pointing to the location
of our patch. Yocto should take care of the rest.

> **Note**: It is best practice to actually include a `.cfg` file together with
> any changes you make. In this case, it is technically optional as our driver
> does not depend on any other driver, and it defaults to `y` in the `Kconfig`.
> But for best practices, you should create something like an
> `example_char.cfg` file that contains `CONFIG_EXAMPLE_CHAR=y` in it and
> place it in the `files` directory. Your `.bbappend` will then look like this:
>
> ```sh
> FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
>
> SRC_URI += "file://0001-Add-example-char-driver.patch \
>             file://example_char.cfg"
>
> KERNEL_CONFIG_FRAGMENTS += "example_char.cfg"
>```

If you have been following along since the beginning,
your `meta-bbb` source tree should look like this:

```sh
ice@wsl2:meta-bbb(kirkstone)$ cd ~/bbb-example/poky/meta-bbb
ice@wsl2:meta-bbb(kirkstone)$ tree
.
├── COPYING.MIT
├── README
├── conf
│   └── layer.conf
├── recipes-example
│   ├── example
│   │   └── example_0.1.bb
│   ├── example-char-user
│   │   ├── example-char-user.bb
│   │   └── files
│   │       └── example_char_user.c
│   └── lighttpd-custom-files
│       ├── files
│       │   └── index.html
│       └── lighttpd-custom-files.bb
├── recipes-extended
│   └── lighttpd
│       └── lighttpd_%.bbappend
├── recipes-images
│   └── images
│       └── mycustom-image.bb
└── recipes-kernel
    └── linux
        ├── files
        │   └── 0001-Add-example-char-driver.patch
        └── linux-yocto_%.bbappend
```

The important folders that should match are:

- `recipes-example/example-char-user`
- `recipes-kernel`
- `recipes-images`

### 6. Building

With our folder structure looking solid, we can go ahead and build.

```sh
cd ~/bbb-example/poky
# Optional if this is still in your path
source oe-init-build-env
bitbake -f -c cleanall linux-yocto
# NOTE: BitBake will provide a WARNING if you followed this tutorial from the
# beginning. This is fine and you can ignore it.
bitbake linux-yocto
bitbake mycustom-image
```

If all goes well, you should have a new `.wic` file you can load onto your
BeagleBone Black! Remember, it's in `tmp/deploy/images/beaglebone-yocto`.

## Testing the Static Kernel Module

Previously, we had to do a manual install of the kernel module with `insmod`.
Now that our kernel module is built into the source itself, we should be
able to see it in `dmesg` itself:

```sh
dmesg | grep "example_char"
# Should print out example_char: loaded with major 247 or similar
```

Enumerating `/dev/example_char` is similar to last time:

```sh
mknod /dev/example_char c <major number> 0
example_char_user "Hello, Kernel!"
```

If all went well, you should see something like the following:

```sh
Device Output: Counter: 1, Message: Hello, Kernel!
```

And that's it! You now know how to build a loadable kernel module or a static
kernel module using Yocto.

You may be wondering if there's a way to prevent the need for `mknod` and have
the kernel module initialize itself at boot like some other modules. It is
indeed possible, but I'll leave that exercise to the reader. 😉

## Wrapping Up

You should now have a good understanding of how to integrate a basic Linux
driver into Yocto, both from the kernel driver side and the userspace side.
We've covered two different approaches to integration, depending on how you
want your driver to be incorporated into the system. Be sure to check out the
official Yocto documentation for kernel developers for more in-depth
information on `.cfg` files, `.scc` files, and other related topics, as these
will be important concepts to explore further in your development journey with
embedded platforms.

## Further Reading

Some useful links that helped me when I first started out:

- [Yocto Linux Kernel Dev Manual](https://docs.yoctoproject.org/kernel-dev/index.html)
- [Linux Device Drivers](https://lwn.net/Kernel/LDD3/)
- [Linux Device Classes](https://www.kernel.org/pub/linux/kernel/people/mochel/doc/text/class.txt)
- [Using devtmpfs for /dev](https://unix.stackexchange.com/questions/77933/using-devtmpfs-for-dev)
- [Udev](https://wiki.archlinux.org/title/Udev)

Happy building!
