---
# https://chirpy.cotes.page/posts/write-a-new-post/
layout: post
title: "Free Software Development Report #4 - Tutorial 4"
date: 2026-04-12 20:50 -0300
categories: [University of São Paulo, Linux]
tags: [kernel, modules, C, character devices, drivers]
toc: false
comments: false
media_subpath: /assets/img

description: Introduction to Linux kernel Character Device Drivers
---

## Tutorial 4 - Introduction to Linux kernel Character Device Drivers
This report will follow my experience through the tutorial that can be found at [https://flusp.ime.usp.br/kernel/char-drivers-intro/](https://flusp.ime.usp.br/kernel/char-drivers-intro/).

This tutorial introduces us to **character devices**, which serve the purpose of handling dynamic data streams between the user and the OS, abstracting hardware and software components.

As someone who hadn't had any experience with character devices, this tutorial in particular felt *very* content-heavy. It goes into detail on how to handle character devices, **minor and major numbers** and different commands and functions to manipulate these devices, such as `mknod`.

### Experimenting with character devices

For the experimentation section, the tutorial gives us an example of a character device driver, which we can build (using previous tutorials) and test on our VM. We then read and write from that driver using two custom functions in C.

I actually had an issue with this part, as I had forgotten to execute the
``` bash
$ mknod simple_char_node c 511 0
```
command, so the driver wasn't accessible from user space, as the special device node file hadn't been created yet.

### Conclusion

Overall, the tutorial doesn't have much to report on since it's more of an info dump, but it's still a very important introduction to some key concepts related to the IIO subsystem in the Linux Kernel.
