# NHRP Redirects and Shortcuts

**DMVPN Phase 3** uses **NHRP Redirects** and **NHRP Shortcuts** to establish direct spoke-to-spoke forwarding even when the overlay routing protocol identifies the **hub as the next hop**. This removes Phase 2's requirement that every remote spoke LAN route use the originating spoke's tunnel address as its routing next hop.

The spoke initially forwards traffic along the normal routed path. If that path unnecessarily passes through the hub, an NHRP Redirect prompts the source spoke to resolve a better path. An NHRP Resolution Reply from the destination spoke then supplies the information needed to install a shortcut for a destination **prefix**, not necessarily just one host.

This page focuses on **redirect generation, resolution, shortcut prefix selection, NHRP RIB/FIB integration, summary maps, and shortcut lifecycle**. Basic NHRP registration and multicast mapping are covered in [NHRP Fundamentals](../../nhrp-fundamentals.md). Full dual-hub configuration and IKEv2/IPsec protection are covered in their own sections.

---

## Phase 3 Versus Phase 2

| Behavior | Phase 2 | Phase 3 |
| --- | --- | --- |
| Tunnel mode | mGRE on hubs and spokes | mGRE on hubs and spokes |
| How the first packet is forwarded | Source attempts to resolve its remote-spoke routing next hop | Source can forward toward the hub using its existing route |
| Routing next hop needed for direct forwarding | Destination spoke's tunnel address | **Hub's tunnel address is sufficient** |
| NHRP shortcut discovery trigger | Need to resolve a remote-spoke next hop | **Redirect from the hub** |
| Hub route summarization | Usually defeats direct paths for summarized routes | Supported alongside direct shortcuts |
| Key additional commands | NHRP mapping/NHS configuration | **`ip nhrp redirect`** on hub; **`ip nhrp shortcut`** on spokes |
| Optimized forwarding | Direct mGRE toward destination spoke | Direct mGRE toward destination spoke |

**Phase 3 changes the relationship between the routing and forwarding planes.** The routing protocol can continue to prefer the hub while NHRP locally refines the forwarding decision for particular destinations.

---

## Lab Topology and Addressing

The same Phase 1/2 underlay is used. Hub2 is not active in the initial single-hub packet-flow examples.

```text
                     DMVPN Phase 3 overlay

                           Hub1 (NHS)
                       Tunnel0 172.16.0.1
                              mGRE
                            /  |  \
                           /   |   \
                          /    |    \
                      Spoke1 Spoke2 Spoke3
                       .11    .12    .13
                      mGRE   mGRE   mGRE

                  Direct Spoke1 <--> Spoke2
                    after NHRP shortcut

                    Tunnel: 172.16.0.0/24
```

| Router | Tunnel0 | NBMA address | Simulated LAN |
| --- | --- | --- | --- |
| Hub1 | `172.16.0.1/24` | `198.51.100.1` | `10.1.1.0/24` |
| Spoke1 | `172.16.0.11/24` | `198.51.100.9` | `10.11.11.0/24` |
| Spoke2 | `172.16.0.12/24` | `198.51.100.13` | `10.12.12.0/24` |
| Spoke3 | `172.16.0.13/24` | `198.51.100.17` | `10.13.13.0/24` |

All DMVPN devices use mGRE and a fixed configuration for reaching Hub1's NHRP Next Hop Server (NHS). The ISP routes between the NBMA addresses and has no routes to the simulated LANs.

For the primary example, Hub1 advertises an **EIGRP summary `10.0.0.0/8`** into the cloud. Spoke1 therefore initially sends traffic for `10.12.12.12` to Hub1, even though Spoke2 is the final destination router.

---

## The Phase 3 Commands

### On the Hub: `ip nhrp redirect`

```cisco
interface Tunnel0
 ip nhrp redirect
```

This enables the hub to generate **NHRP Redirect (Traffic Indication)** messages when it detects a packet taking a suboptimal path through the DMVPN cloud.

The classic triggering condition is **hairpin forwarding**: a packet arrives on the hub's mGRE interface and is routed back out **that same mGRE interface** toward a different spoke.

```text
Spoke1 -> Hub1 Tunnel0 -> Hub1 Tunnel0 -> Spoke2
                  |
                  +-- NHRP Redirect to Spoke1
```

The hub **still forwards the triggering data packet** toward Spoke2. The Redirect is a separate control-plane message that informs Spoke1 that a more direct path may be available. It is **not** an ordinary ICMP Redirect, and it does not by itself provide the complete shortcut mapping.

