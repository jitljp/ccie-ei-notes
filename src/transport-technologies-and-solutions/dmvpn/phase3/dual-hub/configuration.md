# Dual-Hub Phase 3 Configuration

This page builds two complete **DMVPN Phase 3 dual-hub labs** using the same Hub1, Hub2, Spoke1, Spoke2, Spoke3, and ISP addressing used throughout this chapter:

1. **Single-cloud dual hub:** One mGRE tunnel interface on every router; both hubs and all spokes share one DMVPN cloud. Spokes register with both NHSs.
2. **Dual-cloud dual hub:** Each hub participates in one cloud; every spoke has two mGRE tunnel interfaces, one per cloud.

Both examples use **EIGRP AS 100** for overlay routing and **NHRP Redirect/Shortcut** for direct spoke-to-spoke forwarding. They are **alternative, standalone configurations**, not two configurations to apply simultaneously. This page concentrates on configuration and verification. Detailed design tradeoffs are in [Dual-Hub DMVPN Topologies](topologies.md); failure detection and convergence are in [Dual-Hub Failover and Convergence](failover.md). IPsec/IKEv2 is added in the later security section.

---

## Lab Topology and Addressing

```text
             +---------+     +---------+
             |  Hub1   |     |  Hub2   |
             +---------+     +---------+
                  |               |
                  +-------+-------+
                          |
                   DMVPN Overlay
                          |
           +--------------+--------------+
           |              |              |
      +---------+    +---------+    +---------+
      | Spoke1  |    | Spoke2  |    | Spoke3  |
      +---------+    +---------+    +---------+
```

The ISP provides routed reachability between the WAN interfaces. **Do not advertise the DMVPN overlay LANs into the ISP.** Each DMVPN router uses its ISP-facing next hop for underlay reachability.

| Router | WAN interface | WAN address | ISP next hop | Loopback0 address |
| --- | --- | --- | --- | --- |
| Hub1 | Ethernet0/0 | `198.51.100.1/30` | `198.51.100.2` | `10.1.1.1/24` |
| Hub2 | Ethernet0/0 | `198.51.100.5/30` | `198.51.100.6` | `10.2.2.2/24` |
| Spoke1 | Ethernet0/0 | `198.51.100.9/30` | `198.51.100.10` | `10.11.11.11/24` |
| Spoke2 | Ethernet0/0 | `198.51.100.13/30` | `198.51.100.14` | `10.12.12.12/24` |
| Spoke3 | Ethernet0/0 | `198.51.100.17/30` | `198.51.100.18` | `10.13.13.13/24` |

All routers use `Tunnel0` in the single-cloud lab. In the dual-cloud lab, Hub1 uses `Tunnel0`, Hub2 uses `Tunnel1`, and every spoke uses **both `Tunnel0` and `Tunnel1`**.

### Common Underlay Prerequisites

The ISP interfaces from the earlier labs remain unchanged:

| ISP interface | Address |
| --- | --- |
| Toward Hub1 | `198.51.100.2/30` |
| Toward Hub2 | `198.51.100.6/30` |
| Toward Spoke1 | `198.51.100.10/30` |
| Toward Spoke2 | `198.51.100.14/30` |
| Toward Spoke3 | `198.51.100.18/30` |

Before configuring DMVPN, verify that the ISP can forward IP traffic between these interfaces, for example:

```cisco
Spoke1# ping 198.51.100.1 source Ethernet0/0
Spoke1# ping 198.51.100.5 source Ethernet0/0
Spoke2# ping 198.51.100.9 source Ethernet0/0
```

Success here establishes **underlay reachability**, not NHRP registration, routing adjacency, or spoke-to-spoke GRE reachability.

### Common Phase 3 Principles

- **Hubs:** `ip nhrp map multicast dynamic`, `ip nhrp redirect`, and `no ip split-horizon eigrp 100` on the mGRE interface.
- **Spokes:** An NHS mapping and multicast replication toward each hub, with `ip nhrp shortcut` to permit direct forwarding.
- **EIGRP next hops:** Retain normal hub next-hop-self behavior. Do **not** carry forward `no ip next-hop-self eigrp 100` from a Phase 2 lab.
- **Adjacencies:** EIGRP operates between hubs and spokes (and between hubs in the single-cloud example), not over the dynamically discovered spoke-to-spoke shortcuts.
- **Routing versus forwarding:** A spoke's ordinary EIGRP route can continue to point to a hub even after NHRP has installed a direct shortcut.
- **Underlay isolation:** Keep the WAN-facing interfaces passive in EIGRP by using `passive-interface default` and enabling neighbors only on the appropriate tunnel interfaces.

