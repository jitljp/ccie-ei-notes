# VRF-Aware BGP

BGP can run inside a VRF using a **VRF-specific address family**.

In IOS XE, a single BGP process can support multiple VRFs. Each VRF has its own BGP routing context, neighbors, and routes.

For example:

```text
router bgp 65000
 address-family ipv4 unicast vrf CUSTOMER_A
 address-family ipv4 unicast vrf CUSTOMER_B
```

BGP routes learned from a neighbor in `CUSTOMER_A` are associated with `CUSTOMER_A`, not with the global routing table or `CUSTOMER_B`.

## Main Idea

VRF-aware BGP lets a router form BGP sessions using interfaces assigned to different VRFs.

```text
R1 (AS 65000)
├── Global IPv4 unicast BGP address family
├── IPv4 unicast VRF CUSTOMER_A
│   └── EBGP neighbor CE1 (AS 65100)
└── IPv4 unicast VRF CUSTOMER_B
    └── EBGP neighbor CE2 (AS 65200)
```

BGP uses **one local AS number** for the `router bgp` process, but each VRF address family can have different neighbors and remote AS numbers.

The separate VRF address families have their own:

- BGP neighbors and routes
- Network statements and redistribution configuration
- Per-neighbor routing policies
- Next-hop reachability checks within the VRF

**Configuring BGP in one VRF does not automatically advertise routes to another VRF.**

## VRF-Specific BGP Address Families

For IPv4, use:

```text
router bgp 65000
 address-family ipv4 unicast vrf CUSTOMER_A
```

For IPv6, use:

```text
router bgp 65000
 address-family ipv6 unicast vrf CUSTOMER_A
```

The `vrf` keyword associates the address family with a particular VRF.

Compare these two address families:

```text
router bgp 65000
 address-family ipv4 unicast
  network 172.16.1.0 mask 255.255.255.0
 !
 address-family ipv4 unicast vrf CUSTOMER_A
  network 192.168.1.0 mask 255.255.255.0
```

- `address-family ipv4 unicast` is for the **global IPv4 routing context**.
- `address-family ipv4 unicast vrf CUSTOMER_A` is for the **CUSTOMER_A IPv4 routing context**.

The same prefix can exist in the BGP tables of different VRFs without conflicting.

> Basic VRF-aware IPv4 BGP exchanges ordinary IPv4 unicast routes with its neighbors. It does **not** require MPLS, LDP, or an MP-BGP VPNv4 session between routers. RD and RT configuration is not required merely to establish basic VRF-Lite BGP peering.

## Lab Topology

This page uses the same overlapping-address topology as the other VRF-aware routing pages.

```text
                    R1 (AS 65000)
        +--------------------------------+
        | L0: 1.1.1.1/24  L1: 1.1.1.1/24 |
        | E0/0             E0/1          |
        | CUSTOMER_A       CUSTOMER_B    |
        | 10.10.10.1/24    10.10.10.1/24 |
        +-----+------------------+-------+
              |                  |
         EBGP |                  | EBGP
              |                  |
        10.10.10.2/24      10.10.10.2/24
              CE1                CE2
           AS 65100           AS 65200
              |                  |
        192.168.1.0/24    192.168.1.0/24
```

| Item | CUSTOMER_A | CUSTOMER_B |
|---|---|---|
| R1 interface | Ethernet0/0 | Ethernet0/1 |
| R1 interface IP | 10.10.10.1/24 | 10.10.10.1/24 |
| R1 loopback | Loopback0, 1.1.1.1/24 | Loopback1, 1.1.1.1/24 |
| CE router | CE1 | CE2 |
| CE interface IP | 10.10.10.2/24 | 10.10.10.2/24 |
| CE LAN | 192.168.1.0/24 | 192.168.1.0/24 |
| CE BGP AS | 65100 | 65200 |

R1 has two independent EBGP peers with **the same neighbor IP address**, `10.10.10.2`.

This is possible because the peers belong to different VRFs.

The CE routers are not using VRFs. CE1 and CE2 are on separate physical links and have separate routing tables.

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

Verify the interfaces and connected routes.

```text
R1# show vrf
R1# show ip route vrf CUSTOMER_A connected
R1# show ip route vrf CUSTOMER_B connected
```

Each VRF should have its own connected routes for `10.10.10.0/24` and `1.1.1.0/24`.

## CE Interface Configuration

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

Each CE has a connected route to its LAN, `192.168.1.0/24`.

