---
layout: post
title:  "Tutorial 2: Building and booting a custom Linux kernel for ARM using kw"
---
Guilherme Santos Gabriel

**This series of posts serve to show my experiences following the "1st Phase: Linux kernel cycle" of the Free Software Development Course made by FLOSS at USP.**

Today's tutorial showed how to build the linux kernel for the ARM architecture (like the one used in the image downloaded last tutorial) using kw.

### What i did

i start by installing kw correctly: it helps (immensely, really) us to manage the linux kernel builds that i have. i then cloned the IIO linux kernel tree that i will use in future tutorials (and in the patches later on), a lot.

i then start configuring kw inside the kernel tree and configuring it to use the architecture i want (in this case still arm64), letting cross-compilation. i generate a minimal kernel configuration and then build the linux kernel from source (which took a REALLY long time, as expected). The compiled modules i build were successfully copied to the vm we made.

### Problems

Today's only problem was remembering to download the ssh packages inside the vm using *apt install openssh-server*.