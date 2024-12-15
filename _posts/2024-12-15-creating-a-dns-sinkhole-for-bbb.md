---
title: Creating a DNS Sinkhole for the BeagleBone Black in Yocto
description: Learn to create a homemade Pi-Hole for a Yocto-based system
date: 2024-12-15 00:00:00 -0500
categories: [Programming, Shell Scripting, PHP, Linux, Embedded Linux, Yocto]
tags: [linux, embedded linux, scripting, networking]
---

We took a slight detour in the
[last tutorial](https://tomice.github.io/posts/integrating-kernel-modules-for-bbb-in-yocto/)
where we learned what Linux kernel modules are, the different types of modules,
as well as how to implement both dynamic and static kernel modules into a
Yocto-based build.

For this tutorial, let's circle back to when we added the `lighttpd` module to
the system and create our own custom
[DNS sinkhole](https://en.wikipedia.org/wiki/DNS_sinkhole).

We will explore using `lighttpd` to serve our own custom PHP-based site that
will monitor blocked and allowed domains.
[`dnsmasq`](https://thekelleys.org.uk/dnsmasq/doc.html) will do the bulk of the
work for us based on a blocklist that we provide it. All of this will be done
inside of Yocto, and it will output a custom disk image that is immediately
deployable to any BeagleBone Black. Let's get started!

## DNS Sinkhole

A DNS sinkhole is a cybersecurity technique used to redirect DNS queries for
specific domains - often malicious or unwanted ones - to a non-existent or
controlled IP address, typically 127.0.0.1 or another local address.
This redirection effectively prevents access to these domains, offering a
robust defense against malware, phishing attempts, and other online threats.
The concept of DNS sinkholing originated as a method for network administrators
to combat malicious activities by intercepting and redirecting harmful traffic.

Over time, DNS sinkholes have found broader applications, such as blocking
intrusive advertisements or pop-ups that degrade the user experience on the
internet. The technique has become an integral part of ad-blocking and network
security solutions. Among the many DNS sinkhole implementations, the Pi-Hole
project stands out as one of the most widely recognized and user-friendly
systems, offering network-wide ad-blocking capabilities.

## Pi-Hole

Pi-Hole is an open-source project that emerged in 2014 as a network-wide ad
blocker designed to run on lightweight hardware like the Raspberry Pi.
Its primary functionality involves acting as a DNS sinkhole, intercepting DNS
requests and blocking those associated with ads, trackers, or malicious
domains. By utilizing extensive blocklists, Pi-Hole prevents devices on the
network from reaching these domains, enhancing both security and user
experience.

Initially popular among tech enthusiasts and home users, Pi-Hole has since
gained traction in small business networks thanks to its ability to run on
relatively constrained hardware like a Raspberry Pi, and it is a great model
to reference for creating our own custom variant of this using Yocto.

## Dnsmasq

`dnsmasq` is a lightweight DNS and DHCP server that is commonly used in
embedded systems and small networks. It is highly configurable and works well
with DNS sinkhole setups, as it allows us to define blocklists and redirect
unwanted traffic to a designated IP address. In this tutorial, `dnsmasq` will
handle the bulk of the DNS filtering. It will compare DNS queries against a
list of known malicious or unwanted domains and block those requests by
responding with a controlled IP address, typically one that leads to a
"black hole" or a local page indicating that the domain is blocked.
Specifically, we will point every site we don't want to see to the `127.0.0.1`
IP address.

## Why PHP?

[PHP](https://www.php.net/) has been around forever and is still a popular
choice for building dynamic websites and web applications. Despite the rise of
modern JavaScript frameworks and other backend technologies, PHP remains a
widely-used language for web development. It also already has built-in hooks
into all of the technologies we will be leveraging, so it makes integration
much easier. Those of you who might be seasoned web developers can almost
certainly develop something better, but for this simple example showcasing how
an embedded environment would work, PHP is a solid choice.

## Prerequisites

Either the previous tutorial or the one before it, where we create a custom
layer and integrate `lighttpd`, will work for this tutorial. Both are decent
starting points. In fact, it might be best to remove the custom device driver
if you're not using it, as the fewer unnecessary components we have on
purpose-built embedded systems, the better.

You will also want to be able to put this on your local network for it to work
correctly. Additionally, have a machine (such as the one you built this on)
connected to that same local network, with the ability to load a website.

## Creating Our Custom DNS Sinkhole

### 1. Setup Our Environment

Navigate to the `poky` directory that we have been working in from the previous
tutorials:

```sh
cd ~/bbb-example/poky/
```

### 2. Add Extra Layers

The first thing we want to do before we begin is see if any of the components
we want to integrate already exist. This will save us a lot of time as it means
we won't have to create and maintain our our custom recipe. If we were to look
through the recipes in our `poky` directory, we won't find `dnsmasq` or `php`.
However, that doesn't mean they do not exist; it just means that we don't have
any meta layers that might have them.

> *Note:* `bitbake-layers` has a `show-recipes` command you can call that will
> parse the available recipes within your working tree to see if a component
> is available to call. However, I find it to be a bit verbose, so I rarely
> use it. Feel free to experiment by sourcing `oe-init-build-env` and then
> doing something like `bitbake-layers show-recipes | grep <component>`

The first place you should always visit when seeing if recipes exist is the
[Layer Index](https://layers.openembedded.org/layerindex/branch/master/layers/).
Here, you can look across all of the official projects sponsored by
OpenEmbedded. There are tabs at the top for Layers, Recipes, Machines, and
more. Click on "Layers" and look for `dnsmasq`. You'll see a recipe for it
already exists, and it belongs to `meta-networking`. If you look above, you'll
notice `meta-networking` is a subdirectory of `meta-openembedded` - a core
layer adding lots of potential functionality.

This means we need to bring in the `meta-openembedded` layer, so let's go
ahead and add that.

> *Note:* If you still don't see a recipe on that website, try using Google to
> search for a recipe. Also try to look on GitHub to see if there might be
> someone who may have written a recipe. Having something to go off of can
> vastly speed up development for those who are new to Yocto.

```sh
# While inside the poky directory:
git clone https://git.openembedded.org/meta-openembedded -b kirkstone
```

This should clone the `meta-openembedded` meta layer and check it out to
the `kirkstone` branch so we can guarantee it is compatible with our currently
written recipes. Next, we need to add this new layer to our
`conf/bblayers.conf` file. Let's try it:

```sh
source oe-init-build-env
# You should now be in the build directory, but our layers are one level up so:
bitbake-layers add-layer ../meta-openembedded/meta-networking
```

It will actually error with:

```sh
ERROR: Layer 'networking-layer' depends on layer 'meta-python', but this layer is not enabled in your configuration
ERROR: Layer 'networking-layer' depends on layer 'openembedded-layer', but this layer is not enabled in your configuration
ERROR: Parse failure with the specified layer added, exiting.
```

So you might think you need to add the `meta-python` layer first. Let's try it:

```sh
bitbake-layers add-layer ../meta-openembedded/meta-python
```

It shows:

```sh
NOTE: Starting bitbake server...
ERROR: Layer 'meta-python' depends on layer 'openembedded-layer', but this layer is not enabled in your configuration
ERROR: Parse failure with the specified layer added, exiting.
```

What gives? Why does it keep complaining? The secret lies here:

```sh
ice@wsl2:build(kirkstone)$ cat ../meta-openembedded/meta-networking/conf/layer.conf
# We have a conf and classes directory, add to BBPATH
BBPATH .= ":${LAYERDIR}"

# We have a packages directory, add to BBFILES
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
            ${LAYERDIR}/recipes-*/*/*.bbappend"

BBFILE_COLLECTIONS += "networking-layer"
BBFILE_PATTERN_networking-layer := "^${LAYERDIR}/"
BBFILE_PRIORITY_networking-layer = "5"

# This should only be incremented on significant changes that will
# cause compatibility issues with other layers
LAYERVERSION_networking-layer = "1"

LAYERDEPENDS_networking-layer = "core"
LAYERDEPENDS_networking-layer += "openembedded-layer"
LAYERDEPENDS_networking-layer += "meta-python"

LAYERSERIES_COMPAT_networking-layer = "kirkstone"

LICENSE_PATH += "${LAYERDIR}/licenses"

SIGGEN_EXCLUDE_SAFE_RECIPE_DEPS += " \
  wireguard-tools->wireguard-module \
  wireless-regdb->crda \
```

If you recall from a previous tutorial, there are two ways to add layers - the
automatic way with `bitbake-layers` and the manual way by adjusting the
`bblayers.conf` ourselves. If you also recall, it is our `conf/layers.conf`
that defines how our layer operates. Blindly calling `bitbake-layers` without
looking at `conf/layers.conf` first may have you constantly calling BitBake as
you slowly figure out exactly what those layer dependencies are.
The `bitbake-layers` tool isn't intelligent enough to fully parse a chain of
dependencies and add them to the recipe. At least not at the time of writing
this blog post.

The exact area you want to look at is `LAYERDEPENDS` in the `layer.conf` file.
You can see it requires the `core` layer (itself), the `openembedded` layer,
and the `meta-python` layer.

To automatically add everything we need, type the following while inside the
`build` directory:

```sh
bitbake-layers add-layer ../meta-openembedded/meta-oe
bitbake-layers add-layer ../meta-openembedded/meta-python
bitbake-layers add-layer ../meta-openembedded/meta-networking
```

This will automatically add these layers to our `bblayers.conf` file. You can
also simply edit this file and add those layers yourself since you already
learned what layers `meta-networking` requires by reading its `layers.conf`
file. As you get more familiar with Yocto, you might find yourself moving away
from the `bitbake-layers` tool and preferring to manually edit these files
instead, but both methods are perfectly valid.

### 3. Adding Components to the Image

We want to edit our `mycustom-image.bb` file and modify it to install some new
components automatically into our image.
Open up `meta-bbb/recipes-images/images/mycustom-image.bb` and add the following:

```sh
IMAGE_INSTALL += " \
    dnsmasq \
    dnsmasq-config \ # Custom recipe we will create soon
    lighttpd \
    lighttpd-custom-files \ # Our previous custom recipe
    lighttpd-module-access \
    lighttpd-module-fastcgi \
    lighttpd-module-openssl \
    logrotate \
    php \
    php-cgi \
"
```

The module lines for `lighttpd` bring in custom modules that give us more
flexibility and control over the server. Specifically, the `mod_access` module
lets us place control access on specific resources or paths based on client IP
addresses. `mod_fastcgi` enables [FastCGI](https://wiki.wireshark.org/FastCGI)
so we can work with PHP to dynamically generate web content. `mod_openssl` lets
you enable SSL/TLS support so you can handle HTTPS connections securely.
We won't necessarily be configuring all of these in this tutorial, but having
them available on your embedded system will help you when you go to actually
deploy this.

> *Note:* If you're wondering where I got the additional `lighttpd` components
> from, it was from parsing the recipe itself and figuring out what I needed
> in order to get the `access`, `fastcgi` and `openssl` modules working. I also
> built `lighttpd` and checked its build directory to see what components it
> generates from the recipe itself. If you're interested in learning about
> different modules, you can find more info by looking at the
> [`lighttpd` wiki](https://redmine.lighttpd.net/projects/lighttpd/wiki/Mod_access).

### 4. Configuring dnsmasq

Now that we have `dnsmasq-config` in our `IMAGE_INSTALL`, we need to create
the recipe. We will want this to have a `blocked.list` file that will be placed
in the `/etc` directory, so let's go ahead and create that first:

```sh
cd ../meta-bbb
mkdir -p recipes-example/dnsmasq-config/files
touch recipes-example/dnsmasq-config/files/blocked.list
```

> *Note:* There is an argument to be made that `dnsmasq-config` and the older
> `lighttpd-custom-files` are core components to this image. As such, they can
> be moved to `recipes-core` instead of `recipes-example` if you'd like. In the
> end, most of the folder naming structure is arbitrary when you have no other
> recipe or reference to go off of. Just use your best guess. Of course, if a
> recipe already exists, and you want to `.bbappend` it, you will want to
> create an analogous structure within your layer and place the `.bbappend`
> inside the analogous directory.

Populate the `blocked.list` with the following:

```sh
# This file is your blocked list where websites go to DNS la la land
# Redirect your most hated, least trusted websites to 127.0.0.1 like shown:
127.0.0.1   doubleclick.net
127.0.0.1   ads.facebook.com
127.0.0.1   track.example.com
```

Now we need to create the recipe to add this newly created `blocked.list` file.
If you've been following along, it should be pretty familiar by now:

```sh
# Summary and license
SUMMARY = "Configuration files for dnsmasq to provide DNS filtering"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

# Location of the file(s) we want and what our $S dir should be
SRC_URI = "file://blocked.list"
S = "${WORKDIR}"

# This recipe requires dnsmasq to be installed for it to work properly,
# so we add dnsmasq as a runtime dependency
RDEPENDS_${PN} = "dnsmasq"

# Make sure we have an /etc/ folder and place the blocked.list
# in the folder
do_install() {
    install -d ${D}/etc/
    install -m 0644 blocked.list ${D}/etc/blocked.list
}

# Files to be provided by this package
FILES_${PN} += "/etc/blocked.list"
```

We're almost done with the `dnsmasq` modifications. The final modification we
need to make is to have `dnsmasq` look for our newly created `blocked.list`.
In the [`dnsmasq.conf.example`](https://github.com/imp/dnsmasq/blob/master/dnsmasq.conf.example#L141),
there exists a `addn-hosts` section where you can tell `dnsmasq` to look at
another file besides just `/etc/hosts`. We want this because we want `dnsmasq`
to look at our custom `blocked.list` file, so we need to uncomment this.
We also need to make sure we log all `nslookup` entries to a file.
We can do all of this with a `.bbappend`, so let's create it:

```sh
mkdir -p recipes-support/dnsmasq
touch recipes-support/dnsmasq_%.bbappend
```

Open up the newly created `dnsmasq_%.bbappend` and add the following:

```sh
do_install:append() {
    # Convenience variable for the conf file
    DNSMASQ_CONF="${D}${sysconfdir}/dnsmasq.conf"

    # Look for our blocked.list in /etc
    sed -i 's|#addn-hosts=/etc/banner_add_hosts|addn-hosts=/etc/blocked.list|' $DNSMASQ_CONF

    # Log the queries
    sed -i 's|^#log-queries|log-queries|' $DNSMASQ_CONF

    # Set log-facility to log to /var/log/dnsmasq.log
    sed -i '/^log-queries/a log-facility=/var/log/dnsmasq.log' $DNSMASQ_CONF
}
```

This will uncomment that line and redirect `addn-hosts` to point to our newly
created `blocked.list` residing in `/etc`. It will also uncomment `log-queries`
and log the queries to a `.log` file in `/var/log`. This is the file we will
parse with our PHP-based web interface.

### 5. Creating the Web Interface

With `dnsmasq` setup, we can move onto our custom web interface. In a previous
tutorial, we created an `index.html` file. Let's replace that with an
`index.php` instead to make things more streamlined.

```sh
rm meta-bbb/recipes-example/lighttpd-custom-files/files/index.html
touch meta-bbb/recipes-example/lighttpd-custom-files/files/index.php
```

Open up the newly created `index.php` and add the following:

```php
<?php
// A simple PHP script to parse dnsmasq logs and show stats

$logfile = '/var/log/dnsmasq.log'; // Log file we will parse
$blockedList = '/etc/blocked.list'; // Block list we will parse

// Counters and lists
$totalQueries = 0;
$blockedQueries = 0;
$domainCounts = array();

// Parse the log file if it exists
if (file_exists($logfile)) {
    $lines = file($logfile, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);
    foreach ($lines as $line) {
        // Typical dnsmasq query log line: "Feb  3 12:34:56 hostname dnsmasq[123]: query[A] example.com from 192.168.1.100"
        if (preg_match('/query\[[A-Z]+\]\s+([^\s]+)\s+from/i', $line, $matches)) {
            $totalQueries++;
            $domain = $matches[1];
            if (!isset($domainCounts[$domain])) {
                $domainCounts[$domain] = 0;
            }
            $domainCounts[$domain]++;
        }
    }
}

// Build a set of blocked domains from the blocked list
$blockedSet = array();
if (file_exists($blockedList)) {
    $blockedDomains = file($blockedList, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);
    foreach ($blockedDomains as $bDomainLine) {
        // Format: "127.0.0.1   ads.example.com"
        $parts = preg_split('/\s+/', $bDomainLine);
        if (isset($parts[1])) {
            $blockedSet[$parts[1]] = true;
        }
    }
}

// Count how many queries were blocked
foreach ($blockedSet as $bdom => $val) {
    if (isset($domainCounts[$bdom])) {
        $blockedQueries += $domainCounts[$bdom];
    }
}

// Sort domains by count in descending order
arsort($domainCounts);

// Limit top domains shown
// NOTE: We will use CDNs here. In a truly embedded system, we would be
// placing these files on the disk somewhere and reading from them so we could
// guarantee they would always work with our packaged system.
$topDomains = array_slice($domainCounts, 0, 10, true);
?>
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
<title>BBB DNS Filter Stats</title>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@4.6.0/dist/css/bootstrap.min.css">
</head>
<body class="bg-light">
<div class="container mt-4">
    <h1 class="mb-4">BBB DNS Filter Statistics</h1>
    <div class="row">
        <div class="col-md-4">
            <div class="card text-white bg-info mb-3">
                <div class="card-header">Total Queries</div>
                <div class="card-body">
                    <h5 class="card-title"><?php echo $totalQueries; ?></h5>
                </div>
            </div>
        </div>
        <div class="col-md-4">
            <div class="card text-white bg-danger mb-3">
                <div class="card-header">Blocked Queries</div>
                <div class="card-body">
                    <h5 class="card-title"><?php echo $blockedQueries; ?></h5>
                </div>
            </div>
        </div>
    </div>

    <h2>Top Queried Domains</h2>
    <table class="table table-striped">
        <thead>
            <tr>
                <th>Domain</th>
                <th>Count</th>
                <th>Status</th>
            </tr>
        </thead>
        <tbody>
        <?php foreach ($topDomains as $dom => $count): 
            $status = isset($blockedSet[$dom]) ? "Blocked" : "Allowed";
        ?>
            <tr>
                <td><?php echo htmlspecialchars($dom); ?></td>
                <td><?php echo $count; ?></td>
                <td><?php echo $status; ?></td>
            </tr>
        <?php endforeach; ?>
        </tbody>
    </table>

</div>
</body>
</html>
```

This will create our custom web interface that will track blocked and allowed
DNS requests. Now we need to slightly adjust our previously created
`lighttpd-custom-files` recipe to be `index.php` instead of `index.html`.
Open up `recipes-example/lighttpd-custom-files/lighttpd-custom-files.bb` and
change it to look like the following:

```sh
# Summary of the recipe
SUMMARY = "Test webpage files for Lighttpd"

# Our license we will use for this
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

# The location of where the index.php file was placed
# NOTE: BitBake will automatically look for a "files" directory in an attempt
#       to locate the files listed in the SRC_URI path
SRC_URI = "file://index.php"

# We depend on lighttpd, but we now also depend on php components, too
RDEPENDS_${PN} = "lighttpd php php-cgi"

# Our source directory
S = "${WORKDIR}"

# The install action. This mimics RPM install syntax
do_install() {
    install -d ${D}/var/www # Installing the directory /var/www
    install -m 0644 ${WORKDIR}/index.html ${D}/var/www/index.php # Installing the file index.php in /var/www
}

# The file(s) we want to add to the recipe to be installed
FILES_${PN} = "/var/www/*"
```

Our `lighttpd-custom-files` is now done and ready for deployment. We're almost done!

### 6. Tweaking lighttpd

The last thing we need to do is make a couple tweaks to the `lighttpd.conf`
configuration file. In a previous tutorial, we modified the location of where
`lighttpd` looks at to serve its main page with a `.bbappend`. Let's tweak a
couple more things in that configuration file so `lighttpd` can properly serve
the newly created PHP site. Open up `recipes-extended/lighttpd/lighttpd_%.bbappend`
and add the following:

```sh
do_install:append() {
    # Convenience variable pointing to lighttpd.conf
    LIGHTTPD_CONF="${D}${sysconfdir}/lighttpd/lighttpd.conf"

    # Adjust the document root in lighttpd.conf from /www/pages to /var/www
    sed -i 's|"/www/pages/"|"/var/www"|' $LIGHTTPD_CONF

    # Uncomment "mod_fastcgi" in server.modules
    sed -i 's|#\s*\("mod_fastcgi"\)|\1|' $LIGHTTPD_CONF

    # Remove ".php" from static-file.exclude-extensions
    sed -i 's|".php",||' $LIGHTTPD_CONF

    # Uncomment the entire fastcgi.server section
    sed -i '/#fastcgi.server/,/#}/s/^#//' $LIGHTTPD_CONF

    # Update bin-path to point to /usr/bin/php-cgi
    sed -i 's|"/usr/local/bin/php"|"/usr/bin/php-cgi"|' $LIGHTTPD_CONF

    # Comment out ssl.engine and ssl.pemfile to avoid the OpenSSL errors
    # NOTE: Having this use OpenSSL is inherently more secure, but for demo
    # purposes, we can comment this out for now.
    sed -i 's|^\(ssl.engine\)|#\1|' $LIGHTTPD_CONF
    sed -i 's|^\(ssl.pemfile\)|#\1|' $LIGHTTPD_CONF
}
```

> *Note:* The `lighttpd.conf` mentions creating a `php.ini` file and adding
> `cgi.fix_pathinfo = 1` to it. I found this wasn't necessary. However, if you
> would like to add it, I found `php-cgi` will see it if you place it in the
> `/usr/bin` directory. Feel free to create a recipe that creates this file and
> places it in that directory if you'd like.

### 7. Building

With everything in its correct place, we can finally build our image:

```sh
cd build
bitbake mycustom-image
```

If everything was successful, you should have a `.wic` located in the `tmp`
directory as mentioned in previous tutorials.

## Testing the DNS Sinkhole

Insert the MicroSD card, boot, and login. Look up what the IP address is by
typing `ifconfig` and looking at what IP address the BeagleBone Black is
currently set to. On your machine that is connected to the same network as the
BBB, navigate to `http://<BBB-IP-ADDRESS>/index.php`. You should see:

![Landing Page](/assets/img/2024-12-15/dns-sinkhole-opening-page.png)

On the BBB, try typing in the following:

```sh
nslookup www.google.com <BBB-IP-ADDRESS>
```

If `dnsmasq` is working correctly, it should allow you to connect to Google.
Reload the PHP website, and it should show something like this:

![Allowed](/assets/img/2024-12-15/dns-sinkhole-allowed.png)

Next, try typing in the following in the BBB:

```sh
nslookup doubleclick.net <BBB-IP-ADDRESS>
```

You should notice that it will redirect it to `127.0.0.1` if your
`blocked.list` is working correctly. Reload the PHP website again,
and you should see this:

![Blocked](/assets/img/2024-12-15/dns-sinkhole-blocked.png)

As you continue to attempt to navigate to these sites, this website will
continue to parse the `dnsmasq` log and populate it with the latest info.

## Troubleshooting

The Yocto portion of this should go fairly smoothly. If you're having issues
with the layers, make sure `bblayers.conf` is setup correctly and that you have
the correct repos cloned. Make sure they're all checked out to the same branch.
If you run into a `dnf` failure, make sure you're not trying to install a file
in the same location another RPM is attempting to install. Adjust paths as
necessary.

The big headache you might run into is on the configuration side of things.
We are modifying the `lighttpd` and `dnsmasq`configuration files, and any
little mistake or typo can throw off the whole setup. These configuration files
are solidly written to show what needs to be done to integrate everything, so
read the comments in the configuration files and check the logs in `/var/log`.
If you find the services aren't running, use `/etc/init.d/<component> start`
and watch what stdout shows.

## Wrapping Up

You now have a working DNS sinkhole you can expand upon, and it can be
immediately deployable to any BeagleBone Black you might have lying around!
If you would like to block more addresses, simply add them to the
`blocked.list`. If you would like this to handle all devices on the network so
that all of your computers automatically send those blocked IPs to the sinkhole,
update your router's DNS settings so that it points to the BeagleBone Black's
IP as the primary DNS server. The Pi-hole website has a
[great example](https://discourse.pi-hole.net/t/how-do-i-configure-my-devices-to-use-pi-hole-as-their-dns-server/245)
on how to do this.

## Further Reading

Some useful links that helped me when I first started out:

- [Pi-hole Docs](https://docs.pi-hole.net/main/prerequisites/)
- [dnsmasq on ArchWiki](https://wiki.archlinux.org/title/Dnsmasq)
- [PHP Tutorial](https://www.php.net/manual/en/tutorial.php)

Happy blocking!
