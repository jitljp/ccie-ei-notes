# Dual-Hub DMVPN Topologies

A **dual-hub DMVPN Phase 3** design connects spokes to two hubs for redundancy. Both hubs can act as **NHRP Next Hop Servers (NHSs)** and participate in overlay routing, but the way they share a DMVPN *cloud* changes NHRP registration, route selection, and spoke-to-spoke forwarding.

There are two principal designs:

1. **Single-cloud dual hub:** One mGRE tunnel and one overlay subnet on each spoke, with **two NHSs** in the same DMVPN cloud.
2. **Dual-cloud dual hub:** Two mGRE tunnels and two overlay subnets on each spoke, with **one NHS per cloud** in the reference topology.

Both designs support **Phase 3 NHRP Redirect and Shortcut**. Dual hubs do not require two separate spoke-to-spoke connections, and having two NHSs does not by itself make both hubs active forwarding paths. **The routing table chooses the initial forwarding path**; NHRP can subsequently optimize that path within the appropriate cloud.

This page covers the **topology and routing design**. Complete device configurations are in [Dual-Hub Phase 3 Configuration](configuration.md), convergence mechanisms in [Dual-Hub Failover and Convergence](failover.md), and encryption in the [IPsec/IKEv2 section](../ipsec/overview.md).

---

## RLab

The examples extend the existing Hub1, Hub2, and three-spoke lab. The ISP provides **underlay reachability only**. It does not carry the enterprise overlay LAN prefixes.

| Device | WAN interface | Underlay address | ISP next hop | LAN or loopback prefix |
| --- | --- | --- | --- | --- |
| Hub1 | `Ethernet0/0` | `198.51.100.1/30` | `198.51.100.2` | `10.1.1.0/24` |
| Hub2 | `Ethernet0/0` | `198.51.100.5/30` | `198.51.100.6` | `10.2.2.0/24` |
| Spoke1 | `Ethernet0/0` | `198.51.100.9/30` | `198.51.100.10` | `10.11.11.0/24` |
| Spoke2 | `Ethernet0/0` | `198.51.100.13/30` | `198.51.100.14` | `10.12.12.0/24` |
| Spoke3 | `Ethernet0/0` | `198.51.100.17/30` | `198.51.100.18` | `10.13.13.0/24` |

The loopbacks use the host addresses established earlier: `10.1.1.1`, `10.2.2.2`, `10.11.11.11`, `10.12.12.12`, and `10.13.13.13`, respectively.

The tunnel addresses depend on which design is chosen. **The two designs below are alternatives, not simultaneous configurations of the same tunnel interfaces.**

---

## What Defines a DMVPN Cloud?

A **DMVPN cloud** is a logical mGRE/NHRP overlay in which its members can dynamically resolve and reach other members. A cloud has:

- An **overlay subnet** shared by participating tunnel interfaces.
- An **NHRP domain** and one or more configured NHSs.
- A consistent GRE tunnel encapsulation design, including the selected tunnel key.
- Routing adjacencies to the appropriate hub or hubs.
- Its own set of dynamic NHRP mappings and spoke-to-spoke shortcut forwarding state.

Do not confuse a cloud with the physical ISP network. The same Internet connection and underlay address can transport packets for more than one DMVPN cloud. Conversely, two hubs may belong to the **same cloud** even though their WAN addresses are on different underlay subnets.

Also distinguish **NHS redundancy** from **routing redundancy**:

| Component | What it provides | What it does not guarantee |
| --- | --- | --- |
| Second NHS | Another server for spoke registration and NHRP resolution | That both hubs advertise usable enterprise routes |
| Second routing neighbor | Another potential overlay route source | That the alternate route wins or converges quickly |
| Second DMVPN cloud | A separate tunnel and NHRP forwarding domain | A physically independent WAN or Internet path |
| NHRP shortcut | Direct forwarding to a destination spoke | Backup routing after the underlying route disappears |

