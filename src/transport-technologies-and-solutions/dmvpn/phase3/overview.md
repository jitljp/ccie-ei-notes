# 3.3.a Troubleshoot DMVPN Phase 3 with Dual Hub

**DMVPN Phase 3** uses **multipoint GRE (mGRE) on hubs and spokes**, with **NHRP Redirect** and **NHRP Shortcut** to establish direct spoke-to-spoke forwarding. Unlike Phase 2, Phase 3 does **not** require an overlay routing protocol to advertise the originating spoke as the next hop. Spokes can initially forward traffic through a hub, then use an NHRP shortcut to bypass that hub.

This section focuses on **troubleshooting Phase 3 with dual hubs**, including **NHRP**, **routing and forwarding**, and **IPsec/IKEv2 with preshared-key authentication**. It builds on [DMVPN Phase 1](../phase1.md), [DMVPN Phase 2](../phase2.md), and [NHRP Fundamentals](../nhrp-fundamentals.md).

The following pages introduce the mechanisms individually before combining them into a complete dual-hub lab.

---

## Phase 3 Architecture

In Phase 3, the hubs continue to serve as **NHRP Next Hop Servers (NHSs)**, and spokes continue to register their tunnel-to-NBMA address mappings. The hubs exchange overlay routing information with the spokes and can initially forward inter-spoke traffic.

The key addition is that a hub can notify a source spoke when it forwards traffic between spokes on the same DMVPN cloud. The source spoke can then resolve a more direct path and install NHRP shortcut forwarding information.

```text
                    Phase 3 DMVPN overlay

                       Hub1       Hub2
                       (NHS)     (NHS)
                         |         |
                         +----+----+
                              |
                       DMVPN cloud(s)
                        /    |    \
                       /     |     \
                   Spoke1  Spoke2  Spoke3
                    (NHC)   (NHC)   (NHC)
                       \______/
                     Direct path after
                     NHRP shortcut setup
```

**Direct spoke-to-spoke forwarding does not require spoke-to-spoke routing-protocol neighborships.** In the examples, spokes form EIGRP, OSPF, or BGP relationships with the hubs; NHRP handles discovery of direct GRE endpoints.

### How Phase 3 Differs from Phases 1 and 2

| Characteristic | Phase 1 | Phase 2 | Phase 3 |
| --- | --- | --- | --- |
| Hub GRE tunnel | mGRE | mGRE | mGRE |
| Spoke GRE tunnel | Point-to-point GRE | mGRE | mGRE |
| Spoke-to-spoke forwarding | Through hub | Direct after NHRP resolution | Initially via hub; direct after redirect/shortcut resolution |
| Route next hop for a remote spoke LAN | Hub | Destination spoke for direct forwarding | Can remain hub |
| NHRP mechanism for direct forwarding | Not supported | Resolution of spoke tunnel next hop | Redirect, resolution, and shortcut |
| Hub summarization of spoke prefixes | Supported | Generally prevents direct forwarding for summarized routes | Supported alongside direct forwarding |
| Hub `ip nhrp redirect` | No | No | Yes |
| Spoke `ip nhrp shortcut` | No | No | Yes |

The **fundamental Phase 3 improvement** is the separation of the **routing protocol's next hop** from the **optimized forwarding path**. A spoke may retain an EIGRP or OSPF route through the hub while NHRP provides a more direct forwarding entry for traffic to a remote spoke's network.

---

## Reference Lab Topology

The chapter uses the same physical topology as the Phase 1 and Phase 2 pages. Both hubs and all three spokes have independent WAN connections to the ISP.

```text
             Hub1                       Hub2
               \                         /
                \                       /
                 +-------- ISP --------+
                          / | \
                         /  |  \
                    Spoke1 Spoke2 Spoke3
```

