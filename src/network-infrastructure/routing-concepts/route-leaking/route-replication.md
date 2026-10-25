# Route Replication

**Route replication** allows a router to make routes from one VRF available in another VRF or in the global routing table, using the Routing Information Base (RIB) rather than BGP VPN route import/export.

It is useful when separate VRFs need access to shared networks, but the router does not need an additional BGP process, RDs, or route targets to exchange the routes.

Route replication is configured with `route-replicate from vrf`. The command identifies the **source VRF** and is applied in the routing context that must **receive** the routes.

## Operation

Consider two VRFs on R1:

```text
VRF_A                            VRF_B
3.3.3.3/32                       4.4.4.4/32
    |                                |
    |       Route replication        |
    +------------> R1 <--------------+
    |                                |
    v                                v
VRF_B receives 3.3.3.3/32       VRF_A receives 4.4.4.4/32
```

Without route replication, a route learned in `VRF_A` is not normally available in `VRF_B`.

With route replication:

- R1 can make selected routes from `VRF_A` available for forwarding in `VRF_B`.
- R1 can independently make selected routes from `VRF_B` available in `VRF_A`.
- The VRFs remain separate. The route-replication policy determines which routes can be shared.
- No BGP VPNv4 address family, route distinguisher (RD), or route target (RT) is required.

Route replication provides access to the **source VRF's routing information** through a link to its RIB. It is not equivalent to building independent copies of all source routes. It does not require a new routing-protocol adjacency, and it does not originate new routes in the source VRF.

**Replicated routes retain the source route's protocol and administrative distance.** For example, a replicated OSPF route is still displayed as an OSPF route, rather than being converted into a static route.

> **Important:** Route replication is local to the router. It does not automatically advertise the replicated routes to other routers. That requires an appropriate routing-protocol configuration and, when necessary, inter-VRF redistribution.

## Configuration Syntax

Under the **destination VRF** address family:

```text
R1(config)# vrf definition <destination-vrf>
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# route-replicate from vrf <source-vrf> unicast <protocol> [route-map <name>]
```

The important arguments are:

| Argument | Purpose |
| --- | --- |
| `from vrf <source-vrf>` | Identifies the routing table from which routes are selected |
| `unicast` | Selects the unicast routing topology |
| `<protocol>` | Selects routes according to their source routing protocol |
| `route-map <name>` | Optionally filters the routes eligible for replication |

Common protocol selectors include:

| Selector | Routes Selected |
| --- | --- |
| `connected` | Connected routes |
| `static` | Static routes |
| `ospf <process-id>` | Routes learned by the specified OSPF process |
| `eigrp <as-number>` | Routes learned by the specified EIGRP autonomous system |
| `bgp <as-number>` | BGP routes, where supported |
| `all` | All supported source route types |

BGP route replication is supported on platforms and releases that implement the newer BGP route-leaking capability. Cisco introduced that support in IOS XE 17.6.1. Other supported protocol keywords depend on the platform and feature implementation.

For example, to replicate static routes from `VRF_A` into `VRF_B`:

```text
R1(config)# vrf definition VRF_B
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# route-replicate from vrf VRF_A unicast static
```

To replicate OSPF routes learned by process 10:

```text
R1(config-vrf-af)# route-replicate from vrf VRF_A unicast ospf 10
```

To replicate all supported routes:

```text
R1(config-vrf-af)# route-replicate from vrf VRF_A unicast all
```

The `all` option includes connected routes. It is broad, so **specific protocol selectors and route-map filters are generally preferable** when only selected prefixes need to cross VRF boundaries. The `route-replicate` syntax also offers a `multicast` topology option for supported multicast RPF use cases; the configurations below use `unicast`.

## VRF-to-VRF Configuration Example

R1 connects `VRF_A` and `VRF_B`. It runs **OSPF** with R3 in `VRF_A` and **EIGRP** with R4 in `VRF_B`:

```text
        R3                               R4
    Lo0: 3.3.3.3/32                 Lo0: 4.4.4.4/32
       OSPF 10                         EIGRP 100
      E0/0                               E0/0
   10.1.3.3/24                        10.1.4.4/24
        |                                  |
        |                                  |
   10.1.3.1/24                        10.1.4.1/24
      E0/1                               E0/2
     VRF_A ------------ R1 ------------ VRF_B
```

