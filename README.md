Debian Jessie LXC / Packer / Vagrant HOWTO
======================================

My take on howto setup Debian Jessie to run vagrant-lxc (keywords: LXC, bridge-utils, dnsmasq, packer-builder-lxc)

## First things first
```
# apt-get install lxc bridge-utils
```

## Network

### Bridged Network device

Debian Jessie doesn't provide an out-of-the-box network for LXC containers, so first we need to setup a bridged network device so the LXC Containers can access the network. There is many ways to do this. I choose to do it like this.


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

### IP Masquerading and Forwarding

If you are not running any type of iptables firewall on your host, the manual way of allowing traffic to flow through your physical nic (in my case `wlan0`) is something like:

```
# echo 1 > /proc/sys/net/ipv4/ip_forward
# iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o wlan0 -j MASQUERADE
```

I use shorewall for my personal firewall, and choose to use this to handle forwarding/masquarading. With a standard two-interface configuration modified slightly. See config in the repository above

The rules files has two lines:
```
ACCEPT          loc             $FW
ACCEPT          $FW             loc
```

which gives the LXC containers full network access to the host, and the other way around. This is to allow DHCPREQUEST/DHCPOFFER traffic between the container and the host.


### DHCP / DNS

I use `dnsmasq` as DNS and DHCP for the containers. Install it by:
```
# apt-get install dnsmasq
```
Then edit `/etc/dnsmasq.conf` with only two lines enabled:

```
interface=lxcbr0
dhcp-range=10.0.0.100,10.0.0.150,255.255.255.0,12h
```

Then it will server `10.0.0.100-10.0.0.150` addresses to the containers. As the `lxcbr0` interface is not a psysical network interface it will not do package checksums, which is needed for the DHCP process. Thus we need the `ethtool` command above.

[2015-02-24] To be continued...