| Router | WAN / NBMA address | Tunnel0 in the single-cloud design | Loopback0 (simulated LAN) |
| --- | --- | --- | --- |
| Hub1 | `198.51.100.1/30` | `172.16.0.1/24` | `10.1.1.1/24` |
| Hub2 | `198.51.100.5/30` | `172.16.0.2/24` | `10.2.2.2/24` |
| Spoke1 | `198.51.100.9/30` | `172.16.0.11/24` | `10.11.11.11/24` |
| Spoke2 | `198.51.100.13/30` | `172.16.0.12/24` | `10.12.12.12/24` |
| Spoke3 | `198.51.100.17/30` | `172.16.0.13/24` | `10.13.13.13/24` |

The ISP provides **underlay reachability between NBMA addresses**, but it does not participate in NHRP or learn the overlay LAN routes. As in the earlier labs, every DMVPN router uses a route toward the ISP for its remote NBMA destinations.

For the **dual-cloud** lab, a second tunnel network, `172.16.1.0/24`, is introduced with a separate GRE tunnel key. The physical WAN addressing remains unchanged.

---

## NHRP Redirect and Shortcut Operation

Consider Spoke1 sending traffic from `10.11.11.0/24` to Spoke2's `10.12.12.0/24` network. Assume the initial overlay route on Spoke1 points to **Hub1**.

### Before a Shortcut Exists

```text
                   Initial forwarding path

         Spoke1 --------> Hub1 --------> Spoke2
       10.11.11.0/24                  10.12.12.0/24
                  Hub forwards the packet
                  and signals a better path
```

1. **Spoke1 performs a routing lookup.** Its route toward `10.12.12.0/24` uses Hub1 (`172.16.0.1`) as the next hop. This could be an individual route, a summary route, or a suitable default route.
2. **Spoke1 sends the packet to Hub1.** The source uses the existing NHRP mapping for Hub1's tunnel address to select Hub1's NBMA address, `198.51.100.1`.
3. **Hub1 forwards toward Spoke2.** The packet enters and leaves the hub's DMVPN tunnel interface. Hub1 also generates an **NHRP Redirect** toward Spoke1 to indicate that a shorter path may be available.
4. **Spoke1 sends an NHRP Resolution Request.** The request is associated with the remote destination being reached, rather than merely resolving the existing hub next hop. Hub1 forwards the resolution request toward Spoke2.
5. **Spoke2 sends an NHRP Resolution Reply**, normally directly to Spoke1. The reply supplies Spoke2's NBMA reachability information and can identify a destination prefix for which the mapping is valid.
6. **Spoke1 installs shortcut forwarding information.** Depending on the implementation and the resolved prefix, this can appear as an NHRP route or a next-hop override, together with an NHRP mapping and corresponding CEF forwarding state.

### After the Shortcut Is Installed

```text
                    Hub1
                     ^
                     |  Original routing relationship
                     |
                  Spoke1 ==================> Spoke2
                         Direct GRE packets

        Outer GRE source:      198.51.100.9  (Spoke1)
        Outer GRE destination: 198.51.100.13 (Spoke2)
```

Subsequent matching traffic can bypass Hub1 and travel directly between the two spoke NBMA addresses. **The original routing-protocol route can still point to Hub1**; NHRP shortcut forwarding overrides the path used for the resolved destination.

An NHRP Redirect is **not an ICMP Redirect**. It is an NHRP control-plane message specific to Phase 3 shortcut discovery. The first packets may traverse the hub, and direct forwarding is not guaranteed until resolution and any required IPsec negotiation succeed.

For a full examination of redirects, resolution targets, prefix matching, shortcut state, and CEF updates, see [Phase 3 Operation and Packet Forwarding](operation.md) and [NHRP Redirects and Shortcuts](nhrp/redirect-shortcut.md).

---

## Essential Phase 3 Commands

Phase 3 adds two critical NHRP commands to the mGRE/NHRP foundation already used in Phase 2.

### On the Hub

```cisco
interface Tunnel0
 ip nhrp redirect
```

`ip nhrp redirect` enables the hub to notify a source spoke when it detects traffic that can potentially take a more direct path.

