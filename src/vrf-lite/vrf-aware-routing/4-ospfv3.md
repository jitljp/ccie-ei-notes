# VRF-Aware OSPFv3

OSPFv3 can run inside a VRF.

OSPFv3 is different from OSPFv2 in how you configure VRFs:
- OSPFv2 uses a separate process per VRF.
- OSPFv3 uses address families under an OSPFv3 process.

```text
OSPFv2:
router ospf 10 vrf CUSTOMER_A
router ospf 20 vrf CUSTOMER_B

OSPFv3:
router ospfv3 1
 address-family ipv4 unicast vrf CUSTOMER_A
 address-family ipv4 unicast vrf CUSTOMER_B
```

OSPFv3 can support IPv6 and IPv4 address families.

In this page, OSPFv3 is used for IPv4 routing inside VRFs.

## Main Idea

A VRF has its own routing table.

An OSPFv3 address family associated with a VRF installs routes into that VRF routing table.

Example:

```text
router ospfv3 1
 address-family ipv4 unicast vrf CUSTOMER_A
```

This creates an OSPFv3 IPv4 unicast address family for `CUSTOMER_A`.

Routes learned in this address family are installed into the `CUSTOMER_A` IPv4 routing table.

They are not installed into the global routing table.

They are not installed into `CUSTOMER_B`.

Verify with:

```text
show ip route vrf CUSTOMER_A ospf
```

## OSPFv3 Uses Address Families

This is the key point.

OSPFv3 can use one OSPFv3 process with multiple VRF address families.

```text
router ospfv3 1
 address-family ipv4 unicast vrf CUSTOMER_A
 address-family ipv4 unicast vrf CUSTOMER_B
```

Each address family has its own routing context.

```text
OSPFv3 process 1
├── IPv4 unicast VRF CUSTOMER_A
└── IPv4 unicast VRF CUSTOMER_B
```

The process ID is the same.

The VRF address-family context is different.

This is similar to EIGRP named mode and BGP.

```text
EIGRP named mode:
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
 address-family ipv4 unicast vrf CUSTOMER_B autonomous-system 100

BGP:
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
 address-family ipv4 vrf CUSTOMER_B
```

## Lab Topology

This page uses the same overlapping-address topology as the OSPFv2 and static routing pages.

```text
                    R1
        +------------------------------+
        |L0: 1.1.1.1/24  L1: 1.1.1.1/24|
        |   E0/0           E0/1        |
        | CUSTOMER_A     CUSTOMER_B    |
        | 10.10.10.1/24  10.10.10.1/24 |
        +----+----------------+--------+
             |                |
             |                |
       10.10.10.2/24    10.10.10.2/24
            CE1              CE2
             |                |
        192.168.1.0/24   192.168.1.0/24
```

R1 has two VRFs.

```text
CUSTOMER_A
CUSTOMER_B
```

R1 uses the same IP address on both customer-facing interfaces.

```text
E0/0 in CUSTOMER_A: 10.10.10.1/24
E0/1 in CUSTOMER_B: 10.10.10.1/24
```

R1 also uses the same loopback address in both VRFs.

```text
Loopback0 in CUSTOMER_A: 1.1.1.1/24
Loopback1 in CUSTOMER_B: 1.1.1.1/24
```

CE1 and CE2 use the same IP address on their R1-facing interfaces.

```text
CE1: 10.10.10.2/24
CE2: 10.10.10.2/24
```

Both customer sites use the same LAN prefix.

```text
CE1 LAN: 192.168.1.0/24
CE2 LAN: 192.168.1.0/24
```

This would not be possible in a single routing table on R1.

It works because `CUSTOMER_A` and `CUSTOMER_B` are separate routing contexts.

## OSPFv3 IPv4 Address Family Note

OSPFv3 can route IPv4 prefixes.

However, OSPFv3 still uses IPv6 link-local transport for neighbor communication.

On IOS XE, enable IPv6 processing on the interfaces that run OSPFv3.

```text
ipv6 unicast-routing

interface Ethernet0/0
 ipv6 enable
 ospfv3 1 ipv4 area 0
```

This does not mean the routers exchange IPv6 routes.

It allows the interface to have an IPv6 link-local address so OSPFv3 can operate.

The address family controls which routes are carried.

```text
ospfv3 1 ipv4 area 0
```

This enables the IPv4 unicast address family on the interface.

