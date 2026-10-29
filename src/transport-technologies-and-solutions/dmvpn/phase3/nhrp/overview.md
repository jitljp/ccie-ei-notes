# 3.3.a (i) NHRP

In **DMVPN Phase 3**, the **Next Hop Resolution Protocol (NHRP)** does more than register spokes and resolve tunnel addresses to NBMA addresses. It also enables **direct spoke-to-spoke forwarding when the routing protocol initially points traffic toward a hub**.

The two essential Phase 3 features are **NHRP Redirect**, configured on hubs, and **NHRP Shortcut**, configured on spokes. Together, they allow spokes to use routing information learned through the hub, including summary or default routes, while dynamically discovering more direct forwarding paths.

This page introduces the Phase 3 NHRP control plane, shortcut forwarding state, and the verification workflow. The message processing and optional NHRP summary maps are examined in [NHRP Redirects and Shortcuts](redirect-shortcut.md). Command output and failure isolation are covered in [NHRP Verification and Troubleshooting](troubleshooting.md).

For registration, static and dynamic mappings, authentication, timers, and `ip nhrp registration no-unique`, see [NHRP Fundamentals](../../nhrp-fundamentals.md).

---

## NHRP Functions in Phase 3

NHRP performs several related but distinct functions:

| Function | Purpose | Typical participants |
| --- | --- | --- |
| **Registration** | Lets the NHS learn each spoke's tunnel-to-NBMA mapping. | Spoke to hub |
| **Multicast mapping** | Identifies NBMA destinations for GRE-encapsulated multicast routing traffic. | Spokes toward hub; hub toward registered spokes |
| **Redirect** | Informs a source spoke that its traffic is taking a suboptimal path through the hub. | Hub to source spoke |
| **Resolution** | Discovers the tunnel/NBMA endpoint responsible for a destination. | Source spoke, hub, destination spoke |
| **Shortcut** | Installs forwarding information that sends subsequent packets directly toward the destination spoke. | Source spoke |
| **Purge and aging** | Invalidates or expires information that must no longer be used. | NHRP peers |

**NHRP Registration is not an NHRP Shortcut.** A spoke can be registered with the hub and exchange routes normally without having resolved any direct spoke-to-spoke paths.

The key distinction from Phase 2 is **what initiates resolution**:

- **Phase 2:** The routing protocol must normally select the **destination spoke's tunnel address** as the next hop. NHRP resolves that next hop to its NBMA address.
- **Phase 3:** The routing protocol may select the **hub's tunnel address** as the next hop. The hub forwards initial traffic and sends an NHRP Redirect; the source spoke then resolves the destination and installs a shortcut.

Neither phase requires an EIGRP, OSPF, or BGP neighbor relationship directly between spokes.

---

## Reference Topology

The example uses the existing lab's single-cloud overlay with Hub1 and three spokes. Hub2 is introduced in the later dual-hub labs.

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

| Router | Tunnel0 | NBMA (WAN) | Simulated LAN |
| --- | --- | --- | --- |
| Hub1 | `172.16.0.1/24` | `198.51.100.1` | `10.1.1.0/24` |
| Spoke1 | `172.16.0.11/24` | `198.51.100.9` | `10.11.11.0/24` |
| Spoke2 | `172.16.0.12/24` | `198.51.100.13` | `10.12.12.0/24` |
| Spoke3 | `172.16.0.13/24` | `198.51.100.17` | `10.13.13.0/24` |

All routers use **mGRE**. There is no fixed `tunnel destination` on the hub or the spokes. The ISP carries packets between NBMA addresses but does not know the simulated LAN prefixes.

---

## Essential Phase 3 NHRP Configuration

The following examples show the **NHRP and GRE portions** of the tunnel configuration. They assume the underlay and overlay routing protocol are configured separately. The GRE key and NHRP authentication string follow the previous lab examples.

### Hub1: NHRP Redirect

```cisco
interface Tunnel0
 ip address 172.16.0.1 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp network-id 100
 ip nhrp map multicast dynamic
 ip nhrp redirect
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
```

`ip nhrp redirect` allows the hub to notify a spoke when the hub forwards traffic back out the DMVPN tunnel toward another spoke. The hub continues to forward the original packet; the redirect initiates a separate control-plane process to optimize later packets.

`ip nhrp map multicast dynamic` supplies multicast replication destinations from registered spokes for multicast-based routing protocols such as EIGRP or OSPF. It is **not** what enables Phase 3 shortcuts.

