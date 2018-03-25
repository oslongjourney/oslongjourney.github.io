---
layout: post
title:  "Things that I Learned as a Newcomer in the Linux Kernel"
date:   2017-02-26
published: true
categories: linux-kernel
---

The current post is an analysis of my mistakes and observations during my
interactions with the Linux Kernel Community. Just for "setup" reasons follows
profile at the moment of this post:

* I do not have anyone geographically close to me providing any help or
encouraging me;
* My primary source to get help is the IRC and mailing lists;
* I am a master degree student, working with Operating Systems. However, I do
  not have any support from my lab to work on Linux Kernel (actually, sometimes I
  have to hide my work);
* I can only work in my free time;
* I am a newcomer.

From December 2017 until the present day, I had sent many patches. I got many
feedbacks from the community and learned a lot from my mistakes. I tried to
summarize many of the things that I learned from sent contributions to the
Kernel. Some of the things that I discuss here are not directly documented.

Finally, in this post I do not describe how to send patches for Linux Kernel
because you can easily find tons of posts related to it; also, you can read the
excellent tutorial in the Kernel Newbies named ["Kernel Newbies - First Kernel
Patch"](https://kernelnewbies.org/FirstKernelPatch) to understand how you can
send your first patch.

## Do not Send a new Version of a Patch as a Reply to the old Patch

**Quick Answer:**
Always send the updated patches in a new threads.
{: .notice_danger}

### Discussion:

When I got my first patch reviewed, I knew that I had to create a new patch
version. Then I was tortured by the question: Should I reply the original patch
or should I create a new thread? The Kernel Newbies tutorial gives a great hint
that you should send a new email, but you know... you are a newcomer, and you
are not sure about anything. My second approach was looking the Kernel mailing
list to find some examples, to my surprise, I found both cases! At this point,
I was more confused than ever. Then I decided to go to kernelnewbies in the IRC
and ask about the issue. Fortunately, Lars gave me the final answer:

"lars> don't reply to the first one
5:28 PM people don't like that"

End of the question! Simple like that. Do not reply.

## Take Care with Labels

**Quick Answer:**
Do not add labels for direct return and do not use a large name for them.
{: .notice_danger}

### Discussion:

When I was working on a series of the patch for trying to move a module out of
staging, I wrote this piece of code:

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

You can see the full patchset in [2]. Take a careful look at the 'goto' and
`case` statements. I will not try to explain the problem because Dan Carpenter
provided a great explanation that I prefer to quote him:

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

**Quick Answer:**
Take Care of Commit Messages
{: .notice_danger}

### Description:

At the beginning of my interactions with Linux Kernel, I did not care as much
as I should with commit messages. After interactions with the community, I got
a feedback from Daniel Baluta [3] to improve my commit messages he sent me a
link to a tutorial about writing a good message. From his advice and the link
that he provided, I want to highlight my favorite rule of thumbs:

* Capitalize the first letter of commit;
* The trick of "If applied, this commit will your subject line here".

There is one final advice which a newcomer should take care, see the comment
below:

> Use prefix tags to indicate the driver that is changed. Here iio:pressure:ms5611. [3]

This comment agrees with the documentation which recommends using ":" for
separating the subsystem path. However, this is not a strong rule. If you look
at the Kernel mailing list, you will see a vast number variations about this
rule. For example, take a look at the patches sent to the DRM subsystem, and
you will notice the use of "/" instead of ":". In this case, I recommend you to
look how other developers send the patch to the specific subsystem and follows
the same pattern.

## Revise and Revise your patch Over and Over again

**Quick answer:**
Look over and over again the patch, before sending it.
{: .notice_danger}

### Discussion:

Sometimes, I was in a hurry to send the patch. You know... you get excited
about the idea to send patches to the Kernel. So, do not worry to send a patch
as soon as possible. Seriously, take a breath, wait a little bit, reread the
patch line by line. It is not that common that people send a similar patch for
the same thing, it can happen, but seriously, this is really uncommon. If you
make as much review as possible, you increase the chances that you get your
patch accepted. I made this mistake sometimes, one of the best example of this
slip it is this patch:

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

Take a careful look at the above diff; you do not need extra information to
find the error. :)

Srinivas, the author of the module, find the error:

> It should be 
> HID_SENSOR_MAGN_3D_AXIS_CHANNEL(IIO_ROT, IIO_MOD_NORTH_TRUE)

Look again at the diff, the two last lines. I made this error because I was
excited about the code reduction, and I want to send it as soon as possible.
There was no need for hurry. If I wait one day, reviewed it again, probably I
could notice the problem.

