# VRF-Aware EIGRP

EIGRP can run inside a VRF.

In modern IOS XE, EIGRP named mode is commonly used.

EIGRP named mode uses address families.

This is similar to OSPFv3 and BGP.

It is different from OSPFv2.

```text
OSPFv2:
router ospf 10 vrf CUSTOMER_A
router ospf 20 vrf CUSTOMER_B

OSPFv3:
router ospfv3 1
 address-family ipv4 unicast vrf CUSTOMER_A
 address-family ipv4 unicast vrf CUSTOMER_B

EIGRP named mode:
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
 address-family ipv4 unicast vrf CUSTOMER_B autonomous-system 100

BGP:
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
 address-family ipv4 vrf CUSTOMER_B
```

OSPFv2 uses a separate process per VRF.

EIGRP named mode can use one EIGRP named process with multiple VRF address families.

## Main Idea

A VRF has its own routing table.

An EIGRP address family associated with a VRF installs routes into that VRF routing table.

Example:

```text
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
```

This creates an EIGRP IPv4 unicast address family for `CUSTOMER_A`.

Routes learned in this address family are installed into the `CUSTOMER_A` routing table.

They are not installed into the global routing table.

They are not installed into `CUSTOMER_B`.

Verify with:

```text
show ip route vrf CUSTOMER_A eigrp
```

## EIGRP Named Mode Uses Address Families

This is the key point.

EIGRP named mode can use one named process with multiple VRF address families.

```text
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
 address-family ipv4 unicast vrf CUSTOMER_B autonomous-system 100
```

Each address family has its own routing context.

```text
EIGRP named process JITL
├── IPv4 unicast VRF CUSTOMER_A, AS 100
└── IPv4 unicast VRF CUSTOMER_B, AS 100
```

The EIGRP process name is the same.

The AS numbers can be the same, according to each customer's AS number.


The VRF address-family context is different.

Each VRF address family has its own:

```text
Interfaces
Neighbors
Topology table
Routes
EIGRP metrics
Redistribution
```

The same prefixes can exist in multiple EIGRP VRF address families.

The same neighbor IP address can also exist in multiple VRFs.

The VRF context keeps them separate.

## Lab Topology

This page uses the same overlapping-address topology as the static, OSPFv2, and OSPFv3 pages.

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

## R1 VRF Configuration

Create the VRFs.

```text
R1(config)# vrf definition CUSTOMER_A
R1(config-vrf)# address-family ipv4

R1(config)# vrf definition CUSTOMER_B
R1(config-vrf)# address-family ipv4
```

Assign the physical interfaces to the VRFs.

```text
R1(config)# interface Ethernet0/0
R1(config-if)# vrf forwarding CUSTOMER_A
R1(config-if)# ip address 10.10.10.1 255.255.255.0
R1(config-if)# no shutdown

R1(config)# interface Ethernet0/1
R1(config-if)# vrf forwarding CUSTOMER_B
R1(config-if)# ip address 10.10.10.1 255.255.255.0
R1(config-if)# no shutdown
```

Assign the loopbacks to the VRFs.

```text
R1(config)# interface Loopback0
R1(config-if)# vrf forwarding CUSTOMER_A
R1(config-if)# ip address 1.1.1.1 255.255.255.0

R1(config)# interface Loopback1
R1(config-if)# vrf forwarding CUSTOMER_B
R1(config-if)# ip address 1.1.1.1 255.255.255.0
```

The same IP address can be used in both VRFs.

Verify the VRF interfaces.

```text
R1# show vrf

Name                             Default RD            Protocols   Interfaces
CUSTOMER_A                       <not set>             ipv4        Et0/0
                                                                   Lo0
CUSTOMER_B                       <not set>             ipv4        Et0/1
                                                                   Lo1
```

Verify the connected routes in each VRF.

```text
R1# show ip route vrf CUSTOMER_A

C    1.1.1.0/24 is directly connected, Loopback0
L    1.1.1.1/32 is directly connected, Loopback0
C    10.10.10.0/24 is directly connected, Ethernet0/0
L    10.10.10.1/32 is directly connected, Ethernet0/0
```

