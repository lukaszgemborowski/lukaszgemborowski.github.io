---
layout: page
date:   2016-07-16 11:35:21 +0100
categories: linux qemu
---

# Minimalistic Linux system on qemu ARM #

Goal of this post is to show you how to build and run a simple ARM-based linux kernel and system on qemu.
Content of this little distribution will be made of two main parts: Linux kernel and Busybox for simple shell
and user space utils.

## Why? ##

There are some reasons why you want to have ARM Linux running on QEMU:

 * You’re Linux developer and you want to test some changes in kernel, with qemu it’s quick and simple. Qemu is able to run it’s own gdb server so you can attach with gdb to running kernel and debug it!
 * You’re qemu developer and you want to have simple OS for testing
 * You just want to learn how Linux works and how to build simple system.

## Go! ##

I assume that you’re running some Linux distro, preferably Debian or Ubuntu based (because it comes with
pre-built ARM cross-compiler). So first of all you need a… cross-compiler, for Ubuntu you can just “apt
install gcc-arm-linux-gnueabi” it. For other distro it may be different. Unfortunately I need to send you to
your distro documentation. You can also build your own cross-compiler following steps described in my next
post. In my case installed gcc binary is named arm-linux-gnueabi-gcc and it’s available in my PATH environment
variable. The second tool you need is qemu. On Ubuntu you need to install qemu-system-arm package. Create some
empty workspace for our project and navigate there. We will put all the stuff there. To verify if our
cross-compiler is working correctly we can try to compile simple program, create a file main.c with following
content:

{% highlight cpp %}
int main()
{
}
{% endhighlight %}

And then run:

    $ arm-linux-gnueabi-gcc main.cpp main.c -o test

It should compile without any errors. To verify if it’s really arm executable run:

    $ file test
    test: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.3, for GNU/Linux 3.2.0, BuildID[sha1]=2bf38f2c75d90391b4ec5aa4eac3cb2f041a15ee, not stripped

Yeah, it’s ARM executable. We just prove that we have working ARM cross-compiler. This is really good start.
:-) Now we need to get sources for our Linux “distribution”. The main part of our system will be kernel itself,
navigate to [kernel.org](https://www.kernel.org/) ang grab latest stable release. In the time when I’m writing
this document it’s 4.6.3. So:

    $ wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.6.3.tar.xz

Next we need some userland so we can log-in and perform some interactive actions. Kernel alone is not quite
useful for a regular user. For that we will use BusyBox. Once again, navigate to
[busybox.net](https://www.busybox.net/) and grab some fresh stable release. In my case it will be 1.24.2:

    $ wget http://busybox.net/downloads/busybox-1.24.2.tar.bz2

Extract downloaded archives:

    $ tar xf linux-4.6.3.tar.xz
    $ tar xf busybox-1.24.2.tar.bz2

## Building ARM Linux kernel ##

To do this as quick as possible we will use default configuration for qemu. To cross-compile Linux you need to
know two things:

1. Target architecture (in our case it’s arm)
2. Cross compiler name prefix, for example arm-linux-gnueabi-

You can start by navigating to kernel source tree and making default configuration:

    $ cd linux-4.6.3
    $ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- versatile_defconfig

As you can see we need provide those two things discussed before. Target architecture and prefix of gcc cross
compiler. Note that you only provide the prefix, not the full name of binary (so not including the “gcc” part
of the name). After that you can start compilation:

	$ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-

The compilation process will start and take some time. After it finishes you can test your brand new kernel:

    $ qemu-system-arm -M versatilepb -kernel arch/arm/boot/zImage -dtb arch/arm/boot/dts/versatile-pb.dtb  -serial stdio -append "serial=ttyAMA0"

Let’s talk about the arguments passed to qemu:

 * -M – the board name, qemu is able to simulate several different boards but our kernel is especially customized for this one. We have provided this defconfig when configuring the kernel.
 * -kernel – the kernel binary itself
 * -dtb – device tree for the board, I can discus this file on another occasion, but assume that it’s mandatory to boot the system
 * -serial – where the console should be printed, we want it on stdio.
 * -append – append additional kernel command line arguments. We need to inform kernel where to print stuff by default. We want it to be ttyAMA0 device (first serial port)

You should see the kernel booting and finally… kernel panic.

    Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
    CPU: 0 PID: 1 Comm: swapper Not tainted 4.6.3 #1
    Hardware name: ARM-Versatile (Device Tree Support)

Don’t worry, it’s expected. You do not have a device with a valid root file system. We will provide it in a
while and this is the reason why we want to have BusyBox. It will provide basic functionality for our small
system.

## Busybox ##

Go back one directory up to the place where you have extracted your BusyBox sources. Go into this directory
and configure BusyBox, you will need almost the same command line arguments as for Linux kernel:

    $ cd busybox-1.24.2
    $ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- defconfig

Now we we have default configuration file created. We need to tune it a little by making busybox executable
statically linked as we don’t want to provide additional shared libraries. We can do this by invoking:

    $ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- menuconfig

Navigating to Busybox Settings -> Build Options and checking “Build BusyBox as a static binary (no shared
libs)” option. Now we can proceed with compilation:

    $ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-
    $ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- install

This should be quite quick. Now we are ready to create our root filesystem image. We will put there init
script, busybox and also provide proper directory layout.

## Root filesystem ##

So what this thing should contain? What do we want for our minimalistic Linux system? Only a few things:

 * init script – Kernel needs to run something as first process in the system.
 * Busybox – it will contain basic shell and utilities (like cd, cp, ls, echo etc)

Navigate back to our workspace and create directory named rootfs. In this directory create file named init,
eg. vim rootfs/init

{% highlight bash %}
#!/bin/sh

mount -t proc none /proc
mount -t sysfs none /sys
mknod -m 660 /dev/mem c 1 1

echo -e "\nHello!\n"

exec /bin/sh
{% endhighlight %}

Make it executable by:

    $ chmod +x rootfs/init

Now copy busybox stuff there:

    $ cp -av busybox-1.24.2/_install/* rootfs/

Now you should have almost everything, there is only one thing missing, ie. standard directory layout. Let’s
create it:

    $ mkdir -pv rootfs/{bin,sbin,etc,proc,sys,usr/{bin,sbin}}

That’s all. We will use contents of this directory as the init ram disk so we need to create cpio archive and
compress it with gzip:

    $ cd rootfs
    $ find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../rootfs.cpio.gz

Note that we changed the current directory to rootfs first. This is because we don’t want “rootfs” to be
prepended to file paths.

## Running ##

This part is pretty straight forward, we will run our kernel almost exactly as before:

    $ qemu-system-arm -M versatilepb -kernel linux-4.6.3/arch/arm/boot/zImage -dtb linux-4.6.3/arch/arm/boot/dts/versatile-pb.dtb -initrd rootfs.cpio.gz -serial stdio -append "root=/dev/mem serial=ttyAMA0"

There are two minor changes:

 * -initrd argument is added, it points to the initial ram disk
 * root=/dev/mem is added to -append string. It tells kernel from where we want to boot.

Now you should see shell prompt. You’re in your own Linux “distribution”. :-)
