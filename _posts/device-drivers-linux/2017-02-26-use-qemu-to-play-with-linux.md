---
layout: post
title:  "Use QEMU to Play with Linux Kernel"
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
                   -daemonize -m 4G -smp cores=4,cpus=4 kernel_experiments
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption>Summary of all commands used in this post</figcaption>
</figure>

## Introduction

QEMU is a generic machine emulator based on dymamic translation [3]. QEMU can operates in two different modes [2]:

* Full system emulation: This mode completely emulate a computer that can be used to launch different Operating Systems.
* User mode emulation: Enable to launch a processes for one sort of CPU on another CPU.

If you have recent `x86` machine, you can also execute QEMU with kvm which increases the speed. Finally, we decided to use QEMU since it is a very popular open source tool and it provides a many resources for easily play with virtual machine.

## Install QEMU and KVM

Install QEMU and Samba package. We want the samba package for allowing the directory share.

### Arch

Install Debian packages [1]:

```bash
sudo pacman -S qemu samba qemu-arch-extra
```

### Debian

Install Debian packages [2]:

```bash
sudo apt install qemu samba samba-client
```

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

## References

1. [QEMU Arch Linux](https://wiki.archlinux.org/index.php/QEMU)
2. [QEMU Debian](https://wiki.debian.org/QEMU#Installation)
3. [QEMU](https://www.qemu.org/)
