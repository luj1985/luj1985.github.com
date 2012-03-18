---
layout: post
title: "Construct Hadoop environment<br />Part 3 KVM Guest"
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
It's fine to use this method, but there are too many options to run a perfect vm, it's hard to 
use command line to control

##Install libvirt on KVM host server
libvirt is just a front-end of QEMU, but it provide many handy tools to manage the virtual machine.
```
echo "app-emulation/libvirt qemu virt-network -lxc" >> /etc/portage/package.use
emerge app-emulation libvirt
/etc/init.d/libvirtd start
rc-update add libvirtd default
```

###Install management tools on remote client
* app-emulation/virt-manager

There is a command line tools available to connect remote KVM host
```
virsh -c qemu+ssh://root@192.168.2.101/system
```
virt-manager must use the same manchanism to connect remote host. Because virt-manager use ssh to 
connect the remote machine, it need to do authentication. But seems doesn't work well, so I copy 
the ssh public key to the remote server, then it works

###Domain XML
```
<domain type='kvm' id='4'>
  <name>hadoop-template</name>
  <uuid>dbeb78bf-adab-1d79-4592-d1b26d77ccfe</uuid>
  <memory>524288</memory>
  <currentMemory>524288</currentMemory>
  <vcpu>1</vcpu>
  <os>
    <type arch='x86_64' machine='pc-1.0'>hvm</type>
    <boot dev='cdrom'/>
    <boot dev='hd'/>
    <bootmenu enable='yes'/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <pae/>
  </features>
  <clock offset='utc'/>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>restart</on_crash>
  <devices>
    <emulator>/usr/bin/qemu-kvm</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/hadoopimages/hadoop-template.img'/>
      <target dev='vda' bus='virtio'/>
      <alias name='virtio-disk0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/nfs/install-amd64-minimal-20120223.iso'/>
      <target dev='hdc' bus='ide'/>
      <readonly/>
      <alias name='ide0-1-0'/>
      <address type='drive' controller='0' bus='1' unit='0'/>
    </disk>
    <controller type='usb' index='0'>
      <alias name='usb0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x2'/>
    </controller>
    <controller type='ide' index='0'>
      <alias name='ide0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x1'/>
    </controller>
    <interface type='bridge'>
      <mac address='00:00:00:00:00:01'/>
      <source bridge='br0'/>
      <target dev='vnet0'/>
      <model type='virtio'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
    <serial type='pty'>
      <source path='/dev/pts/0'/>
      <target port='0'/>
      <alias name='serial0'/>
    </serial>
    <console type='pty' tty='/dev/pts/0'>
      <source path='/dev/pts/0'/>
      <target type='serial' port='0'/>
      <alias name='serial0'/>
    </console>
    <input type='mouse' bus='ps2'/>
    <graphics type='vnc' port='5900' autoport='yes'/>
    <video>
      <model type='vga' vram='9216' heads='1'/>
      <alias name='video0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </video>
    <memballoon model='virtio'>
      <alias name='balloon0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
    </memballoon>
  </devices>
  <seclabel type='none'/>
</domain>
```

#Install Gentoo Linux (guest system)
##Create partition
Because I use ``VirtIO`` to create my virtual hard disk, the disk symbol has changed to /dev/vda

fdisk /dev/vda
```
   Device Boot      Start         End      Blocks   Id  System
/dev/vda1   *        2048       43007       20480   83  Linux
/dev/vda2           43008      452607      204800   83  Linux
/dev/vda3          452608    40959999    20253696   83  Linux

```
##Create and mount file system
```
mkfs.ext4 /dev/vda3
mount /dev/vda3 /mnt/gentoo

mkfs.ext4 dev/vda1
mkdir /mnt/gentoo/boot
mount /dev/vda1 /mnt/gentoo

mkswap /dev/vda2
swapon /dev/vda2

cd /mnt/gentoo
tar xjpf stage3-amd64-20120301.tar.bz2

mkdir /mnt/gentoo/usr/portage
mount -t nfs 192.168.2.101:/usr/portage /mnt/gentoo/usr/portage

mount -o rbind /dev /mnt/gentoo/dev
mount -o bind /proc /mnt/gentoo/proc
```

##System configuration
##DNS resolve
```
cp -L /etc/resolve.conf /mnt/gentoo/etc
```
##make.conf
```
CFLAGS="-march=k8 -O2 -pipe -fomit-frame-pointer"
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

echo config_eth0=\"dhcp\" >> /etc/conf.d/net
grep -v rootfs /proc/mounts > /etc/mtab

emerge dhcpcd nfs-utils gentoo-sources

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

#Install Grub2
Because Grub Legacy cannot recognize VirtIO disk, have to install Grub 2.

Currently Grub2 hasn't been mark stable in Gentoo

```
echo "GRUB_PLATFORMS=qemu" >> /etc/make.conf
emerge --autounmask-write =sys-boot/grub-1.99-r2
env-update
emerge unifont
emerge -av =sys-boot/grub-1.99-r2
grub2-mkconfig -o /boot/grub2/grub.cfg
```
Grub 2 required ``unifont`` package, should install it first

TODO

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