```text
R1# show ip route vrf CUSTOMER_B

C    1.1.1.0/24 is directly connected, Loopback1
L    1.1.1.1/32 is directly connected, Loopback1
C    10.10.10.0/24 is directly connected, Ethernet0/1
L    10.10.10.1/32 is directly connected, Ethernet0/1
```

## CE Configuration

The CEs are not using VRFs.

CE1:

```text
CE1(config)# interface Ethernet0/0
CE1(config-if)# ip address 10.10.10.2 255.255.255.0
CE1(config-if)# no shutdown

CE1(config)# interface Loopback0
CE1(config-if)# ip address 192.168.1.1 255.255.255.0
```

CE2:

```text
CE2(config)# interface Ethernet0/0
CE2(config-if)# ip address 10.10.10.2 255.255.255.0
CE2(config-if)# no shutdown

CE2(config)# interface Loopback0
CE2(config-if)# ip address 192.168.1.1 255.255.255.0
```

CE1 and CE2 can use the same addresses because they are separate customer routers.

They are not connected to the same Layer 2 segment.

## R1 EIGRP Named Mode Configuration

Configure one EIGRP named process.

```text
R1(config)# router eigrp JITL
```

Configure the IPv4 unicast address family for `CUSTOMER_A`.

```text
R1(config-router)# address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
R1(config-router-af)# eigrp router-id 1.1.1.1
R1(config-router-af)# network 10.10.10.0 0.0.0.255
R1(config-router-af)# network 1.1.1.0 0.0.0.255
```

Configure the IPv4 unicast address family for `CUSTOMER_B`.

```text
R1(config-router)# address-family ipv4 unicast vrf CUSTOMER_B autonomous-system 100
R1(config-router-af)# eigrp router-id 1.1.1.1
R1(config-router-af)# network 10.10.10.0 0.0.0.255
R1(config-router-af)# network 1.1.1.0 0.0.0.255
```

The same EIGRP named process supports both VRFs.

```text
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
 address-family ipv4 unicast vrf CUSTOMER_B autonomous-system 100
```

The two address families use the same router ID in this lab.

That is acceptable here because they are separate EIGRP instances in separate VRFs.

In production, unique router IDs are often easier to troubleshoot.

Example:

```text
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
  eigrp router-id 1.1.1.10
 address-family ipv4 unicast vrf CUSTOMER_B autonomous-system 100
  eigrp router-id 1.1.1.20
```

## CE EIGRP Configuration

The CE routers use normal EIGRP named mode without VRFs.

CE1:

```text
CE1(config)# router eigrp JITL
CE1(config-router)# address-family ipv4 unicast autonomous-system 100
CE1(config-router-af)# eigrp router-id 192.168.1.1
CE1(config-router-af)# network 10.10.10.0 0.0.0.255
CE1(config-router-af)# network 192.168.1.0 0.0.0.255
```

CE2:

```text
CE2(config)# router eigrp JITL
CE2(config-router)# address-family ipv4 unicast autonomous-system 100
CE2(config-router-af)# eigrp router-id 192.168.1.1
CE2(config-router-af)# network 10.10.10.0 0.0.0.255
CE2(config-router-af)# network 192.168.1.0 0.0.0.255
```

CE1 and CE2 can use the same EIGRP process name, autonomous system, and router ID because they are separate customer routers.

They do not form an adjacency with each other.

CE1 forms an adjacency with R1 in `CUSTOMER_A`.

CE2 forms an adjacency with R1 in `CUSTOMER_B`.

## Autonomous System per Address Family

In EIGRP named mode, the autonomous system is configured under the address family.

Example:

```text
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
```

This is different from classic EIGRP configuration.

Classic EIGRP:

```text
router eigrp 100
 network 10.10.10.0 0.0.0.255
```

Named mode:

```text
router eigrp JITL
 address-family ipv4 unicast autonomous-system 100
  network 10.10.10.0 0.0.0.255
```

Named mode with VRF:

```text
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
  network 10.10.10.0 0.0.0.255
```

