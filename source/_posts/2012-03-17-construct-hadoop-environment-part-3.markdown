---
layout: post
title: "Construct Hadoop environment <br />Part 3 KVM Guest"
date: 2012-03-17 11:25
comments: true
categories: 
---
Create a KVM guest as template, then clone this template to create new instance.

I still use Gentoo Linux as guest system, and install with minimal components

In order to run Hadoop guest machine only require Java environment and sshd

#Configure NFS server on KVM host machine
My host machine already has portage tree, I could export portage partition via NFS. Save bandwidth and disk space.

[Virtfs](http://www.linuxplumbersconf.org/2010/ocw/proposals/603) can pass the file system to KVM guest machine directly. But I don't use it here.

##Kernel configuration
```
File systems
  [*] Network File Systems
    <M> NFS server support
    [*]   NFS server support for NFS version 3
```

##Install required packages
```
emerge -av net-fs/nfs-utils
/etc/init.d/nfs start
rc-update add nfs default
```

##Export NFS partition
export file system (/etc/exports)
```
/usr/portage 192.168.2.0/255.255.255.0(async,rw,no_root_squash,no_subtree_check)
```
Reload NFS
```
/etc/init.d/nfs reload
```

#Create KVM guest image
Assign a static IP address in my router.
```
       MAC                IP
00-00-00-00-00-01   192.168.2.103
```
Because I want to use DNS to name all the machine.

##Create kvm image with QEMU tools
Login as root and create a KVM image
```
qemu-img create -f qcow2 -o preallocation=metadata hadoop.img 10G
qemu-kvm -hda hadoop.img -smp 2 -m 1024 -cdrom install-amd64-minimal-20120223.iso \
    -net nic,mac=00:00:00:00:00:01,vlan=0 -net tap,vlan=0 -boot d -vnc :0
```

The IP address of my KVM host machine is ``192.168.2.101``, I connect VNC using ``vncviewer``
```
vncviewer 192.168.2.101:0
```
During Gentoo installation, it will compile a lot of packages. 
So here I request a large mount of memory, after installation I will reduce its size.

##Use libvirt tools to create KVM guest images
libvirt is just a front-end of QEMU, but it provide many handy tools to manage the virtual machine.
```
echo "app-emulation/libvirt qemu virt-network
emerge app-emulation libvirt
/etc/init.d/libvirtd start
rc-update add libvirtd default
```

Run command to connect libvirtd
```
virsh --connect qemu:///system
```
Or connect from a remote machine
```
virsh --connect ssh+qemu:///root@192.168.2.101/system
```

#Install Gentoo Linux (guest system)
##Create partition
fdisk /dev/sda
```
Device    Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048      104447       51200   83  Linux
/dev/sda2          104448     1128447      512000   83  Linux
/dev/sda3         1128448    20971519     9921536   83  Linux
```
##Create and mount file system
```
mkfs.ext4 /dev/sda3
mount /dev/sda3 /mnt/gentoo

mkfs.ext4 dev/sda1
mkdir /mnt/gentoo/boot
mount /dev/sda1 /mnt/gentoo

mkswap /dev/sda2
swapon /dev/sda2

cd /mnt/gentoo
tar xjpf stage3-amd64-20120301.tar.bz2

mkdir /mnt/gentoo/usr/portage
mount -t nfs 192.168.2.101:/usr/portage /mnt/gentoo/usr/portage

mount -o bind /proc /mnt/gentoo/proc
mount -o rbind /dev /mnt/gentoo/dev
mount -o sys /dev /mnt/gentoo/sys
```

##System configuration
##DNS resolve
```
cp -L /etc/resolve.conf /mnt/gentoo/etc
```
##make.conf
```
CFLAGS="-march=k8 -Os -pipe -fomit-frame-pointer"
CXXFLAGS="${CFLAGS}"
CHOST="x86_64-pc-linux-gnu"

USE="mmx sse sse2 -X -gnome -gtk -ipv6 -kde -qt"
```

##Chroot
```
chroot /mnt/gentoo /bin/bash
env-update
source /etc/profile
```

##Configure the new system
Edit /etc/fstab
```
UUID=f06a74d7-e64e-4b61-b37d-23ed46843bc4 /boot ext4 noatime  1 2
UUID=f2493fe6-ee09-4e91-8066-ebafc30436d4 none  swap sw       0 0 
UUID=0acf7f67-b533-4950-8621-2474c6a4e4b9 /     ext4 noatime  0 1 
192.168.2.101:/usr/portage    /usr/portage      nfs  defaults,noauto 0 0
```

```
passwd
eselect profile set 7

echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen

cp /usr/share/zoneinfo/UTC /etc/localtime
echo "Asia/Shanghai" > /etc/timezone

echo config_etho=\"dhcp\" >> /etc/conf.d/net

emerge -av dhcpcd nfs-utils
emerge gentoo-sources
emerge grub-static

rc-update add sshd default
rc-update add rpcbind default
rc-update add nfsmount default

ln -s /etc/init.d/net.lo /etc/init.d/net.eth0
rc-update add net.eth0 default

grub-install --no-floppy /dev/sda

cd /usr/src/linux
make menuconfig
make install
make modules_install
```

edit /boot/grub/grub.conf
```
default 0
timeout 1

title Gentoo Linux 3.2.1-r2
root (hd0,0)
kernel /boot/vmlinuz-3.2.1-gentoo-r2 root=/dev/sda3
```

#Install JDK
Hadoop require Java envrionment, so install jdk
```
emerge -av jdk
```
JAVA_HOME
```
java-config --jdk-home
```

#Free disk space
Because this guest machine is used as a template, will create many instance. 

Refer [this document](http://en.gentoo-wiki.com/wiki/Freeing_Up_Disk_Space) to see how to free disk space

#Compact the KVM guest image
KVM guest disk format is ``qcow2``, after system installation the disk size has grew very big, but
in fact most of them are empty (File created, then deleted)

Firstly, shutdown the KVM guest machine, then run following command to compact the image size
```
qemu convert -c -f qcow2 -O qcow2 hadoop.img hadoop.compact.img
```