**A Redirect is not generated merely because a packet passes through a hub.** It depends on the applicable DMVPN forwarding conditions. For example, traffic destined to the hub's own LAN is not a spoke-to-spoke hairpin.

### On the Spokes: `ip nhrp shortcut`

```cisco
interface Tunnel0
 ip nhrp shortcut
```

This enables the spoke to use NHRP resolution information to install and maintain optimized forwarding paths. The relevant functions include:

- Respond to a suitable Redirect by initiating NHRP resolution for the indicated destination.
- Cache the resulting destination-prefix/remote-spoke information.
- Install an **NHRP route** or a **next-hop override** as appropriate.
- Update CEF forwarding so later packets bypass the hub.

In modern IOS XE, **NHRP smart defaults** enables shortcut-related behavior automatically on multipoint interfaces. The explicit command states the design intent and is used throughout these examples.

### Required Underlying NHRP Configuration

The Redirect/Shortcut commands **supplement**, rather than replace, ordinary DMVPN configuration.

Hub1 still needs NHRP registration information for its spokes; each spoke still needs an NHS and a way to reach its NBMA address. When EIGRP or OSPF is used, spokes must also send routing-protocol multicast to the hub, and the hub must replicate multicast to registered spokes.

For a Phase 3 EIGRP design, the hub normally retains **EIGRP next-hop-self** and disables **EIGRP split horizon** when it must advertise spoke-learned routes back out the tunnel. Full EIGRP routing behavior is discussed in [Routing over DMVPN Phase 3](../routing.md).

---

## NHRP Redirect and Resolution: Complete Packet Flow

Assume that Spoke1 sends a packet from `10.11.11.11` to `10.12.12.12`, behind Spoke2.

### 1. Spoke1 Sends the First Packet to Hub1

Before any shortcut is discovered, Spoke1 has an EIGRP summary:

```text
D        10.0.0.0/8 [90/27008000] via 172.16.0.1, 00:01:29, Tunnel0
```

The packet's destination is `10.12.12.12`. A longest-prefix lookup matches `10.0.0.0/8` and selects Hub1's tunnel address, `172.16.0.1`.

Spoke1 already has a static NHRP mapping for the hub:

```text
172.16.0.1 -> 198.51.100.1
```

It encapsulates the original packet in GRE using outer destination **`198.51.100.1`**. This first packet can follow the existing CEF path through the hub; the source does **not** need to discover Spoke2 before sending it.

### 2. Hub1 Forwards the Packet and Generates a Redirect

Hub1 decapsulates the GRE packet, looks up `10.12.12.12`, and finds a route through `Tunnel0` toward Spoke2 (`172.16.0.12`). It already knows Spoke2's NBMA address from registration:

```text
172.16.0.12 -> 198.51.100.13
```

Hub1 forwards the original packet to Spoke2 using a new GRE outer header destined for **`198.51.100.13`**.

Because the packet **arrived and departed on Tunnel0**, Hub1 can also generate an NHRP Redirect toward Spoke1. The Redirect identifies traffic for which the hub has observed a suboptimal path and prompts resolution. It should not be confused with a resolution reply: **the Redirect itself is not the final tunnel-to-NBMA mapping for the destination**.

### 3. Spoke1 Initiates an NHRP Resolution Request

After receiving the Redirect, Spoke1 requests information for the **original destination IP**, `10.12.12.12`.

This is an important distinction from basic Phase 2 resolution:

- **Phase 2:** A spoke resolves the **routing next hop**, which must identify the remote spoke.
- **Phase 3:** A spoke can initially route via the **hub**, then use the Redirect to resolve the **destination behind the remote spoke**.

Spoke1 sends the NHRP Resolution Request through Hub1, its NHS. Hub1 forwards the request along the overlay route toward Spoke2.

### 4. Spoke2 Sends the Resolution Reply

Spoke2 examines the destination address in the request, determines that it can reach the destination, and returns the applicable **destination prefix** and **its own NBMA/overlay endpoint information**.

In this example, Spoke2 has a connected route for `10.12.12.0/24`, so its reply can describe that **entire /24**, rather than only `10.12.12.12/32`.

The Resolution Reply is normally sent **directly from Spoke2 to Spoke1**, not back through the hub. The request contains the information needed to return the reply to the requesting spoke.

### 5. Spoke1 Installs a Shortcut

After processing the successful Resolution Reply, Spoke1 learns a path equivalent to:

```text
Destination prefix: 10.12.12.0/24
Overlay next hop:   172.16.0.12 (Spoke2)
NBMA endpoint:      198.51.100.13 (Spoke2)
Tunnel:             Tunnel0
```

