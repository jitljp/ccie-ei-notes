# DMVPN Architecture and Components

**DMVPN** (Dynamic Multipoint VPN) combines **multipoint GRE (mGRE)**, the **Next Hop Resolution Protocol (NHRP)**, and IP routing to build a dynamic overlay across an IP underlay. **IPsec** can be added to protect the GRE traffic.

The central architectural idea is to separate three decisions:

1. **Routing:** Which overlay next hop should receive a packet for a destination network?
2. **NHRP:** Which underlay (NBMA) address corresponds to that overlay next hop, or to a discovered shortcut destination?
3. **GRE/IPsec:** How should the packet be encapsulated and, when configured, protected for transmission across the underlay?

This separation allows a hub to serve many spokes with a single tunnel interface and, in DMVPN Phases 2 and 3, allows spokes to communicate directly without preconfiguring a separate point-to-point GRE tunnel for every spoke pair.

---

## Reference Topology

The chapter uses **two hubs, three spokes, and one ISP router**:

```text
          Hub1                         Hub2
            \                           /
             \                         /
              +--------- ISP ---------+
                        / | \
                       /  |  \
                      /   |   \
                 Spoke1 Spoke2 Spoke3
```

This is the **physical underlay**, not a diagram of the GRE tunnels. Each DMVPN router has an independent connection to the ISP. The ISP can reach all five routers' WAN addresses but has no routes to the internal networks behind them.

The same physical topology will support:

- **Phase 1:** Hub1 with three point-to-point GRE spokes.
- **Phase 2:** Hub1 with mGRE spokes and dynamic spoke-to-spoke forwarding.
- **Phase 3:** Hub1 with NHRP redirects and shortcuts.
- **Dual-hub, single-cloud Phase 3:** Both hubs and all spokes share one DMVPN overlay.
- **Dual-hub, dual-cloud Phase 3:** Each hub serves a separate overlay, and each spoke participates in both.

The ISP topology does not have to change when the logical DMVPN design changes.

---

## DMVPN Layers

DMVPN is easiest to troubleshoot by treating the underlay, overlay control plane, overlay forwarding, and security as distinct layers.

| Layer | Components | Responsibility |
| --- | --- | --- |
| **Underlay** | WAN interfaces, ISP, underlay routes | Deliver packets between remote NBMA addresses. |
| **Tunnel encapsulation** | GRE or mGRE, tunnel source, optional GRE key | Encapsulate passenger packets and select a remote tunnel endpoint. |
| **Endpoint discovery** | NHRP clients, NHSs, registrations, mappings | Associate tunnel/shortcut destinations with NBMA addresses. |
| **Overlay routing** | Static routes, EIGRP, OSPF, or BGP | Learn internal prefixes and select overlay next hops. |
| **Forwarding optimization** | NHRP resolution, Phase 3 redirects/shortcuts, CEF | Forward directly between spokes when supported and appropriate. |
| **Optional security** | IKEv2, IPsec ESP, IPsec profiles | Authenticate IPsec peers and protect tunnel traffic. |

These layers depend on one another but perform different jobs. A correct overlay route cannot forward a packet if the corresponding tunnel endpoint is unresolved. A correct NHRP entry does not create general reachability to a remote LAN unless forwarding information also exists.

### Underlay Reachability

The **underlay** carries the outer tunnel packets. In this lab, every DMVPN router uses a default route pointing toward the ISP, and the ISP knows only its directly connected WAN networks.

For example, Spoke1 can reach the WAN address of Hub1, Hub2, and the other spokes through the ISP. This is sufficient for the underlay; the ISP does **not** need a route to Spoke2's internal LAN prefix.

The remote NBMA destination must resolve through a valid underlay path. If a route to a remote tunnel endpoint resolves back through the DMVPN tunnel, recursive forwarding can break tunnel operation.

### Overlay Reachability

The **overlay** consists of logical tunnel interfaces and routes that use those interfaces. A router can learn a route to a remote site through EIGRP, OSPF, or BGP even though the ISP has no knowledge of that site's internal prefix.

The routing protocol runs *over* the GRE tunnel. It is independent of the ISP's routing domain.

---

## Point-to-Point GRE Versus mGRE

A conventional point-to-point GRE tunnel identifies one fixed destination:

```cisco
interface Tunnel0
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.1
 tunnel mode gre ip
```

