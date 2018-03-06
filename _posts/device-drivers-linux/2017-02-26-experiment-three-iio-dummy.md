---
layout: post
title:  "IIO Dummy Experiment Three: Play with IIO Generic Buffers and Trigger"
date:   2017-02-26
published: true
categories: linux-kernel
---

This experiment is the last part of the series that uses Daniel's tasks [1] as
a study guide for IIO subsystem. We provided the following posts:

1. [The iio_simple_dummy Anatomy]({{ site.baseurl }}{% post_url 2017-02-26-iio-dummy-anatomy %})
2. [IIO Dummy module Experiment One: Play with iio_dummy]({{ site.baseurl }}{% post_url 2017-02-26-experiment-one-iio-dummy %})
3. [The iio_simple_dummy_event Events Anatomy]({{ site.baseurl }}{% post_url 2017-02-26-iio-dummy-events-anatomy %})
4. [IIO Dummy Experiment Two: Play with IIO Events]({{ site.baseurl }}{% post_url 2017-02-26-experiment-two-iio-dummy %})

The experiment described in this here aims to accomplish the following tasks:

1. Load the `hrtimer` software trigger;
2. Create a trigger using `configfs`;
3. Inspect `/sys/bus/iio/`;
4. Compile and use `iio_generic_buffer`;

At this moment, I suppose that you are comfortable with compile, load/unload,
and know a little about `/sys/bus/iio`. This post intends to be straightforward
because I presume you already read the previous post or have a good experience
with this kind of tasks.

## Load and unload `hrtimer`

We begin this experiment with the load of the required modules. Before you try
to load the `iio-trig-hrtimer` module, check if you have the `iio-trig-hrtimer`
in your system as follows:

```bash
$ modinfo iio-trig-hrtimer
filename:       /lib/modules/4.16.0-rc4-TORVALDS+/kernel/drivers/iio/trigger/iio-trig-hrtimer.ko.xz
license:        GPL v2
description:    Periodic hrtimer trigger for the IIO subsystem
author:         Daniel Baluta <daniel.baluta@intel.com>
author:         Marten Svanfeldt <marten@intuitiveaerial.com>
srcversion:     F98F2B7577CDE7CC414F0A4
depends:        industrialio-sw-trigger,industrialio
retpoline:      Y
intree:         Y
name:           iio_trig_hrtimer
vermagic:       4.16.0-rc4-TORVALDS+ SMP preempt mod_unload modversions
```

In my particular case, I usually use a customize kernel version and sometimes I
have to configure and compile a specific module manually. If you want to enable
the `iio-trig-*` follows the step in Figure 1.

{% capture fig_img %}
![Foo]({{ site.url }}/images/experiment-three/hrtimer-trigger.png)
{% endcapture %}
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption>Fig 1: Enabling iio-trig-hrtimer </figcaption>
</figure>

Finally, load `iio-trig-hrtimer` and `iio_dummy`:

```bash
$ sudo modprobe iio-trig-hrtimer
$ sudo modprobe iio_dummy
```

## Create a trigger with `configfs`

You already know the `configfs` filesystem from the previous posts. Now, we
want to use it for creating a trigger named `t1`.

```bash
sudo mkdir /mnt/iio_experiments/
sudo mount -t configfs none /mnt/iio_experiments/
sudo mkdir /mnt/iio_experiments/iio/triggers/hrtimer/t1
sudo mkdir /mnt/iio_experiments/iio/devices/dummy/iio_test/
```

Now, look at `/sys/bus/iio/devices/`:

```bash
$ ls /sys/bus/iio/devices/
iio:device0  iio_evgen  trigger0

$ ls /sys/bus/iio/devices/trigger0
name  power  sampling_frequency  subsystem  uevent
```

Notice, the directory `trigger0` and the elements inside it. You will see that
our `t1` trigger is finally created.

## Compile and using `iio_generic_buffer`

Compile tools from `tools/iio` directory with the following command:

```bash
make -C tools/iio
```

Now, let's just look at the `iio_generic_buffer` options with the command:

```bash
$ ./tools/iio/iio_generic_buffer -h
./tools/iio/iio_generic_buffer: invalid option -- 'h'
Usage: generic_buffer [options]...
Capture, convert and output data from IIO device buffer
  -a         Auto-activate all available channels
  -A         Force-activate ALL channels
  -c <n>     Do n conversions
  -e         Disable wait for event (new data)
  -g         Use trigger-less mode
  -l <n>     Set buffer length to n samples
  --device-name -n <name>
  --device-num -N <num>
        Set device by name or number (mandatory)
  --trigger-name -t <name>
  --trigger-num -T <num>
        Set trigger by name or number
  -w <n>     Set delay between reads in us (event-less mode)
```

There are many interesting options, however, for this tutorial we are
interested in only two of them: `-n` and `-t`. So, let's try to see what
happens during the command execution:

```bash
[root@atma]# ./tools/iio/iio_generic_buffer -n iio_test -t t1
iio device number being used is 0
iio trigger number being used is 0
No channels are enabled, we have nothing to scan.
Enable channels manually in /sys/bus/iio/devices/iio:device0/scan_elements/*_en or pass -a to autoenable channels and try again.
```

Ooops... It does not work as expected, no problem! The output message is very
informative. We forgot to enable the channel; we can just use the `-a` option.
Let's try:

```bash
[root@atma]# ./tools/iio/iio_generic_buffer -a -n iio_test -t t1
iio device number being used is 0
iio trigger number being used is 0
Enabling all channels
Enabling: in_voltage3-voltage4_en
Enabling: in_accel_x_en
Enabling: in_voltage1-voltage2_en
Enabling: in_timestamp_en
Enabling: in_voltage0_en
/sys/bus/iio/devices/iio:device0 t1
0.018662 -0.000044 -0.000003 344.000000 1520363584895707167 
0.018662 -0.000044 -0.000003 344.000000 1520363584905685726 
Disabling: in_voltage3-voltage4_en
Disabling: in_accel_x_en
Disabling: in_voltage1-voltage2_en
Disabling: in_timestamp_en
Disabling: in_voltage0_en
```

## References

1. [IIO tasks proposed by Daniel Baluta](https://kernelnewbies.org/IIO_tasks)
1. `tools/iio/iio_event_monitor.c` the source code has many useful comments.
2. [The iio_simple_dummy_event Events Anatomy]({{ site.baseurl }}{% post_url 2017-02-26-iio-dummy-events-anatomy %})
