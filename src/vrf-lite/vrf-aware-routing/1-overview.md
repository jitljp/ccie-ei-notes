# VRF-Aware Routing

A VRF has its own routing table.

VRF-aware routing means a static route, routing protocol, or service is tied to a specific VRF.

Routes learned inside a VRF are installed into that VRF's routing table.

## Main Idea

A router can run routing for multiple VRFs at the same time.

Example:

```text
R1
├── Global routing table
│   └── Default route to the Internet
├── VRF CUSTOMER_A
│   └── OSPF routes for Customer A
└── VRF CUSTOMER_B
    └── EIGRP routes for Customer B
```

The protocols can be the same or different.

```text
CUSTOMER_A can use OSPF.
CUSTOMER_B can use EIGRP.
The global table can use static routes.
```

The important point is the routing context.

A route learned in `CUSTOMER_A` belongs to `CUSTOMER_A`.

A route learned in `CUSTOMER_B` belongs to `CUSTOMER_B`.

## VRF-Aware Routing vs Global Routing

Without VRFs, routing protocols install routes into the global routing table.

Example:

```text
router ospf 1
 network 10.10.10.0 0.0.0.255 area 0
```

The OSPF routes are installed into the global routing table.

With VRF-aware routing, the protocol is associated with a VRF.

Example:

```text
router ospf 1 vrf CUSTOMER_A
 network 10.10.10.0 0.0.0.255 area 0
```

The OSPF routes are installed into the `CUSTOMER_A` routing table.

## Interface VRF Membership

An interface's VRF membership is decided by the `vrf forwarding` command on the interface.

Example:

```text
interface GigabitEthernet0/1
 vrf forwarding CUSTOMER_A
 ip address 10.10.10.1 255.255.255.0
```

This interface belongs to `CUSTOMER_A`.

To activate OSPFv2 on this interface with the `network` command, you must create an OSPFv2 instance in the VRF.

```text
router ospf 1 vrf CUSTOMER_A
 network 10.10.10.0 0.0.0.255 area 0
```

This would not activate OSPF on the interface:

```text
router ospf 1
 network 10.10.10.0 0.0.0.255 area 0
```

## Basic Lab Topology

This section uses a simple topology.

```text
          +--------------------------------+
          |               R1               |
          |                                |
          |  E0/0                 E0/1     |
          | VRF CUSTOMER_A  VRF CUSTOMER_B |
          | 10.10.10.1/24    11.11.11.1/24 |
          +----+-------------------+-------+
               |                   |
               |                   |
              CE1                 CE2
         10.10.10.2/24       11.11.11.2/24
```

R1 has two VRFs.

```text
CUSTOMER_A
CUSTOMER_B
```

CE1 is connected to `CUSTOMER_A`.

CE2 is connected to `CUSTOMER_B`.

Routes learned from CE1 are installed into the `CUSTOMER_A` routing table.

Routes learned from CE2 are installed into the `CUSTOMER_B` routing table.

The two customers remain separate unless route leaking is configured.

## Creating the VRFs

Before configuring VRF-aware routing, create the VRFs.

```text
R1(config)# vrf definition CUSTOMER_A
R1(config-vrf)# address-family ipv4

R1(config)#vrf definition CUSTOMER_B
R1(config-vrf)# address-family ipv4
```

Assign interfaces to the VRFs.

```text
R1(config)# interface Ethernet0/0
R1(config-if)# vrf forwarding CUSTOMER_A
R1(config-if)# ip address 10.10.10.1 255.255.255.0
R1(config-if)# no shutdown

R1(config)# interface Ethernet0/1
R1(config-if)# vrf forwarding CUSTOMER_B
R1(config-if)# ip address 11.11.11.1 255.255.255.0
R1(config-if)# no shutdown
```

After this, connected routes are installed in the VRF routing tables.

```text
R1# show ip route vrf CUSTOMER_A
C        10.10.10.0/24 is directly connected, Ethernet0/0
L        10.10.10.1/32 is directly connected, Ethernet0/0

R1# show ip route vrf CUSTOMER_B
C        11.11.11.0/24 is directly connected, Ethernet0/1
L        11.11.11.1/32 is directly connected, Ethernet0/1
```

## VRF-Aware Static Routing

A VRF-aware static route uses the `vrf` keyword.

```text
ip route vrf CUSTOMER_A 0.0.0.0 0.0.0.0 10.10.10.2
```

This default route is installed into the `CUSTOMER_A` routing table.

It is not installed into the global routing table.

Verify it with:

```text
R1# show ip route 0.0.0.0  
% Network not in table

R1# show ip route vrf CUSTOMER_A 0.0.0.0

Routing Table: CUSTOMER_A
Routing entry for 0.0.0.0/0, supernet
  Known via "static", distance 1, metric 0, candidate default path
  Routing Descriptor Blocks:
  * 10.10.10.2
      Route metric is 0, traffic share count is 1
```

VRF-aware static routing is covered in the next page.

## VRF-Aware OSPFv2

