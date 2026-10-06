# VRF-Aware Static Routing

A VRF-aware static route is a static route installed into a specific VRF routing table.

A normal static route is installed into the global routing table.

A VRF-aware static route is installed into a VRF routing table.

```text
Global static route:
ip route 192.168.1.0 255.255.255.0 10.10.10.2

VRF static route:
ip route vrf CUSTOMER_A 192.168.1.0 255.255.255.0 10.10.10.2
```

The difference is the `vrf` keyword.

## Main Idea

A VRF has its own routing table.

A static route configured for one VRF belongs to that VRF only.

```text
ip route vrf CUSTOMER_A 192.168.1.0 255.255.255.0 10.10.10.2
```

This route is installed into the `CUSTOMER_A` routing table.

It is not installed into the global routing table.

It is not installed into `CUSTOMER_B`.

Verify it with:

```text
show ip route vrf CUSTOMER_A static
```

## Why This Matters

VRFs allow overlapping IP addresses.

In this example, both customers use the same R1-to-CE subnet.

```text
CUSTOMER_A link: 10.10.10.0/24
CUSTOMER_B link: 10.10.10.0/24
```

Both customers use the same CE LAN prefix.

```text
CUSTOMER_A CE LAN: 192.168.1.0/24
CUSTOMER_B CE LAN: 192.168.1.0/24
```

R1 also uses the same loopback prefix in both VRFs.

```text
CUSTOMER_A R1 loopback: 1.1.1.0/24
CUSTOMER_B R1 loopback: 1.1.1.0/24
```

This works because the routes are stored in different routing tables.

```text
CUSTOMER_A routing table:
C    10.10.10.0/24 is directly connected, Ethernet0/0
C    1.1.1.0/24 is directly connected, Loopback0
S    192.168.1.0/24 via 10.10.10.2

CUSTOMER_B routing table:
C    10.10.10.0/24 is directly connected, Ethernet0/1
C    1.1.1.0/24 is directly connected, Loopback1
S    192.168.1.0/24 via 10.10.10.2
```

The prefixes and next-hop addresses look identical.

The VRF context makes them different.

## Basic Syntax

Basic IPv4 syntax:

```text
ip route vrf <vrf-name> <prefix> <mask> <next-hop>
```

Example recursive route:

```text
ip route vrf CUSTOMER_A 192.168.1.0 255.255.255.0 10.10.10.2
```

Example directly connected route:

```text
ip route vrf CUSTOMER_A 192.168.1.0 255.255.255.0 Ethernet0/0
```

Example fully specified route:

```text
ip route vrf CUSTOMER_A 192.168.1.0 255.255.255.0 Ethernet0/0 10.10.10.2
```

For Ethernet links, avoid using only the exit interface unless you specifically want that behavior.

A next-hop route or a fully specified static route is usually clearer.

## Lab Topology

This page uses this topology.

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
R1(config-if)# rf forwarding CUSTOMER_B
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

The same IP address can be used on both R1 Ethernet interfaces because they are in different VRFs.

The same loopback address can also be used in both VRFs.

Verify the VRFs.

```text
R1#show vrf

Name                             Default RD            Protocols   Interfaces
CUSTOMER_A                       <not set>             ipv4        Et0/0
                                                                   Lo0
CUSTOMER_B                       <not set>             ipv4        Et0/1
                                                                   Lo1
```

Verify the connected routes in `CUSTOMER_A`.

```text
R1# show ip route vrf CUSTOMER_A

C    1.1.1.0/24 is directly connected, Loopback0
L    1.1.1.1/32 is directly connected, Loopback0
C    10.10.10.0/24 is directly connected, Ethernet0/0
L    10.10.10.1/32 is directly connected, Ethernet0/0
```

Verify the connected routes in `CUSTOMER_B`.

```text
R1# show ip route vrf CUSTOMER_B

C    1.1.1.0/24 is directly connected, Loopback1
L    1.1.1.1/32 is directly connected, Loopback1
C    10.10.10.0/24 is directly connected, Ethernet0/1
L    10.10.10.1/32 is directly connected, Ethernet0/1
```

The connected routes look the same, but they are in different VRF routing tables.

## CE Configuration

CE1 is not using VRFs in this example.

```text
CE1(config)# interface Ethernet0/0
CE1(config-if)# ip address 10.10.10.2 255.255.255.0
CE1(config-if)# no shutdown

CE1(config)# interface Loopback0
CE1(config-if)# ip address 192.168.1.1 255.255.255.0
```

CE2 does not use VRFs either.

```text
CE2(config)# interface Ethernet0/0
CE2(config-if)# ip address 10.10.10.2 255.255.255.0
CE2(config-if)# no shutdown

CE2(config)# interface Loopback0
CE2(config-if)# ip address 192.168.1.1 255.255.255.0
```

