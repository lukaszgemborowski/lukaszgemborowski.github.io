---
layout: post
title:  "Building your own qemu"
date:   2016-07-24 11:35:21 +0100
categories: linux qemu
---

This post is about where to find, how to clone official qemu repository and how to build fresh copy of it.

## Why? ##

As always there is a question why? There are some reasons why would you like to build your own qemu:

* you don’t have qemu package in your distro (is there such a distro anyway?)
* your distro is providing quite old qemu or you want to try some new dev snapshot
* and most important: you want to become a qemu developer :-)

In the future I will probably write some posts about qemu development so this short post should be good entry
point for the future.

## Start ##

qemu needs some extra packages (libs and tools) installed on your machine before building:


* git
* glib2.0-dev
* libfdt-dev
* libpixman-1-dev
* zlib1g-dev
* … and of course some basic dev stuff like C compiler, make, etc

Now you can get sources, as always create some workspace directory for that, go into it and clone qemu
repository:

	$ git clone git://git.qemu-project.org/qemu.git
	$ cd qemu

Now you should decide, to build fresh qemu you can stick to master branch (ie. branch that you are actually
in), if you want to check some release or release candidate first list available tags:

	$ git tag

You can choose whatever you want or need. Let’s try stable 2.6.0 version:

	$ git checkout v2.6.0

## Configuring ##

This is the most important part of building qemu. You may want to run ./configure –help top see all the
options available. If you want to reduce compile time and output size you may choose only one platform
target, in our example ARM. In this point you can also define “target directory” where make install will put
all the files. You will use prefix switch for that. I often tend to put all custom-built packages in my $HOME/opt directory. Let’s do it this way:

	$ ./configure --target-list=arm-softmmu --prefix=$HOME/opt

Now you can proceed with build:

	$ make
	$ make install

Verify the installation

	$ $HOME/opt/bin/qemu-system-arm --version
	QEMU emulator version 2.6.0, Copyright (c) 2003-2008 Fabrice Bellard

That’s it, you have your own qemu correctly built and installed. In the near feature I will try to write
something about qemu development itself.
