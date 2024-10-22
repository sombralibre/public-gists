# FreeBSD Wireless bridge + bhyve + Alpine Linux.

## Brief 

I always liked FreeBSD because its stability, some servers I've launched years 
ago are still up and working; however I feel like the desktop experience 
cannot satisfy my requirements to be used as a daily driver.

The main stopper to me, is the low wifi performance with the actual available 
drivers; I've been twerking lots of kernel options and others configs, but at 
the end gave up.

In some forum I heard about wifibox, although a workaround already exists, I 
wanted to research and do it by myself, before give it a try or read about it.

Back in the 2010, I had to virtualize a PBX server with asterisk over Centos 
using KVM, I remember having to deal with AMD IOMMU and Intel VT-d, for 
passthrough several E1/T1 pci cards, so that give me an idea about a possible 
way to overcome the slow wifi speed.

My first step was to find a virtualization technology available for FreeBSD 
that supported pci passthrough, It wasn't difficult and my first though was 
Bhyve; I'd already used it in the past but never used these passthrough 
features.

Actually my approach was kinda simple, virtualize a lightweight linux system 
with good support for hardware, then create a network bridge between wireless 
and ethernet interface, so the wireless interface handled the authentication 
stuff and established the layer 1 link, and then layers 2-3 were shared through
bridge. In the host side a tap interface would suffice the network connections 
needs.

## Bhyve Environment

First of all was to setting up bhyve and install required packages.

Installation of required packages.
```sh
# as root
pkg install bhyve-firmware vm-bhyve.
```

Creation and mount of dedicated zfs dataset.

```sh
zfs create zroot/vms
zfs mount
```

Add startup configs for vm-bhyve.

At the end of ***/etc/rc.conf***

```conf
...
vm_enable="YES"
vm_dir="zfs:zroot/vms"
```

vm-bhyve initialization

```sh
# as root
vm init
```

Creation of a switch for bhyve networking, without any Host interface.
```sh
# as root
vm switch create private
```

Tap interface creation.

```sh
# as root
ifconfig tap0 create
```

The following lines ensure tap interface creating at boot:
At the end of ***/etc/rc.conf***
```conf
..
cloned_interfaces="tap0 up"
ifconfig_tap0="DHCP"
```

Copy templates.

```sh
# as root
mkdir /zroot/vms/.templates
cp /usr/local/share/examples/vm-bhyve/* /zroot/vms/.templates/
```

Download installation iso, I've used alpine linux **standard**. I tried with 
**virt** edition before but wireless card doesn't get recognized.
```sh
# as root
vm iso https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-standard-3.20.3-x86_64.iso
```

## Kernel configs

Identify the wireless card pci id for passthrough, and save the value the next 
step.
```sh
pciconf -l|awk '
($0 ~ "class=0x028000")
{split($0,a, ":");print a[2] "/" a[3] "/" a[4]}
'
```

Added these lines to ***/boot/loader.conf*** 

```conf
...
hw.vtnet.csum_disable="1"
vmm_load="YES"
nmdm_load="YES"
if_tap_load="YES"
if_brigde_load="YES"
# The following line value is the returned one from the previous command.
pptdevs="3/0/0"
# In my case "3/0/0" is the pci identifier I get for my wireless card.
```

Creation of installation template for our wifibox.
***/zroot/vms/.templates/wifibox.conf***
```conf
guest="linux"
loader="uefi"
cpu=1
memory=256M
network0_type="virtio-net"
network0_switch="private"
network0_device="tap0"
network0_name="eth0"
disk0_type="virtio-blk"
disk0_name="disk0.img"
# the same pci id as the step before, 3/0/0 for my own card.
passthru0="3/0/0"
```
Reboot the laptop:
```sh
# as root
reboot # or whatever you use to reboot
```

## Alpine installation.

Once the laptop starts over, let check passthrough is working before proceed to 
to installation.

```sh
# as root
vm pass|grep -i yes
```
Output must looks something like this:
```sh
ppt0       3/0/0        Yes          Wireless 7265
```
If so, run the bhyve installation command for the virtual machine.
```sh
# as root
vm create -t wifibox wifiman
# Yes I call my virtual machine "wifiman", I'm bad naming things already know it
vm install -f wifiman alpine-standard-3.20.3-x86_64.iso
```
I'm not gonna go deep down into Alpine installation process, since it's very
simple actually. Just to point out, I've configure the wlan0 interface in the 
during installation process with my SSID, eth0 left unconfigured, added a new 
user, and use "sys" as a installation method. Once finished just run `poweroff`

An advice is to set new user and root a simple password, at least while finish 
the configurations.

Now start and connect to the virtual machine
```sh
# as root
vm start wifiman
vm console wifiman
```

Now in the virtual machine
```sh
# as root
# in the alpine virtual machine
apk update
apk add bridge-utils
```
Add the following network configurations:
***/etc/network/interfaces*** of the virtual machine
```conf
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet manual
    down brctl delif br0 eth0 || true
    allow-hotplug eth0

auto wlan0
pre-up iw dev wlan0 set 4addr on
    down brctl delif br0 wlan0 || true
    allow-hotplug wlan0
    iface wlan0 inet manual

auto br0
iface br0 inet dhcp # this is optional, can be manual as well
    pre-up echo 1 > /proc/sys/net/ipv4/ip_forward
    pre-up brctl addbr br0 || true
    pre-up brctl addif br0 eth0 || true
    pre-up brctl addif br0 wlan0 || true
    post-down brctl delbr br0
    bridge_ports eth0 wlan0
```
Then just reboot the virtual machine networking service
```sh
service networking restart
```

Now in the Host run 
```sh
# as root
dhclient tap0
# or
service netif restart
```

## Speed test
A little speed ![test](./freebsd_speedtest.gif) 
