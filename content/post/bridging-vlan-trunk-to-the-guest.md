+++
title = "Bridging VLAN Trunk to the Guest"
date = "2015-09-07"
slug = "2015/09/07/bridging-vlan-trunk-to-the-guest"
Categories = [ "devops" ]
+++

In this article, we're going to make an entire VLAN trunk on the host accessible to the guest machine. The guest machine can then create VLAN subinterfaces in order to access a particular VLAN.

<!--more-->

Our host and guest machines are running RHEL7. We're using Linux bridges and [libvirt](http://libvirt.org/ "libvirt") for guest and network configuration.

## Bridge configuration on the host

On the host, the physical interface `enp3s0f0` is a trunk interface including VLANs with tags 408, 410 and 412. We'll create a new Linux bridge and add the `enp3s0f0` to this bridge. The virtual machines created by libvirt will also be connected to this bridge. The configuration of the `enp3s0f0` physical interface looks as follows:

{{< highlight-caption lang="shell" linenos="table" title="/etc/sysconfig/network-scripts/ifcfg-enp3s0f0" >}}
TYPE=Ethernet
BOOTPROTO=none
DEVICE=enp3s0f0
ONBOOT=yes
BRIDGE=br-enp3s0f0
{{< / highlight-caption >}}

Please, note that there's no IP address configuration (neither static nor via DHCP) for the `enp3s0f0` interface. The `enp3s0f0` interface is a trunk interface and hence the IP configuration would make no sense here. The `BRIDGE` configuration variable connects the physical interface to the `br-enp3s0f0` bridge. To create the `br-enp3s0f0` bridge the following configuration file is needed:

{{< highlight-caption lang="shell" linenos="table" title="/etc/sysconfig/network-scripts/ifcfg-br-enp3s0f0" >}}
TYPE=Bridge
BOOTPROTO=none
DEVICE=br-enp3s0f0
ONBOOT=yes
DELAY=0
{{< / highlight-caption >}}

After the `enp3s0f0` and `br-enp3s0f0` configuration is in place you might want to restart the networking service using the command:

{{< highlight shell "linenos=table" >}}
$ sudo systemctl restart network
{{< / highlight >}}

## Creating a bridged network in libvirt

Next, we're going to tell libvirt that there's an existing bridge `br-enp3s0f0` we'd like our virtual machines be connected to. First, let's create a libvirt network definition file named just `bridge.xml`:

{{< highlight-caption lang="xml" linenos="table" title="bridge.xml" >}}
<network>
  <name>br-enp3s0f0</name>
  <forward mode='bridge'/>
  <bridge name='br-enp3s0f0' />
</network>
{{< / highlight-caption >}}

To create a libvirt network based on the above definition, type:

{{< highlight shell "linenos=table" >}}
$ sudo virsh net-define bridge.xml
{{< / highlight >}}

We'd like libvirt daemon to start the network automatically on the startup:

{{< highlight shell "linenos=table" >}}
$ sudo virsh net-autostart br-enp3s0f0
{{< / highlight >}}

For the first time, we have to start the `br-enp3s0f0` network manually:

{{< highlight shell "linenos=table" >}}
$ sudo virsh net-start br-enp3s0f0
{{< / highlight >}}

If the above configuration went well, you will find the new network `br-enp3s0f0` on the list of libvirt networks:
{{< highlight shell "linenos=table" >}}
$ virsh net-list
 Name                 State      Autostart     Persistent
----------------------------------------------------------
 br-enp3s0f0          active     yes           yes
{{< / highlight >}}

## Attaching a guest to the network

When creating a new guest (domain) in libvirt, you will need to attach the domain to the `br-enp3s0f0` network. I'm not going to present the complete domain XML configuration here. You should include the following snippet in your domain definition in order to connect the domain to the `br-enp3s0f0` network:

{{< highlight xml "linenos=table" >}}
<interface type='network'>
  <source network='br-enp3s0f0'/>
  <forward mode='route'/>
  <model type='virtio'/>
</interface>
{{< / highlight >}}

## Guest network configuration

After the guest machine boots up successfully, you can create VLAN subinterfaces in order to obtain access to the individual VLANs within the guest. First, let's check the configuration of the VLAN trunk interface `eth0` inside the guest:

{{< highlight-caption lang="shell" linenos="table" title="/etc/sysconfig/network-scripts/ifcfg-eth0" >}}
TYPE=Ethernet
BOOTPROTO=none
DEVICE=eth0
ONBOOT=yes
{{< / highlight-caption >}}

Finally, we can create VLAN subinterfaces to access individual VLANs available in the `eth0` trunk. For example, to access VLAN 408 and obtain the IP configuration via DHCP you can create a new cofiguration file `ifcfg-eth0.408`:

{{< highlight-caption lang="shell" linenos="table" title="/etc/sysconfig/network-scripts/ifcfg-eth0.408" >}}
BOOTPROTO=dhcp
DEVICE=eth0.408
ONBOOT=yes
VLAN=yes
{{< / highlight-caption >}}

When you restart the networking service, your guest should successfully obtain an IP address on the VLAN 408:

{{< highlight shell "linenos=table" >}}
$ sudo systemctl restart network
{{< / highlight >}}

## Caveat

When experimenting with the Linux bridge configuration I made this observation: *If there's a VLAN subinterface defined for a specific VLAN on the host machine, this specific VLAN won't be accessible inside the guest.* For example, when I created the following VLAN 408 subinterface on the host:

{{< highlight-caption lang="shell" linenos="table" title="/etc/sysconfig/network-scripts/ifcfg-enp3s0f0.408" >}}
BOOTPROTO=none
DEVICE=enp3s0f0.408
ONBOOT=yes
VLAN=yes
{{< / highlight-caption >}}

As soon as I brought this interface up using:

{{< highlight shell "linenos=table" >}}
$ sudo ifup enp3s0f0.408
{{< / highlight >}}

the `eth0.408` VLAN subinterface in the guest stopped working.

## References

When writing this blogpost I referred to the very useful article [KVM & BRCTL in Linux – bringing VLANs to the guests](http://blog.davidvassallo.me/2012/05/05/kvm-brctl-in-linux-bringing-vlans-to-the-guests/) describing the issues of VLAN bridging in a great detail.