NHRP updates its cache and installs the appropriate RIB/FIB state. Subsequent packets for addresses covered by the shortcut use **direct mGRE encapsulation to `198.51.100.13`**, bypassing Hub1.

```text
        Spoke1                    Hub1                    Spoke2
           |                        |                         |
           |==== initial packet ===>|=== forwarded packet ===>|
           |<--- NHRP Redirect -----|                         |
           |                        |                         |
           |--- Resolution Req ---->|--- Resolution Req ----->|
           |                        |                         |
           |<----------------- Resolution Reply --------------|
           |                        |                         |
           |================ direct GRE packets =============>|
```

---

## How Wide Is the Shortcut?

**The prefix returned in the NHRP Resolution Reply determines the shortcut's potential scope.** It is not automatically restricted to the IP address that triggered the Redirect.

Suppose Spoke1 sends a packet to `10.12.12.12`:

| Spoke2's resolution result | Potential shortcut prefix at Spoke1 | Addresses covered |
| --- | --- | --- |
| Connected `10.12.12.0/24` | `10.12.12.0/24` | Entire /24 |
| A `/32` host route | `10.12.12.12/32` | That host only |
| Appropriate NHRP summary map `10.12.12.0/23` | `10.12.12.0/23` | `10.12.12.0` through `10.12.13.255` |

The destination router normally consults its RIB to determine an appropriate prefix for the Resolution Reply. A configured **NHRP summary map** can influence that response, covered below.

**The source must still have a valid covering route**, and the resulting shortcut must be eligible for installation. A Resolution Reply containing a prefix does not mean NHRP can override a disconnected or invalid base routing topology.

---

## NHRP Shortcut Routes and Next-Hop Overrides

Phase 3 NHRP does more than cache a tunnel-to-NBMA mapping. It can influence the **RIB** and therefore the **FIB**. Two particularly important outcomes are:

1. A separate **`H` (NHRP) route**.
2. An **NHRP Next-Hop Override (NHO)** attached to an existing route.

### Case 1: More-Specific Shortcut Creates an `H` Route

Spoke1 initially has:

```text
D        10.0.0.0/8 [90/27008000] via 172.16.0.1, 00:00:15, Tunnel0
```

After receiving a Resolution Reply for `10.12.12.0/24`, Spoke1 can install:

```text
D        10.0.0.0/8 [90/27008000] via 172.16.0.1, 00:00:15, Tunnel0
H        10.12.12.0/24 [250/255] via 172.16.0.12, 00:00:15, Tunnel0
```

**Why the `H` route wins:** `10.12.12.0/24` is **more specific** than `10.0.0.0/8`. A longest-prefix lookup selects the /24 first, regardless of the fact that its administrative distance (**250** in this example) is higher than EIGRP's AD (**90**).

NHRP has not replaced the base summary route. It has added a **more-specific forwarding refinement** for the reachable destination behind Spoke2.

Confirm with:

```cisco
show ip route 10.12.12.12
show ip route nhrp
show ip nhrp shortcut
show ip cef 10.12.12.12 detail
```

### Case 2: Identical Prefix Uses a Next-Hop Override

Now assume Spoke1 already has the exact prefix through Hub1:

```text
D        10.1.1.0/24 [90/27008000] via 172.16.0.1, 00:00:02, Tunnel0
```

An NHRP Resolution Reply for **the same prefix**, `10.12.12.0/24`, can produce a **next-hop override** rather than another competing route for the same destination:

```text
D   %    10.12.12.0/24 [90/28288000] via 172.16.0.1, 00:01:50, Tunnel0
                       [NHO][90/255] via 172.16.0.12, 00:01:30, Tunnel0
```

*Conceptual output; exact markers, formatting, and administrative/metric fields depend on the CLI display.*

- **`%`** indicates the original route has an NHRP next-hop override.
- **`[NHO]`** identifies the NHRP-selected forwarding next hop.
- The **EIGRP route remains** under EIGRP's control. NHRP alters the local forwarding next hop; it does not rewrite EIGRP's advertised topology.

Use `show ip route next-hop-override` to inspect the override. An ordinary `show ip route` will show the `%` marker without the NHO details.

### Comparing the Two Results

| Existing best route at Spoke1 | Reply prefix from Spoke2 | Typical Phase 3 result |
| --- | --- | --- |
| `10.0.0.0/8` via Hub1 | `10.12.12.0/24` | New **`H 10.12.12.0/24`** route |
| `10.12.12.0/24` via Hub1 | `10.12.12.0/24` | **`[NHO]`** for existing /24 route |

