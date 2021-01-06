---
title: "Ubuntu and VLANs"
date: 2021-01-06T10:59:29Z
weight: 1
tags:
- VLAN
- GVRP
- netplan
---

In order to identify how VLANs could be used to provide payload isolation, some experimentation into
how Ubuntu supports VLAN configuration was required.

Configuring a static VLAN is relatively straight-forward with either `netplan` or the `ip` command:

```
$ sudo ip link add link eth0 name vlan2 type vlan id 2
$ sudo ip addr add 10.0.0.1/24 brd 10.0.0.255 dev vlan2
$ sudo ip link set dev vlan2 up
```

This is roughly equivalent to this netplan addition and running `netplan apply`:
```
# /etc/netplan/50-cloud-init.yaml
network:
    ethernets:
        eth0:
            dhcp4: true
            match:
                driver: bcmgenet smsc95xx lan78xx
            optional: true
            set-name: eth0
    version: 2
    vlans:
        vclan2:
            id: 2
            link: eth0
            addresses: [ "10.0.0.1/24" ]
```

The new interface can be inspected with `ip -d addr show vlan2`.

## GVRP

The above setup works, but requires the relevant VLAN id and switch port to be properly configured in the switch itself. The GS510TLP does not allow tagged packets to traverse without the VLAN id being registered. This is presumably so that it can ensure not to forward tagged traffic to any port that does not expect it.

Since the use of VLANs for payload isolation will later require automatic provisioning of new VLANs, a solution had to be found.

As it turns out, the switch and the Linux kernel both support GVRP for dynamic registration of VLANs.

Unfortunately, netplan does not as of yet support configuring this feature.
This means that such a dynamic VLAN can only be set up using the `ip` command:

```
$ sudo ip link add link eth0 name vlan2 type vlan id 3 gvrp on
$ sudo ip addr add 10.1.0.1/24 brd 10.0.0.255 dev vlan3
$ sudo ip link set dev vlan3 up
```

After a couple of seconds, the switch indicates that a new, dynamic VLAN id has been registered and traffic begins to flow between the two Pis.

## Netlink and programmatic VLAN configuration

The `ip` command, part of `iproute2`, uses the new-ish netlink socket to communicate with the Kernel instead of the old `ioctl` method. This means that at least in theory it should be easier to perform the same feat programmatically.

While there are a number of Go libraries for interacting with netlink, none seem to currently support GVRP, so some custom development may be required.