Traffic sent through `Tunnel0` is encapsulated toward the configured `tunnel destination`. Connecting to a second remote endpoint normally requires a different point-to-point GRE tunnel.

An **mGRE** interface has no configured `tunnel destination`:

```cisco
interface Tunnel0
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
```

Instead, its remote GRE destination is selected **per packet or per forwarding adjacency** from the mapping associated with the selected overlay next hop.

```text
                             Hub1
					 One mGRE tunnel interface
                               |
                      +--------+--------+
                      |        |        |
                    Spoke1   Spoke2   Spoke3
```

The diagram represents possible GRE destinations, not three permanently configured tunnel interfaces.

**mGRE provides the ability to send to multiple endpoints; NHRP provides the endpoint mappings.** mGRE does not, by itself, discover the NBMA addresses of other routers.

### A Dynamic Spoke-to-Spoke Tunnel Is Not a New Interface

In Phases 2 and 3, the phrase *dynamic spoke-to-spoke tunnel* describes an on-demand **GRE forwarding relationship** between two existing mGRE interfaces. IOS XE does not need to create a new `Tunnel` interface every time Spoke1 starts communicating with Spoke2.

When IPsec is enabled, a direct spoke-to-spoke path can also require a separate IKEv2/IPsec peer relationship and security associations for the two spokes.

The GRE destination mapping, forwarding state, and IPsec security associations are related but are **not the same state**.

---

## Tunnel Addresses, NBMA Addresses, and LAN Addresses

NHRP distinguishes between the address used **inside the DMVPN overlay** and the address used to **reach a router through the underlay**.

**NBMA** means *Non-Broadcast Multiple Access*. In DMVPN documentation, an **NBMA address** is typically the IP address on the router's WAN-facing interface, used as the destination of the outer GRE/IPsec traffic.

Consider the future single-cloud overlay in this lab:

| Address type | Hub1 | Spoke1 |
| --- | --- | --- |
| **NBMA (underlay)** | `198.51.100.1` | `198.51.100.9` |
| **Tunnel (overlay)** | `172.16.0.1` | `172.16.0.11` |
| **Internal network** | `10.1.1.0/24` | `10.11.11.0/24` |

The following NHRP mapping would identify Spoke1's underlay destination:

```text
Tunnel address: 172.16.0.11
NBMA address:   198.51.100.9
```

These addresses have different purposes:

- The **internal network** identifies the ultimate destination of user traffic.
- The **tunnel address** identifies a router or next hop in the overlay.
- The **NBMA address** tells GRE where to send the encapsulated packet across the underlay.

Do not confuse an NHRP tunnel-to-NBMA mapping with a route to the spoke's internal LAN. The mapping alone is not a replacement for overlay routing.

---

## Hub, Spoke, NHS, and NHC Roles

DMVPN typically uses **hub-and-spoke control-plane connectivity** even when user packets eventually take direct spoke-to-spoke paths.

| Role | Responsibility |
| --- | --- |
| **Hub** | Provides a stable overlay contact point, often acts as an NHRP server and routing-protocol neighbor, and can forward spoke-to-spoke traffic when a direct path is not in use. |
| **Spoke** | Registers its tunnel-to-NBMA mapping with an NHS and learns routes through the overlay. In Phases 2/3, it can establish direct paths to other spokes. |
| **NHS (Next Hop Server)** | Receives and maintains NHRP registration information and participates in resolving or forwarding NHRP requests. Usually a hub. |
| **NHC (Next Hop Client)** | Registers with the NHS and requests resolution information when it needs to reach another endpoint. Usually a spoke. |

**Hub** and **spoke** describe the network roles. **NHS** and **NHC** describe the NHRP roles. They are related, but they are not merely different names for GRE encapsulation functions.

### NHRP Registration

Before a hub can dynamically send GRE packets to a spoke, it needs to learn the spoke's NBMA destination.

A simplified registration sequence is:

1. Spoke1 has underlay connectivity to Hub1's NBMA address.
2. Spoke1 is configured with Hub1 as its NHS and knows how to reach Hub1's tunnel address through the underlay.
3. Spoke1 sends an **NHRP Registration Request** containing its tunnel/NBMA information.
4. Hub1 records the registration in its NHRP cache and returns a **Registration Reply**.
5. Spoke1 refreshes its registration as needed; the learned information has a finite lifetime.

```text
Spoke1                                      Hub1 (NHS)
   |                                               |
   | ------------ Registration Request ----------> |
   | <------------ Registration Reply ------------ |
   |                                               |
```

