# Routing over DMVPN Phase 3

In **DMVPN Phase 3**, the overlay routing protocol provides reachability through a **hub**, while **NHRP Redirects and Shortcuts** can optimize traffic onto a direct spoke-to-spoke path. Unlike Phase 2, the source spoke does **not** need a routing-protocol route whose next hop is the destination spoke's tunnel address.

This separation allows the hub to advertise **summarized routes**, and, with suitable underlay/overlay routing separation, **default routes**, without necessarily forcing all spoke-to-spoke data traffic to remain on the hub. However, the capabilities of the routing protocol still matter: EIGRP, OSPF, and BGP differ in how they advertise routes, handle next hops, and support summarization.

This page focuses on **routing behavior and configuration**. For detailed NHRP control messages, shortcut prefix selection, and Route Watch, see [NHRP Redirects and Shortcuts](nhrp/redirect-shortcut.md). The later [Dual-Hub DMVPN Topologies](dual-hub/topologies.md) and [Dual-Hub Phase 3 Configuration](dual-hub/configuration.md) pages cover complete redundant designs.

---

## Phase 3 Routing Model

```text
                       Hub1 (NHS)
                    172.16.0.1/24
                    /     |     \
                   /      |      \
             Spoke1     Spoke2     Spoke3
          172.16.0.11    .12        .13
                \_________/
               Direct shortcut
           after NHRP resolution
```

All four routers use **mGRE**. Spokes register with Hub1 and establish their routing-protocol adjacencies or sessions **with the hub**, not with one another.

| Function | Normal mechanism | Example for Spoke1 to Spoke2 |
| --- | --- | --- |
| Learn destination network | EIGRP, OSPF, BGP, or static routing | `10.12.12.0/24` or covering `10.0.0.0/8` |
| Initial overlay next hop | Hub1 | `172.16.0.1` |
| First data path | Through Hub1 | Spoke1 → Hub1 → Spoke2 |
| Shortcut discovery | NHRP Redirect followed by Resolution | Resolves a destination behind Spoke2 |
| Optimized forwarding | NHRP route or next-hop override | Direct GRE to Spoke2's NBMA address |

### Phase 2 Versus Phase 3

| Routing consideration | Phase 2 | Phase 3 |
| --- | --- | --- |
| Required next hop for direct spoke traffic | Destination spoke's tunnel IP | Hub next hop is acceptable |
| EIGRP next-hop-self on hub | Normally **disabled** | Normally **enabled** (default) |
| EIGRP split horizon on hub | Normally disabled | Normally disabled when re-advertising spoke routes |
| Hub summary routes to spokes | Defeat direct forwarding for covered destinations | Compatible with NHRP shortcuts |
| OSPF tunnel network type | Commonly **broadcast** | Commonly **point-to-multipoint** |
| Need a routing adjacency between spokes | No | No |
| Mechanism for initial direct path | Resolve route's remote-spoke next hop | Hub Redirect and destination-prefix Resolution |

**The routing table and the actual forwarding next hop can differ.** A Phase 3 spoke may retain an EIGRP/OSPF/BGP route via the hub while an NHRP next-hop override directs matching traffic to a remote spoke. A more-specific shortcut may instead appear as an `H` route.

---

## Reference Lab and Addressing

The topology is the same as the Phase 1/2 chapters. **Hub2 is not yet active** in the base example; the final sections discuss how it changes routing.

| Device | WAN / NBMA | Tunnel0 | Loopback0 (LAN) |
| --- | --- | --- | --- |
| Hub1 | `198.51.100.1/30` | `172.16.0.1/24` | `10.1.1.1/24` |
| Spoke1 | `198.51.100.9/30` | `172.16.0.11/24` | `10.11.11.11/24` |
| Spoke2 | `198.51.100.13/30` | `172.16.0.12/24` | `10.12.12.12/24` |
| Spoke3 | `198.51.100.17/30` | `172.16.0.13/24` | `10.13.13.13/24` |

The ISP interfaces are `198.51.100.2`, `.10`, `.14`, and `.18`. Each DMVPN router has an underlay default route via its local ISP neighbor. The ISP does not know the overlay LAN prefixes.

