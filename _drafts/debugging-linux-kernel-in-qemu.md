---
layout: post
title:  "Debugging Linux kernel in qemu"
date:   2016-08-07 11:35:21 +0100
categories: linux qemu kernel debug gdb
---

After all these posts about qemu and buildroot finally we came into the most amazing thing about it. Debugging
live system. Yes, with Linux running under qemu you can actually connect with gdb to the kernel, set break
points and see variables! Of course it’s not so beautiful as you may think because we cannot disable certain
build-time optimizations which affects debugging capabilities but still this is very powerful possibility. I
will use Eclipse as code browser and debugger front-end. But first before we set up Eclipse project let’s
change the way how buildroot is getting source code for kernel. I prefer to have my local copy of git
repository somewhere around so I don’t depend on buildroot downloaded tar.gz with sources. Buildroot is
downloading tarball and extracting it in build directory but it can always clean it on demand (so you may
loose your changes). Note that this is completely optional, you can debug kernel without creating separate
source directory for kernel. What’s more when setting up eclipse project with buildroot kernel build you won’t
point to your source tree, instead you will use temporary build directory. This is because before building
buildroot will copy all the kernel files to it’s build directory and build from there! This is important
because all debug info will point to files in that “temporary” directory instead to your source tree. So if
you prefer to not change this just pay attention in every place I’m mentioning linux-custom directory and
change it accordingly, for example to linux-4.4.1. You can also almost completely omit buildroot configuration,
just remember about “Build cross gdb for the host” option, it’s necessary anyway. The fastest and most
“compatible” way will be to steal kernel downloaded already by buildroot. After going through all the steps
from my previous post you should get all the artifacts in build directory and pre-downloaded tarballs in
buildroot-2016.02/dl. You should find there a file linux-4.4.1.tar.xz (or similar). Now extract it somewhere,
for example in the directory where you have buildroot and build directories.

	$ tar xf buildroot-2016.02/dl/linux-4.4.1.tar.xz

## Configuring buildroot ##

Now you need to tell buildroot that you want to use your custom kernel rather than one downloaded by buildroot
from Internet (even if it’s exactly the same sw package). Invoke menuconfig in buildrooot build directory, go
to Kernel —> Kernel version (Custom version) —> and select Local directory, now you should see new option
Path to the local directory (NEW), press enter on it and provide full path to kernel sources.

![menu]({{ "/assets/buildroot-menuconfig-custom-kernel.png" | abolute_url }})

To be on the safe site you should also go to Toolchain —> Kernel Headers (Same as kernel) —> and change it to Linux 4.4.x kernel headers

![menu]({{ "/assets/buildroot-menuconfig-kernel-headers.png" | absolute_url }})

Last thing in buildroot setup is to tell it that we want host gdb to be built. Go to Toolchain —> and select
Build cross gdb for the host. Exit and save. For some reason there are some options missing in kernel
configuration, for example I have CONFIG_OF unchecked from kernel configuration after this operation. So to
enable missing config options and add some more debug information to the kernel run

	make linux-menuconfig

Go to Device Drivers —> and select (press space) Device Tree and Open Firmware support —>

![Linux menuconfig, device drivers]({{ "/assets/linux-menuconfig-devicedrivers-of.png" | absolute_url }})

Next, navigate to System Type —> Versatile platform type —> and check Support Versatile platform from device tree.
You will also need to make kernel more debugable, go to Kernel hacking —> and make sure that Kernel debugging
is selected. In the same page go to Compile-time checks and compiler options —> and select Compile the kernel with debug info

![Linux menuconfig, debug options]({{ "/assets//linux-menuconfig-debug.png" | absolute_url }})

Exit and save the config, run make. Hopefully you will see no errors.

## Eclipse ##

There’s a [great article](https://wiki.eclipse.org/HowTo_use_the_CDT_to_navigate_Linux_kernel_source) on eclipse wiki about basic setup. Just follow it and remember that our target
architecture is arm (not x86 or arm64), also remember to use build/build/linux-custom (or linux-4.4.1 if you
didn’t change kernel location in buildroot) as source code location, not the directory where you extracted
kernel. After steps from this article you should have project set up. Now you need to set debug configuration.

1. Open Run -> Debug configurations…
1. Right clock C/C++ Remote Application and select New
1. Click Browse under C/C++ Application (or just put the location in the box), you need to select file located in your buildroot build directory, namely: buildroot-build-dir/build/linux-custom/vmlinux
1. Check Disable auto build
1. Switch to Debugger tab
1. Uncheck Stop on startup
1. Below, in Debugger Options (Main tab) change GDB debugger to one we built before: buildroot-build-directory/host/usr/bin/arm-buildroot-linux-uclibcgnueabi-gdb
1. On the bottom of the configuration window you have something like: Using GDB (DSF) Automatic Remote Debugging Launcher – Select other…, click Select other.., window will pop-up, slect Use configuration specific settings and choose GDB (DSF) Manual Remote Debugging Launcher – this is important and easy to miss!
1. Switch to Connection tab under Debugger Options and change Port Number to 1234
1. Switch to Common tab
1. Select Debug under Display in favorites menu
1. Apply and Close

## Debug session ##

Now let’s start qemu. Open terminal and go to buildroot build directory, start qemu normally but append two additional command line options: -s -S

	qemu-system-arm -M versatilepb -kernel images/zImage -dtb images/versatile-pb.dtb -initrd images/rootfs.cpio.gz -serial stdio -append "root=/dev/mem serial=ttyAMA0" -s -S

Press enter, qemu should “hang”. It starts listening for gdb on port 1234 and stops the simulation. Now switch
to Eclipse, suppose that we want to debug serial controller driver. Open drivers/tty/serial/amba-pl011.c,
find probe function:

{% highlight c %}
static int pl011_probe(struct amba_device *dev, const struct amba_id *id)
{% endhighlight %}

and place breakpoint in the first line.

![Setting breakpoint]({{ "/assets/eclipse-pl011-probe.png" | absolute_url }})

Start debugging

![Start debugging]( {{ "/assets/eclipse-start-debug.png" | absolute_url }})

Eclipse ask you to switch to debug perspective, agree with that and continue debugging. If everything is properly
configured you should break on pl011 driver probe function:

![Eclipse, kernel debugging]( {{ "/assets/eclipse-debugging-kernel-first.png" | absolute_url }})

You have learned at least two things today:

1. How to debug kernel under qemu with a help of Eclipse
1. How to change default kernel in buildroot
