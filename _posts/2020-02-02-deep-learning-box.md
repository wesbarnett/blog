---
title: "Building a deep learning box"
description: "Describing how I put together a decent box for less than $500"
layout: post
toc: false
comments: true
image: images/deep-learning-box/completed.jpg
hide: false
search_exclude: false
categories: [deeplearning]
---

During the process of going through the fast.ai course, I decided to build my own deep
learning box. This post is summary of my experience, but it won't tell you every detail
needed to build your own. There are quite a few guides out there.

A couple of decades ago as a teenager I remember my brother ordering parts
and putting together the family's PC. Of course at the time it was used mainly for
playing Alpha Centauri, downloading from Napster, and racking up the long distance phone
bill not knowing area codes did not mean local calling. I also got exposed to HTML and
Javascript at the time so that was my first real introduction to any kind of coding
(Javascript has changed just a little since then).

I remember my brother using a website PC Parts Picker to put a list of compatible parts
together and find where to buy them. That website is still commonly used today and what
I used in picking my parts. [This is the list I ended up
with](https://pcpartpicker.com/list/XJBpmg). (Note: Some items are now discontinued by
the time you're reading this). The important thing for me was to get an Nvidia card to
be able to properly use CUDA on my machine. The card is on the lower end of what is
considered acceptable, but I was on a budget.

Here are most of the parts in their boxes before assembling.

![]({{ site.baseurl }}/images/deep-learning-box/parts.jpg "The parts!")

The process was pretty straightforward. Motherboard went in first, then the processor,
and so on. I did have an issue where the backplate came off when I unscrewed the mounting screws for
the processor fan. I struggled to put the fan on due that until I realized it had
fallen behind the case. I wish I had mounted the fan the other way since there is a
plastic part that now covers one of the RAM ports.

![]({{ site.baseurl }}/images/deep-learning-box/fan-installed.jpg "Motherboard,
processor, and fan installed. Should have turned the fan around.")

Most of the cords can only plug into one spot and everything on the motherboard and
power supply are labeled. Just paint by numbers for the most part. I did make the
mistake of not plugging the GPU into the power but got a nice message when I tried to
boot up.

![]({{ site.baseurl }}/images/deep-learning-box/gpu-power.jpg "Nice message telling me
of my mistake.")


![]({{ site.baseurl }}/images/deep-learning-box/completed.jpg "All the parts assembled.")

After I was able to boot I installed [Arch Linux](https://www.archlinux.org) on it.
I have been involved with the Arch community for quite some time in editing the Wiki and
have installed it numerous times, so I was comfortable doing so (Here's [my own
installation guide; use with
caution](https://wiki.archlinux.org/index.php/User:Rdeckard/Installation_guide).
Additionally all the super computer clusters I used in graduate school were Linux-based,
and our lab setup our own cluster. One downside about using Arch is that sometimes there
are libraries that are made for Ubuntu that aren't as easily compiled for Arch in a
straightforward manner.

![]({{ site.baseurl }}/images/deep-learning-box/arch-boot.jpg "It's working!! Arch
Linux boot screen.")

After installing Arch I installed JupyterLab and JupyterHub, CUDA, and some of the deep
learning frameworks and was able to start running pretty quickly.

![]({{ site.baseurl }}/images/deep-learning-box/fastai.PNG "Running Jupyter on the
machine remotely.")

Thus far, it's been a good experience using the machine for running a couple of Kaggle
competitions.