For the following configurations:

- **Tunnel interface:** `Tunnel0`
- **NHRP network ID and GRE tunnel key:** `100`
- **NHRP authentication:** `NHRP100`
- **EIGRP AS / OSPF process:** `100`

A working mGRE/NHRP foundation is required before any overlay routing protocol will operate correctly. In the configurations below, `ip nhrp redirect` belongs on the hub, and `ip nhrp shortcut` belongs on the spokes.

---

## Complete Phase 3 Configuration with EIGRP

This is the primary routing lab. It adds Phase 3 forwarding to the Phase 2 EIGRP design, **restores the hub's default EIGRP next-hop-self behavior**, and optionally introduces an EIGRP summary afterward.

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

The hub **does not** have `no ip next-hop-self eigrp 100`. Thus, spoke-learned routes re-advertised by Hub1 use **Hub1's tunnel address** as the next hop. `no ip split-horizon eigrp 100` permits those routes to be advertised back out `Tunnel0` to other spokes.

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
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
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
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
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
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
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
```

The combined `ip nhrp nhs ... nbma ... multicast` command identifies Hub1 as the NHS, supplies its NBMA address, and enables multicast forwarding toward it. It replaces the three separate spoke commands used in the Phase 2 example (`ip nhrp map`, `ip nhrp nhs`, and `ip nhrp map multicast`). 

### Migrating from the Phase 2 EIGRP Lab

Keep the existing tunnel sources, mGRE interfaces, addresses, NHRP mappings, and EIGRP processes. The essential changes are:

**Hub1:**

```cisco
interface Tunnel0
 ip next-hop-self eigrp 100
 ip nhrp redirect
```

**Each spoke:**

```cisco
interface Tunnel0
 ip nhrp shortcut   <-- Enabled by default
```

`ip next-hop-self eigrp 100` restores next-hop-self if Phase 2 configured `no ip next-hop-self eigrp 100`. **Do not remove the hub's split-horizon exception** if it is still expected to advertise individual spoke-learned EIGRP routes to other spokes.

---

## EIGRP Route Advertisement and Forwarding

### Why Split Horizon and Next-Hop-Self Are Separate

These features solve different problems:

| Feature | Hub behavior | Phase 3 recommendation |
| --- | --- | --- |
| EIGRP split horizon | Stops a route learned through `Tunnel0` from being advertised back out `Tunnel0` | Disable if advertising specific spoke-learned routes to other spokes |
| EIGRP next-hop-self | Replaces the advertised next hop with Hub1's address | Leave enabled so ordinary spoke routes use the hub |
| NHRP Redirect | Notifies a source spoke that forwarding through the hub is suboptimal | Enable on hub |
| NHRP Shortcut | Allows an NHRP-resolved optimized path | Enable on spokes |

With split horizon disabled and next-hop-self enabled, Spoke1 can learn:

```text
D 10.12.12.0/24 [90/...] via 172.16.0.1, Tunnel0
```

The route initially forwards through Hub1. Once Spoke1 obtains the appropriate NHRP shortcut, forwarding can use Spoke2 without changing EIGRP neighbor relationships.

**A topology with only a hub-originated summary or default route may not need the split-horizon exception for that advertisement**, because the hub originates that advertisement rather than re-advertising each spoke-learned route. Keep the distinction between hub-originated summaries and spoke-learned specifics in mind when troubleshooting missing routes.

### EIGRP Summarization on Hub1

Configure the following **optional** summary on Hub1:

```cisco
interface Tunnel0
 ip summary-address eigrp 100 10.0.0.0 255.0.0.0
