---
layout: post
title:  "Use Qemu to Play with Linux Kernel"
date:   2017-02-26
published: true
categories: linux-kernel
---

## Command summary
```bash
qemu-img create -f raw kernel_experiments 10G

qemu-system-x86_64 -cdrom ~/path/distro_iso.iso -boot order=d -drive \
                file=kernel_experiments,format=raw -m 2G

qemu-system-x86_64 -enable-kvm -net nic -net user,hostfwd=tcp::2222-:22,smb=$PWD/ \
                   -daemonize -m 2G -smp cores=4,cpus=4 kernel_experiments
```

## Introduction

## Install Qemu and KVM

### Arch

### Debian

## Create an image

In order to use Qemu to install and use a Linux Distribution, you have to create a image disk. For illustration, we create an image named kernel_experiments with 15G:

```bash
qemu-img create -f raw kernel_experiments 15G
```

* `qemu-img`: It is the disk image utility.
  * `create`: Create new disk image;
  * `-f format`: Specifies the format of the image. Normally, you want raw or qcow2; we discuss this in more details in this section;
  * `filename`: The command expects a filename to the new image. Here, we decided by kernel_experiments;
  * `size`: The image size, we decided by 15G.


## Installing an Distro

Next, download the distribution ISO of your preference (I recommend Debian and Arch). With your qemu image and your distro ISO, proceed with the command:

```bash
qemu-system-x86_64 -cdrom ~/path/distro_iso.iso -boot order=d -drive \
                   file=kernel_experiments,format=raw -m 2G
```


## Configure ssh

## Start Qemu

Finally, after the installation it is time to start the machine with the command:

```bash
qemu-system-x86_64 -enable-kvm -net nic -net user,hostfwd=tcp::2222-:22,smb=$PWD/ \
                   -daemonize -m 2G -smp cores=4,cpus=4 kernel_experiments
```