An identical prefix is the clearest NHO case. More complicated prefix overlaps and competing routes can change the precise result. Determine the **actual RIB/FIB state** rather than assuming every shortcut produces an `H` route.

### Why NHRP Is Not a Full Routing Protocol

NHRP can install routes or overrides, but it **does not replace EIGRP, OSPF, or BGP**:

- It does not establish a spoke-to-spoke routing-protocol adjacency.
- It does not advertise a complete topology to its NHRP peers.
- It depends on the existing routed topology to reach and resolve destinations.
- Its installed shortcuts are local forwarding information, not general routing updates that must be flooded or re-advertised.

---

## NHRP Summary Maps

An NHRP Resolution Reply normally uses prefix information derived from the replying spoke's RIB. If many individual LAN prefixes exist behind one spoke, source spokes may need **multiple resolutions** to optimize traffic to all of them.

The **`ip nhrp summary-map`** command allows the **destination spoke** to return a **larger prefix** in its Resolution Reply when that prefix covers the requested destination.

### Example: Two LAN Prefixes Behind Spoke2

Assume Spoke2 owns both:

```text
10.12.12.0/24
10.12.13.0/24
```

Without an NHRP summary map, separate Resolution Replies may describe `10.12.12.0/24` and `10.12.13.0/24`.

Configure Spoke2's tunnel interface:

```cisco
interface Tunnel0
 ip nhrp summary-map 10.12.12.0/23
```

When Spoke1 sends a Resolution Request for `10.12.12.12`, Spoke2 can return **`10.12.12.0/23`**, covering **both** /24s.

```text
Requested destination:        10.12.12.12
Spoke2 RIB match:             10.12.12.0/24
Configured NHRP summary map:  10.12.12.0/23
Resolution Reply prefix:      10.12.12.0/23
```

If Spoke1 has only Hub1's `10.0.0.0/8` summary covering these destinations, the result can be a single:

```text
H 10.12.12.0/23 [250/1] via 172.16.0.12, Tunnel0
```

The same shortcut then covers destinations in both networks, avoiding a separate NHRP resolution for `10.12.13.0/24`.

### NHRP Summary Map Versus EIGRP Summary

| Feature | Where configured in this example | What it affects |
| --- | --- | --- |
| `ip summary-address eigrp 100 10.0.0.0 255.0.0.0` | **Hub1 Tunnel0** | The **routing prefix advertised** from hub to spokes |
| `ip nhrp summary-map 10.12.12.0/23` | **Spoke2 Tunnel0** | The **prefix in NHRP Resolution Replies** sent by Spoke2 |

These mechanisms are independent and can be combined:

1. EIGRP summarization reduces the spoke routing table and allows initial forwarding through Hub1.
2. A Redirect triggers a request for the destination.
3. Spoke2's NHRP summary map returns a useful /23.
4. Spoke1 installs a direct shortcut for that /23.

**Do not configure an NHRP summary covering destinations that are actually behind different spokes** unless the design intentionally makes the replying spoke the correct forwarding endpoint for the whole summary. An overly broad shortcut can attract traffic to the wrong spoke and cause loss or loops.

**A summary map on the hub is not the same feature as a spoke-to-spoke NHRP summary map.** Cisco also documents more advanced NHRP default-map/distribution functions using additional `summary-map` forms; those should be distinguished from the simple destination-spoke Resolution Reply example here.

---

## NHRP Route Watch and Shortcut Validity

NHRP shortcuts are **refinements of valid underlying routes**, not arbitrary permanent routes. **Route Watch** tracks the routing information that permits an NHRP shortcut to exist.

### Covering Routes

Suppose Spoke1 has:

```text
D        10.0.0.0/8 [90/27008000] via 172.16.0.1, 00:00:15, Tunnel0
H        10.12.12.0/24 [250/255] via 172.16.0.12, 00:00:15, Tunnel0
```

The EIGRP `10.0.0.0/8` summary is the **covering route** for the more-specific NHRP shortcut.

If this covering route disappears and no other suitable route covers the destination, NHRP removes the associated shortcut. Otherwise the stale `H` route could keep traffic on a forwarding path that is no longer valid under the base routing topology.

### Changes to the Covering Route

A new, more-specific routing-protocol route may become the best covering route for a shortcut. If it implies a different forwarding path or next hop, Route Watch can cause NHRP to withdraw the old shortcut and resolve a path consistent with the new routing state.

The important consequences are:

- **An NHRP `H` route does not survive indefinitely just because its dynamic NHRP mapping has time remaining.**
- The existence and attributes of **covering IGP/routes** affect shortcut validity.
- A routing change can remove an existing shortcut even when the remote spoke's NBMA address has not changed.
- NHRP shortcut state and routing convergence are linked, but each is maintained by a distinct control-plane function.

### Route Watch and Next-Hop Reachability

Modern IOS XE NHRP also supports Route Watch tracking for an NHRP cache entry's **next hop**, including scenarios where the resolved next hop is in a different subnet. This helps invalidate stale cache entries when the underlying next-hop route changes or disappears.

**Do not troubleshoot an unexpected shortcut deletion by examining only NHRP holdtimes.** Check changes to the RIB, the covering route, the resolved overlay next hop, and underlay reachability.

---

## Shortcut Lifetimes, Refresh, and Purge

### Cache Holding Times

NHRP dynamic entries can expire. The mapping's advertised holding time controls how long a peer may use it without refresh. This is distinct from IGP neighbor timers, GRE reachability, and IPsec SA lifetimes.

A heavily used shortcut may be maintained or refreshed. An idle shortcut may expire, after which new traffic can again follow the hub path and trigger fresh resolution.

For the detailed NHRP timer commands and registration refresh behavior, see [NHRP Fundamentals](../../nhrp-fundamentals.md).

### NHRP Purge Requests and Replies

When previously advertised reachability changes, an NHRP node can send a **Purge Request** to peers that learned the now-invalid mapping. Recipients discard the affected mapping and acknowledge with a **Purge Reply**.

This is different from merely waiting for the holdtime to expire. Purge messages provide an explicit way to invalidate stale information after topology or mapping changes.

### Loss of a Direct Spoke-to-Spoke Path

If direct GRE/IPsec forwarding breaks but hub forwarding still works, the source may have valid routes and NHRP state while packets to the remote NBMA address are lost. Distinguish:

1. **NHRP cache entry validity.** Is the shortcut mapping still present?
2. **RIB/FIB state.** Is the shortcut selected for the destination?
3. **Underlay reachability.** Can the NBMA destination be reached without recursion through the overlay?
4. **GRE/IPsec delivery.** Are direct packets permitted and successfully decapsulated?
5. **Return routing.** Does the remote spoke have a correct path back to the source?

NHRP does **not** guarantee that the underlay permits direct spoke-to-spoke communication. A firewall or NAT device may allow hub-to-spoke connections while blocking the direct path.

---

## Redirect Generation, Rate Limiting, and Scale

NHRP Redirects are control-plane events derived from forwarding behavior. In a large Phase 3 deployment, the hub may need to generate many redirects, particularly when traffic repeatedly takes an unresolved spoke-to-spoke path.

### Redirect Suppression and Rate Limits

IOS XE implementations can limit redirect generation to prevent excessive control-plane load. Cisco documents **redirect rate limiting**, **NHRP max-send limits**, and **platform punt-path policing** as distinct possible bottlenecks.

Useful commands include:

```cisco
show ip nhrp redirect
show ip nhrp traffic
show platform software punt-policer
```

The first command can help identify which sending NBMA addresses and destination IPs are repeatedly triggering redirects. The second shows NHRP packet counters and max-send utilization. The platform command can identify a **DMVPN NHRP redirect punt policer**, where supported.

Cisco documents the interface-level timer:

```cisco
interface Tunnel0
 ip nhrp redirect timeout 8
```

The **redirect timeout** is a suppression/rate-limiting interval for repeated redirects; it is **not** the time a shortcut remains installed. Cisco documents 8 seconds as a default in its redirect troubleshooting discussion, with configurable values depending on software support.

The interface-level NHRP max-send controls the volume of NHRP control messages and can also affect successful resolution during bursts. Do not increase control-plane limits blindly: first determine whether repeated redirects are caused by legitimate new flows or by **failed shortcut establishment** that keeps traffic hairpinning.

### Why Repeated Redirects Matter

A source that successfully installs a shortcut stops sending those matching packets through the hub, so it stops generating new hairpin-triggered redirects for that traffic. If a source **never** installs the shortcut, the hub may continue to observe the same suboptimal forwarding pattern.

Repeated Redirect indications can therefore be **a symptom of a downstream resolution or direct-path failure**, not merely an overloaded hub.

---

## Minimal Phase 3 Configuration Example

This example highlights the **NHRP-specific Phase 3 changes**. The WAN interfaces, Loopback0 addresses, ISP default routes, tunnel subnet, GRE tunnel key, and EIGRP AS 100 follow the complete Phase 2 lab.