## R1 VRF Configuration

Enable IPv6 routing globally.

```text
R1(config)#ipv6 unicast-routing
```

Create the VRFs and enable the IPv4 and IPv6 address families:

```text
R1(config)#vrf definition CUSTOMER_A
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# address-family ipv6

R1(config)#vrf definition CUSTOMER_B
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# address-family ipv6
```

Assign the physical interfaces to the VRFs.

```text
R1(config)# interface Ethernet0/0
R1(config-if)# vrf forwarding CUSTOMER_A
R1(config-if)# ip address 10.10.10.1 255.255.255.0
R1(config-if)# ipv6 enable
R1(config-if)# no shutdown

R1(config)# interface Ethernet0/1
R1(config-if)# vrf forwarding CUSTOMER_B
R1(config-if)# ip address 10.10.10.1 255.255.255.0
R1(config-if)# ipv6 enable
R1(config-if)# no shutdown
```

Assign the loopbacks to the VRFs.

```text
R1(config)# interface Loopback0
R1(config-if)# vrf forwarding CUSTOMER_A
R1(config-if)# ip address 1.1.1.1 255.255.255.0
R1(config-if)# ipv6 enable

R1(config)# interface Loopback1
R1(config-if)# vrf forwarding CUSTOMER_B
R1(config-if)# ip address 1.1.1.1 255.255.255.0
R1(config-if)# ipv6 enable
```

The same IP address can be used in both VRFs.

Verify the VRF interfaces.

```text
R1# show vrf
  Name                             Default RD            Protocols   Interfaces
  CUSTOMER_A                       <not set>             ipv4,ipv6   Et0/0
                                                                     Lo0
  CUSTOMER_B                       <not set>             ipv4,ipv6   Et0/1
                                                                     Lo1
```

## CE Configuration

The CEs are not using VRFs.

Enable IPv6 routing globally.

```text
CE1(config)# ipv6 unicast-routing
CE2(config)# ipv6 unicast-routing
```

CE1:

```text
CE1(config)# interface Ethernet0/0
CE1(config-if)# ip address 10.10.10.2 255.255.255.0
CE1(config-if)# ipv6 enable
CE1(config-if)# no shutdown

CE1(config)# interface Loopback0
CE1(config-if)# ip address 192.168.1.1 255.255.255.0
CE1(config-if)# ipv6 enable
```

CE2:

```text
CE2(config)# interface Ethernet0/0
CE2(config-if)# ip address 10.10.10.2 255.255.255.0
CE2(config-if)# ipv6 enable
CE2(config-if)# no shutdown

CE2(config)# interface Loopback0
CE2(config-if)# ip address 192.168.1.1 255.255.255.0
CE2(config-if)# ipv6 enable
```

## R1 OSPFv3 Configuration

Configure one OSPFv3 process.

```text
R1(config)# router ospfv3 1
```

Configure the IPv4 unicast address family for `CUSTOMER_A`.

```text
R1(config-router)# address-family ipv4 unicast vrf CUSTOMER_A
R1(config-router-af)# router-id 1.1.1.1
R1(config-router-af)# capability vrf-lite
```

Configure the IPv4 unicast address family for `CUSTOMER_B`.

```text
R1(config-router)# address-family ipv4 unicast vrf CUSTOMER_B
R1(config-router-af)# router-id 1.1.1.1
R1(config-router-af)# capability vrf-lite
R1(config-router-af)# exit-address-family
```

The same OSPFv3 process supports both VRFs.

```text
router ospfv3 1
 address-family ipv4 unicast vrf CUSTOMER_A
 address-family ipv4 unicast vrf CUSTOMER_B
```

The two address families use the same router ID in this lab.

That is acceptable here because they are separate VRF OSPFv3 address-family instances.

In production, unique router IDs are often easier to troubleshoot.

Example:

```text
router ospfv3 1
 address-family ipv4 unicast vrf CUSTOMER_A
  router-id 1.1.1.10
 address-family ipv4 unicast vrf CUSTOMER_B
  router-id 1.1.1.20
```

## Enable OSPFv3 on R1 Interfaces

OSPFv3 is enabled directly on interfaces.

There is no OSPFv2-style `network` command.

Enable OSPFv3 IPv4 on the `CUSTOMER_A` interfaces.

```text
R1(config)#interface Ethernet0/0
R1(config-if)#ospfv3 1 ipv4 area 0

R1(config)#interface Loopback0
R1(config-if)#ospfv3 1 ipv4 area 0
```