## R1 BGP Configuration

Configure one BGP process on R1.

```text
R1(config)# router bgp 65000
R1(config-router)# bgp router-id 1.1.1.1
```

The local BGP AS is **65000** for both VRFs.

### CUSTOMER_A Neighbor

Enter the `CUSTOMER_A` IPv4 address family, configure CE1 as an EBGP peer, and activate it.

```text
R1(config-router)# address-family ipv4 unicast vrf CUSTOMER_A
R1(config-router-af)# neighbor 10.10.10.2 remote-as 65100
R1(config-router-af)# neighbor 10.10.10.2 activate
```

This creates a neighbor within the `CUSTOMER_A` BGP routing context.

Its TCP session uses the `CUSTOMER_A` routing table to reach `10.10.10.2` through Ethernet0/0.

### CUSTOMER_B Neighbor

Configure CE2 under a *different* VRF address family.

```text
R1(config-router-af)# exit-address-family
R1(config-router)# address-family ipv4 unicast vrf CUSTOMER_B
R1(config-router-af)# neighbor 10.10.10.2 remote-as 65200
R1(config-router-af)# neighbor 10.10.10.2 activate
```

This creates another neighbor with the **same IP address**, but a different VRF and remote AS.

Its TCP session uses the `CUSTOMER_B` routing table to reach `10.10.10.2` through Ethernet0/1.

### Advertise R1's Loopbacks

The `network` command is configured separately in each VRF address family.

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast vrf CUSTOMER_A
R1(config-router-af)# network 1.1.1.0 mask 255.255.255.0
R1(config-router-af)# exit-address-family
R1(config-router)# address-family ipv4 unicast vrf CUSTOMER_B
R1(config-router-af)# network 1.1.1.0 mask 255.255.255.0
```

The prefix exists as a connected route in both VRFs.

- The first `network` command originates `1.1.1.0/24` into BGP for `CUSTOMER_A`.
- The second `network` command originates `1.1.1.0/24` into BGP for `CUSTOMER_B`.

The BGP tables remain separate even though the prefix is identical.

## CE BGP Configuration

The CEs use ordinary BGP, without VRF-specific address families.

### CE1 (AS 65100)

```text
CE1(config)# router bgp 65100
CE1(config-router)# bgp router-id 2.2.2.2
CE1(config-router)# neighbor 10.10.10.1 remote-as 65000
CE1(config-router)# address-family ipv4 unicast
CE1(config-router-af)# neighbor 10.10.10.1 activate
CE1(config-router-af)# network 192.168.1.0 mask 255.255.255.0
```

CE1 advertises `192.168.1.0/24` to R1, using AS **65100**.

### CE2 (AS 65200)

```text
CE2(config)# router bgp 65200
CE2(config-router)# bgp router-id 3.3.3.3
CE2(config-router)# neighbor 10.10.10.1 remote-as 65000
CE2(config-router)# address-family ipv4 unicast
CE2(config-router-af)# neighbor 10.10.10.1 activate
CE2(config-router-af)# network 192.168.1.0 mask 255.255.255.0
```

CE2 advertises the **same prefix**, `192.168.1.0/24`, to R1, but using AS **65200**.

These are two separate EBGP sessions:

```text
CUSTOMER_A:
R1 AS 65000 <---- EBGP ----> CE1 AS 65100
Neighbor IP on R1: 10.10.10.2

CUSTOMER_B:
R1 AS 65000 <---- EBGP ----> CE2 AS 65200
Neighbor IP on R1: 10.10.10.2
```

## BGP Neighbors in Each VRF

Verify the sessions on R1.

```text
R1# show bgp vpnv4 unicast vrf CUSTOMER_A summary
R1# show bgp vpnv4 unicast vrf CUSTOMER_B summary
```

These commands display BGP neighbor information for each VRF on IOS XE. The `vpnv4` keyword in the **show command** does not mean an MPLS VPNv4 session was configured between R1 and the CEs.

Illustrative summary of the expected result:

```text
CUSTOMER_A:
Neighbor        AS       State/PfxRcd
10.10.10.2      65100    1

