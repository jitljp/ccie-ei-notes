# Dynamic Multipoint VPN (DMVPN)

**DMVPN** (Dynamic Multipoint VPN) is a Cisco VPN solution that connects multiple sites across an IP network using GRE tunnels and the **Next Hop Resolution Protocol (NHRP)**.

In a traditional point-to-point GRE design, each tunnel has a configured remote destination. DMVPN uses **multipoint GRE (mGRE)** and NHRP so that routers can discover remote tunnel endpoints dynamically. This allows a hub to support many spokes without requiring a separate tunnel interface for every spoke.

Depending on the DMVPN phase, traffic between two spoke sites either passes through a hub or uses a **direct spoke-to-spoke tunnel**. DMVPN can also be combined with **IPsec** to protect traffic crossing an untrusted underlay.

This chapter introduces DMVPN Phases 1, 2, and 3, then focuses on troubleshooting **Phase 3 with dual hubs**, including NHRP and IPsec/IKEv2 using preshared keys.

---

## Why DMVPN Is Used

Consider a network with a headquarters and many branch sites. If the sites communicate over an IP network such as the Internet, they need a way to exchange private traffic without requiring the provider to know the internal routes.

Static point-to-point GRE tunnels can provide this connectivity, but configuring a separate tunnel interface for every peer becomes difficult as the number of sites increases.

DMVPN addresses this by providing:

- **Scalable hub-and-spoke connectivity:** A hub can communicate with many spokes using a single mGRE tunnel interface.
- **Dynamic spoke-to-spoke connectivity:** In Phases 2 and 3, spokes can send traffic directly to one another without permanently configured tunnel destinations for each peer.
- **Dynamic endpoint discovery:** NHRP associates tunnel addresses with the underlay addresses used to reach the corresponding routers.
- **Support for dynamic routing:** Routing protocols exchange internal routes across the overlay.
- **Redundancy:** Multiple hubs and DMVPN clouds can provide alternative paths.
- **Optional IPsec protection:** GRE traffic can be authenticated and encrypted using IPsec.

DMVPN does not replace the underlying IP network. It builds a logical overlay **on top of the existing underlay**.

---

## Lab Topology

The examples in this chapter use **two hubs, three spokes, and one ISP router**.

```text
              Hub1                 Hub2
                \                   /
                 \                 /
                  +----- ISP ------+
                        / | \
                       /  |  \
                      /   |   \
                 Spoke1 Spoke2 Spoke3
```

The diagram shows the **physical underlay**, not the logical DMVPN tunnels. Each hub and spoke has its own connection to the ISP router.

| Device | Role |
| --- | --- |
| **Hub1** | First DMVPN hub and NHRP next-hop server (NHS). |
| **Hub2** | Second hub for redundancy and dual-hub designs. |
| **Spoke1, Spoke2, Spoke3** | Branch routers participating in the DMVPN overlay. |
| **ISP** | Provides underlay IP connectivity between the hubs and spokes. |

Each DMVPN router has a loopback interface representing a network behind its site. The routers use default routes toward the ISP for underlay reachability; the ISP does not need routes to the internal loopback networks.

The same physical topology is used throughout the chapter. Initial examples use **Hub1 and the three spokes**. Later examples introduce Hub2 and compare single-cloud and dual-cloud designs.

---

## DMVPN Components

DMVPN combines several technologies, each with a different role.

| Component | Purpose |
| --- | --- |
| **GRE** | Encapsulates traffic so it can cross the underlay IP network. |
| **mGRE** | Allows one tunnel interface to send GRE traffic to multiple dynamically selected destinations. |
| **NHRP** | Resolves overlay tunnel addresses to the corresponding underlay addresses. |
| **Hub / NHS** | Receives spoke registrations and assists with NHRP resolution. |
| **Routing protocol** | Advertises internal networks and determines which tunnel next hop to use. |
| **IKEv2** | Authenticates IPsec peers and negotiates the security associations used to protect traffic. |
| **IPsec** | Provides cryptographic protection for GRE traffic when configured. |

### Multipoint GRE

A static point-to-point GRE interface has one configured remote tunnel destination. An **mGRE** interface has no single fixed remote destination. Instead, the router can encapsulate packets toward different remote underlay addresses through the same tunnel interface.