**Two hubs alone do not guarantee end-to-end redundancy.** Both hubs must be able to learn and forward the destinations they are expected to serve, and each spoke must have a valid route when one path fails.

---

## Single-Cloud Dual Hub

In a **single-cloud** design, Hub1, Hub2, and all three spokes share `172.16.0.0/24`. Each device needs only one DMVPN tunnel interface.

```text
                             UNDERLAY / ISP
                        (WAN NBMA connectivity)
                              /          \
                             /            \
                          Hub1           Hub2
                     198.51.100.1   198.51.100.5
                     Tu0 172.16.0.1 Tu0 172.16.0.2
                             \           /
                              \         /
                   +--------------------------------+
                   |  Single DMVPN cloud           |
                   |  172.16.0.0/24                 |
                   |  NHRP NHS: Hub1 and Hub2       |
                   +--------------------------------+
                       /          |          \
                    Spoke1      Spoke2      Spoke3
                   Tu0 .11     Tu0 .12     Tu0 .13
```

The diagram represents **logical reachability**, not a permanent physical full mesh. Each spoke maintains hub relationships; **direct spoke-to-spoke GRE forwarding is created dynamically** when NHRP resolves a shortcut.

### Addressing and Roles

| Router | Tunnel interface | Tunnel IP | NHRP role | GRE key |
| --- | --- | --- | --- | --- |
| Hub1 | `Tunnel0` | `172.16.0.1/24` | NHS | `100` |
| Hub2 | `Tunnel0` | `172.16.0.2/24` | NHS | `100` |
| Spoke1 | `Tunnel0` | `172.16.0.11/24` | NHC, registers with both | `100` |
| Spoke2 | `Tunnel0` | `172.16.0.12/24` | NHC, registers with both | `100` |
| Spoke3 | `Tunnel0` | `172.16.0.13/24` | NHC, registers with both | `100` |

The existing `ip nhrp network-id 100` and NHRP authentication `NHRP100` can be retained throughout this single cloud. The hubs provide **two distinct NHS addresses**:

```text
Hub1: overlay 172.16.0.1 -> NBMA 198.51.100.1
Hub2: overlay 172.16.0.2 -> NBMA 198.51.100.5
```

### Multiple NHSs on One Tunnel

Each spoke configures **both hubs as NHSs on the same tunnel**. These commands illustrate the relationship on Spoke1:

```cisco
interface Tunnel0
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
 ip nhrp nhs 172.16.0.2 nbma 198.51.100.5 multicast
```

- The **NHS overlay address** identifies the server inside the tunnel.
- The **NBMA address** identifies where to send the GRE/NHRP packets over the ISP.
- The `multicast` option provides the corresponding NHRP multicast replication information used by routing-protocol traffic.
- Both servers are on the **same NHRP/tunnel interface**, not separate routing interfaces.

On the hubs, dynamically registered spokes can be used as multicast destinations for the hub's routing-protocol packets. The hubs must also be able to reach each other's tunnel address **if the routing design requires hub-to-hub communication over this cloud**.

An **NHS is not equivalent to the current data-plane next hop**. Spoke1 may be registered with both hubs while its route to `10.12.12.0/24` prefers only Hub1.

### Overlay Routing Adjacencies

A common routing arrangement is:

```text
        Hub1  <---- routing ---->  Spoke1
        Hub2  <---- routing ---->  Spoke1

        Hub1  <---- routing ---->  Spoke2
        Hub2  <---- routing ---->  Spoke2

        Hub1  <---- routing ---->  Spoke3
        Hub2  <---- routing ---->  Spoke3
```

These are **hub-to-spoke adjacencies** over the same spoke tunnel interface. Spoke1 does not need an EIGRP or OSPF adjacency with Spoke2 to forward packets directly after installing an NHRP shortcut.

A hub-to-hub routing adjacency is a **separate design choice**. It may be carried over the DMVPN overlay, a dedicated inter-hub connection, or an enterprise backbone.

### Initial Forwarding and Shortcut Formation

