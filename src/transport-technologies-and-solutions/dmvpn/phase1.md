# DMVPN Phase 1

**DMVPN Phase 1** uses a **multipoint GRE (mGRE) tunnel on the hub** and **point-to-point GRE tunnels on the spokes**. The hub learns the spokes' tunnel-to-NBMA address mappings through NHRP registration, allowing it to communicate with many spokes through a single tunnel interface.

Phase 1 is a **hub-and-spoke forwarding design**. Traffic between two spokes must pass through the hub, even though the spokes can reach one another's WAN addresses through the underlay.

This page builds a complete Phase 1 network using **Hub1 and three spokes**, with **EIGRP** as the primary overlay routing protocol. OSPF and BGP configurations are presented as alternatives. IPsec/IKEv2 protection is covered separately in the Phase 3 section.

---

## Phase 1 Architecture

```text
                       DMVPN overlay
                           Hub1
                    Tunnel0: 172.16.0.1
                           (mGRE)
                         /   |   \\
                        /    |    \\
                       /     |     \\
                      /      |      \\
                   Spoke1  Spoke2  Spoke3
                    .11      .12     .13
                   (p2p)    (p2p)   (p2p)
               Tunnel subnet: 172.16.0.0/24
```

All four tunnel interfaces belong to the same overlay subnet, but the tunnel types differ:

| Router | Tunnel type | Remote GRE destination | NHRP role |

| --- | --- | --- | --- |

| Hub1 | mGRE | Selected dynamically using NHRP mappings | NHS |

| Spoke1 | Point-to-point GRE | Hub1 WAN address | NHC |

| Spoke2 | Point-to-point GRE | Hub1 WAN address | NHC |

| Spoke3 | Point-to-point GRE | Hub1 WAN address | NHC |

The **hub does not configure a `tunnel destination`**. Its mGRE interface uses NHRP information to identify the appropriate remote NBMA address for each spoke.

Each spoke **does configure a `tunnel destination`** pointing to Hub1. The spoke therefore sends all traffic entering its tunnel to Hub1, including packets destined for another spoke's LAN.

Unlike Phase 2 and Phase 3, Phase 1 does **not** dynamically create direct spoke-to-spoke GRE forwarding paths.

### Underlay Versus Overlay

The physical connections are unchanged from the chapter's lab topology:

```text
             Hub1                      Hub2
               \                        /
                \                      /
                 +-------- ISP --------+
                          / | \\
                         /  |  \\
                    Spoke1 Spoke2 Spoke3
```

**Hub2 is not used in this Phase 1 configuration.** It will be introduced in the later dual-hub labs.

The ISP provides ordinary IP connectivity between WAN addresses. It does not participate in NHRP or in the overlay routing protocol, and it does not have routes to the site LAN prefixes.

---

## Addressing Plan

The following addresses are already configured in the CML lab. The Phase 1 configuration adds `Tunnel0` and overlay routing.

| Router | WAN address | Tunnel0 address | Loopback0 (simulated LAN) |

| --- | --- | --- | --- |

| Hub1 | `198.51.100.1/30` | `172.16.0.1/24` | `10.1.1.1/24` |

| Spoke1 | `198.51.100.9/30` | `172.16.0.11/24` | `10.11.11.11/24` |

| Spoke2 | `198.51.100.13/30` | `172.16.0.12/24` | `10.12.12.12/24` |

| Spoke3 | `198.51.100.17/30` | `172.16.0.13/24` | `10.13.13.13/24` |

The ISP uses the other address in each WAN /30. The default routes on the DMVPN routers point toward the ISP.

For this lab:

- **NHRP network ID:** `100`

- **GRE tunnel key:** `100`

- **NHRP authentication string:** `NHRP100`

- **EIGRP AS:** `100`

The NHRP network ID identifies the local NHRP domain/interface context; it is **not transmitted as an NHRP network identifier that must match between routers**. Consistent values across this cloud make the configuration easier to manage. The **GRE tunnel key**, when configured, is carried in GRE packets and must match for successful tunnel communication. The GRE key is **not** a cryptographic key.

---

## Complete Phase 1 Configuration with EIGRP

This configuration includes the underlay, simulated LANs, mGRE/point-to-point GRE, NHRP, and EIGRP. Each router's configuration is independent and can be applied directly to the corresponding device in the lab.

