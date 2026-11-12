# Per-Tunnel QoS for DMVPN

## Overview

**Per-Tunnel QoS for DMVPN** allows a router to apply an **independent outbound QoS policy to each remote DMVPN endpoint**, even though multiple peers share a multipoint GRE (mGRE) tunnel interface.

The original use case is **hub-to-spoke QoS**. A hub with a high-bandwidth connection must avoid overwhelming spokes with much smaller WAN circuits.

```text
                         Hub
                      1-Gbps WAN
                          |
                    MPLS / Internet
                          |
               +----------+----------+
               |          |          |
             Spoke1     Spoke2     Spoke3
             10 Mbps    20 Mbps    50 Mbps
```

Instead of shaping all DMVPN traffic to a single aggregate rate, the hub can create three separate QoS instances:

- Hub -> Spoke1: 10 Mbps
- Hub -> Spoke2: 20 Mbps
- Hub -> Spoke3: 50 Mbps

**The shaper is instantiated per remote endpoint, not per NHRP group.** If three spokes use the `CIR-10M` group, each can have its *own* 10-Mbps policy instance; they do not share one 10-Mbps allowance.

Per-tunnel QoS can use **hierarchical QoS (HQoS)**: a parent policy shapes traffic for a remote peer, while a child policy allocates bandwidth and provides priority treatment among traffic classes.

### Traffic Directions

| Traffic | Typical QoS mechanism | Where configured |
|---|---|---|
| Hub -> spoke | NHRP group-based per-tunnel QoS | Hub |
| Spoke -> hub | Conventional aggregate upstream shaping; platform-dependent tunnel QoS or adaptive QoS if required | Spoke |
| Spoke -> spoke, via hub | Hub-to-spoke policy on the forwarding hub | Hub |
| Spoke -> spoke, direct Phase 3 shortcut | NHRP attribute-based spoke-to-spoke per-tunnel QoS | Sending spoke |

**NHRP signaling selects a local policy; it does not exchange QoS configurations, dynamically discover the CIR, or automatically shape the sending spoke's complete WAN uplink.**

## Hub-to-Spoke QoS: NHRP Groups

The hub selects a per-spoke policy from an **NHRP group name** advertised by the spoke.

1. The spoke is configured with `nhrp group GROUP` on its DMVPN interface.
2. The spoke includes that group name in its periodic **NHRP Registration Requests** to the hub.
3. The hub looks up its local `nhrp map group` entry.
4. The hub creates or updates an outbound policy instance for that particular spoke.
5. Packets transmitted by the hub toward the spoke are processed by that policy.

```text
Spoke1                                       Hub
   |                                          |
   | NHRP Registration Request                |
   | Group = CIR-10M                          |
   |----------------------------------------->|
   |                                          | Maps CIR-10M
   |                                          | to PARENT-10M
   |<-----------------------------------------|
   |      Hub -> Spoke1: shaped to 10 Mbps    |
```

The basic configuration is:

**Spoke:**

```text
Spoke1(config)# interface Tunnel0
Spoke1(config-if)# nhrp group CIR-10M
```

**Hub:**

```text
Hub(config)# interface Tunnel0
Hub(config-if)# nhrp map group CIR-10M service-policy output PARENT-10M
```

The group name is an **identifier only**. `CIR-10M` does not itself configure 10 Mbps; the actual rate is determined by `PARENT-10M` on the hub.

Use the protocol-independent `nhrp group` and `nhrp map group` syntax on current IOS XE. Older `ip nhrp group` and `ip nhrp map group` forms were deprecated and removed from the CLI in later IOS XE releases.

### Group Membership and Policy Lifecycle

- A spoke can belong to **one NHRP group per GRE tunnel interface**.
- Different DMVPN tunnel interfaces can advertise different group names.
- Many spokes can use the same group while receiving **independent** policy instances.
- A group configuration change is usually signaled in the **next periodic registration**, not necessarily immediately.
- If a newly received group differs from the previous one, the hub removes the old policy instance and applies the new matching policy.
- If a registration no longer includes a group, an existing group-selected policy is removed.
- If a group has no matching map on the hub, no group-selected QoS policy is applied for that group.
- With dual hubs, **each hub needs its own mappings and policy definitions**; a spoke's registration can advertise the same group to both hubs.

NHRP group membership is separate from EIGRP/OSPF route selection, IPsec security associations, and NHRP shortcut selection.

