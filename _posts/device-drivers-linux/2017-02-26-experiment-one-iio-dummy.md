---
layout: post
title:  "Experiment One: Play with iio_dummy"
date:   2017-02-26
published: true
categories: linux-kernel
---

In this section we conduct a series of basic experiment with `iio_dummy`. Here, we will do the following tasks:

1. Enabled IIO dummy via nconfig;
2. Compile IIO dummy module;
3. Load and unload module;
4. Play a little with `/sys/bus/iio/*`;
5. Add channels for a 3-axis compass to `iio_simple_dummy` module.

Finally, before you read this post I recommend you to read "[Compile In-tree Driver]({{ site.baseurl }}{% post_url 2017-02-26-compile-in-tree-kernel-module %})". In this post I explain some details about compile, load, unload, and some troubleshoots.

## Enabling `iio_dummy`

To compile the `iio_dummy`, first we have to enable it in the `.config` file. The picture below shows the steps:

{% capture fig_img %}
![Foo]({{ site.url }}/images/experiment-one/menu-activation-steps.png)
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption>Steps to activate iio_dummy </figcaption>
</figure>

After you enabled the module, you can verify if the options are correctly enabled by inpect the `.config` file. You should see something similar to this:

```yaml
...
#
# IIO dummy driver
#
CONFIG_IIO_DUMMY_EVGEN=m
CONFIG_IIO_SIMPLE_DUMMY=m
CONFIG_IIO_SIMPLE_DUMMY_EVENTS=y
CONFIG_IIO_SIMPLE_DUMMY_BUFFER=y
...
```

## Compile Module, Load, and Unload

To compile the module, load, and unload the module follow the steps below:

```bash
make M=driver/iio/dummy
sudo make modules_install
sudo modprobe iio_dummy
```

If you want to know the details about the compilation/load or have any problem in this step, then read my post "Compile In-tree Driver" [1]. After the above steps, you can check the module information as following:

```bash
$ modinfo iio_dummy
filename:       /lib/modules/4.16.0-rc3-TORVALDS+/kernel/drivers/iio/dummy/iio_dummy.ko.xz
license:        GPL v2
description:    IIO dummy driver
author:         Jonathan Cameron <jic23@kernel.org>
srcversion:     B2B5E23A9B1B98D882091B3
depends:        industrialio-sw-device,industrialio,iio_dummy_evgen,kfifo_buf
retpoline:      Y
name:           iio_dummy
vermagic:       4.16.0-rc3-TORVALDS+ SMP preempt mod_unload modversions
```

Then, for verifing if the module is correct loaded you can use the `lsmod` and `grep` together. After you execute the command, you should see an output similar to this: 

```bash
$ lsmod | grep iio_dummy
iio_dummy              16384  0
industrialio_sw_device    16384  1 iio_dummy
kfifo_buf              16384  1 iio_dummy
iio_dummy_evgen        16384  1 iio_dummy
industrialio           81920  3 iio_dummy,iio_dummy_evgen,kfifo_buf
```

Finally, let's take a look at the `/sys/bus/iio/devices/` directory as a final confirmation that everything is right. Try the command below, and verify if your output is similar:

```bash
$ ls -l /sys/bus/iio/devices
total 0
lrwxrwxrwx 1 root root 0 Mar  2 15:55 iio_evgen -> ../../../devices/iio_evgen

$ ls -l /sys/bus/iio/devices/iio_evgen/
total 0
--w------- 1 root root 4096 Mar  2 15:56 poke_ev0
--w------- 1 root root 4096 Mar  2 15:56 poke_ev1
--w------- 1 root root 4096 Mar  2 15:56 poke_ev2
--w------- 1 root root 4096 Mar  2 15:56 poke_ev3
--w------- 1 root root 4096 Mar  2 15:56 poke_ev4
--w------- 1 root root 4096 Mar  2 15:56 poke_ev5
--w------- 1 root root 4096 Mar  2 15:56 poke_ev6
--w------- 1 root root 4096 Mar  2 15:56 poke_ev7
--w------- 1 root root 4096 Mar  2 15:56 poke_ev8
--w------- 1 root root 4096 Mar  2 15:56 poke_ev9
drwxr-xr-x 2 root root    0 Mar  2 15:56 power
lrwxrwxrwx 1 root root    0 Mar  2 15:56 subsystem -> ../../bus/iio
-rw-r--r-- 1 root root 4096 Mar  2 15:52 uevent
```

Finally, at this step, if you want to unload the module just try:

```bash
sudo modprobe -r iio_dummy
```

## Play around with `iio_dummy`

### The configfs

The last step to create your dummy device, it is mount a `configfs` filesystem in whatever place you want. I prefer to do it in the `/mnt` directory, as you can see in the commands below:

```bash
sudo mkdir /mnt/iio_experiments/
sudo mount -t configfs none /mnt/iio_experiments/
```

If you take a look at the `/mnt/iio_experiments`, you can see something similar to:

```bash
$ ls /mnt/iio_experiments/
iio  pci_ep

$ ls /mnt/iio_experiments/iio/devices/
dummy
```

How about create a new device? Just create a new directory inside `dummy` directory:

```bash
$ sudo mkdir /mnt/iio_experiments/iio/devices/dummy/my_glorious_dummy_device

$ ls /mnt/iio_experiments/iio/devices/dummy/
my_glorious_dummy_device
```

### Inspect `/sys/bus/iio/devices` again

How about look again the `/sys/bus/iio/devices/`?

```bash
$ ls -l /sys/bus/iio/devices/
total 0
lrwxrwxrwx 1 root root 0 Mar  2 16:07 iio:device0 -> ../../../devices/iio:device0
lrwxrwxrwx 1 root root 0 Mar  2 15:55 iio_evgen -> ../../../devices/iio_evgen
```

Notice that `iio:device0` now appear in the tree. Let's take look inside it:

```c
$ ls -l /sys/bus/iio/devices/iio:device0/ 
total 0
drwxr-xr-x 2 root root    0 Mar  2 16:09 buffer
-rw-r--r-- 1 root root 4096 Mar  2 16:09 current_timestamp_clock
-r--r--r-- 1 root root 4096 Mar  2 16:09 dev
drwxr-xr-x 2 root root    0 Mar  2 16:09 events
-rw-r--r-- 1 root root 4096 Mar  2 16:09 in_accel_x_calibbias
-rw-r--r-- 1 root root 4096 Mar  2 16:09 in_accel_x_calibscale
-rw-r--r-- 1 root root 4096 Mar  2 16:09 in_accel_x_raw
-rw-r--r-- 1 root root 4096 Mar  2 16:09 in_activity_running_input
-rw-r--r-- 1 root root 4096 Mar  2 16:09 in_activity_walking_input
-rw-r--r-- 1 root root 4096 Mar  2 16:09 in_sampling_frequency
-rw-r--r-- 1 root root 4096 Mar  2 16:09 in_steps_calibheight
-rw-r--r-- 1 root root 4096 Mar  2 16:09 in_steps_en
-rw-r--r-- 1 root root 4096 Mar  2 16:09 in_steps_input
-rw-r--r-- 1 root root 4096 Mar  2 16:09 in_voltage0_offset
-rw-r--r-- 1 root root 4096 Mar  2 16:09 in_voltage0_raw
-rw-r--r-- 1 root root 4096 Mar  2 16:09 in_voltage0_scale
-rw-r--r-- 1 root root 4096 Mar  2 16:09 in_voltage1-voltage2_raw
-rw-r--r-- 1 root root 4096 Mar  2 16:09 in_voltage3-voltage4_raw
-rw-r--r-- 1 root root 4096 Mar  2 16:09 in_voltage-voltage_scale
-r--r--r-- 1 root root 4096 Mar  2 16:09 name
-rw-r--r-- 1 root root 4096 Mar  2 16:09 out_voltage0_raw
drwxr-xr-x 2 root root    0 Mar  2 16:09 power
drwxr-xr-x 2 root root    0 Mar  2 16:09 scan_elements
lrwxrwxrwx 1 root root    0 Mar  2 16:09 subsystem -> ../../bus/iio
drwxr-xr-x 2 root root    0 Mar  2 16:09 trigger
-rw-r--r-- 1 root root 4096 Mar  2 16:02 uevent
```

As you can see, there is many attributes and other stuffs inside it. Each attribute has some information, for example:

```bash
$ cat /sys/bus/iio/devices/iio:device0/name
my_glorious_dummy_device

$ cat /sys/bus/iio/devices/iio:device0/in_accel_x_raw
34

$ cat /sys/bus/iio/devices/iio:device0/in_voltage0_raw
73
```

### Busy device

When I was trying to unload `iio_dummy`, I constantly get this message:

```bash
$ sudo modprobe -r iio_dummy
modprobe: FATAL: Module iio_dummy is in use.
```

I tried many things, I actually tried the command `rmmod -r iio_dummy` and realise that I put my kernel in an unstable state (I got oops message during the reboot). After some hours trying to figure out, I realise the problem is related with the directory `my_glorious_dummy_device` previously create. To solve this problem, I just did:

```bash
sudo rmdir /mnt/iio_experiments/iio/devices/dummy/my_glorious_dummy_device/
sudo modprobe -r iio_dummy
```

## Adding Channels for a 3-axis compass