### ISP

```cisco
hostname ISP
!
interface Ethernet0/0
 ip address 198.51.100.2 255.255.255.252
 no shutdown
!
interface Ethernet0/1
 ip address 198.51.100.6 255.255.255.252
 no shutdown
!
interface Ethernet0/2
 ip address 198.51.100.10 255.255.255.252
 no shutdown
!
interface Ethernet0/3
 ip address 198.51.100.14 255.255.255.252
 no shutdown
!
interface Ethernet1/0
 ip address 198.51.100.18 255.255.255.252
 no shutdown
```

All WAN subnets are connected to the ISP, so no static routes are necessary on this router. The `198.51.100.6/30` connection to Hub2 remains configured but is not used by the Phase 1 overlay.

### Hub1

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

Key hub commands:

- `tunnel mode gre multipoint` allows **one tunnel interface** to communicate with all three spokes.

- `ip nhrp map multicast dynamic` instructs the hub to replicate multicast traffic to registered spoke NBMA addresses. This enables multicast-based routing protocol exchanges over mGRE.

> In modern IOS XE, `ip nhrp map multicast dynamic` is enabled by default, so you don't actually have to configure it.

- `ip nhrp network-id 100` enables NHRP on the tunnel interface.

- `no ip split-horizon eigrp 100` allows the hub to advertise a route learned from one spoke back out `Tunnel0` toward the other spokes.

- The default **EIGRP next-hop-self behavior is retained**, so remote spoke LAN routes advertised by the hub use the hub as the next hop. That is appropriate for Phase 1.

There is no `ip nhrp nhs` command on Hub1 because **Hub1 is the NHS**, not a client registering with another NHS.

### Spoke1

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
 ip nhrp map 172.16.0.1 198.51.100.1
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.1
 tunnel key 100
!
ip route 0.0.0.0 0.0.0.0 198.51.100.10
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255
 network 10.11.11.11 0.0.0.0
```

### Spoke2

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
 ip nhrp map 172.16.0.1 198.51.100.1
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.1
 tunnel key 100
!
ip route 0.0.0.0 0.0.0.0 198.51.100.14
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255
 network 10.12.12.12 0.0.0.0
```

### Spoke3

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
 ip nhrp map 172.16.0.1 198.51.100.1
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.1
 tunnel key 100
!
ip route 0.0.0.0 0.0.0.0 198.51.100.18
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255
 network 10.13.13.13 0.0.0.0
