---
layout: post
title:  "Kernel Compilation and Installation"
date:   2017-02-26
published: true
categories: linux-kernel
---

## Command Summary

[//]: <> (TODO: REVISAR)
[//]: <> (TODO: Lembrar de falar dos pacotes necessários)

Follows the list of commands used in the sequence:

`.config` manipulations:

```bash
zcat /proc/config.gz > .config
```
or
```bash
cp /boot/config-`uname -r` .config
```

Change `.config` file:

```bash
make nconfig
make olddefconfig
make kvmconfig
make localmodconfig
```

Compile:

```bash
make ARCH=x86_64 -j8
make modules_install
```

Install:

```bash
sudo make modules_install
sudo make headers_install INSTALL_HDR_PATH=/usr
sudo make install
sudo make install
```

Remove:

```bash
rm -rf /boot/vmlinuz-[target]
rm -rf /boot/initrd-[target]
rm -rf /boot/System-map-[target]
rm -rf /boot/config-[target]
rm -rf /lib/modules/[target]
rm -rf /var/lib/initramfs/[target]
```


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

If you want to use Qemu for work, I recommend you to read my post about it in "[Use Qemu to play with Linux Kernel]({{ site.baseurl }}{% post_url 2017-02-26-use-qemu-to-play-with-linux %})"

## Get Your kernel

Linux project has many subsystems and most of them keep their clone of the
Kernel. Usually, the maintainer(s) of each subsystem is responsible for
receiving patches, and decide to apply or not the change. Later, the maintainer
says to Torvalds which branch to merge. This explanation is an
oversimplification of the process; you can find more details in the
documentation. It is important to realise that you have to figure out which
system you intend to contribute, discover their repository, and work based on
their branch. For example, if you want to contribute to RISC-V subsystem you
have to work on Palmer Dabbelt repository; if you're going to contribute to iio
use Jonathan Cameron repository. You can quickly figure out the target branch
by looking at the MAINTAINERS file. For this tutorial, we use the Torvalds
repository.  

```bash
git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

There are thousands of forks from Linux Kernel spread around the Internet. It
is possible to find entire organisations that keep their branch of the Kernel
with their specific changes. Also, you can find a mirror from Torvalds on
Github. Few free to use it, but keep in mind that you may encounter some
problems with non-official repositories. I prefer to use the git.kernel.org to
use because it simplifies future works.

## The Super `.config` file

[comment]: <> (Revisar daqui para baixo)

The `.config` file keeps all the configurations that will be used during the Kernel compilation. In this file you can find which driver or subsystem will be compiled or not. The `.config` file has three possible anwser per target: (1) m, (2) y, and (3) n. The character 'm' means that the target will be compiled as module; the 'y' and 'n' specifies if the subsystem will be compiled/installed or not.

Every Linux Distribution (e.g., Arch, Debian, and Fedora) usually maintain and distribute their own `.config` file. The `.config` file from distributions, normally enables most of the available options (specially the device drivers) because they have to run a large variety of hardware. It means, that you may have various device driver in your computer that you do not really need. However, the important thing here is: the more options you have enabled in the `.config` file, more time consume will be the compilation.

If it is your first attempt to use your own compiled kernel version, I strongly recommend you to use the `.config` file provided by your distribution to increase the chances of success. Later, you can expand the modification as we describe in this tutorial.

**Attention:**
The `.config` file is Super power, invest some time to better understand it. Also, save your working `.config` files, it will save time for you.
{: .notice_danger}

### Get your `.config` file

Depending on the distribution which you use, there is two options to get `.config` file: from `/proc`, or `/boot`. Both cases produces the same results, but it is not all distribution that enables `/proc` option (e.g., Arch enable it, but Debian not). The command bellow make the copy, notice that we suppose that you are in the clone directory of Kernel.

1. Get `.config` from `/proc`

```bash
zcat /proc/config.gz > .config
```

2. Get `.config` from `/boot`

```bash
cp /boot/config-`uname -r` .config
```

### Make your customizations

**Attention:**
There is a basic rule about `.config` file: NEVER CHANGE IT BY HAND, ALWAYS USE A TOOL
{: .notice_danger}

There is several option to manually change the `.config` file. I will introduce only two:

```bash
make nconfig
```

`nconfig` looks like this:

{% capture fig_img %}
![Foo]({{ site.url }}/images/nconfig.png)
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption>nconfig</figcaption>
</figure>

And finally, we have `menuconfig`:

```bash
make menuconfig
```

`menuconfig` looks like this:

{% capture fig_img %}
![Foo]({{ site.url }}/images/menuconfig.png)
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption>Menuconfig</figcaption>
</figure>

### Final considerations about `.config` and tips

When you use a configuration file provided by a Linux Distribution, hundreds of device drivers are enabled; normally, you just need a few driver. All the enabled drivers will raise the compile time with no need for you, fortunetally, there is an option that automatically change the `.config` file in order to enabled only required drivers. However, before use the command, it is highly recommended to enable all the devices that you use with your computer in order to ensure that `.config` file have all the required driver by your machine enabled. In other words, plug all the devices that you normally use before execute the command:

```bash
make localmodconfig
```

**Remember:**
Enables all the device
{: .notice_info}

This command, basically uses `lsmod` to check the enables or disables devices drivers in the `.config` file.

[//]: <> (TODO: Seria legal simular isso aqui)

Sometimes, when you rebase your local master branch with the remote you will notice when try to compile that some questions are raised related to the ativation or deactiavion of features. This happens, because during the evolution of the Kernel new features are added that was not in present in you `.config` file. As a result, you are asked to take a decision. Sometimes, there is a way to partially reduce the amount of asked question with the command:

```bash
make olddefconfig
```

Finally, one last tip is related for someone that make experiments in the Qemu with Kvm. There is an option that enables some important features for this scenario:

```bash
make kvmconfig
```

## Compile!

Now, it timeeeeee! After a bunch of setup, I am quite sure that you anxious for this part. So, here we go... type:

```bash
make -j [two_times_the_number_of_core]
```

[//]: <> (TODO: Seria legal expandir a explicação do dobro dos cores)
Just replace the `two_times_the_number_of_core` for the number of cores you have by two. For example, if you have 8 cores you should add 16.

If you want to compile your kernel for one specific computer arquitecture, you can use:

```bash
make ARCH=x86_64 -j [two_times_the_number_of_core]
```

For compiling the kernel modules, type:

```bash
make modules_install
```

## Compilation Outputs

[//]: <> (TODO: Seria legal expandir a explicação do dobro dos cores)

## Install new your custom kernel

It is important to pay attention in the installation order, we install modules following:

1. Install modules
2. Install header
3. Install Image
4. Update bootloader (Grub)

**Attention:**
Double your attention in the install steps. You can crash your system because you have execute all commands as a root user.
{: .notice_danger}

### Install modules and headers

Just type:

```bash
sudo make modules_install
```

If you want to check the changes, take a look at `/lib/modules/$(uname -r)`

Finally, install headers:

```bash
sudo make headers_install INSTALL_HDR_PATH=/usr
```

### Install new image (works on Debian)

Finally, it is time to install your Kernel image. This step does not work on Arch Linux, see next section if you are interested in Arch.

To install the Kernel module just type:

```bash
sudo make install
```

We are, reallyyyyy close to finish the process. We just have to update the bootloader, and here we suppose you are using Grub. Type:

```bash
sudo update-grub2
```

Notice, that the command below is an wrapper to the following command:

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

So, if the first command fails try the last one.

Now, reboot your system and check if everything is ok.

### Arch Linux Installation

If you use Arch Linux, we present the basics steps to install your custom image. You can find a detailed explanation of this processes in the Arch Linux [wiki](https://wiki.archlinux.org/index.php/Kernels/Traditional_compilation).

First, you have copy your kernel image to the `/boot/` directory with the command:

```bash
sudo cp -v arch/x86_64/boot/bzImage /boot/vmlinuz-[name]
```

Replace [name] by any name. It could be your name.

Second, you have to create a new `mkinitcpio`. Follow the steps below:

1. Copy an existing `mkinitcpio`

```bash
sudo cp /etc/mkinitcpio.d/linux.present /etc/mkinitcpio.d/linux-[name].present
```

2. Open the copied file, and look it line by line and replaces old kernel name by the name you assigned. See the example:

**Attention:**
Keep in mind, that you have to adapt this file by yourself. There is no blind copy and paste here.
{: .notice_danger}

3. Generate he initramfs

```bash
sudo mkinitcpio -p linux-[name].prensent
```

## Remove

Finally, you may want to remove an old Kernel version for space or organization reasons. First of all, boot in another version of the Kernel that you want to remove and follow the steps below:

```bash
rm -rf /boot/vmlinuz-[target]
rm -rf /boot/initrd-[target]
rm -rf /boot/System-map-[target]
rm -rf /boot/config-[target]
rm -rf /lib/modules/[target]
rm -rf /var/lib/initramfs/[target]
```

## References

1. [Install customised kernel in Arch Linux](https://wiki.archlinux.org/index.php/Kernels/Traditional_compilation)
2. [Overview about Initramfs](https://en.wikipedia.org/wiki/Initramfs)
3. [Practical stuffs about Initramfs](http://nairobi-embedded.org/initramfs_tutorial.html)
4. [Great explanation about initramfs](https://landley.net/writing/rootfs-intro.html)
5. [More about initramfs](https://landley.net/writing/rootfs-howto.html)
6. [Nice discussion in Stack Exchange about bzimage in qemu](https://unix.stackexchange.com/questions/48302/running-bzimage-in-qemu-unable-to-mount-root-fs-on-unknown-block0-0)
7. [Kernel Readme](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git/tree/README?id=refs/tags/v4.3.3)
8. [Speedup kernel compilation](http://www.h-online.com/open/features/Good-and-quick-kernel-configuration-creation-1403046.html)
9. [Stack Overflow with some tips about speedup kernel compilation](https://stackoverflow.com/questions/23279178/how-to-speed-up-linux-kernel-compilation)
10. [SYSTEMD-BOOT](https://wiki.archlinux.org/index.php/systemd-boot#Adding_boot_entries)
11. [cpio tutorial](https://www.gnu.org/software/cpio/manual/html_mono/cpio.html)
12. [Busybox, build system](https://busybox.net/FAQ.html#build_system)
