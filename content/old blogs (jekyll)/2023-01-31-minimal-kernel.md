---
layout: post
title: Compiling and Running a Minimal Kernel with Busybox
categories: [Linux, Kernel]
---
The exploration logs are found at [here](https://github.com/tch1001/kernel).

## Building the Kernel 
Of course the small kernel must still be functional, so we will test it using [qemu](https://www.qemu.org/). One can also use virtualbox but I chose qemu because I will be working remotely using SSH and VNC takes too much mobile data lol.

```bash
$ git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git --depth 1 --branch v6.1.4
$ cd linux
$ git log
commit 2cb8e624295ffa0c4d659fcec7d9e7a6c48de156 (grafted, HEAD, tag: v6.1.4)
    Author: Greg Kroah-Hartman <gregkh@linuxfoundation.org>
        Date:   Sat Jan 7 11:12:04 2023 +0100
	Linux 6.1.4
$ apt install -y libncurses5 libncurses5-dev bison flex
$ make menuconfig 
# go enable PVH if u want to run in QEMU
$ vim .config # remove "debian" things, and the "certs/signing_key.pem"
$ apt install -y libssl-dev libelf-dev
$ make -j$(nproc) # change -jN to your liking
# if they prompt something involving certs, just press enter (default choices)
```
Important: Don't forget to enable `CONFIG_PVH=y` in `.config` if you want to run in QEMU.
```
$ ls -lh vmlinux
```
`vmlinux` should exist, if it doesn't, something went wrong and go back to [build](#building-the-kernel).

Otherwise, let's run our kernel! 

## Running in QEMU
We need an Initrd? 
What's Init RAM Disk? 
It is the first thing that the kernel looks for when it wakes up.

You can try the following command and see it complain that it was unable to find an `init`.
```
$ qemu-system-x86_64 -kernel ./vmlinux -nographic --append "console=tty0 console=ttyS0 panic=1 root=/dev/sda rootfstype=ext2" -hda rootfs.ext2 -m 1024 -vga none -display none -serial mon:stdio -no-reboot 
# Some kernel logs before we see
[    1.512696] Run /sbin/init as init process
[    1.513360] Run /etc/init as init process
[    1.513575] Run /bin/init as init process
[    1.513772] Run /bin/sh as init process
[    1.514056] Kernel panic - not syncing: No working init found.  Try passing init= option to kernel.
```
The kernel was frantically looking for an `init` executable in `/sbin`,`/etc`,`/bin`. Really just any executable (can be a shell script, or a binary executable). 
### Our own init
Let's write our own init for now. The `sleep` is to prevent the kernel from panicing (it will panic if `init` exits).
```
$ mkdir vfs && cd vfs
$ cat << EOF > hello-kernel.c
#include <stdio.h>

int main(){
        printf("Hello, kernel!\n");
        sleep(9999999999999);
}
EOF
$ gcc --static hello-kernel.c -o init
```
Now we just need to package it up into a `cpio` format.
```
$ find . | cpio -o -H newc | gzip > root.cpio.gz
```
Then we can run QEMU successfully
```
$ qemu-system-x86_64 -kernel ./vmlinux -nographic --append "console=tty0 console=ttyS0 panic=1 root=/dev/sda rootfstype=ext2" -hda rootfs.ext2 -m 1024 -vga none -display none -serial mon:stdio -no-reboot -initrd initrd/root.cpio.gz
[    1.461412] x86/mm: Checked W+X mappings: passed, no W+X pages found.
[    1.461651] Run /init as init process
Hello, kernel!
[    1.946056] tsc: Refined TSC clocksource calibration: 3399.960 MHz
[    1.946430] clocksource: tsc: mask: 0xffffffffffffffff max_cycles: 0x3102293934f, max_idle_ns: 440795316s
[    1.946791] clocksource: Switched to clocksource tsc
```
Great! But now we don't just want our init to say hello, we want our init to be able to put us into a shell and let us navigate and do productive work! That is where [busybox](https://busybox.net/) comes in!

### Busybox
Git clone and compile Busybox
```
$ cd .. # don't do stuff in the linux directory, git will be confused
$ git clone --depth=1 https://github.com/mirror/busybox.git && cd busybox
$ make defconfig 
$ vi .config # set CONFIG_STATIC=y
$ make -j$(nproc)
$ make CONFIG_PREFIX=$PWD/BUSYBOX install
$ ls BUSYBOX 
bin  linuxrc  sbin  usr
```
Then we make out initrd with busybox 
```
$ cd ../linux # return back to linux directory
$ mkdir initrd && cd initrd # like vfs (from previously), this will house our new initrd
$ mkdir etc proc sys
$ cat << EOF > init                              
#!/bin/sh
                           
mount -t proc proc /proc
mount -t sysfs none /sys
                                                      
# https://busybox.net/FAQ.html#job_control
                                                      
mknod /dev/ttyS0 c 4 64
setsid sh -c 'exec sh </dev/ttyS0 >/dev/ttyS0 2>&1'
EOF                                                   
$ chmod +x init 
```
The above `init` shell script will be our entrypoint
```
$ cp ../../busybox/BUSYBOX/* . # copy over our compiled busybox utilities
$ find . | cpio -o -H newc | gzip > root.cpio.gz # package it
```
Basically, `init` will call our busybox utilities! Internally, `/bin/ls` is a symlink to `/bin/busybox ls`

Fun fact: Alpine linux uses busybox, Alpine linux is popular in docker containers due to it's small size.

### Booting
```
$ qemu-system-x86_64 -kernel ./vmlinux -nographic --append "console=tty0 console=ttyS0 panic=1 root=/dev/sda rootfstype=ext2" -hda rootfs.ext2 -m 1024 -vga none -display none -serial mon:stdio -no-reboot -initrd initrd/root.cpio.gz 
```
We should be dropped into a shell and now we can type `busybox` to see the commands we can use.


## Running the Kernel on Hardware
If you're on ubuntu, you can replace your current kernel (`uname -r`) with the one you compiled
```
$ update-grub2 # make sure you're using the grub bootloader
$ make modules_install
$ make install
```
When booting up, press and hold `<Shift>` to go to the grub menu, then go to advanced options to select the kernel you want to boot with.

# Minimizing the Kernel Size
Let's try to compile a kernel that is as small as possible.

## Measuring Size
The kernel size can be determined using 
```bash
$ ls -lh vmlinux
-rwxr-xr-x 1 tch tch 1.1G Jan 31 16:19 vmlinux
```
In this case, I will measure the size of `vmlinux` instead of `vmlinuz` since compression is cheating.

Using the default, we get a size of `1103946480` or `1.1G`. 

## My Personal Best
I managed to get it down to 51M with [this](/downloads/minimal.config) config.
```
ls -lh vmlinux
-rwxrwxr-x 1 tch tch 51M Feb  4 11:49 vmlinux
```
Theoretically it should be able to get down to around 10M as that is what I saw on some embedded linux forums. But I have other projects on hand that require my immediate attention, I'll come back to it one day... 

# Happy Coding!
If you run into any errors, you may drop me a text on telegram!
