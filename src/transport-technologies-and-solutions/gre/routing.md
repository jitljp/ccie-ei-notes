# Routing over GRE Tunnels

A **point-to-point GRE tunnel** acts as a logical Layer 3 link between two routers. Once the tunnel is configured, the routers can exchange routes and forward traffic over it using **static routes** or **dynamic routing protocols**.

GRE is particularly useful because it can transport multicast packets used by protocols such as OSPF and EIGRP, even when the network between the tunnel endpoints provides only ordinary IPv4 unicast connectivity.

This page covers routing **through the GRE overlay**, not the routing needed to establish reachability between the tunnel endpoints.

---

## Underlay Routing vs. Overlay Routing

There are two separate routing decisions in a GRE network:

| Routing layer | Purpose | Example |
| --- | --- | --- |
| **Underlay** | Delivers the outer GRE-encapsulated packet to the remote tunnel endpoint. | R1 reaches `198.51.100.2` through the ISP. |
| **Overlay** | Carries the original packet to a remote network through the tunnel. | R1 reaches `10.2.2.0/24` through `Tunnel0`. |

```text
                 Underlay (IPv4)

R1 E0/0 --------------- ISP --------------- E0/0 R2
192.0.2.1                                  198.51.100.2
    |                                            |
    |             GRE overlay                    |
    +============================================+
    |                                            |
Tunnel0                                       Tunnel0
10.0.12.1/30                                  10.0.12.2/30
    |                                            |
Loopback1                                      Loopback1
10.1.1.1/24                                    10.2.2.2/24
```

When R1 sends a packet to `10.2.2.2`:

1. The overlay route selects `Tunnel0`.
2. R1 encapsulates the original packet in GRE and an outer IPv4 header.
3. The underlay route sends the outer packet toward `198.51.100.2` through the ISP.
4. R2 decapsulates the packet and forwards it according to the inner destination.

```text
Inner destination:  10.2.2.2       (overlay routing)
Outer destination:  198.51.100.2   (underlay routing)
```

The ISP does not need routes to the overlay prefixes or to the tunnel subnet.

> The **route to the GRE tunnel destination must not resolve through the GRE tunnel itself**. Otherwise, the router can encounter recursive routing and disable the tunnel.

---

## Base Configuration

The examples continue the lab from [Point-to-Point GRE Configuration](configuration.md).

### R1

```cisco
interface Ethernet0/0
 ip address 192.0.2.1 255.255.255.252
 no shutdown
!
interface Loopback1
 ip address 10.1.1.1 255.255.255.0
!
interface Tunnel0
 ip address 10.0.12.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.2
!
ip route 198.51.100.2 255.255.255.255 192.0.2.2
```

### ISP

```cisco
interface Ethernet0/0
 ip address 192.0.2.2 255.255.255.252
 no shutdown
!
interface Ethernet0/1
 ip address 198.51.100.1 255.255.255.252
 no shutdown
```

### R2

```cisco
interface Ethernet0/0
 ip address 198.51.100.2 255.255.255.252
 no shutdown
!
interface Loopback1
 ip address 10.2.2.2 255.255.255.0
!
interface Tunnel0
 ip address 10.0.12.2 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 192.0.2.1
!
ip route 192.0.2.1 255.255.255.255 198.51.100.1
```

The explicit `/32` static routes are **underlay routes**. They provide reachability to the remote GRE endpoint via the ISP, regardless of what routing protocol is later configured over `Tunnel0`.

Verify the tunnel before adding overlay routing:

```cisco
R1# ping 198.51.100.2 source 192.0.2.1
R1# ping 10.0.12.2 source 10.0.12.1
R1# show ip route 198.51.100.2
R1# show interfaces Tunnel0
```

The first ping tests the underlay. The second ping tests communication through the GRE tunnel.

**Each routing method below is an independent example.** When moving from one method to another, remove the previous overlay static routes or routing-protocol configuration. Leave the underlay routes in place.

---

## Static Routing over GRE

The simplest method is to install routes to the remote site's networks with the tunnel as the outgoing path.

### R1

```cisco
ip route 10.2.2.0 255.255.255.0 10.0.12.2
```

### R2

```cisco
ip route 10.1.1.0 255.255.255.0 10.0.12.1
```

Both routes use the **remote tunnel IP address** as the next hop.

Alternatively, because `Tunnel0` is a point-to-point interface, the static route can specify the outgoing interface:

```cisco
R1(config)# ip route 10.2.2.0 255.255.255.0 Tunnel0
```

For this point-to-point GRE tunnel, either method can direct traffic into the tunnel. Unlike a multipoint Ethernet interface, a point-to-point GRE interface has one configured remote endpoint.

Verify on R1:

```cisco
R1# show ip route 10.2.2.0
R1# ping 10.2.2.2 source 10.1.1.1
```

Expected routing decision:

```text
10.2.2.0/24
  via 10.0.12.2, Tunnel0
```

The route toward the remote LAN is separate from the route toward the remote **tunnel destination**:

```text
10.2.2.0/24    -> Tunnel0        (overlay)
198.51.100.2   -> Ethernet0/0    (underlay)
```

Static routes do not automatically detect loss of the remote GRE endpoint. If `Tunnel0` stays up while the remote endpoint is unreachable, traffic can be blackholed. Tunnel keepalives and other failure-detection mechanisms are covered in [GRE Tunnel State and Keepalives](keepalives.md).

---

## Dynamic Routing over GRE

A GRE tunnel can carry routing protocol packets as passenger traffic.

Common choices include:

| Protocol | Packet transport over the tunnel | Neighbor discovery / peering |
| --- | --- | --- |
| **OSPFv2** | IPv4 protocol 89, inside GRE | Typically multicast to `224.0.0.5` and `224.0.0.6` as appropriate |
| **EIGRP for IPv4** | IPv4 protocol 88, inside GRE | Multicast to `224.0.0.10` and unicast |
| **BGP** | TCP port 179, inside GRE | Explicitly configured TCP peers |
| **OSPFv3** | IPv6 protocol 89, inside GRE | IPv6 multicast to `ff02::5` and `ff02::6` as appropriate |

GRE encapsulates these packets in **outer unicast IP packets** addressed to the configured tunnel destination.

For example, an OSPFv2 Hello sent by R1 can look like this:

```text
Outer IPv4 header:
  Source:      192.0.2.1
  Destination: 198.51.100.2
  Protocol:    47 (GRE)

GRE header:
  Protocol Type: 0x0800 (IPv4)

Inner IPv4 header:
  Source:      10.0.12.1
  Destination: 224.0.0.5
  Protocol:    89 (OSPF)
```

The ISP does **not** need multicast routing or OSPF configured. It only needs to permit and forward the outer GRE packets (IPv4 protocol 47).

BGP does not rely on multicast and can also form a session over a GRE tunnel.

---

## OSPF over GRE

On a normal point-to-point GRE interface, the default OSPF network type is **point-to-point**.

This means:

- No DR or BDR election is required.
- OSPF can discover the remote neighbor using multicast Hellos.
- Both routers must place the tunnel in a compatible OSPF area.
- OSPF cost depends on the interface's configured bandwidth and OSPF reference bandwidth unless explicitly set.

For this example, enable OSPF process 10 on the tunnel and on the simulated LAN loopback. Do **not** enable this overlay OSPF process on the ISP-facing interfaces.

### R1

```cisco
router ospf 10
 router-id 1.1.1.1
!
interface Tunnel0
 ip ospf 10 area 0
!
interface Loopback1
 ip ospf network point-to-point
 ip ospf 10 area 0
```

### R2

```cisco
router ospf 10
 router-id 2.2.2.2
!
interface Tunnel0
 ip ospf 10 area 0
!
interface Loopback1
 ip ospf network point-to-point
 ip ospf 10 area 0
```

### Why Configure the Loopbacks as Point-to-Point?

By default, OSPFv2 treats a loopback as a **host route** and advertises its IP address with a `/32` mask, even if the interface is configured with a `/24`.

```text
R1 Loopback1: 10.1.1.1/24

Default OSPF advertisement:
  10.1.1.1/32

With ip ospf network point-to-point on Loopback1:
  10.1.1.0/24
```

The `ip ospf network point-to-point` command on `Loopback1` ensures that OSPF advertises the site's entire `/24` network in this lab. It does not mean the loopback is physically connected to another router.

The `Tunnel0` interface already uses the point-to-point OSPF network type by default, so the command is not needed there.

### Verify OSPF

```cisco
R1# show ip ospf neighbor
R1# show ip ospf interface Tunnel0
R1# show ip route ospf
R1# ping 10.2.2.2 source 10.1.1.1
```

R1 should form a **FULL** adjacency with R2 over `Tunnel0` and learn the remote `/24` as an OSPF route.