```

All simulated LANs fall within `10.0.0.0/8`. The hub continues to learn the individual spoke subnets, but advertises the aggregate out its tunnel interface.

Spoke1 might therefore have:

```text
D 10.0.0.0/8 [90/...] via 172.16.0.1, Tunnel0
```

This summary **covers** Spoke2's `10.12.12.0/24`, but Spoke1 has no specific EIGRP route to that subnet. A packet sent to `10.12.12.12` initially travels through Hub1. Hub1 forwards it toward Spoke2 and can generate an NHRP Redirect; the subsequent Resolution Reply can install a **more-specific `H` shortcut** for `10.12.12.0/24`.

Spoke1's routing table then contains:

```text
D        10.0.0.0/8 [90/27008000] via 172.16.0.1, 00:00:15, Tunnel0
H        10.12.12.0/24 [250/255] via 172.16.0.12, 00:00:15, Tunnel0
```

**Longest-prefix matching**, not administrative distance, makes the `/24` shortcut win over the `/8` EIGRP summary for the affected destination.

On Hub1, EIGRP's interface summarization normally creates a local summary-discard route to **Null0**, with an administrative distance of **5**, to prevent routing loops for addresses covered by the summary but not by a more-specific route. Hub1's specific connected and EIGRP routes still win by longest-prefix matching for valid site networks.

```cisco
show ip route 10.0.0.0 255.0.0.0
show ip route eigrp
show ip eigrp topology
```

An overly broad summary can unintentionally cover unrelated networks or create black holes where Hub1 has no more-specific route. In production, summarize only address blocks that the hub can actually reach and is responsible for advertising.

### EIGRP Summary Versus NHRP Summary Map

These two features operate at different layers:

| Feature | Configured on | Changes |
| --- | --- | --- |
| `ip summary-address eigrp` | Hub tunnel interface | **EIGRP advertisements** toward spokes |
| `ip nhrp summary-map` | Destination spoke tunnel interface | **Prefix returned in NHRP Resolution Replies** |

For example, Hub1 might advertise `10.0.0.0/8`, while Spoke2 answers resolution requests for its connected `10.12.12.0/24`. If Spoke2 owns multiple subnets in an aggregatable block, an NHRP summary map can reduce the number of separate shortcuts needed. See [NHRP Redirects and Shortcuts](nhrp/redirect-shortcut.md) for the configuration and safety considerations.

### EIGRP Stub Spokes

In a larger EIGRP hub-and-spoke deployment, making ordinary branch routers **EIGRP stubs** can reduce query scope:

```cisco
router eigrp 100
 eigrp stub connected summary
```

This still permits the spoke to advertise connected networks and summaries. It does not disable NHRP shortcuts or require an EIGRP adjacency with other spokes. Do not apply a restrictive stub advertisement policy that suppresses routes the hub needs to reach that branch.

### Advertising a Default Route: Underlay Versus Overlay

Phase 3 can work with a hub-advertised `0.0.0.0/0` covering remote spoke networks, **but this lab already has a static default route on each spoke pointing to its ISP**. That route normally has AD 1 and wins over an EIGRP-learned default (AD 90), so an EIGRP default from the hub will not automatically be used for overlay traffic.

A straightforward solution for this lab is to advertise the **`10.0.0.0/8` overlay summary**, leaving the physical default route to carry the NBMA underlay. Designs that rely on an overlay default must deliberately separate or control underlay routing, often with specific NBMA host routes or separate transport VRFs, to avoid recursive tunnel routing.

---

## Alternative Routing Protocol: OSPF

### Recommended Network Type: Point-to-Multipoint

For Phase 3, configure **`ip ospf network point-to-multipoint`** on the hub and spokes. This models the overlay as individual hub-to-spoke links, avoids a DR/BDR election, and causes the initial SPF path to another spoke to run **through the hub**.

```text
OSPF topology (before NHRP optimization):

Spoke1 -------- Hub1 -------- Spoke2

OSPF next hop on Spoke1: 172.16.0.1
```

By comparison, the **broadcast** network type commonly used in Phase 2 models the tunnel subnet as a common transit network and can result in a remote spoke being selected directly as the OSPF next hop. Phase 3 does not need that behavior. It allows the OSPF next hop to be the hub and uses NHRP to optimize forwarding later.

Point-to-multipoint is a suitable choice **even though all tunnel interfaces have addresses in the same `/24`**. The OSPF network model, not the configured IP mask alone, controls the logical topology represented in OSPF LSAs.

### Replacing EIGRP with OSPF

Remove EIGRP on the hub and spokes before configuring this alternative:

```cisco
no router eigrp 100
```

The NHRP and mGRE configurations remain unchanged. On the hub, remove the now-unused EIGRP split-horizon command if it is still present.

**Hub1:**

```cisco
interface Loopback0
 ip ospf network point-to-point
