# VASI

**VRF-Aware Software Infrastructure (VASI)** provides an internal packet-forwarding path between two routing contexts on the same IOS XE router.

VASI uses a pair of virtual interfaces, `vasileft` and `vasiright`. When a packet is forwarded out one interface, it enters the paired interface and undergoes another routing lookup in that interface's routing table.

Unlike route-target import/export or route replication, **VASI does not copy routes between routing tables**. Instead, each routing context has routes that forward traffic toward its side of the VASI pair.

VASI is particularly useful when inter-VRF traffic must pass through services such as **NAT**, **zone-based firewall**, or **IPsec**. It supports inter-VRF connectivity using **static or dynamic routing** between the VASI interfaces.

## VASI Interface Pairs

A VASI pair consists of two virtual interfaces with the same number:

```text
             Internal connection
    VRF_A                           VRF_B
  Vasileft100 <----------------> Vasiright100
```

The interfaces are paired automatically by their number. For example, `Vasileft100` pairs with `Vasiright100`, not `Vasiright200`.

- `vasileft` and `vasiright` are **symmetrical**. Either interface can send or receive packets.
- Each interface has its own IP address and belongs to a routing context.
- Both ends of the pair must be configured for the pair to operate.
- **One side can belong to the global routing table.** Simply omit `vrf forwarding` on that interface.

A VASI interface pair behaves like an internal Layer 3 link between two routing contexts.

A route must direct the packet out the appropriate VASI interface, and a second route lookup occurs after the packet enters its partner.

### Packet Forwarding

For example, consider a packet from `VRF_A` destined for a network in `VRF_B`:

```text
Packet arrives on R1 in VRF_A
              |
              v
     VRF_A routing lookup
              |
       Out Vasileft100
              |
         VASI pair
              |
      In Vasiright100
              |
              v
     VRF_B routing lookup
              |
              v
     Out VRF_B interface
```

**Two routing lookups** occur on R1, one in each routing context.

Cisco also documents a **TTL decrement at each lookup**, so a packet crossing a VASI pair normally has its TTL decremented twice on R1.

## Lab Topology

We'll use the same four-router topology from the preceding route-leaking lessons:

```text
                     R2
               Lo0: 2.2.2.2/32
                      |
                G1 (Global)
                      |
R3 ----- G2 -------- R1 -------- G3 ----- R4
Lo0:     (VRF_A)                 (VRF_B)     Lo0:
3.3.3.3/32                               4.4.4.4/32
```

> R1 uses a CSR1000v instead of IOL because the IOL image in CML doesn't support VASI interfaces.

| Device | Interface | IPv4 Address | Routing Context on R1 |
| --- | --- | --- | --- |
| R1 | `GigabitEthernet1` | `10.1.2.1/24` | Global |
| R2 | `Ethernet0/0` | `10.1.2.2/24` | Connected to global |
| R2 | `Loopback0` | `2.2.2.2/32` | Behind global |
| R1 | `GigabitEthernet2` | `10.1.3.1/24` | `VRF_A` |
| R3 | `Ethernet0/0` | `10.1.3.3/24` | Connected to `VRF_A` |
| R3 | `Loopback0` | `3.3.3.3/32` | Behind `VRF_A` |
| R1 | `GigabitEthernet3` | `10.1.4.1/24` | `VRF_B` |
| R4 | `Ethernet0/0` | `10.1.4.4/24` | Connected to `VRF_B` |
| R4 | `Loopback0` | `4.4.4.4/32` | Behind `VRF_B` |

We'll demonstrate two independent VASI pairs on **R1**:

| VASI Pair | Left Side | Right Side | Goal |
| --- | --- | --- | --- |
| `100` | `VRF_A` | `VRF_B` | Connect R3 Loopback0 and R4 Loopback0 |
| `200` | `VRF_A` | Global | Connect R3 Loopback0 and R2 Loopback0 |

The **VRFs and all physical interface and loopback IP addresses are already configured**, but there is **no routing configured**.

The following examples provide all routes needed for the tests. R2, R3, and R4 use their global routing tables; only R1 has the two named VRFs.

First, we'll establish both VASI paths using static routes.

Then we'll replace the **pair 100 static routing** with OSPF, while retaining the independent pair 200 static routing.

> **Before starting:** Remove the route replication, BGP import/export, and cross-VRF routes from earlier labs if still configured.
> Otherwise, competing routes may bypass VASI.