**Goal:** Replicate R3's OSPF-learned loopback route into `VRF_B` and R4's EIGRP-learned loopback route into `VRF_A`. Then advertise those routes to R4 and R3, respectively, so the loopbacks can communicate without static routes.

| Prefix | Source Router | Routing Protocol on R1 | Original VRF | Destination VRF |
| --- | --- | --- | --- | --- |
| `3.3.3.3/32` | R3 | OSPF 10 | `VRF_A` | `VRF_B` |
| `4.4.4.4/32` | R4 | EIGRP 100 | `VRF_B` | `VRF_A` |

### Step 1: Configure VRFs and Interfaces on R1

```text
R1(config)# vrf definition VRF_A
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# exit-address-family

R1(config)# vrf definition VRF_B
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# exit-address-family

R1(config)# interface Ethernet0/1
R1(config-if)# vrf forwarding VRF_A
R1(config-if)# ip address 10.1.3.1 255.255.255.0
R1(config-if)# no shutdown

R1(config)# interface Ethernet0/2
R1(config-if)# vrf forwarding VRF_B
R1(config-if)# ip address 10.1.4.1 255.255.255.0
R1(config-if)# no shutdown
```

### Step 2: Configure OSPF Between R1 and R3

On R3, configure the link and loopback:

```text
R3(config)# interface Ethernet0/0
R3(config-if)# ip address 10.1.3.3 255.255.255.0
R3(config-if)# no shutdown

R3(config)# interface Loopback0
R3(config-if)# ip address 3.3.3.3 255.255.255.255

R3(config)# interface Loopback1
R3(config-if)# ip address 33.33.33.33 255.255.255.255

R3(config)# router ospf 10
R3(config-router)# router-id 3.3.3.3
R3(config-router)# network 10.1.3.0 0.0.0.255 area 0
R3(config-router)# network 3.3.3.3 0.0.0.0 area 0
R3(config-router)# network 33.33.33.33 0.0.0.0 area 0
```

On R1, enable OSPF in `VRF_A`:

```text
R1(config)# router ospf 10 vrf VRF_A
R1(config-router)# router-id 1.1.1.1
R1(config-router)# network 10.1.3.0 0.0.0.255 area 0
```

Verify that R1 learns `3.3.3.3/32` and `33.33.33.33/32` as OSPF routes:

```text
R1# show ip ospf neighbor          

Neighbor ID     Pri   State           Dead Time   Address         Interface
3.3.3.3           0   FULL/  -        00:00:37    10.1.3.3        Ethernet0/1

R1# show ip route vrf VRF_A ospf
...
      3.0.0.0/32 is subnetted, 1 subnets
O        3.3.3.3 [110/11] via 10.1.3.3, 00:03:12, Ethernet0/1
      33.0.0.0/32 is subnetted, 1 subnets
O        33.33.33.33 [110/11] via 10.1.3.3, 00:00:16, Ethernet0/1
```

### Step 3: Configure EIGRP Between R1 and R4

On R4, configure the link, loopback, and EIGRP named mode:

```text
R4(config)# interface Ethernet0/0
R4(config-if)# ip address 10.1.4.4 255.255.255.0
R4(config-if)# no shutdown

R4(config)# interface Loopback0
R4(config-if)# ip address 4.4.4.4 255.255.255.255

R4(config)# interface Loopback1
R4(config-if)# ip address 44.44.44.44 255.255.255.255

R4(config)# router eigrp EIGRP
R4(config-router)# address-family ipv4 unicast autonomous-system 100
R4(config-router-af)# eigrp router-id 4.4.4.4
R4(config-router-af)# network 10.1.4.0 0.0.0.255
R4(config-router-af)# network 4.4.4.4 0.0.0.0
R4(config-router-af)# network 44.44.44.44 0.0.0.0
```

On R1, configure EIGRP within `VRF_B`:

```text
R1(config)# router eigrp EIGRP
R1(config-router)# address-family ipv4 unicast vrf VRF_B autonomous-system 100
R1(config-router-af)# eigrp router-id 1.1.1.1
R1(config-router-af)# network 10.1.4.0 0.0.0.255
```

Verify that R1 learns `4.4.4.4/32` and `44.44.44.44/32` as internal EIGRP routes:

```text
R1# show ip eigrp vrf VRF_B neighbors 
EIGRP-IPv4 VR(EIGRP) Address-Family Neighbors for AS(100)
           VRF(VRF_B)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
                                                   (sec)         (ms)       Cnt Num
0   10.1.4.4                Et0/2                    11 00:00:18    7   100  0  2

R1# show ip route vrf VRF_B eigrp
...
      4.0.0.0/32 is subnetted, 1 subnets
D        4.4.4.4 [90/1024640] via 10.1.4.4, 00:03:34, Ethernet0/2
      44.0.0.0/32 is subnetted, 1 subnets
D        44.44.44.44 [90/1024640] via 10.1.4.4, 00:02:14, Ethernet0/2
```

At this stage, R1 knows both loopback routes, but each exists only in its **original VRF**.

### Step 4: Configure Prefix Lists and Route Maps

We'll restrict the route replication to R3 and R4's `Loopback0` prefixes only.

```text
R1(config)# ip prefix-list A_TO_B permit 3.3.3.3/32
R1(config)# ip prefix-list B_TO_A permit 4.4.4.4/32

R1(config)# route-map A_TO_B permit 10
R1(config-route-map)# match ip address prefix-list A_TO_B

R1(config)# route-map B_TO_A permit 10
R1(config-route-map)# match ip address prefix-list B_TO_A
```

- `A_TO_B` permits only R3's loopback.
- `B_TO_A` permits only R4's loopback.

The route maps will be used to limit **route replication**, so only the selected prefixes cross the routing-table boundary and are advertised to the other router.

### Step 5: Configure Bidirectional Route Replication

Replicate the **OSPF** route from `VRF_A` into `VRF_B`:

```text
R1(config)# vrf definition VRF_B
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# route-replicate from vrf VRF_A unicast ospf 10 route-map A_TO_B
```

Replicate the **EIGRP** route from `VRF_B` into `VRF_A`:

```text
R1(config)# vrf definition VRF_A
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# route-replicate from vrf VRF_B unicast eigrp 100 route-map B_TO_A
```

| Configured Under | Source VRF | Protocol Selector | Replicated Prefix |
| --- | --- | --- | --- |
| `VRF_B` | `VRF_A` | `ospf 10` | `3.3.3.3/32` |
| `VRF_A` | `VRF_B` | `eigrp 100` | `4.4.4.4/32` |

R1 now has the following routing information:

```text
R1# show ip route vrf VRF_A      
...
      3.0.0.0/32 is subnetted, 1 subnets
O        3.3.3.3 [110/11] via 10.1.3.3, 00:08:46, Ethernet0/1
      4.0.0.0/32 is subnetted, 1 subnets
D   +    4.4.4.4 [90/1024640] via 10.1.4.4 (VRF_B), 00:07:03, Ethernet0/2
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.1.3.0/24 is directly connected, Ethernet0/1
L        10.1.3.1/32 is directly connected, Ethernet0/1
      33.0.0.0/32 is subnetted, 1 subnets
O        33.33.33.33 [110/11] via 10.1.3.3, 00:05:50, Ethernet0/1

R1# show ip route vrf VRF_B     
...
      3.0.0.0/32 is subnetted, 1 subnets
O   +    3.3.3.3 [110/11] via 10.1.3.3 (VRF_A), 00:08:49, Ethernet0/1
      4.0.0.0/32 is subnetted, 1 subnets
D        4.4.4.4 [90/1024640] via 10.1.4.4, 00:07:06, Ethernet0/2
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.1.4.0/24 is directly connected, Ethernet0/2
L        10.1.4.1/32 is directly connected, Ethernet0/2
      44.0.0.0/32 is subnetted, 1 subnets
D        44.44.44.44 [90/1024640] via 10.1.4.4, 00:05:46, Ethernet0/2
```