Enable OSPFv3 IPv4 on the `CUSTOMER_B` interfaces.

```text
R1(config)#interface Ethernet0/1
R1(config-if)#ospfv3 1 ipv4 area 0

R1(config)#interface Loopback1
R1(config-if)#ospfv3 1 ipv4 area 0
```

The interface VRF membership decides which OSPFv3 VRF address family the interface belongs to.

```text
Ethernet0/0 -> CUSTOMER_A -> OSPFv3 process 1 IPv4 VRF CUSTOMER_A
Ethernet0/1 -> CUSTOMER_B -> OSPFv3 process 1 IPv4 VRF CUSTOMER_B
```

## CE OSPFv3 Configuration

The CE routers use normal OSPFv3 IPv4 address-family configuration.

CE1:

```text
CE1(config)# router ospfv3 1
CE1(config-router)# address-family ipv4 unicast
CE1(config-router-af)# router-id 192.168.1.1
```

CE2:

```text
CE2(config)# router ospfv3 1
CE2(config-router)# address-family ipv4 unicast
CE2(config-router-af)# router-id 192.168.1.1
```

Enable OSPFv3 on CE1 interfaces.

```text
CE1(config)# interface Ethernet0/0
CE1(config-if)# ospfv3 1 ipv4 area 0

CE1(config)#interface Loopback0
CE1(config-if)# ospfv3 1 ipv4 area 0
```

Enable OSPFv3 on CE2 interfaces.

```text
CE2(config)# interface Ethernet0/0
CE2(config-if)# ospfv3 1 ipv4 area 0

CE2(config)# interface Loopback0
CE2(config-if)# ospfv3 1 ipv4 area 0
```

## OSPFv3 Neighbors

R1 should form one OSPFv3 neighbor in each VRF.

```text
R1#show ospfv3 vrf CUSTOMER_A neighbor

          OSPFv3 1 address-family ipv4 vrf CUSTOMER_A (router-id 1.1.1.1)

Neighbor ID     Pri   State           Dead Time   Interface ID    Interface
192.168.1.1       1   FULL/BDR        00:00:38    1               Ethernet0/0
```

```text
R1#show ospfv3 vrf CUSTOMER_B neighbor

          OSPFv3 1 address-family ipv4 vrf CUSTOMER_B (router-id 1.1.1.1)

Neighbor ID     Pri   State           Dead Time   Interface ID    Interface
192.168.1.1       1   FULL/BDR        00:00:39    1               Ethernet0/1
```

```text
CUSTOMER_A adjacency:
OSPFv3 process 1
IPv4 unicast VRF CUSTOMER_A
Interface Ethernet0/0

CUSTOMER_B adjacency:
OSPFv3 process 1
IPv4 unicast VRF CUSTOMER_B
Interface Ethernet0/1
```

## OSPFv3 Routes in Each VRF

R1 learns CE1's LAN in `CUSTOMER_A`.

```text
R1# show ip route vrf CUSTOMER_A ospfv3

O        192.168.1.1 [110/10] via 10.10.10.2, 00:06:08, Ethernet0/0
```

R1 learns CE2's LAN in `CUSTOMER_B`.

```text
R1# show ip route vrf CUSTOMER_B ospfv3

O        192.168.1.1 [110/10] via 10.10.10.2, 00:06:07, Ethernet0/1
```

The routes look the same, but they are in different VRF routing tables.

```text
CUSTOMER_A:
192.168.1.0/24 via 10.10.10.2, Ethernet0/0

CUSTOMER_B:
192.168.1.0/24 via 10.10.10.2, Ethernet0/1
```

`CUSTOMER_A` does not learn `CUSTOMER_B` routes.

`CUSTOMER_B` does not learn `CUSTOMER_A` routes.

OSPFv3 is VRF-aware, but it does not leak routes between VRFs.

## CE Routes to R1 Loopbacks

CE1 learns R1's `CUSTOMER_A` loopback.

```text
CE1# show ip route ospfv3

O        1.1.1.1 [110/10] via 10.10.10.1, 00:06:54, Ethernet0/0
```

CE2 learns R1's `CUSTOMER_B` loopback.

```text
CE2# show ip route ospfv3

O        1.1.1.1 [110/10] via 10.10.10.1, 00:06:53, Ethernet0/0
```

