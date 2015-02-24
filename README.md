Debian Jessie LXC / Packer / Vagrant HOWTO
======================================

My take on howto setup Debian Jessie to run vagrant-lxc (keywords: LXC, bridge-utils, dnsmasq, packer-builder-lxc)

## First things first
```
# apt-get install lxc bridge-utils
```

## Network

### Bridged Network device

First we need to setup a bridged network device so the LXC Containers can access the network. There is many ways to do this. I choose to do it like this.


Edit `/etc/network/interfaces` and add the lxcbr0 interface as below:

```
auto lxcbr0
iface lxcbr0 inet static
  address 10.0.0.1
  netmask 255.255.255.0
  pre-up (ip addr show |grep lxcbr0) || brctl addbr lxcbr0
  post-up ethtool -K lxcbr0 tx off

```
The `pre-up` is to create the device if it doesn't exist and the ethtool command in the `post-up` is to turn off checksum offloading on the bridge device (see about dhcp below)

### DHCP / DNS

[2015-02-24] To be continued...