The EIGRP process name is not the autonomous system number.

The autonomous system is part of the address family.

## EIGRP Neighbors

R1 should form one EIGRP neighbor in each VRF.

```text
R1# show ip eigrp vrf CUSTOMER_A neighbors

EIGRP-IPv4 VR(JITL) Address-Family Neighbors for AS(100)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
0   10.10.10.2              Et0/0                    13 00:01:20    1   100  0  5
```

```text
R1# show ip eigrp vrf CUSTOMER_B neighbors

EIGRP-IPv4 VR(JITL) Address-Family Neighbors for AS(100)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
0   10.10.10.2              Et0/1                    12 00:01:18    1   100  0  5
```

The neighbor address looks the same.

The EIGRP named process is the same.

The autonomous system is the same.

The VRF address-family context is different.

The interface is different.

```text
CUSTOMER_A adjacency:
EIGRP named process JITL
IPv4 unicast VRF CUSTOMER_A
AS 100
Interface Ethernet0/0

CUSTOMER_B adjacency:
EIGRP named process JITL
IPv4 unicast VRF CUSTOMER_B
AS 100
Interface Ethernet0/1
```

## EIGRP Routes in Each VRF

R1 learns CE1's LAN in `CUSTOMER_A`.

```text
R1# show ip route vrf CUSTOMER_A eigrp

D    192.168.1.0/24 [90/130816] via 10.10.10.2, Ethernet0/0
```

R1 learns CE2's LAN in `CUSTOMER_B`.

```text
R1# show ip route vrf CUSTOMER_B eigrp

D    192.168.1.0/24 [90/130816] via 10.10.10.2, Ethernet0/1
```

The routes look the same.

They are in different VRF routing tables.

```text
CUSTOMER_A:
192.168.1.0/24 via 10.10.10.2, Ethernet0/0

CUSTOMER_B:
192.168.1.0/24 via 10.10.10.2, Ethernet0/1
```

`CUSTOMER_A` does not learn `CUSTOMER_B` routes.

`CUSTOMER_B` does not learn `CUSTOMER_A` routes.

EIGRP is VRF-aware, but it does not leak routes between VRFs.

## CE Routes to R1 Loopbacks

CE1 learns R1's `CUSTOMER_A` loopback.

```text
CE1# show ip route eigrp

D    1.1.1.0/24 [90/130816] via 10.10.10.1, Ethernet0/0
```

CE2 learns R1's `CUSTOMER_B` loopback.

```text
CE2# show ip route eigrp

D    1.1.1.0/24 [90/130816] via 10.10.10.1, Ethernet0/0
```

The routes look identical on the CE routers.

But they point to different VRF instances on R1.

```text
CE1 -> 1.1.1.1 reaches R1 Loopback0 in CUSTOMER_A
CE2 -> 1.1.1.1 reaches R1 Loopback1 in CUSTOMER_B
```

The destination IP address is the same.

The routing instance on R1 is different.

## Network Statement Behavior

The EIGRP `network` command matches interfaces in the routing context of the EIGRP address family.

Example:

```text
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
  network 10.10.10.0 0.0.0.255
```

This activates EIGRP on interfaces in `CUSTOMER_A` with matching IP addresses.

It does not activate EIGRP on `Ethernet0/1` in `CUSTOMER_B`, even though `Ethernet0/1` also uses `10.10.10.1/24`.

That interface belongs to a different VRF address family.

For `CUSTOMER_B`, configure its own address family.

```text
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_B autonomous-system 100
  network 10.10.10.0 0.0.0.255
```

The same `network` command can be used.

The VRF address-family context makes it separate.

## EIGRP Router IDs

Each EIGRP address family can have a router ID.

In this lab, both R1 EIGRP VRF address families use the same router ID.

```text
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
  eigrp router-id 1.1.1.1
 !
 address-family ipv4 unicast vrf CUSTOMER_B autonomous-system 100
  eigrp router-id 1.1.1.1
```

This works because the address-family instances are in separate VRFs.

They do not share topology tables.

They do not form adjacencies with each other.