## Initial Routing to the End Routers

Before configuring the VASI pairs, R1 needs routes to the loopbacks directly behind its three physical interfaces. Configure one ordinary static route in each routing context:

```text
R1(config)# ip route 2.2.2.2 255.255.255.255 10.1.2.2
R1(config)# ip route vrf VRF_A 3.3.3.3 255.255.255.255 10.1.3.3
R1(config)# ip route vrf VRF_B 4.4.4.4 255.255.255.255 10.1.4.4
```

These routes **do not provide inter-VRF connectivity**.

They only allow R1 to reach the three loopbacks in their respective original routing contexts.

We will later replace R1's two VRF static routes with OSPF-learned routes in the dynamic-routing example.

## VRF_A to VRF_B Using VASI

**Goal:** Allow R3's `3.3.3.3/32` to communicate with R4's `4.4.4.4/32` using VASI pair `100`.

### Step 1: Configure the VASI Pair

Create the first interface in `VRF_A`:

```text
R1(config)# interface Vasileft100
R1(config-if)# vrf forwarding VRF_A
R1(config-if)# ip address 172.16.100.1 255.255.255.252
```

Create its partner in `VRF_B`:

```text
R1(config)# interface Vasiright100
R1(config-if)# vrf forwarding VRF_B
R1(config-if)# ip address 172.16.100.2 255.255.255.252
```

Both ends use the same `/30` subnet, but the addresses belong to **different routing tables**:

```text
VRF_A                          VRF_B
Vasileft100                    Vasiright100
172.16.100.1/30  <---------->  172.16.100.2/30
```

The same connected subnet can exist in two different routing tables because the VRFs remain isolated. VASI supplies the internal connection between the two interfaces.

### Step 2: Configure Routes Through the VASI Pair

In `VRF_A`, point R4's loopback route toward the **VRF_A side** of the VASI pair:

```text
R1(config)# ip route vrf VRF_A 4.4.4.4 255.255.255.255 Vasileft100 172.16.100.2
```

In `VRF_B`, point R3's loopback route toward the **VRF_B side**:

```text
R1(config)# ip route vrf VRF_B 3.3.3.3 255.255.255.255 Vasiright100 172.16.100.1
```

These are **fully specified static routes**: each includes the local outgoing VASI interface and the IP address of its partner.

| Route Installed In | Destination | Outgoing Interface | Next Hop |
| --- | --- | --- | --- |
| `VRF_A` | `4.4.4.4/32` | `Vasileft100` | `172.16.100.2` |
| `VRF_B` | `3.3.3.3/32` | `Vasiright100` | `172.16.100.1` |

### Step 3: Configure Routes on R3 and R4

R3 must send traffic for R4's loopback to R1:

```text
R3(config)# ip route 4.4.4.4 255.255.255.255 10.1.3.1
```

R4 needs the reverse route:

```text
R4(config)# ip route 3.3.3.3 255.255.255.255 10.1.4.1
```

These end-router static routes complete the first example without a routing protocol. We will replace them with OSPF-learned routes in the dynamic-routing example below.

### Step 4: Verify

Check the routes on R1:

```text
R1# show ip route vrf VRF_A static
...
      3.0.0.0/32 is subnetted, 1 subnets
S        3.3.3.3 [1/0] via 10.1.3.3
      4.0.0.0/32 is subnetted, 1 subnets
S        4.4.4.4 [1/0] via 172.16.100.2, vasileft100

R1# show ip route vrf VRF_B static
...
      3.0.0.0/32 is subnetted, 1 subnets
S        3.3.3.3 [1/0] via 172.16.100.1, vasiright100
      4.0.0.0/32 is subnetted, 1 subnets
S        4.4.4.4 [1/0] via 10.1.4.4

R1# show ip cef vrf VRF_A 4.4.4.4 detail
4.4.4.4/32, epoch 0
  nexthop 172.16.100.2 vasileft100

R1# show ip cef vrf VRF_B 3.3.3.3 detail
3.3.3.3/32, epoch 0
  nexthop 172.16.100.1 vasiright100
```

The relevant routes point to the corresponding VASI interfaces.

Test from the loopbacks:

```text
R3# ping 4.4.4.4 source l0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 4.4.4.4, timeout is 2 seconds:
Packet sent with a source address of 3.3.3.3 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/3/10 ms
```