This exchange supplies the hub with information needed to encapsulate traffic toward Spoke1. It is **not a routing-protocol adjacency** and does not, by itself, advertise Spoke1's LAN routes.

### NHRP Resolution

A spoke can also use NHRP to discover another spoke's NBMA address:

```text
Spoke1                   Hub1 (NHS)                 Spoke2
   |                           |                        |
   | ---- Resolution Request ->| ---- Request --------> |
   | <------------- Resolution Reply ------------------ |
   |                           |                        |
```

The diagram shows a common resolution path. The NHS helps direct the request toward the appropriate destination; the responding spoke can send the reply directly to the requester. The exact behavior depends on the DMVPN phase and NHRP configuration.

Registration and resolution are **different operations**:

| Operation | Main question answered |
| --- | --- |
| **Registration** | "How can the NHS reach this spoke's tunnel address?" |
| **Resolution** | "Which NBMA address should I use to reach the requested tunnel endpoint or shortcut destination?" |

NHRP registration, resolution, cache entries, timers, and message types are covered in detail in [NHRP Fundamentals](nhrp-fundamentals.md).

---

## Static and Dynamic NHRP Mappings

The first contact with an NHS cannot depend entirely on an unknown dynamic mapping. In a conventional configuration, the spoke is given a **static mapping for the hub**:

```cisco
ip nhrp map 172.16.0.1 198.51.100.1
ip nhrp nhs 172.16.0.1
```

The first command maps Hub1's **overlay tunnel address** to its **underlay NBMA address**. The second identifies the NHS by its **tunnel address**, not its WAN address.

Once a spoke has registered, its mapping at the hub is learned dynamically. Spoke-to-spoke resolution in Phases 2 and 3 can create further dynamic mappings at the initiating spoke.

A useful distinction is:

- **Static hub mapping at a spoke:** Provides initial reachability to a known NHS.
- **Dynamic spoke registration at a hub:** Allows the hub to reach spokes without maintaining a separate static mapping for each one.
- **Dynamic spoke-to-spoke mapping:** Allows a spoke to encapsulate directly toward another spoke's NBMA address.

The conventional use of static hub mappings does not mean that all remote spoke mappings must be configured manually.

---

## Multicast and Broadcast over mGRE

Routing protocols frequently send control traffic to multicast destinations. For example, EIGRP and OSPF commonly use multicast packets to discover or maintain neighbors.

The ISP does not have to participate in this overlay multicast. A DMVPN router **replicates** the GRE-encapsulated traffic to the appropriate remote NBMA destinations, using unicast packets across the underlay.

The NHRP **multicast mapping list** determines where an mGRE interface sends this replicated traffic.

### At the Hub

```cisco
ip nhrp map multicast dynamic
```

This allows the hub to add dynamically registering spokes to its multicast destination list, rather than configuring every spoke's NBMA address individually.

### At the Spoke

```cisco
ip nhrp map multicast 198.51.100.1
```

This tells Spoke1 to send overlay multicast and broadcast traffic toward Hub1's NBMA address. In a conventional dual-hub, single-cloud setup, a spoke can have multicast mappings to **both** hubs.

**Important:** `ip nhrp map multicast` controls where the router sends **encapsulated overlay multicast/broadcast traffic**. It does not enable IP multicast routing in the ISP or create a multicast distribution tree in the underlay.

This distinction matters when a router can ping a hub's tunnel address but cannot form an EIGRP or OSPF adjacency. Unicast NHRP mappings may be correct while multicast replication is missing.

---

## NHRP Network ID, GRE Tunnel Key, and Authentication

These three settings are easy to confuse because each can appear under a tunnel interface.

| Setting | Purpose | Relationship between peers |
| --- | --- | --- |
| `ip nhrp network-id` | Enables/identifies an NHRP network on the local interface. | **Locally significant**; using the same value throughout a cloud is a practical convention, not an NHRP protocol matching requirement. |
| `tunnel key` | Optional GRE header key used to identify or distinguish tunnel traffic. | When configured, the communicating endpoints must use a compatible **matching GRE key** for that cloud. |
| `ip nhrp authentication` | Optional NHRP authentication string. | Must be consistent between NHRP peers using the same authentication scheme. |

### NHRP Network ID

For example:

```cisco
ip nhrp network-id 100
```