!
interface Tunnel0
 ip ospf network point-to-multipoint
 ip nhrp redirect
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
 ip nhrp shortcut
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
 ip nhrp shortcut
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
 ip nhrp shortcut
!
router ospf 100
 router-id 10.13.13.13
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255 area 0
 network 10.13.13.13 0.0.0.0 area 0
```

With this configuration, Hub1 should form **three FULL OSPF adjacencies**, and each spoke should form **one**, with Hub1. There is **no DR/BDR**, and spokes do not need to form OSPF adjacencies with other spokes.

The `ip ospf network point-to-point` command on **Loopback0** serves a different purpose: it allows OSPF to advertise the configured `/24` rather than its default loopback `/32` host route.

### Important Limitation: OSPF Summarization

**Phase 3 supports route summarization, but OSPF cannot summarize intra-area routes at an ABR.** In this single-cloud example, Hub1 and all spokes share **area 0**. Spoke LAN prefixes are intra-area routes, and OSPF floods their link-state information within the area.

Therefore, placing all spokes in one area does **not** let Hub1 replace all spoke LAN prefixes with a single OSPF `area ... range` summary. ABR summarization applies to **inter-area routes**, and ASBR summarization applies to **redistributed external routes**.

This means the Phase 3 *forwarding mechanism* allows summaries, but the **single-area OSPF control plane** does not gain the same intra-area route-reduction benefit as EIGRP or BGP. Other designs involving additional areas and appropriate ABR placement can summarize routes at area boundaries.

A hub could originate a default route into a suitable OSPF area, but the spoke's existing physical static default may still be selected over the OSPF default. Do not confuse capability to advertise a default with its **actual selection in the RIB**.

---

## Alternative Routing Protocol: BGP

BGP can also provide the overlay routes. Because BGP uses **unicast TCP sessions**, it does not require NHRP multicast mapping **for BGP itself**. NHRP still requires the spoke's unicast mapping to its NHS and successful registration.

The routing goal in this Phase 3 example is for spokes to learn remote prefixes or aggregates **through Hub1**. Unlike the Phase 2 BGP example, there is no need to preserve the originating spoke's tunnel address as the BGP NEXT_HOP.

### Option 1: eBGP on Hub and Spokes

Use AS `65000` on Hub1 and a distinct AS for each spoke:

| Router | AS |
| --- | --- |
| Hub1 | `65000` |
| Spoke1 | `65101` |
| Spoke2 | `65102` |
| Spoke3 | `65103` |

Remove the current IGP before testing this alternative. Use the same working GRE/NHRP interfaces, including `ip nhrp redirect` and `ip nhrp shortcut`.

**Hub1:**

```cisco
router bgp 65000
 bgp router-id 10.1.1.1
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

**Spoke1:**

```cisco
router bgp 65101
 bgp router-id 10.11.11.11
 no bgp default ipv4-unicast
 neighbor 172.16.0.1 remote-as 65000
 address-family ipv4 unicast
  network 10.11.11.0 mask 255.255.255.0
  neighbor 172.16.0.1 activate
 exit-address-family
```

**Spoke2:**

```cisco
router bgp 65102
 bgp router-id 10.12.12.12
 no bgp default ipv4-unicast
 neighbor 172.16.0.1 remote-as 65000
 address-family ipv4 unicast
  network 10.12.12.0 mask 255.255.255.0
  neighbor 172.16.0.1 activate
 exit-address-family
```

**Spoke3:**

```cisco
router bgp 65103
 bgp router-id 10.13.13.13
 no bgp default ipv4-unicast
 neighbor 172.16.0.1 remote-as 65000
 address-family ipv4 unicast
  network 10.13.13.0 mask 255.255.255.0
  neighbor 172.16.0.1 activate
 exit-address-family
```