```
R4# ping 3.3.3.3 source l0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 3.3.3.3, timeout is 2 seconds:
Packet sent with a source address of 4.4.4.4 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```

For traffic from R3 to R4, R1's forwarding sequence is:

```text
R3 -> R1 G2 (VRF_A)
           |
     Vasileft100 (VRF_A)
           |
     Vasiright100 (VRF_B)
           |
     R1 G3 (VRF_B) -> R4
```

Traffic from R4 to R3 traverses the same pair in the opposite direction.

## Global Routing Table to VRF_A Using VASI

VASI can also connect a **VRF and the global routing table**.

Only the VRF-facing VASI interface needs `vrf forwarding`; the global-facing interface has no VRF assignment.

**Goal:** Allow R2's `2.2.2.2/32` to communicate with R3's `3.3.3.3/32` using a **second pair**, `200`.

This configuration coexists with VASI pair `100`; both pairs have a side in `VRF_A`, but each leads to a different routing context.

### Step 1: Configure the Second VASI Pair

Configure the `VRF_A` side:

```text
R1(config)# interface Vasileft200
R1(config-if)# vrf forwarding VRF_A
R1(config-if)# ip address 172.16.200.1 255.255.255.252
```

Configure the **global routing table** side. Notice there is **no `vrf forwarding` command**:

```text
R1(config)# interface Vasiright200
R1(config-if)# ip address 172.16.200.2 255.255.255.252
```

The resulting pair is:

```text
VRF_A                          Global
Vasileft200                    Vasiright200
172.16.200.1/30  <---------->  172.16.200.2/30
```

> **Important:** The global routing table does not require a configured VRF named `global` for VASI. The interface belongs to the global routing table because no VRF is assigned to it.

### Step 2: Configure Routes Through the Pair

In `VRF_A`, forward R2's loopback toward the **VRF_A side** of the pair:

```text
R1(config)# ip route vrf VRF_A 2.2.2.2 255.255.255.255 Vasileft200 172.16.200.2
```

In the global routing table, forward R3's loopback toward the **global side** of the pair:

```text
R1(config)# ip route 3.3.3.3 255.255.255.255 Vasiright200 172.16.200.1
```

Notice that the second static route uses the ordinary `ip route` command, without `vrf`:

| Route Installed In | Destination | Outgoing Interface | Next Hop |
| --- | --- | --- | --- |
| `VRF_A` | `2.2.2.2/32` | `Vasileft200` | `172.16.200.2` |
| Global | `3.3.3.3/32` | `Vasiright200` | `172.16.200.1` |

The routes steer packets through pair `200`; no `route-replicate`, `global-address-family`, or BGP import/export configuration is required.

### Step 3: Configure Routes on R2 and R3

R2 needs a return route toward R3 through R1's global interface:

```text
R2(config)# ip route 3.3.3.3 255.255.255.255 10.1.2.1
```

R3 needs a route toward R2 through R1's `VRF_A` interface:

```text
R3(config)# ip route 2.2.2.2 255.255.255.255 10.1.3.1
```

R1 must also retain its original global route toward R2 and its original `VRF_A` route toward R3.

### Step 4: Verify

Check the routes on R1:

```text
R1# show ip route vrf VRF_A | i 2.2.2.2
S        2.2.2.2 [1/0] via 172.16.200.2, vasileft200

R1# show ip route | i 3.3.3.3
S        3.3.3.3 [1/0] via 172.16.200.1, vasiright200

R1# show ip cef vrf VRF_A 2.2.2.2 detail
2.2.2.2/32, epoch 0
  nexthop 172.16.200.2 vasileft200

R1# show ip cef 3.3.3.3 detail          
3.3.3.3/32, epoch 2
  nexthop 172.16.200.1 vasiright200
```

The relevant route entries should point to `Vasileft200` and `Vasiright200` respectively.

Test from the end routers:

```text
R2# ping 3.3.3.3 source l0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 3.3.3.3, timeout is 2 seconds:
Packet sent with a source address of 2.2.2.2 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```

```
R3# ping 2.2.2.2 source l0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2.2.2.2, timeout is 2 seconds:
Packet sent with a source address of 3.3.3.3 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```

R2-to-R3 forwarding on R1 follows this path:

```text
R2 -> R1 G1 (Global)
           |
     Vasiright200
           |
     Vasileft200
           |
     R1 G2 (VRF_A) -> R3
```