The value `100` is **not** inserted into the GRE header.

Different routers can use different NHRP network IDs and still communicate if the rest of the NHRP/tunnel configuration is compatible. Using consistent IDs makes a lab easier to maintain and troubleshoot.

### GRE Tunnel Key

```cisco
tunnel key 100
```

The GRE key is **not a cryptographic key** and provides no encryption. It can distinguish logical GRE tunnels, especially when multiple DMVPN clouds share the same WAN source address.

For example, the dual-cloud lab can use GRE key `100` in Cloud 1 and key `200` in Cloud 2. Routers in the same cloud must use the matching key when the key is configured.

### NHRP Authentication Versus IPsec Authentication

`ip nhrp authentication` is **not** IKEv2 preshared key authentication. It does not encrypt GRE traffic and is not a replacement for IPsec.

IKEv2 authentication verifies peers as part of establishing IPsec security associations; the optional NHRP authentication setting protects a different protocol function.

---

## Overlay Routing and Data-Plane Forwarding

An NHRP mapping answers **where a tunnel endpoint is located**. A routing protocol answers **which path should carry traffic to a destination prefix**.

Suppose Spoke1 wants to reach Spoke2's internal network, `10.12.12.0/24`.

A Phase 2 route might identify Spoke2's tunnel address as the next hop:

```text
Destination:    10.12.12.0/24
Overlay next hop: Spoke2 tunnel address
Outgoing interface: Tunnel0
```

Spoke1 then needs an NHRP mapping from Spoke2's tunnel address to its NBMA address. The mapping supplies the outer GRE destination.

In Phase 3, the initial route can instead point to Hub1:

```text
Destination:    10.12.12.0/24 (or an appropriate summary)
Overlay next hop: Hub1 tunnel address
Outgoing interface: Tunnel0
```

Spoke1 can initially forward through Hub1. A subsequent NHRP redirect and resolution exchange can establish a **direct shortcut toward Spoke2** without requiring the routing protocol to advertise every remote spoke LAN with that spoke's tunnel address as the next hop.

### Phase 3 and the RIB/FIB

A Phase 3 shortcut can affect forwarding in more than one way:

- NHRP can install a **shortcut route** in the RIB, displayed with the `H` (NHRP) route code.
- NHRP can install a **next-hop override** (NHO) associated with an existing route, changing the effective forwarding next hop without replacing the route's original next hop in the same way.

The resulting path is programmed into the **FIB/CEF** for packet forwarding. Consequently, the routing-protocol next hop alone may not fully describe the path currently used by Phase 3 traffic.

Useful commands include:

```cisco
show ip route nhrp
show ip route next-hop-override
show ip nhrp shortcut
show ip cef
```

This is why **NHRP is not a general-purpose dynamic routing protocol**, even though Phase 3 NHRP can contribute shortcut forwarding information and, in some cases, routes to the RIB. It does not replace EIGRP, OSPF, or BGP for general topology discovery and route advertisement.

---

## Encapsulation and Packet Forwarding

Assume a direct Spoke1-to-Spoke2 path has already been resolved.

```text
Original packet:
  Source:      Host behind Spoke1
  Destination: Host behind Spoke2

GRE transport packet:
  Outer source:      Spoke1 NBMA address
  Outer destination: Spoke2 NBMA address
  GRE payload:       Original packet
```

The forwarding process is:

1. Spoke1 receives a packet destined for Spoke2's internal network.
2. The overlay forwarding information selects the relevant tunnel next hop or NHRP shortcut.
3. The associated NHRP mapping supplies Spoke2's NBMA address.
4. Spoke1 adds GRE and an outer IP header addressed to Spoke2's NBMA address.
5. If configured, IPsec protects the GRE traffic for that peer.
6. The ISP forwards the outer packet based on the **NBMA destination**, without examining the original LAN destination.
7. Spoke2 removes the applicable protection and tunnel encapsulation, then forwards the original packet toward its LAN.

For a direct path, **Hub1 and Hub2 are not data-plane transit hops**, even though a hub may have participated in discovering the path.

For a hub-routed path, packets are encapsulated toward the hub first and then forwarded onward through another GRE encapsulation toward the remote spoke. With IPsec, the hub may need to decrypt and re-encrypt traffic on the two tunnel legs.

A resolved NHRP mapping does not guarantee that traffic is successfully flowing. Underlay filtering, the IPsec state, CEF adjacency, or return routing can still prevent end-to-end connectivity.