```

The spokes use **point-to-point GRE**, the default GRE-over-IPv4 tunnel mode. There is deliberately no `tunnel mode gre multipoint` on any spoke.

Their configured `tunnel destination` forces all GRE traffic through Hub1.

The NHRP commands on each spoke have different jobs:

| Command | Purpose |

| --- | --- |

| `ip nhrp map 172.16.0.1 198.51.100.1` | Static mapping of Hub1's overlay IP address to its WAN NBMA address. |

| `ip nhrp nhs 172.16.0.1` | Configures Hub1 as the next-hop server to which the spoke registers. |

| `tunnel destination 198.51.100.1` | Fixes the GRE transport endpoint at Hub1, regardless of the destination site LAN. |

Alternatively, the first two NHRP commands can be combined into **one command** on each spoke:

```cisco
ip nhrp nhs 172.16.0.1 nbma 198.51.100.1
```

This specifies both the NHS tunnel address and its NBMA address, replacing the separate `ip nhrp map 172.16.0.1 198.51.100.1` and `ip nhrp nhs 172.16.0.1` commands.

It does **not** replace `tunnel destination 198.51.100.1`: Phase 1 spokes still need their fixed point-to-point GRE destination.

The `multicast` keyword is unnecessary on Phase 1 spokes because point-to-point GRE sends multicast traffic to that fixed destination.

> In Phases 2 and 3, mGRE spokes can use the `ip nhrp nhs ... nbma ... multicast` form when multicast replication toward the hub is required.

On a point-to-point GRE interface, the tunnel destination is already known; the static NHRP mapping identifies Hub1 for NHRP purposes. **The spoke does not need to resolve the NBMA addresses of other spokes** to forward Phase 1 traffic.

### Why the Hub Needs a Multicast Mapping but the Spokes Do Not

Hub1 uses **mGRE**, which has no single configured `tunnel destination`. When it sends an EIGRP multicast packet, it needs to identify which remote NBMA destinations should receive a GRE-encapsulated copy.

`ip nhrp map multicast dynamic` instructs Hub1 to replicate multicast traffic to dynamically registered spokes.

Each Phase 1 spoke uses **point-to-point GRE** with `tunnel destination 198.51.100.1`. Both unicast and multicast packets sent into that tunnel are encapsulated toward the same fixed destination, Hub1. The spokes therefore **do not need** `ip nhrp map multicast`.

In Phases 2 and 3, the spokes use mGRE instead of point-to-point GRE. They then need a multicast destination mapping for routing protocols such as EIGRP, which can be configured with `ip nhrp map multicast` or the appropriate `ip nhrp nhs ... nbma ... multicast` form.

---

## NHRP Registration and Hub Forwarding

After the spoke's tunnel becomes operational, it registers its tunnel address and NBMA address with Hub1.

For example, Spoke2 registers:

```text
NHRP Registration Request
Tunnel address: 172.16.0.12
NBMA address:   198.51.100.13
NHS:            172.16.0.1 (Hub1)
```

Hub1 can then maintain dynamic mappings resembling:

| Tunnel address | NBMA address | Mapping type |

| --- | --- | --- |

| `172.16.0.11` | `198.51.100.9` | Dynamic registration |

| `172.16.0.12` | `198.51.100.13` | Dynamic registration |

| `172.16.0.13` | `198.51.100.17` | Dynamic registration |

The NHRP cache allows Hub1's single mGRE interface to select the correct GRE outer destination for each spoke.

### Why the Hub Needs Multicast Replication

EIGRP uses multicast (`224.0.0.10`) for certain neighbor discovery and routing exchanges. Hub1's mGRE interface has no single fixed destination, so it must know where to send a copy of each required multicast packet.

With `ip nhrp map multicast dynamic`, Hub1 can send separate GRE copies to each registered spoke:

```text
                       Hub1
             EIGRP multicast packet
                         |
              +----------+----------+
              |          |          |
            copy 1     copy 2     copy 3
              |          |          |
            Spoke1     Spoke2     Spoke3
```

An NHRP registration alone does not advertise the spoke's LAN prefixes. **NHRP provides endpoint reachability; EIGRP provides routes.**

### NHRP Registration Versus Resolution

In this Phase 1 design:

- **Registration** is required so Hub1 learns each spoke's tunnel-to-NBMA mapping.

- The spokes have a **static mapping for Hub1** and a fixed GRE tunnel destination.

- Spokes do not perform dynamic NHRP spoke-to-spoke resolution to create direct paths.

- Hub1's dynamic registrations can expire and be refreshed according to NHRP timers.

NHRP Resolution Requests and Replies become especially important when the spokes use mGRE in **Phase 2**.

---

## EIGRP Routing Behavior

The hub forms three EIGRP neighbor relationships across `Tunnel0`. The spokes form neighbor relationships with Hub1, **not with one another**.

```text
            EIGRP AS 100
                 Hub1
              /   |   \\
             /    |    \\
         Spoke1 Spoke2 Spoke3
  No direct EIGRP spoke-to-spoke neighbors
```

### Why EIGRP Split Horizon Must Be Disabled on the Hub

EIGRP split horizon normally prevents a router from advertising a route back out the interface on which it was learned.

Hub1 learns `10.11.11.0/24` from Spoke1 on `Tunnel0`. Without an exception, it would not advertise that route to Spoke2 or Spoke3 because those neighbors are also reached via `Tunnel0`.

Configure the exception **on the hub tunnel interface**:

```cisco
interface Tunnel0
 no ip split-horizon eigrp 100