The reverse direction uses the same pair in the opposite direction.

## Dynamic Routing over VASI: OSPF

VASI supports **dynamic routing neighborships between paired interfaces**, so the two VRFs can exchange routes dynamically rather than relying on static routes for inter-VRF forwarding.

We'll replace the static routes used for `VRF_A` ↔ `VRF_B` through pair 100 with OSPF.

We will also use OSPF between R1 and R3, and between R1 and R4, so that **no static routes are needed for the R3-to-R4 path**.

The global-to-`VRF_A` connectivity through **pair 200** remains statically routed and is unaffected by this example.

### Step 1: Remove the Static Routes Replaced by OSPF

Remove the **two static routes across pair 100** from R1:

```text
R1(config)# no ip route vrf VRF_A 4.4.4.4 255.255.255.255 Vasileft100 172.16.100.2
R1(config)# no ip route vrf VRF_B 3.3.3.3 255.255.255.255 Vasiright100 172.16.100.1
```

Remove R1's original static routes to the directly attached loopbacks. OSPF will now provide those routes:

```text
R1(config)# no ip route vrf VRF_A 3.3.3.3 255.255.255.255 10.1.3.3
R1(config)# no ip route vrf VRF_B 4.4.4.4 255.255.255.255 10.1.4.4
```

Remove the corresponding R3 and R4 static routes that pointed to the other router's loopback:

```text
R3(config)# no ip route 4.4.4.4 255.255.255.255 10.1.3.1

R4(config)# no ip route 3.3.3.3 255.255.255.255 10.1.4.1
```

### Step 2: Configure OSPF on R1

Configure an OSPF process in `VRF_A`. It runs on the R3-facing interface and `Vasileft100`:

```text
R1(config)# interface g2
R1(config-if)# ip ospf 10 area 0

R1(config)# interface vasileft 100
R1(config-if)# ip ospf 10 area 0
```

Configure a second OSPF process in `VRF_B`. It runs on the R4-facing interface and `Vasiright100`:

```text
R1(config)# interface g3
R1(config-if)# ip ospf 20 area 0

R1(config)# interface vasilright 100
R1(config-if)# ip ospf 20 area 0
```

Each VRF uses a different OSPF process.

The OSPF area and neighbor parameters must be compatible, however, and each process needs a **unique router ID**.

Both VASI interfaces belong to `172.16.100.0/30`, although each is in a different VRF:

```text
R1 OSPF 10 (VRF_A)                    R1 OSPF 20 (VRF_B)
Vasileft100                           Vasiright100
172.16.100.1/30 <--- VASI pair --->  172.16.100.2/30
```

OSPF Hellos can cross the VASI pair, allowing the separate OSPF processes to form an adjacency.

Each VRF's OSPF process sees the other as a neighbor, using the VASI pair interfaces:

```
R1# show ip ospf 10 neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.16.100.2      0   FULL/  -        00:00:39    172.16.100.2    vasileft100

R1# show ip ospf 20 neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.16.200.1      0   FULL/  -        00:00:36    172.16.100.1    vasiright100
```

### Step 3: Configure OSPF on R3 and R4

On R3, enable OSPF on the R1-facing link and advertise Loopback0:

```text
R3(config)# router ospf 10
R3(config-router)# router-id 3.3.3.3
R3(config-router)# network 10.1.3.0 0.0.0.255 area 0
R3(config-router)# network 3.3.3.3 0.0.0.0 area 0
```

On R4, enable OSPF on the R1-facing link and advertise Loopback0:

```text
R4(config)# router ospf 20
R4(config-router)# router-id 4.4.4.4
R4(config-router)# network 10.1.4.0 0.0.0.255 area 0
R4(config-router)# network 4.4.4.4 0.0.0.0 area 0
```

The resulting OSPF neighbor relationships are:

| Local Router / Process | Interface | Neighbor | Routing Context |
| --- | --- | --- | --- |
| R1, OSPF 10 | `GigabitEthernet2` | R3 | `VRF_A` |
| R1, OSPF 10 | `Vasileft100` | R1, OSPF 20 | `VRF_A` |
| R1, OSPF 20 | `Vasiright100` | R1, OSPF 10 | `VRF_B` |
| R1, OSPF 20 | `GigabitEthernet3` | R4 | `VRF_B` |

Even though both sides of the VASI adjacency belong to **R1**, OSPF treats them as distinct OSPF routers because they run as separate processes with different router IDs.