Assume Spoke1 needs to reach Spoke2's LAN `10.12.12.0/24`, and its preferred EIGRP route uses Hub1.

```text
Before NHRP shortcut:

Spoke1 ----> Hub1 ----> Spoke2

After NHRP redirect and resolution:

Spoke1 ----------------> Spoke2
```

1. Spoke1 forwards the first packets to Hub1 using its ordinary route.
2. Hub1 forwards traffic toward Spoke2 and, where the Phase 3 redirect conditions are met, sends an **NHRP Redirect** to Spoke1.
3. Spoke1 resolves the destination through NHRP and learns the destination spoke's NBMA address, `198.51.100.13`.
4. Spoke1 can forward subsequent packets directly to Spoke2, **without Hub1 or Hub2 in the data path**.

If the initial route instead prefers Hub2, the same direct Spoke1-to-Spoke2 outcome is possible.

### Primary and Secondary Hub Selection

The **routing protocol**, rather than the order of the `ip nhrp nhs` commands, determines which hub is the next hop for enterprise traffic. For example, Spoke1 could learn the same destination through both hubs:

```text
Candidate 1: 10.12.12.0/24 via 172.16.0.1 (Hub1)
Candidate 2: 10.12.12.0/24 via 172.16.0.2 (Hub2)
```

The installed route depends on the routing protocol's selection rules and policy. **NHS registration and route preference are separate decisions.**

Because both hub adjacencies share `Tunnel0`, ordinary *interface-level* EIGRP delay, tunnel bandwidth, or OSPF interface-cost changes affect the shared interface, rather than distinguishing one hub from the other. Per-neighbor or per-route mechanisms may be needed to create a deterministic preference.

Possible designs include:

- **Active/standby:** Prefer Hub1 routes, use Hub2 after a failure.
- **Active/active by destination:** Prefer Hub1 for some prefixes and Hub2 for others.
- **Equal-cost routing:** Install both hub next hops where the routing protocol and design support ECMP.

**Active/active hubs do not imply that a packet is forwarded through both hubs.** The FIB selects the path for each packet or flow, and Phase 3 shortcuts can subsequently bypass the hubs.

### Strengths and Limitations

**Advantages**

- One tunnel interface and one overlay subnet per spoke.
- One shared NHRP shortcut domain for direct spoke-to-spoke forwarding.
- Less tunnel-interface and addressing overhead than dual cloud.
- Simple addition of another NHS without creating an entirely separate spoke tunnel.

**Limitations**

- Less granular **per-hub interface-metric control** because both adjacencies use the same tunnel interface.
- Potential ambiguity in routing preference when two hubs advertise identical prefixes with equal metrics.
- NHS registration, routing adjacency, and shortcut state can fail independently, so diagnosing hub selection requires checking all three.
- A single tunnel interface is a shared local logical dependency, even when the hubs themselves are redundant.

---

## Dual-Cloud Dual Hub

In a **dual-cloud** design, Hub1 and Hub2 terminate **different DMVPN clouds**. Each spoke participates in both clouds by using two mGRE tunnel interfaces.

```text
                  +------------+------------+
                  |                         |
                 Hub1                      Hub2
            NBMA 198.51.100.1        NBMA 198.51.100.5
            Tu0 172.16.0.1           Tu1 172.16.1.2
                  |                         |
          +----------------+       +----------------+
          | Cloud 100      |       | Cloud 200      |
          | 172.16.0.0/24  |       | 172.16.1.0/24  |
          | NHS: Hub1      |       | NHS: Hub2      |
          +----------------+       +----------------+
               |   |   |                |   |   |
               |   |   |                |   |   |
             Tu0 Tu0 Tu0              Tu1 Tu1 Tu1
              S1  S2  S3               S1  S2  S3
             .11 .12 .13              .11 .12 .13
```

Spoke1, Spoke2, and Spoke3 each have **two overlay IP addresses**, one in each subnet. They can continue using the same physical WAN interface for both tunnels.

