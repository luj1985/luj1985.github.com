---
layout: post
title: "Construct Hadoop Environment<br />Part 1 Gentoo Linux"
date: 2012-03-12 16:38
comments: true
categories: Linux
---
I want to do some Hadoop experiments, but currently I don't have enough machine to run Hadoop in full-distribute mode. 

But I have a PC in my hand, and I'd like to configure this machine as hypervisor and run several guest machines on it. 
All guest machines should be able to communicate with each other, and should be able to access from a remote machine.

Here I choose Gentoo as the operation system of hypervisor.

# Preparation
Firstly download the minimal installation media from [Gentoo offical site](http://www.gentoo.org/main/en/where.xml), 
Use any Live CD is OK, the purpose of this media is to boot system up.

The CD-ROM of my machine does not function well, so I create a bootable Live USB drive 
using [UNetbootin](http://unetbootin.sourceforge.net/), and boot from USB drive.

# Gentoo Installation
Below steps refer [Gentoo Handbook](http://www.gentoo.org/doc/en/handbook/handbook-amd64.xml) and [Gentoo LVM Guide](http://www.gentoo.org/doc/en/lvm2.xml)
## Setup network
Gentoo installation process relies on network heavily.
The installation media only contains some nesessary programs to bootstrap system. 
Other source code packages require download from remote site (Gentoo portage)

Active network card first, then use DHCP client to obtain an IP address and DNS information.
```
ifconfig eth0 up
dhcpcd eth0
```

## Manage hard disk
There are three hard disks installed in my machine
```
/dev/sda 120G
/dev/sdb 500G
/dev/sdc 500G
```
Though my _/dev/sda_ is a **PATA** device, but it still be recoginized as **SATA**. 
I did some search and found that kernel 2.6.19 has merged PATA drivers into **SATA**, 
this make the device file name starts with _sd_ 
([Libata PATA (Parallel ATA) merge](http://kernelnewbies.org/Linux_2_6_19#head-cdcbaa9c1b476decdc064e0a75d23d1328b1ddce))

I want to use LVM to manage my hard disks. 
Grub does not support boot from LVM, so need to create a separate _/boot_ partition. (Grub2 may support boot from LVM directly)

In order to start LVM `vgscan`, `vgchange` and some other system tools are required, and these tools are stores in the file system, 
but during the boot process the file system (LVM) haven't been activated yet. 
Read file system to active file sytem. This came to a Chicken or the egg question.
```
vgscan
vgchange -a y
```

To prevent this situation Linux need to create an initial RAM disk including those system tools, mount this RAM disk as a fake root file system , 
then run these tools to active LVM, finally switch the root file system to LVM.

The detail boot process is descript below (Referred from [here](http://www.midhgard.it/docs/lvm/html/install.kernel.html#boot.process.overview)):

1. Grub is loaded
2. Kernel image is loaded
3. RAM disk image with LVM support is loaded
4. RAM disk is mounted as root
5. LVM started
6. Volume groups are detected and activated
7. Real root file system over LVM is mounted read-only as new root file system
8. RAM disk is unmounted
9. LVM is started again (root has changed)
10. Root file system is checked for errors
11. Re-mount root read-write
12. Now root is ready, start the normal boot

In fact, Grub loading Kernel image and inital RAM disk is not via the file system interface ([VFS](http://en.wikipedia.org/wiki/Virtual_file_system)), 
Grub read hard disk directly but follow the file system format.

This process is called **Grub stage1.5**. You could find many file under _/boot/grub_ directory end with _1_5_, 
like _e2fs_stage1_5_, _fat_stage1_5_ etc.
If there is no corresponding 1.5 stage file for you _/boot_ paritition file system, you couldn't load kernel to start system.

I think [btrfs](https://btrfs.wiki.kernel.org/) is a better choice than LVM, but currently btrfs is still in development.

### Create partition
Create a separate partition to mount _/boot_
```
   Device  Boot     Start         End      Blocks   Id  System
/dev/sda1   *        2048      104447       51200   83  Linux
/dev/sda2          104448   234441647   117168600   83  Linux LVM
```

### Construct LVM
[LVM (Logical Volume Management)](http://en.wikipedia.org/wiki/Logical_Volume_Manager_\(Linux\))

* Create PV (Physical Volume). PV indicate that which hard disk or partitions will be used to assemble the LVM.
* Create VG (Volume Group). VG act like a pool or virtual disk, hide the real disk behind.
* Create LV (Logical Volume). LV is like the traditional hard disk partition which can contain file system.

In fact, create a LV is like create a block device but with dynamic capacity.
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
Could use LVM strip to accelerate the disk speed, like
```
lvcreate -i 3 -L 10G -n home vg
```
Here `-i 3` option means map the LV to 3 PV, act like **[RAID 0](http://en.wikipedia.org/wiki/Standard_RAID_levels#RAID_0)**.

LVM2 can do [snapshot](http://tldp.org/HOWTO/LVM-HOWTO/snapshots_backup.html), I could use this feature to perform the backup, 
that's why I create so many partitions.

### Create file system and mount
Create file systems and mount block devices to correspoding mount point.
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

mkdir -p /mnt/gentoo/{boot,home,var,tmp,usr/portage}
mount /dev/sda1 /mnt/gentoo/boot

mount /dev/vg/home    /mnt/gentoo/home
mount /dev/vg/var     /mnt/gentoo/var
mount /dev/vg/tmp     /mnt/gentoo/tmp
mount /dev/vg/portage /mnt/gentoo/usr/portage

mkdir /mnt/gentoo/var/tmp
mount /dev/vg/vartmp /mnt/gentoo/var/tmp

swapon /dev/vg/swap

chmod 1777 /mnt/gentoo/var/tmp
chmod 1777 /mnt/gentoo/tmp
```

## Install Gentoo base system
Download _stage3_ file and the latest _portage_ snapshot package, then extract them to /mnt/gentoo.

Stage3 file contains a full function Linux with mininal features. There are also stage1, stage2, but not supported by Gentoo right now.
```
cd /mnt/gentoo
wget http://mirrors.sohu.com/gentoo/releases/amd64/current-stage3/stage3-amd64-20120301.tar.bz2
wget http://mirrors.sohu.com/gentoo/snapshots/portage-latest.tar.bz2

tar xjpf stage3-amd64-20120216.tar.bz2
tar xjf portage-latest.tar.bz2 -C /mnt/gentoo/usr

mount -o bind /proc /mnt/gentoo/proc
mount -o bind /dev  /mnt/gentoo/dev
```

### Configure Gentoo base system
Copy DNS resolve configuration file, it contains the DNS server information.
(This file was generated after network was setupped)
```
cp -L /etc/resolv.conf /mnt/gentoo/etc/
```

Modify make.conf
/mnt/gentoo/etc/make.conf
```
CFLAGS="-march=k8 -O3 -pipe -fomit-frame-pointer"
USE="-X -gtk -qt -gnome -kde -ipv6 mmx sse sse2"
MAKEOPTS="-j3"
```

## Chroot to new Gentoo base system
Configure the newly created Gentoo system, and make sure it can be started up next time.
```
chroot /mnt/gentoo /bin/bash

env-update
source /etc/profile
```

### Assign the password of root user
```
passwd
```
Otherwise you couldn't login into the newly created system after reboot.

### Sync portage tree
```
emerge --sync
```
or use
```
emerge-webrsync
```
From the manual page, `emerge-webrsync` will download the entire portage tree as a tarball, if it's the first time do the sync it will be much efficient.

### Install and configure kernel source
```
emerge gentoo-sources
```
Some packages like `grub` will read kernel configuration, it's better to configure kernel first.

#### Configurre Linux kernels
Run command `lspci -k` to view the driver current used by LiveCD, and enable them in Linux kernel.

[Here](http://www.linuxtopia.org/online_books/linux_kernel/kernel_configuration/index.html) is a comprehensive document explain how to manually configure a kernel, 
especially how to find the correct device driver.

Because I need to create an initial RAM disk, install `sys-kernel/genkernel` package can make things much simpler.
```
emerge genkernel
genkernel --menuconfig --lvm --bootloader=grub all
```

### Select profile
```
eselect profile list
eselect profile set 7
```
Once you selected `no-multilib`, it's a little hard to get back.

### Other configuration
```
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen

cp /usr/share/zoneinfo/UTC /etc/localtime
echo "UTC" > /etc/timezone
```
Install and configure support tools

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

### Edit /etc/fstab
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
Recommand use UUID to identify the block device. Because the device label like _/dev/sda_ many change after reboot. 

Run command `blkid` or `ls -l /dev/disk/by-uuid` to discover the device UUID.

### Install Grub
I have chosen _no-multilib_ profile, it ask to install _grub-static_ package instead of _grub_
```
emerge grub-static
grep -v rootfs /proc/mounts > /etc/mtab

grub
grub> find /boot/grub/stage1
grub> root (hd0,0)
grub> setup (hd0)
grub> quit
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

# Post installation
Run command to recompile whole system, and remove the unneeded packages
```
emerge --emptytree world
emerge --depclean
revdep-rebuild
```