The explicit `neighbor ... next-hop-self` commands are intentional. **Third-party next-hop behavior** can cause eBGP to preserve an originating spoke's NEXT_HOP because all tunnel peers share `172.16.0.0/24`. In Phase 3, explicitly setting the next hop to **Hub1** ensures the desired initial path for the NHRP Redirect mechanism.

With the above configuration, Spoke1 should learn Spoke2's `10.12.12.0/24` with BGP NEXT_HOP `172.16.0.1`. A subsequent NHRP shortcut may use an **NHO** because the resolved prefix matches the existing route, or an `H` route if it is more specific than a covering BGP route.

### BGP Aggregation on Hub1

Rather than advertising each spoke's subnet, Hub1 can advertise a **locally originated aggregate** for the DMVPN LANs. Under `address-family ipv4 unicast` on Hub1:

```cisco
router bgp 65000
 address-family ipv4 unicast
  aggregate-address 10.0.0.0 255.0.0.0 summary-only
 exit-address-family
```

BGP requires contributing more-specific prefixes in its BGP table to create the aggregate. The `summary-only` option suppresses advertisement of the contributing specific prefixes to neighbors, while Hub1 retains those routes for forwarding to the correct spokes.

This example is deliberately broad because all simulated lab LANs are within `10.0.0.0/8`. The aggregate must be chosen carefully in a real network, and the hub must maintain usable specific routes for destinations inside it. An aggregate alone does not make unreachable addresses inside the summary reachable.

**For a separate-AS eBGP design**, ensure the aggregate and any advertised contributing routes obey the applicable AS-path loop-prevention and policy rules. Here the spokes have different AS numbers, and Hub1 learns their specific `/24` prefixes before originating the summary.

A route such as `B 10.0.0.0/8 via 172.16.0.1` on Spoke1 can trigger the same redirected forwarding pattern as the EIGRP summary example.

### Option 2: iBGP with Hub1 as a Route Reflector

All routers can use AS `65000`. Hub1 acts as an **iBGP route reflector (RR)** and the spokes are its RR clients. In Phase 3, arrange for the hub to be the initial NEXT_HOP when advertising reflected spoke routes.

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

For the spokes, use the same `router bgp` and `network` structure as above, but with **AS 65000** on each spoke and on its neighbor statement:

```cisco
router bgp 65000
 bgp router-id 10.11.11.11
 no bgp default ipv4-unicast
 neighbor 172.16.0.1 remote-as 65000
 address-family ipv4 unicast
  network 10.11.11.0 mask 255.255.255.0
  neighbor 172.16.0.1 activate
 exit-address-family
```

Spoke2 and Spoke3 substitute their own router IDs and `network` prefixes.

**Why `all`?** An RR ordinarily **preserves the NEXT_HOP** of an iBGP route it reflects. Ordinary `next-hop-self` does not necessarily rewrite reflected routes; `next-hop-self all` explicitly rewrites both ordinary and reflected routes on IOS/IOS XE implementations supporting that syntax. This makes the initial hub-routed behavior intentional, rather than depending on recursive reachability to a third-party next hop.

An alternative is to have Hub1 advertise a BGP aggregate (`aggregate-address ... summary-only`) to spokes, which makes Hub1 the originator of the aggregate and the corresponding NEXT_HOP. Be aware that BGP aggregation and route reflection have different path-selection and advertisement implications.

---

## Static Routing and Route Coverage

A static summary route can also provide the **initial hub path** without an IGP or BGP:

**Spoke1, Spoke2, and Spoke3:**

```cisco
ip route 10.0.0.0 255.0.0.0 172.16.0.1
```

**Hub1:**

```cisco
ip route 10.11.11.0 255.255.255.0 172.16.0.11
ip route 10.12.12.0 255.255.255.0 172.16.0.12
ip route 10.13.13.0 255.255.255.0 172.16.0.13
```

Keep the existing NHRP configuration on the tunnels. The hub needs specific routes to forward traffic toward the destination spoke and generate the Redirect; spokes need the covering route toward Hub1. The hub next hops require valid NHRP registrations to obtain their NBMA destinations.

This is useful for isolating NHRP Phase 3 behavior from dynamic routing-protocol problems. A **route via Hub1 alone does not establish a shortcut** unless the initial forwarding path and NHRP Redirect/Resolution process are operating.

