# Phase 3 Operation and Packet Forwarding

**DMVPN Phase 3** allows spokes to learn remote networks **through the hub** while forwarding traffic **directly to other spokes** after NHRP discovers a shortcut. Unlike Phase 2, the routing protocol does not have to advertise each remote spoke's tunnel address as the next hop. The hub can remain the routing next hop and can advertise summarized routes.

The essential mechanism is **NHRP Redirect and Shortcut**:

1. A source spoke initially forwards traffic to the hub using its normal overlay route.
2. The hub forwards that traffic toward the destination spoke and notifies the source spoke of the suboptimal path with an **NHRP Redirect**.
3. The source spoke sends an **NHRP Resolution Request** for the original packet's destination.
4. The destination spoke returns the information needed for a **direct spoke-to-spoke path**.
5. NHRP installs shortcut forwarding information, so subsequent packets bypass the hub.

This page examines the forwarding process in detail, including routing-table behavior, CEF, NHRP shortcut routes, next-hop overrides, and the implications of dual hubs. The detailed NHRP command options and troubleshooting procedures are covered in the following pages.

---

## Phase 3 Architecture

Like Phase 2, Phase 3 uses **multipoint GRE (mGRE) on both the hub and the spokes**. There is no fixed `tunnel destination` on any of these routers.

```text
                           Hub1
                     172.16.0.1/24
                         (mGRE)
                        /   |   \
                       /    |    \
                  Spoke1  Spoke2  Spoke3
                   .11     .12     .13
                  (mGRE)  (mGRE)  (mGRE)
                      \____/
                   Direct path
                 after shortcut

               Tunnel subnet: 172.16.0.0/24
```

The **routing topology** normally remains hub-and-spoke: each spoke exchanges routing updates with Hub1, not with the other spokes. The **forwarding topology** can become a partial mesh as traffic triggers shortcuts.

| Property | Phase 2 | Phase 3 |
| --- | --- | --- |
| Hub tunnel | mGRE | mGRE |
| Spoke tunnels | mGRE | mGRE |
| Normal routing next hop for a remote spoke LAN | Destination spoke's tunnel IP | Hub's tunnel IP can be used |
| First packet toward a remote spoke LAN | Triggers resolution of the remote tunnel next hop | Follows the ordinary route through the hub |
| Trigger for direct forwarding | NHRP resolution of the routing next hop | NHRP Redirect followed by NHRP resolution |
| Hub route summarization | Defeats direct forwarding for summarized destinations | Compatible with direct forwarding through shortcuts |
| Shortcut route / next-hop override | Not the defining mechanism | Central forwarding feature |

**Direct spoke-to-spoke forwarding does not require spoke-to-spoke EIGRP, OSPF, or BGP neighborships.** The hub still distributes the routes; NHRP optimizes the forwarding path independently.

### Lab Addressing

The examples use the same underlay and overlay addresses as the Phase 1 and Phase 2 labs.

| Router | NBMA / WAN address | Tunnel0 | Loopback0 / site LAN |
| --- | --- | --- | --- |
| Hub1 | `198.51.100.1` | `172.16.0.1/24` | `10.1.1.0/24` |
| Spoke1 | `198.51.100.9` | `172.16.0.11/24` | `10.11.11.0/24` |
| Spoke2 | `198.51.100.13` | `172.16.0.12/24` | `10.12.12.0/24` |
| Spoke3 | `198.51.100.17` | `172.16.0.13/24` | `10.13.13.0/24` |

Hub2 (`172.16.0.2`, NBMA `198.51.100.5`) is introduced in the dual-hub pages. For now, **Hub1 is the only active NHS**. The ISP supplies reachability between the NBMA addresses and has no routes to the site LAN prefixes.

---

## Phase 3 Commands That Affect Forwarding

Phase 3 builds on the same mGRE and NHRP registration mechanisms as Phase 2. Two commands enable the characteristic forwarding behavior.

**Hub1:**

```cisco
interface Tunnel0
 ip nhrp redirect
```

**Each spoke:**

```cisco
interface Tunnel0
 ip nhrp shortcut
```

`ip nhrp redirect` permits the hub to notify a source spoke when forwarding through the hub is unnecessary. `ip nhrp shortcut` enables the spoke to use NHRP-discovered shortcut paths for forwarding.

> As mentioned in the previous page, `ip nhrp shortcut` is **enabled** by default. `ip nhrp redirect`, however, is **not**.

The rest of the baseline NHRP configuration is still needed. For example, with multicast EIGRP:

**Hub1:**

```cisco
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
```

**Spoke1:**

```cisco
interface Tunnel0
 ip address 172.16.0.11 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
 ip nhrp shortcut
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
```

The other spokes use their own tunnel addresses and the same NHS information. On the hub, `ip nhrp map multicast dynamic` enables multicast replication to registered spokes. On the spokes, the `multicast` keyword in `ip nhrp nhs ... nbma ... multicast` permits multicast routing-protocol traffic to reach Hub1.

**EIGRP next-hop-self remains enabled on the hub in Phase 3.** Do not carry over the Phase 2-specific `no ip next-hop-self eigrp 100` command when the goal is to demonstrate hub-next-hop forwarding and NHRP shortcuts. The split-horizon exception is still needed when the hub advertises individual spoke-learned routes out the same tunnel interface; summarization can change which advertisements are necessary.

The complete dual-hub and IKEv2/IPsec configurations appear later in this chapter.

---

## Initial State: Routing Through the Hub

Before sending traffic to Spoke2, Spoke1 may have **no NHRP entry for Spoke2's LAN**. It only needs:

- A working underlay route to Hub1's NBMA address.
- A successful NHRP registration with Hub1.
- A route toward Spoke2's LAN through the DMVPN tunnel, normally via Hub1.

For example, Hub1 could advertise an EIGRP summary covering the spoke LANs:

```cisco
interface Tunnel0
 ip summary-address eigrp 100 10.0.0.0 255.0.0.0
```

Spoke1 can then have a route resembling:

```text
D 10.0.0.0/8 via 172.16.0.1, Tunnel0
```

The next hop is **Hub1 (`172.16.0.1`)**, not Spoke2 (`172.16.0.12`). The local `10.11.11.0/24` connected route still takes precedence for Spoke1's own LAN because it is more specific than `10.0.0.0/8`.

The summary is optional. Phase 3 also works if Spoke1 learns the individual route `10.12.12.0/24` with **Hub1 as the next hop**. Summarization is important because it demonstrates that Phase 3 no longer depends on distributing every remote spoke prefix with the destination spoke as next hop.

### Two Different Destinations

For a packet sent by Spoke1 from `10.11.11.11` to `10.12.12.12`, distinguish:

| Field | Initial value | Meaning |
| --- | --- | --- |
| Original packet destination | `10.12.12.12` | Host at Spoke2's site |
| Overlay route / forwarding next hop | `172.16.0.1` | Hub1 tunnel address |
| GRE outer destination | `198.51.100.1` | Hub1 NBMA address |

**The original packet destination is not the same as the route's next hop or the GRE outer destination.** This distinction is essential for understanding why the Phase 3 NHRP request can discover Spoke2 even when the route initially points at Hub1.

---

## Phase 3 Packet-Forwarding Sequence

Consider an ICMP echo request from **Spoke1's `10.11.11.11`** to **Spoke2's `10.12.12.12`**.

```text
  Spoke1                       Hub1                       Spoke2
  172.16.0.11                  172.16.0.1                  172.16.0.12
  198.51.100.9                 198.51.100.1                198.51.100.13
      |                           |                            |
      |--- First data packet ---->|                            |
      |  GRE to Hub1              |--- Forward data packet --->|
      |                           |   GRE to Spoke2            |
      |<--- NHRP Redirect --------|                            |
      |                           |                            |
      |--- Resolution Request --->|                            |
      |   Target: 10.12.12.12     |--- Resolution Request ---->|
      |                           |                            |
      |<----------------------- Resolution Reply --------------|
      |              Spoke2 tunnel + NBMA information          |
      |                           |                            |
      |<=============== Direct GRE traffic ===================>|
      |                Hub1 no longer forwards                 |
      |                spoke-to-spoke packets                  |
```

### Step 1: Spoke1 Uses Its Ordinary Route

Spoke1 receives a packet for `10.12.12.12`. Its forwarding lookup initially matches the route pointing to Hub1, such as `10.0.0.0/8` via `172.16.0.1`.

At this point, **Spoke1 does not need to know Spoke2's NBMA address**. It already knows how to reach Hub1 through the static NHS mapping.

This **initial packet can follow the ordinary CEF forwarding path to Hub1**. Unlike Phase 2, Spoke1 is not waiting for an unresolved remote-spoke routing next hop before it can forward the packet. NHRP optimization occurs afterward.

Spoke1 GRE-encapsulates the original packet:

```text
Outer IPv4 source:      198.51.100.9    (Spoke1)
Outer IPv4 destination: 198.51.100.1    (Hub1)
GRE payload source:     10.11.11.11
GRE payload dest:       10.12.12.12
```

The ISP forwards the GRE packet to Hub1 using the **outer IPv4 destination**. The ISP never needs a route to `10.12.12.12`.

### Step 2: Hub1 Forwards the Packet to Spoke2

Hub1 decapsulates the GRE packet, performs a route lookup for the **original destination** (`10.12.12.12`), and finds its specific route toward Spoke2:

```text
D 10.12.12.0/24 via 172.16.0.12, Tunnel0
```

Hub1 already has Spoke2's tunnel-to-NBMA mapping from Spoke2's NHRP registration:

```text
172.16.0.12 -> 198.51.100.13
```

It forwards the packet back out `Tunnel0` in a **new GRE packet**:

```text
Outer IPv4 source:      198.51.100.1    (Hub1)
Outer IPv4 destination: 198.51.100.13   (Spoke2)
GRE payload source:     10.11.11.11
GRE payload dest:       10.12.12.12
```

Hub1 has performed **hairpin forwarding**: the original packet arrived through the DMVPN tunnel and is forwarded back out the same DMVPN tunnel toward a different spoke.

Because the hub has routed the original IP packet, its **inner IP TTL is decremented**. The outer IPv4 header is newly generated for the second GRE leg and has its own TTL.

### Step 3: Hub1 Sends an NHRP Redirect

With `ip nhrp redirect` enabled, the hub detects the opportunity to avoid the extra tunnel hop and sends an **NHRP Redirect (Traffic Indication)** to Spoke1.

The redirect tells Spoke1 that a more direct path might exist for traffic matching the original packet's destination. It is **not an ICMP Redirect**, and it does **not** itself contain all the information required to send GRE traffic directly to Spoke2.

The redirect serves as a **trigger for NHRP resolution**. It does not cause the routing protocol to form a new neighborship or advertise a different route.

A hub does not issue a redirect for every forwarded packet indefinitely. Redirect generation and retransmission are subject to implementation-specific control and rate limiting. The important trigger in this topology is **spoke-to-spoke hairpin forwarding through the hub**.

### Step 4: Spoke1 Sends an NHRP Resolution Request

After receiving the redirect, Spoke1 uses NHRP to resolve the **destination of the original data packet**, `10.12.12.12`.

This is a key difference from Phase 2:

- **Phase 2:** NHRP resolves the **remote tunnel next hop**, such as `172.16.0.12`, already supplied by the routing protocol.
- **Phase 3:** NHRP can resolve the **remote LAN destination**, such as `10.12.12.12`, because the routing protocol's next hop may still be the hub.

The request initially travels through Hub1, the NHS. Hub1 forwards the request toward the destination spoke, using its routing information and NHRP registrations. In this example, it reaches **Spoke2**.

**Hub1 does not simply replace the reply from Spoke2 with its own answer in this normal spoke-to-spoke resolution sequence.**

### Step 5: Spoke2 Returns a Resolution Reply

Spoke2 identifies the requested destination as belonging to its local site and sends an **NHRP Resolution Reply** toward Spoke1, normally directly across the underlay.

The reply provides information conceptually equivalent to:

```text
Resolved destination prefix: 10.12.12.0/24
Remote tunnel next hop:      172.16.0.12
Remote NBMA address:         198.51.100.13
```

The destination spoke can reply with **prefix information**, not only a mapping for the single host. The ordinary result for this lab is a mapping covering `10.12.12.0/24`, which can be reused for other destinations in that subnet.

The prefix in the reply is based on the destination spoke's reachability information. An optional NHRP summary-map feature can influence the reported coverage, but it is separate from EIGRP route summarization on the hub.

**NHRP resolution is not BGP, EIGRP, or OSPF route advertisement.** It is a forwarding-discovery exchange initiated by traffic.

### Step 6: Spoke1 Installs Shortcut Forwarding Information

Spoke1 now has sufficient information to reach Spoke2 directly. NHRP creates a dynamic mapping and programs the appropriate forwarding state.

At a conceptual level:

```text
Destination:       10.12.12.0/24
Tunnel next hop:   172.16.0.12
NBMA destination:  198.51.100.13
Outgoing tunnel:   Tunnel0
```

Depending on the relationship between the NHRP prefix and existing routing entries, this can appear as either an **NHRP route** or an **NHRP next-hop override**. Both are explained below.

