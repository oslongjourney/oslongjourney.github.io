---
layout: post
title:  "The iio_dummy Events Anatomy"
date:   2017-02-26
published: true
categories: linux-kernel
---

For several days I dove into the Industrial I/O (IIO) subsystem as an attempt
to finish the set of tasks proposed by Daniel Baluta [3]. I decided to put a
lot of efforts in Daniel's assignments, due to my plans to write a device
drivers for IIO subsystem future (I hope). As a result, I decided to
share/register all the knowledge acquired after long hours studying the
`iio_dummy` module (I hope to support newcomer like me). This "anatomy study"
is a part of a set of post based on the IIO tasks. Finally, you will find some
general information about the IIO system and details about the `iio_dummy`
module internals.

## Introduction

Usually, I approach a new Device Drivers (DD) that I want to learn
systematically. First, I try to identify the initialization function. Second, I
concentrate efforts to discover the essential elements of the subsystem. Third,
I seek the read/write functions and dissect them to comprehend the overall
behavior of them. Finally, I manage to put all the concepts together. I do not
know if it is a great approach, but I use it in this post.

The best place to begin with IIO subsystem, it is the `iio_dummy` module, which
is a toy application. This module has thousands of valuable comments that help
to understand the IIO internal. We look at the `iio_dummy`, starting on
`iio_simple_dummy`; then we go through `iio_simple_dummy_event`,
`iio_dummy_evgen`, and `iio_simple_dummy_buffer` in an other posts.

## The IIO Dummy Channels Setup

Let's start the `iio_dummy` dissection by looking at the struct `iio_chan_spec`
since it provides several valuable information that helps to understand the
implementation of the other modules. Also, this struct says a lot about the
target devices. We have to start by define and explain the concept of a
channel:

Definition: Channel
: An IIO device channel is a representation of a data channel; A single IIO
device may have one or more channels [1].

To better understand this concept, the IIO documentation provides the following
excellent examples [1]:

* An accelerometer can have up to 3 channels representing acceleration on X, Y,
  and Z axes;
* A light sensor with two channels indicating the measurements in the visible
  and infrared spectrum;
* A thermometer sensor has one channel representing the temperature measurement.

To represent a channel in the IIO subsystem it is necessary to use a struct
named `iio_chan_spec`, declared at `include/linux/iio/iio.h`, which has
arguments to set up the channel. The `iio_chan_spec` is massively configurable;
this feature adds flexibility to the subsystem because the same struct is used
to configure a different set of devices. It is worth look at the code
documentation located on top of this struct definition.

The `iio_dummy` create multiple channels to simulate a variety of sensors. As a
result, this part of the code is a little bit large although simple. See the
snippet of code extracted from  `drivers/iio/dummy/iio_simple_dummy.c`:


```c
static const struct iio_chan_spec iio_dummy_channels[] = {
  /* indexed ADC channel in_voltage0_raw etc */
  {
    .type = IIO_VOLTAGE,
    /* Channel has a numeric index of 0 */
    .indexed = 1,
    .channel = 0,
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 1: First part of iio_dummy channel configurations </figcaption>
</figure>

Code 1 is the beginning of the `iio_dummy_channels` declaration. It is an array
of `iio_chan_spec` in which each element describes a different channel. As we
can see in the first element, the field of the first element it is `.type`.
This field specifies the type of measurements collected by the channel, in this
case, it is `IIO_VOLTAGE`. 

In the above piece of code (Code 1), pay attention to the `.indexed` and
`.channel` field. The `.indexed` value is a bit field element. The value '1' in
this field means that the channel has a numerical index; otherwise, there is no
index value. The `.channel` is the number index assigned to the channel. In
summary, this is a way to distinguish multiple data channels in the same
channel (remember the example of the light sensor).

```c
    /* What other information is available? */
    .info_mask_separate =
    /*
     * in_voltage0_raw
     * Raw (unscaled no bias removal etc) measurement
     * from the device.
     */
    BIT(IIO_CHAN_INFO_RAW) |
    /*
     * in_voltage0_offset
     * Offset for userspace to apply prior to scale
     * when converting to standard units (microvolts)
     */
    BIT(IIO_CHAN_INFO_OFFSET) |
    /*
     * in_voltage0_scale
     * Multipler for userspace to apply post offset
     * when converting to standard units (microvolts)
     */
    BIT(IIO_CHAN_INFO_SCALE),
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 2: .info_mask_separate field </figcaption>
</figure>

