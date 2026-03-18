---
# https://chirpy.cotes.page/posts/write-a-new-post/
layout: post
title: "Free Software Development - Report #1"
date: 2026-03-16 16:09 -0300
categories: [University of São Paulo, Linux]
tags: [QEMU, libvirt, SSH, arch-linux, arm64]     # TAG names should always be lowercase
toc: true
comments: false
media_subpath: /assets/img

description: Setting up a test environment for Linux Kernel Dev using QEMU and libvirt
---

## (Prologue) Free Software Development
[MAC0470 - Desenvolvimento em Software Livre](https://uspdigital.usp.br/jupiterweb/obterDisciplina?nomdis=&sgldis=mac0470), or "Free Software Development", is a course offered at the University of São Paulo that aims to expose the students to the FOSS development community and its tools, the advantages/disadvantages of its use, its *modus operandi* and the hardships one might go through in the process of adopting its philosophy.

This series of reports will follow my journey throughout my time on this course, the choices I made, hardships I overcame and what I've learned through it all.

It is my hope that - by the end of this experience - I will have gotten a deep and fundamental understanding of the inner workings of the FOSS development community, and what it takes to contribute to its growth.

The projects and experiments described in this series were executed in my personal laptop running an [Arch Linux](https://archlinux.org/) build.

## Tutorial 1 - Setting up a test environment for Linux Kernel Dev using QEMU and libvirt
This report will follow my experience through the tutorial that can be found at [https://flusp.ime.usp.br/kernel/qemu-libvirt-setup/](https://flusp.ime.usp.br/kernel/qemu-libvirt-setup/).

The tutorial goes through setting up a safe environment for Linux Kernel Development by creating an isolated virtual machine utilizing [QEMU](https://www.qemu.org/) and [libvirt](https://libvirt.org/) to manage it.

### Setting up and configuring the VM
After finishing up the test environment directory setup, the tutorial instructs you to download [this](https://cdimage.debian.org/cdimage/cloud/bookworm/daily/20250217-2026/debian-12-nocloud-arm64-daily-20250217-2026.qcow2) debian OS disk image, which was unfortunately removed at some point in time. It is therefore recommended to simply download the [latest debian image](https://cdimage.debian.org/cdimage/cloud/bookworm/daily/) currently available in the official debian cloud images archive.

The image we will be working with is built for the **ARM 64-bit architecture**. As Arch Linux currently only officially supports the **x86-64** architecture, some adjustments will need to be made in the future to properly adapt the test environment to my system. To start with, however, it is necessary to install the ```qemu-system-aarch64``` package, as it emulates **ARM64** hardware.

The tutorial sets up a custom bash session with custom global variables, such as ```LK_DEV_DIR``` and ```VM_DIR``` to facilitate and standardize commands. As such, it is my personal recommendation to also set up ```KERNEL``` and ```INITRD``` variables as follows:
``` bash
export INITRD="initrd.img-6.1.0-43-arm64"
export KERNEL="vmlinuz-6.1.0-43-arm64"
```
Of course, you will need to specify the correct kernel and initrd filenames to match your own.

It was important to adjust the ```create_vm_virsh``` command during the tutorial, adding ```device=disk,bus=virtio``` to the ```--disk``` flag, as the hypervisor was trying to use **x86-64** drivers instead of the correct **ARM64** ones.

As a sysadmin at the [Rede/GNU Linux](https://linux.ime.usp.br/), I am fairly used to managing virtual machines and containers, so the rest of the tutorial went pretty smoothly - except for one small caveat.

### Configuring SSH access from the host to the VM
The tutorial explains how to enable SSH access to the VM. Problem is, the debian image downloaded - the one used to replace the image that was removed - was missing the ```openssh-server``` package.

This wouldn't usually be a problem as you can just download it from the ```apt``` package manager, however the VM had no access to the internet.

Luckily, the tutorial has a troubleshooting section that covers this exact scenario:

> If you are developing on a host device with an Arch-based distribution, however, there is a chance that your ```libvirt``` network contains an incorrect firewall configuration. On the host device, open ```/etc/libvirt/network.conf``` with your favorite text editor and set ```firewall_backend=iptables```.

After editing the ```/etc/libvirt/network.conf``` config file, the VM was able to properly connect to the internet, and I was able to connect to the VM through an SSH connection from the host.

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> If using ```iptables```, the ```iptables-nft``` package may be necessary for the above fix to work.
{: .prompt-info}
<!-- markdownlint-restore -->

## Conclusion
In the end, I was able to complete the first tutorial having properly set up a test environment for my future Linux Kernel development endeavors.

For reference, this was what my ```activate.sh``` file looked like by the end of the first tutorial:

``` bash
#!/usr/bin/env bash

# environment variables
export LK_DEV_DIR='/home/lk_dev' # path to testing environment directory
export VM_DIR="${LK_DEV_DIR}/vm" # path to VM directory
export BOOT_DIR="${VM_DIR}/arm64_boot" # path to boot artifacts

export INITRD="initrd.img-6.1.0-43-arm64"
export KERNEL="vmlinuz-6.1.0-43-arm64"

# utility functions

function launch_vm_qemu() {
    qemu-system-aarch64 \
        -M virt,gic-version=3 \
        -m 2G -cpu cortex-a57 \
        -smp 2 \
        -netdev user,id=net0 -device virtio-net-device,netdev=net0 \
        -initrd "${BOOT_DIR}/$INITRD" \
        -kernel "${BOOT_DIR}/$KERNEL" \
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
      --boot kernel=${BOOT_DIR}/${KERNEL},initrd=${BOOT_DIR}/${INITRD},kernel_args="loglevel=8 root=/dev/vda2 rootwait" \
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