On modern IOS XE, `ip nhrp shortcut` is enabled by default and might not appear in `show running-config`. It is shown explicitly in the examples for clarity.

---

## Single-Cloud Dual-Hub Lab

### Architecture

```text
                   DMVPN 100: 172.16.0.0/24

                 Hub1                Hub2
              172.16.0.1          172.16.0.2
                   \                 /
                    \               /
                +-----------------------+
                |   Shared DMVPN Cloud  |
                |        Tunnel0        |
                +-----------------------+
                    /       |       \
                   /        |        \
                Spoke1    Spoke2    Spoke3
             172.16.0.11   .12       .13

              Both hubs act as NHSs for all spokes
              One mGRE tunnel per router
```

| Property | Value |
| --- | --- |
| Overlay subnet | `172.16.0.0/24` |
| Hub1 Tunnel0 | `172.16.0.1/24` |
| Hub2 Tunnel0 | `172.16.0.2/24` |
| Spoke1/2/3 Tunnel0 | `172.16.0.11/24`, `.12/24`, `.13/24` |
| NHRP network ID | `100` |
| NHRP authentication | `NHRP100` |
| GRE key | `100` |
| EIGRP AS | `100` |

Hub1 acts as an NHS and accepts dynamic registrations. Hub2 also acts as an NHS for the spokes, **and registers as a client of Hub1** to establish an NHRP mapping between the hubs. This is one supported way to make their mGRE endpoints mutually reachable and allows a hub-to-hub EIGRP adjacency on the shared tunnel. It does not make Hub2 dependent on Hub1 to accept its *own* spoke registrations.

A symmetric alternative is to configure explicit hub-to-hub NHRP mappings and multicast replication on both hubs. Do not assume that two hubs automatically discover one another merely because they share an NHRP network ID.

### Complete Configuration: Hub1

```cisco
hostname Hub1
!
interface Loopback0
 ip address 10.1.1.1 255.255.255.0
!
interface Ethernet0/0
 ip address 198.51.100.1 255.255.255.252
 no shutdown
!
interface Tunnel0
 ip address 172.16.0.1 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp map multicast dynamic
 ip nhrp network-id 100
 ip nhrp redirect
 no ip split-horizon eigrp 100
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
ip route 0.0.0.0 0.0.0.0 198.51.100.2
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255
 network 10.1.1.1 0.0.0.0
```

Hub1 has no `ip nhrp nhs` command because it is not registering with another NHS. Its `ip nhrp map multicast dynamic` permits multicast replication toward registered peers, including Hub2 once Hub2 registers.

### Complete Configuration: Hub2

```cisco
hostname Hub2
!
interface Loopback0
 ip address 10.2.2.2 255.255.255.0
!
interface Ethernet0/0
 ip address 198.51.100.5 255.255.255.252
 no shutdown
!
interface Tunnel0
 ip address 172.16.0.2 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp map multicast dynamic
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
 ip nhrp redirect
 no ip split-horizon eigrp 100
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
ip route 0.0.0.0 0.0.0.0 198.51.100.6
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255
 network 10.2.2.2 0.0.0.0
```

The command below has **two roles**:

```cisco
ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
```

1. Identify Hub1 as an NHS with overlay address `172.16.0.1` and NBMA address `198.51.100.1`.
2. Make Hub2 send routing-protocol multicast traffic toward Hub1 and register its tunnel-to-NBMA mapping with Hub1.

Hub2 still accepts registrations from Spoke1, Spoke2, and Spoke3 because it also has `ip nhrp map multicast dynamic` and participates as their configured NHS. Hub1's dynamic multicast list can include Hub2 after its registration succeeds.

### Complete Configuration: Spoke1

```cisco
hostname Spoke1
!
interface Loopback0
 ip address 10.11.11.11 255.255.255.0
!
interface Ethernet0/0
 ip address 198.51.100.9 255.255.255.252
 no shutdown
!
interface Tunnel0
 ip address 172.16.0.11 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
 ip nhrp nhs 172.16.0.2 nbma 198.51.100.5 multicast
 ip nhrp shortcut
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
ip route 0.0.0.0 0.0.0.0 198.51.100.10
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255
 network 10.11.11.11 0.0.0.0
 eigrp stub connected
```

### Complete Configuration: Spoke2