---

## How the Three Phases Change the Architecture

The core components are similar across all three phases, but the forwarding decision changes.

| Behavior | Phase 1 | Phase 2 | Phase 3 |
| --- | --- | --- | --- |
| Hub interface | mGRE | mGRE | mGRE |
| Spoke interface | Point-to-point GRE | mGRE | mGRE |
| Spoke registration with NHS | Yes | Yes | Yes |
| Spoke-to-spoke forwarding | Via hub | Direct when next hop resolves to remote spoke | Initially via hub when necessary, then direct after shortcut resolution |
| Routing next hop needed for direct path | Not applicable | Normally remote spoke's tunnel IP | Can initially be hub's tunnel IP |
| Hub summarization while retaining dynamic shortcuts | Not applicable | Constrained | Supported |
| Phase 3 redirect/shortcut | No | No | Yes |

The most important difference between **Phase 2 and Phase 3** is not whether both can create direct tunnels. **Both can.** The difference is **how a spoke discovers and installs the forwarding information needed for the direct path**.

In Phase 2, direct forwarding depends heavily on the routing protocol preserving the remote spoke as next hop. In Phase 3, the hub can remain the routing next hop and use NHRP redirects to initiate the shortcut process.

Detailed packet exchanges and configurations are covered in the individual phase pages.

---

## Dual-Hub Architecture: Single Cloud and Dual Cloud

Both designs use the same six physical routers. The difference is the **logical overlay arrangement**.

### Dual-Hub, Single-Cloud Design

```text
                     DMVPN Cloud 1
                +-------------------+
                |   Hub1    Hub2    |
                |     \      /      |
                |   Spoke1 Spoke2   |
                |       Spoke3      |
                +-------------------+
```

All five DMVPN routers participate in **one overlay subnet**. Each spoke uses one mGRE interface and can register with **both hubs as NHSs**.

For example, the spoke's NHRP configuration can include mappings and NHS statements for both hub tunnel addresses. The routing protocol determines which hub is preferred for a particular destination; simply defining two NHSs does not dictate the forwarding preference.

The single-cloud design minimizes the number of tunnel interfaces but gives less direct per-cloud control over routing behavior.

### Dual-Hub, Dual-Cloud Design

```text
                  Cloud 1              Cloud 2
                 (Tunnel0)            (Tunnel1)
                    Hub1                 Hub2
                     |                    |
              +------+------+      +------+------+ 
              |      |      |      |      |      |
             Sp1    Sp2    Sp3    Sp1    Sp2    Sp3
```

Each spoke has **two logical tunnel interfaces**:

- `Tunnel0` belongs to Cloud 1 and registers with Hub1.
- `Tunnel1` belongs to Cloud 2 and registers with Hub2.

The clouds have separate overlay addressing and normally distinct GRE keys. The same physical WAN interface can be used as the `tunnel source` for both tunnels.

Routing metrics or policy can prefer Cloud 1, prefer Cloud 2, or use both where appropriate. Each cloud has its own NHRP registration and mapping state. When both mGRE tunnels share the same tunnel source, pay attention to the `shared` option on tunnel IPsec protection when required by the platform/design.

### Redundancy Does Not Eliminate Underlay Dependencies

Two hubs do not mean two independent Internet connections. In this lab, both DMVPN clouds use the **same ISP router**.

A hub failure can be recovered through the surviving hub if NHRP registration, routing, and (when enabled) IPsec have the necessary connectivity and converge. Failure of the shared ISP router, however, removes reachability to both hubs and both clouds.

**Dual NHS registration, routing-path selection, and actual data-plane reachability are separate things to verify.** A spoke may remain registered with two NHSs but still select only one hub for a particular route.

---

## Optional IPsec Protection

GRE and NHRP provide tunneling and endpoint discovery, **not encryption**.

DMVPN can be protected by applying an **IPsec profile** to the tunnel interface. With IKEv2 and preshared keys:

1. IKEv2 authenticates the peers and negotiates the cryptographic parameters.
2. IPsec creates the security associations used to protect GRE packets.
3. The mGRE/NHRP mechanisms still determine the logical peer and destination NBMA address.

In a Phase 3 spoke-to-spoke design, the two spokes can establish their own IPsec protection for direct traffic, without relaying that traffic through the hub.