---

## Shortcut Routes, Next-Hop Overrides, and Route Watch

The base route supplied by EIGRP, OSPF, BGP, or static routing affects how a resolved NHRP shortcut appears in the routing table.

| Existing route on Spoke1 | NHRP-resolved shortcut | Result |
| --- | --- | --- |
| `10.0.0.0/8` via Hub1 | `10.12.12.0/24` via Spoke2 | **New `H` route** for the more-specific `/24` |
| `10.12.12.0/24` via Hub1 | `10.12.12.0/24` via Spoke2 | **Next-hop override**, `%` marker and `[NHO]` |
| Existing route is more specific than NHRP response | Less-specific NHRP information | Can use a next-hop override rather than installing a less-specific shortcut route |

### More-Specific NHRP Route

For a summary learned via Hub1:

```text
D 10.0.0.0/8 [90/...] via 172.16.0.1, Tunnel0
H 10.12.12.0/24 [250/...] via 172.16.0.12, Tunnel0
```

Even with its higher AD, the `H` route wins for destinations within its `/24` because it is **more specific**.

### NHRP Next-Hop Override

For a specific route with the same prefix length as the resolved shortcut, the routing-protocol route can remain visible with the **`%` next-hop override indicator**, while its forwarding next hop is rewritten:

```text
D  % 10.12.12.0/24 [90/...] via 172.16.0.1, Tunnel0
                         [NHO] via 172.16.0.12, Tunnel0
```

This is **illustrative output**, not a literal capture. Display formatting varies, but the important distinction is that the routing-protocol route is still present while NHRP changes its forwarding next hop.

### Route Watch and Covering Routes

NHRP shortcuts depend on the ordinary routing information that makes the destination reachable in the first place. If the covering route disappears, changes its usable outgoing tunnel, or becomes unsuitable, **Route Watch can invalidate the associated shortcut** rather than leaving stale direct forwarding indefinitely.

A new routing-protocol route can also supersede an NHRP route through **longest-prefix matching**, or change the context in which a next-hop override is applied. During failover, inspect both the routing table and NHRP state; do not assume that a previously working shortcut remains valid just because its timer has not expired.

Useful commands:

```cisco
show ip route 10.12.12.12
show ip route nhrp
show ip route next-hop-override
show ip nhrp shortcut
show ip cef 10.12.12.12 detail
```

See [Phase 3 Operation and Packet Forwarding](operation.md) and [NHRP Redirects and Shortcuts](nhrp/redirect-shortcut.md) for the complete life cycle.

---

## Dual-Hub Routing Considerations

Phase 3's ability to route through a hub before discovering a shortcut makes **dual-hub designs** more flexible than Phase 2. However, **NHRP registration does not by itself choose the active routing path**. Routing policy, route metrics, BGP attributes, and the tunnel topology determine which hub receives a spoke's initial traffic.

### Single-Cloud Dual Hub

Both hubs and spokes share the same overlay (`172.16.0.0/24`) on one tunnel interface per spoke:

```text
                Hub1 (.1)       Hub2 (.2)
                    \           /
                     \         /
                   Spoke1 (.11)
                    (Tunnel0)
```

A spoke can register with **both NHSs** but learn different candidate paths to the same remote network. The selected route points toward the chosen hub, and a redirect from that hub can initiate a direct shortcut.

For EIGRP, costs toward Hub1 and Hub2 influence path selection. For OSPF, interface/network costs and LSDB topology determine the SPF path. For BGP, ordinary BGP best-path rules determine which route is selected. Configure **both hubs** for NHRP Redirect if either is expected to forward hairpinned spoke traffic.

### Dual-Cloud Dual Hub

Each spoke has separate mGRE tunnel interfaces, commonly `Tunnel0` toward Hub1 and `Tunnel1` toward Hub2, with different overlay subnets and GRE keys:

```text
Spoke1
  Tunnel0 -> Hub1, 172.16.0.0/24, key 100
  Tunnel1 -> Hub2, 172.16.1.0/24, key 200
```