### Addressing and Roles

| Device | Cloud 100 | Cloud 200 |
| --- | --- | --- |
| Hub1 | `Tunnel0`: `172.16.0.1/24` (NHS) | Not a member |
| Hub2 | Not a member | `Tunnel1`: `172.16.1.2/24` (NHS) |
| Spoke1 | `Tunnel0`: `172.16.0.11/24` | `Tunnel1`: `172.16.1.11/24` |
| Spoke2 | `Tunnel0`: `172.16.0.12/24` | `Tunnel1`: `172.16.1.12/24` |
| Spoke3 | `Tunnel0`: `172.16.0.13/24` | `Tunnel1`: `172.16.1.13/24` |

Suggested cloud-specific identifiers:

| Property | Cloud 100 | Cloud 200 |
| --- | --- | --- |
| Overlay subnet | `172.16.0.0/24` | `172.16.1.0/24` |
| NHRP network ID | `100` | `200` |
| GRE tunnel key | `100` | `200` |
| NHRP authentication | `NHRP100` | `NHRP200` |
| Primary NHS in example | `172.16.0.1` | `172.16.1.2` |
| Spoke interface | `Tunnel0` | `Tunnel1` |

The numbers serve different purposes. **NHRP network ID, GRE tunnel key, and overlay subnet are not interchangeable concepts.** Using distinct values here makes the two forwarding domains straightforward to recognize and troubleshoot.

### Separate NHS Relationships

Spoke1's two NHS relationships are configured under **different interfaces**:

```cisco
interface Tunnel0
 ip address 172.16.0.11 255.255.255.0
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
 ! Cloud 100: Hub1
!
interface Tunnel1
 ip address 172.16.1.11 255.255.255.0
 ip nhrp nhs 172.16.1.2 nbma 198.51.100.5 multicast
 ! Cloud 200: Hub2
```

These snippets show only the **NHS relationships and addressing**, not a working tunnel configuration. GRE mode, keys, NHRP domain settings, tunnel sources, routing and, later, IPsec must also be configured. See [Dual-Hub Phase 3 Configuration](configuration.md).

### Separate Routing Adjacencies and Metrics

With EIGRP, Spoke1 can maintain two independent overlay adjacencies:

```text
Spoke1 Tunnel0 (172.16.0.11)  <---->  Hub1 Tunnel0 (172.16.0.1)
Spoke1 Tunnel1 (172.16.1.11)  <---->  Hub2 Tunnel1 (172.16.1.2)
```

Because these are **different tunnel interfaces**, different EIGRP delay values or OSPF interface costs can be applied to prefer one cloud. For example:

```cisco
interface Tunnel0
 delay 1000
!
interface Tunnel1
 delay 2000
```

Routing metric design must also account for what each hub advertises and the potential path back through the network.

The same principle applies to OSPF:

```cisco
interface Tunnel0
 ip ospf cost 10
!
interface Tunnel1
 ip ospf cost 20
```

For Phase 3, **OSPF point-to-multipoint** remains a common choice for the hub-and-spoke tunnel adjacencies. Exact hub advertisement and route-selection behavior depends on the OSPF area layout and LSDB topology.

### Initial Forwarding and Direct Shortcuts

Assume Spoke1 learns `10.12.12.0/24` through both clouds but prefers Hub1's path on `Tunnel0`.

```text
Initial route: Spoke1 Tunnel0 -> Hub1 Tunnel0 -> Spoke2 Tunnel0

Phase 3 shortcut:
Spoke1 Tunnel0 --------------------------> Spoke2 Tunnel0
```

If Hub2's route becomes preferred, traffic can instead follow Cloud 200:

```text
Initial route: Spoke1 Tunnel1 -> Hub2 Tunnel1 -> Spoke2 Tunnel1

Phase 3 shortcut:
Spoke1 Tunnel1 --------------------------> Spoke2 Tunnel1
```