If IPsec protects the mGRE tunnel, the direct path also requires suitable IKE/IPsec security associations for the two spokes. NHRP discovery and IPsec negotiation are distinct operations.

### Step 7: Subsequent Packets Bypass Hub1

Once the shortcut is usable, Spoke1 encapsulates packets for the resolved destination **directly to Spoke2's NBMA address**:

```text
Outer IPv4 source:      198.51.100.9    (Spoke1)
Outer IPv4 destination: 198.51.100.13   (Spoke2)
GRE payload source:     10.11.11.11
GRE payload dest:       10.12.12.12
```

The route originally learned from Hub1 can remain present. The **forwarding decision has been optimized locally** without establishing a new routing-protocol adjacency.

A direct spoke-to-spoke path in mGRE is **not a new `Tunnel` interface**. It is a dynamically selected peer/encapsulation destination on the existing mGRE interface, together with NHRP and, when applicable, IPsec state.

---

## NHRP Shortcut Routes and Next-Hop Overrides

A common oversimplification is that Phase 3 *only changes the CEF adjacency* while the RIB remains unchanged. Cisco's enhanced shortcut switching supports **NHRP routes in the RIB** and **next-hop overrides** as well as the underlying NHRP mappings.

### Case 1: A More-Specific NHRP Route

Suppose Spoke1 initially has this EIGRP summary:

```text
D        10.0.0.0/8 [90/27008000] via 172.16.0.1, 00:00:15, Tunnel0
```

After resolving `10.12.12.12`, NHRP may install a more-specific route resembling:

```text
H        10.12.12.0/24 [250/255] via 172.16.0.12, 00:00:15, Tunnel0
```

`H` indicates an **NHRP-learned route**. The newly learned `/24` is preferred over the EIGRP `/8` because of **longest-prefix matching**, even though the NHRP route can have a higher administrative distance.

This is not NHRP redistributing routes into EIGRP. NHRP supplies a more-specific **local forwarding route** on Spoke1. Other spokes do not automatically receive it as a new routing-protocol advertisement.

### Case 2: An NHRP Next-Hop Override

Suppose Spoke1 instead already has the exact remote LAN prefix from EIGRP:

```text
D        10.1.1.0/24 [90/27008000] via 172.16.0.1, 00:00:02, Tunnel0
```

An NHRP shortcut covering the **same prefix** can be represented as a **next-hop override (NHO)** rather than replacing the EIGRP route with a separate `H` route.

It appears in the RIB like this:

```
D   %    10.12.12.0/24 [90/28288000] via 172.16.0.1, 00:00:21, Tunnel0
```

The `%` code indicates NHO, although the displayed next hop is still the hub's (`172.16.0.1`).

To view the actual NHRP-learned next hop, use `show ip route next-hop-override`:

```
Spoke1# show ip route next-hop-override
...
D   %    10.12.12.0/24 [90/28288000] via 172.16.0.1, 00:01:50, Tunnel0
                       [NHO][90/255] via 172.16.0.12, 00:01:30, Tunnel0
...
```

The EIGRP route remains owned by EIGRP, but the shortcut can change its **effective forwarding next hop** to `172.16.0.12`.

Check both the RIB and the actual forwarding information:

| Condition | Possible Phase 3 outcome |
| --- | --- |
| Covering route is `10.0.0.0/8` via Hub1; shortcut covers `10.12.12.0/24` | More-specific NHRP route (`H`) |
| Existing route is `10.12.12.0/24` via Hub1; shortcut also covers `/24` | NHRP next-hop override (`%` / NHO) |
| No valid shortcut mapping | Ordinary route through Hub1 |

**The NHRP prefix determines whether an NHRP route or next-hop override is appropriate.** Do not assume every Phase 3 shortcut must appear as a separate `H` route.

### RIB, FIB, and NHRP Mapping Are Different

| Component | Responsibility | Useful command |
| --- | --- | --- |
| Routing Information Base (RIB) | Stores EIGRP/OSPF/BGP routes, NHRP routes, and supported next-hop overrides | `show ip route`, `show ip route nhrp`, `show ip route next-hop-override` |
| Forwarding Information Base (FIB) / CEF | Selects the programmed forwarding path | `show ip cef`, `show ip cef exact-route` |
| NHRP cache | Tracks dynamically learned shortcut prefixes and NBMA endpoints | `show ip nhrp`, `show ip nhrp shortcut` |
| DMVPN peer state | Shows static hub and dynamically discovered spoke peers | `show dmvpn`, `show dmvpn detail` |