### Hub1

```cisco
interface Tunnel0
 ip address 172.16.0.1 255.255.255.0
 no ip redirects
 ip nhrp authentication NHRP100
 ip nhrp map multicast dynamic
 ip nhrp network-id 100
 ip nhrp redirect
 no ip split-horizon eigrp 100
 ip summary-address eigrp 100 10.0.0.0 255.0.0.0
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255
 network 10.1.1.1 0.0.0.0
```

**Key distinctions:**

- `ip nhrp redirect` is the Phase 3 forwarding trigger on the hub.
- The EIGRP summary provides spokes a valid **hub-next-hop route** that covers the remote LANs.
- EIGRP **next-hop-self remains enabled** on the hub; do **not** carry forward Phase 2's `no ip next-hop-self eigrp 100` command.
- `no ip split-horizon eigrp 100` is retained when the hub must advertise learned spoke routes back out the tunnel. If the design advertises only hub-originated summaries, the need to disable split horizon depends on which routes must actually be passed to spokes.
- `no ip redirects` disables ordinary **ICMP Redirects**, not NHRP Redirects. The latter are controlled independently by `ip nhrp redirect`.

### Spoke1

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
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255
 network 10.11.11.11 0.0.0.0
```

The consolidated `ip nhrp nhs ... nbma ... multicast` command establishes the NHS and its unicast/multicast NBMA mapping. It does **not** dynamically map other spokes; those mappings are learned through NHRP resolution.

### Spoke2

```cisco
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
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255
 network 10.12.12.12 0.0.0.0
```

### Spoke3

```cisco
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
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255
 network 10.13.13.13 0.0.0.0
```

### Adding the Optional Summary-Map

On Spoke2, add a second simulated LAN and advertise it into EIGRP:

```cisco
interface Loopback1
 ip address 10.12.13.13 255.255.255.0
!
router eigrp 100
 network 10.12.13.13 0.0.0.0
!
interface Tunnel0
 ip nhrp summary-map 10.12.12.0/23