```cisco
hostname Spoke2
!
interface Loopback0
 ip address 10.12.12.12 255.255.255.0
!
interface Ethernet0/0
 ip address 198.51.100.13 255.255.255.252
 no shutdown
!
interface Tunnel0
 ip address 172.16.0.12 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
 ip nhrp nhs 172.16.0.2 nbma 198.51.100.5 multicast
 ip nhrp shortcut
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
ip route 0.0.0.0 0.0.0.0 198.51.100.14
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255
 network 10.12.12.12 0.0.0.0
 eigrp stub connected
```

### Complete Configuration: Spoke3

```cisco
hostname Spoke3
!
interface Loopback0
 ip address 10.13.13.13 255.255.255.0
!
interface Ethernet0/0
 ip address 198.51.100.17 255.255.255.252
 no shutdown
!
interface Tunnel0
 ip address 172.16.0.13 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
 ip nhrp nhs 172.16.0.2 nbma 198.51.100.5 multicast
 ip nhrp shortcut
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
ip route 0.0.0.0 0.0.0.0 198.51.100.18
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255
 network 10.13.13.13 0.0.0.0
 eigrp stub connected
```

### Why Configure EIGRP Stub on the Spokes?

Each spoke originates only its own connected loopback/LAN network and its tunnel connected subnet. It is **not intended to act as transit between hubs** or between other spokes. `eigrp stub connected` expresses that role, restricts which route types the spoke advertises, and limits EIGRP query propagation. The hubs are **not** stubs.

If the spoke must also advertise static routes, use an appropriate stub option (such as `eigrp stub connected static`) and ensure the routes are redistributed or otherwise originated into EIGRP. Stub does not itself redistribute static routes.

### Verify NHRP Registration with Two NHSs

On each spoke:

```cisco
show ip nhrp nhs
show ip nhrp
show dmvpn detail
show ip nhrp traffic
```

Expected relationships:

| Device | NHS | Peer overlay | Peer NBMA |
| --- | --- | --- | --- |
| Spoke1 | Hub1 | `172.16.0.1` | `198.51.100.1` |
| Spoke1 | Hub2 | `172.16.0.2` | `198.51.100.5` |
| Spoke2 | Hub1 | `172.16.0.1` | `198.51.100.1` |
| Spoke2 | Hub2 | `172.16.0.2` | `198.51.100.5` |
| Spoke3 | Hub1 | `172.16.0.1` | `198.51.100.1` |
| Spoke3 | Hub2 | `172.16.0.2` | `198.51.100.5` |
| Hub2 | Hub1 | `172.16.0.1` | `198.51.100.1` |

On Hub1 and Hub2:

```cisco
show ip nhrp
show ip nhrp multicast
show dmvpn
```

Confirm that each NHS knows all three spokes and that Hub1 knows Hub2's tunnel-to-NBMA mapping. An NHS registration is **not the same as an EIGRP adjacency**.

### Verify EIGRP Neighbor Relationships

```cisco
show ip eigrp neighbors
show ip eigrp interfaces detail
show ip protocols
```

In the example, the intended EIGRP adjacency graph is:

| Router | Expected tunnel EIGRP neighbors |
| --- | --- |
| Hub1 | Hub2, Spoke1, Spoke2, Spoke3 |
| Hub2 | Hub1, Spoke1, Spoke2, Spoke3 |
| Spoke1 | Hub1, Hub2 |
| Spoke2 | Hub1, Hub2 |
| Spoke3 | Hub1, Hub2 |

Spokes do not exchange EIGRP Hellos across dynamically formed spoke-to-spoke shortcuts because their multicast replication is configured toward the two hubs, not toward other spokes.

### Hub-to-Hub Adjacency Is Deliberate

With two hubs in the **same** tunnel subnet, their EIGRP adjacency requires usable GRE/NHRP reachability **and multicast replication between them**. Hub2's `ip nhrp nhs ... multicast` and its registration with Hub1 provide that relationship in this example.

### EIGRP Next-Hop Behavior and Spoke-to-Spoke Shortcuts

Unlike Phase 2, the hubs retain normal **EIGRP next-hop-self** behavior. A route learned by Spoke1 through Hub1 initially points to `172.16.0.1`; the same prefix learned via Hub2 points to `172.16.0.2`. EIGRP chooses a successor based on its computed metrics, and NHRP can subsequently optimize forwarding.

Example:

```text
Spoke1 EIGRP route to 10.12.12.0/24
  via 172.16.0.1 (Hub1), OR
  via 172.16.0.2 (Hub2), depending on the selected path

After an NHRP redirect and resolution:
  direct next hop 172.16.0.12 (Spoke2)
```

