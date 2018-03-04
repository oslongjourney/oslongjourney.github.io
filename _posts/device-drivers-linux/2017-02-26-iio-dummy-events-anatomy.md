---
layout: post
title:  "The iio_simple_dummy_event Events Anatomy"
date:   2017-02-26
published: true
categories: linux-kernel
---

Here we continue with our dissection of `iio_dummy` module. In the "[The iio_dummy Anatomy]({{ site.baseurl }}{% post_url 2017-02-26-use-qemu-to-play-with-linux %})", we looked at the `iio_simple_dummy.c` file and with focus on channels configurations and read/write operations. In this post, we going to look at the events used by `iio_dummy`.

First, it is important to know what is an event

Definition: Trigger
: It is a mechanisms for a driver capture data based on external event (trigger) as opposed to periodically polling for data [1].

## Event Elements

The event elements are spread around different files, let's start by looking the header files. The first file to inspect it is `iio_simple_dummy.h`, see the struct in Code 1:

```c
struct iio_dummy_state {
	int dac_val;
	int single_ended_adc_val;
	int differential_adc_val[2];
	int accel_val;
    ...
#ifdef CONFIG_IIO_SIMPLE_DUMMY_EVENTS
	int event_irq;
	int event_val;
	bool event_en;
	s64 event_timestamp;
#endif /* CONFIG_IIO_SIMPLE_DUMMY_EVENTS */
};
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 1: Elements inside CONFIG_IIO_SIMPLE_DUMMY_EVENTS </figcaption>
</figure>

For simplicity sake, we omit some elements from the struct because it is necessary for the event part. Notice that all elements related to events are between the conditional directive `ifdef` based on the condition of `CONFIG_IIO_SIMPLE_DUMMY_EVENTS`. First, in another steps an irq number will be required by `iio_dummy` and the field `event_irq` keeps this value. Second, it is `event_val` element responsible for store any event value. Third, it is the `event_en` which is used to check if the event is enabled or not. Finally, the `event_timestamp` register the timestamp of the event.

```c
#ifdef CONFIG_IIO_SIMPLE_DUMMY_EVENTS

struct iio_dev;

int iio_simple_dummy_read_event_config(struct iio_dev *indio_dev,
				       const struct iio_chan_spec *chan,
				       enum iio_event_type type,
				       enum iio_event_direction dir);

int iio_simple_dummy_write_event_config(struct iio_dev *indio_dev,
					const struct iio_chan_spec *chan,
					enum iio_event_type type,
					enum iio_event_direction dir,
					int state);

int iio_simple_dummy_read_event_value(struct iio_dev *indio_dev,
				      const struct iio_chan_spec *chan,
				      enum iio_event_type type,
				      enum iio_event_direction dir,
				      enum iio_event_info info, int *val,
				      int *val2);

int iio_simple_dummy_write_event_value(struct iio_dev *indio_dev,
				       const struct iio_chan_spec *chan,
				       enum iio_event_type type,
				       enum iio_event_direction dir,
				       enum iio_event_info info, int val,
				       int val2);

int iio_simple_dummy_events_register(struct iio_dev *indio_dev);
void iio_simple_dummy_events_unregister(struct iio_dev *indio_dev);

#else /* Stubs for when events are disabled at compile time */
...
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 2: Event functions signatures </figcaption>
</figure>

The Code 2 illustrate the methods used by the event operations. Functions `iio_simple_dummy_read_event_config` and `iio_simple_dummy_write_event_config` are dedicated to read and write configuration data. The configuration steps it is flexible to allows user to configure it or just get information. For read and write information from event, there is function `iio_simple_dummy_read_event_value` and `iio_simple_dummy_write_event_value`. Finally, there is two signature for register and unregister events. The implementation of these is in the `iio_simple_dummy_events.c` file, before we digging in this file we going to look at `iio_simple_dummy.c` 

## Event elements in the `iio_simple_dummy.c`

The `iio_simple_dummy.c` keeps the main code related to the `iio_dummy` module, as a result this file has some important definitions for event. See the code below extracted from `iio_simple_dummy.c`:

```c
#ifdef CONFIG_IIO_SIMPLE_DUMMY_EVENTS

/*
 * simple event - triggered when value rises above
 * a threshold
 */
static const struct iio_event_spec iio_dummy_event = {
	.type = IIO_EV_TYPE_THRESH,
	.dir = IIO_EV_DIR_RISING,
	.mask_separate = BIT(IIO_EV_INFO_VALUE) | BIT(IIO_EV_INFO_ENABLE),
};

/*
 * simple step detect event - triggered when a step is detected
 */
static const struct iio_event_spec step_detect_event = {
	.type = IIO_EV_TYPE_CHANGE,
	.dir = IIO_EV_DIR_NONE,
	.mask_separate = BIT(IIO_EV_INFO_ENABLE),
};

/*
 * simple transition event - triggered when the reported running confidence
 * value rises above a threshold value
 */
static const struct iio_event_spec iio_running_event = {
	.type = IIO_EV_TYPE_THRESH,
	.dir = IIO_EV_DIR_RISING,
	.mask_separate = BIT(IIO_EV_INFO_VALUE) | BIT(IIO_EV_INFO_ENABLE),
};

/*
 * simple transition event - triggered when the reported walking confidence
 * value falls under a threshold value
 */
static const struct iio_event_spec iio_walking_event = {
	.type = IIO_EV_TYPE_THRESH,
	.dir = IIO_EV_DIR_FALLING,
	.mask_separate = BIT(IIO_EV_INFO_VALUE) | BIT(IIO_EV_INFO_ENABLE),
};
#endif
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 2: Event channel specification </figcaption>
</figure>

Code 2 illustrate all the events defined for `iio_dummy`, notice that each one is a struct of `iio_event_spec` which is very similar to `iio_chan_spec`. The `.type` field specifies the type of the event, which can be: `IIO_EV_TYPE_THRESH`, `IIO_EV_TYPE_MAG`, `IIO_EV_TYPE_ROC`, `IIO_EV_TYPE_THRESH_ADAPTIVE`, `IIO_EV_TYPE_MAG_ADAPTIVE` or `IIO_EV_TYPE_CHANGE`. The `.dir` field describes the direction, there is four possible value for it: `IIO_EV_DIR_EITHER`, `IIO_EV_DIR_RISING`, `IIO_EV_DIR_FALLING`, and `IIO_EV_DIR_NONE`. Finally, the `.mask_separate` configures the mask set to the channel which can be: `IIO_EV_INFO_ENABLE`, `IIO_EV_INFO_VALUE`, `IIO_EV_INFO_HYSTERESIS`, `IIO_EV_INFO_PERIOD`, `IIO_EV_INFO_HIGH_PASS_FILTER_3DB`, and `IIO_EV_INFO_LOW_PASS_FILTER_3DB`.

These structs are defined separately, but in some way it need to be tied with the channel. The way to register an event into the channel, it is the field `event_spec` from `iio_chan_spec`. See the code:

```c
static const struct iio_chan_spec iio_dummy_channels[] = {
	/* indexed ADC channel in_voltage0_raw etc */
	{
		.type = IIO_VOLTAGE,
		/* Channel has a numeric index of 0 */
		.indexed = 1,
		.channel = 0,
        ...
#ifdef CONFIG_IIO_SIMPLE_DUMMY_EVENTS
		.event_spec = &iio_dummy_event,
		.num_event_specs = 1,
#endif /* CONFIG_IIO_SIMPLE_DUMMY_EVENTS */
	},
    ...
	{
		.type = IIO_STEPS,
        ...
#ifdef CONFIG_IIO_SIMPLE_DUMMY_EVENTS
		.event_spec = &step_detect_event,
		.num_event_specs = 1,
#endif /* CONFIG_IIO_SIMPLE_DUMMY_EVENTS */
	},
	{
		.type = IIO_ACTIVITY,
		...
#ifdef CONFIG_IIO_SIMPLE_DUMMY_EVENTS
		.event_spec = &iio_running_event,
		.num_event_specs = 1,
#endif /* CONFIG_IIO_SIMPLE_DUMMY_EVENTS */
	},
	{
		.type = IIO_ACTIVITY,
		...
#ifdef CONFIG_IIO_SIMPLE_DUMMY_EVENTS
		.event_spec = &iio_walking_event,
		.num_event_specs = 1,
#endif /* CONFIG_IIO_SIMPLE_DUMMY_EVENTS */
	},
...
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 3: Event inside iio_chan_spec </figcaption>
</figure>

Notice from Code 3 that each `iio_event_spec` was assigned to the `.event_spec` field.

Finally, it is necessary to fill out the `iio_info` struct as the code below shows:

```c
/*
 * Device type specific information.
 */
static const struct iio_info iio_dummy_info = {
	.read_raw = &iio_dummy_read_raw,
	.write_raw = &iio_dummy_write_raw,
#ifdef CONFIG_IIO_SIMPLE_DUMMY_EVENTS
	.read_event_config = &iio_simple_dummy_read_event_config,
	.write_event_config = &iio_simple_dummy_write_event_config,
	.read_event_value = &iio_simple_dummy_read_event_value,
	.write_event_value = &iio_simple_dummy_write_event_value,
#endif /* CONFIG_IIO_SIMPLE_DUMMY_EVENTS */
};
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 4: The iio_info </figcaption>
</figure>

With Code 4, we finish our hunting for event elements inside `iio_simple_dummy.c`. 

## The read/write config

Now, we start to look at the core of the `iio_simple_dummy_events.c`; this file has the implementation of the functions signature from Code 2. All the read/write operation related with the event, directly work with the `iio_dummy_state` and the field present in Code 1. Let's start looking at the `iio_simple_dummy_write_event_config`:

```c
int iio_simple_dummy_write_event_config(struct iio_dev *indio_dev,
					const struct iio_chan_spec *chan,
					enum iio_event_type type,
					enum iio_event_direction dir,
					int state)
{
	struct iio_dummy_state *st = iio_priv(indio_dev);
	/*
	 *  Deliberately over the top code splitting to illustrate
	 * how this is done when multiple events exist.
	 */
	switch (chan->type) {

```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 4: The iio_info </figcaption>
</figure>

The `iio_simple_dummy_write_event_config` receives many parameters that helps to identify the correct channel. The `iio_dummy_state` it is retrieve at the beginning of the read function and it going to be used in many places. Finally, the `chan` parameter provides a way to correctly identify the type. 

```c
	case IIO_VOLTAGE:
		switch (type) {
		case IIO_EV_TYPE_THRESH:
			if (dir == IIO_EV_DIR_RISING)
				st->event_en = state;
			else
				return -EINVAL;
			break;
		default:
			return -EINVAL;
		}
		break;
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 5: The event type verification </figcaption>
</figure>

Code 5 illustrate the processes to write a new configuration to the right channel. Remember from the channel configuration in the previously section that the first channel was defined as `IIO_VOLTAGE` and the event assotiated with it has the type `IIO_EV_TYPE_THRESH`. Finally, we are interested only in rising event to update the state. The rest of this function, follow a similar pattern.

The `iio_simple_dummy_read_event_config()` it is very simple, and it just return the value of `st->event_en`. 

## The read/write value

The read/write value are two simple funtion responsible for handle the `event_val` field from the struct `iio_dummy_state`.

```c
int iio_simple_dummy_read_event_value(struct iio_dev *indio_dev,
				      const struct iio_chan_spec *chan,
				      enum iio_event_type type,
				      enum iio_event_direction dir,
				      enum iio_event_info info,
				      int *val, int *val2)
{
	struct iio_dummy_state *st = iio_priv(indio_dev);

	*val = st->event_val;

	return IIO_VAL_INT;
}

int iio_simple_dummy_write_event_value(struct iio_dev *indio_dev,
				       const struct iio_chan_spec *chan,
				       enum iio_event_type type,
				       enum iio_event_direction dir,
				       enum iio_event_info info,
				       int val, int val2)
{
	struct iio_dummy_state *st = iio_priv(indio_dev);

	st->event_val = val;

	return 0;
}
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 6: The read/write value </figcaption>
</figure>

## The Event Handling Function

The `iio_simple_dummy_event_handler()` function is responsible for handling the incoming event. For handling the event, the function inspect the value in the `reg_data` (from `iio_dummy_state`) in order to check the option. In the specific case of `iio_dummy`, there is four options (case 0 to 3) in which each option represents one sort of event. Finally, after the target event is identified the `iio_push_event` add the event to the list of userspace reading.

```c
static irqreturn_t iio_simple_dummy_event_handler(int irq, void *private)
{
	struct iio_dev *indio_dev = private;
	struct iio_dummy_state *st = iio_priv(indio_dev);

	dev_dbg(&indio_dev->dev, "id %x event %x\n",
		st->regs->reg_id, st->regs->reg_data);
	switch (st->regs->reg_data) {
	case 0:
		iio_push_event(indio_dev,
			       IIO_EVENT_CODE(IIO_VOLTAGE, 0, 0,
					      IIO_EV_DIR_RISING,
					      IIO_EV_TYPE_THRESH, 0, 0, 0),
			       st->event_timestamp);
		break;
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 7: Handler per event </figcaption>
</figure>

Code 7 illustrate the processes of get the information from `reg_data`, verify the option, and if it is 0 push the event. Notice the macro `IIO_EVENT_CODE` which is responsible for create an event identifier. Finally, the timestamp related to the event is also stored. The rest of the code, follows a similar patterns just by changing the set of flags.

**Attention:**
If you play with `iio_dummy` events, you will notice that `iio_simple_dummy_event_handler` play an important task in the event handler processes.
{: .notice_danger}

## References

1. [IIO Documentation](https://01.org/linuxgraphics/gfx-docs/drm/driver-api/iio/index.html)
2. [A nice slide about IIO which helped me to understand the subsystem](https://www.slideshare.net/anilchowdary2050/127-iio-anewsubsystem?from_action=save)