IPsec introduces additional considerations: authentication failures, IKEv2/IPsec SA state, permitted underlay protocols, NAT traversal where needed, MTU overhead, and anti-replay processing. These are covered in the [IPsec/IKEv2 section](phase3/ipsec/overview.md).

The key architectural distinction is:

```text
Routing:  Where should the inner packet go?
NHRP:     Which NBMA destination reaches that overlay next hop?
mGRE:     How is the packet tunneled to that destination?
IPsec:    How is the tunneled traffic protected?
```

---

## Architectural Troubleshooting Checklist

When a DMVPN path does not work, identify **which layer is failing** before changing configurations.

| Symptom | Likely area to investigate | Useful verification |
| --- | --- | --- |
| Remote WAN address is unreachable | Underlay routing, interface state, filtering | `show ip route`, `ping ... source Ethernet0/0` |
| Spoke cannot register with hub | NHS tunnel address, static hub mapping, tunnel/NHRP settings | `show ip nhrp nhs`, `show ip nhrp`, `show dmvpn` |
| Hub cannot send overlay multicast to spokes | NHRP multicast destination list | `show ip nhrp`, tunnel configuration |
| Overlay routing neighbors are absent | Multicast mappings, routing-protocol configuration, tunnel reachability | Routing-protocol neighbor commands, `show dmvpn` |
| Remote LAN prefix is not reachable | Overlay routes, selected next hop, NHRP/CEF resolution | `show ip route`, `show ip nhrp`, `show ip cef` |
| Phase 3 traffic always passes through hub | NHRP redirect/shortcut operation, shortcut forwarding state | `show ip nhrp shortcut`, `show ip route nhrp`, `show ip route next-hop-override` |
| Tunnel works without IPsec but fails with it | IKEv2 identities/PSKs, IPsec profile, SA negotiation, filtering | `show crypto ikev2 sa`, `show crypto ipsec sa` |
| Traffic fails after a hub outage | Surviving NHS, overlay route convergence, shortcut state, IPsec | `show ip nhrp nhs`, `show ip route`, `show dmvpn` |

The commands are diagnostic starting points rather than a complete troubleshooting procedure. Later pages examine the expected outputs and failure cases in detail.

---

## Key Points

- DMVPN separates **underlay reachability**, **GRE encapsulation**, **NHRP endpoint discovery**, **overlay routing**, and **optional IPsec protection**.
- A **multipoint GRE interface** can send traffic to multiple NBMA destinations without configuring a different tunnel interface for each remote peer.
- **NHRP registers and resolves overlay-to-NBMA reachability**; it does not replace a general dynamic routing protocol.
- The **NHS** is commonly the hub, and **NHCs** are commonly the spokes.
- A spoke normally uses a **static mapping to reach its hub**, while the hub learns **dynamic spoke registrations**.
- `ip nhrp map multicast dynamic` lets a hub replicate overlay multicast/broadcast traffic to dynamically registered spokes.
- The **NHRP network ID is locally significant**; a configured **GRE tunnel key must match** between communicating peers in the same cloud.
- The GRE tunnel key is **not a cryptographic key**, and NHRP authentication is **not IKEv2 authentication**.
- Phase 2 usually requires routes whose next hops identify remote spokes; Phase 3 can start with routes pointing to a hub and establish direct shortcuts afterward.
- Phase 3 NHRP can install **RIB routes** or **next-hop overrides**, which affect CEF forwarding without making NHRP a general routing protocol.
- Dual-hub **single-cloud** DMVPN uses one spoke tunnel per cloud; **dual-cloud** DMVPN uses a separate spoke tunnel per cloud.
- A working NHRP registration does not prove that routing, shortcuts, IPsec, or end-to-end traffic forwarding are working.

---

## References

- [Cisco: Dynamic Multipoint VPN](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-conn-dmvpn-dmvpn-0.html)
- [Cisco: Configuring NHRP](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-addressing/ip-addressing/m_config-nhrp-0.html)
- [Cisco: Troubleshoot DMVPN Issues](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/111976-dmvpn-troubleshoot-00.html)
- [Cisco: Shortcut Switching Enhancements for NHRP in DMVPN Networks](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-addressing/ip-addressing/m_nhrp-switch-enhancemts-dmvpn-0.html)
- [Cisco: Dynamic Multipoint IPsec VPNs](https://www.cisco.com/c/en/us/support/docs/security-vpn/ipsec-negotiation-ike-protocols/41940-dmvpn.html)
