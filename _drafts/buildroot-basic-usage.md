---
layout: post
title:  "Buildroot basic usage"
date:   2016-07-16 11:35:21 +0100
categories: linux qemu
---

	

In two previous posts you have learned how to build a corss-compiler and how to build and run simple Linux
system on Qemu ARM. In this post we will simplify the whole process (in terms of steps you need to perform)
and create more advanced Linux system than before.

## Buildroot ##

For this purpose we will use tool called [buildroot](https://buildroot.org/). In short, buildroot is a set of
makefiles which automates process of building the embedded Linux system. It will build several things for us:

 * cross-compiler
 * Linux kernel
 * busybox
 * libc
 * final system image

It will do many other things, for example it provide default configuration or build several other tools and
packages. We can just select packages (like ssh server) from the list in configuration menu similar to kernel
or busybox menuconfig. But let’s start with simple things. Let’s build minimalistic system which does not have
much more functionality than the system built before.

## Building ##

First of all you need to download buildroot package. In the time of writing this post newest version is
2016.02. You can download it from official webpage or from github. You can take archive or clone repository.
Let’s get tar.gz release from github and unpack it.

    $ wget https://github.com/buildroot/buildroot/archive/2016.02.tar.gz
    $ tar xf 2016.02.tar.gz

you should see buildroot-2016.02 directory, now create separate build directory (in fact it is not needed but
will separate sources from actual build stuff making it more clear) and initialize project with default
configuration:

    $ mkdir build
    $ make -C $(pwd)/buildroot-2016.02 O=$(pwd)/build qemu_arm_versatile_defconfig

we’re almost done but to retain the same output format as in previous post we need to change one configuration
option. Let’s do this now:

    $ cd build
    $ make menuconfig

you should see the menu, go to Filesystem images, uncheck ext2/3/4 root filesystem, instead check cpio the
root filesystem and set compression to gzip.

![menuconfig]({{ "/assets/menu-1.png" | absolute_url }})

You can exit the menu saving configuration file (remember to press “Yes” when you’re asked about save). After
this step you can just run make and watch how your system is being build. Do not pass -j option to make here,
buildroot will automatically pass proper -j option to each package makefile when building (you can change this
in menu also: Build options -> Number of jobs to run simultaneously (0 for auto)).

    $ make

now you have quite long coffee-break.

## Running the system ##

We will run our fresh system exactly like the one built by ourselves before. The output of the build is now
placed in images subdirectory of our build directory. Let’s go:

    $ qemu-system-arm -M versatilepb -kernel images/zImage -dtb images/versatile-pb.dtb -initrd images/rootfs.cpio.gz -serial stdio -append "root=/dev/mem serial=ttyAMA0"

After a while you should see login prompt:

    Welcome to Buildroot
    buildroot login:

type “root”, press enter and you’re in the shell. So what do we get by using buildroot? Simplified build
process for sure. Buildroot also comes with many predefined configs for popular boards like:

 * raspberrypi2_defconfig
 * cubieboard2_defconfig
 * beaglebone_defconfig

just list buildroot-2016.02/configs directory to see them all. Do we get something more? Kill qemu (ctrl+c)
and re-run menuconfig by make menuconfig. Go to Target packages -> Games and check sl. Exit and save
configuration, run make, wait a while and run qemu the same way as before. Login as root and type “sl” in
terminal.

![sl command]({{ "/assets/sl.png" | absolute_url }})

Easy, isn’t it? You can select any predefined package from this menu or even add your own custom package.