## Every Time You Work in a Module Add the Author in the Email

**Quick Answer:**
Always look the author name in the module and add him/her in the send patch
{: .notice_danger}

### Discussion:

Related to the patch described above, I did not add the module author in the
original patch. Then Jonathan, the maintainer of the subsystem, told me:

> Whilst this looks fine to me, I'd like Srinivas to take a look before
I apply it as this is his driver.  Please do make sure to cc the author.
Whilst many such email addresses bounce as they have moved on they
don't always.

## If you Send Someone Patches, You have to careful Think who is the Proper Author of the Pactch

**Quick Answer**
You have to check if you changed the original patch or not. If you make
considerable changes in the original patch, then you are the author otherwise
keep the original author.
{: .notice_danger}

### Discussion:

One day, Jonathan sent an email to the community saying that he will remove the
"meter" module from staging because the chip is "Not Recommended for New
Designs". In this period, John Syne said that he had some patches with fixes on
the module and also want to help to improve/test the module. He said that he
could send his patches to anyone interested in updating it and send the
contributions back to the upstream.

As a result, I emailed him and told that I want to help. In summary, I analyzed
his work and updated some of his patches. Before sending the patches to the
upstream, I had to face the question: "Should I send John or me as an author?".
After a long time trying to figure out, I read the following statement in the
kernelnewbies documentation:

> Make sure that the email you specify here is the same email you used to set
up sending mail. The Linux kernel developers will not accept a patch where the
"From" email differs from the "Signed-off-by" line, which is what will happen
if these two emails do not match. [1]

After read that, I decided to send the patch with me as the author and
signed-off-by John. This decision was a mistake because John was the original
author of the patch. As a result, I prepared a new version with John as the
author of the original patches.

## Add Fixes Tag

**Quick Answer**
If you fixe something, you should add Fixes tag
{: .notice_danger}

### Discussion:

When I was working on John Syne patches, I separate my patches based on his fixes. I send a first patchset to Jonathan (the maintainer of IIO), and one of his feedbacks was:

> [..] Also for these fixes ideally include a 'fixes' tag to the original commit. It makes life easy for the scripts the stable maintainers use to work out when things should apply.

Instead of going back to the documentation and carefully read about the Fixes, I went to git log and tried to find examples. However, I made the mistake of not carefully check things. As a result I sent another patcheset with the wrong way to handle Fixes. Than, Jonathan said to me:

> Please look at the Submitting patches documentation.  This is not what a fixes tag is about!  I'll fix it up this time but please look at it.

So, I made the same mistake twice. I felt really shame, but now I understand what I did wrong. Basically I had to add something like this:

> Fixes: <hash of previous commit fixes> ("message")

In my specific case, Jonathan added:

```
Fixes: 8d97a5877 ("staging: iio: meter: new driver for ADE7754 devices")
```

## Always Add a Cover-letter in the Patchset

**Quick Answer:** Add a Cover-letter to all patchset you send
{: .notice_danger}

### Discussion:

Sometime, I send some very simple patchset with the intention to make some cleanups. That time, I was considering "Do I really need to add a cover letter for this simple patchset?". Before send the patch without the cover-letter, I asked again in the kernelnewbies channel on IRC. The answer was unanimous: sending patch with cover-letter is always a good thing to do.

## Configure Your Name Correctly in the Git

**Quick Answer:** Before send the patch, check if your name is correct the email
{: .notice_danger}

### Discussion:

My first patches to the Kernel, have my name written as "rodrigosiqueira". I did not payed attention on this, maybe because it does not look wrong for me at first glance. Than, Dan Carpenter alert me about it [7]. I fixed it by set my name in my git globals configure.

[//]: <> (TODO: Pensar Sobre adicionar: Checkpatch is not About Make Contribution, It is a about Learn Work Flow)

## References

1. [Kernel Newbies - First Kernel Patch](https://kernelnewbies.org/FirstKernelPatch)
2. [Original Patchset that Dan Carpenter Replied](https://lkml.org/lkml/2018/3/12/766)
3. [Full Explanation of Dan Carpenter About gotos](https://lkml.org/lkml/2018/3/13/611)
4. [Daniel Baluta providing some help with commit messages](https://lkml.org/lkml/2018/2/16/349)
6. [Feedback from Srinivas](https://lkml.org/lkml/2018/3/12/669)
7. [Wrong name](https://lkml.org/lkml/2018/2/20/19)
8. [My mistake on Fixes tag](https://lkml.org/lkml/2018/3/24/81)
9. [Correct way to add Fixes tag](https://www.kernel.org/doc/html/v4.12/process/submitting-patches.html#describe-your-changes)
