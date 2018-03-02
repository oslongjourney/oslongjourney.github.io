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

## Enabling `iio_dummy`

## Compile Module

If you want to know the details about the compilation and load, then read my post [compile in-tree driver]({{ site.baseurl }}{% post_url 2017-02-26-compile-in-tree-kernel-module %}). For compile the `iio_dummy`, just type:

{% highlight bash %}
make M=driver/iio/dummy
{% endhighlight %}

## Load and Unload Module

## The configfs

### Mount configfs

## Play around with `iio_dummy`

## Adding Channels for a 3-axis compass

## Troubleshoot