OSPFv2 can run inside a VRF.

Example:

```text
router ospf 10 vrf CUSTOMER_A
 network 10.10.10.0 0.0.0.255 area 0
```

This OSPF process is associated with `CUSTOMER_A`.

OSPF neighbors are formed using interfaces in `CUSTOMER_A`.

OSPF routes are installed into the `CUSTOMER_A` routing table.

```text
R1# show ip route vrf CUSTOMER_A ospf
O        1.1.1.0 [110/11] via 10.10.10.2, 00:00:10, Ethernet0/0
```

To use OSPF with CE2, R1 would need another OSPF process in the `CUSTOMER_B` VRF:

```text
router ospf 11 vrf CUSTOMER_A
 network 11.11.11.0 0.0.0.255 area 0
```

OSPFv2 VRF-aware routing is covered in its own page.

## VRF-Aware OSPFv3

OSPFv3 can also be VRF-aware.

OSPFv3 uses address-family configuration.

Whereas OSPFv2 requires a unique OSPF process for each VRF, OSPFv3 can use a single process.

You can then create an address-family for each VRF:

```text
router ospfv3 1
 !
 address-family ipv4 unicast vrf CUSTOMER_B
 exit-address-family
 !
 address-family ipv4 unicast vrf CUSTOMER_A
 exit-address-family
```

OSPFv3 is then enabled directly on each interface (already assigned to the appropriate VRF).

```text
interface Ethernet0/0
 ospfv3 1 ipv4 area 0
!
interface Ethernet0/1
 ospfv3 1 ipv4 area 0
```

OSPFv3 VRF-aware routing is covered in its own page.

## VRF-Aware EIGRP

EIGRP can run inside a VRF.

In modern IOS XE, EIGRP named mode is commonly used.

Like OSPFv3, a single EIGRP process can support multiple VRFs:

```text
router eigrp CUSTOMER_EIGRP
!
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
  network 10.10.10.0 0.0.0.255
!
 address-family ipv4 unicast vrf CUSTOMER_B autonomous-system 100
  network 11.11.11.0 0.0.0.255
```

EIGRP VRF-aware routing is covered in its own page.

## VRF-Aware BGP

Like OSPFv3 and EIGRP, BGP uses address families for VRF-aware routing.

Example:

```text
router bgp 65000
!
 address-family ipv4 vrf CUSTOMER_A
  neighbor 10.10.10.2 remote-as 65100
  neighbor 10.10.10.2 activate
!
 address-family ipv4 vrf CUSTOMER_B
  neighbor 11.11.11.2 remote-as 65200
  neighbor 11.11.11.2 activate
```

VRF-aware BGP is covered in its own page.

## Redistribution Inside a VRF

Redistribution can be done inside a VRF.

Example:

```text
router ospf 10 vrf CUSTOMER_A
 redistribute static subnets
```

This redistributes static routes from the `CUSTOMER_A` routing table into OSPF for `CUSTOMER_A`.

It does not redistribute static routes from the global routing table.

It does not redistribute static routes from `CUSTOMER_B`.

The redistribution happens inside the VRF's routing context.

Example with BGP:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute connected
```

This redistributes connected routes from `CUSTOMER_A` into BGP for `CUSTOMER_A`.

## VRF-Aware Services

Some services also need to be VRF-aware.

Examples:

```text
ping
traceroute
SSH
SNMP
Syslog
NTP
TACACS+
RADIUS
DNS
NetFlow
IP SLA
```

For example, a normal ping uses the global routing table.

```text
ping 10.10.10.2
```

A VRF-aware ping uses a VRF routing table.

```text
ping vrf CUSTOMER_A 10.10.10.2
```

If a router needs to reach an NTP server through a management VRF, the NTP configuration needs a VRF option.

Example:

```text
ntp server vrf MGMT 192.0.2.10
```

VRF-aware services are covered in their own page.

## Route Leaking 

VRF-aware routing and VRF route leaking are related, but they are not the same thing.

VRF-aware routing controls how routes are learned inside a VRF.

Route leaking controls how routes are shared between routing tables.

```text
VRF-aware routing:
    Learn routes inside CUSTOMER_A.

Route leaking:
    Share selected CUSTOMER_A routes with CUSTOMER_B or the global table.
```

```text
CUSTOMER_A learns 10.10.10.0/24 through OSPF.
CUSTOMER_B does not automatically learn 10.10.10.0/24.
```

To make that route available in `CUSTOMER_B`, route leaking must be configured.

Route leaking is covered in a separate section.

## Key Points

- VRF-aware routing means routing is performed inside a specific VRF.
- A VRF has its own routing table and forwarding table.
- Interfaces assigned to a VRF are used by routing protocols in that VRF.
- Global routing and VRF routing are separate.
- Static routes can be configured inside a VRF.
- OSPFv2 uses a separate process for each VRF.
- OSPFv3, EIGRP, and BGP support multiple VRFs under a single process using address families.
- Redistribution happens inside the selected routing context.
- VRF-aware services must use the correct VRF.