**A routing-table next hop of Hub1 is not proof that the traffic still traverses Hub1.** Likewise, an NHRP mapping by itself does not prove that traffic is using the mapping. Examine the route, override or shortcut, CEF, and actual encapsulation path together.

---

## Why Route Summarization Works in Phase 3

In Phase 2, direct spoke-to-spoke forwarding ordinarily requires each spoke to learn a remote network with **that remote spoke's tunnel IP as the route next hop**. A hub-advertised summary points to the hub, so the Phase 2 spoke lacks the necessary remote next-hop information for direct forwarding.

Phase 3 removes that dependency. A hub summary is sufficient to deliver the **initial packet** to a hub that can reach the destination. The redirect/resolution process then determines the **actual destination spoke** and installs a shortcut.

Example:

```text
Before shortcut:
  10.0.0.0/8    -> Hub1 (172.16.0.1)

After shortcut for Spoke2 LAN:
  10.0.0.0/8    -> Hub1 (covering EIGRP route)
  10.12.12.0/24 -> Spoke2 (NHRP shortcut)
```

This allows a large DMVPN to reduce the number of specific routes that hubs must advertise to their spokes.

### The Covering Route Still Matters

A Phase 3 shortcut is **not an independent replacement for the routing topology**. Cisco's **NHRP Route Watch** mechanism associates shortcuts with appropriate covering routing information.

For example, when an NHRP shortcut for `10.12.12.0/24` depends on a covering EIGRP `10.0.0.0/8` route, losing that covering route can invalidate the shortcut. A routing change that introduces a different covering path can also trigger reevaluation or removal of the shortcut.

This prevents shortcut information from indefinitely overriding routing changes after the network topology or intended path has changed.

### EIGRP Versus OSPF Summarization

- **EIGRP:** The hub can summarize spoke-learned routes toward other spokes. The summary points at the hub; NHRP can still establish direct shortcuts.
- **OSPF:** The **point-to-multipoint** tunnel network type is suitable for Phase 3 and normally produces hub-directed forwarding. However, an OSPF router **cannot summarize arbitrary intra-area routes within a single area**. OSPF summarization follows area-boundary or redistribution rules. NHRP shortcuts still work without OSPF summarization.
- **BGP:** Hub-originated or reflected routes must have reachable next hops. A design that intentionally uses the hub as the next hop is compatible with Phase 3 shortcuts; route reflection itself does not automatically rewrite reflected next hops.

The full routing configurations and design choices are covered in [Routing over DMVPN Phase 3](routing.md).

---

## Shortcut Scope, Direction, and Lifetime

### A Shortcut Can Cover a Network, Not Just One Host

The source spoke initially resolves an individual packet destination such as `10.12.12.12`, but the resolution reply can identify the containing **`10.12.12.0/24` prefix**.

After installing that shortcut, Spoke1 can potentially use the same direct path for additional hosts behind Spoke2 in that prefix without repeating the entire resolution exchange.

If Spoke2 has several distinct LAN prefixes, the source may need separate shortcuts for those prefixes. The **`ip nhrp summary-map`** feature can influence the prefix advertised in replies; it is different from the hub's EIGRP summary.

> More on that command in a later page.

### Forward and Return Paths Are Independent

A shortcut learned by Spoke1 for traffic to Spoke2 does **not** guarantee Spoke2 already has a matching shortcut for traffic toward Spoke1's LAN.

For example:

```text
Spoke1 -> Spoke2: direct shortcut exists
Spoke2 -> Spoke1: route may still point through Hub1
```

The reverse traffic can trigger its own redirect and resolution process. Temporary **asymmetric paths** can therefore be normal during shortcut establishment. If the receiving spoke has no valid route back to the source prefix, return traffic can fail even though the forward shortcut exists.

### Shortcuts Are Dynamic

NHRP mappings advertise holding times and can expire, be refreshed, or be purged when forwarding information changes. A shortcut can also be removed when its covering route disappears or changes in a way that invalidates it.

After a shortcut is removed, the router can fall back to ordinary routing through the hub if that route remains usable. Later traffic can trigger a new redirect and resolution exchange.

Do not equate **NHRP cache lifetime**, **routing-protocol convergence**, and **IPsec security-association lifetime**. These are separate mechanisms.

---

## Packet Forwarding and GRE/IPsec

### A Direct GRE Path Is Still Carried Over the Underlay

After the shortcut is installed, the underlay must deliver a packet whose **outer destination is the remote spoke's NBMA address**. The ISP routes that outer packet normally; it does not participate in the overlay routing protocol.