The `+` identifies a replicated route. The source VRF's protocol code is retained, so an OSPF route appears as `O +` and an EIGRP route appears as `D +`.

**R1 can use the replicated routes for inter-VRF forwarding, but R3 and R4 have not yet learned the remote loopbacks.**

Route replication does not automatically redistribute routes into OSPF or EIGRP.

### Step 6: Redistribute the Replicated Routes

R3 needs to learn `4.4.4.4/32` through OSPF. On R1, redistribute the selected EIGRP route from `VRF_B` into OSPF process 10 in `VRF_A`:

```text
R1(config)# router ospf 10 vrf VRF_A
R1(config-router)# redistribute vrf VRF_B eigrp 100
```

R4 needs to learn `3.3.3.3/32` through EIGRP. On R1, redistribute the selected OSPF route from `VRF_A` into EIGRP AS 100 in `VRF_B`:

```text
R1(config)# router eigrp EIGRP
R1(config-router)# address-family ipv4 unicast vrf VRF_B autonomous-system 100
R1(config-router-af)# topology base
R1(config-router-af-topology)# redistribute vrf VRF_A ospf 10 metric 10000 10 255 1 1500
```

The two operations are different:

1. `route-replicate` provides **local inter-VRF routing on R1**.
2. `redistribute vrf` makes the selected source routes available to a **routing protocol running in the destination VRF**, so R1 can advertise them to R3 or R4.

### Step 7: Verify Routing and Connectivity

We have already verified the routes on R1. Let's verify on R3 and R4:

```
R3# show ip route ospf
...
      4.0.0.0/32 is subnetted, 1 subnets
O E2     4.4.4.4 [110/20] via 10.1.3.1, 00:05:06, Ethernet0/0
```

```
R4# show ip route eigrp
...
      3.0.0.0/32 is subnetted, 1 subnets
D EX     3.3.3.3 [170/1075200] via 10.1.4.1, 00:04:10, Ethernet0/0
```

| Device | Expected Route Code | Prefix | Reason |
| --- | --- | --- | --- |
| R3 | `O E2` | `4.4.4.4/32` | R1 redistributed EIGRP into OSPF |
| R4 | `D EX` | `3.3.3.3/32` | R1 redistributed OSPF into EIGRP |

The routing-protocol external codes on R3 and R4 are **different from the replicated route codes on R1**.

R1's replicated routes retain the original protocol identity; redistribution originates new external routes in the other protocol.

Finally, test loopback-to-loopback communication:

```text
R3# ping 4.4.4.4 source l0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 4.4.4.4, timeout is 2 seconds:
Packet sent with a source address of 3.3.3.3 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```

```
R4# ping 3.3.3.3 source l0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 3.3.3.3, timeout is 2 seconds:
Packet sent with a source address of 4.4.4.4 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```

The forwarding paths are:

```text
R3 (3.3.3.3) -> R1 VRF_A -> R1 VRF_B -> R4 (4.4.4.4)
R4 (4.4.4.4) -> R1 VRF_B -> R1 VRF_A -> R3 (3.3.3.3)
```

This example deliberately leaks **only the two loopback prefixes**.

Replicating the connected transit networks is not necessary for this traffic because the original next hops are reachable in their source VRFs.

## Replicating connected routes

You can replicate connected routes from another VRF if necessary.

For example, let's replicate VRF_B's connected interface into VRF_A:

```
R1(config)# vrf definition VRF_A
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)#route-replicate from vrf VRF_B unicast connected 
```

Here's how the replicated routes look in VRF A's routing table:

```
R1#show ip route vrf VRF_A           
...
      3.0.0.0/32 is subnetted, 1 subnets
O        3.3.3.3 [110/11] via 10.1.3.3, 00:22:47, Ethernet0/1
      4.0.0.0/32 is subnetted, 1 subnets
D   +    4.4.4.4 [90/1024640] via 10.1.4.4 (VRF_B), 00:21:04, Ethernet0/2
      10.0.0.0/8 is variably subnetted, 4 subnets, 2 masks
C        10.1.3.0/24 is directly connected, Ethernet0/1
L        10.1.3.1/32 is directly connected, Ethernet0/1
C   +    10.1.4.0/24 is directly connected, Ethernet0/2
L   &    10.1.4.1/32 is directly connected, Ethernet0/2
      33.0.0.0/32 is subnetted, 1 subnets
O        33.33.33.33 [110/11] via 10.1.3.3, 00:19:51, Ethernet0/1
```