```

Then test destinations in **both** `10.12.12.0/24` and `10.12.13.0/24` from Spoke1. The NHRP summary map should allow a single /23 shortcut to cover both networks, assuming the path and covering route are valid.

**This is a GRE/NHRP forwarding lab, not an encrypted VPN lab.** IKEv2/IPsec protection and its additional peer-state requirements appear in [IPsec/IKEv2 Using Preshared Key](../ipsec/overview.md).

---

## Verification Workflow

### 1. Verify Registrations Before Looking for Shortcuts

On Hub1:

```cisco
show dmvpn
show ip nhrp
```

The hub should have registrations for `172.16.0.11`, `.12`, and `.13`, mapped to the correct WAN addresses. Registration alone does **not** establish shortcuts.

On Spoke1:

```cisco
show ip nhrp nhs detail
show ip nhrp
```

Check that Hub1 is a valid NHS and its static mapping resolves to `198.51.100.1`.

### 2. Verify the Base Route Points to Hub1

On Spoke1:

```cisco
show ip route 10.12.12.12
show ip eigrp neighbors
```

Before a shortcut, the EIGRP summary should cover the destination and use Hub1 as next hop. If the spoke already has a direct-spoke next hop from the IGP, this example no longer tests the Phase 3 Redirect path as intended.

### 3. Observe the First Flow and Redirect

On Hub1, take baseline NHRP counters:

```cisco
show ip nhrp traffic
show ip nhrp redirect
```

On Spoke1, generate traffic:

```cisco
ping 10.12.12.12 source 10.11.11.11 repeat 10
```

On Hub1, examine the counters again for **Traffic Indication/Redirect** activity. On the source, look for Resolution Requests; on Spoke2, look for requests received and replies sent.

A missing first echo reply does not automatically mean the design is broken. Initial NHRP resolution, CEF updates, and optional IPsec negotiations may take time.

### 4. Verify the Installed Shortcut

On Spoke1:

```cisco
show ip nhrp shortcut
show ip nhrp
show ip route nhrp
show ip route next-hop-override
show ip route 10.12.12.12
show ip cef 10.12.12.12 detail
show dmvpn detail
```

With Hub1's /8 summary and Spoke2's /24 LAN, the expected result is typically **`H 10.12.12.0/24` via `172.16.0.12`**. If an identical /24 EIGRP route already exists, inspect **NHO** instead.

### 5. Verify Direct Underlay Encapsulation

The actual direct GRE outer destination should be **Spoke2's NBMA `198.51.100.13`**, not Hub1's `198.51.100.1`.

Use packet captures on the ISP-facing links, or tunnel peer/CEF/DMVPN details as supported by the platform. For repeated tests, note that an installed shortcut can already be active when the test begins.

### 6. Observe the Return Path Independently

Spoke2 must also route return packets to `10.11.11.11`. Its own shortcut to Spoke1 may be learned **independently** after reverse traffic and relevant redirects/resolution.

**Do not assume that one successful forward shortcut guarantees a matching reverse shortcut.** NHRP mapping state, routed paths, and any IPsec peer state must support the traffic in both directions.

---

## Troubleshooting Redirects and Shortcuts

| Symptom | Likely area | What to inspect |
| --- | --- | --- |
| Hub registers spokes but never sends Redirects | `ip nhrp redirect` absent; packets not hairpinning on the same mGRE interface; control-plane rate limits | Hub tunnel config; ingress/egress interfaces; `show ip nhrp redirect`; punt counters |
| Redirects sent but Spoke1 sends no Resolution Request | Shortcut not enabled; Redirect never reaches spoke; control message issue | `ip nhrp shortcut`; spoke NHRP traffic counters; NHRP debug |
| Resolution Request reaches hub but not destination | Hub lacks a correct route/mapping for the destination spoke | Hub RIB; NHRP registration; NHRP traffic counters |
| Spoke2 receives request but Spoke1 receives no Reply | Direct underlay or IPsec path blocked; bad return NBMA information | Spoke-to-spoke WAN reachability; packet capture; GRE/IPsec state |
| Resolution Reply received but no `H` route or NHO | Missing or changed covering route; noninstallable prefix; next-hop resolution issue | `show ip nhrp shortcut`; RIB; Route Watch; CEF |
| Only `/32` or `/24` shortcuts appear | Destination-spoke RIB prefix is that narrow | Destination RIB and optional `ip nhrp summary-map` |
| Shortcut covers other sites' LANs and breaks traffic | NHRP summary map too broad | Which spoke actually owns every address in advertised summary |
| Shortcut disappears after routing convergence | Covering route or overlay next-hop route changed | EIGRP/OSPF/BGP RIB changes; Route Watch; NHS reachability |
| `H` route exists but packets still fail | Direct GRE/IPsec/underlay issue; missing return path | CEF, NBMA reachability, security SAs, reverse RIB |
| High hub Redirect counts and no reduction in hairpin traffic | Failed repeated shortcut setup; redirect storms/rate limiting | `show ip nhrp redirect`, `show ip nhrp traffic`, punt policers |
| `show ip route nhrp` is empty despite direct forwarding | A **next-hop override** was installed, not an `H` route | `show ip route next-hop-override` and `%` marker |

### Failure-Domain Isolation

For a Phase 3 direct path, separately validate:

1. **IGP/BGP routing:** Source has a valid route to the destination via the hub.
2. **Registration:** The hub knows how to reach both spokes.
3. **Redirect:** The hub forwards the initial packet and generates the expected indication.
4. **Resolution request:** The source receives the indication and sends the request; the hub forwards it.
5. **Resolution reply:** The destination answers, and the response reaches the source.
6. **Shortcut programming:** NHRP installs an appropriate RIB route or override, and CEF uses it.
7. **Data forwarding:** GRE/IPsec and the underlay permit direct packets in both directions.

The dedicated [NHRP Verification and Troubleshooting](troubleshooting.md) page develops the commands and failure analysis for each stage.

---

## Dual-Hub Implications

With multiple NHSs, each spoke must have usable registration and underlay reachability to the relevant hub(s). Phase 3 Redirects are **associated with the hub that actually forwards the suboptimal traffic**; do not assume every registered hub will generate a Redirect for every flow.

- **Single-cloud dual-hub:** Both hubs and spokes share one DMVPN overlay. A spoke commonly has one mGRE tunnel and multiple NHS entries. Routing decides which hub initially carries traffic; NHRP then discovers the shortcut.
- **Dual-cloud dual-hub:** Spokes have separate mGRE tunnels, often with different overlay subnets and tunnel keys. Resolution and shortcut forwarding must work within the selected cloud and its routing/security context.
- **Failover:** A change of active hub, covering route, or tunnel path can invalidate existing shortcut state. Route Watch and normal routing convergence determine whether a shortcut remains usable.

See [Dual-Hub DMVPN Topologies](../dual-hub/topologies.md) and [Dual-Hub Failover and Convergence](../dual-hub/failover.md) for configuration and failure scenarios.

---

## Lab Exercises

1. **Confirm initial hub forwarding:** Configure Hub1's EIGRP `/8` summary, remove prior shortcut state, and confirm Spoke1's route to `10.12.12.12` initially points to Hub1.
2. **Watch a Redirect:** Generate a sourced ping from Spoke1 to Spoke2 and compare `show ip nhrp traffic` counters on all three devices.
3. **Verify a more-specific `H` route:** Confirm the `10.12.12.0/24` shortcut and its next hop at Spoke1.
4. **Prove the direct GRE destination:** Capture underlay traffic and compare outer destinations before and after the shortcut.
5. **Test NHO:** Replace the broad EIGRP summary with an advertised exact `/24` via Hub1 and inspect `%` and `[NHO]` on Spoke1.
6. **Test summary maps:** Add `10.12.13.0/24` behind Spoke2, configure `ip nhrp summary-map 10.12.12.0/23`, and verify the resulting /23 shortcut.
7. **Test Route Watch:** Withdraw the route covering Spoke2's LAN at Spoke1 and observe whether shortcut state is removed.
8. **Disable Redirect:** Remove `ip nhrp redirect` on Hub1, clear shortcut state, and compare initial and subsequent paths.
9. **Disable Shortcut:** Remove `ip nhrp shortcut` on Spoke1 and verify that normal hub forwarding continues without local shortcut installation.
10. **Break direct underlay reachability:** Block direct GRE between Spoke1 and Spoke2 while keeping hub reachability; distinguish routing, NHRP, and transport failures.
11. **Inspect redirect suppression:** Generate traffic to many previously unresolved destinations and examine Redirect counters and rate limiting without changing policer settings.
12. **Evaluate reverse traffic:** Determine whether Spoke2 also has a direct shortcut to Spoke1 and what triggers its installation.

---

## Key Points

- **Phase 3 allows routes to point to the hub** while NHRP installs local shortcuts to remote spokes.
- The hub uses **`ip nhrp redirect`**; spokes use **`ip nhrp shortcut`**.
- A Redirect is normally triggered when a hub forwards traffic **in and out the same DMVPN mGRE interface** toward another spoke.
- The hub **forwards the initial packet** and separately sends the NHRP Redirect to the source spoke.
- The source spoke then sends an **NHRP Resolution Request for the destination**, forwarded through the NHS toward the destination spoke.
- The destination spoke normally sends an **NHRP Resolution Reply directly to the source**.
- The reply can describe a **subnet prefix**, not only the original destination IP.
- A shortcut **more specific than the covering route** can be installed as an **`H` route** (typically AD 250).
- When the NHRP shortcut matches an existing route's prefix, NHRP can install a **`[NHO]`** next-hop override. The original route is marked with **`%`**.
- **`ip nhrp summary-map` on the destination spoke** can return an aggregate prefix in Resolution Replies, reducing repeated resolutions.
- **EIGRP summarization on the hub** changes routing advertisements; **NHRP summary maps on spokes** change shortcut resolution replies.
- **Route Watch** ties shortcuts to valid covering routing information; an `H` route is not an independent substitute for the base routing topology.
- Purges, mapping expiration, routing changes, and security/underlay failures can remove or invalidate shortcuts.
- Repeated hub Redirects can indicate **failed shortcut establishment**, not just high traffic volume.
- The key verification commands are **`show ip nhrp shortcut`**, **`show ip route nhrp`**, **`show ip route next-hop-override`**, **`show ip nhrp traffic`**, **`show ip nhrp redirect`**, and **`show ip cef`**.

---

## References

- [Cisco: Shortcut Switching Enhancements for NHRP in DMVPN Networks](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-addressing/ip-addressing/m_nhrp-switch-enhancemts-dmvpn-0.html)
- [Cisco: Spoke-to-Spoke NHRP Summary Maps](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m-sec-conn-dmvpn-summ-maps.html)
- [Cisco: Configure BGP over DMVPN Phase 3](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/222585-configure-bgp-over-dmvpn-phase-3.html)
- [Cisco: Troubleshoot DMVPN Phase 3 NHRP Redirect Issues](https://www.cisco.com/c/en/us/support/docs/security-vpn/dynamic-multi-point-vpn-dmvpn/218427-troubleshoot-dmvpn-phase3-nhrp-redirect.html)
- [Cisco: Troubleshoot DMVPN Issues](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/111976-dmvpn-troubleshoot-00.html)
- [RFC 2332: NBMA Next Hop Resolution Protocol](https://www.rfc-editor.org/rfc/rfc2332.html)