The hub also requires ordinary NHRP configuration, such as `ip nhrp network-id`, and appropriate NHRP multicast replication when running multicast routing protocols:

```cisco
interface Tunnel0
 ip nhrp authentication NHRP100
 ip nhrp map multicast dynamic
 ip nhrp network-id 100
 ip nhrp redirect
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
```

### On a Spoke

```cisco
interface Tunnel0
 ip nhrp shortcut
```

`ip nhrp shortcut` allows the spoke to use NHRP redirect and resolution information to install optimized forwarding paths.

> This command is enabled by default in modern IOS XE, so it does not have to be manually configured, but I am showing it here for clarity.

For example, Spoke1's NHRP/mGRE settings for a **single-hub** Phase 3 cloud can be expressed as:

```cisco
interface Tunnel0
 ip nhrp authentication NHRP100
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
 ip nhrp shortcut
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
```

The consolidated `ip nhrp nhs ... nbma ... multicast` syntax identifies Hub1's **tunnel IP**, its **NBMA IP**, and the **multicast replication destination**. The `multicast` option is needed when the spoke must send multicast routing traffic, such as EIGRP or OSPF hellos, toward the hub. It is not inherently required by unicast-only BGP.

These are **illustrative interface excerpts**, not the complete Phase 3 dual-hub configuration. Tunnel addresses, routing, the second NHS or tunnel, and IPsec are introduced in the dedicated configuration pages.

---

## Routing over Phase 3

In Phase 2, the routing protocol must generally retain the **destination spoke's tunnel address** as the next hop for direct spoke-to-spoke forwarding. Phase 3 removes this restriction by using NHRP shortcuts.

### EIGRP

- The hub can keep the normal **next-hop-self** behavior. `no ip next-hop-self eigrp 100`, which is important for Phase 2 direct forwarding, is **not required** for Phase 3.
- When the hub must advertise spoke-learned EIGRP routes back out the same tunnel interface, configure **`no ip split-horizon eigrp 100`**.
- The hub can advertise **summary routes or a default route** to spokes. NHRP shortcuts can still provide direct forwarding toward destination spoke networks included in those routes.
- If the hub advertises only locally originated aggregates or a default instead of re-advertising individual spoke routes, the split-horizon exception may not be needed for that specific advertisement behavior.

### OSPF

**Point-to-multipoint** is commonly used with Phase 3:

```cisco
interface Tunnel0
 ip ospf network point-to-multipoint
```

Point-to-multipoint models the hub-to-spoke adjacencies without a DR/BDR election. A spoke can initially calculate OSPF paths through a hub; NHRP redirects and shortcuts provide the direct forwarding optimization. This contrasts with the **broadcast** network type commonly used in Phase 2 to obtain remote spoke next hops.

### BGP

Both eBGP and iBGP with route reflection can be used. BGP NEXT_HOP handling must be considered separately from the NHRP shortcut path. In Phase 3, the hub can be the advertised BGP next hop for remote spoke routes while NHRP establishes direct forwarding for eligible destinations.

See [Routing over DMVPN Phase 3](routing.md) for protocol-specific configuration, summarization, and verification.

---

## Dual-Hub Phase 3 Designs

Two hubs improve availability, but **configuring two NHSs does not by itself guarantee routing failover or successful direct spoke-to-spoke forwarding**. Registration, routing, underlay reachability, and security state must all be considered.

| Feature | Single-cloud dual hub | Dual-cloud dual hub |
| --- | --- | --- |
| DMVPN overlay subnets | One shared tunnel subnet | Separate tunnel subnet for each cloud |
| Spoke tunnel interfaces | One mGRE tunnel | Two mGRE tunnels |
| Hub1 / Hub2 tunnel IPs | `172.16.0.1` / `172.16.0.2` | Example: Hub1 `172.16.0.1`, Hub2 `172.16.1.2` |
| Spoke NHS configuration | Two NHSs on the same tunnel | An NHS in each corresponding cloud |
| GRE tunnel keys | Shared key for the cloud | Typically distinct keys per cloud |
| Routing and failover | Select preferred hub using routing policy and metrics | Select preferred cloud/hub using routing and tunnel reachability |