### Step 4: Verify the OSPF Neighbors and Routes

On R1, verify the adjacencies:

```text
R1# show ip ospf 10 neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.16.100.2      0   FULL/  -        00:00:38    172.16.100.2    vasileft100
3.3.3.3           1   FULL/BDR        00:00:30    10.1.3.3        GigabitEthernet2

R1# show ip ospf 20 neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.16.200.1      0   FULL/  -        00:00:32    172.16.100.1    vasiright100
4.4.4.4           1   FULL/BDR        00:00:30    10.1.4.4        GigabitEthernet3
```

Each process should have two neighbors: one on the physical interface and one across VASI pair `100`.

Check that the destination routes now point through VASI as **OSPF-learned routes**:

```text
R1# show ip route vrf VRF_A ospf
...
      3.0.0.0/32 is subnetted, 1 subnets
O        3.3.3.3 [110/2] via 10.1.3.3, 00:02:33, GigabitEthernet2
      4.0.0.0/32 is subnetted, 1 subnets
O        4.4.4.4 [110/3] via 172.16.100.2, 00:00:25, vasileft100
      10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
O        10.1.4.0/24 [110/2] via 172.16.100.2, 00:00:25, vasileft100

R1# show ip route vrf VRF_B ospf
...
      3.0.0.0/32 is subnetted, 1 subnets
O        3.3.3.3 [110/3] via 172.16.100.1, 00:00:33, vasiright100
      4.0.0.0/32 is subnetted, 1 subnets
O        4.4.4.4 [110/2] via 10.1.4.4, 00:02:27, GigabitEthernet3
      10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
O        10.1.3.0/24 [110/2] via 172.16.100.1, 00:00:33, vasiright100
```

On the end routers, confirm that the remote loopback is also learned through OSPF:

```text
R3# show ip route | i 4.4.4.4
O        4.4.4.4 [110/13] via 10.1.3.1, 00:01:09, Ethernet0/0
```

```
R4# show ip route | i 3.3.3.3
O        3.3.3.3 [110/13] via 10.1.4.1, 00:01:33, Ethernet0/0
```

Finally, test both directions:

```text
R3# ping 4.4.4.4 source l0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 4.4.4.4, timeout is 2 seconds:
Packet sent with a source address of 3.3.3.3 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/3 ms
```

```
R4# ping 3.3.3.3 source l0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 3.3.3.3, timeout is 2 seconds:
Packet sent with a source address of 4.4.4.4 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```

The packet still crosses **VASI pair 100** and undergoes the same two routing lookups. The difference is that OSPF, rather than static routes, now provides the routes that direct traffic through the pair.

OSPF does not require `route-replicate`, VPN route targets, or `redistribute vrf` for this design.

The two OSPF processes exchange their routing information through a **normal OSPF adjacency over the VASI pair**.

> **Note:** This example intentionally leaves pair `200` on static routing. If OSPF were enabled on both VASI pairs and all three routing contexts, additional routes could propagate between the global table and `VRF_B` through `VRF_A`. Use route filtering or a separate routing design when those paths must remain isolated.

## VASI Compared with Route Replication

Both mechanisms can provide connectivity between routing contexts, but they work differently:

| Feature | Route Replication | VASI |
| --- | --- | --- |
| Main function | Copies eligible routes into another routing table | Transfers packets between routing contexts |
| Key configuration | `route-replicate from vrf` | `interface vasileft`, `interface vasiright` |
| Routes required | Replicated routes installed in destination RIB | Static or dynamically learned routes directed toward the VASI interfaces |
| Routing-table indicators | `+` and `&` for replicated routes | Normal routing protocol or static route codes |
| Packet path | Platform-supported forwarding for replicated routes | Internal VASI interface pair, with another lookup on the receiving side |
| Primary benefit | Direct route sharing | Inter-VRF forwarding with supported services such as NAT or firewalls |

**VASI does not replicate routes automatically.** Dynamic routing protocols such as OSPF can form adjacencies across the pair and exchange routes normally. Otherwise, static routes or another routing mechanism must supply the forwarding paths.

## VASI and Network Services

A major benefit of VASI is that packets can pass through **supported IOS XE services** at the boundary between routing contexts.

For example, to apply NAT to traffic traveling from `VRF_A` into `VRF_B`, a design can use:

```text
      VRF_A                   VRF_B
R3 -> R1 G2 -> Vasileft100 -> Vasiright100 -> R1 G3 -> R4
                                    |           |
                               NAT inside    NAT outside
```

In this design, `Vasiright100` might be configured as a NAT inside interface, and R1's `GigabitEthernet3` as a NAT outside interface.

The NAT rule then operates in the `VRF_B` context. This is only a conceptual service placement: **the preceding lab does not configure NAT**, and the appropriate translation and policy configuration must be added separately.

Other supported integrations include zone-based firewall and IPsec-related services.

VASI is a forwarding framework; creating a VASI pair does not automatically enable any of these services.

## Troubleshooting

### VASI Interface or Pair Is Down

Verify that **both numbered interfaces exist** and that their numbers match:

```text
R1# show interfaces Vasileft100
R1# show interfaces Vasiright100
R1# show interfaces Vasileft200
R1# show interfaces Vasiright200
```

Check VRF membership and IP addresses. The left and right interfaces must belong to the intended routing contexts.

### Route Exists but Traffic Does Not Cross VASI

Check the **selected route** and **CEF forwarding entry** on both sides:

```text
R1# show ip route vrf VRF_A 4.4.4.4
R1# show ip cef vrf VRF_A 4.4.4.4 detail
R1# show ip route vrf VRF_B 3.3.3.3
R1# show ip cef vrf VRF_B 3.3.3.3 detail
```

If route replication or a previously configured route has a better administrative distance, the packet might **not** be forwarded into VASI at all. Remove unwanted competing routes or adjust routing policy. In the OSPF example, remove the earlier static routes, whose administrative distance of 1 would otherwise override the OSPF routes (administrative distance 110).

### OSPF Adjacency Does Not Form

Check the two OSPF processes and both VASI interfaces:

```text
R1# show ip ospf vrf VRF_A neighbor
R1# show ip ospf vrf VRF_B neighbor
R1# show ip ospf vrf VRF_A interface Vasileft100
R1# show ip ospf vrf VRF_B interface Vasiright100
```

Confirm that the paired interfaces are operational, have the correct VRF assignments and IP addresses, belong to the same subnet and OSPF area, and use **different OSPF router IDs**. An OSPF neighbor failure across pair `100` prevents dynamic inter-VRF route exchange.

### Return Traffic Fails

Verify the routing in **both** contexts and on the end routers. A route toward the destination is not enough if the far side lacks a route back toward the source.

### Platform and Feature Restrictions

Cisco documents VASI support and limitations that depend on the platform and feature combination:

- **MPLS traffic over VASI interfaces is not supported.**
- Queue-based QoS actions such as `bandwidth`, `priority`, `shape`, and `fair-queue` are not supported on VASI interfaces.
- VASI supports static routing and supported dynamic routing protocols, including OSPF, EIGRP, and BGP.
- Some multicast and high-scale VASI combinations have additional limitations.

## Key Points

- **VASI** creates a virtual forwarding path between two routing contexts on the **same router**.
- `Vasileft` and `Vasiright` interfaces are **paired automatically by matching numbers**.
- The interfaces are symmetrical, so traffic can cross in either direction.
- Each side of the pair belongs to its own routing context. **One side can use the global routing table** by omitting `vrf forwarding`.
- VASI **does not replicate routes**. Use static routing or a supported dynamic routing protocol across the paired interfaces.
- **Dynamic routing protocols can form adjacencies/peerings across a VASI pair**, exchange routes between the routing contexts, and eliminate the need for the corresponding inter-VRF static routes.
- Packets crossing VASI undergo **two routing lookups**, one in each context, and normally incur **two TTL decrements** on the router.
- VASI can provide the forwarding path for inter-VRF **NAT, firewall, and other supported services**.
- Both **return routing** and the original routes to local destinations must work for end-to-end connectivity.

## References

- [Cisco: Configuring the VRF-Aware Software Infrastructure](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_conf-vasi.html)
- [Cisco: Configure VRF-Aware Software Infrastructure NAT on IOS XE](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/200255-Configure-VRF-Aware-Software-Infrastruct.html)
- [Cisco: Configuring the VRF-Aware Service Infrastructure](https://www.cisco.com/en/US/docs/ios/ios_xe/sec_data_plane/configuration/guide/sec_vasi_vrf_aware_software_infrastructure_ps9587_TSD_Products_Configuration_Guide_Chapter.html)
