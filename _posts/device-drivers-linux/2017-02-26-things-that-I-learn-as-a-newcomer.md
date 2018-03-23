---
layout: post
title:  "Things that I Learned as a Newcomer in the Linux Kernel"
date:   2017-02-26
published: true
categories: linux-kernel
---

In this post I will not describe how to send a patch for Linux Kernel because you can easy find a tons of post related to it. Additionally you can read the great post in the Kernel Newbies ([Kernel Newbies - First Kernel Patch](https://kernelnewbies.org/FirstKernelPatch)) that describes how you can send your first patch. This post is about my experience as a newcomer, without any help, and working on it only on my free time. I had send many patches, with this I got many feedbacks from the community and learned a lot with may mistakes. I will try to summarize here, many of the things not directly documented that I learned and discuss some of my mistakes.

## Do not Send a new Version of the Patch as a Reply to the old Patch

Fast Reply: Always send the updated patches in a new threads.

Discussion:

When I got my first patch reviewed, I understood that I had to create a new patch version. Then I was smashed by the question: Should I reply the V1 patch or should I create a new thread? The Kernel Newbies tutorial gives a great hint that you should send a new email, but you know, you are a newcomer and you are not sure about anything. My second approach was looking the Kernel mailling list to find some examples, for my surprise I fond both cases. At this point I was more confused than ever. Then I decided to go to kernelnewbies in the IRC and ask. Lars replied to me:

"lars> don't reply to the first one
5:28 PM people don't like that"

End of the question. Simple like that. Do not reply.

## Take Care with Labels

Quick Reply: Do not add labels for direct return and do not use large name for them

Discussion:

When I was working in a series of patch for trying to move a module out of staging, I wrote this piece of code:

```c
...
static int ad2s1210_write_raw(struct iio_dev *indio_dev,
			      struct iio_chan_spec const *chan, int val,
			      int val2, long mask)
{
...
	switch (mask) {
	case IIO_CHAN_INFO_FREQUENCY:
		switch (chan->channel) {
		case FCLKIN:
			if (clk < AD2S1210_MIN_CLKIN ||
			    clk > AD2S1210_MAX_CLKIN) {
...
				ret = -EINVAL;
				goto error_ret;
...
		case FEXCIT:
			if (clk < AD2S1210_MIN_EXCIT ||
			    clk > AD2S1210_MAX_EXCIT) {
...
				ret = -EINVAL;
				goto error_ret;
...
	}
	ret = ad2s1210_update_frequency_control_word(st);
	if (ret < 0)
		goto error_unlock_mutex;
	ret = ad2s1210_soft_reset(st);
error_unlock_mutex:
	mutex_unlock(&st->lock);
error_ret:
	return ret;
}
...
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 1: Excessive use of labels</figcaption>
</figure>

You can see the full patchset in [2]. Take a carefully look at the 'goto' and `case`. I will not try to explain the problem, because Dan Carpenter provided a great explanation that I prefer to quote him:

> "[..]
This is a do-nothing goto.  I normally consider do-nothing gotos and
do-everything gotos basically cousins but in this case it's probably
unfair since it already has other labels.
>
> Do-everything gotos are the most error prone way of doing error
handling.  I've reviewed a lot of static checker warnings and it really
is true.  I can't give you like a percent figure but do-everything error
handling is a lot buggier than normal kernel style.
>
> This style of error handling is supposed to prevent returning without
unlocking bugs.  I once looked through the git log and counted missing
unlock bugs and my conclusion was that it basically doesn't work for
preventing bugs.  The kind of people who just add random returns will do
it regardless of the error handling style.  There was even one driver
that indented locked code like this:
>
>	lock(); {
		blah blah blah;
	} unlock();
>
> When the driver was first submitted, it already had a missing unlock
bug.  I don't think style helps slow people down who are in a hurry.
>
> The other thing about do-nothing gotos is that you introduce the
possibility of "forgot to set the error code" bugs which wasn't there
before.
[..]" [3]

Really nice explanation, hum?

## Take Care with Commit Messages

Quick answer: Take care with your commit messages

Description:

In the begining of my interactions with Linux Kernel I did not care as much as I should with commit messages. After some interactions, I got a feedback from Daniel Baluta [3] to improve my messages he sent me a link with some guidlines about commit message. From his advices and the link that he provided, I want to highlight my favorite topics:

* Capitalize the first letter of commit
* The trick of "If applied, this commit will your subject line here"

There is one final advice, that a newcomer should take care related with the following comment:

> Use prefix tags to indicate the driver that is changed. Here iio:pressure:ms5611. [3]

This comment agrees with the documentation which recommends to use ":" for separating the subsystem path. However, this is not a strong rule. If you look at the Kernel mailing list you will see a vast number variations about this rule. For example, take a look at the patches send to the DRM subsystem and you will notice the use of "/" instead of ":". In this case, I recommend you to look how other developers send the patch for the specific subsystem and follows the same pattern.

## Revise and Revise your patch before send

Quick answer: Look over and over again the patch, before send.

Discussion:

Sometimes, I was in a hurry to send the patch. You know... you get excited about the idea to send patches to the Kernel. So, do not worry to send a patch as soon as possible. Seriously, take a breaf, wait a little bit, reread the patch line by line. It is not that common that people send similar patch for the same thing, it can heppen, but seriously, this is really not common. If you make as much review as possible you increases the chances that you get your patch accepted. I made this mistake sometime, one of the best example of this mistake it is this patch:

```c
-	}, {
-		.type = IIO_ROT,
-		.modified = 1,
-		.channel2 = IIO_MOD_NORTH_TRUE_TILT_COMP,
-		.info_mask_separate = BIT(IIO_CHAN_INFO_RAW),
-		.info_mask_shared_by_type = BIT(IIO_CHAN_INFO_OFFSET) |
-		BIT(IIO_CHAN_INFO_SCALE) |
-		BIT(IIO_CHAN_INFO_SAMP_FREQ) |
-		BIT(IIO_CHAN_INFO_HYSTERESIS),
-	}, {
-		.type = IIO_ROT,
-		.modified = 1,
-		.channel2 = IIO_MOD_NORTH_MAGN,
-		.info_mask_separate = BIT(IIO_CHAN_INFO_RAW),
-		.info_mask_shared_by_type = BIT(IIO_CHAN_INFO_OFFSET) |
-		BIT(IIO_CHAN_INFO_SCALE) |
-		BIT(IIO_CHAN_INFO_SAMP_FREQ) |
-		BIT(IIO_CHAN_INFO_HYSTERESIS),
-	}, {
-		.type = IIO_ROT,
-		.modified = 1,
-		.channel2 = IIO_MOD_NORTH_TRUE,
-		.info_mask_separate = BIT(IIO_CHAN_INFO_RAW),
-		.info_mask_shared_by_type = BIT(IIO_CHAN_INFO_OFFSET) |
-		BIT(IIO_CHAN_INFO_SCALE) |
-		BIT(IIO_CHAN_INFO_SAMP_FREQ) |
-		BIT(IIO_CHAN_INFO_HYSTERESIS),
-	}
+	HID_SENSOR_MAGN_3D_AXIS_CHANNEL(IIO_MAGN, IIO_MOD_X),
+	HID_SENSOR_MAGN_3D_AXIS_CHANNEL(IIO_MAGN, IIO_MOD_Y),
+	HID_SENSOR_MAGN_3D_AXIS_CHANNEL(IIO_MAGN, IIO_MOD_Z),
+	HID_SENSOR_MAGN_3D_AXIS_CHANNEL(IIO_ROT, IIO_MOD_NORTH_MAGN_TILT_COMP),
+	HID_SENSOR_MAGN_3D_AXIS_CHANNEL(IIO_ROT, IIO_MOD_NORTH_TRUE_TILT_COMP),
+	HID_SENSOR_MAGN_3D_AXIS_CHANNEL(IIO_ROT, IIO_MOD_NORTH_MAGN),
+	HID_SENSOR_MAGN_3D_AXIS_CHANNEL(IIO_ROT, IIO_MOD_NORTH_MAGN),
 };