### Spoke1: NHRP Shortcut

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

- `ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast` identifies the hub's **tunnel address** and **NBMA address**, and uses the hub as a multicast destination. The `multicast` keyword is needed here for the multicast routing-protocol example, not for NHRP registration itself.
- `ip nhrp shortcut` enables the spoke to process redirects and install direct forwarding shortcuts based on successful NHRP resolution.

Spoke2 and Spoke3 use the same commands with their respective tunnel and WAN addresses.

**The redirect command belongs on the hub; the shortcut command belongs on the spokes.** The hub does not need `ip nhrp shortcut` merely to act as the NHS, and the spokes do not need `ip nhrp redirect` merely to respond to resolution requests.

IKEv2/IPsec protection is configured separately in the [IPsec/IKEv2 section](../ipsec/overview.md). GRE and NHRP alone do not encrypt traffic.

---

## NHRP Redirect and Shortcut Sequence

Assume Spoke1 sends traffic from `10.11.11.11` to `10.12.12.12`, which is behind Spoke2. The overlay routing protocol has given Spoke1 a covering route such as `10.0.0.0/8` via Hub1 (`172.16.0.1`).

```text
 Spoke1                    Hub1 (NHS)                   Spoke2
    |                          |                          |
    |--- Data packet --------->|---- Data packet -------->|
    |    via hub               |    forwarded to Spoke2   |
    |                          |                          |
    |<--- NHRP Redirect -------|                          |
    |                          |                          |
    |--- Resolution Request -->|--- Resolution Request -->|
    |                          |                          |
    |<------------------- Resolution Reply ---------------|
    |             destination prefix and NBMA             |
    |                          |                          |
    |<================ Direct GRE data ==================>|
```

1. **Initial route selection:** Spoke1's routing table selects the covering route via Hub1. The first packets use Spoke1's existing mapping for the hub's NBMA address, `198.51.100.1`.
2. **Hub forwarding:** Hub1 decapsulates the packet and forwards it through its mGRE tunnel toward Spoke2's NBMA address, `198.51.100.13`.
3. **Redirect:** Because traffic arrived and departed through the DMVPN tunnel, Hub1 can send an **NHRP Redirect** to Spoke1, indicating that the hub-forwarded path should be optimized.
4. **Resolution Request:** Spoke1 requests information for the destination through the NHRP control plane. The request reaches Hub1, which forwards it toward Spoke2.
5. **Resolution Reply:** Spoke2 normally replies **directly to Spoke1**, identifying the reachable destination prefix and Spoke2's tunnel/NBMA endpoint.
6. **Shortcut installation:** Spoke1 updates its NHRP and routing/forwarding information so subsequent packets for the resolved prefix can use Spoke2 directly.
7. **Direct data path:** Spoke1 encapsulates GRE packets with **outer destination `198.51.100.13`**, bypassing Hub1 for that traffic.

The routing-protocol adjacency remains **Spoke1 to Hub1**. NHRP changes the forwarding path without requiring a new spoke-to-spoke routing-protocol adjacency.

The first packets do **not** have to wait for NHRP resolution to complete: they already have a valid route through the hub. A direct path is an optimization of an initially working hub-routed path.

---

## What Prefix Does an NHRP Shortcut Cover?

Phase 3 resolution can describe a **destination subnet**, not only the individual IP address that triggered the redirect.

For example:

```text
Original destination:       10.12.12.12
Spoke2's matching LAN route: 10.12.12.0/24
NHRP Resolution Reply:      10.12.12.0/24 via Spoke2
Spoke2 tunnel/NBMA mapping: 172.16.0.12 -> 198.51.100.13
```

The resulting shortcut can serve **the entire `10.12.12.0/24` prefix**. Another destination in that /24 does not necessarily require a separate resolution. If Spoke2 responds with a /32, the shortcut covers only that host.

The destination spoke normally uses routing information to determine the returned prefix. The optional `ip nhrp summary-map` feature can cause a spoke to return a broader prefix in Resolution Replies. This reduces repeated resolution requests for multiple LAN subnets, provided the summarized destinations are reachable through that spoke.

**EIGRP summarization on the hub and NHRP summary maps on a spoke serve different purposes:**

| Feature | Controls |
| --- | --- |
| **EIGRP summary on the hub** | Which routing prefixes spokes learn through EIGRP. |
| **NHRP summary map on the destination spoke** | Which prefix the source spoke can learn for a direct NHRP shortcut. |