```

The same exception is not needed on the spokes because each has a single EIGRP neighbor on its point-to-point tunnel.

### Keep Next-Hop-Self Enabled

Consider the route to Spoke2's LAN learned by Spoke1:

```text
Destination:  10.12.12.0/24
Next hop:     172.16.0.1 (Hub1)
Outgoing:     Tunnel0
```

Hub1 uses its NHRP mapping for Spoke2 to forward the packet onward. The **hub must remain the forwarding next hop** in Phase 1.

Do **not** configure the following Phase 2-specific command in this Phase 1 lab:

```cisco
no ip next-hop-self eigrp 100
```

Preserving the originating spoke as the advertised next hop is useful for Phase 2 direct forwarding, but it is not required or desirable for the basic Phase 1 design.

### EIGRP Summarization on the Hub

**Phase 1 supports route summarization on the hub.** Because spokes already forward all traffic for remote spoke networks through the hub, they do not need individual routes to every other spoke LAN. A summary route reduces the number of EIGRP routes advertised to each spoke.

In this lab, Hub1 and the three spokes use LAN prefixes within `10.0.0.0/8`. To advertise one aggregate instead of the individual LAN prefixes, add the following optional command on **Hub1**:

```cisco
interface Tunnel0
 ip summary-address eigrp 100 10.0.0.0 255.0.0.0
```

Each spoke can then learn `10.0.0.0/8` with **Hub1 (`172.16.0.1`) as the next hop**. Hub1 still retains the more-specific routes learned from the spokes and uses them to forward packets to the correct destination. A locally generated EIGRP summary route to `Null0` (administrative distance **5**) discards traffic matching the summary when no more-specific route exists.

This summary is intentionally broad for the lab. In a production network, choose an aggregate that matches the actual address plan to avoid unnecessarily covering unused destinations. The command is **not included in the base configurations above**, so those examples continue to advertise the individual /24 networks.

**Phase comparison:** Summarization is compatible with Phase 1 because all spoke-to-spoke forwarding traverses the hub. In Phase 2, summarization toward the spokes generally prevents direct spoke-to-spoke forwarding because the route points to the hub instead of the destination spoke. Phase 3 restores the ability to summarize while still using NHRP shortcuts for direct forwarding.

### End-to-End Packet Forwarding

Assume Spoke1 sends an ICMP echo request sourced from `10.11.11.11` to `10.12.12.12`.

1. **Spoke1** looks up `10.12.12.12` and selects its EIGRP route via `172.16.0.1`, `Tunnel0`.

2. Spoke1 encapsulates the packet in GRE with **outer destination `198.51.100.1`**, Hub1's WAN IP.

3. The **ISP** forwards the GRE packet to Hub1 based on the outer header.

4. **Hub1** decapsulates the packet and looks up the original destination `10.12.12.12`.

5. Hub1 selects `Tunnel0` toward Spoke2 and resolves **`172.16.0.12` to `198.51.100.13`** using NHRP.

6. Hub1 encapsulates the packet again, now with outer destination **`198.51.100.13`**.

7. Spoke2 decapsulates it and delivers it to `Loopback0`.

```text
Overlay path:
10.11.11.11             Hub1                10.12.12.12
  Spoke1 --------------> Hub1 --------------> Spoke2
         GRE to Hub1           GRE to Spoke2
Underlay GRE destinations:
Spoke1 -> 198.51.100.1  (Hub1)
Hub1   -> 198.51.100.13 (Spoke2)
```

**The hub decapsulates and re-encapsulates spoke-to-spoke traffic.** A spoke does not directly encapsulate a packet for another spoke in Phase 1, even if it knows the other spoke's WAN IP address through underlay routing.

---

## Alternative Routing Protocol: OSPF

OSPF works well for Phase 1, but the tunnel **OSPF network type** matters because the hub is mGRE while the spokes are point-to-point GRE.

### Recommended OSPF Network Type for Phase 1

Use **point-to-multipoint** on the hub and spokes:

```cisco
interface Tunnel0
 ip ospf network point-to-multipoint
```

This models the topology as individual logical point-to-point neighbor relationships, does not use a DR/BDR election, and allows spoke LAN routes to use Hub1 as the forwarding next hop.

| Network type | Phase 1 consideration |
| --- | --- |
| Point-to-multipoint | Natural hub-and-spoke topology; no DR/BDR; recommended here. |
| Broadcast | Possible in some DMVPN designs, but introduces DR/BDR and network-model constraints; commonly used for Phase 2. |
| Point-to-point | Appropriate for each spoke's GRE endpoint individually, but should not be used as a common network model on a hub serving multiple neighbors. |

### Replacing EIGRP with OSPF

Remove EIGRP from all four routers before applying OSPF:

```cisco
no router eigrp 100
```

**Hub1:**

```cisco
interface Loopback0
 ip ospf network point-to-point