## Per-Tunnel HQoS

A common design uses a **parent shaper** and a **child queuing policy**:

```text
           Hub -> Spoke1
                  |
        +--------------------+
        | Parent: PARENT-10M |
        | shape average 10M  |
        +--------------------+
                  |
        +--------------------+
        | Child: CHILD       |
        | VOICE   -> LLQ     |
        | CONTROL -> CBWFQ   |
        | DEFAULT -> other   |
        +--------------------+
                  |
                Spoke1
```

The parent defines the available shaping rate. A percentage-based child policy derives its rates from that parent; it need not be duplicated for every CIR tier.

For example, with `priority percent 20` and `bandwidth percent 10`:

| Class | 10-Mbps parent | 20-Mbps parent |
|---|---:|---:|
| VOICE: priority 20% | 2 Mbps | 4 Mbps |
| CONTROL: bandwidth 10% | 1 Mbps | 2 Mbps |

The priority rate provides LLQ treatment and an appropriate congestion-time rate limit. The bandwidth statement supplies a minimum scheduling guarantee under congestion; these numbers are not permanently consumed reservations.

See [Hierarchical QoS](../../infrastructure-security-and-services/quality-of-service/hqos/overview.md) for generic parent-child behavior.

## Configuration: Hub-to-Spoke QoS

Assume DMVPN, NHRP, IKEv2/IPsec, and routing are already working:

```text
                            Hub
                       Tunnel0 10.0.0.1
                              |
                        MPLS network
                              |
                  +-----------+-----------+
                  |                       |
                Spoke1                  Spoke2
          Tunnel0 10.0.0.2        Tunnel0 10.0.0.3
             CIR 10 Mbps             CIR 20 Mbps
```

### 1. Define Traffic Classes on the Hub

```text
Hub(config)# class-map match-any VOICE
Hub(config-cmap)# match dscp ef
Hub(config-cmap)# exit
Hub(config)# class-map match-any CONTROL
Hub(config-cmap)# match dscp cs6
```

The example assumes that the traffic is already correctly marked and that the platform supports classifying these fields in the selected policy path. Verify DSCP preservation across encapsulation and encryption when the WAN provider also performs QoS.

### 2. Create a Child Queuing Policy

```text
Hub(config)# policy-map CHILD
Hub(config-pmap)# class VOICE
Hub(config-pmap-c)# priority percent 20
Hub(config-pmap-c)# exit
Hub(config-pmap)# class CONTROL
Hub(config-pmap-c)# bandwidth percent 10
Hub(config-pmap-c)# exit
Hub(config-pmap)# class class-default
```

### 3. Create Parent Shapers

```text
Hub(config)# policy-map PARENT-10M
Hub(config-pmap)# class class-default
Hub(config-pmap-c)# shape average 10000000
Hub(config-pmap-c)# service-policy CHILD
!
Hub(config)# policy-map PARENT-20M
Hub(config-pmap)# class class-default
Hub(config-pmap-c)# shape average 20000000
Hub(config-pmap-c)# service-policy CHILD
```

### 4. Map Groups to the Policies on the Hub

```text
Hub(config)# interface Tunnel0
Hub(config-if)# nhrp map group CIR-10M service-policy output PARENT-10M
Hub(config-if)# nhrp map group CIR-20M service-policy output PARENT-20M
```

### 5. Advertise a Group from Each Spoke

```text
Spoke1(config)# interface Tunnel0
Spoke1(config-if)# nhrp group CIR-10M
!
Spoke2(config)# interface Tunnel0
Spoke2(config-if)# nhrp group CIR-20M
```

**No spoke-side QoS policy map is required for this basic hub-to-spoke function.** The actual policies are locally defined and enforced on the hub.

## Spoke-to-Hub QoS: Upstream Shaping

The NHRP configuration above **does not shape spoke-to-hub traffic**. The hub is the transmitting router for hub-to-spoke traffic; the spoke is the transmitting router for the reverse direction.

A conventional design is to shape on the spoke's **outbound WAN/tunnel-source interface**, protecting its upstream circuit. For a 10-Mbps contracted uplink, an illustrative policy might be:

```text
Spoke1(config)# policy-map SPOKE-UPLINK
Spoke1(config-pmap)# class class-default
Spoke1(config-pmap-c)# shape average 9000000
!
Spoke1(config)# interface GigabitEthernet0/0
Spoke1(config-if)# service-policy output SPOKE-UPLINK
```