CUSTOMER_B:
Neighbor        AS       State/PfxRcd
10.10.10.2      65200    1
```

A numeric prefix count in `State/PfxRcd` indicates that the corresponding session is established and receiving routes.

On each CE, you can also verify its normal IPv4 BGP session.

```text
CE1# show ip bgp summary
CE2# show ip bgp summary
```

## BGP Routes in Each VRF

Check R1's BGP table for `CUSTOMER_A`.

```text
R1# show bgp vpnv4 unicast vrf CUSTOMER_A
```

The BGP table contains:

```text
1.1.1.0/24       Locally originated with the network command
192.168.1.0/24   Learned from CE1, next hop 10.10.10.2, AS_PATH 65100
```

Check `CUSTOMER_B`.

```text
R1# show bgp vpnv4 unicast vrf CUSTOMER_B
```

The BGP table contains:

```text
1.1.1.0/24       Locally originated with the network command
192.168.1.0/24   Learned from CE2, next hop 10.10.10.2, AS_PATH 65200
```

The prefixes and next-hop IP addresses are the same, but their VRF contexts and AS paths are different.

### VRF Routing Tables

Check the routes installed into each VRF's routing table.

```text
R1# show ip route vrf CUSTOMER_A bgp
R1# show ip route vrf CUSTOMER_B bgp
```

The expected learned route in each VRF is:

```text
CUSTOMER_A:
B    192.168.1.0/24 [20/0] via 10.10.10.2

CUSTOMER_B:
B    192.168.1.0/24 [20/0] via 10.10.10.2
```

Both routes have an EBGP administrative distance of **20**.

The routing entries do not conflict because they belong to different VRFs. The `1.1.1.0/24` route remains connected in each VRF, even though it is also originated into BGP.

### CE Routing Tables

CE1 learns the `1.1.1.0/24` route from R1.

```text
CE1# show ip route bgp
B    1.1.1.0/24 [20/0] via 10.10.10.1
```

CE2 learns the same prefix from R1.

```text
CE2# show ip route bgp
B    1.1.1.0/24 [20/0] via 10.10.10.1
```

Although both CEs use the same destination IP address, they reach **different Loopback interfaces** on R1.

## Testing Connectivity

From CE1:

```text
CE1# ping 1.1.1.1 source Loopback0
```

The packet enters R1 through Ethernet0/0 in `CUSTOMER_A` and is delivered to R1's Loopback0.

From CE2:

```text
CE2# ping 1.1.1.1 source Loopback0
```

The packet enters R1 through Ethernet0/1 in `CUSTOMER_B` and is delivered to R1's Loopback1.

The two traffic flows remain isolated even though the source and destination addresses overlap.

## The `network` Command and Route Origination

In a VRF-specific address family, the `network` command checks for an **exact matching route in that VRF's IPv4 routing table**.

Example:

```text
router bgp 65000
 address-family ipv4 unicast vrf CUSTOMER_A
  network 1.1.1.0 mask 255.255.255.0
```

R1 must have an exact `1.1.1.0/24` route in the `CUSTOMER_A` routing table for the prefix to be originated.

A matching route only in the global routing table or in `CUSTOMER_B` does **not** satisfy this requirement.

Likewise, configuring this command in the global address family does not originate the prefix into `CUSTOMER_A`:

```text
router bgp 65000
 address-family ipv4 unicast
  network 1.1.1.0 mask 255.255.255.0
```

The address-family context is essential.

## Redistribution Inside a VRF

BGP can also originate routes from other routing sources using redistribution.

Example:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast vrf CUSTOMER_A
R1(config-router-af)# redistribute connected
```

This redistributes eligible connected routes from `CUSTOMER_A` into the `CUSTOMER_A` BGP address family.

It does **not** redistribute connected routes from `CUSTOMER_B` or the global routing table.

Be careful: `redistribute connected` may advertise customer-facing transit subnets as well as loopbacks. Use route maps if you need to restrict which prefixes are originated.

For example, to advertise only `1.1.1.0/24`:

```text
R1(config)# ip prefix-list LOOPBACK-ONLY permit 1.1.1.0/24
R1(config)# route-map CONNECTED-TO-BGP permit 10
R1(config-route-map)# match ip address prefix-list LOOPBACK-ONLY

R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast vrf CUSTOMER_A
R1(config-router-af)# redistribute connected route-map CONNECTED-TO-BGP
```

Redistribution and `network` statements are two different ways of originating routes into BGP. You do not need both methods for the same prefix in this example.

## VRF-Aware BGP and the Global Address Family

A global BGP neighbor and a VRF BGP neighbor do not automatically share routes.

For example:

```text
router bgp 65000
 address-family ipv4 unicast
  neighbor 172.16.1.2 activate
 !
 address-family ipv4 unicast vrf CUSTOMER_A
  neighbor 10.10.10.2 remote-as 65100
  neighbor 10.10.10.2 activate
```

The global address family and the `CUSTOMER_A` address family are separate routing contexts.

- Routes received from `172.16.1.2` belong to the global IPv4 BGP context.
- Routes received from `10.10.10.2` belong to the `CUSTOMER_A` IPv4 BGP context.
- Their routes are not automatically exchanged just because both neighbors are configured under `router bgp 65000`.

Sharing routes between VRFs or with the global table requires an additional mechanism, which is covered in the **Route Leaking** section.

## VRF-Aware IPv6 BGP

VRF-aware BGP also supports IPv6 unicast routing.

For example, extend `CUSTOMER_A` with IPv6 addressing.

On R1, enable IPv6 routing and the IPv6 address family for the VRF:

```text
R1(config)# ipv6 unicast-routing
R1(config)# vrf definition CUSTOMER_A
R1(config-vrf)# address-family ipv6
```

Configure IPv6 on the existing customer-facing interface:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 address 2001:DB8:10::1/64
```

Configure the IPv6 BGP address family:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv6 vrf CUSTOMER_A
R1(config-router-af)# neighbor 2001:DB8:10::2 remote-as 65100
R1(config-router-af)# neighbor 2001:DB8:10::2 activate
```

On CE1:

```text
CE1(config)# ipv6 unicast-routing
CE1(config)# interface Ethernet0/0
CE1(config-if)# ipv6 address 2001:DB8:10::2/64

CE1(config)# interface Loopback0
CE1(config-if)# ipv6 address 2001:DB8:100::1/64

CE1(config)# router bgp 65100
CE1(config-router)# address-family ipv6
CE1(config-router-af)# neighbor 2001:DB8:10::1 remote-as 65000
CE1(config-router-af)# neighbor 2001:DB8:10::1 activate
CE1(config-router-af)# network 2001:DB8:100::/64
```

The IPv6 BGP routes learned by R1 are associated with `CUSTOMER_A`'s **IPv6** routing table.

Verify IPv6 reachability and routes:

```text
R1# show ipv6 route vrf CUSTOMER_A
R1# show bgp vpnv6 unicast vrf CUSTOMER_A
```

The IPv4 and IPv6 address families are separate; configuring IPv4 BGP neighbors does not automatically enable IPv6 BGP route exchange.

## BGP Router IDs

BGP uses a 32-bit router ID, even when carrying IPv6 routes.

In this example:

```text
R1:  bgp router-id 1.1.1.1
CE1: bgp router-id 2.2.2.2
CE2: bgp router-id 3.3.3.3
```

The router ID is distinct from the BGP neighbor's IP address, and it is not required to be reachable as a destination.

Unlike OSPFv2, you do **not** create a separate `router bgp` process for each VRF. The VRFs share R1's BGP process and local AS number.

## VRF-Aware BGP Does Not Leak Routes Between VRFs

In this topology, both `CUSTOMER_A` and `CUSTOMER_B` have BGP routes for `192.168.1.0/24`.

```text
CUSTOMER_A: 192.168.1.0/24 learned from CE1, AS 65100
CUSTOMER_B: 192.168.1.0/24 learned from CE2, AS 65200
```

These are **not routes leaked between the VRFs**. Each VRF learned its own route from its own BGP neighbor.

R1 does not advertise CE1's LAN route to CE2 merely because both neighbors are configured in the same BGP process.

Route leaking is a separate topic and is covered under [1.2.f Route Leaking](../route-leaking/overview.md).

## Verification Commands

| Command | Purpose |
|---|---|
| `show vrf` | Verify VRF definitions and associated interfaces |
| `show ip route vrf CUSTOMER_A` | View the IPv4 routing table for `CUSTOMER_A` |
| `show ip route vrf CUSTOMER_A bgp` | View BGP routes installed in the VRF routing table |
| `show bgp vpnv4 unicast vrf CUSTOMER_A summary` | Check BGP neighbors for `CUSTOMER_A` |
| `show bgp vpnv4 unicast vrf CUSTOMER_A` | View BGP prefixes for `CUSTOMER_A` |
| `show ip route vrf CUSTOMER_A 10.10.10.2` | Check reachability to the BGP neighbor within the VRF |
| `ping vrf CUSTOMER_A 10.10.10.2` | Test the VRF-specific path to the neighbor |
| `show ipv6 route vrf CUSTOMER_A` | View the VRF's IPv6 routing table |
| `show bgp vpnv6 unicast vrf CUSTOMER_A` | View IPv6 BGP prefixes for the VRF |