CE1 and CE2 can use the same addresses because they are separate customer routers.

They are not connected to the same Layer 2 segment.

R1 keeps the two customer contexts separate with VRFs.

> VRFs are local to the router on which they are configured.

## R1 Static Route in CUSTOMER_A

Configure a static route in `CUSTOMER_A` toward CE1's LAN.

```text
R1(config)# ip route vrf CUSTOMER_A 192.168.1.0 255.255.255.0 10.10.10.2
```

This tells R1:

```text
To reach 192.168.1.0/24 in VRF CUSTOMER_A,
send the packet to 10.10.10.2.
```

The next hop `10.10.10.2` is resolved in the `CUSTOMER_A` routing table.

R1 uses Ethernet0/0.

Verify the route.

```text
R1#show ip route vrf CUSTOMER_A static

S    192.168.1.0/24 [1/0] via 10.10.10.2
```

Verify the forwarding path.

```text
R1#show ip cef vrf CUSTOMER_A 192.168.1.0
192.168.1.0/24
  nexthop 10.10.10.2 Ethernet0/0
```

## R1 Static Route in CUSTOMER_B

Configure a static route in `CUSTOMER_B` toward CE2's LAN.

```text
R1(config)# ip route vrf CUSTOMER_B 192.168.1.0 255.255.255.0 10.10.10.2
```

This looks almost identical to the `CUSTOMER_A` static route.

The difference is the VRF name.

```text
CUSTOMER_A:
ip route vrf CUSTOMER_A 192.168.1.0 255.255.255.0 10.10.10.2

CUSTOMER_B:
ip route vrf CUSTOMER_B 192.168.1.0 255.255.255.0 10.10.10.2
```

The destination prefix is the same.

The next hop is the same.

The routing table is different.

Verify the route.

```text
R1# show ip route vrf CUSTOMER_B static

S    192.168.1.0/24 [1/0] via 10.10.10.2
```

Verify the forwarding path.

```text
R1# show ip cef vrf CUSTOMER_B 192.168.1.0
192.168.1.0/24
  nexthop 10.10.10.2 Ethernet0/1
```

For `CUSTOMER_A`, R1 forwards out Ethernet0/0.

For `CUSTOMER_B`, R1 forwards out Ethernet0/1.

```text
CUSTOMER_A: 192.168.1.0/24 via 10.10.10.2, Ethernet0/0
CUSTOMER_B: 192.168.1.0/24 via 10.10.10.2, Ethernet0/1
```

The routes can look identical, but they are not the same route.

## Routes on the CE Routers

The CE routers need return routes to R1's loopbacks.

In this topology, both R1 loopbacks use the same prefix.

```text
CUSTOMER_A R1 Loopback0: 1.1.1.1/24
CUSTOMER_B R1 Loopback1: 1.1.1.1/24
```

CE1 installs a specific route to `1.1.1.0/24` through R1.

```text
CE1(config)# ip route 1.1.1.0 255.255.255.0 10.10.10.1
```

CE2 installs the same specific route through its own R1-facing link.

```text
CE2(config)# ip route 1.1.1.0 255.255.255.0 10.10.10.1
```

These routes look identical on the CE routers.

```text
CE1:
ip route 1.1.1.0 255.255.255.0 10.10.10.1

CE2:
ip route 1.1.1.0 255.255.255.0 10.10.10.1
```

They do not conflict because CE1 and CE2 are separate routers.

CE1 reaches R1's `CUSTOMER_A` loopback.

CE2 reaches R1's `CUSTOMER_B` loopback.

```text
CE1 -> 1.1.1.1 reaches R1 Loopback0 in CUSTOMER_A
CE2 -> 1.1.1.1 reaches R1 Loopback1 in CUSTOMER_B
```

This is the key point of the lab.

The same destination IP address can refer to different routing instances.

## Testing from the CE Routers

Test from CE1 using CE1's loopback as the source.

```text
CE1# ping 1.1.1.1 source Loopback0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 1.1.1.1, timeout is 2 seconds:
Packet sent with a source address of 192.168.1.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```

CE1 sends the packet to R1.

The packet enters R1 on Ethernet0/0.

Ethernet0/0 belongs to `CUSTOMER_A`.

R1 processes the packet in the `CUSTOMER_A` VRF.

The destination `1.1.1.1` matches Loopback0 in `CUSTOMER_A`.

Test from CE2 using CE2's loopback as the source.

```text
CE2# ping 1.1.1.1 source Loopback0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 1.1.1.1, timeout is 2 seconds:
Packet sent with a source address of 192.168.1.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```

CE2 sends the packet to R1.

The packet enters R1 on Ethernet0/1.