!
interface Tunnel0
 ip ospf network point-to-multipoint
!
router ospf 100
 router-id 10.1.1.1
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255 area 0
 network 10.1.1.1 0.0.0.0 area 0
```

**Spoke1:**

```cisco
interface Loopback0
 ip ospf network point-to-point
!
interface Tunnel0
 ip ospf network point-to-multipoint
!
router ospf 100
 router-id 10.11.11.11
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255 area 0
 network 10.11.11.11 0.0.0.0 area 0
```

**Spoke2:**

```cisco
interface Loopback0
 ip ospf network point-to-point
!
interface Tunnel0
 ip ospf network point-to-multipoint
!
router ospf 100
 router-id 10.12.12.12
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255 area 0
 network 10.12.12.12 0.0.0.0 area 0
```

**Spoke3:**

```cisco
interface Loopback0
 ip ospf network point-to-point
!
interface Tunnel0
 ip ospf network point-to-multipoint
!
router ospf 100
 router-id 10.13.13.13
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255 area 0
 network 10.13.13.13 0.0.0.0 area 0
```

All OSPF tunnel interfaces are in **area 0**. Hub1 should have three **FULL** OSPF neighbors on `Tunnel0`; each spoke should have one FULL neighbor, Hub1.

The `ip ospf network point-to-point` command on **Loopback0** has a separate purpose from the point-to-multipoint setting on `Tunnel0`. OSPF normally advertises a loopback as a **/32**, even if the interface is configured with a /24 address. Setting its OSPF network type to point-to-point causes the **actual /24 subnet** to be advertised in this lab.

#### OSPF Phase 1 Versus Phase 2

In Phase 1, OSPF point-to-multipoint is suitable because spoke-to-spoke forwarding is supposed to pass through the hub. In Phase 2, OSPF **broadcast** network type with a hub DR and spoke priorities of 0 is commonly used to support routing next-hop behavior for direct spoke paths. Phase 3 can again use point-to-multipoint because NHRP shortcuts optimize the forwarding independently of the original OSPF next hop.

---

## Alternative Routing Protocol: BGP

BGP can also exchange the spoke LAN prefixes over Phase 1 DMVPN. For a simple example, use **eBGP** with a different AS on each spoke. This avoids introducing iBGP route-reflector behavior into the initial Phase 1 lab.

| Router | BGP AS | BGP peer(s) |
| --- | --- | --- |
| Hub1 | `65000` | Spoke1, Spoke2, Spoke3 |
| Spoke1 | `65101` | Hub1 |
| Spoke2 | `65102` | Hub1 |
| Spoke3 | `65103` | Hub1 |

Use the existing `Tunnel0` interfaces and underlay configuration. Remove EIGRP or OSPF first so this test has a single overlay routing protocol.

### Hub1 eBGP Configuration

```cisco
router bgp 65000
 no bgp default ipv4-unicast
 neighbor 172.16.0.11 remote-as 65101
 neighbor 172.16.0.12 remote-as 65102
 neighbor 172.16.0.13 remote-as 65103
 address-family ipv4 unicast
  network 10.1.1.0 mask 255.255.255.0
  neighbor 172.16.0.11 activate
  neighbor 172.16.0.11 next-hop-self
  neighbor 172.16.0.12 activate
  neighbor 172.16.0.12 next-hop-self
  neighbor 172.16.0.13 activate
  neighbor 172.16.0.13 next-hop-self
 exit-address-family
```

### Spoke1 eBGP Configuration

```cisco
router bgp 65101
 no bgp default ipv4-unicast
 neighbor 172.16.0.1 remote-as 65000
 address-family ipv4 unicast
  network 10.11.11.0 mask 255.255.255.0
  neighbor 172.16.0.1 activate
 exit-address-family
```

### Spoke2 eBGP Configuration

```cisco
router bgp 65102
 no bgp default ipv4-unicast
 neighbor 172.16.0.1 remote-as 65000
 address-family ipv4 unicast
  network 10.12.12.0 mask 255.255.255.0
  neighbor 172.16.0.1 activate
 exit-address-family