This example deliberately uses 9 Mbps rather than the full 10 Mbps to provide illustrative overhead/headroom; the correct value depends on the provider's accounting model and encapsulation. The physical policy controls **all traffic** exiting that interface, including packets toward the hub, other spokes, and any non-DMVPN destinations.

On a platform supporting a suitable outbound tunnel-level policy, shaping may also be possible on `Tunnel0`; however, that is **an aggregate policy for the tunnel interface**, not automatically one independent policy per hub/peer. Queuing-policy support and combinations with NHRP per-tunnel policies are platform-dependent. In particular, do not assume a generic tunnel-attached CBWFQ configuration will work unchanged on every mGRE/IPsec platform.

**If using a physical/subinterface shaper together with DMVPN per-tunnel QoS**, Cisco documents restrictive policy combinations: generally a `class-default`-only shaper on the tunnel-source physical interface or subinterface, applied **before** the tunnel's per-tunnel QoS configuration. Do not simultaneously attach these aggregate policies at both the main interface and its subinterface.

## DMVPN Phase 3: Direct Spoke-to-Spoke QoS

In Phase 3, the first packets between two spokes can pass through a hub, while NHRP redirects and shortcuts allow subsequent packets to use a **direct spoke-to-spoke path**.

```text
Before shortcut:  Spoke1 ------ Hub ------ Spoke2

After shortcut:   Spoke1 ----------------- Spoke2
```

Once the direct shortcut is active, the hub no longer transmits those packets to Spoke2, so the hub's `nhrp map group` policy **cannot protect that direct traffic**.

The **spoke-to-spoke per-tunnel QoS** enhancement lets each sending spoke select a local outbound policy using the *destination spoke's* advertised group identity.

### Registration Groups vs. Resolution Attributes

These are **two separate NHRP signaling mechanisms**:

| Command on spoke | Purpose | NHRP signaling |
|---|---|---|
| `nhrp group CIR-10M` | Tell hubs which policy to use when transmitting *to this spoke* | Registration Request |
| `nhrp attribute group CIR-10M` | Tell other spokes which group represents *this spoke* | Resolution Request/Reply Vendor Private Extension (VPE) |
| `nhrp map group CIR-20M service-policy output PARENT-20M` | When transmitting directly to a peer advertising `CIR-20M`, use the local `PARENT-20M` policy | Local mapping; not a message |

For Phase 3 deployments requiring **both hub-to-spoke and direct spoke-to-spoke per-tunnel QoS**, spokes can have **both** `nhrp group` and `nhrp attribute group` on the same tunnel interface. The names can match for clarity, but they are configured independently and serve different signaling purposes.

### Example: Both Spokes Support Direct QoS

Assume Spoke1 has a 10-Mbps receive CIR, and Spoke2 has a 20-Mbps receive CIR. Both share `Tunnel0` with the hubs. The hub is configured as in the preceding section.

**Spoke1:**

```text
! Define a suitable PARENT-20M policy locally on Spoke1.
! (For example, reuse the CHILD and PARENT-20M definitions above.)
Spoke1(config)# interface Tunnel0
Spoke1(config-if)# nhrp group CIR-10M
Spoke1(config-if)# nhrp attribute group CIR-10M
Spoke1(config-if)# nhrp map group CIR-20M service-policy output PARENT-20M
```

**Spoke2:**

```text
! Define a suitable PARENT-10M policy locally on Spoke2.
Spoke2(config)# interface Tunnel0
Spoke2(config-if)# nhrp group CIR-20M
Spoke2(config-if)# nhrp attribute group CIR-20M
Spoke2(config-if)# nhrp map group CIR-10M service-policy output PARENT-10M
```

The policy maps referenced above must exist **locally on the router applying them**; policies are never copied from the hub or from the other spoke.

The resulting directions are:

```text
                     Hub
                |           |
         10-Mbps policy  20-Mbps policy
                |           |
              Spoke1      Spoke2
                |           ^
                +---------->|  Spoke1 uses local PARENT-20M
                <-----------+  Spoke2 uses local PARENT-10M
```

