---
layout: post
title:  "Tutorial 7: IIO Dummy module Experiment One: Play with iio_dummy"
---
Guilherme Santos Gabriel

**This series of posts serve to show my experiences following the "1st Phase: Linux kernel cycle" of the Free Software Development Course made by FLOSS at USP.**


### What i did

This tutorial is complementary to the last one. Now, instead of reading the iio_dummy we will instead use it as a template to making a brand new driver! Following the tutorial i ended up with a compass that has reads in 3 axis. To achieve this i needed to modify the simple_dummy header, channels (so we have three different iio_chan_spec, one per axis) dedicated to IIO_MAGN reads and update the iio_dummy_read_raw so it reads the IIO_MAGN type. 

### Problems

Like the tutorial before, it was kind of hard to follow what we were doing in the steps, but that was the extent of the "problems" i had.  