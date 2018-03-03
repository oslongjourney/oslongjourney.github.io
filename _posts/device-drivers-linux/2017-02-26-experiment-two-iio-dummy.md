---
layout: post
title:  "IIO Dummy Experiment Two: Play with IIO Events"
date:   2017-02-26
published: true
categories: linux-kernel
---

```bash
$ make -C tools/iio

$ ls tools/iio/
Build              iio_event_monitor.c     iio_event_monitor.o  iio_generic_buffer.c     iio_generic_buffer.o  iio_utils.h  include  lsiio.c     lsiio.o
iio_event_monitor  iio_event_monitor-in.o  iio_generic_buffer   iio_generic_buffer-in.o  iio_utils.c           iio_utils.o  lsiio    lsiio-in.o  Makefile
```

```bash
$ ./tools/iio/lsiio 
Device 000: my_glorious_dummy_device
```

```bash
$ sudo ./tools/iio/iio_event_monitor my_glorious_dummy_device
Found IIO device with name my_glorious_dummy_device with device number 0
```

```bash
 echo 0 > /sys/bus/iio/devices/iio_evgen/poke_ev0
 echo 1 > /sys/bus/iio/devices/iio_evgen poke_ev0
 echo 2 > /sys/bus/iio/devices/iio_evgen poke_ev0
 echo 3 > /sys/bus/iio/devices/iio_evgen poke_ev0
```

```bash
$ sudo ./tools/iio/iio_event_monitor my_glorious_dummy_device
Found IIO device with name my_glorious_dummy_device with device number 0
Event: time: 1520102773326469013, type: voltage, channel: 0, evtype: thresh, direction: rising
Event: time: 1520102776145258970, type: activity(running), channel: 0, evtype: thresh, direction: rising
Event: time: 1520102778636096750, type: activity(walking), channel: 0, evtype: thresh, direction: falling
Event: time: 1520102781584365213, type: steps, channel: 0, evtype: change
```
