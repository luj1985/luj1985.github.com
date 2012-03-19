---
layout: post
title: "Construct Hadoop Environment<br />Part 2 KVM Host"
date: 2012-03-14 01:03
comments: true
categories: 
---
KVM has two components

* A device driver for managing the virtulization hardware; this driver exposes its capabilities via a character device ``/dev/kvm``
* A user-space component for emulating PC hardware; this is a lightly modified ``QEMU`` process.

The modified ``QEMU`` process ``mmap()`` the guest's physical memory and calls the kernel mode driver to execute in guest mode.

The I/O model is directly derived from ``QEMU``'s, with support for copy-on-write disk images and other ``QEMU`` features.

#Prerequisites
I need remote connect to KVM guest, and all KVM guest should be able to communicate with each other, so I use bridge networking.

KVM use a modified QEMU to emulate the system, I could follow the QEMU way to configure the network.

##Hardware 
CPU has to support full virtualization. Intel CPUs need to have the ``vmx`` capability, AMD CPUs require ``svm``: 
```
grep '(vmx|svm)' /proc/cpuinfo
```
* svm = secure virtual machine (AMD)
* vmx = virtual machine extensions (Intel) 
My CPU is AMD based, the instruction set ``svm`` found

##Kernel
Kernel configuration
```
Processor type and features
  [*] Enable KSM for page merging

[*] Networking support
  Networking options
    <M> 802.1d Ethernet Bridging
    <M> 802.1Q VLAN Support

Device Drivers
[*] Network device support
  [*] Network core driver support
    <M> Universal TUN/TAP device driver support

[*] Virtualization
  <M> Kernel-based Virtual Machine (KVM) support
  < >   KVM for Intel processors support
  <M>   KVM for AMD processors support
  <M> Host kernel accelerator for virtio net
```
/etc/conf.d/modules
```
modules="kvm-amd bridge 8021q tun vhost-net"
```
##Packages
```
app-emulation/qemu-kvm qemu-ifup
net-misc/bridge-utils
```

#Configuration
Direct bridging (guests use an IP address on the same subnet as the host)
{% blockquote KVM http://en.gentoo-wiki.com/wiki/KVM %}
The most transparent option to allow your guests access to the internet is the "virtual hub". In this scheme, the bridge connects eth0 and your tuntap interfaces together, routing packets as if it were a real "old fashioned" hub (not a switch). The key to this approach is to make sure you have unique mac addresses on both the host's tuntap interface as well as the guest. 
{% endblockquote %}

Because I use QEMU Tap networking backend, it require to invoke QEMU as root (http://wiki.qemu.org/Documentation/Networking#Tap)

##init.d
```
ln -s /etc/init.d/net.lo /etc/init.d/net.br0
rc-update add net.br0 default
```

##/etc/conf.d/net
```
bridge_br0="eth0"
config_eth0="null"

brctl_br0="setfd 0 sethello 1 stp off"
config_br0="dhcp"

depend_br0() {
    need net.eth0
}
```
##Create script to manage network device
I enabled ``qemu-ifup`` USE flag of app-emulation/qemu-kvm package. But the generated script were under ``/etc/qemu/`` which is not the default location qemu-kvm use. 

And the program path in script ``/etc/qemu/qemu-ifup`` is hard coded and  is not suite for Gentoo system. So I created the following two scripts instead.
###/etc/qemu-ifup
```
#!/bin/sh

switch=$(/sbin/ip route list | awk '/^default / { print $5 }')
/sbin/ifconfig $1 0.0.0.0 up
/sbin/brctl addif ${switch} $1
```

###/etc/qemu-ifdown
```
#!/bin/sh

switch=$(/sbin/ip route list | awk '/^default / { print $5 }')
/sbin/ifconfig $1 0.0.0.0 down
/sbin/brctl delif ${switch} $1
```

###Grant permission
```
chmod a+x /etc/qemu-if*
```

##Make configuration take effect
```
modprobe kvm-amd bridge 8021q tun vhost-net
/etc/init.d/net.eth0 restart
/etc/init.d/net.br0 start
```
Or simply reboot the system.

#Test KVM
##Create guest machine
Login as root and run following command to create guest machine.
```
qemu-img create -f qcow2 -o preallocation=metadata gentoo.img 10G
qemu-kvm -hda gentoo.img -cdrom install-amd64-minimal-20120223.iso -net nic,mac=52.54:00:12:34:56,vlan=0 -net tap,vlan=0 -boot d -vnc :0
```
Use option ``-vnc :0`` to open a VNC port, then I could use ``vncviewer`` to connect it remotely.