I already mentioned that `+` indicates a replicated route.

Here's how it differs from the `&` next to the replicated local route:

- `+` = **Replicated route**. The route was copied from another VRF using `route-replicate`.
- `&` = **Replicated local route overridden by a connected route** in the receiving VRF.

In this example:

- `D + 4.4.4.4/32`: EIGRP route replicated from `VRF_B`.
- `C + 10.1.4.0/24`: Connected route replicated from `VRF_B`.
- `L & 10.1.4.1/32`: Local host route replicated from `VRF_B`, but overridden by the connected route for that interface.

The `&` symbol does not mean that Ethernet0/2 belongs to `VRF_A`. The interface still belongs to `VRF_B`.

And like previously, simply replicating a route to the local routing table doesn't mean that R1 will advertise it to its neighbor, R2.

Let's redistribute it:

```
R1(config-vrf-af)# router ospf 10
R1(config-router)# redistribute vrf VRF_B connected 
```

R3 has now learned two replicated routes:

```
      4.0.0.0/32 is subnetted, 1 subnets
O E2     4.4.4.4 [110/20] via 10.1.3.1, 00:13:57, Ethernet0/0
      10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
O E2     10.1.4.0/24 [110/20] via 10.1.3.1, 00:07:02, Ethernet0/0
```

For reference, here's how to configure the same toward R4 in VRF_B:

```
R1(config)# vrf definition VRF_B
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# route-replicate from vrf VRF_A unicast connected

R1(config)# router eigrp EIGRP
R1(config-router)# address-family ipv4 unicast vrf VRF_B autonomous-system 100
R1(config-router-af)# topology base
R1(config-router-af-topology)# redistribute vrf VRF_A connected
```

R1's routing table:

```
R1# show ip route vrf VRF_B
...
      3.0.0.0/32 is subnetted, 1 subnets
O   +    3.3.3.3 [110/11] via 10.1.3.3 (VRF_A), 00:29:41, Ethernet0/1
      4.0.0.0/32 is subnetted, 1 subnets
D        4.4.4.4 [90/1024640] via 10.1.4.4, 00:27:58, Ethernet0/2
      10.0.0.0/8 is variably subnetted, 4 subnets, 2 masks
C   +    10.1.3.0/24 is directly connected, Ethernet0/1
L   &    10.1.3.1/32 is directly connected, Ethernet0/1
C        10.1.4.0/24 is directly connected, Ethernet0/2
L        10.1.4.1/32 is directly connected, Ethernet0/2
      44.0.0.0/32 is subnetted, 1 subnets
D        44.44.44.44 [90/1024640] via 10.1.4.4, 00:26:38, Ethernet0/2
```

R4 has learned the connected route now:

```
R4# show ip route eigrp
...
      3.0.0.0/32 is subnetted, 1 subnets
D EX     3.3.3.3 [170/1075200] via 10.1.4.1, 00:15:10, Ethernet0/0
      10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
D EX     10.1.3.0/24 [170/1536000] via 10.1.4.1, 00:00:44, Ethernet0/0
```

## Replication Between a VRF and the Global Routing Table

Route replication also supports routes crossing the boundary between a named VRF and the **global routing table (GRT)**.

### Global to VRF

To make global static routes available in `VRF_A`:

```text
R1(config)# vrf definition VRF_A
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# route-replicate from vrf global unicast static
```

The keyword `global` identifies the global routing table as the source.

### VRF to Global

To make connected routes from `VRF_A` available in the global routing table, use `global-address-family`:

```text
R1(config)# global-address-family ipv4 unicast
R1(config-af)# route-replicate from vrf VRF_A unicast connected
```

Here the global routing table is the **destination**, and `VRF_A` is the source.