A **Cloud 100 shortcut** resolves the destination spoke's **Cloud 100 tunnel address**, and a **Cloud 200 shortcut** resolves its **Cloud 200 tunnel address**. The two shortcuts are independent NHRP forwarding entries even when they ultimately use the **same spoke NBMA address** (`198.51.100.13` in this lab).

The two tunnels do not cause one packet to traverse two parallel GRE paths automatically. Use routing and, when appropriate, ECMP to decide how traffic is distributed.

### Can Shortcuts Cross Between Clouds?

Do **not** treat the two NHRP clouds as one shared address-resolution database.

For example, if Spoke1 sends traffic on Cloud 100, it cannot simply use Spoke2's **Cloud 200** tunnel address as if it were another endpoint on Cloud 100. A shortcut needs a reachable destination endpoint and valid resolution information **for its own cloud**.

This has an important failure implication: if Spoke2 loses Cloud 100 registration but remains reachable on Cloud 200, Spoke1 needs an **appropriate routing path via Cloud 200**. A stale Cloud 100 shortcut should not be assumed to migrate itself to Cloud 200.

If different spokes belong to different clouds or regions, connectivity between those populations may require **inter-hub routing/transit** or a more advanced hierarchical design. That is not the same thing as a direct same-cloud spoke shortcut.

### Same Tunnel Source on Both Clouds

Both of Spoke1's mGRE tunnels may use `Ethernet0/0` as the tunnel source:

```text
Spoke1 Tunnel0 -> outer source 198.51.100.9
Spoke1 Tunnel1 -> outer source 198.51.100.9
```

The GRE tunnel keys (`100` and `200`) distinguish the GRE tunnels. **When IPsec protection is added, however, sharing a tunnel source can introduce IPsec profile and demultiplexing requirements.** Cisco documents the use of the `shared` option with tunnel protection for appropriate multi-mGRE designs. This will be covered in the [IPsec/IKEv2 configuration](../ipsec/configuration.md) rather than mixed into this unencrypted topology discussion.

A second DMVPN cloud is therefore **not necessarily a second physical Internet link**. If Spoke1 has only one WAN interface or one ISP access circuit, failure of that access may take down both clouds simultaneously.

### Strengths and Limitations

**Advantages**

- Separate tunnel interfaces support **per-cloud interface costs, delay, bandwidth, QoS, and policy**.
- Clear separation of NHRP registrations and shortcut entries.
- Straightforward primary/backup or destination-based path preference using routing metrics.
- More control over how traffic uses the two hubs when both are healthy.

**Limitations**

- Two overlay address spaces and two mGRE tunnels per spoke.
- More NHRP, routing, GRE, and potentially IPsec state.
- Multiple direct spoke-to-spoke shortcuts may be established independently in the two clouds.
- Additional care is required when the tunnels share the same WAN source and use IPsec.
- Routing must account for a remote spoke becoming reachable in one cloud but not the other.

---

## Single Cloud Versus Dual Cloud

| Design aspect | Single cloud | Dual cloud |
| --- | --- | --- |
| Spoke tunnel interfaces | One | Two |
| Overlay subnets | One | Two |
| NHSs per spoke in reference design | Two on `Tunnel0` | One on each of `Tunnel0` and `Tunnel1` |
| Hub tunnel interfaces | One per hub | One per hub |
| NHRP shortcut domain | Shared | Separate per cloud |
| Per-hub interface metric control | Limited by the shared spoke interface | Independent metrics on the two spoke tunnels |
| Spoke-to-spoke forwarding | Direct via the shared cloud | Direct within the selected cloud |
| IPsec state | One cloud's set of peers/SAs | Separate cloud relationships; often greater state |
| Best fit | Simpler redundancy with shared overlay | Greater path-control and fault-domain separation |

**Do not interpret “single cloud” as “single hub.”** The first design has two hubs, both participating in one logical tunnel network. Similarly, **dual cloud does not imply two ISPs**.

---

## Routing and Control-Plane Design Considerations