```
<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption> Code 2: Diff of the patch with problem </figcaption>
</figure>

Take a carefully look in the above diff, you do not need an extra information to find the error. :)

Srinivas, the author of the module, find the error:

> It should be 
> HID_SENSOR_MAGN_3D_AXIS_CHANNEL(IIO_ROT, IIO_MOD_NORTH_TRUE)

Look again to the diff, the two last lines. I made this error because I was excited about the code reduction, and I want to send it fast with no need. If I wait one day, reviewed it again, probably I could noticed the problem.

## Every Time You Work in a Module Add the Author in the email

Quick answer: Always look the author name in the module and add him/her in the send patch

Discussion:

Related to the patch described above, I did not added the name of the module author in the original patch. Then Jonathan, the maintainer of the subsystem, told me:

> Whilst this looks fine to me, I'd like Srinivas to take a look before
I apply it as this is his driver.  Please do make sure to cc the author.
Whilst many such email addresses bounce as they have moved on they
don't always.

## If you Send Someone Patches, Think about the Author of the commit

Quick answer: You have to check if you really changed the original patch or not. If so, you are the author othewise keep the original author.

Discussion:

## Always Add a Cover-letter in the Patchset

Quick answer: Add a Cover-letter to all patchset you send

Discussion:

Sometime, I send some very simple patchset with the intention to make some cleanups. That time, I was considering "Do I really need to add a cover letter for this simple patchset?". Before send the patch without the cover-letter, I asked again in the kernelnewbies channel on IRC. The answer was unanimous: sending patch with cover-letter is always a good thing to do.

## Configure Your Name Correctly in the Git

Quick answer: Before send the patch, check if your name is correct the email

Discussion:

My first patches to the Kernel, have my name written as "rodrigosiqueira". I did not payed attention on this, maybe because it does not look wrong for me at first glance. Than, Dan Carpenter alert me about it [7]. I fixed it by set my name in my git globals configure.

## Checkpatch is not About Make Contribution, It is a about Learn Work Flow

## References

1. [Kernel Newbies - First Kernel Patch](https://kernelnewbies.org/FirstKernelPatch)
2. [Original Patchset that Dan Carpenter Replied](https://lkml.org/lkml/2018/3/12/766)
3. [Full Explanation of Dan Carpenter About gotos](https://lkml.org/lkml/2018/3/13/611)
4. [Daniel Baluta providing some help with commit messages](https://lkml.org/lkml/2018/2/16/349)
6. [Feedback from Srinivas](https://lkml.org/lkml/2018/3/12/669)
7. [Wrong name](https://lkml.org/lkml/2018/2/20/19)