| Stage | Outer source | Outer destination | Forwarding path |
| --- | --- | --- | --- |
| Initial Spoke1-to-Hub1 leg | `198.51.100.9` | `198.51.100.1` | Spoke1 to Hub1 |
| Initial Hub1-to-Spoke2 leg | `198.51.100.1` | `198.51.100.13` | Hub1 to Spoke2 |
| After shortcut | `198.51.100.9` | `198.51.100.13` | Directly from Spoke1 to Spoke2 |

These are the **GRE outer IPv4 header values without IPsec**. With IPsec tunnel protection, the on-wire encapsulation also includes IPsec-related headers, and NAT traversal may add UDP encapsulation.

### TTL and Traceroute Behavior

With normal routed site-to-site traffic, passing through Hub1 requires an additional **inner-IP routing hop** compared with direct spoke-to-spoke forwarding.

- **Before the shortcut:** Hub1 decapsulates, routes, and re-encapsulates the original IP packet. It decrements the original IP TTL.
- **After the shortcut:** Hub1 no longer routes that packet. The source spoke sends directly to the destination spoke's tunnel endpoint.

A traceroute can therefore show a change after the shortcut is established. However, the hop visibility and displayed addresses depend on ICMP handling, tunnel processing, and any security or filtering policies. A single traceroute should not be treated as the only proof of the forwarding path.

### Direct IPsec Requires Direct Peer Reachability

If IPsec is applied to the DMVPN tunnel, spoke-to-spoke traffic requires **compatible IKEv2/IPsec settings** and a usable transport path between the spokes. Successful NHRP resolution is not sufficient if IPsec negotiation or ESP/NAT-T delivery fails.

Also consider **GRE/IPsec overhead, tunnel MTU, fragmentation, and TCP MSS**. A path can succeed with small ICMP packets but fail for larger data flows even when the NHRP shortcut is correct.

---

## Dual-Hub Forwarding Considerations

With two hubs, each spoke may have multiple potential ways to reach a remote prefix. **The routing protocol selects the initial forwarding hub**; NHRP registration and shortcut mechanisms then determine whether traffic can be optimized.

### Single-Cloud Dual-Hub Design

In a single-cloud design, the spokes typically have **one mGRE tunnel interface** and register with **both Hub1 and Hub2** as NHSs.

```text
                 Hub1              Hub2
                   \                /
                    \              /
                    Spoke1 Tunnel0
                      |        \
                      |         \   Direct spoke paths
                    Spoke2     Spoke3

            One shared DMVPN tunnel subnet
```

Both hubs can advertise routes into the shared cloud. The selected routing next hop determines which hub first receives a packet when no shortcut exists. The hub forwarding the packet toward another spoke can generate the redirect if its NHRP and forwarding conditions are satisfied.

**Registration with two NHSs does not guarantee identical routing choices or synchronized shortcut state.** Each spoke must have working registration and overlay routing toward an appropriate hub.

### Dual-Cloud Dual-Hub Design

In a dual-cloud design, each spoke typically has **two mGRE tunnel interfaces**, each in a separate DMVPN overlay:

```text
Spoke Tunnel0 -> Cloud 1 -> Hub1
Spoke Tunnel1 -> Cloud 2 -> Hub2
```

Routing preference determines the initial cloud used for traffic. When troubleshooting a shortcut, verify the **correct tunnel interface, NHRP context, route next hop, and NBMA endpoint** rather than assuming the two clouds share forwarding state.

The detailed routing preferences, NHS configuration, failover, and convergence behavior are covered in [Dual-Hub DMVPN Topologies](dual-hub/topologies.md), [Dual-Hub Phase 3 Configuration](dual-hub/configuration.md), and [Dual-Hub Failover and Convergence](dual-hub/failover.md).

---

## Verification: Observe the Transition from Hub to Shortcut

The most useful test is to **observe a clean initial forwarding state, generate traffic, and then inspect the new shortcut**.

### 1. Verify Hub and Spoke Roles

On Hub1:

```cisco
show running-config interface Tunnel0
show ip nhrp
show dmvpn
```

Confirm mGRE, the hub's `ip nhrp redirect`, and successful spoke registrations.

On Spoke1:

```cisco
show running-config interface Tunnel0
show ip nhrp nhs detail
show dmvpn
```

Confirm mGRE, `ip nhrp shortcut`, and a registered NHS. Spoke1 should initially know Hub1's NBMA address even if it has no dynamic shortcut for Spoke2.

