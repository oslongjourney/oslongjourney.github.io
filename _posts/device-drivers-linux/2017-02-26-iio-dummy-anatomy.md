---
layout: post
title:  "The iio_simple_dummy Anatomy"
date:   2017-02-26
published: true
categories: linux-kernel
---

For several days I dove into the Industrial I/O (IIO) subsystem as an attempt
to finish the set of tasks proposed by Daniel Baluta [3]. I decided to put a
lot of effort in Daniel's assignments, due to my plans to write device
drivers for the IIO subsystem in the near future (I hope). As a result, I decided to
share/register all the knowledge acquired after long hours studying the
`iio_dummy` module (I believe this will help newcomers like me). This "anatomy study"
is a part of a set of posts based on the IIO tasks (Acho essa frase
desnecessária). Finally, you will find general information about the
IIO system and details about the `iio_dummy` module internals.

## Introduction

Usually, I approach a new Device Driver (DD) that I want to learn
systematically. First, I try to identify the initialization function. Second, I
concentrate efforts on discovering the essential elements of the subsystem. Third,
I seek the read/write functions and dissect them to comprehend their overall
behavior. Finally, I manage to put all the concepts together. I do not
know if it is a great approach, but I use it in this post.

The best place to begin with IIO subsystem, is the `iio_dummy` module, which
is a toy application. This module has thousands of valuable comments that help
to understand the IIO internal. We look at the `iio_dummy`, starting on
`iio_simple_dummy`; then we go through `iio_simple_dummy_event`,
`iio_dummy_evgen`, and `iio_simple_dummy_buffer` in another posts.

## The IIO Dummy Channels Setup

Let's start the `iio_dummy` dissection by looking at the struct `iio_chan_spec`
since it provides several valuable information that helps understanding the
implementation of the other modules. Also, this struct says a lot about the
target devices. We have to start off by defining and explaining the concept of a
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
to configure different set of devices. It is worth looking at the code
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
of `iio_chan_spec`, where each element describes a different channel. The field
of the first element it is `.type`.
This field specifies the type of measurements collected by the channel, in this
case, it is an `IIO_VOLTAGE`. 

In the above piece of code (Code 1), pay attention to the `.indexed` and
`.channel` field. The `.indexed` value is a bit field element. The value '1' in
this field means that the channel has a numerical index; otherwise, there is no
index value. The `.channel` is the index number assigned to the channel. In
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

**Remember:**
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

The field `info_mask_shared_by_dir` is the information mask shared by
direction; which means that the channel information is shared with other
channels that have the same direction.
(Na real que não entendi aqui o que você quer dizer com direction...)

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
are arranged inside the buffer. Going back to Code 4, `.scan_index` is 0
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

There are two final configurations at the end of the first channel, both related
to events. Notice that events became available only if
`CONFIG_IIO_SIMPLE_DUMMY_EVENTS` is enabled in the `.config` file.
In a few words, the `.event_spec` field register an array of events, and the
`.num_event_specs` is the size of the array. We discuss events in the "X" (Faltou atualizar esse X). 
"[The iio_dummy Events Anatomy]({{ site.baseurl }}{% post_url 2017-02-26-iio-dummy-events-anatomy %})"

Now we keep looking the other channels; however, we do not repeat ourselves and
just explain new things. See:

```c
/* Differential ADC channel in_voltage1-voltage2_raw etc*/
	{
		.type = IIO_VOLTAGE,
		.differential = 1,
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 6: Differential </figcaption>
</figure>

Code 6 shows an ADC converter, which is configured as a differential.

```c
		/*
		 * Indexing for differential channels uses channel
		 * for the positive part, channel2 for the negative.
		 */
		.indexed = 1,
		.channel = 1,
		.channel2 = 2,
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 7: Multiple Channels </figcaption>
</figure>

The field `.channel2` and `.differential` are directly related to the above
Code; The `.channel2` keeps the differential value number. Here we have the
first example of a single channel, with multiple information.

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
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 8: .info_mask_separate and .info_maks_shared_by_type </figcaption>
</figure>

Finally, the field `.info_mask_shared_by_type` represents a shared channel
attribute of the same type which means that other channels can access this
information. The rest of this channel definition is similar to the last one,
let's skip the rest of the code and look at the next channel.