The routing protocol can prefer one tunnel, use both where the design permits, or fail over to the other based on metrics and policy. **NHRP shortcut state is associated with a specific DMVPN interface/cloud**. A shortcut on one cloud does not automatically transfer to the other after routing convergence.

### Failover and Asymmetric Paths

When the primary hub or overlay route fails:

1. The routing protocol selects an alternative path through the remaining hub/cloud.
2. Existing shortcut state may need to be invalidated or refreshed according to Route Watch, route selection, and NHRP timers.
3. New flows can initially traverse the new hub and trigger new Redirect/Resolution exchanges.
4. Return traffic can select a different route or hub, so verify **both directions** independently.

A spoke-to-spoke IPsec security association is separate from the routing adjacency and NHRP shortcut. A working new route does not guarantee that a new direct protected session is already established.

The exact single-cloud and dual-cloud configurations, routing preferences, and convergence tests appear in [Dual-Hub DMVPN Topologies](dual-hub/topologies.md), [Dual-Hub Phase 3 Configuration](dual-hub/configuration.md), and [Dual-Hub Failover and Convergence](dual-hub/failover.md).

---

## Routing Verification Workflow

Perform these checks in order. **Validate the route that initiates forwarding, then validate the shortcut that optimizes it.**

### 1. Confirm the Underlay Is Independent

On Spoke1:

```cisco
show ip route 198.51.100.1
show ip route 198.51.100.13
ping 198.51.100.13 source Ethernet0/0
```

The NBMA destination must resolve over the physical underlay, not recursively through the mGRE tunnel.

### 2. Confirm NHRP Registration and Routing Neighbors

```cisco
show dmvpn
show ip nhrp nhs detail
show ip nhrp
show ip eigrp neighbors
show ip ospf neighbor
show bgp ipv4 unicast summary
```

Use only the routing protocol currently configured in the lab. EIGRP/OSPF should ordinarily show the hub as the spoke's neighbor; BGP should show the configured hub session established.

### 3. Examine the Original Route

On Spoke1:

```cisco
show ip route 10.12.12.12
show ip cef 10.12.12.12 detail
```

For the Phase 3 examples, the **initial route via the hub** should be observable before the shortcut is installed. If the route already points directly at Spoke2, investigate an inherited Phase 2 next-hop policy or existing shortcut state.

### 4. Generate Traffic and Check the Shortcut

On Spoke1:

```cisco
ping 10.12.12.12 source 10.11.11.11
show ip nhrp shortcut
show ip route nhrp
show ip route next-hop-override
show ip cef 10.12.12.12 detail
```

On Hub1:

```cisco
show ip nhrp traffic
```

Look for **Redirects sent by Hub1** and Resolution activity on spokes, then compare the RIB and FIB. Traceroute and WAN packet captures can help prove the **direct GRE outer destination** after convergence.

### 5. Observe Route and Shortcut Changes

To study a changed next hop or summary, observe the route before and after the change, then check the NHRP cache:

```cisco
show ip route 10.12.12.12
show ip nhrp shortcut
show ip nhrp traffic
```

A routing update does not always immediately remove every NHRP entry; inspect the effective forwarding state rather than relying on a single timer or show command.

---

## Common Routing Problems

