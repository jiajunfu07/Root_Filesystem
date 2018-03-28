# Root_Filesystem

## Create the staging directory
Create a staging directory in myworkspace/ on my host computer where I can assemble the files that will eventually be transferred to the target.
```
$ mkdir ~/myworkspace/rootfs
$ cd ~/myworkspace/rootfs
$ mkdir bin dev etc home lib proc sbin sys tmp usr var
$ mkdir -p var/log
```
Use tree command to see the directory hierarchy
```
$ tree -d
```

## Use BusyBox to generate utility
Clone BusyBox
```
$ cd ~/myworkspace
$ git clone git://busybox.net/busybox.git
$ cd busybox
$ git checkout 1_28_1
```
Build BusyBox
```
$ make distclean
$ make defconfig
$ make menuconfig
```
Change the install path from ./_ install to ../rootfs in Busybox Setting | Installation Options
```
$ make ARCH=arm CROSS_COMPILE=arm-unknown-linux-gnueabi-
$ make ARCH=arm CROSS_COMPILE=arm-unknown-linux-gnueabi- install
```
There will be four files/directories in the staging directory: bin, linuxrc, sbin, usr.

## Copy libraries to the staging directory
Use the readelf command to see those libraries I need
```
$ cd ~/myworkspace/rootfs
$ arm-unknown-linux-gnueabi-readelf -a bin/busybox | grep "program interpreter"
$ arm-unknown-linux-gnueabi-readelf -a bin/busybox | grep "Shared library"

$ export SYSROOT=$(arm-unknown-linux-gnueabi-gcc -print-sysroot)
$ cd $SYSROOT
$ ls -l lib/ld-linux.so.3
$ ls -l libc.so.6
$ ls -l libm.so.6
```
Copy the each library using cp -a, which will preserve the symbolic link
```
$ cd ~/myworkspace/rootfs
$ cp -a $SYSROOT/lib/ld-linux.so.3 lib
$ cp -a $SYSROOT/lib/ld-2.25.so lib
$ cp -a $SYSROOT/lib/libc.so.6 lib
$ cp -a $SYSROOT/lib/libc-2.25.so lib
$ cp -a $SYSROOT/lib/libm.so.6 lib
$ cp -a $SYSROOT/lib/libm-2.25.so lib
```
## Reduce the size by stripping
```
$ file rootfs/lib/libc-2.25.so
```
It shows not stripped
```
$ ls -og rootfs/lib/libc-2.25.so
$ arm-unknown-linux-gnueabi-strip rootfs/lib/libc-2.25.so
$ file rootfs/lib/libc-2.25.so
$ ls -og rootfs/lib/libc-2.25.so
```

## Create device nodes
In a really minimal root filesystem, only two nodes needed to boot with BusyBox: console and null
```
$ cd ~/myworkspace/rootfs
$ sudo mknod -m 666 dev/null c 1 3
$ sudo mknod -m 600 dev/console c 5 1
$ ls -l dev
```

## Install the kernel modules
The kernel modules are built in [kernel](https://github.com/jiajunfu07/Kernel#build-a-kernel-for-qemu)
```
make ARCH=arm CROSS_COMPILE=arm-known-linux-gnueabi- INSTALL_MOD_PATH=~/myworkspace/rootfs modules_install
```
The modules/ directory will be shown in /lib/

## Create a boot initramfs
```
$ cd ~/myworkspace/rootfs
$ find . | cpio -H newc -ov --owner root:root > ../initramfs.cpio
$ cd ..
$ gzip initramfs.cpio
$ mkimage -A arm -O linux -T ramdisk -d initramfs.cpio.gz uRamdisk
```
cpio with the option: --owner root:root will fix the file owner problem. \
initramfs.cpio.gz and uRamdisk will be generated in ~/myworkspace/

## Boot with QEMU
```
$ QEMU_AUDIO_DRV=none \
qemu-system-arm -m 256M -nographic -M versatilepb -kernel ../linux-stable/arch/arm/boot/zImage \
-append "console=ttyAMA0 rdinit=/bin/sh" -dtb ../linux-stable/arch/arm/boot/dts/versatile-pb.dtb \
-initrd initramfs.cpio.gz
```

## Add init program and mount vfs
The init program is from BusyBox, it begins by reading the configuration file, /etc/inittab.(BusyBox init provide a default inittab if none is present) Create /etc/inittab file in the staging directory.
```
::sysinit:/etc/init.d/rcS
::askfirst:-/bin/ash
```
Create /etc/init.d/rcS file in the staging directory. Mount the proc and sysfs filesystems.
```
#!/bin/sh
mount -t proc proc /proc
mount -t sysfs sysfs /sys
```
make rcS executable.
```
$ cd rootfs
$ chmod +x etc/init.d/rcS
```
Recreate initramfs and change rdinit=/bin/sh to rdinit=/sbin/init when booting with QEMU.

## Start the log daemon process
Add daemon to /etc/inittab
```
::respawn:/sbin/syslogd -n
```

## Add user accounts
Create /etc/passwd in the staging directory
```
root:x:0:0:root:/root:/bin/sh
daemon:x:1:1:daemon:/usr/sbin:/bin/false
```
Create /etc/shadow in the staging directory
```
root::10933:0:99999:7:::
daemon:*:10933:0:99999:7:::
```
Create /etc/group in the staging directory
```
root:x:0:
daemon:x:1:
```
Change the permission of the files
```
$ chmod 0600 /etc/passwd /etc/shadow /etc/group 
```
Modify inittab, Add
```
::respawn:/sbin/getty 115200 console
```
Delete 
```
::askfirst:-/bin/ash
```
Recreate ramdisk(initramfs) and try it out using QEMU.

## Mount device nodes
Checkout CONFIG_DEVTMPFS in kernel configuration using 
```
make ARCH=arm menuconfig
```
Rebuild kernel.
Add the following to inittab.
```
mount -t devtmpfs devtmpfs /dev
```
Recreate ramdisk(initramfs) and try it out using QEMU.

## Configuring the network
Here an Ethernet interface is required