```text
OSPF neighbor:
  Router ID: 2.2.2.2
  Interface: Tunnel0
  State: FULL

Learned route:
  10.2.2.0/24 via 10.0.12.2, Tunnel0
```

`show ip ospf interface Tunnel0` also shows the interface's network type, cost, and Hello/Dead timers.

---

## EIGRP over GRE

EIGRP can form neighbor relationships across GRE because GRE transports its multicast and unicast packets.

The following example uses **named EIGRP** with AS 100. The underlay interfaces are not included in the overlay EIGRP address family.

### R1

```cisco
router eigrp GRE
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 1.1.1.1
  network 10.0.12.0 0.0.0.3
  network 10.1.1.0 0.0.0.255
  af-interface default
   passive-interface
  exit-af-interface
  af-interface Tunnel0
   no passive-interface
  exit-af-interface
 exit-address-family
```

### R2

```cisco
router eigrp GRE
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 2.2.2.2
  network 10.0.12.0 0.0.0.3
  network 10.2.2.0 0.0.0.255
  af-interface default
   passive-interface
  exit-af-interface
  af-interface Tunnel0
   no passive-interface
  exit-af-interface
 exit-address-family
```

The `network` commands enable EIGRP on the matching interfaces. The `af-interface default` setting makes all EIGRP interfaces passive, and `no passive-interface` under `Tunnel0` allows EIGRP Hellos and adjacency formation across the GRE tunnel.

The LAN loopbacks remain passive but their networks are advertised.

### Verify EIGRP

```cisco
R1# show ip eigrp neighbors
R1# show ip eigrp interfaces detail
R1# show ip route eigrp
R1# ping 10.2.2.2 source 10.1.1.1
```

R1 should see R2 as an EIGRP neighbor at `10.0.12.2` on `Tunnel0`, with a learned route to `10.2.2.0/24`.

### EIGRP Bandwidth Considerations

The `bandwidth` value on a tunnel interface is a **logical interface setting** used by features such as EIGRP. It is not the physical speed of the GRE tunnel.

GRE tunnel interfaces often have a low default configured bandwidth (Cisco documents **9.6 kbps** for many IOS/IOS XE tunnel implementations).

For example, configure 100 Mbps as the logical bandwidth on both tunnel interfaces:

```cisco
interface Tunnel0
 bandwidth 100000
```

EIGRP uses the interface bandwidth and delay in its metric calculations, and by default limits its own traffic to **50% of the configured interface bandwidth**.

For classic EIGRP configuration, the bandwidth percentage can be adjusted with:

```cisco
R1(config-if)# ip bandwidth-percent eigrp 100 25
```

For named EIGRP, configure it under the tunnel's address-family interface instead:

```cisco
router eigrp GRE
 address-family ipv4 unicast autonomous-system 100
  af-interface Tunnel0
   bandwidth-percent 25
  exit-af-interface
 exit-address-family
```

The `bandwidth` command does not shape or police the GRE tunnel. The real available bandwidth still depends on the underlay.

---

## BGP over GRE

BGP can establish a TCP session between the tunnel addresses, just as it would over a physical point-to-point link.

For example, use eBGP with R1 in AS 65001 and R2 in AS 65002.

### R1

```cisco
router bgp 65001
 bgp router-id 1.1.1.1
 neighbor 10.0.12.2 remote-as 65002
 address-family ipv4 unicast
  neighbor 10.0.12.2 activate
  network 10.1.1.0 mask 255.255.255.0
 exit-address-family
```

### R2

```cisco
router bgp 65002
 bgp router-id 2.2.2.2
 neighbor 10.0.12.1 remote-as 65001
 address-family ipv4 unicast
  neighbor 10.0.12.1 activate
  network 10.2.2.0 mask 255.255.255.0
 exit-address-family
```

Because the BGP peers use **directly connected tunnel addresses**, eBGP does not require `ebgp-multihop` for this example.

The `network` statements originate the connected `/24` prefixes already present in the routing tables.

### Verify BGP

```cisco
R1# show bgp ipv4 unicast summary
R1# show bgp ipv4 unicast 10.2.2.0/24
R1# show ip route bgp
R1# ping 10.2.2.2 source 10.1.1.1
```

R1 should learn `10.2.2.0/24` with `10.0.12.2` as its BGP next hop.

```text
BGP next hop: 10.0.12.2
       |
       v
Connected tunnel network: 10.0.12.0/30
       |
       v
Tunnel0 -> GRE encapsulation -> ISP -> R2
```