| Direction | Configuration Context | Example Command |
| --- | --- | --- |
| VRF to VRF | Destination VRF address family | `route-replicate from vrf VRF_A unicast static` |
| Global to VRF | Destination VRF address family | `route-replicate from vrf global unicast static` |
| VRF to Global | `global-address-family ipv4 unicast` | `route-replicate from vrf VRF_A unicast connected` |

Use selective replication when exchanging routes with the global table. In particular, avoid broad `unicast all` replication into the global table when it could reintroduce routes and create routing loops.

### Example

As an example of route replication between a VRF and the global routing table, let's enable communication between R2's Loopback0 (`2.2.2.2/32`)
and R4's Loopback1 (`44.44.44.44/32`).

Here's the full topology from an earlier page:

```text
                     R2
               Lo0: 2.2.2.2/32
                      |
                E0/0 (Global)
                      |
R3 ----- E0/1 ------- R1 ------- E0/2 ----- R4
Lo0:     (VRF_A)                 (VRF_B)     Lo0:
3.3.3.3/32                               4.4.4.4/32
```

**Goal:** Replicate R2's OSPF route into `VRF_B` and R4's EIGRP route into the global routing table, then redistribute the routes so R2 and R4 can communicate.

#### Step 1: Configure Routing

Configure R2's interfaces and OSPF process:

```text
R2(config)# interface Ethernet0/0
R2(config-if)# ip address 10.1.2.2 255.255.255.0
R2(config-if)# no shutdown

R2(config)# interface Loopback0
R2(config-if)# ip address 2.2.2.2 255.255.255.255

R2(config)# router ospf 20
R2(config-router)# router-id 2.2.2.2
R2(config-router)# network 10.1.2.0 0.0.0.255 area 0
R2(config-router)# network 2.2.2.2 0.0.0.0 area 0
```

Configure R1's global interface and OSPF process:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip address 10.1.2.1 255.255.255.0
R1(config-if)# no shutdown

R1(config)# router ospf 20
R1(config-router)# router-id 1.1.1.20
R1(config-router)# network 10.1.2.0 0.0.0.255 area 0
```

> R4's EIGRP process is already configured as shown previously.

R1 should now have these routes in separate routing tables:

| Routing Table | Prefix | Protocol |
| --- | --- | --- |
| Global | `2.2.2.2/32` | OSPF 20 |
| `VRF_B` | `44.44.44.44/32` | EIGRP 100 |

```
R1# show ip route | i 2.2.2.2
O        2.2.2.2 [110/11] via 10.1.2.2, 00:00:32, Ethernet0/0

R1# show ip route vrf VRF_B | i 44.44.44.44
D        44.44.44.44 [90/1024640] via 10.1.4.4, 00:01:44, Ethernet0/2
```

#### Step 2: Configure Route Maps

Create prefix lists and route maps to permit only the two loopback routes:

```text
R1(config)# ip prefix-list GLOBAL_TO_B permit 2.2.2.2/32
R1(config)# ip prefix-list B_TO_GLOBAL permit 44.44.44.44/32

R1(config)# route-map GLOBAL_TO_B permit 10
R1(config-route-map)# match ip address prefix-list GLOBAL_TO_B

R1(config)# route-map B_TO_GLOBAL permit 10
R1(config-route-map)# match ip address prefix-list B_TO_GLOBAL
```

#### Step 3: Configure Route Replication

First, replicate the OSPF route from the global routing table into `VRF_B`:

```text
R1(config)# vrf definition VRF_B
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# route-replicate from vrf global unicast ospf 20 route-map GLOBAL_TO_B
```

Next, replicate the EIGRP route from `VRF_B` into the global routing table:

```text
R1(config)# global-address-family ipv4 unicast
R1(config-af)# route-replicate from vrf VRF_B unicast eigrp 100 route-map B_TO_GLOBAL
```

Notice the difference:

- Global to VRF: Configure `route-replicate` under the destination VRF's address family, specifying `global` as the source.
- VRF to Global: Configure `route-replicate` under `global-address-family`, specifying the source VRF.

Verify the replicated routes:

```text
R1# show ip route vrf VRF_B | i 2.2.2.2
O   +    2.2.2.2 [110/11] via 10.1.2.2, 00:03:23, Ethernet0/0

