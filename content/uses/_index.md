+++
title = "Uses"
description = "A page listing my preferred tools..."
template = "prose.html"
insert_anchor_links = "none"

[extra]
lang = 'en'
math = false
mermaid = false
copy = false
comment = false
+++

# What I Use

This page is inspired by the one found [here](https://www.paritybit.ca/uses/).

## General

Over the last year or so, my preferences have drifted towards stability
and minimal configuration. While I make using FOSS whenever and wherever
possible a priority, I also don't want to spend a ton of time
reconfiguring my operating system or tweaking plugins for a favourite app
to get that extra bit of performance and customization. There's probably
stuff I'm missing out on, but it ends up being so much more to maintain.
That means that in some cases my favourites are actually older than I
am!

## Operating System

[Debian](https://www.debian.org/) 12 (Bookworm), everywhere. I recently made the
switch from Fedora, which I was a longtime user of (after having started my
Linux journey on Ubuntu). There were a lot of reasons that slowly built towards
this shift (see [General](#general), above), but the big two were:

1. The increasing frequency of issues trying to build the latest changes
  to Yocto's core layers with the hot-off-the-presses tooling in Fedora;
2. Better default toolchain access for cross-compiling, e.g. to
  arm/arm64.

I know that there are lots of ways to work around these issues
(containers, VMs, other machines, etc.), but following the
do-less-plumbing principle means using the OS that reduces the grind
time. It turns out that I also like what I see from Debian's design
choices a bit more, especially its preference for not allowing global
installs of Python modules with pip, recommending a virtualenv (or pipx)
instead.

## Shell

[Bash](https://www.gnu.org/software/bash/). Default, default, default.

## Text Editor

[Vim](https://www.vim.org/), specifically with the most barebones
configuration that I can stand. As of now, I use the following plugins with
[pathogen](https://github.com/tpope/vim-pathogen):

- [Airline](https://github.com/vim-airline/vim-airline)
- [Fugitive](https://github.com/tpope/vim-fugitive)
- [gitgutter](https://github.com/airblade/vim-gitgutter)

Here's my entire `.vimrc`:

	execute pathogen#infect()
	syntax on
	colorscheme desert
	" Use filetype detection and file-based automatic indenting
	if has('filetype')
	    filetype plugin indent on
	endif

	set pastetoggle=<F2> " add hotkey for changing to paste mode to avoid extra indentation
	set updatetime=100 " reduce time between updates from 4000 to 100
	set textwidth=80 " set wrap width to 80
	set laststatus=2 " start airline immediately
	set autoindent " automatically indent
	set visualbell " don't beep
	set mouse=

	if has("autocmd")
		" Use actual tab chars in Makefiles
		autocmd FileType make set tabstop=8 shiftwidth=8 softtabstop=0 noexpandtab
		autocmd FileType python set expandtab shiftwidth=4 softtabstop=4
		autocmd FileType rust set expandtab shiftwidth=4 softtabstop=4
		autocmd FileType yaml setlocal ts=2 sts=2 sw=2 expandtab
		autocmd BufNewFile,BufRead *.v,*.vs set ts=4 sts=4 sw=4 expandtab
	endif

	let g:airline_powerline_fonts = 1

I've had many discussions over the years about text editor preferences, and once
had a brief stint using [neovim](https://neovim.io/) (which has some cool
distros), but using what I do for a text editor gives me that extra peace of
mind that I'm not using too many extras that might break unexpectedly. Also, it
seems to serve me well given the frequency with which I find myself logging into
servers and custom-built embedded systems that might not have anything but the
stock version (or even just the [busybox](https://www.busybox.net/) version).

### Web Browser

[Firefox](https://www.mozilla.org/en-CA/firefox/) with [UBlock Origin](https://ublockorigin.com/).

### Mail Client

[Thunderbird](https://www.thunderbird.net/en-US/).

### VPN

I've been using [Tailscale](https://tailscale.com/) for a while now. I'm
extremely impressed with how easy it is to setup and make use of for home
networks.

### Continuous Integration

My [first conference talk](https://www.youtube.com/watch?v=luxMUcOB_JM) was a
virtual one about using [Tekton CI](https://tekton.dev/) with a single-node
Kubernetes cluster, but I've long-since soured on k8s. I've used others in the
past (e.g. Jenkins, Buildbot), but I like using [Laminar
CI](https://laminar.ohwg.net/) now because it's easy to setup and is basically a
static frontend for bash scripts and cronjobs.

## Hardware

### Laptop

My daily driver is a Lenovo ThinkPad T14s with 16GB of DDR4 RAM and a
Ryzen 5650U in it. It was acquired used from eBay, and I'm quite happy
with it. I've recently added a 1TB CT1000P3PSSD8 drive, replacing the
256GB NVMe drive it came with. It feeds two LG 27" 1440p monitors
through a Lenovo USB-C dock. I wouldn't say it's a powerful workhorse by
any means, but it gets the job done.

### Build Server

I offload most of my heavier tasks (Vivado, Yocto and kernel builds,
nightly CI runs, etc.) to a custom mini-ITX build with the following
specs:

- **CPU:** AMD Ryzen 9 7900 (**not** the 7900X)
- **Cooler:** Wraith Prism RGB (the stock 7900 one)
- **Motherboard:** MSI MPG B650I Edge WiFi Gaming Motherboard
- **RAM:** Corsair Vengeance 64GB DDR5 6000MHz RAM
- **GPU:** Radeon RX 6600 8GB
- **Main Storage:** Crucial P3 Plus PCIe Gen4 2TB NVMe M.2 SSD
- **Secondary Storage:** Western Digital SA510 2TB 2.5" SSD (volatile build area)
- **Tertiary Storage:** Western Digital Blue 3D NAND 1TB 2.5" SSD (mostly for Yocto sstate-cache and downloads)
- **Case:** Thermaltake Core V1 Black Edition SPCC Mini ITX Cube Computer Chassis
- **PSU:** Corsair RM850e (2023) Fully Modular Low-Noise ATX Power Supply
- **Monitor:** LG Ultragear 27GL83A-B (x2)

The goal with this build (completed summer 2024) was as much
cost-effectiveness and power consumption as it was performance. None of
the parts were the most jaw-dropping options even when I started pricing
it out, but it gets the job done and is surprisingly quiet, while taking
up very little space. I used to buy off-lease Dell, HP, and Lenovo
workstations to use as my build servers, but those only get
hobbyist-cheap after they've been around for 6+ years, and at that point
they seem a lot less efficient than just building something new like
this. It also handles what little gaming I still do these days, thanks
to that lower-mid-range RX 6600 GPU.

### Peripherals

- **Keyboard:** [Logitech MX Mechanical Mini](https://www.logitech.com/en-ca/shop/p/mx-mechanical-mini)
- **Mouse:** [Logitech G G502 Hero](https://www.logitechg.com/en-ca/products/gaming-mice/g502-hero-gaming-mouse.910-005469.html)
- **Headphones:** [Logitech G435](https://www.logitechg.com/en-ca/products/gaming-audio/g435-wireless-bluetooth-gaming-headset.html)

You can probably sense a favourite brand. All of these are swapped between my
laptop and server (when I'm not just using SSH) via an [Aimos KVM
Switch](https://www.amazon.ca/Selector-AIMOS-Switcher-Computers-One-Button/dp/B085915CTB).
### Phone

Whatever's relatively cheap and comes with an acceptable level of bloat.
This pretty much always means Android. Right now it's a **Samsung Galaxy
A54 5G** I got off of Amazon.

### Test Equipment

- **Oscilloscope:** [Rigol DS1054Z](https://www.rigolna.com/products/digital-oscilloscopes/1000z/)
- **Function/Waveform Generator:** [Sigilent SDG 1032X](https://siglentna.com/product/sdg1032x/)
- **Logic Analyzer:** [Saleae Logic 8](https://www.saleae.com/products/saleae-logic-8)
- **Multimeter:** [Fluke 117](https://www.fluke.com/en-ca/product/electrical-testing/digital-multimeters/fluke-117)

I've got a variety of random electronics parts for building stuff (like most
hobbyists, I think) that sit in drawers for long periods of time before being
pulled out to test something or tinker. The DS1054 and SDG 1032X, followed by
the Logic 8, see by far the most action as I find myself working on IIO drivers
for the Linux kernel. I do have a Weller soldering iron and a 300W test bench
power supply, but since most dev kits seem to run off of USB power or 5V/9V/12V
adapters these days, those tend to sit in a box rather than taking up desk
space.

### Homelab

Various Raspberry Pi 4s and ThinkCentre Tinys with AMD processors in them. The
network is held together by a few Netgear switches like the
[GS308E](https://www.netgear.com/ca-en/business/wired/switches/plus/gs308e/). If
what I'm doing can't be accomplished between my build server and these pieces,
then it's probably an awaylab :).