- **Hub -> Spoke1:** Hub uses its `PARENT-10M` instance.
- **Hub -> Spoke2:** Hub uses its `PARENT-20M` instance.
- **Spoke1 -> Spoke2 (direct):** Spoke1 sees Spoke2's `CIR-20M` attribute and uses *its local* `PARENT-20M` policy.
- **Spoke2 -> Spoke1 (direct):** Spoke2 sees Spoke1's `CIR-10M` attribute and uses *its local* `PARENT-10M` policy.

**A remote-endpoint shaper is not an uplink capacity guarantee.** If Spoke1 has only a 10-Mbps upload CIR, its 20-Mbps per-peer policy toward Spoke2 does not prevent it from overdriving its own uplink. An appropriate aggregate upstream shaper is still needed.

The NHRP attribute-based mechanism is only useful when the NHRP exchange and shortcut forwarding actually establish direct spoke-to-spoke endpoints. It does not turn a Phase 1 hub-and-spoke design into a direct spoke-to-spoke design, nor does it replace NHRP shortcut/redirect configuration.

## Aggregate Hub QoS and Encapsulation Overhead

Per-tunnel shapers protect each remote spoke, but **the sum of their configured rates may exceed the hub's WAN CIR**. For example, 30 spokes each shaped at 10 Mbps could offer 300 Mbps toward a hub whose provider contract is only 200 Mbps.

An additional *aggregate* egress shaper may be required:

```text
Hub(config)# policy-map HUB-WAN-200M
Hub(config-pmap)# class class-default
Hub(config-pmap-c)# shape average 200000000
!
Hub(config)# interface GigabitEthernet0/0
Hub(config-if)# service-policy output HUB-WAN-200M
```

The physical policy must follow the platform's restrictions when combined with NHRP-selected per-tunnel policies; notably, Cisco specifies a `class-default`-only shaping policy and an attachment-order requirement.

Cisco's per-tunnel processing accounts for **GRE, IPsec, and physical Layer 2 encapsulation overhead** when calculating shaping and bandwidth-queuing packet sizes. However, the provider's definition of CIR and any additional transport overhead still affect the appropriate configured shaping rate.

If QoS classification depends on original inner IP addresses or transport ports, evaluate `qos pre-classify` and the supported platform forwarding path. DSCP-only classification typically does **not** require `qos pre-classify`; confirm how DSCP is copied or remarked through the chosen encapsulation and IPsec processing.

## Adaptive QoS over DMVPN

Ordinary per-tunnel QoS uses **static** shaping rates. This is appropriate when bandwidth is predictable, such as a committed MPLS service, but less effective when Internet bandwidth fluctuates.

**Adaptive QoS over DMVPN** is a separate feature that adjusts a shaper according to measured available bandwidth. Cisco documents the ability to deploy adaptive QoS on a hub, a spoke, or both, with additional prerequisites including IPsec and supported spoke-side egress per-SA QoS.

Adaptive QoS should not be confused with `nhrp attribute group` alone: **attribute-based spoke-to-spoke QoS selects a policy**, while **adaptive QoS changes shaping rates based on bandwidth measurements**. They address different problems.

## Verification

| Command | What to check |
|---|---|
| `show dmvpn detail` | Peer/NHS state, received NHRP group, applied policy information |
| `show nhrp` | NHRP cache entries and group information |
| `show nhrp group-map` | Local group-to-policy mappings and attached endpoints |
| `show policy-map multipoint` | Dynamically instantiated per-endpoint policy statistics |
| `show policy-map multipoint tunnel 0` | Per-endpoint statistics for a particular multipoint tunnel |
| `show tunnel endpoints` | Tunnel source/destination endpoints and policies |
| `show policy-map interface GigabitEthernet0/0` | Aggregate uplink policy/counters where supported |
| `show policy-map` | Policy definitions (does not prove dynamic attachment) |

### Example: Group-to-Policy Mapping

The following is **illustrative output**, not a captured device transcript:

```text
Hub# show nhrp group-map
Interface: Tunnel0
 NHRP group: CIR-10M
  QoS policy: PARENT-10M
  Tunnels using the QoS policy:
   10.0.0.2/192.0.2.2
 NHRP group: CIR-20M
  QoS policy: PARENT-20M
  Tunnels using the QoS policy:
   10.0.0.3/192.0.2.3
```

Follow with `show policy-map multipoint tunnel 0` to verify **independent** per-endpoint shaper rates and counters. A correct policy map definition by itself is insufficient evidence that NHRP selected and installed the policy for a particular peer.