R1# show ip route | i 44.44.44.44
D   +    44.44.44.44 [90/1024640] via 10.1.4.4 (VRF_B), 00:04:39, Ethernet0/2
```

The `+` indicates that the routes were replicated from another routing table.

R1 now has the routes necessary for inter-VRF forwarding. However, R2 and R4 do not yet know how to reach each other's loopbacks.

#### Step 4: Redistribute the Replicated Routes

Redistribute R4's EIGRP route into the global OSPF process:

```text
R1(config)# router ospf 20
R1(config-router)# redistribute vrf VRF_B eigrp 100
```

Redistribute R2's OSPF route into EIGRP in `VRF_B`:

```text
R1(config)# router eigrp EIGRP
R1(config-router)# address-family ipv4 unicast vrf VRF_B autonomous-system 100
R1(config-router-af)# topology base
R1(config-router-af-topology)# redistribute vrf global ospf 20 metric 10000 10 255 1 1500
```

The existing route maps on the replication commands already select the two loopback routes, so additional route maps are not necessary.

#### Step 5: Verify Connectivity

On R2, verify the route to R4's Loopback1:

```text
R2# show ip route | i 44.44.44.44
O E2     44.44.44.44 [110/20] via 10.1.2.1, 00:00:52, Ethernet0/0
```

On R4, verify the route to R2's Loopback0:

```text
R4# show ip route | i 2.2.2.2
D EX     2.2.2.2 [170/1075200] via 10.1.4.1, 00:00:56, Ethernet0/0
```

Finally, test connectivity between the loopbacks:

```text
R2# ping 44.44.44.44 source l0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 44.44.44.44, timeout is 2 seconds:
Packet sent with a source address of 2.2.2.2 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```

```text
R4# ping 2.2.2.2 source l1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2.2.2.2, timeout is 2 seconds:
Packet sent with a source address of 44.44.44.44 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/3 ms
```

The resulting forwarding paths are:

```text
R2 -> R1 Global -> R1 VRF_B -> R4
R4 -> R1 VRF_B -> R1 Global -> R2
```

This example demonstrates that route replication works in both directions between a named VRF and the global routing table, using different routing protocols and selective route maps.


## Route Table Symbols

IOS XE displays special markers for routes made available through route replication:

| Symbol | Meaning |
| --- | --- |
| `+` | Replicated route |
| `&` | Replicated local route overridden by a connected route |
| `(VRF_A)` | The indicated next hop belongs to source VRF `VRF_A` |

The OSPF/EIGRP example produces these types of entries in the receiving VRFs:

```text
D + 4.4.4.4/32 via 10.1.4.4 (VRF_B)
O + 3.3.3.3/32 via 10.1.3.3 (VRF_A)
```

The protocol codes are unchanged by replication: `D` still represents EIGRP and `O` still represents OSPF. The `+` marker distinguishes the replicated entries from native routes.

Replicating **connected** routes can also produce this type of output:

```text
C   +    10.1.3.0/24 is directly connected, Ethernet0/1
L   &    10.1.3.1/32 is directly connected, Ethernet0/1
```

- `C +` identifies the replicated connected network.
- `L &` identifies the associated replicated **local /32** entry that is overridden by a connected route. 
- Ordinary `C` and `L` routes without replication markers are native routes in the displayed VRF.

The connected-route example is separate from the main lab, which replicates only the OSPF- and EIGRP-learned loopback routes.

## Route Preference

A replicated route retains its original **administrative distance** and **routing-protocol identity**. The destination routing table can also have its own route to the same prefix.

Route preference follows these rules:

1. If competing routes belong to the same protocol and source VRF, prefer the nonreplicated route.
2. Otherwise, prefer the lower administrative distance.
3. If the distances are equal, prefer the lower default administrative distance.
4. If there is still a tie, prefer the nonreplicated route.
5. Remaining ties can use the original VRF name, address-family type, and route age.

For example, when the destination has a native OSPF route and a replicated static route to the same prefix, the lower administrative distance of the static route generally wins, assuming both are otherwise eligible.

Route replication does not guarantee that every eligible source route will become the selected route in the destination context.

## Replication vs. Redistribution

**Route replication and routing-protocol redistribution are separate operations.**

- `route-replicate` makes selected routes from another routing context available **locally**.
- `redistribute vrf` injects selected routes from a source VRF into a **routing protocol** in the destination routing context, allowing them to be advertised to remote neighbors.

The OSPF/EIGRP lab demonstrates both operations for the same prefix:

```text
R1(config)# vrf definition VRF_B
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# route-replicate from vrf VRF_A unicast ospf 10 route-map A_TO_B