### Single-Cloud Dual Hub

All participants use the **same overlay subnet and mGRE cloud**. Each spoke can register independently with Hub1 and Hub2. For example, a spoke identifies both NHSs:

```cisco
interface Tunnel0
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
 ip nhrp nhs 172.16.0.2 nbma 198.51.100.5 multicast
 ip nhrp shortcut
```

Both hubs must support the required Phase 3 NHRP behavior. The routing design determines which hub is used for initial forwarding, and NHRP must resolve destinations successfully when a shortcut is requested. **Each NHS's registration state is independent**; a successful registration to one hub does not prove that registration to the other has succeeded.

### Dual-Cloud Dual Hub

Each spoke has a **separate mGRE tunnel interface for each cloud**. For this chapter, the clouds use `172.16.0.0/24` and `172.16.1.0/24`, with separate tunnel keys. The spoke can use Hub1 through one tunnel and Hub2 through the other.

This makes the overlay paths and routing choices distinct, but introduces separate NHRP mappings, routing neighbors, and potentially IPsec SAs for each cloud. A failure in one cloud does not automatically prove the other is ready to carry traffic.

Details are covered in [Dual-Hub DMVPN Topologies](dual-hub/topologies.md), [Dual-Hub Phase 3 Configuration](dual-hub/configuration.md), and [Dual-Hub Failover and Convergence](dual-hub/failover.md).

---

## IPsec/IKEv2 with Preshared Keys

**GRE and NHRP do not encrypt traffic.** In a protected DMVPN deployment, an IPsec profile is applied to the mGRE tunnel, and IKEv2 authenticates peers and negotiates the SAs needed for IPsec ESP protection.

The Phase 3 security section uses **IKEv2 preshared-key authentication** and covers:

1. **IKEv2 negotiation:** proposals, policies, keyrings, profiles, identities, and preshared-key matching.
2. **IPsec ESP:** transform sets, IPsec profiles, protection of GRE traffic, SA lifetimes, and encapsulation overhead.
3. **Hub-to-spoke and spoke-to-spoke security:** a direct GRE shortcut also needs a usable protected path between the two spoke NBMA endpoints.
4. **Troubleshooting:** identifying whether a failure is in IKEv2 authentication, CHILD_SA/IPsec negotiation, NHRP, overlay routing, or forwarding.

IKEv2/IPsec failures can occur independently of NHRP and routing failures. **A working route and NHRP mapping do not prove that a protected spoke-to-spoke path can pass traffic.**

Start with [IPsec/IKEv2 Using Preshared Key](ipsec/overview.md).

---

## A Layered Troubleshooting Approach

When a Phase 3 lab fails, verify the network in the order that traffic depends on it. The presence of a tunnel interface or an EIGRP/OSPF/BGP neighbor is not sufficient proof of direct shortcut forwarding.

| Layer | What must work | Useful commands |
| --- | --- | --- |
| **1. Underlay** | NBMA addresses are reachable through the ISP without recursive tunnel routing | `show ip route`, `ping <nbma> source Ethernet0/0`, `traceroute <nbma>` |
| **2. IKEv2/IPsec** | Protected peers negotiate IKEv2 and IPsec SAs, when tunnel protection is configured | `show crypto ikev2 sa`, `show crypto ipsec sa`, `show crypto session detail` |
| **3. mGRE/NHRP registration** | Spokes register with the intended NHS or NHSs; GRE/NHRP parameters agree | `show dmvpn detail`, `show ip nhrp`, `show ip nhrp nhs detail` |
| **4. Overlay routing** | Spokes have usable remote LAN routes and correct initial hub preference | `show ip route`, `show ip eigrp neighbors`, `show ip ospf neighbor`, `show bgp ipv4 unicast summary` |
| **5. NHRP redirect** | The forwarding hub recognizes an eligible suboptimal path and sends a redirect | `show ip nhrp redirect`, `show ip nhrp traffic` |
| **6. NHRP resolution / shortcut** | The source learns the remote spoke NBMA address and installs shortcut forwarding state | `show ip nhrp`, `show ip nhrp shortcut`, `show ip route nhrp`, `show ip route next-hop-override` |
| **7. Data forwarding** | The outer destination changes to the remote spoke NBMA address and traffic succeeds in both directions | `show ip cef`, `show dmvpn`, sourced `ping` and `traceroute`, packet capture |