```text
Point-to-point GRE:  One tunnel interface -> One remote endpoint
Multipoint GRE:      One tunnel interface -> Multiple remote endpoints
```

For spoke-to-spoke DMVPN, routers use NHRP mappings to determine where GRE packets should be sent. A dynamically established spoke-to-spoke path does **not** require creating another tunnel interface.

### NHRP

**NHRP** (Next Hop Resolution Protocol) maintains mappings between two types of addresses:

- **Tunnel address:** The router's address in the DMVPN overlay.
- **NBMA address:** The address used to reach the router across the underlay, typically its WAN address.

**NBMA** stands for *Non-Broadcast Multiple Access*.

Spokes register their tunnel-to-NBMA mappings with an NHRP **next-hop server (NHS)**, typically a hub. When a router needs to reach a dynamically selected tunnel peer, NHRP can provide the peer's NBMA address.

NHRP does **not** advertise LAN prefixes or choose the best route to a destination. Those tasks belong to the routing protocol. NHRP supplies the information needed to forward GRE packets to a particular tunnel peer.

---

## DMVPN Phases

DMVPN has three commonly described phases. The main differences concern **spoke-to-spoke forwarding** and the routing information required for that forwarding.

| Characteristic | Phase 1 | Phase 2 | Phase 3 |
| --- | --- | --- | --- |
| Hub tunnel type | mGRE | mGRE | mGRE |
| Spoke tunnel type | Point-to-point GRE | mGRE | mGRE |
| Spoke-to-spoke data path | Through hub | Direct path supported | Direct path supported |
| NHRP registration | Yes | Yes | Yes |
| Direct spoke-to-spoke resolution | No | Yes | Yes, with redirect/shortcut mechanisms |
| Hub route summarization while retaining direct spoke-to-spoke forwarding | Not applicable | Restricted | Supported |

### Phase 1: Hub-and-Spoke Forwarding

In **Phase 1**, spokes use point-to-point GRE tunnels directed at the hub. The hub uses mGRE to communicate with multiple spokes.

Even if a packet originates at one spoke and is destined for another, it must traverse the hub:

```text
Spoke1 -> Hub1 -> Spoke2
```

Phase 1 provides straightforward hub-and-spoke connectivity but does not support direct spoke-to-spoke GRE forwarding.

### Phase 2: Dynamic Spoke-to-Spoke Forwarding

In **Phase 2**, both hubs and spokes use mGRE. Spokes can resolve one another's NBMA addresses using NHRP and then send traffic directly:

```text
Spoke1 =============> Spoke2
         Direct GRE
```

Phase 2 requires suitable routing information at the spokes. For a route to a network behind another spoke to use a direct shortcut, its next hop generally needs to identify the **remote spoke's tunnel address**, rather than the hub's tunnel address.

This imposes restrictions on routing-protocol configuration and hub-to-spoke route summarization.

### Phase 3: NHRP Redirects and Shortcuts

**Phase 3** also supports direct spoke-to-spoke forwarding, but it introduces **NHRP redirects** and **NHRP shortcuts** to reduce the routing restrictions of Phase 2.

For example, a spoke can initially forward traffic toward the hub using a summarized or default route:

```text
Initial path:
Spoke1 -> Hub1 -> Spoke2

After NHRP shortcut resolution:
Spoke1 ============> Spoke2
          Direct GRE
```

At a high level:

1. Spoke1 initially forwards a packet for a network behind Spoke2 toward Hub1.
2. Hub1 forwards the packet toward Spoke2 and sends an **NHRP redirect** to Spoke1, indicating that a more direct path may exist.
3. Spoke1 uses **NHRP resolution** to discover the information needed to reach Spoke2 directly.
4. Spoke1 installs a **shortcut** so subsequent matching traffic can bypass the hub.

The ability to use a summarized or default route toward the hub while still establishing direct spoke-to-spoke forwarding is a major advantage of Phase 3.

---

## Single-Cloud and Dual-Cloud Designs

A **DMVPN cloud** is a logical DMVPN network whose participating tunnel interfaces share an overlay address subnet.

The two-hub topology supports two designs:

| Design | Tunnel arrangement | Main characteristic |
| --- | --- | --- |
| **Dual-hub, single-cloud** | Both hubs and all spokes participate in one DMVPN cloud; each spoke has one mGRE tunnel interface. | Spokes can use both hubs as NHSs within the same overlay. |
| **Dual-hub, dual-cloud** | Each hub belongs to a separate DMVPN cloud; each spoke has a tunnel interface in each cloud. | Separate overlay subnets and interfaces provide more direct control over routing preference and failover. |

A dual-cloud design does **not** require two separate ISP networks. The two logical DMVPN clouds can share the same physical underlay, as they will in this lab.

The dual-hub pages examine hub preference, route selection, spoke-to-spoke forwarding, and failure recovery in both designs.

---

## DMVPN and IPsec

GRE and NHRP do not encrypt user traffic. A DMVPN overlay can operate without IPsec, but traffic crossing an untrusted network is commonly protected with **IPsec**.

In this chapter, IPsec protection uses **IKEv2 with preshared key authentication**:

- **IKEv2** authenticates peers and establishes the security associations and keying material used by IPsec.
- **IPsec ESP** protects GRE packets as they cross the underlay.
- An **IPsec profile** is applied to the DMVPN tunnel interface using tunnel protection.

IPsec protects the traffic, while mGRE and NHRP provide dynamic tunnel endpoint selection. They perform different functions and are troubleshot separately.

---

## Troubleshooting DMVPN

When a DMVPN network fails, isolate the problem by checking its dependencies in order:

1. **Underlay reachability:** Can the routers reach the required remote NBMA addresses through the ISP?
2. **Tunnel and NHRP operation:** Are the tunnel interfaces configured correctly? Have spokes registered with the appropriate NHSs? Are resolution mappings present?
3. **Overlay routing:** Are the required LAN routes installed, and do they select the intended tunnel next hops?
4. **Phase 3 shortcut operation:** Are redirects received and shortcuts established when direct spoke-to-spoke forwarding is expected?
5. **IKEv2/IPsec:** If tunnel protection is configured, do authentication and security association negotiation succeed, and do the IPsec counters increment?
6. **Dual-hub behavior:** Does routing and NHRP connectivity recover correctly when a hub or path fails?

A working tunnel interface by itself does not prove that NHRP, overlay routing, spoke-to-spoke shortcuts, or IPsec are functioning correctly.

---

## Key Points

- DMVPN uses **GRE/mGRE** and **NHRP** to provide scalable connectivity between sites.
- **mGRE** allows one tunnel interface to send traffic toward multiple remote endpoints.
- **NHRP** maps overlay tunnel addresses to underlay NBMA addresses; routing protocols advertise LAN prefixes and determine routes.
- **Phase 1** forwards spoke-to-spoke traffic through the hub.
- **Phase 2** supports direct spoke-to-spoke forwarding but has routing next-hop and summarization restrictions.
- **Phase 3** uses **NHRP redirects and shortcuts**, allowing direct spoke-to-spoke forwarding even when initial routes point toward the hub.
- **Single-cloud** dual-hub designs use one overlay cloud; **dual-cloud** designs give spokes separate tunnel interfaces for the two clouds.
- **IPsec is not inherent to GRE or NHRP**. IKEv2/IPsec can be used to authenticate peers and protect DMVPN traffic.
- DMVPN troubleshooting requires checking the **underlay, NHRP, overlay routing, shortcuts, and IPsec** separately.

---

## References

- [Cisco: Dynamic Multipoint IPsec VPNs (Using Multipoint GRE/NHRP to Scale IPsec VPNs)](https://www.cisco.com/c/en/us/support/docs/security-vpn/ipsec-negotiation-ike-protocols/41940-dmvpn.html)
- [Cisco: Shortcut Switching Enhancements for NHRP in DMVPN Networks](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-addressing/ip-addressing/m_nhrp-switch-enhancemts-dmvpn-0.html)
- [Cisco: Configure BGP over DMVPN Phase 3](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/222585-configure-bgp-over-dmvpn-phase-3.html)
- [Cisco: Sharing IPsec with Tunnel Protection](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_conn_dmvpn/configuration/xe-3s/sec-conn-dmvpn-xe-3s-book/sec-conn-dmvpn-share-ipsec-w-tun-protect.html)
