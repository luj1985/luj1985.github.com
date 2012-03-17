---
layout: post
title: "Construct Hadoop environment <br />Part 1 Gentoo Linux"
date: 2012-03-12 16:38
comments: true
categories: Linux
---

I want to do some Hadoop experiment, but currently I don't have enough machines to run Hadoop in full-distribute mode. 
So I decide to configure a KVM host and run several guests. 

# Preparation
First download the minimal installation media from [Gentoo offical site](http://www.gentoo.org/main/en/where.xml), 
use any Live CD is OK, the purpose of this media is used to boot system up.

And the CD-ROM of my machine does not function well, I create a bootable Live USB drive 
using [UNetbootin](http://unetbootin.sourceforge.net/), and boot from USB drive

# Manage hard disk
There are three hard disks in my machine
```
/dev/sda 120G
/dev/sdb 500G
/dev/sdc 500G
```
Though my `/dev/sda` is a PATA device, but it still be recoginized as SATA. 
Because kernel 2.6.19 has merged PATA drivers into SATA 
([Libata PATA (Parallel ATA) merge](http://kernelnewbies.org/Linux_2_6_19#head-cdcbaa9c1b476decdc064e0a75d23d1328b1ddce))

I want to use LVM to manage the hard disk, Grub does not support boot from LVM, so have to separate a /boot partition. (Grub2 may support boot from LVM)

In order to start LVM must use ``vgscan``, ``vgchange`` and some other system tools, these tools are stores in the file system, but during the boot process
the file system (LVM) haven't been activated yet. This came to a Chicken or the egg question.

To prevent this situation, create a initial RAM disk including those system tools, mount this RAM disk as a fake root file system first, 
launch the tools and active LVM, finally change the root to LVM file system.

The detail boot process is like below (Referred from [here](http://www.midhgard.it/docs/lvm/html/install.kernel.html#boot.process.overview)):

* Grub is loaded.
* Kernel image is loaded
* RAM disk image with LVM support is loaded
* RAM disk is mounted as root
* LVM started
* Volume groups are detected and activated
* Real root file system over LVM is mounted read-only as new root file system
* RAM disk is unmounted
* LVM is started again (root has changed)
* Root file system is checked for errors
* Re-mount root read-write
* Now root is ready, start the normal boot

In fact, loading Kernel image and inital RAM disk is not via the file system, Grub read hard disk directly but follow the file system format.
This is called Grub stage1.5. You could find many file under /boot/grub end with 1_5, like ``e2fs_stage1_5``, ``fat_stage1_5`` etc.
If there is no corresponding 1.5 stage file for you /boot paritition file system, you couldn't boot your system.

I think [btrfs](https://btrfs.wiki.kernel.org/) is a better choice, but currently btrfs is under heavy develop.

## Create partition
Create a separate partition to mount /boot
```
   Device  Boot     Start         End      Blocks   Id  System
/dev/sda1   *        2048      104447       51200   83  Linux
/dev/sda2          104448   234441647   117168600   83  Linux LVM
```

## Construct LVM
LVM (Logical Volume Management)

* Create PV (Physical Volume). PV indicate that which hard disk or partition you want to use to assemble the LVM.
* Create VG (Volume Group). VG act like a pool or virtual disk, hide the real disk behind.
* Create LV (Logical Volume). LV is like the traditional hard disk partition which can contain file system.

In fact, create a LV is like create a block device which size can be changed.
```
pvcreate /dev/sda2 /dev/sdb /dev/sdc

vgcreate vg /dev/sda2 /dev/sdb /dev/sdc

lvcreate -L 2G  -n swap    vg
lvcreate -L 4G  -n root    vg
lvcreate -L 10G -n home    vg
lvcreate -L 5G  -n var     vg
lvcreate -L 5G  -n vartmp  vg
lvcreate -L 10G -n portage vg
lvcreate -L 5G  -n tmp     vg
```
To accelerate the disk speed, could use LVM strip, like
```
lvcreate -i 3 -L 10G -n home vg
```
``-i 3`` option means map the LV to 3 PV, act like RAID 0

{% blockquote %}
LVM2 can do snapshot, I could use this feature to do the backup, that's why I create so many partitions.
{% endblockquote %}
[How to do LVM snapshot](http://tldp.org/HOWTO/LVM-HOWTO/snapshots_backup.html)

## Create file system and mount
```
mkfs.ext4 /dev/sda1
mkfs.ext4 /dev/vg/root
mkfs.ext4 /dev/vg/home
mkfs.ext4 /dev/vg/portage
mkfs.ext4 /dev/vg/var
mkfs.ext4 /dev/vg/vartmp
mkfs.ext4 /dev/vg/tmp
mkswap /dev/vg/swap

mount /dev/vg/root /mnt/gentoo

mkdir /mnt/gentoo/boot
mount /dev/sda1 /mnt/gentoo/boot

mkdir /mnt/gentoo/home
mount /dev/vg/home /mnt/gentoo/home

mkdir /mnt/gentoo/var
mount /dev/vg/var /mnt/gentoo/var

mkdir /mnt/gentoo/var/tmp
mount /dev/vg/vartmp /mnt/gentoo/var/tmp

mkdir /mnt/gentoo/tmp
mount /dev/vg/tmp /mnt/gentoo/tmp

mkdir -p /mnt/gentoo/usr/portage
mount /dev/vg/portage /mnt/gentoo/usr/portage

swapon /dev/vg/swap

chmod 1777 /mnt/gentoo/var/tmp
chmod 1777 /mnt/gentoo/tmp
```

# Install Gentoo base system
Download stage3 file and the latest portage snapshot and extract them to /mnt/gentoo.
```
cd /mnt/gentoo
wget http://mirrors.sohu.com/gentoo/releases/amd64/current-stage3/stage3-amd64-20120301.tar.bz2
wget http://mirrors.sohu.com/gentoo/snapshots/portage-latest.tar.bz2

tar xjpf stage3-amd64-20120216.tar.bz2
tar xjf portage-latest.tar.bz2 -C /mnt/gentoo/usr

mount -o bind /proc /mnt/gentoo/proc
mount -o rbind /dev /mnt/gentoo/dev
mount -o bind /sys /mnt/gentoo/sys
```

## Configure Gentoo base system
```
cp /etc/resove.conf /mnt/gentoo/etc/resove.conf
mirrorselect -i -o >> /mnt/gentoo/etc/make.conf
mirrorselect -i -r -o >> /mnt/gentoo/etc/make.conf
```

Modify make.conf
/mnt/gentoo/etc/make.conf
```
CFLAGS="-march=k8 -O3 -pipe -fomit-frame-pointer"
USE="-X -gtk -qt -gnome -kde -ipv6 mmx sse sse2"
MAKEOPTS="-j3"
```

# Chroot to new Gentoo base system
Below steps are follow [this Guide](http://www.gentoo.org/doc/en/handbook/handbook-amd64.xml).
Configure the newly created Gentoo system, and make sure it can be started up next time.
```
chroot /mnt/gentoo /bin/bash

env-update
source /etc/profile
```

## Assign the password of root user
```
passwd
```
Otherwise you couldn't login into the newly created system.

## Sync portage tree
```
emerge --sync
```
or use
```
emerge-webrsync
```
From the manual page, ``emerge-webrsync`` will download the entire portage tree as a tarball, if it's the first time do the sync it will be much faster.

##Install kernel source
```
emerge gentoo-sources
```

## Select profile
```
eselect profile list
eselect profile set 7
```
Once you selected ``no-multilib``, it's hard to get back.

## glibc locale
```
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
```

## Timezone
```
cp /usr/share/zoneinfo/UTC /etc/localtime
echo "Asia/Shanghai" > /etc/timezone
```

## Edit /etc/fstab
```
/dev/mapper/vg-root     /             ext4  noatime  1 1
UUID=b10055bb-2e44-485e-9964-c37e4fea0f71 /boot   ext4    noatime  1 2
/dev/mapper/vg-home     /home         ext4  noatime  0 0
/dev/mapper/vg-var      /var          ext4  noatime  0 0
/dev/mapper/vg-vartmp   /var/tmp      ext4  noatime  0 0
/dev/mapper/vg-tmp      /tmp          ext4  noatime  0 0
/dev/mapper/vg-portage  /usr/portage  ext4  noatime  0 0
/dev/mapper/vg-swap     swap          swap  sw       0 0
```
Recommand use UUID to identify the block device. Because the device label like /dev/sda many change when reboot. 
Run command `blkid` or `ls -l /dev/disk/by-uuid` to view the device UUID.

## Install Grub
I have chosen 'no-multilib' profile, it require to install 'grub-static' package instead of 'grub'
```
emerge grub-static
grep -v rootfs /proc/mounts > /etc/mtab

grub
grub> find /boot/grub/stage1
grub> root (hd0,0)
grub> setup (hd0)
grub> quit
```

## Install and configure support tools
* app-admin/logrotate
* app-admin/syslog-ng
* app-portage/ufed
* app-portage/eix
* app-portage/gentoolkit
* app-editors/vim
* net-misc/dhcpcd
* sys-apps/mlocate
* sys-process/vixie-cron
* sys-process/htop
* sys-fs/e2fsprogs
* sys-fs/lvm2
```
rc-update add syslog-ng default
rc-update add vixie-cron default
rc-update add sshd default
rc-update add lvm boot

ln -s /etc/init.d/net.lo /etc/init.d/net.eth0
rc-update add net.eth0 default
echo config_eth0=\"dhcp\" >> /etc/conf.d/net
```

## Configurre Linux kernels
[Here is a comprehensive document](http://www.linuxtopia.org/online_books/linux_kernel/kernel_configuration/index.html) 
explain how to manually configure a kernel, especially the device driver which is the main part of Linux kernel.

Because I must use initial RAM disk, use ``sys-kernel/genkernel`` to make it simpler.
```
emerge genkernel
genkernel --menuconfig --lvm --bootloader=grub all
```
Edit grub.conf
```
default 0
timeout 1

title Gentoo Linux
root (hd0,0)
kernel /boot/kernel-genkernel-x86_64-3.2.1-gentoo-r2 root=/dev/ram0 real_root=/dev/vg/root dolvm rootfstype=ext4 vga=791
initrd /boot/initramfs-genkernel-x86_64-3.2.1-gentoo-r2
```

# Post the installation
Run command to recompile whole system
```
emerge --emptytree world
```