The configuration and implications of `ip nhrp summary-map` are covered in [NHRP Redirects and Shortcuts](redirect-shortcut.md).

---

## How a Shortcut Appears in the RIB and FIB

Phase 3 NHRP can affect routing and forwarding information in **two important ways**. The result depends on the prefix returned by NHRP relative to the routes already installed on the source spoke.

### More-Specific NHRP Route (`H`)

Suppose Spoke1 learns a summary through EIGRP:

```text
D        10.0.0.0/8 [90/27008000] via 172.16.0.1, 00:00:15, Tunnel0
```

Spoke2 replies with `10.12.12.0/24`. Because this is more specific than the existing `/8`, NHRP can install a separate route:

```text
D        10.0.0.0/8 [90/27008000] via 172.16.0.1, 00:00:15, Tunnel0
H        10.12.12.0/24 [250/255] via 172.16.0.12, 00:00:15, Tunnel0
```

The `H` code identifies a route installed by NHRP. **Longest-prefix matching** selects the `/24` over the `/8` for matching traffic, even though the NHRP route has a higher administrative distance.

### Next-Hop Override (`%` and `[NHO]`)

Suppose Spoke1 already has an EIGRP route for `10.12.12.0/24` through Hub1, and NHRP returns a shortcut for the **same prefix**. Rather than adding a competing `H` route, NHRP can override the forwarding next hop of the existing route:

```text
D   %    10.12.12.0/24 [90/28288000] via 172.16.0.1, 00:01:50, Tunnel0
                       [NHO][90/255] via 172.16.0.12, 00:01:30, Tunnel0
```

The EIGRP route remains present, but forwarding uses the **NHRP next-hop override (NHO)** toward Spoke2. The `%` marker identifies the route as having an override.

NHRP can also use an NHO when the resolved prefix is less specific than the existing longest-matching route, with the override associated with that route rather than replacing it with a broad NHRP route.

**Do not rely only on `show ip route nhrp`.** That command displays NHRP-installed `H` routes, but a working shortcut may instead be represented as an NHO attached to another route. Use both `show ip route next-hop-override` and `show ip cef` when checking the effective forwarding path.

---

## Shortcut Validity and Route Watch

An NHRP shortcut is **not a replacement for general routing reachability**. The source spoke still depends on appropriate route information, including the route that originally covered the destination through the hub.

For an NHRP shortcut installed as a more-specific route, **Route Watch** tracks its relationship with a covering route in the RIB. If the covering route disappears and no valid replacement covers the shortcut, the NHRP route is removed. Changes to covering-route next hops can also invalidate the shortcut.

This prevents a shortcut from remaining indefinitely when its underlying routing context has changed. NHRP cache lifetimes and refresh/purge behavior impose additional limits on shortcut state.

The practical implication is that a shortcut can disappear **without any direct GRE interface going down**. When a spoke-to-spoke path disappears, inspect routing changes and NHRP state separately.

---

## NHRP in Dual-Hub Phase 3

With dual hubs, NHRP registration and forwarding selection must be distinguished.

| Design | Spoke tunnel interfaces | NHS configuration |
| --- | --- | --- |
| **Dual-hub, single-cloud** | One mGRE tunnel per spoke | Two NHSs on the same tunnel. |
| **Dual-hub, dual-cloud** | One mGRE tunnel per cloud, typically two per spoke | NHS for each respective cloud/interface. |

For example, a single-cloud spoke can register with both Hub1 (`172.16.0.1`, NBMA `198.51.100.1`) and Hub2 (`172.16.0.2`, NBMA `198.51.100.5`). **Each NHS maintains its own registration state**; registration with one hub does not prove registration with the other.

Multiple NHS registrations do not automatically define primary/backup routing or guarantee a working shortcut. A spoke's selected hub path depends on **overlay routing**, while direct spoke-to-spoke transport still requires reachable NBMA endpoints and, if enabled, compatible IPsec protection.

See [Dual-Hub DMVPN Topologies](../dual-hub/topologies.md) and [Dual-Hub Failover and Convergence](../dual-hub/failover.md) for the topology and failover details.

---

## NHRP Verification Overview

Use these commands to identify **which stage** of Phase 3 operation is failing:

| Command | Primary purpose |
| --- | --- |
| `show dmvpn` | Display peers and DMVPN tunnel/NHRP state. |
| `show ip nhrp` | Display registered, static, and dynamically resolved mappings. |
| `show ip nhrp nhs detail` | Check configured NHSs and registration status on spokes. |
| `show ip nhrp traffic` | Examine NHRP message counters, including registration, resolution, and redirect activity. |
| `show ip nhrp shortcut` | Display NHRP entries associated with a shortcut route or next-hop override. |
| `show ip route nhrp` | Display `H` routes installed by NHRP. |
| `show ip route next-hop-override` | Display effective NHRP next-hop overrides (`[NHO]`). |
| `show ip route <destination>` | Inspect the route that originally covers a destination. |
| `show ip cef <destination> detail` | Inspect the destination's forwarding adjacency and actual next hop. |
| `traceroute <destination> source <source>` | Compare the initial hub path against the optimized path after a shortcut forms. |

A useful verification sequence is:

1. **Before traffic:** Check that spokes are registered with the hub and have an overlay route toward the destination via the hub.
2. **Generate traffic:** Send traffic from Spoke1's LAN to Spoke2's LAN.
3. **At the hub:** Check whether the packet traverses the hub and whether redirect activity is observed.
4. **At Spoke1:** Check whether resolution/shortcut state appears, either as an `H` route or an NHO.
5. **Check the data plane:** Confirm that subsequent packets use Spoke2's NBMA address rather than Hub1's.

**Successful registration is not proof of successful shortcut forwarding.** Likewise, an NHRP cache entry does not by itself prove that the expected FIB entry was installed.

### Common Failure Boundaries

| Observation | Investigate |
| --- | --- |
| Spoke cannot register with the NHS | Underlay reachability, GRE parameters, NHRP authentication, NHS configuration, and any IPsec protection. |
| Spoke reaches remote LAN through hub but never receives a redirect | Hub `ip nhrp redirect`, actual ingress/egress interfaces, routing path, and NHRP counters. |
| Redirect arrives but no usable Resolution Reply returns | Spoke `ip nhrp shortcut`, NHS forwarding, destination-spoke reachability, and return transport path. |
| Resolution succeeds but traffic still uses the hub | RIB/FIB shortcut state, covering route, NHO state, and effective CEF next hop. |
| Direct mapping exists but packets fail | Spoke-to-spoke NBMA routing, GRE delivery, IPsec peer SAs, and return routing. |
| Shortcut disappears during a routing change | Route Watch/covering-route changes, NHRP holding time, purge state, and hub failover. |

The detailed command outputs and step-by-step fault isolation belong in [NHRP Verification and Troubleshooting](troubleshooting.md).

---

## Key Points

- **Phase 3 NHRP Redirect** allows the hub to inform a spoke that its traffic is following a suboptimal hub path.
- **Phase 3 NHRP Shortcut** allows the source spoke to discover and use direct spoke-to-spoke forwarding.
- The hub can remain the **routing-protocol next hop**. Direct forwarding does not require the routing protocol to advertise the destination spoke as next hop.
- The **NHRP Resolution Request** is forwarded toward the destination spoke; the destination spoke normally sends its Resolution Reply directly to the source.
- A Resolution Reply may identify an **entire destination subnet**, not just the host that triggered resolution.
- `ip nhrp summary-map` on a spoke can influence the prefix returned for shortcuts; this is different from hub EIGRP summarization.
- A **more-specific shortcut** can appear as an `H` route, while a matching-prefix shortcut can appear as a `%` route with `[NHO]`.
- **Route Watch** ties an NHRP shortcut route to its covering-route reachability.
- **Registration, routing, resolution, forwarding, and IPsec** are separate troubleshooting layers.
- In a dual-hub topology, NHS redundancy and the selected overlay forwarding path must be verified independently.

---

## Further Reading

- [NHRP Redirects and Shortcuts](redirect-shortcut.md)
- [NHRP Verification and Troubleshooting](troubleshooting.md)
- [Phase 3 Operation and Packet Forwarding](../operation.md)
- [Routing over DMVPN Phase 3](../routing.md)
- [NHRP Fundamentals](../../nhrp-fundamentals.md)

## References

- [Cisco: Shortcut Switching Enhancements for NHRP in DMVPN Networks](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-addressing/ip-addressing/m_nhrp-switch-enhancemts-dmvpn-0.html)
- [Cisco: Spoke-to-Spoke NHRP Summary Maps](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m-sec-conn-dmvpn-summ-maps.html)
- [Cisco: Configure BGP over DMVPN Phase 3](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/222585-configure-bgp-over-dmvpn-phase-3.html)
- [Cisco: Troubleshoot DMVPN Issues](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/111976-dmvpn-troubleshoot-00.html)
