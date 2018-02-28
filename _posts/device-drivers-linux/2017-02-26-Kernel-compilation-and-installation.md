---
layout: post
title:  "Kernel compilation and installation"
date:   2017-02-26
published: true
categories: linux-kernel
---

## Choose Your Weapon

Nowadays we have multiple options to play around with Linux Kernel. For
simplicity sake, I classify the available approaches into three broad areas:
virtualisation, Desktop/Laptop, and embedded device. The virtualisation
technique is the safer way to conduct experiments with Linux kernel because any
fatal mistake has a few consequences. For example, if you crash the entire
system, you can create another virtual machine or take one of your backups
(yes, make a backup of your running kernel images). Experiment on your computer
is more fun and risk. Any potential problem could break all your system.
Finally, for the embedded device, you can make tests in a developing kit. In
this section, we choose our weapons. We will work first with Qemu, followed by
the local computer. 

Next section presents the basic setup for creating any virtual machine using
Qemu, few free to skip this section.

### Setup a Qemu image and running

iI am not interested in providing details related to virtualization in this
here; I just present the steps in a very straightforward way. At the end of
this post, you can find some useful links. First of all, install  Qemu on your
machine. Then, create a new image with the command:

{% highlight bash %}
qemu-img create -f raw image_file 4G
{% endhighlight %}

## Get Your kernel

## The Super `.config` file

## Get Your `.config` file

## Make Your customizations

## Compile!

## Compilation Outputs

## Arch Linux Installation

## Debian Installation

## Remove

## Some Final Tips