The field `.info_mask_separate` designates which attributes are specific for
this channel. Read the comment on the options to understand the meaning of each
one (the comments are very descriptive).

**Remeber:**
Go back and look again at the above fields, we will see it again in the next
section. They play a very important role in the device.
{: .notice_info}


```c
    /*
     * sampling_frequency
     * The frequency in Hz at which the channels are sampled
     */
    .info_mask_shared_by_dir = BIT(IIO_CHAN_INFO_SAMP_FREQ),
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 3: .info_mask_shared_by_dir field </figcaption>
</figure>

The field `info_mask_shared_by_dir` it is the information mask shared by
direction; which means that the channel information is shared with other
channels that have the same direction.

```c
    /* The ordering of elements in the buffer via an enum */
    .scan_index = DUMMY_INDEX_VOLTAGE_0,
    .scan_type = { /* Description of storage in buffer */
      .sign = 'u', /* unsigned */
      .realbits = 13, /* 13 bits */
      .storagebits = 16, /* 16 bits used for storage */
      .shift = 0, /* zero shift */
},

```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 4: .scan_index and .scan_type field </figcaption>
</figure>

The `.scan_index` and `.scan_type` are directly related to buffers. If
`.scan_index` receives -1, it means the channels do not support buffered
capture hence the `.scan_type` is not necessary in this case. However, if
`.scan_index` gets a positive number it means the channel implements a buffer;
it is important to highlight that `.scan_index` must be unique. The
`.scan_index` value is used to organize the order in which the enabled channels
are arranged inside the buffer. Going back to the Code 4, `.scan_index` is 0
because of the `DUMMY_INDEX_VOLTAGE_0` which is declared in
`dummy/iio_simple_dummy.h`. Finally, the `.scan_type` represents the buffer
data description.

```c
#ifdef CONFIG_IIO_SIMPLE_DUMMY_EVENTS
		.event_spec = &iio_dummy_event,
		.num_event_specs = 1,
#endif /* CONFIG_IIO_SIMPLE_DUMMY_EVENTS */
	},
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 5: .event_spec </figcaption>
</figure>

There is two final configuration at the end of the first channel, both related
to events. Notice that events became available if
`CONFIG_IIO_SIMPLE_DUMMY_EVENTS` only if it is enabled in the `.config` file.
In a few words, the `.event_spec` field register an array of events, and the
`.num_event_specs` is the size of the array. We discuss events in the "X". 

```c
/* Differential ADC channel in_voltage1-voltage2_raw etc*/
	{
		.type = IIO_VOLTAGE,
		.differential = 1,
```

TODO: I don't know what is differential, it could be nice to discover and add a brief explanation with example here.

```c
		/*
		 * Indexing for differential channels uses channel
		 * for the positive part, channel2 for the negative.
		 */
		.indexed = 1,
		.channel = 1,
		.channel2 = 2,
```

The field `.channel2` is tied with the `.differential` characteristic, it keep a second number for the differential channel.

```c
		/*
		 * in_voltage1-voltage2_raw
		 * Raw (unscaled no bias removal etc) measurement
		 * from the device.
		 */
		.info_mask_separate = BIT(IIO_CHAN_INFO_RAW),
		/*
		 * in_voltage-voltage_scale
		 * Shared version of scale - shared by differential
		 * input channels of type IIO_VOLTAGE.
		 */
		.info_mask_shared_by_type = BIT(IIO_CHAN_INFO_SCALE),
```

