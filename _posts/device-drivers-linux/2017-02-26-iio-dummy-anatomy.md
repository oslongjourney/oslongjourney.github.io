---
layout: post
title:  "IIO dummy anatomy"
date:   2017-02-26
published: true
categories: linux-kernel
---

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
    /*
     * sampling_frequency
     * The frequency in Hz at which the channels are sampled
     */
    .info_mask_shared_by_dir = BIT(IIO_CHAN_INFO_SAMP_FREQ),
    /* The ordering of elements in the buffer via an enum */
    .scan_index = DUMMY_INDEX_VOLTAGE_0,
    .scan_type = { /* Description of storage in buffer */
      .sign = 'u', /* unsigned */
      .realbits = 13, /* 13 bits */
      .storagebits = 16, /* 16 bits used for storage */
      .shift = 0, /* zero shift */
},

```

## The read_raw Function

## The write_raw Function

## Putting Things Together With Probe Function

Every IIO device is directly related with a sensor, and provide the required informations for handling the device [1]. 

## References

1. [IIO Documentation](https://01.org/linuxgraphics/gfx-docs/drm/driver-api/iio/index.html)
2. [Slides with an overview of IIO subsystem](https://pt.slideshare.net/anilchowdary2050/127-iio-anewsubsystem)