The routes look identical on the CE routers.

But they point to different VRF instances on R1.

```text
CE1 -> 1.1.1.1 reaches R1 Loopback0 in CUSTOMER_A
CE2 -> 1.1.1.1 reaches R1 Loopback1 in CUSTOMER_B
```

The destination IP address is the same.

The routing instance on R1 is different.

## OSPFv3 Does Not Use Network Commands

OSPFv2 commonly uses `network` commands.

```text
router ospf 10 vrf CUSTOMER_A
 network 10.10.10.0 0.0.0.255 area 0
```

OSPFv3 is enabled directly on the interface.

The OSPFv3 process and address family exist under router configuration mode.

The interface activation happens under interface configuration mode.

## Why capability vrf-lite Is Used

In a VRF-lite design, the router is using VRFs without an MPLS L3VPN core.

When OSPFv3 runs inside a VRF, IOS XE can apply PE-CE behavior used in MPLS L3VPN designs.

That behavior is useful for MPLS L3VPN PE-CE OSPFv3.

It can be undesirable in plain VRF-lite.

Use `capability vrf-lite` under the OSPFv3 VRF address family.

```text
router ospfv3 1
 address-family ipv4 unicast vrf CUSTOMER_A
  capability vrf-lite
 !
 address-family ipv4 unicast vrf CUSTOMER_B
  capability vrf-lite
```

Before:

```
R1# show ospfv3 vrf CUSTOMER_A
 OSPFv3 1 address-family ipv4 vrf CUSTOMER_A
 Router ID 1.1.1.1
 Supports NSSA (compatible with RFC 3101)
 Supports Database Exchange Summary List Optimization (RFC 5243)
 Domain ID (none)
 Connected to MPLS VPN Superbackbone
 Maximum number of non self-generated LSA allowed 50000
 ...
 ```

 It states `Connected to MPLS VPN Superbackbone`.

 After:

 ```
R1# show ospfv3 vrf CUSTOMER_A
 OSPFv3 1 address-family ipv4 vrf CUSTOMER_A
 Router ID 1.1.1.1
 Supports NSSA (compatible with RFC 3101)
 Supports Database Exchange Summary List Optimization (RFC 5243)
 Maximum number of non self-generated LSA allowed 50000
 ...
 ```

In this simple single-area setup, the command may not be required for basic neighbor formation.

However, it is good practice in VRF-lite OSPFv3 labs because it prevents IOS XE from applying MPLS L3VPN PE-CE behavior in a non-MPLS design.

## OSPFv3 Router IDs

Each OSPFv3 VRF address family has a router ID.

In this lab, both R1 OSPFv3 VRF address families use the same router ID.

```text
router ospfv3 1
 address-family ipv4 unicast vrf CUSTOMER_A
  router-id 1.1.1.1
 !
 address-family ipv4 unicast vrf CUSTOMER_B
  router-id 1.1.1.1
```

This works because the address-family instances are in separate VRFs.

They do not share routes.

They do not form adjacencies with each other.

However, unique router IDs are usually easier to troubleshoot.

Example:

```text
router ospfv3 1
 address-family ipv4 unicast vrf CUSTOMER_A
  router-id 1.1.1.10
 !
 address-family ipv4 unicast vrf CUSTOMER_B
  router-id 1.1.1.20
```

The rule is:

```text
Router IDs must be unique inside an OSPF domain.
Separate VRFs are separate OSPF domains.
```

## OSPFv3 IPv4 vs OSPFv3 IPv6

This page uses OSPFv3 for IPv4 routing.

```text
address-family ipv4 unicast vrf CUSTOMER_A
ospfv3 1 ipv4 area 0
```

OSPFv3 can also be used for IPv6 routing.

```text
address-family ipv6 unicast vrf CUSTOMER_A
ospfv3 1 ipv6 area 0
```

The address family decides which routes are exchanged.

```text
IPv4 AF -> IPv4 routes
IPv6 AF -> IPv6 routes
```

The process ID can be the same.

```text
router ospfv3 1
 address-family ipv4 unicast vrf CUSTOMER_A
 address-family ipv6 unicast vrf CUSTOMER_A
```

That is one reason OSPFv3 is organized around address families.

> In either case, IPv6 routing must be enabled and each interface must have an IPv6 LLA.

## Redistribution Inside a VRF

Redistribution happens inside the selected OSPFv3 VRF address family.

