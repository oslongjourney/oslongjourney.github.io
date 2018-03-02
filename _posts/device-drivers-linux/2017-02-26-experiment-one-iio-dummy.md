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

If you want to know the details about the compilation/load or have any problem in this step, then read my post "Compile In-tree Driver" [1].

To verify if the module is correct loaded, try:

```bash
lssmod | grep dummy
```

Then, you should see an output similar to this:

TODO

```bash
```

Finally, take a look in the `/sys` directory for a final confirmation that everything is right. Try the command below, and verify if your output is similar:

TODO
```bash
ls -l /config/iio/devices/dummy/
ls -l /sys/bus/iio/devices/iio:device0/
ls -l /sys/bus/iio/devices/iio_evgen/ 
```

Finally, at this step, if you want to unload the module just try:

```bash
sudo modprobe -r iio_dummy
```

## Play around with `iio_dummy`

### The configfs

The last step to create your dummy device, it is mount a `configfs` partion in any place that you want. I prefer to it in the `/mnt` directory, as you can see the command below:

```bash
sudo mkdir /mnt/iio_experiments/
sudo mount -t configfs none /mnt/iio_experiments/
```
### Look inside `/sys` after the driver load

TODO: MOSTRAR A PASTA SYS DEPOIS DO COMANDO

### Busy device

TODO: EXPLICAR AQUELE PROBLEMA DA REMOÇÃO SENDO IMPEDIDA 

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

Done! Compile and install the module again to test the new channels. You should see something similar to:

```bash
```

### Test the changes

## References

1. [Compile In-tree Driver]({{ site.baseurl }}{% post_url 2017-02-26-compile-in-tree-kernel-module %})
2. [The `iio_dummy` Anatomy]({{ site.baseurl }}{% post_url 2017-02-26-compile-in-tree-kernel-module %})