```c
/*
	 * 'modified' (i.e. axis specified) acceleration channel
	 * in_accel_z_raw
	 */
	{
		.type = IIO_ACCEL,
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 9: .type </figcaption>
</figure>

Here we have a different type of device represented by the `IIO_ACCEL` which
means acceleration channel.

```c
		.modified = 1,
		/* Channel 2 is use for modifiers */
		.channel2 = IIO_MOD_X,
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 10: .modified </figcaption>
</figure>

Here we have another technique to distinguish between data channels in the same
channel. If the `.modified` field is assigned, the `.channel2` changes its
meaning and expects a unique characteristic for the channel. In the Code 10,
this channel receives `IIO_MOD_X`. The rest of the channel definition is
similar to the last one, so let's go to another different thing.

```c
	/*
	 * Convenience macro for timestamps. 4 is the index in
	 * the buffer.
	 */
	IIO_CHAN_SOFT_TIMESTAMP(4),
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 11: Timestamp </figcaption>
</figure>

TODO: I did not understand what is this macro

Oww... a lot of information. I recommend you stop reading for a while, and
go back to the code and read `iio_dummy_channels` in the `iio_dummy` module. Then
come back here.

## The `iio_dummy_read_raw()` Function

Now it is time to look into some pieces of code. Do you remember the `.type`
field from the last section? So, here this field will play an important role.

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
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 12: iio_dummy_read_raw </figcaption>
</figure>

The `iio_dummy_read_raw` function read data from the `iio_dummy_state` and save
it on the `*val` and `*val2` parameters. The 'mask' argument, provides the
required information to support the correct identification of the channel value
attribute. Finally, look at the `mutex_lock`; the read operation is
under the lock to keep reading operations consistent.

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
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 13: Find the correct output </figcaption>
</figure>

Notice that 'mask' refers to the field `.info_mask_separate`, which is used to
identify the correct channel to read. Second, inside the switch statement, the
`type` field (from `iio_chan_spec`) is inspected and based on its value an
action occurs. In this section of the code, the `*val` pointer receives a value
provided by the `iio_dummy_state` struct that has the sensor state. We will
stop the explanation of the read function here because the rest of it is
similar to the above code.

## The `*write_raw` Function

The `iio_dummy_write_raw()` function behaves similarly to
`iio_dummy_read_raw()`; It has to identify the channel responsible for the
action and write the information in the `iio_dummy_state`. Different
from the read function, the write function receives the values in the arguments
and writes them in the iio_dummy_state. Let's start lolooking at the important
things about
`iio_dummy_write_raw()`:

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
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 14: write data </figcaption>
</figure>

As can be seen in Code 14, the implementation resembles the read function.
However, notice that locking is not on the whole `switch`, it is just in a
small part related to the state since this is the critical section of the code.

## Putting Things Together with Probe Function

Until now, we just looked at distinct parts of each element of the `iio_dummy`
module. Now it is time to understand how things connect with each other.
Typically IIO drivers register itself as an I2C or SPI driver [1], and requires
two functions: `probe` and `remove`. The former is responsible for allocating
memory, initialize device fields, and register the device. The latter,
basically undo what `probe` function did. Let's start looking at the
`iio_dummy_probe`:

```c
static struct iio_sw_device *iio_dummy_probe(const char *name)
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 15: Signature </figcaption>
</figure>

This signature is specifically used for software devices, frequently, an
interface that works with I2C or SPI have a different signature.

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
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 16: Allocate </figcaption>
</figure>

The `probe` function has three main tasks. The first task allocates memory
for an IIO device. In Code 16 we can see the space allocated to store the
`iio_dummy_state`.

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
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 17: Data initialization </figcaption>
</figure>

The second task of `probe` is data initialization. Take a careful look at
Code 17, and you will notice how it handles basic initializations.

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
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 18: Register </figcaption>
</figure>


The final task of the probe function, is the device register.
(Acho que podia ter um encerramento melhor de parágrafo)

## References

1. [IIO Documentation](https://01.org/linuxgraphics/gfx-docs/drm/driver-api/iio/index.html)
2. [A nice slide about IIO which helped me to understand the subsystem](https://www.slideshare.net/anilchowdary2050/127-iio-anewsubsystem?from_action=save)
3. [IIO tasks proposed by Daniel Baluta](https://kernelnewbies.org/IIO_tasks)