```

### Spoke3 eBGP Configuration

```cisco
router bgp 65103
 no bgp default ipv4-unicast
 neighbor 172.16.0.1 remote-as 65000
 address-family ipv4 unicast
  network 10.13.13.0 mask 255.255.255.0
  neighbor 172.16.0.1 activate
 exit-address-family
```

Each `network` statement matches an **existing connected /24 route** from `Loopback0`. The BGP peers use directly connected tunnel addresses, so **`ebgp-multihop` is not required** in this example.

The explicit `neighbor ... next-hop-self` commands make Hub1 advertise itself as the next hop for routes learned from the other spokes. This is important because **eBGP can preserve a third-party next hop** when the original next hop and the receiving peer share a subnet, as they do in this overlay. Setting next-hop-self avoids relying on that behavior and makes the intended hub-routed Phase 1 forwarding explicit. Spoke-to-spoke traffic therefore crosses Hub1.


### What About iBGP and Route Reflectors?

An iBGP design is also possible. All routers use the same AS, and Hub1 acts as a **route reflector (RR)** for the three spokes.

Without route reflection, the hub would not advertise routes learned from one iBGP spoke to another iBGP spoke.

**Hub1:**

```cisco
router bgp 65000
 bgp router-id 10.1.1.1
 no bgp default ipv4-unicast
 neighbor 172.16.0.11 remote-as 65000
 neighbor 172.16.0.12 remote-as 65000
 neighbor 172.16.0.13 remote-as 65000
 address-family ipv4 unicast
  network 10.1.1.0 mask 255.255.255.0
  neighbor 172.16.0.11 activate
  neighbor 172.16.0.11 route-reflector-client
  neighbor 172.16.0.11 next-hop-self all
  neighbor 172.16.0.12 activate
  neighbor 172.16.0.12 route-reflector-client
  neighbor 172.16.0.12 next-hop-self all
  neighbor 172.16.0.13 activate
  neighbor 172.16.0.13 route-reflector-client
  neighbor 172.16.0.13 next-hop-self all
 exit-address-family
