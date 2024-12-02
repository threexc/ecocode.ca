+++
title = "My Journey Into Open Source"
date = 2024-11-03
+++

## The TL;DR

One part schismogenesis, one part STEM education, and many parts
patience and good luck in the job market.

## Beginnings

I'm just barely old enough that being exposed to early versions of Linux
operating systems could've been possible, but that's not how it
happened. While my first computer was a second-hand Commodore 64, and
the second (a year or two later) ran Windows 98 SE with a command
prompt, I didn't even take advantage of access to the available tools
except to try and play video games. It wasn't actually until I started
experimenting with more efficient messaging apps (and loading my systems
with plenty of viruses in the process) around 2009 that I learned it was
possible to run an operating system other than Windows on the computer
hardware I had.  During that year in university, I downloaded what at
the time was a recent Ubuntu release (based on the release history, I'm
guessing Ubuntu 8.04.2) and installed it on a spare laptop. However, I
didn't have access to internet other than via a dial-up modem at home -
perks of growing up in rural New Brunswick - and my install didn't seem
to include a driver for that, so (not knowing what else to do) I let it
sit for the summer.

## Back to Class

The following term (fall 2010) I was registered in a full slate of
physics, mathematics, and computer science courses, and I became
convinced that using Linux and various open-source alternatives to
popular tools (e.g. GNU Octave instead of MATLAB or Maple) set me apart,
in a way that was a little elitist. I wasn't even considering
open-source contributions yet, but I was breaking my operating system
installations (or at least putting them in strange states with no
backups) often enough to learn the basics of system administration and
what **not** to do. Combine that with Git basics and slowly figuring out
how to compile some things from source, it'd put me in better shape to
start doing these things full-time in a few years.

## Professional Experience and Personal Projects

After a short, unpleasant experience as a junior developer at a drone
startup in Halifax, I ended up at a defense contractor in the Ottawa
area, where I picked up a bit more practice and a few new system admin
skills before becoming one of the go-to people in the organization for
system integration, probably because of my comfort with Linux systems
(even if they were old releases with limited support). This wasn't a
great experience for me, so after a couple of years there I was actively
looking for a role elsewhere that might mean more open-source work with
different technology. I was getting pretty good at Googling topics
effectively here, though.

During this time I was starting some of my first personal projects with
Python - in particular some weather modeling using METAR and TAF data
from Canadian airports. I also continued insisting on using Python and C
programs written on whatever flavor of Ubuntu or Fedora I happened to be
using at the time to meet the needs of the masters program I had
started, rather than the recommended tooling available in university
computer labs.

## My First Yocto Patch

The first real opportunity I got to contribute professionally to open
source was in 2019, nearly ten years after my first foray into personal
use of the operating system. I got hired at a large tech company to work
on their Yocto-based operating system, which meant submitting most of my
changes (usually recipe upgrades and CVE fixes) upstream first. I'm
pretty sure the very first Yocto patch I ever wrote was [this
one](https://git.openembedded.org/openembedded-core/commit/?id=c2559ab9b41b823b23dc675745bbaefd45362a08),
updating ptest dependencies for gzip.

I was hooked at this point. It felt good to see my contributions not
only be displayed on a public repository alongside many other highly-skilled
and well-known developers, but to know that my changes (however niche)
were making their way into custom Linux systems of all sorts. It wasn't
long before I was collaborating with others in the community to make
bigger changes, including taking on a bigger role as maintainer of the
meta-python layer and [presenting the
system](https://www.youtube.com/watch?v=luxMUcOB_JM) I used to do so at
a virtual conference during the COVID days.

## And Then...

Things got rockier over the next few years as I made a poor decision in
switching to another role for more money, but lost opportunities to work
on projects I found interesting and ended up resigning from my
maintainer roles not long after. I almost immediately sought to find my
way back, but it probably comes as no surprise that it took time to find
the right one. During all this, though, I wrote [the
code](https://github.com/threexc/routesignal) for my masters' thesis in
Python and published it on GitHub. Someday I'll hopefully go back and
reimagine it.

As of now I'm back contributing regularly to the Yocto Project, as well
as finally writing my first kernel drivers, which has been a huge
personal goal of mine for a long time. I'm sure my involvement in all
things Linux and embedded systems will only grow from here.

## The Point?

This post is a drawn-out way for me to say the following things to
others who might be struggling to find their way into open-source and/or
a tech career that feels right:

1. You don't need to be doing this from a very young age to succeed (I
   didn't seriously start programming or playing with electronics until
   my 20s);
2. It might take time - it was about a decade from the time I first
   started experimenting with Linux until I got my first job where I
   could get paid to contribute, and the job market seems to be
   challenging even when you have a desirable set of skills on your
   resume;
1. What you learn will serve you well, even if it isn't the latest fad
   technology. Compared to some of my peers, my career has been very
   short, but even in that time I've seen a lot of tools and paradigms
   come and go. Knowing how to debug and rule things out will always be
   useful;
1. Doing this kind of work is **hard**, even when you're an expert - try
   not to be too harsh on yourself, because a) failure is often the best
   way to learn, and b) even the experts will make mistakes, struggle, or
   forget things they used to be good at.