### 1. Two NHSs Do Not Automatically Mean Two Best Routes

A successful NHRP registration confirms an NHS has a mapping for the spoke. The routing protocol independently determines which hub has an eligible route for a destination.

For a dual-hub design, verify:

1. The spoke has a registration relationship with each intended NHS.
2. Both hubs have a valid route to the spoke's LAN prefixes.
3. Each spoke learns usable remote-site routes from the intended hub(s).
4. The preferred route uses the intended hub/cloud.
5. A fallback route is available when the preferred hub or cloud becomes unusable.

**A working NHS relationship without an alternate routed path is not a complete failover solution.**

### 2. The Hub Is the Initial Next Hop, Not Necessarily the Final Data Path

In Phase 3, hub next-hop-self behavior and hub-originated summary routes are compatible with direct spoke-to-spoke forwarding. When the source initially forwards traffic through a hub, the hub can send an NHRP Redirect. The resulting shortcut may be represented as:

- A more-specific **`H` route** when NHRP installs a prefix more specific than the existing covering route.
- An **NHRP next-hop override** (`%` / `[NHO]`) against an existing matching route, when applicable.

The **routing-protocol route**, **NHRP shortcut**, and **CEF next hop** are distinct pieces of state. A traceroute and CEF lookup establish which path packets actually take.

For complete prefix-selection and Route Watch behavior, see [NHRP Redirects and Shortcuts](../nhrp/redirect-shortcut.md).

### 3. Summarization Requires a Valid Path Behind Both Hubs

Both hubs can advertise summary routes toward the spokes in Phase 3, but a summary is only useful if the hub advertising it can forward the covered destinations.

Example:

```text
Hub1 advertises 10.0.0.0/8 toward spokes.
Hub2 advertises 10.0.0.0/8 toward spokes.
```

Both hubs must have routes to the actual LAN prefixes for the traffic they receive. If Hub2 advertises the summary but has **no way to reach Spoke2's `10.12.12.0/24`**, selecting Hub2 can result in blackholing instead of successful NHRP resolution.

A hub should also avoid using a summary route that resolves back into its own DMVPN interface as a substitute for real reachability to a destination, particularly when some spokes are registered only through the other hub.

### 4. Hub-to-Hub Connectivity Is a Design Decision

**Both hubs being members of one cloud does not imply an automatic routing relationship between them.** The same is true when the hubs terminate separate clouds.

An enterprise network may connect the hubs through a backbone, dedicated inter-hub link, or routed overlay. Consider the case where:

- Spoke2 can currently reach **only Hub2**.
- Spoke1 prefers a route toward **Hub1**.

If Hub1 has no valid path to Spoke2, it cannot forward the first packet merely because both hubs exist. A **hub-to-hub transit path**, changed route advertisements, or a route directly through Hub2 may be needed to restore connectivity.

The more the design relies on *partial* spoke attachment to each hub, the more important inter-hub reachability becomes.

### 5. Active/Active and Asymmetric Routing

**Active/active** means both hubs are eligible to carry traffic. It does not guarantee symmetric forwarding or equal utilization.

A source spoke might send traffic toward Hub1 while the destination spoke's return route prefers Hub2. NHRP shortcuts can also form **independently in each direction**. This is not automatically a problem for IP forwarding, but asymmetric paths can matter for stateful firewalls, tunnel/IPsec state, path monitoring, and traffic engineering.

For troubleshooting, always inspect the **forward and reverse routes**, not only the route at the source spoke.

### 6. Underlay Independence Must Be Tested Separately

Two different NHSs can still share:

- The same spoke WAN link and default route.
- The same ISP or intermediate transport network.
- The same upstream firewall or NAT device.
- The same central site or power infrastructure.

Thus **hub redundancy is not the same as transport diversity**. A dual-cloud topology using one underlay source may still fail both tunnels when that underlay fails.

---

## Operational State to Expect

In a dual-hub lab, an engineer should be able to distinguish these states:

| State | Single cloud | Dual cloud |
| --- | --- | --- |
| NHRP server configuration | Two NHS addresses on `Tunnel0` | NHS on `Tunnel0` and NHS on `Tunnel1` |
| NHRP registration | Registrations associated with two NHSs | Independent registrations in each cloud |
| Routing neighbors | Two hub neighbors reachable through `Tunnel0` | One hub neighbor per tunnel in reference design |
| Base route | May select Hub1, Hub2, or ECMP on `Tunnel0` | May select `Tunnel0`, `Tunnel1`, or ECMP |
| Shortcut entry | Direct spoke mapping in one cloud | Shortcut state within the selected cloud |
| Expected direct GRE outer destination to Spoke2 | `198.51.100.13` | `198.51.100.13` in this single-WAN lab, regardless of chosen cloud |
| GRE demultiplexing | Same cloud key | Separate keys (`100` / `200`) |

**The key questions are different at each layer:**

- **NHRP:** Is the spoke registered with the intended NHS, and is the destination's NBMA mapping known?
- **Routing:** Which hub or tunnel wins for the destination, and is a usable backup installed or available?
- **Forwarding:** Does CEF use the hub or a direct spoke next hop after shortcut installation?
- **Underlay:** Can the chosen outer NBMA destination be reached without recursive routing through the DMVPN tunnel?

The [configuration](configuration.md) and [failover](failover.md) chapters use these commands to verify the designs in detail.

---

## Key Points

- **Single-cloud dual hub:** One mGRE tunnel and one overlay subnet per spoke, with **two NHSs on the same tunnel**.
- **Dual-cloud dual hub:** Two mGRE tunnels and two overlay subnets per spoke, with an NHS in each cloud.
- A second NHS provides **NHRP redundancy**, but routing must still provide a usable alternate forwarding path.
- **Routing**, not the order of `ip nhrp nhs` commands, determines which hub initially carries enterprise traffic.
- In **Phase 3**, a hub can be the initial routing next hop while a later NHRP shortcut sends packets **directly from spoke to spoke**.
- Single cloud has **limited interface-level per-hub metric control** because both hub adjacencies use the same spoke tunnel interface.
- Dual cloud permits **independent tunnel metrics and policies**, at the cost of additional NHRP, GRE, routing, and possibly IPsec state.
- **Shortcuts are scoped to their DMVPN cloud**. Do not assume a shortcut automatically migrates between clouds after a failure.
- Two clouds using the **same WAN interface** do not provide physical WAN redundancy.
- Successful **NHRP registration, overlay routing, and direct forwarding** are separate conditions that must be verified independently.
- Hub-to-hub transit routing may be necessary when a destination spoke is reachable through only one hub or cloud.
- Active/active hubs do not guarantee symmetric traffic, equal utilization, or simultaneous packet forwarding through both hubs.

---

## References

- [Cisco: Dynamic Multipoint IPsec VPNs, Dual-Hub Single and Dual DMVPN Layouts](https://www.cisco.com/c/en/us/support/docs/security-vpn/ipsec-negotiation-ike-protocols/41940-dmvpn.html)
- [Cisco: Scalable DMVPN Design and Implementation Guide](https://community.cisco.com/kxiwq67737/attachments/kxiwq67737/5991-discussions-wan-routing-switching/319468/1/dmvpn_design_guide%281%29.pdf)
- [Cisco: Troubleshoot DMVPN Issues](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/111976-dmvpn-troubleshoot-00.html)
- [Cisco: Implement DMVPN Phase 3 Multi-Subnet Design](https://www.cisco.com/c/en/us/support/docs/security-vpn/dynamic-multi-point-vpn-dmvpn/220217-consider-dmvpn-phase3-routing-for-multi.html)
- [Cisco: Configure Phase-3 Hierarchical DMVPN with Multi-Subnet Spokes](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/211292-Configure-Phase-3-Hierarchical-DMVPN-wit.html)