Not every platform displays shortcut information in the same way. Compare the routing table, NHRP cache, forwarding state, and actual packet path rather than relying on a single command.

### Common Phase 3 Failure Patterns

| Symptom | Area to investigate |
| --- | --- |
| Tunnel shows `up/up` but a spoke is missing from the hub | NHS mapping, NHRP authentication, registration, transport or IPsec reachability |
| A spoke registers with Hub1 but not Hub2 | Second NHS configuration, Hub2 underlay reachability, protection and NHRP state |
| Spoke can reach the hub but not other spoke LANs | Overlay routing, hub forwarding, return routes, split horizon where applicable |
| Spoke-to-spoke traffic always traverses the hub | Missing `ip nhrp redirect` or `ip nhrp shortcut`, redirect generation, unresolved shortcut |
| Redirects are observed but no direct path forms | Resolution Request forwarding, Resolution Reply return path, remote NBMA reachability |
| NHRP shortcut appears but direct traffic fails | GRE or ESP filtering, missing spoke-to-spoke IPsec SA, asymmetric path, MTU/fragmentation |
| Traffic fails after the preferred hub goes down | NHS registration to backup hub, routing convergence, stale NHRP/CEF state, cloud-specific IPsec |
| Small pings work but large packets fail | GRE/IPsec overhead, PMTUD, fragmentation and TCP MSS |

**A useful diagnostic distinction:** failure to **register** is different from failure to **learn a route**, failure to **receive a redirect**, failure to **resolve a shortcut**, and failure to **forward through the shortcut**. Each stage has different evidence and corrective actions.

---

## Phase 3 Troubleshooting Exercises

Use the lab to demonstrate each of the following, rather than checking only that the spoke-to-spoke ping succeeds:

1. **Baseline path:** Confirm that traffic from Spoke1's `10.11.11.0/24` network toward Spoke2's `10.12.12.0/24` network initially has a route through Hub1.
2. **Shortcut establishment:** Generate traffic, inspect NHRP redirects and resolution counters, and confirm Spoke2's NBMA mapping and the direct forwarding path.
3. **Summarization:** Advertise a suitable summary from the hub and confirm that direct spoke-to-spoke forwarding can still be established for a destination covered by it.
4. **Dual NHS:** Register spokes with Hub1 and Hub2; inspect the registration state for each NHS independently.
5. **Failure simulation:** Disable the preferred hub or cloud, observe routing and NHRP convergence, and verify both forward and return paths.
6. **Protected forwarding:** With IPsec enabled, confirm that a direct spoke-to-spoke shortcut has matching IKEv2/IPsec state and passing encrypted traffic counters.

The detailed configurations and expected observations are provided in the subsequent pages.

---

## Chapter Roadmap

The remainder of this section is organized around the **control plane**, **forwarding plane**, **security**, and **dual-hub operations**:

- [Phase 3 Operation and Packet Forwarding](operation.md) examines the initial hub path, redirects, shortcut installation, and packet forwarding changes.
- [3.3.a (i) NHRP](nhrp/overview.md) introduces the NHRP-specific troubleshooting objectives.
  - [NHRP Redirects and Shortcuts](nhrp/redirect-shortcut.md) examines the complete shortcut exchange.
  - [NHRP Verification and Troubleshooting](nhrp/troubleshooting.md) covers NHRP state, counters, and failure scenarios.