However, unique router IDs are usually easier to troubleshoot.

Example:

```text
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
  eigrp router-id 1.1.1.10
 !
 address-family ipv4 unicast vrf CUSTOMER_B autonomous-system 100
  eigrp router-id 1.1.1.20
```

The rule is:

```text
Router IDs should be unique inside an EIGRP routing domain.
Separate VRFs are separate EIGRP routing domains.
```

## Classic EIGRP and VRFs

Older EIGRP configurations may use classic mode.

Example:

```text
router eigrp 100
 network 10.10.10.0 0.0.0.255
```

For modern IOS XE notes, named mode is usually clearer because it uses address families.

In named mode, the VRF is configured directly under the address family.

```text
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
```

This also makes the contrast with OSPFv2 clear.

```text
OSPFv2:
separate process per VRF

EIGRP named mode:
address family per VRF
```

## EIGRP Metrics

EIGRP routes still use composite metrics.

Example route:

```text
D    192.168.1.0/24 [90/130816] via 10.10.10.2, Ethernet0/0
```

The first number is the administrative distance.

```text
90
```

The second number is the EIGRP metric.

```text
130816
```

The metric calculation is not changed just because the route is inside a VRF.

The VRF controls the routing table.

EIGRP controls how the metric is calculated.

## Passive Interfaces

Passive interfaces can be configured inside the EIGRP address family.

Example:

```text
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
  af-interface Loopback0
   passive-interface
```

You can also configure passive-interface default behavior.

```text
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
  af-interface default
   passive-interface
  af-interface Ethernet0/0
   no passive-interface
```

This configuration belongs to the `CUSTOMER_A` address family.

It does not affect `CUSTOMER_B`.

For `CUSTOMER_B`, configure passive-interface behavior under the `CUSTOMER_B` address family.

## Redistribution Inside a VRF

Redistribution happens inside the selected EIGRP VRF address family.

Example:

```text
R1(config)# ip route vrf CUSTOMER_A 172.16.1.0 255.255.255.0 Null0

R1(config)# router eigrp JITL
R1(config-router)# address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
R1(config-router-af)# topology base
R1(config-router-af-topology)# redistribute static
```

This redistributes static routes from `CUSTOMER_A` into EIGRP for `CUSTOMER_A`.

It does not redistribute static routes from the global routing table.

It does not redistribute static routes from `CUSTOMER_B`.

For `CUSTOMER_B`, use the `CUSTOMER_B` address family.

```text
R1(config)# router eigrp JITL
R1(config-router)# address-family ipv4 unicast vrf CUSTOMER_B autonomous-system 100
R1(config-router-af)# topology base
R1(config-router-af-topology)# redistribute static
```

Redistribution is address-family-specific and VRF-specific.

## EIGRP Does Not Leak Routes Between VRFs

EIGRP running in a VRF learns routes for that VRF.

It does not automatically share them with another VRF.

Example:

```text
EIGRP process JITL IPv4 AF in CUSTOMER_A learns 192.168.1.0/24.
EIGRP process JITL IPv4 AF in CUSTOMER_B also learns 192.168.1.0/24.
```

These are separate routes in separate routing tables.

```text
show ip route vrf CUSTOMER_A 192.168.1.0
show ip route vrf CUSTOMER_B 192.168.1.0
```

There is no automatic exchange between the two VRF address families.

To share routes between VRFs, configure route leaking.

Route leaking is covered in a separate section.

## Verification Commands

Show the VRFs.

```text
show vrf
show vrf interfaces
show ip interface brief vrf all
```

Show EIGRP address-family information for each VRF.

```text
show ip eigrp vrf CUSTOMER_A
show ip eigrp vrf CUSTOMER_B
```

Show EIGRP neighbors.

```text
show ip eigrp vrf CUSTOMER_A neighbors
show ip eigrp vrf CUSTOMER_B neighbors
```

Show EIGRP interfaces.

```text
show ip eigrp vrf CUSTOMER_A interfaces
show ip eigrp vrf CUSTOMER_B interfaces
```

Show the EIGRP topology table.

