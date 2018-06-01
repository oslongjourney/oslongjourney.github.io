---
layout: post
title:  "Start to work on DRM"
date:   2018-06-1
published: true
categories: linux-kernel
---

In this post, I describe a little about me, GSoC and VKMS. I want to share a
little about my experience with GSoC and DRM community, an overview of the VKMS
project, report project progress, and provide a roadmap for future posts.

## About me, GSoC and VKMS

When I was in the graduate school, I became aware of Operating Systems (OS)
field for the first time, since then, my most wanted professional desire is:
work with OS from theoretical to the practical level. Nonetheless, entering in
this field is not an easy task when you are not in a favorable environment, but
you know... I could not give up on something that I'm passionate about. To
solve this problem, I tried to improve my knowledge in OS by starting a
master's degree. On the one hand, my master provides some understanding of the
OS state-of-the-art; on the other side, it not offered me practical skills in a
real-world OS. I tried to start to contribute to Linux Kernel to fulfill my
urge to work in a production-oriented OS, but it was hard to start it alone.
However, the conditions begin to change for me at the end of 2017 due to a
conference named "Linux Developer Conference Brazil," (linuxdev-br)[1] in this
event I had the fantastic opportunity to talk with many Brazilians kernel
developer. I also heard about the experience of people that started to work
with Linux Kernel due to GSoC. Finally, I had the opportunity to meet Gustavo
Padovan in the conference, I got his contact and interacted with him with
questions related to DRM tasks.

Since linuxdev-br I started to put as much effort as I could to become a Kernel
developer. During this processes, I often mailed Gustavo to get some help and
some ideas on stuff that I could do on DRM subsystem. He recommended me to look
at a TODO list on the site, and there I heard for the first time about Virtual
KMS (VKMS). In summary, from this idea in the TODO list started my journey with
DRM in GSoC.

## About VKMS

The Kernel Mode-Setting (KMS) is a mechanism that enables a process to command
the kernel to set a mode (screen resolution, color depth, and rate) which is in
a range of values supported by graphics cards and the display screen [2].
Create a Virtual KMS (VKMS) bring some benefits, for developers:

* It could be used for testing;
* It could be used for testing; second, it can be valuable for running X or
  Wayland on a headless machine enabling the use of GPU.
* It can allow to fake KMS feature that real hardware might not have;
* It can provide tools for Window Manager developers test their code.

In this regard, VKMS have to fake outputs and provide just a basic set of KMS
elements such as a single CRTC, encoder, connector, and plane (all up and
running). Finally, this module has a potential to evaluate to receive many more
features in the future.

## About people involved in VKMS project

Due to the importance of VKMS project, some people are directly engaged in the
project. First, there are two interns from a different programs working
together which is Haneen (Outreachy) and I (GSoC). Officially Sean Paul mentor
Haneen, and Gustavo Padovan mentor me. Additionally, Daniel Vetter provided a
lot of support and work as a mentor for both of us. Additionally, many
different people from the community helped us with valuable information.

## About VKMS progress

For the first task, Daniel suggested that we create an elementary DRM module
which could be used by Haneen and me during the project. The requirements
provided by Daniel for this module was:

1. Add the necessary elements in the Linux-tree for enabling and compiling VKMS;
2. create a simple initialization of the module (KMS initialization, register,
   and basic data structures);
3. Add one CRTC;
4. Add one encoder;
5. Add one connector;
6. Add one plane.

These tasks were really challenging for me due to my lack of experience and
knowledge. As a result, I put a lot effort to try to understand the details in
the DRM subsystem; for me, the task was hard but really enjoyable since mentors
and the community helped me with my questions. Additionally, work with Haneen
is fantastic since she always helps me to understand some of my mistakes and
also provided me some great feedbacks. After three weeks of interactions, we
create the basic structure of VKMS which can be checked in the "drm-misc" on
branch "topic/vkms" [3].

Now the work was split into two parts: add a fake page flip simulation ( real
and virtual hardware), and CRC. I am working on the first task and Haneen on
the second one. With these two features, VKMS will become valuable and ready
for expansion regarding features. I believe these two tasks will take much more
time than the first task, and it will be much more exciting.

## About IGT

We wish to make VKMS pass on various of the IGT tests. As a result, we had to
learn how to use the IGT and how it works. For example, we had to change the
target module in the ``__open_driver`` function to use VKMS. From my studies in
the IGT source code, I noticed a lot of things that I could work for helping to
improve the project. I recently decided to try to upstream some of my
contributions, some of them are:

1. I sent an RFC patch with a new feature for forcing the load a specific
  module indicated via command line. Status: waiting for comments [4];
2. A patchset fixing some annoying GCC warnings. Status: waiting for review [5].

I have other small things on my local branches to try to upstream, but I
decided to wait for a little to get feedback in the previous patches.

## Roadmap on the next posts

Since the beginning of the GSoC, I put a lot of effort to dive into the DRM.  I
faced some challenges related to basic stuff, such as understand the basic
concepts and jargon, to technical problems associated with starting a new
module from scratch. Inspired by all the obstacle that Haneen and I faced, I
believe that could be worth for newcomer a series of posts based on VKMS. From
my current experience, I want to write the following posts:

1. **Establishing the necessary knowledge and jargon**. In this post, I want to
handle all the basic concepts and jargon that can be useful to know before
starting to work. I think this post is essential for newcomers to take more
advantage of the DRM documentation and also understand the vocabulary used by
DRM developers (sometimes someone said to me a specific jargon that I did not
know in the dri-devel);
2. **Kworkflow: Create a safe and automate environment to work**. Work with
Kernel on your host machine can cause severe problems to your system, I
discover it by myself during the work with VKMS. After a talk with Padovan, he
provided me a set of scripts to automate the task. From his scripts, I created
a new project named kworkflow [6] that manage the workflow. This sort of
information can save time and problems for newcomers.
3. **Create a simple DRM module: from module to user space**. In this post, I
want to describe how to create a simple DRM module structure based on my
experience on VKMS. I also want to show some very basic user space program that
uses libdrm to interact with the module;
4. **I-G-T: from testing to the source code**. In this post I will introduce
the IGT concepts, code, and how to use it;
5. **VKMS page flip**. In this post, I will describe how I implemented the KMS
flip in VKMS.

## References

1. [linuxdev-br](https://linuxdev-br.net/)
2. [Direct Rendering Manage](https://en.wikipedia.org/wiki/Direct_Rendering_Manager)
3. [VKMS branch](https://cgit.freedesktop.org/drm/drm-misc/?h=topic%2Fvkms)
4. [RFC for force module](https://lists.freedesktop.org/archives/igt-dev/2018-May/003308.html)
5. [GCC warning fixes](https://lists.freedesktop.org/archives/igt-dev/2018-May/003309.html)
6. [Kworkflow](https://github.com/rodrigosiqueira/kworkflow)
