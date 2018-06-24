---
layout: post
title:  "Status Report"
date:   2018-06-1
published: true
categories: linux-kernel
---

This is simple blog post reporting my work status in the GSoC.

## Brief about the task

During the interaction with my mentors and the community, two tasks were
raised:

1. Vblank and page flip events
2. CRC support

I get the first task, and Haneen the second one.

In my task I have to handle the Vblank and Page Flip considering two cases:

1. Real hardware, with very regular vblank events simulated through hrtimers.
2. Virtual hardware, which doesn't support the vblank interrupt and completes
page_flip events right away.

Additionally, it is required to double check what compositors (like weston, or
xfree86-video-modesetting) are using to decide between these two, that's a bit
ill-defined.

## Development Status

For the development of this task, I am focusing on making the kms_flip pass
(IGT project). My development cycle resembles a Test-driven development (TDD),
because I execute kms_flip over VKMS, look for failures, and fix/implement the
required feature. As a result, new features were implemented in the VKMS.

I submitted a patchset [1] with all the new features, and I’m waiting for the
review. Also, I make the subtest basic-plain-flip from kms_flip completely pass
(I expected to send the patch on Monday - June 25).

Finally, I also interacted with IGT community and got my first two patches
accepted in the project (commit 6be300d405 and afff8ad2f). I also proposed a
new feature to force a specific module to be used by the IGT, I got some
excellent feedbacks and already start to work in this feature.

## References

1. [Patchset](https://lists.freedesktop.org/archives/dri-devel/2018-June/180823.html)