The field `.info_mask_shared_by_type` is a channel attribute shared by all channels of the same type. The rest of this channel definition is similar to the last one described, because of this we skip the rest of the code and look at the next channel.

```c
/*
	 * 'modified' (i.e. axis specified) acceleration channel
	 * in_accel_z_raw
	 */
	{
		.type = IIO_ACCEL,
```

Here we have a different type of device represented by the `IIO_ACCEL` which means acceleration channel.

```c
		.modified = 1,
		/* Channel 2 is use for modifiers */
		.channel2 = IIO_MOD_X,
```

Here we have another technique to distinguish between data channels in the same channel. If the field `.modified` is assigned the `.channel2` receive another meaning, by expecting an unique characteristic for the channel. In this case, this channel receives `IIO_MOD_X`. The rest of the channel definition is similar to the last one. Let's jump to another different thing.

```c
	/*
	 * Convenience macro for timestamps. 4 is the index in
	 * the buffer.
	 */
	IIO_CHAN_SOFT_TIMESTAMP(4),
```

TODO: I did not understand what is this macro

Stop your reading for a while, and go to the code and read `iio_chan_spec` for `iio_dummy` module. Then came back here.

## The `*read_raw` Function

Now it is time to look for some pieces of code. Do you remember the `.type` field from the last section? So, here this field will play an important role.

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
```

The parameters `*val` and `*val2` will be filled in this function in order to return an read value. The `mask` argument, provides the required information for helping to identify the correct way to read the channel value. Finally, look at the `mutex_lock`, all the read operation should happen in a safer way and the lock is reposible to keep the consistence for the read data.  

```c
	switch (mask) {
	case IIO_CHAN_INFO_RAW: /* magic value - channel value read */
		switch (chan->type) {
		case IIO_VOLTAGE:
			if (chan->output) {
				/* Set integer part to cached value */
				*val = st->dac_val;
				ret = IIO_VAL_INT;
			} else if (chan->differential) {
				if (chan->channel == 1)
					*val = st->differential_adc_val[0];
				else
					*val = st->differential_adc_val[1];
				ret = IIO_VAL_INT;
			} else {
				*val = st->single_ended_adc_val;
				ret = IIO_VAL_INT;
			}
			break;
		case IIO_ACCEL:
			*val = st->accel_val;
			ret = IIO_VAL_INT;
			break;
		default:
			break;
		}
		break;
```

Notice that `mask` refers to the field `.info_mask_separate`, and now we use it to identify the correct channel to read. Second, we inspect the `type` field and based on it an action are taken. In this section of the code the `*val` pointer receives a value provided by the `st` variable that have the sensor state. We will stop the explanation of the read function here, because the rest of it is similar to the above code.

## The `*write_raw` Function

The `*write_raw()` functions have to identify which channel info, invoke the write operation and then verify the type which is very similar to the `*read_raw()` function. Different from `read_raw` function, that takes values from the state and save in `val` and `val`. Let's start to look the important things about `*write_function`:


```c
static int iio_dummy_write_raw(struct iio_dev *indio_dev,
			       struct iio_chan_spec const *chan,
			       int val,
			       int val2,
			       long mask)
{
	int i;
	int ret = 0;
	struct iio_dummy_state *st = iio_priv(indio_dev);

	switch (mask) {
	case IIO_CHAN_INFO_RAW:
		switch (chan->type) {
		case IIO_VOLTAGE:
			if (chan->output == 0)
				return -EINVAL;

			/* Locking not required as writing single value */
			mutex_lock(&st->lock);
			st->dac_val = val;
			mutex_unlock(&st->lock);
			return 0;
		default:
			return -EINVAL;
		}
```

As can be seen in the above snippet of code, it is very similar to `read_raw()`. Notice, that locking is not on the whole `switch` it is just in a small part related to the state since this is the critical section of the code.

## Putting Things Together with Probe Function

Until now, we just looked at separated parts of each element of the `iio_dummy` module. Now it is time to understand how things connects with each other. Typically IIO drivers register itself as an I2C or SPI driver [1], and requires two functions: `probe` and `remove`. The former is reponsable for allocate memory, initialize device fields, and register the device. The latter, basically undo what `probe` function did.

Let's start to looking at `iio_dummy_probe`:

```c
static struct iio_sw_device *iio_dummy_probe(const char *name)
```

This signature is specifically used for software device, normally, an interface that works with I2C or SPI have a different signature.

```c
	int ret;
	struct iio_dev *indio_dev;
	struct iio_dummy_state *st;
	struct iio_sw_device *swd;

	swd = kzalloc(sizeof(*swd), GFP_KERNEL);
	if (!swd) {
		ret = -ENOMEM;
		goto error_kzalloc;
	}
	/*
	 * Allocate an IIO device.
	 *
	 * This structure contains all generic state
	 * information about the device instance.
	 * It also has a region (accessed by iio_priv()
	 * for chip specific state information.
	 */
	indio_dev = iio_device_alloc(sizeof(*st));
	if (!indio_dev) {
		ret = -ENOMEM;
		goto error_ret;
	}
```

The `probe` function has three main main tasks. The first task it is allocate memory for an IIO device, in the above code we can seen the allocation space for keep the `iio_dummy_state` in memory.

```c

	st = iio_priv(indio_dev);
	mutex_init(&st->lock);

	iio_dummy_init_device(indio_dev);
	/*
	 * With hardware: Set the parent device.
	 * indio_dev->dev.parent = &spi->dev;
	 * indio_dev->dev.parent = &client->dev;
	 */

	 /*
	 * Make the iio_dev struct available to remove function.
	 * Bus equivalents
	 * i2c_set_clientdata(client, indio_dev);
	 * spi_set_drvdata(spi, indio_dev);
	 */
	swd->device = indio_dev;

	/*
	 * Set the device name.
	 *
	 * This is typically a part number and obtained from the module
	 * id table.
	 * e.g. for i2c and spi:
	 *    indio_dev->name = id->name;
	 *    indio_dev->name = spi_get_device_id(spi)->name;
	 */
	indio_dev->name = kstrdup(name, GFP_KERNEL);

	/* Provide description of available channels */
	indio_dev->channels = iio_dummy_channels;
	indio_dev->num_channels = ARRAY_SIZE(iio_dummy_channels);

	/*
	 * Provide device type specific interface functions and
	 * constant data.
	 */
	indio_dev->info = &iio_dummy_info;

	/* Specify that device provides sysfs type interfaces */
	indio_dev->modes = INDIO_DIRECT_MODE;
```

The second task of `probe` it is data initialisation. Take a careful look at the code above, and you will notice the code handle basic initialisations.

```c
	ret = iio_simple_dummy_events_register(indio_dev);
	if (ret < 0)
		goto error_free_device;

	ret = iio_simple_dummy_configure_buffer(indio_dev);
	if (ret < 0)
		goto error_unregister_events;

	ret = iio_device_register(indio_dev);
	if (ret < 0)
		goto error_unconfigure_buffer;

	iio_swd_group_init_type_name(swd, name, &iio_dummy_type);

	return swd;
```

The final task of the probe function, it is the device register.

## References

1. [IIO Documentation](https://01.org/linuxgraphics/gfx-docs/drm/driver-api/iio/index.html)
2. [A nice slide about IIO which helped me to understand the subsystem](https://www.slideshare.net/anilchowdary2050/127-iio-anewsubsystem?from_action=save)
3. [IIO tasks proposed by Daniel Baluta](https://kernelnewbies.org/IIO_tasks)