R1(config)# router eigrp EIGRP
R1(config-router)# address-family ipv4 unicast vrf VRF_B autonomous-system 100
R1(config-router-af)# topology base
R1(config-router-af-topology)# redistribute vrf VRF_A ospf 10 metric 10000 10 255 1 1500 route-map A_TO_B
```

The first command makes `3.3.3.3/32` available for forwarding in R1's `VRF_B`. The second redistributes that OSPF route into EIGRP so R4 can learn it as an **EIGRP external** route.

Ordinary `redistribute ospf 10` under the EIGRP process in `VRF_B` is **not a substitute for** `redistribute vrf VRF_A ospf 10` when the original OSPF process belongs to `VRF_A`.

Do not redistribute a route back into its source domain indiscriminately. Use filters to avoid feedback and routing loops.

## Verification and Troubleshooting

When replication is not working, check:

1. **Direction:** `route-replicate` must be configured in the **receiving VRF**, or under `global-address-family` when the global table is the receiver.
2. **Source route:** The intended prefix must exist as an eligible route in the source routing table.
3. **Protocol selector:** `static`, `connected`, `ospf <id>`, and other keywords must match the actual source route.
4. **Route map:** An unmatched prefix is not replicated.
5. **Route preference:** A native or otherwise preferred route may be selected instead.
6. **Forwarding:** Confirm the CEF entry and the path through the source VRF. Route-table visibility alone is not proof of successful forwarding.
7. **Return path:** Both endpoints must be able to route replies back through R1.
8. **Redistribution:** If remote routers need the route, configure `redistribute vrf` under the appropriate routing process and verify its advertisements (`O E2` or `D EX` in the example).
9. **Loops and overlapping addresses:** Avoid reintroducing routes into their source routing domains or importing ambiguous overlapping prefixes into the same destination table.

## Key Points

- **Route replication** makes eligible routes available across VRF or global routing-table boundaries without requiring BGP VPN import/export.
- Configure `route-replicate from vrf` in the **destination** routing context. The named VRF is the **source**.
- Use `global` as the source name to replicate global routes into a VRF, and `global-address-family` to replicate VRF routes into the global table.
- Routes can be selected by their **source protocol** and optionally filtered with a **route map**.
- Replicated routes retain their **source protocol** and **administrative distance**.
- `+` identifies a replicated route; `&` identifies a replicated local route overridden by a connected route.
- Bidirectional connectivity generally requires routes in **both directions**, plus valid forwarding and return paths.
- `route-replicate` **does not automatically advertise routes to other routers**. In the OSPF/EIGRP example, `redistribute vrf` makes the replicated prefixes available to R3 and R4 as external routes.
- Avoid indiscriminate replication and redistribution that can create routing loops.

## References

- [Cisco: Configure Route Leaking with Route Replication Feature](https://www.cisco.com/c/en/us/support/docs/ip/interior-gateway-protocol-igp/226098-configure-route-leaking-with-route.html)
- [Cisco: Configuring Easy Virtual Network Shared Services](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-addressing/ip-addressing/m_evn-shared-svcs.html)
- [Cisco: Configuring Route Leaking and Redistribution](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m-configure-route-leaking-and-redistribution.html)
- [Cisco: Configure EIGRP Named Mode](https://www.cisco.com/c/en/us/support/docs/ip/enhanced-interior-gateway-routing-protocol-eigrp/200156-Configure-EIGRP-Named-Mode.html)
- [Cisco: Configure Easy Virtual Network](https://www.cisco.com/c/en/us/support/docs/ip/ip-routing/117974-configure-evn-00.html)