A single-cloud topology uses **one Tunnel0 interface on the spoke** for both hubs. Thus, configuring `delay` or `bandwidth` on Spoke1's Tunnel0 changes its interface metric for **both** hub paths. It cannot by itself make Hub1 preferred over Hub2. Preference may instead depend on advertisements from the hubs or routing-protocol policy.

#### Optional: Summarize Routes Advertised by the Hubs

To demonstrate Phase 3 summarization, apply an EIGRP interface summary on **both** hubs:

```cisco
interface Tunnel0
 ip summary-address eigrp 100 10.0.0.0 255.0.0.0
```

The hub needs valid more-specific routes for the destinations it summarizes. With the example loopbacks (`10.1.1.0/24`, `10.2.2.0/24`, `10.11.11.0/24`, etc.), spokes can use a covering `10.0.0.0/8` route via a hub. An NHRP shortcut for `10.12.12.0/24` can then appear as a **more-specific `H` route** to Spoke2. An exact-prefix shortcut may instead appear as a `%` next-hop override on an existing route.

Summarization also creates a local discard route for the aggregate, so missing more-specific routes can cause packets to be dropped at a hub. Further, a broad aggregate on the hub-to-hub tunnel can hide routes from the other hub. **Treat the `/8` summary as a controlled forwarding experiment**, not as a universal production recommendation. For production, design aggregates around reachable address blocks and routing domains.

### Single-Cloud Verification Workflow

1. Confirm Spoke1 can ping the WAN addresses of both hubs.
2. Confirm Spoke1 is registered with **both** NHSs.
3. Confirm Hub1 and Hub2 discover each other's NBMA addresses and form an EIGRP adjacency.
4. Confirm all five loopback/LAN prefixes are learned where needed.
5. Verify a first flow from Spoke1 to `10.12.12.12` follows the selected hub.
6. Generate repeated traffic and verify an NHRP shortcut to Spoke2.
7. Compare `show ip route`, `show ip route next-hop-override`, and `show ip cef 10.12.12.12 detail`.

```cisco
Spoke1# show ip route 10.12.12.12
Spoke1# traceroute 10.12.12.12 numeric
Spoke1# ping 10.12.12.12 repeat 10
Spoke1# show ip nhrp shortcut
Spoke1# show ip route next-hop-override
Spoke1# show ip cef 10.12.12.12 detail
Spoke1# traceroute 10.12.12.12 numeric
```

The EIGRP route in `show ip route` can continue to display the hub as the base next hop. An NHRP `H` route, `%`/`[NHO]` override, and the CEF next hop explain the actual forwarding path. The final traceroute can show Spoke2 as the **first** tunnel hop.

---

## Dual-Cloud Dual-Hub Lab

### Architecture

```text
                     Cloud 100                  Cloud 200
                    172.16.0.0/24              172.16.1.0/24

                Hub1 Tunnel0                  Hub2 Tunnel1
                172.16.0.1                   172.16.1.2
                   / | \                        / | \
                  /  |  \                      /  |  \
             Spoke1 Spoke2 Spoke3        Spoke1 Spoke2 Spoke3
            T0 .11  T0 .12  T0 .13     T1 .11  T1 .12  T1 .13

           Each spoke: Tunnel0 to Cloud 100, Tunnel1 to Cloud 200
```

| Property | Cloud 100 (Hub1) | Cloud 200 (Hub2) |
| --- | --- | --- |
| Hub tunnel | Hub1 `Tunnel0` | Hub2 `Tunnel1` |
| Hub tunnel IP | `172.16.0.1/24` | `172.16.1.2/24` |
| Spoke tunnel | `Tunnel0` | `Tunnel1` |
| Spoke tunnel addresses | `.11`, `.12`, `.13` in `172.16.0.0/24` | `.11`, `.12`, `.13` in `172.16.1.0/24` |
| NHRP network ID | `100` | `200` |
| NHRP authentication | `NHRP100` | `NHRP200` |
| GRE tunnel key | `100` | `200` |
| NHS | `172.16.0.1` | `172.16.1.2` |
| EIGRP AS | `100` | `100` |

Each spoke has **two separate NHRP registration relationships**, one per tunnel interface. Hub1 does not share an overlay subnet with Hub2, and the hubs do **not** become EIGRP neighbors over their respective DMVPN tunnels.

The two spoke tunnels share their WAN interface `Ethernet0/0`, so **different GRE tunnel keys** distinguish the GRE streams. When IPsec is added, additional considerations for shared tunnel-source addresses and protection profiles apply; they are covered in the IPsec chapter.