Ethernet0/1 belongs to `CUSTOMER_B`.

R1 processes the packet in the `CUSTOMER_B` VRF.

The destination `1.1.1.1` matches Loopback1 in `CUSTOMER_B`.

The destination IP of both pings is the same.

The incoming interface determines which VRF routing table is used for R1's route lookup.

## Testing from R1

From R1, use a VRF-aware ping.

Test `CUSTOMER_A`.

```text
R1# ping vrf CUSTOMER_A 192.168.1.1 source Loopback0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.1.1, timeout is 2 seconds:
Packet sent with a source address of 1.1.1.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/3 ms
```

This ping uses the `CUSTOMER_A` routing table.

R1 sends the packet toward CE1.

Test `CUSTOMER_B`.

```text
R1# ping vrf CUSTOMER_B 192.168.1.1 source Loopback1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.1.1, timeout is 2 seconds:
Packet sent with a source address of 1.1.1.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```

This ping uses the `CUSTOMER_B` routing table.

R1 sends the packet toward CE2.

The destination address is the same in both tests.

The VRF selects the routing table.

```text
ping vrf CUSTOMER_A 192.168.1.1 source Loopback0 -> CE1 LAN
ping vrf CUSTOMER_B 192.168.1.1 source Loopback1 -> CE2 LAN
```

A normal ping uses the global routing table.

```text
R1# ping 192.168.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.1.1, timeout is 2 seconds:
.....
Success rate is 0 percent (0/5)
```

This fails because the global routing table does not have a route to `192.168.1.0/24`.

## Default Route in a VRF

A VRF can have its own default route.

Example for `CUSTOMER_A`:

```text
R1(config)# ip route vrf CUSTOMER_A 0.0.0.0 0.0.0.0 10.10.10.2
```

Example for `CUSTOMER_B`:

```text
R1(config)# ip route vrf CUSTOMER_B 0.0.0.0 0.0.0.0 10.10.10.2
```

These default routes look the same except for the VRF name.

They are installed into different routing tables.

```text
CUSTOMER_A default route -> 10.10.10.2 through Ethernet0/0
CUSTOMER_B default route -> 10.10.10.2 through Ethernet0/1
```

## Next-Hop Resolution

A VRF static route normally resolves its next hop inside the same VRF.

Example:

```text
ip route vrf CUSTOMER_A 192.168.1.0 255.255.255.0 10.10.10.2
```

R1 checks the `CUSTOMER_A` routing table to resolve `10.10.10.2`.

This works because `10.10.10.0/24` is connected in `CUSTOMER_A`.

```text
R1# show ip route vrf CUSTOMER_A 10.10.10.0

C    10.10.10.0/24 is directly connected, Ethernet0/0
```

For `CUSTOMER_B`, the next hop is also `10.10.10.2`.

But it resolves in `CUSTOMER_B`.

```text
R1# show ip route vrf CUSTOMER_B 10.10.10.0

C    10.10.10.0/24 is directly connected, Ethernet0/1
```

## The global Keyword

The `global` keyword tells the router that the next hop is resolved using the global routing table.

Example:

```text
ip route vrf CUSTOMER_A 0.0.0.0 0.0.0.0 203.0.113.1 global
```

This installs the default route into `CUSTOMER_A`.

But the next hop `203.0.113.1` is resolved through the global routing table.

```text
Route is installed into:
CUSTOMER_A

Next hop is resolved in:
Global routing table
```

This is not the normal same-VRF next-hop behavior.

Use this only when you intentionally want the VRF route to use a global next hop.

More detailed global-to-VRF static leaking examples are covered later in the route leaking section.

## Static Routes and Redistribution

Static routes can be redistributed inside a VRF.

Example:

```text
router ospf 10 vrf CUSTOMER_A
 redistribute static subnets
```

This redistributes static routes from the `CUSTOMER_A` routing table into OSPF for `CUSTOMER_A`.

It does not redistribute static routes from the global routing table.

It does not redistribute static routes from `CUSTOMER_B`.

Example with BGP:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute static
```

This redistributes static routes from `CUSTOMER_A` into the BGP IPv4 address family for `CUSTOMER_A`.

---

## Key Points

- A VRF-aware static route is installed into a specific VRF routing table.
- The `vrf` keyword identifies the VRF.
- A normal static route is installed into the global routing table.
- Overlapping prefixes can exist in different VRFs.
- The same next-hop IP address can exist in different VRFs.
- The same loopback prefix can exist in different VRFs.
- A VRF static route usually resolves its next hop inside the same VRF.
- The incoming interface selects the VRF for packets entering R1.
- The `global` keyword makes the next hop resolve through the global routing table.
- Static routes have an administrative distance of 1 by default.
- Static routes inside a VRF do not automatically leak to other VRFs.