- [Routing over DMVPN Phase 3](routing.md) covers EIGRP, OSPF, BGP, and summarization.
- [Dual-Hub DMVPN Topologies](dual-hub/topologies.md) compares single-cloud and dual-cloud arrangements.
- [3.3.a (ii) IPsec/IKEv2 Using Preshared Key](ipsec/overview.md) introduces the security troubleshooting objectives.
  - [IKEv2 Negotiation and Preshared Key Authentication](ipsec/ikev2.md) explains keyrings, profiles, identities, and negotiation.
  - [IPsec ESP and Tunnel Protection](ipsec/esp.md) covers ESP and tunnel protection.
  - [IKEv2/IPsec Configuration for DMVPN](ipsec/configuration.md) builds the secure overlay.
  - [IKEv2/IPsec Verification and Troubleshooting](ipsec/troubleshooting.md) isolates IKEv2 and IPsec issues.
- [Dual-Hub Phase 3 Configuration](dual-hub/configuration.md) provides the combined labs.
- [Dual-Hub Failover and Convergence](dual-hub/failover.md) examines routing and NHRP recovery.
- [End-to-End DMVPN Phase 3 Troubleshooting](troubleshooting.md) integrates every layer into a troubleshooting workflow.

---

## Key Points

- **Phase 3 uses mGRE on both hubs and spokes.** Neither uses a fixed `tunnel destination` for the DMVPN cloud.
- Spokes register with NHSs so that hubs learn their **tunnel-to-NBMA mappings**.
- **`ip nhrp redirect`** is configured on hubs; **`ip nhrp shortcut`** is configured on spokes.
- **Initial spoke-to-spoke traffic can traverse a hub**, which generates an NHRP Redirect to help the source discover a better path.
- An NHRP Resolution Reply from the destination spoke supplies information used to establish **direct spoke-to-spoke forwarding**.
- **The routing-protocol next hop may remain the hub** even after NHRP installs an optimized forwarding path.
- **Hub summarization and default routing are compatible with Phase 3 shortcuts**, unlike the usual direct-forwarding requirements of Phase 2.
- For EIGRP, keep **next-hop-self** on the hub; disable split horizon if it must re-advertise spoke-learned routes out the same tunnel.
- **OSPF point-to-multipoint** is commonly used because Phase 3 does not depend on OSPF calculating a remote spoke next hop.
- In single-cloud dual-hub designs, spokes can configure **multiple NHSs on one tunnel**; in dual-cloud designs, spokes use **separate tunnel interfaces and NHRP contexts**.
- **An NHS registration is not the same as routing convergence or shortcut availability.** Check both hubs and both forwarding directions.
- **GRE and NHRP do not encrypt traffic.** Protected DMVPN requires working IKEv2/IPsec SAs, including for direct spoke-to-spoke paths.
- Troubleshoot **underlay, IKEv2/IPsec, NHRP registration, routing, redirect, resolution, and forwarding** as separate stages.

---

## References

- [Cisco: Troubleshoot DMVPN Issues](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/111976-dmvpn-troubleshoot-00.html)
- [Cisco: Configure BGP over DMVPN Phase 3](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/222585-configure-bgp-over-dmvpn-phase-3.html)
- [Cisco: Shortcut Switching Enhancements for NHRP in DMVPN Networks](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-addressing/ip-addressing/m_nhrp-switch-enhancemts-dmvpn-0.html)
- [Cisco: Spoke-to-Spoke NHRP Summary Maps](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m-sec-conn-dmvpn-summ-maps.html)
- [Cisco: Troubleshoot DMVPN Phase 3 NHRP Redirect Issues](https://www.cisco.com/c/en/us/support/docs/security-vpn/dynamic-multi-point-vpn-dmvpn/218427-troubleshoot-dmvpn-phase3-nhrp-redirect.html)
