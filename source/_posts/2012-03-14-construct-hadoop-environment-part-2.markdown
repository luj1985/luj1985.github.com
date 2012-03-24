---
layout: post
title: "Construct Hadoop Environment<br />Part 2 KVM Host"
date: 2012-03-14 01:03
comments: true
categories: 
---
KVM has two components

* A device driver for managing the virtulization hardware; this driver exposes its capabilities via a character device _/dev/kvm_
* A user-space component for emulating PC hardware; this is a lightly modified _QEMU_ process.

The modified _QEMU_ process _mmap()_ the guest's physical memory and calls the kernel mode driver to execute in guest mode.

The I/O model is directly derived from _QEMU_'s, with support for copy-on-write disk images and other _QEMU_ features.
But use _QEMU_ to do full virtualization will reduce the performance, so KVM introduce VirtIO to perform paravirtualization, 
notice guest machine that it was running in a virtual environment.

[Here](http://www.ibm.com/developerworks/linux/library/l-virtio/) is a very good document described the virtio.

# KVM based hypervisor configuration
## Hardware 
CPU must have virtualization instruction set support
```
grep '(vmx|svm)' /proc/cpuinfo
```
* svm = secure virtual machine (AMD)
* vmx = virtual machine extensions (Intel) 

My CPU is AMD based, the instruction set `svm` was found.

## KVM host(hypervisor) configuration
Tune kernel to support KVM, and enable network protocol to support bridge.

### Kernel configuration (with paravirtualization support)
[Here](http://www.linux-kvm.org/page/Tuning_Kernel#Kernel_for_host) is a official document about tuning the kernel of KVM host
```
General setup
  [*] Control Group support

Processor type and features
  [*] High Resolution Timer Support
  [*] Allow for memory compaction
  [*] Page migration
  [*] Enable KSM for page merging
  [*] transparent Hugepage Support

Device Drivers
  Character devices
    [*] HPET - High Precision Event Timer

[*] Virtualization
  <M> Kernel-based Virtual Machine (KVM) support
  < >   KVM for Intel processors support
  <M>   KVM for AMD processors support
  <M> Host kernel accelerator for virtio net
```
### Network bridge support
I want to connect to KVM guest remotely, need enable bridge networking.

To enable network bridge need following kernel options.
```
[*] Networking support
  Networking options
    <M> 802.1d Ethernet Bridging
    <M> 802.1Q VLAN Support

Device Drivers
[*] Network device support
  [*] Network core driver support
    <M> Universal TUN/TAP device driver support
```

### /etc/conf.d/modules
Create config file to load kernel module
```
modules="kvm-amd bridge 8021q tun vhost-net"
```

## User space packages
emerge following user space tools
```
app-emulation/qemu-kvm qemu-ifup xattr tls
net-misc/bridge-utils
```

# Bridge network configuration
Direct bridging (guests use an IP address on the same subnet as the host)
{% blockquote KVM http://en.gentoo-wiki.com/wiki/KVM %}
The most transparent option to allow your guests access to the internet is the "virtual hub". In this scheme, the bridge connects eth0 and your tuntap interfaces together, routing packets as if it were a real "old fashioned" hub (not a switch). The key to this approach is to make sure you have unique mac addresses on both the host's tuntap interface as well as the guest. 
{% endblockquote %}

I use QEMU Tap networking backend, [it require to invoke QEMU as root](http://wiki.qemu.org/Documentation/Networking#Tap)

## /etc/init.d
Create network device
```
ln -s /etc/init.d/net.lo /etc/init.d/net.br0
rc-update add net.br0 default
```

## /etc/conf.d/net
Network startup options
```
bridge_br0="eth0"
config_eth0="null"

brctl_br0="setfd 0 sethello 1 stp off"
config_br0="dhcp"

depend_br0() {
    need net.eth0
}
```

## Create script to manage network device
I enabled `qemu-ifup` USE flag of app-emulation/qemu-kvm package. 
But the script were under _/etc/qemu/_ which is not the default location _qemu-kvm_ use. 

And the program path in script _/etc/qemu/qemu-ifup_ is hard coded and is not suite for Gentoo system. 
So I created the following two scripts instead, and grant executable permission.

### /etc/qemu-ifup
```
#!/bin/sh

switch=$(/sbin/ip route list | awk '/^default / { print $5 }')
/sbin/ifconfig $1 0.0.0.0 up
/sbin/brctl addif ${switch} $1
```

### /etc/qemu-ifdown
```
#!/bin/sh

switch=$(/sbin/ip route list | awk '/^default / { print $5 }')
/sbin/ifconfig $1 0.0.0.0 down
/sbin/brctl delif ${switch} $1
```

## Make configuration take effect
```
modprobe kvm-amd bridge 8021q tun vhost-net
/etc/init.d/net.eth0 restart
/etc/init.d/net.br0 start
```
Or simply reboot the system.

# Test KVM guest machine
Login hypervisor as root and run following command to create guest machine.
```
qemu-img create -f qcow2 -o preallocation=metadata gentoo.img 10G
qemu-kvm -hda gentoo.img -cdrom install-amd64-minimal-20120223.iso \
         -net nic,mac=00:00:00:00:00:01,vlan=0 -net tap,vlan=0 -boot d -vnc :0
```
Use option `-vnc :0` to open a VNC port, then I could use `vncviewer` to connect it remotely. 
IP address of my KVM host server is 192.168.2.101, then use following command to connect VNC
```
vncviewer 192.168.2.101:0
```

After VNC connected, active network card in guest machine, start SSH server. 
```
ifconfig eth0 up
dhcpcd eth0
passwd
/etc/init.d/sshd start
```

The guest machine gains IP address 192.168.2.103. Then it should be able to access from outside world.
```
ssh root@192.168.2.103
```