Example:

```text
R1(config)# ip route vrf CUSTOMER_A 172.16.1.0 255.255.255.0 Null0

R1(config)# router ospfv3 1
R1(config-router)# address-family ipv4 unicast vrf CUSTOMER_A
R1(config-router-af)# redistribute static
```

This redistributes static routes from `CUSTOMER_A` into the OSPFv3 IPv4 address family for `CUSTOMER_A`.

It does not redistribute static routes from the global routing table.

It does not redistribute static routes from `CUSTOMER_B`.

For `CUSTOMER_B`, use the `CUSTOMER_B` address family.

```text
R1(config)# router ospfv3 1
R1(config-router)# address-family ipv4 unicast vrf CUSTOMER_B
R1(config-router-af)# redistribute static
```

Redistribution is address-family-specific and VRF-specific.

## OSPFv3 Does Not Leak Routes Between VRFs

OSPFv3 running in a VRF learns routes for that VRF.

It does not automatically share them with another VRF.

Example:

```text
OSPFv3 process 1 IPv4 AF in CUSTOMER_A learns 192.168.1.0/24.
OSPFv3 process 1 IPv4 AF in CUSTOMER_B also learns 192.168.1.0/24.
```

These are separate routes in separate routing tables.

```text
show ip route vrf CUSTOMER_A 192.168.1.0
show ip route vrf CUSTOMER_B 192.168.1.0
```

There is no automatic exchange between the two VRF address families.

To share routes between VRFs, configure route leaking.

Route leaking is covered in a separate section.

## Troubleshooting

### IPv6 Is Not Enabled

OSPFv3 uses IPv6 link-local transport.

If the interface does not have IPv6 enabled, the OSPFv3 adjacency may not form.

Check the interface configuration.

```text
interface Ethernet0/0
 ipv6 enable
 ospfv3 1 ipv4 area 0
```

Also check global IPv6 routing.

```text
ipv6 unicast-routing
```

### OSPFv3 Was Not Enabled on the Interface

OSPFv3 does not use OSPFv2-style `network` commands.

This is not the correct OSPFv3 activation method:

```text
router ospfv3 1
 network 10.10.10.0 0.0.0.255 area 0
```

Enable OSPFv3 under the interface.

```text
interface Ethernet0/0
 ospfv3 1 ipv4 area 0
```

### The Address Family Is Missing

The interface command points to OSPFv3 process 1.

```text
interface Ethernet0/0
 ospfv3 1 ipv4 area 0
```

The router process also needs an IPv4 address family for the VRF.

```text
router ospfv3 1
 address-family ipv4 unicast vrf CUSTOMER_A
```

Without the correct address family, the route installation and process behavior will not match the intended VRF.

### The Interface Is in the Wrong VRF

The interface VRF membership controls the routing context.

Check it with:

```text
show vrf interfaces
```

Example:

```text
Ethernet0/0 -> CUSTOMER_A
Ethernet0/1 -> CUSTOMER_B
```

If `Ethernet0/0` is accidentally placed in the wrong VRF, OSPFv3 will also run in the wrong routing context.

### capability vrf-lite Is Missing

When using OSPFv3 with VRF-lite, include `capability vrf-lite` under the OSPFv3 VRF address family.

```text
router ospfv3 1
 address-family ipv4 unicast vrf CUSTOMER_A
  capability vrf-lite
 !
 address-family ipv4 unicast vrf CUSTOMER_B
  capability vrf-lite
```

## Key Points

- OSPFv3 can run inside a VRF.
- OSPFv3 uses address-family configuration for VRF-aware routing.
- OSPFv3 is different from OSPFv2, which uses a separate process per VRF.
- A single OSPFv3 process can contain multiple VRF address families.
- Use `address-family ipv4 unicast vrf <vrf-name>` for IPv4 routing in a VRF.
- Enable OSPFv3 directly on interfaces with `ospfv3 <process-id> ipv4 area <area-id>`.
- OSPFv3 does not use OSPFv2-style `network` commands.
- Enable IPv6 processing on interfaces that run OSPFv3.
- The interface VRF membership decides which VRF address-family context the interface belongs to.
- Use `capability vrf-lite` under OSPFv3 VRF address families in VRF-lite designs.
- OSPFv3 routes are installed into the associated VRF routing table.
- OSPFv3 does not leak routes between VRFs by itself.