Command availability and output formatting can vary by IOS XE platform and release.

## Troubleshooting

### BGP Neighbor Configured in the Wrong Address Family

If CE1 is configured under the global IPv4 address family rather than `CUSTOMER_A`, R1 will not use the correct VRF to establish the intended session.

Correct VRF-aware configuration:

```text
router bgp 65000
 address-family ipv4 unicast vrf CUSTOMER_A
  neighbor 10.10.10.2 remote-as 65100
  neighbor 10.10.10.2 activate
```

### Neighbor Is Not Activated

Defining a neighbor does not replace enabling the intended address family.

```text
router bgp 65000
 address-family ipv4 unicast vrf CUSTOMER_A
  neighbor 10.10.10.2 remote-as 65100
  neighbor 10.10.10.2 activate
```

Make sure the neighbor is active under the **correct VRF address family**.

### Cannot Reach the Neighbor in the VRF

Check the VRF membership, interface status, and connected route:

```text
R1# show vrf
R1# show ip interface brief
R1# show ip route vrf CUSTOMER_A 10.10.10.2
R1# ping vrf CUSTOMER_A 10.10.10.2
```

A route to the peer in the global table does not substitute for reachability in `CUSTOMER_A`.

### Wrong Remote AS

The remote AS configured on R1 must match the CE's actual BGP AS:

```text
CUSTOMER_A neighbor 10.10.10.2 remote-as 65100
CUSTOMER_B neighbor 10.10.10.2 remote-as 65200
```

Even though the neighbor IP is identical, R1 can distinguish the two neighbors by VRF context.

### Session Is Up, but No Routes Are Received

Check the relevant CE's `network` statement and routing table.

```text
CE1# show ip route 192.168.1.0
CE1# show ip bgp
```

For `network 192.168.1.0 mask 255.255.255.0` to originate the route, the CE needs an exact `192.168.1.0/24` route in its local routing table.

Also check any outbound BGP policy on the CE and inbound policy on R1.

### Route Is in BGP but Not in the VRF Routing Table

A BGP prefix can be present in the BGP table without being installed in the VRF's routing table.

Possible reasons include:

- Another routing source provides a route that wins the routing-table selection process.
- BGP next-hop resolution fails in the VRF.
- The path is not the selected BGP best path.

Check both tables:

```text
R1# show bgp vpnv4 unicast vrf CUSTOMER_A
R1# show ip route vrf CUSTOMER_A
```

### Checking Only the Global Table

```text
R1# show ip bgp summary
R1# show ip route
```

These commands are not substitutes for checking the VRF-specific BGP and IP routing tables.

Use the VRF-specific `show` commands instead.

## Key Points

- BGP uses a **VRF-specific address family** to exchange routes inside a VRF.
- A single `router bgp` process can support multiple VRF address families.
- The BGP process has one local AS number, while neighbors in different VRFs can have different remote AS numbers.
- Configure IPv4 BGP using `address-family ipv4 unicast vrf <vrf-name>`.
- Configure IPv6 BGP using `address-family ipv6 unicast vrf <vrf-name>`.
- Configure neighbors and activate them in the correct VRF address family.
- The **same neighbor IP address** can be configured in different VRFs.
- The **same prefix** can be learned and originated independently in different VRFs.
- A `network` statement must match an exact route in the **same VRF's** routing table.
- Redistribution under a VRF address family redistributes routes from that VRF, not from other VRFs.
- Use `show bgp vpnv4 unicast vrf <vrf-name>` to inspect the VRF's IPv4 BGP table on IOS XE.
- VRF-aware BGP does **not** automatically leak routes between VRFs or to the global routing table.
- Basic VRF-Lite BGP peering does not require MPLS labels, VPNv4 peering, or route targets.

## References

- [Cisco IOS XE: Configuring a Basic BGP Network](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-basic-net-0.html)
- [Cisco IOS XE: Multi-VRF Support](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_mp-multi-vrf-vrf-lite.html)
- [Cisco IOS XE: Configuring VRF-Lite](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9600/software/release/17-17/configuration_guide/rtng/b_1717_rtng_9600_cg/configuring_vrf_lite.html)