### Complete Configuration: Hub1 (Cloud 100)

```cisco
hostname Hub1
!
interface Loopback0
 ip address 10.1.1.1 255.255.255.0
!
interface Ethernet0/0
 ip address 198.51.100.1 255.255.255.252
 no shutdown
!
interface Tunnel0
 ip address 172.16.0.1 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp map multicast dynamic
 ip nhrp network-id 100
 ip nhrp redirect
 no ip split-horizon eigrp 100
 bandwidth 100000
 delay 1000
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
ip route 0.0.0.0 0.0.0.0 198.51.100.2
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255
 network 10.1.1.1 0.0.0.0
```

### Complete Configuration: Hub2 (Cloud 200)

```cisco
hostname Hub2
!
interface Loopback0
 ip address 10.2.2.2 255.255.255.0
!
interface Ethernet0/0
 ip address 198.51.100.5 255.255.255.252
 no shutdown
!
interface Tunnel1
 ip address 172.16.1.2 255.255.255.0
 ip nhrp authentication NHRP200
 ip nhrp map multicast dynamic
 ip nhrp network-id 200
 ip nhrp redirect
 no ip split-horizon eigrp 100
 bandwidth 100000
 delay 1000
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 200
!
ip route 0.0.0.0 0.0.0.0 198.51.100.6
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel1
 network 172.16.1.0 0.0.0.255
 network 10.2.2.2 0.0.0.0
```

### Complete Configuration: Spoke1

```cisco
hostname Spoke1
!
interface Loopback0
 ip address 10.11.11.11 255.255.255.0
!
interface Ethernet0/0
 ip address 198.51.100.9 255.255.255.252
 no shutdown
!
interface Tunnel0
 ip address 172.16.0.11 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
 ip nhrp shortcut
 bandwidth 100000
 delay 1000
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
interface Tunnel1
 ip address 172.16.1.11 255.255.255.0
 ip nhrp authentication NHRP200
 ip nhrp network-id 200
 ip nhrp nhs 172.16.1.2 nbma 198.51.100.5 multicast
 ip nhrp shortcut
 bandwidth 100000
 delay 5000
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 200
!
ip route 0.0.0.0 0.0.0.0 198.51.100.10
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 no passive-interface Tunnel1
 network 172.16.0.0 0.0.0.255
 network 172.16.1.0 0.0.0.255
 network 10.11.11.11 0.0.0.0
 eigrp stub connected
```

### Complete Configuration: Spoke2

```cisco
hostname Spoke2
!
interface Loopback0
 ip address 10.12.12.12 255.255.255.0
!
interface Ethernet0/0
 ip address 198.51.100.13 255.255.255.252
 no shutdown
!
interface Tunnel0
 ip address 172.16.0.12 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
 ip nhrp shortcut
 bandwidth 100000
 delay 1000
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
interface Tunnel1
 ip address 172.16.1.12 255.255.255.0
 ip nhrp authentication NHRP200
 ip nhrp network-id 200
 ip nhrp nhs 172.16.1.2 nbma 198.51.100.5 multicast
 ip nhrp shortcut
 bandwidth 100000
 delay 5000
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 200
!
ip route 0.0.0.0 0.0.0.0 198.51.100.14
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 no passive-interface Tunnel1
 network 172.16.0.0 0.0.0.255
 network 172.16.1.0 0.0.0.255
 network 10.12.12.12 0.0.0.0
 eigrp stub connected
```

### Complete Configuration: Spoke3

```cisco
hostname Spoke3
!
interface Loopback0
 ip address 10.13.13.13 255.255.255.0
!
interface Ethernet0/0
 ip address 198.51.100.17 255.255.255.252
 no shutdown
!
interface Tunnel0
 ip address 172.16.0.13 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
 ip nhrp shortcut
 bandwidth 100000
 delay 1000
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
interface Tunnel1
 ip address 172.16.1.13 255.255.255.0
 ip nhrp authentication NHRP200
 ip nhrp network-id 200
 ip nhrp nhs 172.16.1.2 nbma 198.51.100.5 multicast
 ip nhrp shortcut
 bandwidth 100000
 delay 5000
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 200
!
ip route 0.0.0.0 0.0.0.0 198.51.100.18
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 no passive-interface Tunnel1
 network 172.16.0.0 0.0.0.255
 network 172.16.1.0 0.0.0.255
 network 10.13.13.13 0.0.0.0
 eigrp stub connected
```

### Why the Spokes Use Different Tunnel Delays