### 2. Verify the Initial Overlay Route

On Spoke1:

```cisco
show ip route 10.12.12.12
show ip cef 10.12.12.12 detail
show ip route next-hop-override
```

Before a shortcut, the effective route should normally use Hub1 as next hop. If the routing protocol already advertises **Spoke2** as the next hop, you are not demonstrating the typical Phase 3 redirect-triggered forwarding process.

### 3. Generate Spoke-to-Spoke Traffic

From Spoke1:

```cisco
ping 10.12.12.12 source 10.11.11.11 repeat 20
traceroute 10.12.12.12 source 10.11.11.11
```

The first packets can take the hub path. After NHRP resolution, the path should be eligible for direct forwarding. Some initial packet loss or changing traceroute behavior can occur during resolution or IPsec establishment.

### 4. Examine the NHRP Shortcut and Routing State

On Spoke1:

```cisco
show ip nhrp
show ip nhrp shortcut
show ip route nhrp
show ip route next-hop-override
show ip cef 10.12.12.12 detail
show ip cef exact-route 10.11.11.11 10.12.12.12
show dmvpn detail
```

Look for a dynamic entry for Spoke2's LAN or tunnel address, with the expected NBMA `198.51.100.13`. Depending on the route relationships, expect a **more-specific NHRP route** or a **next-hop override**.

### 5. Examine NHRP Control-Plane Counters

On Hub1 and the spokes:

```cisco
show ip nhrp traffic
```

Correlate **redirects**, **resolution requests**, and **resolution replies** with the traffic test. Registration and NHRP resolution are different exchanges, so successful registration counters alone do not prove shortcut operation.

### 6. Verify the Actual Data Path

If possible, capture traffic on the ISP-facing links. With unprotected GRE, compare the outer destination before and after shortcut establishment:

```text
Before: Spoke1 -> 198.51.100.1  (Hub1)
After:  Spoke1 -> 198.51.100.13 (Spoke2)
```

This is more conclusive than relying solely on an IGP next-hop display. With IPsec protection, inspect the corresponding IKEv2/IPsec peer and encapsulation state as well.

---

## Forwarding Problems and Their Causes

| Observation | Likely issue | What to check |
| --- | --- | --- |
| Spoke cannot reach hub or register | Underlay, NHS mapping, NHRP authentication, tunnel configuration | `show ip nhrp nhs detail`, `show dmvpn`, WAN route |
| Spoke registers but has no route to remote LAN | Overlay routing failure | IGP/BGP neighbors, advertised prefixes, route filters |
| Initial packet never reaches destination | Hub lacks remote-spoke route or valid mapping | Hub routing table, registration for target spoke |
| Traffic hairpins through hub indefinitely | Redirect missing, not generated, or not processed; shortcut missing | Hub `ip nhrp redirect`, spoke `ip nhrp shortcut`, NHRP counters |
| Redirect counters increase, but resolution fails | Request forwarding or return Reply failure | Source-to-NHS, NHS-to-destination, destination-to-source paths |
| Shortcut entry exists, but data still uses hub | NHRP route/next-hop override not installed or usable | `show ip nhrp shortcut`, RIB, NHO, CEF |
| More-specific `H` route is absent | Matching prefix uses next-hop override instead, or resolution failed | `show ip route next-hop-override`, `show ip nhrp` |
| Shortcut disappears after an IGP change | Covering route withdrawn or changed; Route Watch invalidated shortcut | Covering route, NHRP entries, route updates |
| Spoke-to-spoke control plane works but data fails | GRE/IPsec transport, ACL, NAT, MTU, return route | NBMA reachability, IPsec SAs, packet capture |
| Only one traffic direction uses the shortcut | Independent forward/reverse shortcut state | Test traffic and NHRP state on both spokes |
| Dual-hub traffic uses unexpected hub/cloud | Routing preference or tunnel/NHS selection | RIB/FIB on source spoke, both hub routes, NHRP interfaces |

**Troubleshooting should proceed in order: underlay, tunnel, NHS registration, overlay routing, hub forwarding, redirect, resolution, shortcut installation, and direct data transport.** A failure at an earlier step can make later symptoms misleading.

For detailed NHRP counters, cache flags, debugs, and message interpretation, see [NHRP Verification and Troubleshooting](nhrp/troubleshooting.md). For complete cross-layer fault isolation, see [End-to-End DMVPN Phase 3 Troubleshooting](troubleshooting.md).

---

## Lab Exercises

Use the existing Hub1/Spoke1/Spoke2/Spoke3 topology to examine Phase 3 behavior.