The BGP neighbor uses the tunnel as a normal IP link; BGP does not need to understand the GRE header.

---

## IPv6 Routing over GRE

The same `gre ip` tunnel can transport **IPv6 routing traffic** over the IPv4 underlay.

As configured in [GRE Tunnels for IPv6 over IPv4](ipv6-over-ipv4.md), assume:

```text
R1 Tunnel0:     2001:db8:12::1/64
R2 Tunnel0:     2001:db8:12::2/64
R1 Loopback1:   2001:db8:1::1/64
R2 Loopback1:   2001:db8:2::2/64
```

To use IPv6 static routes:

### R1

```cisco
ipv6 unicast-routing
!
interface Tunnel0
 ipv6 address 2001:db8:12::1/64
!
interface Loopback1
 ipv6 address 2001:db8:1::1/64
!
ipv6 route 2001:db8:2::/64 2001:db8:12::2
```

### R2

```cisco
ipv6 unicast-routing
!
interface Tunnel0
 ipv6 address 2001:db8:12::2/64
!
interface Loopback1
 ipv6 address 2001:db8:2::2/64
!
ipv6 route 2001:db8:1::/64 2001:db8:12::1
```

Verify:

```cisco
R1# show ipv6 route 2001:db8:2::/64
R1# ping ipv6 2001:db8:2::2 source 2001:db8:1::1
```

OSPFv3, EIGRP for IPv6, and MP-BGP for IPv6 unicast can also use the tunnel to exchange IPv6 routes. The underlying ISP still only needs IPv4 reachability between the outer GRE endpoints.

---

## Recursive Routing: The Most Important GRE Routing Failure

A GRE tunnel requires the router to reach its **outer tunnel destination** using the underlay.

For R1:

```text
Tunnel destination: 198.51.100.2

Correct:
198.51.100.2 -> 192.0.2.2 -> Ethernet0/0

Incorrect:
198.51.100.2 -> Tunnel0
```

If the router resolves the tunnel destination through the same tunnel, it creates a recursive forwarding dependency:

```text
To send traffic through Tunnel0:
  Encapsulate packet to 198.51.100.2
      |
      v
To reach 198.51.100.2:
  Route points through Tunnel0
      |
      v
To send traffic through Tunnel0:
  Encapsulate packet to 198.51.100.2
      |
      +---- recursion / forwarding loop
```

Cisco IOS/IOS XE can report:

```text
%TUN-5-RECURDOWN: Tunnel0 temporarily disabled due to recursive routing
```

The tunnel may repeatedly flap, taking OSPF, EIGRP, or BGP neighbors down with it.

### How Dynamic Routing Can Cause Recursion

Suppose R1 normally reaches `198.51.100.2` through a default route toward the ISP. Then R1 learns `198.51.100.0/30` through the GRE tunnel because an overlay routing protocol is accidentally advertising an underlay prefix.

```text
Default route:
0.0.0.0/0         -> ISP

More-specific route:
198.51.100.0/30   -> Tunnel0
```

The more-specific route wins for `198.51.100.2`, even though it is learned over a less-preferred routing protocol or has a higher administrative distance than the default route.

**Longest-prefix matching takes precedence over administrative distance when the matching prefixes have different lengths.**

### Preventing Recursion

In this lab, keep explicit underlay host routes on both routers:

```cisco
! R1
ip route 198.51.100.2 255.255.255.255 192.0.2.2

! R2
ip route 192.0.2.1 255.255.255.255 198.51.100.1
```

Other design approaches include keeping underlay and overlay routing in separate processes, filtering underlay prefixes from overlay advertisements, or using separate VRFs for underlay and overlay routing.

Check the active route **to the tunnel destination**, not just the route to the remote LAN:

```cisco
R1# show ip route 198.51.100.2
R1# show ip cef 198.51.100.2
R1# show interfaces Tunnel0
```

The route must resolve through the underlay.

---

## Routing Metrics and Tunnel Path Selection

A GRE tunnel is a logical interface, but routing protocols still assign metrics to it.

| Protocol | Main consideration |
| --- | --- |
| **Static** | Administrative distance and longest-prefix matching determine route selection. |
| **OSPF** | Cost is based on interface bandwidth and reference bandwidth unless overridden with `ip ospf cost`. |
| **EIGRP** | Uses configured bandwidth and delay in its classic metric calculation. |
| **BGP** | BGP path selection applies; the tunnel is the path used to reach a BGP next hop. |

