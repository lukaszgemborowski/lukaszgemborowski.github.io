---
layout: post
title:  "Building custom kernel module with buildroot"
date:   2016-08-01 11:35:21 +0100
categories: linux kernel drivers buildroot
---

In previous post we have created simple kernel module. We are able to run it in our current system. What about
some embedded platform or Qemu? Some time ago I showed you how to prepare a Linux system with buildroot. What
about using this system to run our custom module? We can choose from at least two options:

* putting the module in kernel tree
* creating separate project just for our module

as long as we don’t want to upstream our module the best way would be to put it in separate project. But to
build and install such project using buildroot we need a special kind of makefile called a recipe. This small
makefile will tell buildroot how to build our project. It also add it in buildroot menu so the user can check
it from menuconfig as any other usual package. Prepare a clean build of Linux system using buildroot described
by me some posts ago, also create a directory with your kernel module, for example the one described in
previous post. Go to package directory under your buildroot source tree, eg. buildroot-2016.02/package and
create new directory there, in our case example-module.

	$ cd buildroot-2016.02/package
	$ mkdir example-module
	$ cd example-module/

Create two files there, Config.in:

{% highlight bash %}
comment "example-module needs a Linux kernel to be built"
	depends on !BR2_LINUX_KERNEL

config BR2_PACKAGE_EXAMPLE_MODULE
	bool "example-module"
	depends on BR2_LINUX_KERNEL
	help
		Example kernel module.
{% endhighlight %}

And example-module.mk:

{% highlight make %}
EXAMPLE_MODULE_VERSION = 0.1
EXAMPLE_MODULE_SITE = /home/$USER/Projects/article/
EXAMPLE_MODULE_LICENSE = GPLv2

$(eval $(kernel-module))
$(eval $(generic-package))
{% endhighlight %}

now buildroot will expect that example-module-0.1.tar.gz ({package name}-{version}.tar.gz) will be located in
path defined in EXAMPLE_MODULE_SITE variable. You need to tar and gzip your example module. Rename your
directory where you have example.c and Makefile files to example-module and pack it:

	$ tar cf example-module.tar example-module
	$ gzip example-module.tar

go to your buildroot build directory and run:

	make example-module

then run

	make

to build whole system image. Run qemu as described previously and see what will happen

{% highlight bash %}
$ qemu-system-arm -M versatilepb -kernel images/zImage -dtb images/versatile-pb.dtb -initrd images/rootfs.cpio.gz -serial stdio -append "root=/dev/mem serial=ttyAMA0"
.
.
.
.
Welcome to Buildroot
buildroot login: root
# modprobe example.ko
# ls /sys/kernel
fscaps         mm             rcu_expedited  uevent_helper
jiffies        notes          slab           uevent_seqnum
# cat /sys/kernel/jiffies
4294951490
{% endhighlight %}

voilà! But it’s not all, at this point you need to manually run make example-module to bring your module to
system’s rootfs. Last thing will be registering your package in buildroot system so you will be able to tick
the option from menuconfig. To do this you need to add 

		source "package/example-module/Config.in"

in the buildroot-2016.02/package/Config.in file in your “favorite” submenu. Review content of Config.in to
find most suitable place for your package and place it there, then you can run make menuconfig to see your
brand new package in buildroot. In my case I’ve put it under

	menu "Development tools"

Run menuconfig and go to Target packages —> Development tools —>

![menuconfig]({{ "/assets/buildroot-example-menu.png" | absolute_url }})