```text
show ip eigrp vrf CUSTOMER_A topology
show ip eigrp vrf CUSTOMER_B topology
```

Show EIGRP routes in each VRF.

```text
show ip route vrf CUSTOMER_A eigrp
show ip route vrf CUSTOMER_B eigrp
```

Show a specific route in each VRF.

```text
show ip route vrf CUSTOMER_A 192.168.1.0
show ip route vrf CUSTOMER_B 192.168.1.0
```

Test from R1 using the VRF.

```text
ping vrf CUSTOMER_A 192.168.1.1 source Loopback0
ping vrf CUSTOMER_B 192.168.1.1 source Loopback1
```

Test from the CE routers.

```text
CE1# ping 1.1.1.1 source Loopback0
CE2# ping 1.1.1.1 source Loopback0
```

## Troubleshooting

### EIGRP Was Configured Globally

This is a global named-mode EIGRP address family.

```text
router eigrp JITL
 address-family ipv4 unicast autonomous-system 100
  network 10.10.10.0 0.0.0.255
```

It does not run inside `CUSTOMER_A`.

For a VRF, specify the VRF in the address family.

```text
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
  network 10.10.10.0 0.0.0.255
```

### The Address Family Is Missing

The EIGRP named process by itself does not activate EIGRP.

```text
router eigrp JITL
```

You need an address family.

```text
address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
```

Without the correct address family, EIGRP will not run in the intended VRF.

### The Autonomous System Is Missing or Wrong

EIGRP neighbors must be in the same autonomous system.

R1:

```text
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
```

CE1:

```text
router eigrp JITL
 address-family ipv4 unicast autonomous-system 100
```

If one side uses AS 100 and the other uses AS 200, the neighbor relationship will not form.

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

If `Ethernet0/0` is accidentally placed in the wrong VRF, EIGRP will also run in the wrong routing context.

### The Wrong Routing Table Was Checked

This checks the global routing table.

```text
show ip route eigrp
```

This checks the `CUSTOMER_A` routing table.

```text
show ip route vrf CUSTOMER_A eigrp
```

This checks the `CUSTOMER_B` routing table.

```text
show ip route vrf CUSTOMER_B eigrp
```

### The Neighbor Address Looks Duplicated

In this topology, both CE routers use `10.10.10.2`.

That is not a problem.

```text
CUSTOMER_A neighbor on Ethernet0/0
CUSTOMER_B neighbor on Ethernet0/1
```

The neighbors are in different VRF address-family contexts.

### The Same Route Appears Twice

This is expected.

```text
CUSTOMER_A:
D 192.168.1.0/24 via 10.10.10.2, Ethernet0/0

CUSTOMER_B:
D 192.168.1.0/24 via 10.10.10.2, Ethernet0/1
```

The prefix is the same.

The next hop is the same.

The VRF is different.

### Passive Interface Blocks the Adjacency

Check whether the interface is passive.

```text
show ip eigrp vrf CUSTOMER_A interfaces
```

If the interface is passive, EIGRP will not form a neighbor relationship on that interface.

Example fix:

```text
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
  af-interface Ethernet0/0
   no passive-interface
```

### Network Statement Does Not Match

Check the network statements under the correct address family.

```text
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
  network 10.10.10.0 0.0.0.255
```

The network statement must match the interface address inside that VRF.

## Key Points

- EIGRP can run inside a VRF.
- Modern IOS XE EIGRP named mode uses address-family configuration.
- EIGRP named mode is different from OSPFv2, which uses a separate process per VRF.
- A single EIGRP named process can contain multiple VRF address families.
- Use `address-family ipv4 unicast vrf <vrf-name> autonomous-system <as-number>`.
- The EIGRP process name is not the autonomous system number.
- EIGRP neighbors form inside the selected VRF address-family context.
- The same prefixes can exist in multiple EIGRP VRF address families.
- The same next-hop or neighbor IP address can exist in multiple VRFs.
- EIGRP routes are installed into the associated VRF routing table.
- Redistribution is address-family-specific and VRF-specific.
- EIGRP does not leak routes between VRFs by itself.