Now, we will modify simple dummy to add 3-axis compass channel. I will not detailed all the steps, because I already dig into the `iio_dummy` in the "[The `iio_dummy` Anatomy]({{ site.baseurl }}{% post_url 2017-02-26-compile-in-tree-kernel-module %})" post. In case you are a newcomer on IIO subsystem, I strongly recommend you to read the post about `iio_dummy` anatomy in order to perfectly understand this section.

### Update simple dummy header

Ok, let's start to work. We want to add three new channels (one per axes), we start by update the file `drivers/iio/dummy/iio_simple_dummy.h`. First of all, define a value for the 3 new channels as following:

```c
#ifndef _IIO_SIMPLE_DUMMY_H_
#define _IIO_SIMPLE_DUMMY_H_
#include <linux/kernel.h>

struct iio_dummy_accel_calibscale;
struct iio_dummy_regs;

#define DUMMY_AXIS_XYZ 3
``` 

Secondly, update the struct `iio_dummy_state` to accept keep the data for the 3 axis. We added `u16 buffer_compass[DUMMY_AXIS_XYZ]` in the code below:

```c
struct iio_dummy_state {
	int dac_val;
	int single_ended_adc_val;
	int differential_adc_val[2];
	int accel_val;
	int accel_calibbias;
	int activity_running;
	int activity_walking;
	const struct iio_dummy_accel_calibscale *accel_calibscale;
	struct mutex lock;
	struct iio_dummy_regs *regs;
	int steps_enabled;
	int steps;
	int height;
	u16 buffer_compass[DUMMY_AXIS_XYZ];
#ifdef CONFIG_IIO_SIMPLE_DUMMY_EVENTS
	int event_irq;
	int event_val;
	bool event_en;
	s64 event_timestamp;
#endif /* CONFIG_IIO_SIMPLE_DUMMY_EVENTS */
};
```

Finally, we have to update `iio_simple_dummy_scan_elements` in order to add a new index for each channel. See the last four elements in the code below:

```c
enum iio_simple_dummy_scan_elements {
	DUMMY_INDEX_VOLTAGE_0,
	DUMMY_INDEX_DIFFVOLTAGE_1M2,
	DUMMY_INDEX_DIFFVOLTAGE_3M4,
	DUMMY_INDEX_ACCELX,
	DUMMY_INDEX_SOFT_TIMESTAMP,
	DUMMY_MAGN_X,
	DUMMY_MAGN_Y,
	DUMMY_MAGN_Z,
};

```

Notice that I declared `DUMMY_INDEX_SOFT_TIMESTAMP`, I did it for comprehension issues (you will see the reason in the next section). With this changes, we finished with `iio_simple_dummy.h`

### Add channels to `iio_chan_spec`

Now it is time to configure the channel to add one channel per axis. See:

```c
	{
		.type = IIO_MAGN,
		.modified = 1,
		.channel2 = IIO_MOD_X,
		.info_mask_separate = BIT(IIO_CHAN_INFO_RAW),
		.info_mask_shared_by_type = BIT(IIO_CHAN_INFO_SCALE),
		.scan_index = DUMMY_MAGN_X,
		.scan_type = {
			.sign = 'u',
			.realbits = 16,
			.storagebits = 16,
			.shift = 0,
		},
	},
```

We will add the new channels at the end of the `iio_chan_spec iio_dummy_channels` struct. We declared the type as `IIO_MAGN`, and used `.modified` to configure `.channel2` as `IIO_MOD_X`. Next, the field `.info_mask_shared_by_type` made the scale shared for this channel and we make the same for all the others channels. Note that `.scan_index` receives `DUMMY_MAGN_X`, this value should be unique. Finally, there is `.scan_type` that configures the buffer type as unsigned and with 16 bits for resolution and storage.

The channels for the axis Y and Z are similar, they differ by the field `.channel2` and `.scan_index`. Do you remember from the last section that I told to rember of `DUMMY_INDEX_SOFT_TIMESTAMP`? So, go to iio_chan_spec again and find for:

```c
	/*
	 * Convenience macro for timestamps. 4 is the index in
	 * the buffer.
	 */
	IIO_CHAN_SOFT_TIMESTAMP(4),
```

The first time that I tried to add a new channel I receive an error that same index used. After a long time trying to understand, I relise the above line use the scan_index 4 and I tried to use it. To make the code more readable (from my perpective), I decided to add this elment in the `iio_simple_dummy_scan_elements` and finally replace the magic number 4 by "DUMMY_INDEX_SOFT_TIMESTAMP".

### Initialize values

The last part added new channels, we have to initialize them. We do it in the `iio_dummy_init_device` function, as descbribed below:

```c
static int iio_dummy_init_device(struct iio_dev *indio_dev)
{
	struct iio_dummy_state *st = iio_priv(indio_dev);

    ...
	st->buffer_compass[0] = 78;
	st->buffer_compass[1] = 10;
	st->buffer_compass[2] = 3;

	return 0;
}
```

### Update `*read_raw()` to handling the new channel

In order to make the data provided by our new channel accessible in the user space, we have to expand the `iio_dummy_read_raw()` function. Look the code below:

```c
static int iio_dummy_read_raw(struct iio_dev *indio_dev,
			      struct iio_chan_spec const *chan,
			      int *val,
			      int *val2,
			      long mask)
{
	struct iio_dummy_state *st = iio_priv(indio_dev);
	int ret = -EINVAL;

	mutex_lock(&st->lock);
	switch (mask) {
	case IIO_CHAN_INFO_RAW: /* magic value - channel value read */
		case IIO_VOLTAGE:
            ...
			break;
		case IIO_ACCEL:
            ...
			break;
		case IIO_MAGN:
			switch(chan->scan_index) {
				case DUMMY_MAGN_X:
					*val = st->buffer_compass[0];
					break;
				case DUMMY_MAGN_Y:
					*val = st->buffer_compass[1];
					break;
				case DUMMY_MAGN_Z:
					*val = st->buffer_compass[2];
					break;
				default:
					*val = 99;
					break;
			}
			ret = IIO_VAL_INT;
			break;
		default:
			break;
		}
...

```

Notice that we add `IIO_MAGN` inside `IIO_CHAN_INFO_RAW`, and collected each data provided by the state. Now, we just have to add the shared channel:

```c
	case IIO_CHAN_INFO_SCALE:
		switch (chan->type) {
		case IIO_VOLTAGE:
		    ...
			break;
		case IIO_MAGN:
			// Just add some dummy values
			*val = 0;
			*val2 = 2;
			ret = IIO_VAL_INT_PLUS_MICRO;
			break;
		default:
			break;
		}
```

Almost done, I just want to add one final touch. Go to the end of this file and change the description:

```c
MODULE_DESCRIPTION("IIO dummy driver -> IIO dummy modified by Me");
```

Done! Compile and install the module again to test the new channels. You should see something similar to:

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

### Test the changes

For finish this tutorial, let's look again the `/sys/bus/iio/devices/iio:device0/`.

```bash
$ ls -l /sys/bus/iio/devices/iio:device0/
total 0
drwxr-xr-x 2 root root    0 Mar  2 16:27 buffer
-rw-r--r-- 1 root root 4096 Mar  2 16:27 current_timestamp_clock
-r--r--r-- 1 root root 4096 Mar  2 16:27 dev
drwxr-xr-x 2 root root    0 Mar  2 16:27 events
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_accel_x_calibbias
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_accel_x_calibscale
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_accel_x_raw
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_activity_running_input
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_activity_walking_input
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_magn_scale
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_magn_x_raw
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_magn_y_raw
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_magn_z_raw
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_sampling_frequency
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_steps_calibheight
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_steps_en
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_steps_input
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_voltage0_offset
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_voltage0_raw
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_voltage0_scale
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_voltage1-voltage2_raw
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_voltage3-voltage4_raw
-rw-r--r-- 1 root root 4096 Mar  2 16:27 in_voltage-voltage_scale
-r--r--r-- 1 root root 4096 Mar  2 16:27 name
-rw-r--r-- 1 root root 4096 Mar  2 16:27 out_voltage0_raw
drwxr-xr-x 2 root root    0 Mar  2 16:27 power
drwxr-xr-x 2 root root    0 Mar  2 16:27 scan_elements
lrwxrwxrwx 1 root root    0 Mar  2 16:28 subsystem -> ../../bus/iio
drwxr-xr-x 2 root root    0 Mar  2 16:27 trigger
-rw-r--r-- 1 root root 4096 Mar  2 16:27 uevent
```

Now, you can see four new attributes: in_magn_x_raw, in_magn_y_raw, in_magn_z_raw, and in_magn_scale. Take a look at each one:

```bash
$ cat /sys/bus/iio/devices/iio:device0/in_magn_scale
0.000002

$ cat /sys/bus/iio/devices/iio:device0/in_magn_x_raw
78

$ cat /sys/bus/iio/devices/iio:device0/in_magn_y_raw
10

$ cat /sys/bus/iio/devices/iio:device0/in_magn_z_raw
3
```

## References

1. [Compile In-tree Driver]({{ site.baseurl }}{% post_url 2017-02-26-compile-in-tree-kernel-module %})
2. [The `iio_dummy` Anatomy]({{ site.baseurl }}{% post_url 2017-02-26-compile-in-tree-kernel-module %})
