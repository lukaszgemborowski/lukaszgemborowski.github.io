---
layout: post
title:  "Simple kernel module and sysfs interface for exporting jiffies value"
date:   2016-07-31 11:35:21 +0100
categories: linux kernel drivers
---

	

I will start with a very basic kernel module example. As such you will probably find thousands of similar
posts on the Internet. Nevertheless I’ve decided to write yet another guide for it because this is related to
contents of this blog. Anyway after short introduction we will create read-only entry under /sys/kernel
exporting current value of jiffies. Let’s start with simple example, as I said before you can find thousands
of such examples in the Internet so I won’t describe it in much details, example.c:

{% highlight c %}
#include <linux/module.h>
#include <linux/kernel.h>

static int __init example_init(void)
{
	printk(KERN_INFO "Hello world\n");
	return 0;
}

static void __exit example_exit(void)
{
}

module_init(example_init);
module_exit(example_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Lukasz Gemborowski <lukasz.gemborowski@gmail.com>");
MODULE_DESCRIPTION("Kernel module example");

{% endhighlight %}

and a Makefile:

{% highlight make %}
ifneq ($(KERNELRELEASE),)
obj-m := example.o
endif
{% endhighlight %}

you can build it for your current kernel by calling:

	make -C /lib/modules/`uname -r`/build M=$PWD

just remember you must have you kernel sources (or linux headers) installed. In Debian/Ubuntu you can just
apt-get install linux-headers-generic. Ok, so what does our module do? Nothing besides one print. You can
load it by issuing insmod example.ko command (you need to be root to do that, you can also prepend this
command with sudo). Dump your kernel log buffer with dmesg, you should see “Hello world” print somewhere on
the bottom.

	$ sudo insmod example.ko
	$ dmesg | grep Hello
	[48951.781400] Hello world
	$ sudo rmmod example.ko

## Creating sysfs entry ##

Let’s add some functionality to our module. Till now it could only print a message to the kernel log. We will
now extend it by exporting a file under /sys, this is standard way to communicate between user space and
kernel. You can use sysfs to export properties of your module or driver that can be read, write or read&write
in runtime. The mechanism is simple, you define some attributes and provide read or write callbacks:

{% highlight c %}

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/sysfs.h>

static ssize_t jiffies_show(struct kobject *kobj, struct kobj_attribute *attr, char *buf)
{
	return sprintf(buf, "%lu\n", jiffies);
}

static const struct kobj_attribute jiffies_attribute = __ATTR_RO(jiffies);

static int __init example_init(void)
{
	int res;

	if ((res = sysfs_create_file(kernel_kobj, &jiffies_attribute.attr)) < 0) {
		printk(KERN_ERR "Failed to create sysfs entry (%d)\n", res);
		return res;
	}

	return 0;
}

static void __exit example_exit(void)
{
	sysfs_remove_file(kernel_kobj, &jiffies_attribute.attr);
}

module_init(example_init);
module_exit(example_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Lukasz Gemborowski <lukasz.gemborowski@gmail.com>");
MODULE_DESCRIPTION("Kernel module example");

{% endhighlight %}

Now in our init function there’s call to sysfs_create_file. As a first argument it takes the kobject to which
attributes will be added, in our case it’s kernel_kobj. This variable is exported in kobject.h header.

{% highlight c %}
/* The global /sys/kernel/ kobject for people to chain off of */
extern struct kobject *kernel_kobj;
{% endhighlight %}

we use it just to attach our “jiffie” entry to it. The second argument of this function is attribute. We have
defined kobj_attribute with help of __ATTR_RO macro. It initializes kobj_attribute structure with some default
values suited for read only attribute. It also require us to provide attribute_name_show() function which is a
callback called when someone reads from /sys/kernel/jiffies. jiffies_show() purpose is to fill a provided
buffer with jiffies value in ASCII format so it can be easily interpreted by humans. Keep in mind that this
buffer is limited to PAGE_SIZE, in our case output is small enough that I don’t care about any checks, in fact
when writing such a function you should really care about this kind of checks as it can compromise your system,
you’re in kernel space so you can do almost anything! Let’s test our code:

	$ make -C /lib/modules/`uname -r`/build M=$PWD
	$ sudo insmod example.ko
	$ ls -l /sys/kernel/jiffies 
	-r--r--r-- 1 root root 4096 lip 31 13:31 /sys/kernel/jiffies
	$ cat /sys/kernel/jiffies 
	4296949098
	$ sudo rmmod example.ko

That’s all for today. In case of any questions or problems drop me a comment.
