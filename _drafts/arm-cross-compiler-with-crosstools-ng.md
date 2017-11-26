---
layout: post
title:  "ARM cross compiler with crosstols-ng"
date:   2016-07-16 11:35:21 +0100
categories: linux qemu
---

In previous post I have showed you how to build small Linux system running on Qemu ARM. Most probably some of
you faced a problem that ARM cross compiler is not delivered with your linux distribution. We can easily solve
this issue with project named [crosstools-ng](http://crosstool-ng.org/). From the project web page:

> crosstool-NG aims at building toolchains. Toolchains are an essential component in a software development
> project. It will compile, assemble and link the code that is being developed. Some pieces of the toolchain
> will eventually end up in the resulting binary/ies: static libraries are but an example.

Other reason to have your own toolchain is repeatability. If you have bigger project with multiple people
involved then most probably you want to have exactly the same environment for everyone. Compilers can differ a
little bit from version to version so having the same toolchain across the team is very important especially
in embedded development.

## Getting the toolchain ##

Navigate to http://crosstool-ng.org/ to find latest release. You will find announces on the top of the page in
“news” section. If not, you can directly go to download page: http://crosstool-ng.org/download/crosstool-ng/.
Naming scheme is crosstool-ng-VERSION.tar.bz2, where version is X.Y.Z, so you need to get highest X, highest Y
and Z. They usually put a file named 00-LATEST-is-VERSION so you can easily find newest version. Now it’s
1.22.0. Grab and extract it:

    $ wget http://crosstool-ng.org/download/crosstool-ng/crosstool-ng-1.22.0.tar.bz2
    $ tar xf crosstool-ng-1.22.0.tar.bz2
    $ cd crosstool-ng/

This is standard autotools package so the building is straightforward. One thing you need to consider is
location where you want to install your toolchain. I don’t want to install it system-wide so my destination
directory will be $HOME/opt. If you want to install it system-wide you can omit prefix option in configure
invocation:

    $ ./configure --prefix=$HOME/opt
    $ make
    $ make install

At this point you will have tool called ct-ng installed. If you installed it in non standard place you need to
add it to your PATH variable. In my case:

    $ export PATH=$HOME/opt/bin:$PATH

This is build and management script for toolchains. You can see all preconfigured toolchains:

    $ ct-ng list-samples

select desired toolchain

    $ ct-ng arm-unknown-linux-gnueabi

After the build toolchain will be installed by default in ${HOME}/x-tools. If you want to change this edit
.config file, search for CT_PREFIX_DIR variable and modify it if you want.

    $ ct-ng build

You should see something like:

    ...
    [EXTRA]    Building a toolchain for:
    [EXTRA]      build  = x86_64-pc-linux-gnu
    [EXTRA]      host   = x86_64-pc-linux-gnu
    [EXTRA]      target = arm-unknown-linux-gnueabi
    ...

It means that you are building x86 toolchain for ARM target. So cross compiling from you PC architecture to
ARM. Now it’s coffee break time as downloading and building gcc will take a while, really. Done!

## Usage ##

In the previous post I have described how to build Linux kernel for ARM architecture. To provide
cross-compiler for build system we have set variable CROSS_COMPILE=arm-linux-gnueabi-. Now if you want to use
your brand new toolchain when building the linux kernel you can simply use:

    $ make ARCH=arm CROSS_COMPILE=${HOME}/x-tools/arm-unknown-linux-gnueabi/bin/arm-unknown-linux-gnueabi-