```

The spokes use the same AS (`65000`) and establish iBGP sessions with Hub1. They do not need the `route-reflector-client` command.

**Key commands:**

- `neighbor ... route-reflector-client` configures each spoke as an RR client, allowing Hub1 to reflect routes learned from one spoke to the other spokes.
- `neighbor ... next-hop-self all` changes the BGP next hop to Hub1's tunnel address, **including for reflected iBGP routes**.
- Ordinary `neighbor ... next-hop-self` does not change the next hop of reflected iBGP routes.

For example, Spoke2 learns Spoke1's LAN prefix (`10.11.11.0/24`) through Hub1, with BGP next hop `172.16.0.1`.

The `next-hop-self all` commands ensure that the BGP next hops match Phase 1's **hub-and-spoke forwarding topology**. Since the spokes use point-to-point GRE tunnels with fixed destinations, spoke-to-spoke traffic must pass through Hub1.

Although route reflection is necessary for this iBGP design, next-hop rewriting is a separate consideration. The default RR behavior is to preserve the original BGP next hop of reflected routes.

---

## Verification

### 1. Verify Underlay Reachability

From Spoke1:

```cisco
ping 198.51.100.1 source Ethernet0/0
ping 198.51.100.13 source Ethernet0/0
ping 198.51.100.17 source Ethernet0/0
```

All three WAN destinations should be reachable through the ISP. This proves the **underlay** is reachable; it does not prove that the DMVPN overlay works.

### 2. Verify the Tunnel Interfaces

```cisco
show ip interface brief
show interfaces Tunnel0
show running-config interface Tunnel0
```

Confirm that Hub1 uses `tunnel mode gre multipoint` and the spokes use fixed `tunnel destination 198.51.100.1` commands. Verify that tunnel source and destination routes resolve through the ISP, not back through a tunnel.

### 3. Verify NHRP

On Hub1:

```cisco
show ip nhrp
show dmvpn
show ip nhrp traffic
```

Look for **three registered spokes** and the expected NBMA addresses. For example, Hub1's NHRP cache should contain entries corresponding to:

```text
172.16.0.11 -> 198.51.100.9
172.16.0.12 -> 198.51.100.13
172.16.0.13 -> 198.51.100.17
```

On each spoke:

```cisco
show ip nhrp
show ip nhrp nhs detail
show dmvpn
```

Verify the static NHS mapping and successful registration. NHRP detail and flag formatting can vary by platform and software version.

### 4. Verify Overlay Routing

With EIGRP:

```cisco
show ip eigrp neighbors
show ip route eigrp
show ip protocols
```

With OSPF:

```cisco
show ip ospf neighbor
show ip ospf interface Tunnel0
show ip route ospf
```

With BGP:

```cisco
show bgp ipv4 unicast summary
show bgp ipv4 unicast
show ip route bgp
```

The expected overlay neighbor counts are:

| Router | EIGRP neighbors | OSPF neighbors | eBGP neighbors |

| --- | ---: | ---: | ---: |

| Hub1 | 3 | 3 | 3 |

| Each spoke | 1 | 1 | 1 |

### 5. Confirm the Actual Spoke-to-Spoke Path

From Spoke1:

```cisco
show ip route 10.12.12.12
traceroute 10.12.12.12 source 10.11.11.11
ping 10.12.12.12 source 10.11.11.11
```

With the EIGRP configuration, the next hop for Spoke2's LAN should be **Hub1 (`172.16.0.1`)**. A traceroute should reveal Hub1 as the intermediate overlay router before the destination. Depending on the device's ICMP behavior, the reported source addresses and hop visibility can vary.

On Hub1, the route for `10.12.12.0/24` should point to **Spoke2 (`172.16.0.12`)** through `Tunnel0`.

When Phase 1 is functioning normally:

- Spoke1 reaches Spoke2's LAN through Hub1.

- Hub1 maintains NHRP registrations for all three spokes.

- Spoke1 does **not** install a dynamic NHRP mapping or direct mGRE tunnel to Spoke2 for spoke-to-spoke forwarding.

---

## Troubleshooting Phase 1

| Symptom | Possible cause | Verification or correction |
| --- | --- | --- |
| Spoke's tunnel is down | WAN source down, hub NBMA unreachable, recursive route | Check `show ip interface brief`, WAN routing, and tunnel source/destination. |
| Hub has no NHRP registration from a spoke | NHRP misconfiguration, authentication mismatch, GRE filtering, or reachability failure | Check `show ip nhrp nhs detail` on spoke and `show ip nhrp` on hub. |
| Hub has a registration but no EIGRP neighbors | Missing multicast replication mapping, EIGRP not enabled on tunnel, passive tunnel, AS mismatch | Check NHRP multicast configuration and `show ip eigrp neighbors`. |
| Spoke can reach hub LAN but not other spoke LANs | EIGRP split horizon enabled at hub, missing advertised networks, or hub forwarding issue | Verify `no ip split-horizon eigrp 100` on Hub1 and both routing tables. |
| OSPF neighbors do not become FULL | Tunnel network-type mismatch, area/authentication/timer mismatch, or missing GRE multicast delivery | Check `show ip ospf interface Tunnel0` and neighbor states. |
| BGP session established but LAN prefix missing | Incorrect `network` statement or no exact matching route in the local RIB | Check `show bgp ipv4 unicast` and the connected loopback prefix. |
| NHRP works but spoke-to-spoke ping fails | Missing overlay routes, source address selection, or return-path issue | Ping with a loopback source; inspect the route on both spokes and the hub. |
| GRE control traffic passes but large packets fail | MTU/fragmentation/PMTUD issue | Check tunnel MTU, DF behavior, and TCP MSS adjustment as appropriate. |

Useful targeted debugs include:

```cisco
debug nhrp
debug nhrp detail
debug eigrp packets
```

Use debugs selectively. Packet capture on the ISP-facing links is particularly useful for distinguishing **GRE transport reachability**, **NHRP control traffic**, and **overlay routing behavior**.

### Tunnel Status Does Not Guarantee NHRP or Routing

A GRE tunnel can show `up/up` even when the remote router is not correctly processing the tunnel traffic. A functional Phase 1 deployment requires **all** of the following:

1. Reachability through the underlay to the hub's NBMA address.

2. Matching GRE tunnel encapsulation and key parameters.

3. Successful NHRP spoke registrations at the hub.

4. Overlay routing protocol adjacencies and route exchange.

5. Correct forward and return paths through Hub1.

GRE tunnel keepalives are **not supported for DMVPN tunnels**; rely on the relevant routing protocol, NHRP, and, when configured, IKEv2/IPsec mechanisms to detect and diagnose failures.

---

## Phase 1 Limitations and Transition to Phase 2

Phase 1's simplicity comes with limitations:

- **Hub hairpinning:** Inter-spoke traffic is decapsulated, routed, and re-encapsulated by the hub.

- **Hub resource requirements:** The hub forwards all spoke-to-spoke traffic, which can become a bandwidth and processing bottleneck.

- **No direct spoke-to-spoke GRE:** Point-to-point spoke tunnels always use their configured hub destination.

- **Hub dependency:** Failure of the only active hub interrupts the overlay until another design provides recovery.

Phase 2 changes the **spoke tunnels from point-to-point GRE to mGRE**, allowing NHRP to resolve the NBMA address of a remote spoke and establish direct spoke-to-spoke forwarding.

That change also introduces important **routing next-hop requirements**. For EIGRP, the hub generally must both disable split horizon and **preserve the originating spoke as the next hop** for routes that should use direct spoke-to-spoke paths.

See [DMVPN Phase 2](phase2.md) for the next stage and [DMVPN Phase 3](phase3/overview.md) for NHRP redirects and shortcuts.

---

## Key Points

- **Phase 1 uses mGRE on the hub and point-to-point GRE on the spokes.**

- The hub uses **one Tunnel interface** for all spokes; the spokes each configure a fixed **`tunnel destination` pointing to the hub**.

- Spokes **register** their tunnel/NBMA mappings with the hub's NHRP NHS.

- The hub uses registered mappings to select the NBMA destination for GRE traffic toward each spoke.

- `ip nhrp map multicast dynamic` lets the hub replicate routing protocol multicast traffic to registered spokes.

- Phase 1 spokes do **not** need `ip nhrp map multicast` because point-to-point GRE sends multicast traffic toward the configured `tunnel destination`.

- Phase 1 does **not** use direct spoke-to-spoke tunnels. Spoke-to-spoke traffic always traverses the hub.

- With EIGRP, the hub typically needs **`no ip split-horizon eigrp <as>`** so it can advertise spoke-learned routes back out its mGRE interface.

- EIGRP **next-hop-self remains enabled** in Phase 1 because the hub should remain the next hop for remote spoke prefixes.
- **EIGRP summarization on the hub is supported in Phase 1**; spokes can use a summary route via Hub1, which continues forwarding traffic to the destination spoke.

- **OSPF point-to-multipoint** is a suitable tunnel network type in Phase 1 and requires no DR/BDR election.

- OSPF advertises loopbacks as **/32s** by default; `ip ospf network point-to-point` on a loopback advertises its configured subnet.

- **eBGP** can provide simple hub-and-spoke routing. Configure `next-hop-self` on the hub to avoid third-party next-hop behavior on the shared tunnel subnet; an iBGP design requires route-reflection and additional next-hop considerations.

- NHRP resolves overlay **tunnel addresses to NBMA addresses**; it does **not** exchange site LAN routes.

- GRE and NHRP do **not** provide encryption. IPsec/IKEv2 protection is introduced later.

---

## References

- [Cisco: IP Routing Configuration Guide, EIGRP Route Summarization](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_ire-enhanced-igrp.html)

- [Cisco: Dynamic Multipoint IPsec VPNs (Using Multipoint GRE/NHRP to Scale IPsec VPNs)](https://www.cisco.com/c/en/us/support/docs/security-vpn/ipsec-negotiation-ike-protocols/41940-dmvpn.html)

- [Cisco: Troubleshoot DMVPN Issues](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/111976-dmvpn-troubleshoot-00.html)

- [Cisco: DMVPN Phase 1 Debugs Troubleshoot Guide](https://www.cisco.com/c/en/us/support/docs/security-vpn/dynamic-multi-point-vpn-dmvpn/116957-technote-dmvpn-00.html)

- [Cisco: Dynamic Multipoint VPN Configuration Guide](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-conn-dmvpn-dmvpn-0.html)

- [Cisco: Review OSPF Frequently Asked Questions](https://www.cisco.com/c/en/us/support/docs/ip/open-shortest-path-first-ospf/9237-9.html)

- [Cisco: BGP Next Hop Unchanged (Third-Party Next-Hop Behavior)](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-next-hop-0.html)
