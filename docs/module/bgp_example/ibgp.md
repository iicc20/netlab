# IBGP Data Center Fabric

We want to create a leaf-and-spine fabric running IBGP on top of OSPF. The fabric will have two leafs (l1, l2) and two spines (s1, s2).

All devices run BGP and OSPF (we need OSPF within AS 65000 to propagate loopback interfaces):

```
module: [ bgp,ospf ]
```

Default BGP AS number is 65000. Default OSPF area is 0.0.0.0. Default device type is Cisco Nexus 9300v:

```
bgp:
  as: 65000
ospf:
  area: 0.0.0.0
defaults:
  device: nxos
```

Fabric point-to-point links are unnumbered:

```
addressing:
  p2p:
    unnumbered: true
```

The network topology has four nodes. One of the leafs is an Arista switch.

```
nodes:
  s1:
  s2:
  l1:
  l2:
    device: eos
```

The switches are connected into a leaf-and-spine fabric:

```
links:
- s1-l1
- s1-l2
- s2-l1
- s2-l2
```

We could use a full mesh of IBGP sessions, but it's more interesting to use spine switches as BGP route reflectors. As we don't need any other node-specific attributes, we'll use the global **rr_list** to specify the route reflectors

```
bgp:
  as: 65000
  rr_list: [ s1, s2 ]
```

## Resulting Data Structures

Data structures generated by the BGP data transformation module include the list of BGP neighbor. On a route reflector (S1), all other switches are IBGP neighbors:

```
- bgp:
    as: 65000
    neighbors:
    - as: 65000
      ipv4: 10.0.0.1
      name: l1
      type: ibgp
    - as: 65000
      ipv4: 10.0.0.2
      name: l2
      type: ibgp
    - as: 65000
      ipv4: 10.0.0.4
      name: s2
      rr: true
      type: ibgp
    next_hop_self: true
    rr: true
```

A leaf switch has IBGP sessions with route reflectors (both spine switches):

```
- bgp:
    as: 65000
    neighbors:
    - as: 65000
      ipv4: 10.0.0.3
      name: s1
      rr: true
      type: ibgp
    - as: 65000
      ipv4: 10.0.0.4
      name: s2
      rr: true
      type: ibgp
    next_hop_self: true
```

## Resulting Device Configurations

The above topology generates the following BGP-related device configuration for a NX-OS spine switch (S1):

```
feature bgp
!
router bgp 65000
 address-family ipv4 unicast
!
  network 10.0.0.3/32
!
 neighbor 10.0.0.1 remote-as 65000
  description l1
  update-source loopback0
  address-family ipv4 unicast
   next-hop-self
   route-reflector-client
!
 neighbor 10.0.0.2 remote-as 65000
  description l2
  update-source loopback0
  address-family ipv4 unicast
   next-hop-self
   route-reflector-client
!
 neighbor 10.0.0.4 remote-as 65000
  description s2
  update-source loopback0
  address-family ipv4 unicast
   next-hop-self
```

And this is the BGP configuration for an Arista EOS leaf switch:

```
router bgp 65000
  neighbor 10.0.0.3 remote-as 65000
  neighbor 10.0.0.3 description s1
  neighbor 10.0.0.3 update-source Loopback0
  neighbor 10.0.0.3 next-hop-self
!
  neighbor 10.0.0.4 remote-as 65000
  neighbor 10.0.0.4 description s2
  neighbor 10.0.0.4 update-source Loopback0
  neighbor 10.0.0.4 next-hop-self
!
 address-family ipv4
!
  network 10.0.0.2/32
!
!
  neighbor 10.0.0.3 activate
  neighbor 10.0.0.4 activate
```

## Complete network topology

```
#
# Simple BGP example (see documentation)
#
module: [ bgp,ospf ]

addressing:
  p2p:
    unnumbered: true

bgp:
  as: 65000
  rr_list: [ s1, s2 ]
ospf:
  area: 0.0.0.0
defaults:
  device: nxos

nodes:
  s1:
  l1:
  l2:
    device: eos

links:
- s1-l1
- s1-l2
- s2-l1
- s2-l2
```