For a direct Phase 3 shortcut, verify (1) the active shortcut/NHRP peer information, (2) the received remote group attribute, and (3) the **policy on the sending spoke**, not merely the hub's policy.

## Troubleshooting

| Symptom | Likely explanation or next check |
|---|---|
| No hub-to-spoke QoS policy | Missing `nhrp group`, no matching hub `nhrp map group`, or registration not yet updated |
| Group change not visible immediately | Group is normally signaled at the next NHRP registration |
| All spokes appear under one group | Normal; verify one QoS instance per endpoint rather than an aggregate shaper |
| Wrong shaping rate | Group-to-policy mapping references an incorrect parent rate |
| Hub QoS works, but shortcut traffic is unaffected | The direct path bypasses the hub; configure and verify spoke-to-spoke attributes and maps |
| Direct spoke-to-spoke peer has no policy | Missing `nhrp attribute group`, no local map for the **remote** group, or no usable shortcut |
| Spoke still congests its own uplink | Remote-peer policies do not replace the spoke's aggregate upstream shaper |
| Physical uplink and per-tunnel policies conflict | Check supported `class-default`-only policy combination and order of application |
| QoS counters look inconsistent | Check platform-specific accounting and whether counters are gathered at the tunnel endpoint or underlying interface |
| Commands exist but feature does not operate | Confirm the exact IOS XE release, platform, image, and forwarding feature combination |

## Restrictions and Design Considerations

1. **Egress only:** NHRP per-tunnel service policies apply as outbound policies toward selected remote DMVPN endpoints.
2. **CEF and DMVPN:** CEF is required, and basic DMVPN connectivity must already work before adding group-based QoS.
3. **Avoid fair queuing:** Cisco cautions against fair queuing within per-tunnel DMVPN policies because outer tunnel headers can cause unrelated inner flows to hash into the same fair queue.
4. **Post-encapsulation processing:** Queuing and shaping for the feature occur in the outbound forwarding path for GRE/IPsec packets; the feature does not add arbitrary user-configurable pre-crypto scheduling.
5. **Transport and image support:** Supported IPv4/IPv6 passenger and transport combinations vary by release/platform. Do not assume all virtual images implement the same forwarding/QoS behavior.
6. **Physical-interface coexistence:** An aggregate tunnel-source main-interface or subinterface policy used alongside per-tunnel QoS is subject to the `class-default` shaping-only and attachment-order restrictions described earlier.
7. **Port-channel sources:** Selected platforms support per-tunnel QoS sourced from port-channels and may require `platform qos port-channel-aggregate 1` *before* creating the port-channel.
8. **No automatic admission control:** Configuring many independent shapers does not guarantee that their combined offered traffic fits the hub, spoke, or service provider's actual capacity.
9. **No routing change:** NHRP QoS group membership does not alter EIGRP/OSPF metrics or force direct spoke-to-spoke paths to be established.

## Key Points

- **Hub-to-spoke:** `nhrp group` on each spoke; `nhrp map group ... service-policy output ...` on each hub.
- **Spoke-to-spoke Phase 3:** `nhrp attribute group` advertises a spoke's group during NHRP resolution; local `nhrp map group` selects a policy based on the **remote** spoke's advertised group.
- **Both directions require policies at the transmitter.** A hub policy cannot control packets that a spoke sends directly to another spoke.
- A group can have multiple members, but each remote endpoint receives an **independent** policy instance.
- A spoke's aggregate upstream shaper is a **separate design decision** from per-remote-endpoint QoS.
- Use parent shaping with child LLQ/CBWFQ where supported; verify the actual dynamic policy with `show policy-map multipoint`.
- Static NHRP group-selected QoS and **Adaptive QoS over DMVPN** are distinct capabilities.

## References

- [Cisco IOS XE 17.x — Per-Tunnel QoS for DMVPN](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_sec-conn-dmvpn-per-tunnel-qos.html)
- [Cisco IOS XE 17.x — Adaptive QoS over DMVPN](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-adaptive-qos-dmvpn.html)
- [Cisco IOS XE 17.x — Per-Tunnel QoS Support for Multiple Policy Maps (MPOL)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-conn-dmvpn-mpol-support.html)
- [Cisco — Quality of Service Options on GRE Tunnel Interfaces](https://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-policing/10106-qos-tunnel.html)
