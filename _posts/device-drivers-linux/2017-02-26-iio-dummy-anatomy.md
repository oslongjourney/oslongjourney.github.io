---
layout: post
title:  "The iio_dummy Anatomy"
date:   2017-02-26
published: true
categories: linux-kernel
---

Since my times as a software engineering student, when I wrote my first bare metal code for blinking a green led, I became passionate for the relationship between hardware and software. As a result, I am extremely interested to understand how Operating Systems really works. I am specially attracted by the visualisation mechanisms, the input and output from sensors, and the kernel internals (scheduler, memory management, and filesystem). As a result, I constantly look at the Linux Kernel code in order to learn something.

For several days I dove into the Industrial I/O subsystem (IIO) subsystem as an attempt to finish the set of tasks proposed by Daniel Baluta [3]. I also put a lot of efforts, because I have plans to write a device drivers for IIO subsystem in a future (I hope). As a result my studies I intend to produce several future posts about the anatomy of some Kernel internals and here you can find my dissect of a tiny part of the IIO (in this case, `iio_dummy`). You will find some general information about the IIO and some details about the internals of the iio_dummy module.

## Introduction

I like to approach Device Drivers (DD) from a specific subsystem in a systematic way. First, I try to identify which is the function responsible for the initialization. Second, I try to understand the basic elements of the subsystem. Third, I identify the read write function and understand the overall behaviour. Finally, I work to put all the concepts together. I do not know if it is a good approach, but I used it in this post.a

A good place to start with IIO subsystem, it is the `iio_dummy` module. This is a toy application, with thousands of valuable comments that helps to understand the internals of the IIO. We look at the `iio_dummy` DD. We start focusing on `iio_simple_dummy`, then we go through `iio_simple_dummy_event`, `iio_dummy_evgen`, and `iio_simple_dummy_buffer`.

## The Channels Setup

We start the `iio_dummy` dissection by looking at the struct `iio_chan_spec`, since it provide many valuable informations that helps to understand the implementation of the other functions. Also, this struct says a lot about the devices handled by it. However, first we have to define a channel:

Definition: Channel
: An IIO device channel is a representation of a data channel. An IIO device can have one or multiple channels [1].

In practical way, the Documentation provide the following great examples [1]:

* An accelerometer can have up to 3 channels representing acceleration on X, Y, and Z axes;
* A light sensor with two channels indicating the measurements in the visible and infrared spectrum;
* A thermometer sensor has one channel representing the temperature measurement.

IIO subsystem provide a struct named `iio_chan_spec`, declared at `include/linux/iio/iio.h`, which has arguments to setup the channel. The `iio_chan_spec` is heavily configurable, this feature add flexibility to the subsystem because the same struct is used to configure different set of devices. It is worth to careful read the great documentation found on top of this struct.

The `iio_dummy` create multiple channels to simulate a variety of different sensors, as a result, this part of the code is a little bit large (although simple). See the snippet of code extracted from  `drivers/iio/dummy/iio_simple_dummy.c`:

```c
static const struct iio_chan_spec iio_dummy_channels[] = {
  /* indexed ADC channel in_voltage0_raw etc */
  {
    .type = IIO_VOLTAGE,
    /* Channel has a numeric index of 0 */
    .indexed = 1,
    .channel = 0,
```

This is the beginning of the `iio_chan_spec` declaration for the `iio_dummy` module. Notice that `iio_dummy_channels` is an array of `iio_chan_spec` and each channel is directly describe as elements of the array. Looking inside the first element we can see the field `.type` that specify this is sort of measurements collected by the channel.

Now we can see `.indexed` and `.channel`. The `.indexed` field specify that the channel has a numerical index, the value could be 1 for use index value or 0 for not. The `.channel` is the number to be assigned to the channel. In summary, this is a way to distinguish multiple data channels in the same channel (remember of the example of the light sensor).

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

The attribute `.info_mask_separate` indicates the specific attributes for this channel. Read the commend above the options to understand the meaning of each one (the comments are very descriptive).

**Remeber:**
Look again to this fields, we will see it again in the next section. They play a very important role in the device.
{: .notice_info}


```c
    /*
     * sampling_frequency
     * The frequency in Hz at which the channels are sampled
     */
    .info_mask_shared_by_dir = BIT(IIO_CHAN_INFO_SAMP_FREQ),
```

The field `info_mask_shared_by_dir` can be read as "information mask shared by direction", which means the information of this channel is shared with other channels and has the same direction. 

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

The `.scan_index` and `.scan_type` are directly related with buffers. If `.scan_index` has a number different of -1, it represents the order in which the enabled channels are placed inside the buffer. It is important to highlight that the value assigned to `.scan_index` must to be unique. In this specific case, `.scan_index` is 0 because of the `DUMMY_INDEX_VOLTAGE_0` which is declared in `dummy/iio_simple_dummy.h`. Finally, the `.scan_type` represents the data description in the buffer.

```c
#ifdef CONFIG_IIO_SIMPLE_DUMMY_EVENTS
		.event_spec = &iio_dummy_event,
		.num_event_specs = 1,
#endif /* CONFIG_IIO_SIMPLE_DUMMY_EVENTS */
	},
```

For finish the specification of this channel, we have a configuration for events (only work if events is enabled). The `.event_spec` register an array of events to be registered, and the `.num_event_specs` is the size of the array. With this, we finish the first channel. Follows the second channel:

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
