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

## LXC

edit `/etc/lxc/default.conf` to contain:

```
lxc.network.type=veth
lxc.network.link=lxcbr0
lxc.network.ipv4 = 0.0.0.0/24
lxc.network.flags=up
lxc.network.mtu=1500
```

You are now ready to try out LXC

```
# lxc-create -n mycontainer -t debian -- -r jessie
# lxc-start -n mycontainer
...
```
Read more about LXC elsewhere

## vagrant-lxc
to use `vagrant-lxc` just install:

```
# apt-get install vagrant
```

and then as your normal user:

```
$ vagrant plugin install vagrant-lxc
```

You might need some exstra libraries to compile this plugin. Now you are ready to control LXC through vagrant. To test with a base box from the cloud, run as a unprivileged user:

```
$ mkdir mytestvagrant
$ vagrant init fgrehm/wheezy64-lxc
$ vagrant up --provider=lxc
....
```
`vagrant-lxc` requires sudo rigts, because user namespaces are not supported yet

## packer-builder-lxc

As of this writing vagrant-lxc doesn't seem to work so well with systemd on Debian Jessie. So if you target that platform, do something else.

Here is what I did to get `packer-builder-lxc` working with latest` packer.

On:

https://github.com/ustream/packer-builder-lxc/releases

there is a binary release of the builder. But this binary is incompatible with the latest release of Packer. So I choose to build my own version. This requires [gox](https://github.com/mitchellh/gox), which is a crossplatform compiler for the Go Language. [gox](https://github.com/mitchellh/gox) can't be installed on the debian packgaged version of go without using sudo, and thus "tainting" the root system. As I don't want that, I choose to fetch my own binary distribution of Go as an unprivileged user:

### Building gox

```
$ wget https://storage.googleapis.com/golang/go1.4.2.linux-amd64.tar.gz
$ mkdir gostuff
$ cd gostuff
$ tar xvzf ../go1.4.2.linux-amd64.tar.gz
$ export GOROOT=$(pwd)/go
$ mkdir path
$ export GOPATH=$(pwd)/path
$ export PATH=$GOROOT/bin:$PATH
$ go get github.com/mitchellh/gox
$ export PATH=$GOPATH/bin:$PATH
```

Now `gox` is installed, before in can be used we need to run:

```
$ gox -build-toolchain
```
### building packer-builder-lxc


Now we need to fetch some dependencies for `packer-builder-lxc`:

```
$ go get github.com/mitchellh/packer/packer/plugin
$ go get github.com/ustream/packer-builder-lxc/builder/lxc
```

The last one will fail with:

```
# github.com/ustream/packer-builder-lxc/builder/lxc
path/src/github.com/ustream/packer-builder-lxc/builder/lxc/builder.go:106: cannot use artifact (type *Artifact) as type packer.Artifact in return argument:
	*Artifact does not implement packer.Artifact (missing State method)
```
because github.com/ustream/packer-builder-lxc/builder/lxc is not ready for the latest version of packer. I solve this by implement the missing State method as they do in:

https://github.com/lmars/packer-post-processor-vagrant-s3/pull/7/files

so fire up your editor on `path/src/github.com/ustream/packer-builder-lxc/builder/lxc/artifact.go` and add the missing method:

```
func (a *Artifact) State(name string) interface{} {
  return nil
}
```

save it, and run:
```
$ go get github.com/ustream/packer-builder-lxc/builder/lxc
```

again. Now we are ready to build the packer builder:

```
$ git clone https://github.com/ustream/packer-builder-lxc
$ cd packer-builder-lxc/
$ gox -os=linux -arch=amd64 -output=pkg/{{.OS}}_{{.Arch}}/packer-builder-lxc
```

### Installing packer-builder-lxc

Deploy the builder to your packer directory by:

```
cp pkg/linux_amd64/packer-builder-lxc PATHTOYOURPACKERINSTALLATION
```

and edit the file `~/.packerconfig` to at least contain something:

```
{
  "builders": {
      "lxc": "PATHTOYOURPACKERINSTALLATION/packer-builder-lxc"
  }
}
```

Now you are ready to toy with the sample configuration in this repository by changing to ` packer directory` and running:

```
packer build jessie.json
```

the packer post processor doesn't support building vagrant boxes out of this yet, so resulting files (`lxc-config ` and `rootfs.tar.gz`) in the `output-lxc` directory need to be packed together with a third file, containing something like the `metadata.json` e.g. by:

```
$ cp metadata.json output-lxc/
$ cd output-lxc
$ tar cvf jessie-lxc.box *
```

Then you have the box

[2015-02-25] To be continued...

## Resources
* https://wiki.debian.org/LXC
* https://wiki.archlinux.org/index.php/Linux_Containers
* https://github.com/ustream/packer-builder-lxc
* https://github.com/fgrehm/vagrant-lxc
* https://github.com/mitchellh/gox
* https://packer.io/
