---
# https://chirpy.cotes.page/posts/write-a-new-post/
layout: post
title: "Free Software Development - Report #2"
date: 2026-03-18 09:50 -0300
categories: [University of São Paulo, Linux]
tags: [kw, kernel, SSH, arch-linux, arm64]
toc: true
comments: false
media_subpath: /assets/img

description: Building and booting a custom Linux kernel for ARM using kw
---

## Tutorial 2 - Building and booting a custom Linux kernel for ARM using kw
This report will follow my experience through the tutorial that can be found at [https://flusp.ime.usp.br/kernel/build-linux-for-arm-kw/](https://flusp.ime.usp.br/kernel/build-linux-for-arm-kw/).

The tutorial describes how to build/compile the Linux kernel for the **ARM Architecture** and boot test it utilizing tools provided by [kw](https://kworkflow.org/), a Kernel Workflow tool that simplifies the setup overhead of developing for Linux.

### Cloning a Linux kernel tree and configuring kw
After going through the installation process for **kw**, we move on to cloning the Linux kernel tree that we will be working with from now on. The chosen kernel tree was the *Industrial I/O* (IIO) subsystem tree, which deals with developing drivers for the Linux kernel.

The actual cloning process of the Linux tree is quite simple, you simply ```git clone``` the repository of choice from [this list of kernel repositories](https://git.kernel.org/pub/scm/linux/kernel/git/) with

``` bash
git clone git://git.kernel.org/pub/scm/linux/kernel/git/jic23/iio.git "${IIO_TREE}" --branch testing --single-branch --depth 10
```
specifying a ```--depth``` to restrict the git history size so it doesn't occupy too much disk space.

Having configured the SSH connection on the last tutorial, we are now able to configure **kw** to add a default SSH connection route to our VM.

``` bash
kw remote --add arm64 root@<VM-IP-address> --set-default
```

Now we can access it easily with ```kw ssh```:

``` bash
$ kw remote --add arm64 root@192.168.122.238 --set-default
$ kw ssh
Linux localhost ...
...
Last login: Fri Mar 13 16:39:50 2026 from 192.168.122.1
root@localhost:~#
```

## Configuring the Linux kernel compilation
The *Kernel Build System*, or ```kbuild```, allows a highly modular and customizable build process, for the Linux kernel, in a similar way to ```make```.

Since we are compiling a Linux kernel with an **ARM** architecture in an Arch Linux build, we need to specify some environment variables so the process goes smoothly. I thus recommend setting the following variables on your ```activate.sh```:

``` bash
export CROSS_COMPILE="aarch64-linux-gnu-"
export ARCH=arm64
```

- ```CROSS_COMPILE="aarch64-linux-gnu-"``` tells ```kbuild``` to use ```aarch64-linux-gnu-*``` tools when compiling the kernel, such as ```aarch64-linux-gnu-gcc```, ```aarch64-linux-gnu-ld```, etc. This is necessary because we are compiling an **ARM** kernel on an **x86-64** system, so the default ```gcc``` builds **x86-84** binaries. Make sure you have ```aarch64-linux-gnu-gcc``` installed with
``` bash
which aarch64-linux-gnu-gcc
```
or install it with
``` bash
sudo pacman -S aarch64-linux-gnu-gcc
```
This is recommended even after setting ```kw config build.cross_compile 'aarch64-linux-gnu-'```, as when I was going through the tutorial ```kbuild``` seemed to ignore it.

- ```ARCH=arm64``` tells ```kbuild``` we are building for the **ARM** architecture.

Having made these adjustments, the building process should work without any hiccups.

When configuring the ```.config``` file with ```kw build --menu```, we get to append a name after the kernel version. Since this is my first custom kernel, I shall be calling it version **alpha**.

```bash
$ kw build --info
Kernel source information
  Name: 7.0.0-rc1alpha+
  Version: 7.0.0-rc1
  Total modules to be compiled: 1402
```

After doing that and setting some other configurations, I finally executed ```kw build``` and compiled my first kernel.

### Deploying modules
Before building the kernel, we need to actually install the modules into the VM.
```
kw deploy --modules
```
With that, we can edit our ```activate.sh``` commands to utilize our built kernel image, and we can see our kernel was built correctly:

``` bash
root@localhost:~# uname --kernel-release
7.0.0-rc1alpha+
```

## Conclusion
This tutorial makes it clear how useful **kw** can be when compiling kernels, and how much it simplifies the procedure. As long as you pay attention to the architectures of the target and host machines and the adjustments that need to be made accordingly, the whole process can be completed simple with just a short list of commands.

For reference, this was what my ```activate.sh``` file looked like by the end of the second tutorial:

``` bash
#!/usr/bin/env bash

# environment variables
export LK_DEV_DIR='/home/lk_dev' # path to testing environment directory
export VM_DIR="${LK_DEV_DIR}/vm" # path to VM directory
export BOOT_DIR="${VM_DIR}/arm64_boot" # path to boot artifacts
export KW_DIR="${LK_DEV_DIR}/kw" # path to `kw` repository
export IIO_TREE="${LK_DEV_DIR}/iio" # path to IIO subsystem Linux kernel tree

export INITRD="initrd.img-6.1.0-43-arm64"
export KERNEL="vmlinuz-6.1.0-43-arm64"

export CROSS_COMPILE="aarch64-linux-gnu-"
export ARCH=arm64

# utility functions

function launch_vm_qemu() {
    qemu-system-aarch64 \
        -M virt,gic-version=3 \
        -m 2G -cpu cortex-a57 \
        -smp 2 \
        -netdev user,id=net0 -device virtio-net-device,netdev=net0 \
        -initrd "${BOOT_DIR}/$INITRD" \
        -kernel "${IIO_TREE}/arch/arm64/boot/Image" \
        -append "loglevel=8 root=/dev/vda2 rootwait" \
        -device virtio-blk-pci,drive=hd \
        -drive if=none,file="${VM_DIR}/arm64_img.qcow2",format=qcow2,id=hd \
        -nographic
}

function create_vm_virsh() {
    sudo virt-install \
      --name "arm64" \
      --memory 2048 \
      --arch aarch64 --machine virt \
      --osinfo detect=on,require=off \
      --import \
      --features acpi=off \
      --disk path="${VM_DIR}/arm64_img.qcow2",device=disk,bus=virtio \
      --boot kernel=${IIO_TREE}/arch/arm64/boot/Image,initrd=${BOOT_DIR}/${INITRD},kernel_args="loglevel=8 root=/dev/vda2 rootwait console=ttyAMA0" \
      --network bridge:virbr0 \
      --graphics none
}



# export functions so they persist in the new Bash shell session
export -f launch_vm_qemu
export -f create_vm_virsh

# prompt preamble
prompt_preamble='(LK-DEV)'

# colors
GREEN="\e[32m"
PINK="\e[35m"
BOLD="\e[1m"
RESET="\e[0m"

# launch Bash shell session w/ env vars and utility functions defined
echo -e "${GREEN}Entering shell session for Linux Kernel Dev${RESET}"
echo -e "${GREEN}To exit, type 'exit' or press Ctrl+D.${RESET}"
exec bash --rcfile <(echo "source ~/.bashrc; PS1=\"\[${BOLD}\]\[${PINK}\]${prompt_preamble}\[${RESET}\] \$PS1\"")
```