The dual-cloud design gives each spoke two independent tunnel interfaces. In this example:

```cisco
interface Tunnel0
 bandwidth 100000
 delay 1000
!
interface Tunnel1
 bandwidth 100000
 delay 5000
```

The **lower delay on Tunnel0 biases EIGRP toward routes learned through Hub1**, assuming the other metric components and advertised paths are comparable. If Hub1's route is withdrawn and Hub2's route remains eligible, EIGRP can select the route learned over Tunnel1.

EIGRP's composite metric is derived from bandwidth and delay by default. `bandwidth` is a routing metric input and does not police tunnel traffic; `delay` is a configurable routing metric input, not a measurement of actual WAN latency. Verify the computed feasible distance and successor rather than assuming a particular path from configuration alone.

To experiment with **equal-cost multipath**, set identical delay and bandwidth values on both spoke tunnels and inspect EIGRP's resulting metrics and installed routes. The result also depends on the metrics advertised by the hubs and the rest of the routing topology.

### Why the Hubs Do Not Need Overlay Adjacencies with One Another

Hub1 is reachable over Cloud 100; Hub2 is reachable over Cloud 200. They have **different overlay subnets and different NHRP domains**. Accordingly, there is no hub-to-hub EIGRP neighborship over `Tunnel0`/`Tunnel1` in the basic dual-cloud topology.

Each hub learns the connected LAN networks of spokes registered in its own cloud. This lets it forward initial traffic between those spokes within that cloud and send NHRP Redirects. Neither hub needs the other hub as its NHS.

However, **the absence of a hub-to-hub tunnel adjacency does not mean no inter-hub routing is ever necessary**. If the hubs also provide access to different headquarters or data-center networks, they need an appropriate core route-exchange design. Without such a path, the hubs might not know how to reach each other's private networks.

### Optional: Separate Inter-Hub Core Adjacency

If the lab includes a direct Ethernet0/1 connection between Hub1 and Hub2, use `192.0.2.0/30` for a separate EIGRP adjacency **outside the two DMVPN clouds**:

**Hub1:**

```cisco
interface Ethernet0/1
 ip address 192.0.2.1 255.255.255.252
 no shutdown
!
router eigrp 100
 network 192.0.2.0 0.0.0.3
 no passive-interface Ethernet0/1
```

**Hub2:**

```cisco
interface Ethernet0/1
 ip address 192.0.2.2 255.255.255.252
 no shutdown
!
router eigrp 100
 network 192.0.2.0 0.0.0.3
 no passive-interface Ethernet0/1
```

This is an **optional physical addition** to the reference topology, not part of the original ISP-only wiring. With it, the hubs can exchange `10.1.1.0/24` and `10.2.2.0/24` and additional headquarters prefixes independently of their separate DMVPN overlays.

Keep `eigrp stub connected` on the spokes so they advertise their local networks, not routes learned from one hub toward the other. The optional core link should carry hub-to-hub route exchange, rather than relying on spoke transit.

### Optional: Per-Cloud EIGRP Summaries

The hubs can each advertise a summary of reachable overlay destinations:

**Hub1:**

```cisco
interface Tunnel0
 ip summary-address eigrp 100 10.0.0.0 255.0.0.0
```

**Hub2:**

```cisco
interface Tunnel1
 ip summary-address eigrp 100 10.0.0.0 255.0.0.0
```

This provides covering EIGRP routes through the two clouds. The spoke can initially forward traffic to the selected hub, then establish a direct NHRP shortcut **within the cloud used for resolution**. Prefer narrower aggregates in a production design, and ensure each hub actually has more-specific routes for the traffic it attracts; otherwise its summary discard route can blackhole packets.

A spoke's static underlay default route to the ISP must **not** be confused with an overlay default route to an NHS. The ordinary underlay default allows packets to reach the hubs' public/NBMA addresses; it is not a safe substitute for a properly designed overlay route to private site prefixes.

### Understand the Two Independent Shortcut Domains

Suppose Spoke1 selects Hub1 as the overlay next hop for `10.12.12.12`:

1. Spoke1 sends traffic to `172.16.0.1` through its `Tunnel0`.
2. Hub1 forwards it toward Spoke2's `172.16.0.12` on Cloud 100 and sends an NHRP Redirect to Spoke1.
3. Spoke1 resolves the destination within Cloud 100.
4. Its direct GRE destination becomes Spoke2's NBMA `198.51.100.13`; forwarding uses `Tunnel0`.

If routing later selects Hub2 instead:

1. Spoke1 forwards to `172.16.1.2` through `Tunnel1`.
2. Hub2 redirects toward Spoke2's `172.16.1.12` within Cloud 200.
3. Spoke1 can resolve a **separate** Cloud 200 shortcut, still using NBMA `198.51.100.13` but carried by `Tunnel1` and GRE key `200`.

**NHRP shortcuts are associated with an NHRP cloud/tunnel interface.** A cached Cloud 100 mapping does not turn into a Cloud 200 shortcut merely because the same NBMA address is used by both clouds.

### Dual-Cloud Expected Adjacencies

| Router | Expected EIGRP peers over DMVPN |
| --- | --- |
| Hub1 `Tunnel0` | Spoke1 `172.16.0.11`, Spoke2 `.12`, Spoke3 `.13` |
| Hub2 `Tunnel1` | Spoke1 `172.16.1.11`, Spoke2 `.12`, Spoke3 `.13` |
| Spoke1 | Hub1 on Tunnel0; Hub2 on Tunnel1 |
| Spoke2 | Hub1 on Tunnel0; Hub2 on Tunnel1 |
| Spoke3 | Hub1 on Tunnel0; Hub2 on Tunnel1 |

There is **no Hub1-to-Hub2 tunnel adjacency** and **no spoke-to-spoke EIGRP adjacency** in the normal design.

### Dual-Cloud Verification Workflow

On a spoke:

```cisco
show ip interface brief
show interfaces Tunnel0
show interfaces Tunnel1
show ip nhrp nhs
show ip nhrp
show ip eigrp neighbors
show ip eigrp topology
show ip route 10.12.12.12
```

Check registrations separately:

```cisco
show dmvpn interface Tunnel0
show dmvpn interface Tunnel1
show ip nhrp shortcut
```

The intended results are:

1. Spoke1 has an NHS registration to Hub1 using **Tunnel0**.
2. Spoke1 has a separate NHS registration to Hub2 using **Tunnel1**.
3. Spoke1 has two EIGRP adjacencies, one on each tunnel.
4. The EIGRP route to a remote spoke's LAN prefers Tunnel0 with the configured metrics, while Tunnel1 remains an alternate path.
5. A completed NHRP redirect/resolution sequence establishes the direct shortcut in the selected cloud.

Confirm the actual data path:

```cisco
Spoke1# ping 10.12.12.12 source Loopback0 repeat 10
Spoke1# show ip nhrp shortcut
Spoke1# show ip route next-hop-override
Spoke1# show ip cef 10.12.12.12 detail
Spoke1# traceroute 10.12.12.12 numeric
```

A shortcut over Tunnel0 should resolve to Spoke2's **Cloud 100 tunnel IP `172.16.0.12`** and NBMA `198.51.100.13`. A shortcut over Tunnel1 should use **Cloud 200 tunnel IP `172.16.1.12`** and the same NBMA. In each case the tunnel interface and GRE key determine the cloud.

---

## Comparing the Configuration Models

| Configuration element | Single cloud | Dual cloud |
| --- | --- | --- |
| Hub1 tunnels | `Tunnel0` | `Tunnel0` |
| Hub2 tunnels | `Tunnel0` | `Tunnel1` |
| Spoke tunnels | One (`Tunnel0`) | Two (`Tunnel0`, `Tunnel1`) |
| Spoke NHS configuration | Both hubs under one tunnel | One NHS under each tunnel |
| Hub-to-hub NHRP mapping | Needed for a hub-to-hub tunnel adjacency | Not applicable across different clouds |
| Hub-to-hub overlay EIGRP | Present in this example | Absent |
| Routing peerings per spoke | Two peers on a shared tunnel | Two peers on separate tunnels |
| Use spoke tunnel delay to prefer a hub | Cannot distinguish peers on same interface | Can distinguish Tunnel0 and Tunnel1 |
| Direct spoke-to-spoke shortcuts | Within shared cloud | Within selected cloud |
| Routing process | EIGRP AS 100 | EIGRP AS 100 |
| Underlay default route | Toward ISP | Toward ISP |

### Common Configuration Errors

