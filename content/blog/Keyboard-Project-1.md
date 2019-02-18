+++
date = "2019-01-16T18:00:00+13:00"
title = "Handwired Keyboard Project: Part 1"
description = "What happens when you mix software debugging processes and hardware..."
tags = ["Linux","Kernel","C","Planck","Keyboards","ARM"]
topics = ["Blog", "Hardware"]
comments=true
+++

## Idea
  So I've been typing on an ortholinear mechanical keyboard called the [Planck](https://olkb.com/planck) for roughly 8 months, and have been thoroughly enjoying it. But I felt as though it could benefit from being a standalone linux board, offering wireless / bluetooth, HDMI, oh and a full linux distribution.

  I've worked with the Odroid u3's in the past, but only to set up a mpich cluster, beowolf or docker swarm cluster in university, but I never really got beyond the software side of things. This project will touch on kernel modules, hardware interrupts, input matrices (for allowing n-key rollover and prevent ghosting). 

## The SBC

![NanoPi M4](http://wiki.friendlyarm.com/wiki/index.php/File:NanoPi_M4-01B.jpg)

I've chosen the [NanoPi M4](http://wiki.friendlyarm.com/wiki/index.php/NanoPi_M4) for this task, which is certainly overkill, but I'd like to do some heavier tasks on the board, play 4K out, headphones out, and have potentially an SSD installed. 

![Planck Hi-pro case](https://static1.squarespace.com/static/5701bc562eeb810fd9247c88/5701c35bf699bbaade59f4b1/5a329522085229c7f4c9adfc/1525238746595/PLKBOT-HIPRO-BLKMAT-rside.png?format=1500w)

This is all well and good, but wait this thing has to sit inside a planck hi-pro case, whith a layer of cherry mx switches sitting above it? Aha, yes... Well we can do without the usb ports and the ethernet port, this brings us down to the pin height on the GPIO as the tallest peice on the board - but wait, how do I get those giant things off the PCB?

I tried removing them with a soldering iron, even with some desoldering braid these things were stubborn, and they were not coming out easily. So with a defeated feeling, I took some side cutters to the board...

![NanoPi M4: One port down](https://imgur.com/1vMmkmn.jpg)
One port down!

![NanoPi M4: Low profile](https://imgur.com/Q093I99.jpg)

Oh wait, what that worked? Yes.

Well it did for a week or so, but I ended up shorting some of the gpio pins while writing the next section. The NanoPi will no longer boot or show any power leds... But makes for a good hand warmer.
I ordered another one, and proceeded forward!

## Linux Distro

So there are quite possibly a million different linux distributions out there, for testing I'll be using Armbian which comes with the kernel and uboot firmware for the rk3399 board. I may later try and install somehing crazy, like Gentoo...

More on the Armbian M4 image [here](https://www.armbian.com/nanopi-m4/)

## That whole keyboard thing

Great now that we can fit the board in the case, we need to sort out how we want to interface with the keyboard switches. We could just buy another arduino like input board or a teensy and run it off the linux board, but wheres the fun in that? Plus we can save on some space to fit more battery later!

Right, so the Planck has a 12 column, 4 row design (a so called 40% keyboard). For this, we will need an input for each column and row, making a total of 16 required GPIO inputs. To interface with the GPIO I've opted to make a linux kernel module; you could alternatively write something in python with wiringpi or something in high level userspace, or go the other way with embedding into the kernel... But I want something portable without the entire board falling over, and make use of interrupts - so kernel module it is!

### The Kernel Module

I heavily relied on some articles I found by [Derek Molloy](http://derekmolloy.ie/kernel-gpio-programming-buttons-and-leds/) which helped me learn how things work.

More on this implementation in Part 2! (Coming soon)
