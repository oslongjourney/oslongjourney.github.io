---
layout: post
title:  "Compile in-tree Kernel module"
date:   2017-02-26
published: true
categories: linux-kernel
---

[//]: <> (TODO: REVISAR)

## Command summary

```bash
make modules
make M=driver/iio/dummy
sudo make modules_install
sudo modprobe iio_dummy
sudo modprobe -r iio_dummy
```

## Introduction

I read many books and tutorials about device drivers, most of them are focused in the explanation of how to compile an external module. However, when the issue is compile and install in-tree module things was not so clear for newcomers. How we can compile a single in-tree module? How we can install it? I admit that I have difficult to understand how I can do it, and It took me a half of day I discover. After I figured out how to finish the task, I realise how simple it is.

For this tutorial, I will work on the `driver/iio/dummy` module [1]. However, the steps and tips described here can be extended to other modules.

## Compile In-tree Module

When I started to work with the in-tree modules, I always compiled all modules with the command:

```bash
make modules
```

My `.config` file is narrowed to my computer characteristics because I use `localmodconfig` which enables 132 modules. It means, that every single change in the in-tree module that I am working, I wast a lot of time with unsuful stuffs for my task. This really bothered me.

After some minutes googling about the subject, I found the command:

```bash
make M=driver/iio/dummy
```

This command, enables the compilation of a single module. However, there is a small trick: your module must enabled in the `.config` file.

## Install In-tree Module

For installing the module, I use the command:

```bash
sudo make modules_install
```

I tried to use the command below:

```bash
sudo make modules_install M=driver/iio/dummy
```

However, it not works. I do not know why, if I discover I will update this part.

### Load and unload

I tried many times to work with `insmod` command, however I really suffer with it and I realise why people always recommend to use `modprobe`. To understand how the load works, keep in mind that all drivers installed in the system can be find in `/lib/modules/$(uname -r)`. Also, it is important to know that `modprobe` command is a wrapper for `insmod` and `rmmod`. Use `modprobe` make the work with load and unload module easy, because it handles decencies.

After the installation, you just need to load the module with the command:

```bash
sudo modprobe iio_dummy
```

To remove the module, use:

```bash
sudo modprobe -r iio_dummy
```

## Tips and Troubleshoots

### Check installed module

Sometimes, I am not feel confident that my module is correct loaded and I used to add some prints in the code. Another approach, it is update the `MODULE_DESCRIPTION` and check the metadate with the command `modinfo`. For example, change the line in any driver that you use with something like that:

```c
MODULE_DESCRIPTION("IIO dummy driver -> IIO dummy modified by Me");
```

Then, after you install the module you can check the description as following:

```bash
$ modinfo iio_dummy
filename:       /lib/modules/4.16.0-rc3-TORVALDS+/kernel/drivers/iio/dummy/iio_dummy.ko.xz
license:        GPL v2
description:    IIO dummy driver -> IIO dummy modified by Me
author:         Jonathan Cameron <jic23@kernel.org>
srcversion:     1C4C5F875A87E3DFD4F2820
depends:        industrialio-sw-device,industrialio,iio_dummy_evgen,kfifo_buf
retpoline:      Y
name:           iio_dummy
vermagic:       4.16.0-rc3-TORVALDS+ SMP preempt mod_unload modversions
```

This is really useful, to help you to figure out if your module is correctly load. An common mistake, it is compile the module with a different kernel target which produces errors like:

**Error:**
FATAL: Module drivers/<path> not found in directory /lib/modules/<kernel_version>
{: .notice_danger}

If you get this error, the tip above can help you to figure out what is wrong. Lastly, remember to check the kernel Makefile to verify the version or your distro version.

### Get decencies information

Sometimes, it is important to know the modules decencies. You can use the following command:

```bash
$ modprobe --show-depends <module>
```

Why this is important? If you try something really radical as `rmmod -f`, it is a good idea to know the exactly decencies related to your device.

**My Experience:**
My experiece, say to me that is not a good idea to use `rmmod -f` with large modules. It is ok, if the module is small or if it is made by you. If you really know what you are doing, use it; if not, avoid. Prefere `modprobe -r`
{: .notice_info}

## References

1. [Link to a question in askubuntu that helped me](https://askubuntu.com/questions/168279/how-do-i-build-a-single-in-tree-kernel-module)
2. [IIO tasks proposed by Daniel Baluta](https://kernelnewbies.org/IIO_tasks)
