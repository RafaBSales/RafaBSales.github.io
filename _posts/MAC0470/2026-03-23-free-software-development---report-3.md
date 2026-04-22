---
# https://chirpy.cotes.page/posts/write-a-new-post/
layout: post
title: "Free Software Development Report #3 - Tutorial 3"
date: 2026-03-23 15:03 -0300
categories: [University of São Paulo, Linux]
tags: [kernel, modules, C]
toc: true
comments: false
media_subpath: /assets/img

description: Introduction to Linux kernel build configuration and modules
---

## Tutorial 3 - Introduction to Linux kernel build configuration and modules
This report will follow my experience through the tutorial that can be found at [https://flusp.ime.usp.br/kernel/modules-intro/](https://flusp.ime.usp.br/kernel/modules-intro/).

The tutorial goes over building and testing a simple kernel module, enabling new features through *menuconfig* and creating your own build configurations.

### Creating a simple kernel module
This tutorial itself is pretty straightforward, not much to comment here, so I will just summarize the module building/configuring process, as it will also serve as reference for me in the future:

- The first step is to create the actual source code, which in the Linux kernel case, it's **mostly written in C**. In our example case, we created a function that will output messages to the kernel ring buffer on initialization and on exit.
To do so, we used the following functions:
``` c
module_init(simple_mod_init);
module_exit(simple_mod_exit);
```
that guarantee they will run on initialization and on exit, and
``` c
pr_info("Hello world!\n");
```
to output to the kernel ring buffer. We also call ```MODULE_LICENSE("GPL");``` at the bottom, to abide by the Linux kernel license.\\
All of the above are provided by the ```<linux/module.h>``` library.\\
The functions also used the ```__init``` and ```__exit``` annotations/macros provided by the ```<linux/init.h>``` library, which optimize the memory usage on compilation.

- Next we create a Kconfig configuration symbol. We want to do this in the Kconfig in the same directory our module is located. Ours looks like this:
```
config SIMPLE_MOD
	tristate "Simple example Linux kernel module"
	default	n
	help
	  This option enables a simple module that says hello upon load and
	  bye on unloading.
```
We use **tristate** to define that the symbol may be compiled as a *module*, not necessarily built into the kernel image (basically, it adds the option *m* for module, instead of the standard *y* for built-in compiled or *n* for not compiled).\\
We must also add the module to the list of modules in the Makefile of the same directory, like we did here:
```
obj-$(CONFIG_SIMPLE_MOD)		+= simple_mod.o
```
- After that, we need to enable the module on *menuconfig* with ```make menuconfig```.
- Once that's done, we may clean the build system with ```kw build --clean``` then build it again with ```kw build```.
- Now we install the kernel modules into the kernel image (make sure the VM is shut off at this point):
``` bash
mkdir "${VM_DIR}/arm64_rootfs"
### ADAPT THE COMMAND BELOW ###
sudo guestmount --rw --add "${VM_DIR}/arm64_img.qcow2" --mount /dev/<rootfs> "${VM_DIR}/arm64_rootfs"
sudo --preserve-env make -C "${IIO_TREE}" INSTALL_MOD_PATH="${VM_DIR}/arm64_rootfs" modules_install
sudo guestunmount "${VM_DIR}/arm64_rootfs"
```
- With that done, we can start the VM and the modules should be installed.

### Testing the modules
I was already familiar with commands like ```modinfo``` and ```modprobe```, as I had already meddled with the [Samsung Galaxy Book driver](https://docs.kernel.org/admin-guide/laptops/samsung-galaxybook.html) in the past. And with that, I tested the module as instructed:
``` bash
modprobe simple_mod # loads module of name `simple_mod` 
dmesg | tail
modprobe -r simple_mod # unloads module of name `simple_mod` 
dmesg | tail
```

The tutorial also shows how you can import functions from other modules with a very similar process, by calling some new functions like
``` c
EXPORT_SYMBOL_NS_GPL();
```
and
``` c
MODULE_IMPORT_NS();
```

On Kconfig, you must use ```depends on MODULE_NAME``` instead of ```tristate```.

## Conclusion
So far in this series we had only been laying out the groundwork for building kernel modules, so it was exciting to actually build one from scratch in this tutorial. Overall I didn't have much trouble following through this tutorial, I simply followed the steps and everything seemed to work as intended, as all the Arch-Linux related problems had already been fixed in previous tutorials.
