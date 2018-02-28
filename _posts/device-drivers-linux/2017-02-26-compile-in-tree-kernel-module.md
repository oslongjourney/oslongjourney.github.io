---
layout: post
title:  "Compile in-tree Kernel module"
date:   2017-02-26
published: true
categories: linux-kernel
---

[//]: <> (TODO: REVISAR)

## Command summary

{% highlight bash %}
make modules
make M=driver/iio/dummy
sudo make modules_install
sudo modprobe iio_dummy
sudo modprobe -r iio_dummy
{% endhighlight %}

## Introduction

I read many books and tutorials about device drivers, most of them are focused in the explanation of how to compile an external module. However, when the issue is compile and install in-tree module things was not so clear for newcomers. How we can compile a single in-tree module? How we can install it? I admit that I have difficult to understand how I can do it, and It took me a half of day I discover. After I figured out how to finish the task, I realise how simple it is.

For this tutorial, I will work on the `driver/iio/dummy` module [1]. However, the steps and tips described here can be extended to other modules.

## Compile In-tree Module

When I started to work with the in-tree modules, I always compiled all modules with the command:

{% highlight bash %}
make modules
{% endhighlight %}

My `.config` file is narrowed to my computer because I use `localmodconfig` which enables 132 modules. It means, that every single change in the in-tree module that I am working, I wast a lot of time with unsuful stuffs for my task. This really bothered me.

After some minutes googling about the subject, I found the command:

{% highlight bash %}
make M=driver/iio/dummy
{% endhighlight %}

This command, enables the compilation of a single module. However, there is a small trick: your module must enabled in the `.config` file.

## Install In-tree Module

For installing the module, I use the command:

{% highlight bash %}
sudo make modules_install
{% endhighlight %}

I tried to use the command below:

{% highlight bash %}
sudo make modules_install M=driver/iio/dummy
{% endhighlight %}

However, it not works. I do not know why, if I discover I will update this part.

### Load and unload

I tried many times to work with `insmod` command, however I really suffer with it and I realise why people always recommend to use `modprobe`. To understand how the load works, keep in mind that all drivers installed in the system can be find in `/lib/modules/$(uname -r)`. Also, it is important to know that `modprobe` command is a wrapper for `insmod` and `rmmod`. Use `modprobe` make the work with load and unload module easy, because it handles decencies.

After the installation, you just need to load the module with the command:

{% highlight bash %}
sudo modprobe iio_dummy
{% endhighlight %}

To remove the module, use:

{% highlight bash %}
sudo modprobe -r iio_dummy
{% endhighlight %}

## Rereference

1. [Link to a question in askubuntu that helped me](https://askubuntu.com/questions/168279/how-do-i-build-a-single-in-tree-kernel-module)
2. [IIO tasks proposed by Daniel Baluta](https://kernelnewbies.org/IIO_tasks)