| Symptom | Likely routing cause | Check or correction |
| --- | --- | --- |
| Spoke reaches Hub1 but has no other spoke LAN routes | EIGRP split horizon; missing route advertisement or BGP reflection; OSPF area/network mismatch | Check protocol neighbor status and advertised prefixes |
| EIGRP remote spoke next hop remains remote spoke in an unsummarized Phase 3 design | Hub retained `no ip next-hop-self eigrp 100` from Phase 2 | Restore default `ip next-hop-self eigrp 100` |
| Spoke has a summary via Hub1 but no shortcut | Redirect/Resolution failed, destination lacks a route, or no direct underlay path | Check hub Redirect counters, source Resolution, return Reply, shortcut state |
| An `H` route appears even though a protocol route exists | Shortcut prefix is more specific than the existing covering route | Compare both prefix lengths and longest-prefix matching |
| Protocol route shows `%` and `[NHO]` | NHRP overrode the next hop of a matching route | Inspect `show ip route next-hop-override` and CEF |
| OSPF uses unexpected DR/BDR election | Phase 2 broadcast network type still configured | Configure point-to-multipoint on hub and spokes |
| OSPF does not reduce intra-area spoke routes to one hub summary | OSPF cannot summarize intra-area LSAs at the hub | Use appropriate ABR/area design or another protocol if this is a requirement |
| eBGP spokes learn remote routes via the remote spoke | Third-party NEXT_HOP preserved on shared tunnel subnet | Configure hub `neighbor ... next-hop-self` for the Phase 3 design |
| RR clients learn remote routes with original spoke NEXT_HOP | RR preserved reflected routes' NEXT_HOP | Consider `next-hop-self all` or advertise hub-originated aggregate |
| Hub summary route causes traffic to disappear | Hub has no specific route for a destination inside the summary; summary discard to Null0 | Check longest match on hub and prefix ownership |
| Hub-advertised default is not installed on spoke | Static ISP default route has lower AD | Use overlay summaries or deliberate underlay/overlay separation |
| Routing converges to secondary hub but shortcut remains stale or disappears | Shortcut tied to old covering route or cloud | Check Route Watch, new route, interface, and NHRP state |
| Routing and NHRP look correct, but large packets or encrypted traffic fail | GRE/IPsec, underlay, MTU or SA issue rather than route advertisement | Inspect GRE encapsulation, IPsec state, MTU/PMTUD separately |

---

## Key Points

- **Phase 3 separates ordinary routing next-hop selection from optimized spoke-to-spoke forwarding.** A hub next hop does not prevent direct NHRP shortcuts.
- **EIGRP:** Normally disable split horizon on the hub when re-advertising spoke routes, but keep **next-hop-self enabled**. Hub summarization is compatible with shortcuts.
- **EIGRP summary:** The hub keeps specific routes, advertises a covering prefix, and can have a summary discard route to Null0. A more-specific NHRP `H` route on a spoke can override the summary through longest-prefix matching.
- **OSPF:** Point-to-multipoint is recommended for the Phase 3 hub-and-spoke topology; no DR/BDR is elected. OSPF **cannot summarize intra-area spoke prefixes** merely because Phase 3 supports shortcuts.
- **eBGP:** Explicit `neighbor ... next-hop-self` on the hub prevents unwanted third-party next-hop behavior in the shared tunnel subnet.
- **iBGP:** A route reflector normally preserves reflected NEXT_HOP; `next-hop-self all` or a hub-originated aggregate can establish the intended Phase 3 hub next hop.
- **NHRP `H` routes** arise when a resolved prefix is more specific than the existing covering route. A matching route can retain its protocol code with a **`%` / `[NHO]`** next-hop override.
- **Do not confuse an EIGRP/BGP summary with `ip nhrp summary-map`.** The former changes routing advertisements, while the latter changes shortcut prefix information in NHRP replies.
- **Overlay default routes require care** when spokes already have underlay defaults pointing to the ISP. In the reference lab, a `10.0.0.0/8` overlay summary is simpler.
- **Dual-hub registration and routing selection are separate decisions.** After failover, inspect the new route, the effective forwarding next hop, and per-cloud NHRP shortcut state.
- A correct route alone is not proof of successful direct forwarding. Check **RIB, FIB, NHRP, underlay NBMA reachability, and IPsec where configured**.

---

## References

- [Cisco: Troubleshoot DMVPN Issues](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/111976-dmvpn-troubleshoot-00.html)
- [Cisco: Shortcut Switching Enhancements for NHRP in DMVPN Networks](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-addressing/ip-addressing/m_nhrp-switch-enhancemts-dmvpn-0.html)
- [Cisco: Configure BGP over DMVPN Phase 3](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/222585-configure-bgp-over-dmvpn-phase-3.html)
- [Cisco: Spoke-to-Spoke NHRP Summary Maps](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m-sec-conn-dmvpn-summ-maps.html)
- [Cisco: BGP neighbor next-hop-self Command Reference](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [Cisco: Dynamic Multipoint VPN Configuration Guide](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-conn-dmvpn-dmvpn-0.html)