You can set OSPF cost explicitly:

```cisco
interface Tunnel0
 ip ospf cost 100
```

Or adjust EIGRP's metric-related interface attributes:

```cisco
interface Tunnel0
 bandwidth 100000
 delay 1000
```

The `delay` value is specified in tens of microseconds. Neither command guarantees a particular amount of physical bandwidth or latency over the ISP.

With multiple tunnels, routing protocols can select preferred paths or support load sharing according to their routing and forwarding rules. However, configuring a routing metric does **not** control which physical ISP path the outer GRE packet uses. That is determined by the underlay route to the remote tunnel destination.

---

## Verification Commands

| Command | What it verifies |
| --- | --- |
| `show interfaces Tunnel0` | Tunnel state, transport endpoints, bandwidth, and encapsulation. |
| `show ip route <tunnel-destination>` | Underlay route to the remote GRE endpoint. |
| `show ip cef <tunnel-destination>` | Forwarding resolution of the outer destination. |
| `show ip route <remote-prefix>` | Overlay path toward the remote LAN. |
| `show ip ospf neighbor` | OSPF adjacency over GRE. |
| `show ip ospf interface Tunnel0` | OSPF network type, cost, area, and timers. |
| `show ip eigrp neighbors` | EIGRP adjacency over GRE. |
| `show bgp ipv4 unicast summary` | BGP neighbor state. |
| `show ipv6 route <remote-prefix>` | IPv6 overlay route. |

A useful order for troubleshooting is:

1. Verify the outer tunnel destination is reachable through the underlay.
2. Verify tunnel state and ping the remote tunnel address.
3. Verify routing-protocol neighbor formation, if dynamic routing is used.
4. Verify that the remote LAN prefix is installed with the expected next hop.
5. Test end-to-end traffic using a source address from the local LAN.

A tunnel that is `up/up` does not guarantee that GRE packets are reaching the peer. Basic point-to-point GRE does not establish a session or confirm remote reachability by default.

---

## Key Points

- GRE provides a logical point-to-point interface over an IP underlay.
- **Underlay routing** delivers outer GRE packets to the remote tunnel endpoint; **overlay routing** forwards original packets through the tunnel.
- The ISP only needs reachability between the outer tunnel endpoints. It does not need overlay routes or multicast routing for OSPF/EIGRP passenger traffic.
- Static routes can use the remote tunnel IP as the next hop or the point-to-point tunnel interface as the outgoing interface.
- OSPF and EIGRP can exchange multicast routing packets through GRE; BGP uses a TCP session across the tunnel.
- The default OSPF network type on a point-to-point GRE tunnel is **point-to-point**, so no DR/BDR election occurs.
- OSPFv2 normally advertises loopbacks as `/32` host routes. Configure `ip ospf network point-to-point` on a loopback to advertise its configured subnet mask.
- Named EIGRP can use the GRE interface as its only non-passive interface while advertising passive LAN interfaces.
- eBGP between directly connected tunnel IP addresses does not require `ebgp-multihop`.
- IPv6 traffic and IPv6 routing protocols can be carried over `tunnel mode gre ip` using an IPv4 underlay.
- `bandwidth`, `delay`, and `ip ospf cost` influence routing decisions, not the physical capacity of the GRE tunnel.
- The route to the **outer tunnel destination must not use the same tunnel**. Recursive resolution can cause `%TUN-5-RECURDOWN` and tunnel flapping.
- If a more-specific route to the tunnel destination is learned over the overlay, it can override an underlay default route. Use appropriate route separation or explicit underlay routes.

---

## References

- [Cisco: Implementing Tunnels (IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_ir-impl-tun-xe.html)
- [Cisco: The `%TUN-5-RECURDOWN` Error Message and Flapping EIGRP/OSPF/BGP Neighbors Over a GRE Tunnel](https://www.cisco.com/c/en/us/support/docs/ip/enhanced-interior-gateway-routing-protocol-eigrp/22327-gre-flap.html)
- [Cisco: Review OSPF Frequently Asked Questions](https://www.cisco.com/c/en/us/support/docs/ip/open-shortest-path-first-ospf/9237-9.html)
- [Cisco: EIGRP Configuration Guide](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_ire-enhanced-igrp.html)
- [Cisco: EIGRP Command Reference, `ip bandwidth-percent eigrp`](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_eigrp/command/ire-cr-book/ire-i1.html)
