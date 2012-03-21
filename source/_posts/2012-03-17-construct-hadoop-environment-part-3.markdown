---
layout: post
title: "Construct Hadoop environment<br />Part 3 KVM Guest"
date: 2012-03-17 11:25
comments: true
categories: 
---
Plan to construct a KVM guest as template, then clone this template to create new instance. 

I still use Gentoo Linux as guest system, and install with minimal components. 
In order to run Hadoop, guest machine only need Java environment and SSH support.

My host machine already has portage tree, I could passthrough entire portage partition via [VirtFS](http://www.linux-kvm.org/page/9p_virtio)

# Install libvirt on a hypervisor machine
libvirt is just a front-end of QEMU, but it provide many handy tools to manage the virtual machine.
```
echo "app-emulation/libvirt qemu virt-network -lxc" >> /etc/portage/package.use
emerge app-emulation libvirt
/etc/init.d/libvirtd start
rc-update add libvirtd default
```

# Install management tools on remote client
* app-emulation/virt-manager ~amd64
Use latest one(0.9.1) to support VirtFS configuration options.

There is a command line tools available to connect remote KVM host
```
virsh -c qemu+ssh://root@192.168.2.101/system
```
virt-manager use the same manchanism to connect remote host (via SSH tunnel)

# Domain XML
Create a vm as template, the domain xml is like:
```
<domain type='kvm' id='5'>
  <name>gentoo</name>
  <uuid>e7fdd218-e74d-f821-cc25-6424d4e69fc4</uuid>
  <memory>1048576</memory>
  <currentMemory>1048576</currentMemory>
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
      <source file='/hadoopimages/gentoo.img'/>
      <target dev='vda' bus='virtio'/>
      <alias name='virtio-disk0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/nfs/ubuntu-11.10-desktop-amd64+mac.iso'/>
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
    <filesystem type='mount' accessmode='passthrough'>
      <driver type='path' wrpolicy='immediate'/>
      <source dir='/usr/portage'/>
      <target dir='/hostportage'/>
      <alias name='fs0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
    </filesystem>
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
      <model type='cirrus' vram='9216' heads='1'/>
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
Here I request 1GB RAM during construct Gentoo system. The compilation will consume a lot of RAM, after system installation I could reduce the RAM amount.

#Install Gentoo Linux (guest)
By default, Gentoo installation media kernel do not support `9p` which is the filesystem type VirtFS used. So here I use Ubuntu LiveCD instead.

Run `sudo -i` to grab a root prompt.

##Create partition
Because I choose `VirtIO` as my virtual hard disk driver, the disk label is /dev/vda.

Run `fdisk /dev/vda` to create disk partition.
```
   Device Boot      Start         End      Blocks   Id  System
/dev/vda1   *        2048       22527       10240   83  Linux
/dev/vda2           22528      432127      204800   83  Linux
/dev/vda3          432128    40959999    20206592   83  Linux

```
##Create and mount file system
```
mkfs.ext4 /dev/vda3
mkdir /mnt/gentoo
mount /dev/vda3 /mnt/gentoo

mkfs.ext4 dev/vda1
mkdir /mnt/gentoo/boot
mount /dev/vda1 /mnt/gentoo

mkswap /dev/vda2
swapon /dev/vda2

cd /mnt/gentoo
tar xjpf stage3-amd64-20120301.tar.bz2

mkdir /mnt/gentoo/usr/portage
mount -t 9p -otrans=virtio,version=9p2000.L /hostportage /mnt/gentoo/usr/portage

mount -o bind /proc /mnt/gentoo/proc
mount -o bndd /sys  /mnt/gentoo/sys
mount -o bind /dev  /mnt/gentoo/dev
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

USE="mmx sse sse2 minimal -X -gnome -gtk -ipv6 -kde -qt"

FEATURES="nodoc noinfo noman"
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
UUID=b65f0153-1cbf-4b3c-83f0-967781e62cea /boot ext4 noatime 1 2
UUID=6a2fd05c-058f-4fee-a8b6-8cdef2d9c4ad none  swap sw      0 0
UUID=b3a6d092-1ef2-4855-b296-af56ee653b54 /     ext4 noatime 0 1
```

```
passwd
eselect profile set 7

echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen

cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo "Asia/Shanghai" > /etc/timezone

echo config_eth0=\"dhcp\" >> /etc/conf.d/net

emerge dhcpcd gentoo-sources

rc-update add sshd default

ln -s /etc/init.d/net.lo /etc/init.d/net.eth0
rc-update add net.eth0 default


cd /usr/src/linux
make menuconfig
make install
make modules_install
```
Kernel must contains _virtio_ drivers
```
Processor type and features
  [*] Paravirtualized guest support
    [*] KVM paravirtualized clock
    [*] KVM Guest support
    [*] Enable paravirtualization code
  [*] Allow for memory hot-add
  [*]   Allow for memory hot remove
  [*] Page migration

Bus options
  [*] Message signaled Interrupts (MSI and MSI-X)
  <M> Support for PCI Hotplug

Device Drivers
  [*] Block devices
    <*> Virtio block driver
  [*] Network device support
    <*> Virtio network driver
  Character devices
    <*> Virtio console
  <*> Hardware Random Number Generator Core support
    <*> VirtIO Random number Generator support
  Virtio drivers
    <*> PCI driver for virtio devices
    <*> Virtio balloon driver
    <*> Platform bus driver for memory mapped virtio devices
```

Configure kernel for _virtfs_ support
```
[*] Networking support
  <M> Plan 9 Resource Sharing Support
    <M> 9P Virtio Transport

File systems
  Network File Systems
    <M> Plan 9 Resource Sharing Support
      [*] 9P POSIX Access Control
      [*] Enable 9P client caching support
```

#Install Grub
```
grep -v rootfs /proc/mounts > /etc/mtab
emerge -av grub-static
```
Because Grub Legacy cannot recognize VirtIO disk automatically

Edit /boot/grub/device.map
```
(hd0)   /dev/vda
```
Then install Grub to MBR
```
grub-install --no-floppy /dev/vda
```
Edit /boot/grub/grub.conf
```
default 0
timeout 1

title Gentoo Linux 3.2.1
root (hd0,0)
kernel /boot/vmlinuz-3.2.1-gentoo-r2 root=/dev/vda3

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
