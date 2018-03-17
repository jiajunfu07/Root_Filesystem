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
-append "console=ttyAMA0,115200" -dtb ../linux-stable/arch/arm/boot/dts/versatile-pb.dtb \
-initrd initramfs.cpio.gz
```
