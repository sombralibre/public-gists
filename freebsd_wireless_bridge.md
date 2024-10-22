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

## Bhyve

First of all was to setting up bhyve environment and install required packages.

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

***/etc/rc.conf***

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

Copy templates.

```sh
# as root
mkdir /zroot/vms/.templates
cp /usr/local/share/examples/vm-bhyve/* /zroot/vms/.templates/
```

Identify the wireless card for passthrough, and save the value the next step.

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
# In this case "3/0/0" is the pci identifier I get for my wireless card.
```