| Symptom | Likely cause | What to check |
| --- | --- | --- |
| Spoke registers with Hub1 but not Hub2 | Second NHS missing, wrong NBMA, or Hub2 not reachable | `show ip nhrp nhs`, `show dmvpn detail`, ISP reachability |
| Spoke has only one EIGRP adjacency in single cloud | Multicast mapping to second hub missing; hub not replicating Hellos | `show ip nhrp multicast`, `show ip eigrp neighbors` |
| Hubs do not form single-cloud EIGRP adjacency | Hub-to-hub mapping/registration or multicast replication absent | NHRP and multicast entries on **both** hubs |
| Dual-cloud Tunnel1 fails to exchange GRE | Wrong tunnel key, mismatched cloud settings, or underlay problem | `show run interface Tunnel1`, NHRP, GRE reachability |
| Dual-cloud spoke sees Hub1 on both tunnels | Cloud 200 NHS points to the wrong hub | `ip nhrp nhs` on Tunnel1 |
| Routes are present but traffic always passes through a hub | No Redirect, no shortcut, or forwarding override absent | `show ip nhrp traffic`, `show ip nhrp shortcut`, CEF |
| Phase 3 uses remote spoke as ordinary EIGRP next hop | Phase 2 `no ip next-hop-self eigrp 100` remains on the hub | `show running-config interface Tunnel...` |
| Hub forwards traffic toward incorrect cloud | Conflicting summaries or unintended transit routing | EIGRP topology, specific routes, summary Null0 routes |
| Different GRE clouds appear to share state | Reused GRE key or incorrect tunnel interface/NHRP domain | `tunnel key`, `ip nhrp network-id`, tunnel source |
| Cloud 200 works until IPsec is applied | IPsec profile/source association problem | Later IPsec/IKEv2 chapter; IKEv2/IPsec SAs |

### Troubleshooting Order

Use a consistent sequence rather than changing multiple parameters at once:

1. **Underlay:** Verify WAN interfaces, ISP next hops, and remote hub NBMA reachability.
2. **GRE:** Verify the correct source, `tunnel mode gre multipoint`, and GRE tunnel key for each cloud.
3. **NHRP registration:** Verify each spoke's NHS configuration and registration response independently.
4. **Multicast replication:** Verify the EIGRP Hello transport path from each hub to each intended neighbor.
5. **EIGRP:** Verify neighbors, stub state, learned prefixes, successors, and metric differences.
6. **Base forwarding:** Confirm the selected hub and outgoing tunnel for each destination.
7. **Phase 3 optimization:** Observe Redirect, Resolution Request/Reply, shortcut route or next-hop override, and CEF forwarding.
8. **Protection and failure handling:** After the unprotected lab works, add IPsec/IKEv2 and test the failover cases in the following chapters.

---

## Key Points

- **Single-cloud dual hub** uses one mGRE interface on every router and two NHS entries on each spoke.
- The single-cloud example makes **Hub2 an NHRP client of Hub1** so the hubs can resolve one another and establish an EIGRP adjacency on the shared overlay.
- Both single-cloud hubs still function as **NHSs for the spokes** and generate NHRP Redirects.
- **Dual-cloud dual hub** uses one cloud per hub and two mGRE interfaces on each spoke, with separate overlay subnets, NHRP network IDs, and GRE keys.
- In a dual-cloud design, the hubs **do not form a routing adjacency over their separate DMVPN tunnels**. An optional separate core link can provide hub-to-hub route exchange.
- Configure **`no ip split-horizon eigrp 100` on hub tunnels**, and keep ordinary **EIGRP next-hop-self** behavior for Phase 3.
- `eigrp stub connected` prevents the spokes from being general EIGRP transit devices while allowing them to advertise their connected LANs.
- In a **single cloud**, spoke Tunnel0 delay affects paths through both hubs. In **dual clouds**, independent tunnel delays can bias EIGRP toward the preferred hub.
- Both designs support NHRP Redirect/Shortcut. The **routing next hop** may remain a hub while **CEF** uses a direct shortcut to a spoke.
- In a dual-cloud design, a shortcut learned through Tunnel0 is **not interchangeable** with a shortcut learned through Tunnel1.
- IPsec/IKEv2 and detailed failure/convergence mechanisms are covered in their separate chapters.

## References

- [Cisco: Dynamic Multipoint IPsec VPNs (Using Multipoint GRE/NHRP to Scale IPsec VPNs)](https://www.cisco.com/c/en/us/support/docs/security-vpn/ipsec-negotiation-ike-protocols/41940-dmvpn.html)
- [Cisco: Troubleshoot DMVPN Issues](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/111976-dmvpn-troubleshoot-00.html)
- [Cisco: Configure Phase-3 Hierarchical DMVPN with Multi-Subnet Spokes](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/211292-Configure-Phase-3-Hierarchical-DMVPN-wit.html)
- [RFC 2332: NBMA Next Hop Resolution Protocol (NHRP)](https://www.rfc-editor.org/rfc/rfc2332)