1. **Baseline:** Confirm the spokes register with Hub1 and learn LAN routes using Hub1 as next hop. Do not generate spoke-to-spoke traffic yet.
2. **First packet:** Send traffic from Spoke1 to Spoke2. Inspect the hub's forwarding route and the NHRP redirect counters.
3. **Resolution:** Inspect NHRP request/reply counters and the dynamic entry created on Spoke1 for Spoke2's LAN.
4. **Direct path:** Compare `show ip cef exact-route`, NHRP, and outer NBMA destinations before and after resolution.
5. **Summarization:** Configure an EIGRP summary on Hub1. Verify Spoke1 can install a more-specific NHRP shortcut while retaining the covering summary route.
6. **Next-hop override:** Repeat the test without summarization, with an exact EIGRP route through Hub1. Check whether the shortcut is represented as a next-hop override rather than an `H` route.
7. **Redirect disabled:** Remove `ip nhrp redirect` on the hub, clear the relevant dynamic shortcut state, and confirm that traffic follows ordinary hub routing when no shortcut exists.
8. **Shortcut disabled:** Disable `ip nhrp shortcut` on one spoke, clear its shortcut state, and observe whether that spoke can install a new shortcut.
9. **Return path:** Initiate traffic in the opposite direction and observe whether the reverse shortcut is installed independently.
10. **Route withdrawal:** Remove the covering overlay route and examine how NHRP shortcut entries and forwarding behavior change.

Restore the desired Phase 3 settings after each test before proceeding to the dual-hub labs.

---

## Key Points

- **Phase 3 uses mGRE on both hubs and spokes.** Spokes exchange routing information with the hub but can forward directly to other spokes.
- A spoke's **ordinary routing next hop can be the hub**, even for routes behind remote spokes.
- The **first data packet normally follows the route through the hub** when no shortcut exists.
- The hub can send an **NHRP Redirect** when forwarding a spoke's packet back through the DMVPN cloud toward another spoke.
- The redirect is a **trigger**, not the final mapping. The source spoke sends an **NHRP Resolution Request** for the original packet destination.
- The destination spoke normally sends the **NHRP Resolution Reply directly** to the requesting spoke.
- The reply can describe a **destination prefix**, allowing the resulting shortcut to be reused for multiple hosts.
- Phase 3 can install a **more-specific NHRP route (`H`)** or a **next-hop override** when the shortcut matches an existing route prefix.
- A shortcut can change the effective next hop **without changing the routing protocol's advertised NEXT_HOP or creating a spoke-to-spoke routing neighborship**.
- **Longest-prefix matching**, not administrative distance alone, explains why a specific NHRP route can win over a less-specific EIGRP summary.
- **Route Watch** ties the validity of NHRP shortcuts to appropriate covering routes.
- **Hub summarization is compatible with Phase 3 shortcuts**, unlike normal Phase 2 direct-forwarding requirements.
- In Phase 3 with EIGRP, retain **next-hop-self on the hub**. Disable split horizon where needed for hub-to-spoke readvertisement.
- **OSPF point-to-multipoint** is suitable for Phase 3; OSPF intra-area summarization rules still apply.
- Spoke-to-spoke forwarding is **directional**: each spoke may independently need a shortcut for the return path.
- The direct path must have **underlay reachability** and, if tunnel protection is enabled, working **IKEv2/IPsec**.
- In dual-hub deployments, **routing selects the initial hub/cloud**; NHRP shortcuts optimize forwarding after the initial path is established.
- Verify **NHRP mappings, NHRP routes/next-hop overrides, CEF, and actual outer encapsulation**, not only the IGP route.

---

## References

- [Cisco: Shortcut Switching Enhancements for NHRP in DMVPN Networks](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-addressing/ip-addressing/m_nhrp-switch-enhancemts-dmvpn-0.html)
- [Cisco: Troubleshoot DMVPN Issues](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/111976-dmvpn-troubleshoot-00.html)
- [Cisco: Spoke-to-Spoke NHRP Summary Maps](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m-sec-conn-dmvpn-summ-maps.html)
- [Cisco: Configure Phase-3 Hierarchical DMVPN with Multi-Subnet Spokes](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/211292-Configure-Phase-3-Hierarchical-DMVPN-wit.html)
- [Cisco: Dynamic Multipoint IPsec VPNs](https://www.cisco.com/c/en/us/support/docs/security-vpn/ipsec-negotiation-ike-protocols/41940-dmvpn.html)
