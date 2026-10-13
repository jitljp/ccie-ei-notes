
# SSM Mapping

**Source-Specific Multicast (SSM) Mapping** allows receivers that do not support source-specific membership signaling to receive multicast traffic through an SSM network.

Native SSM requires:

- **IPv4:** IGMPv3, which allows receivers to request specific `(S,G)` channels.
- **IPv6:** MLDv2, which provides equivalent source-specific listener signaling.

However, older receivers using **IGMPv1, IGMPv2, or MLDv1** can only request membership in a multicast group `G`. They cannot specify a source `S`.

SSM Mapping solves this problem by allowing the **last-hop router (LHR)** to translate a group-only membership request into one or more source-specific `(S,G)` memberships.

For example:

```text
Legacy Receiver
      |
      | IGMPv2/MLDv1 Report for G
      v
  Last-Hop Router
      |
      | SSM Mapping Database
      | G → S1, S2
      |
      | PIM Join(S1,G)
      | PIM Join(S2,G)
      v
  Multicast Network
```

The receiver continues using its existing IGMP or MLD implementation, while the multicast network uses PIM-SSM forwarding.

## Why SSM Mapping Is Necessary

SSM normally requires receivers to identify the sources they want.

For example, an IGMPv3 receiver can request:

```text
Group:   232.1.1.1
Mode:    INCLUDE
Sources: {10.1.1.10}
```

An MLDv2 receiver can similarly request:

```text
Group:   ff35::fd00:101
Mode:    INCLUDE
Sources: {2001:db8:10::10}
```

These Reports contain enough information for the last-hop router to establish `(S,G)` forwarding state.

However, a legacy receiver sends only a group membership request:

**IPv4:**

```text
IGMPv2 Membership Report

Group: 232.1.1.1
```

**IPv6:**

```text
MLDv1 Listener Report

Group: ff35::fd00:101
```

Neither Report identifies a multicast source.

Without SSM Mapping, these group-only Reports cannot establish native SSM forwarding.

RFC 4607 requires SSM routers to ignore non-source-specific membership requests for groups in the SSM range.

**Cisco SSM Mapping provides a transition mechanism** by supplying the missing source information from an administrator-controlled mapping database.

## SSM Mapping Operation

SSM Mapping operates on the **last-hop router**, which receives membership reports from directly connected receivers.

The basic process is the same for IPv4 and IPv6.

### Step 1: Legacy Receiver Joins Group G

A legacy receiver sends a group-only Report.

For example:

```text
IPv4:
IGMPv2 Report → 232.1.1.1

IPv6:
MLDv1 Report → ff35::fd00:101
```

The receiver does not provide a source address.

### Step 2: Last-Hop Router Looks Up the Sources

The LHR checks its SSM Mapping database for the multicast group.

For example:

**IPv4:**

```text
Group: 232.1.1.1

SSM Mapping:
232.1.1.1 → 10.1.1.10
```

**IPv6:**

```text
Group: ff35::fd00:101

SSM Mapping:
ff35::fd00:101 → 2001:db8:10::10
```

The mapping can be obtained from either:

- **Static SSM Mapping:** Locally configured group-to-source mappings.
- **DNS-Based SSM Mapping:** Source addresses obtained through DNS lookups.

The mapped sources must be known in advance. SSM Mapping does not automatically discover active multicast senders.

### Step 3: Router Converts the Membership into Source-Specific State

The LHR internally translates the group-only request into source-specific receiver interest.

For IPv4:

```text
Original IGMPv2 Report:
Join 232.1.1.1

Mapped Membership:
INCLUDE {10.1.1.10}

Resulting Channel:
(10.1.1.10, 232.1.1.1)
```

For IPv6:

```text
Original MLDv1 Report:
Join ff35::fd00:101

Mapped Membership:
INCLUDE {2001:db8:10::10}

Resulting Channel:
(2001:db8:10::10, ff35::fd00:101)
```

The router processes the resulting interest as though it had learned equivalent source-specific membership information.

> **Important:** The receiver does not actually send an IGMPv3 or MLDv2 Report. The conversion happens internally on the LHR.

### Step 4: Router Sends PIM (S,G) Joins

The LHR performs an RPF lookup toward each mapped source and sends source-specific PIM Joins upstream.

For example:

```text
IPv4:
Join(10.1.1.10, 232.1.1.1)

IPv6:
Join(2001:db8:10::10, ff35::fd00:101)
```

PIM constructs the source-specific distribution tree as usual.

No RP, shared tree, or PIM Register process is required.

**Once the source-specific state has been established, multicast forwarding works exactly as it does with native SSM.**

## Multiple Sources per Group

SSM Mapping can associate **multiple sources** with the same multicast group.

For example:

```text
IPv4 Mapping:

232.1.1.1 → 10.1.1.10
            10.2.2.20
```

The LHR translates a single group-only membership into:

```text
INCLUDE {10.1.1.10, 10.2.2.20}
```

It then establishes two source-specific channels:

```text
(10.1.1.10, 232.1.1.1)
(10.2.2.20, 232.1.1.1)
```

IPv6 operates similarly:

```text
ff35::fd00:101 → 2001:db8:10::10
                 2001:db8:20::20
```

Resulting channels:

```text
(2001:db8:10::10, ff35::fd00:101)
(2001:db8:20::20, ff35::fd00:101)
```

**The LHR joins all mapped sources**, rather than selecting one source from the list.

Cisco IOS XE supports up to **20 mapped sources per IPv4 group**.

### Source Redundancy

Multiple source mappings can be useful for redundant multicast applications.

For example:

```text
                   S1 (Primary)
                        |
                        |
                     Network
                        |
                        |
                   Last-Hop Router
                        |
                        |
                     Receiver

                   S2 (Backup)
                        |
                        +--- Network
```

Suppose both sources are mapped to the same group:

```text
G → S1, S2
```

The LHR maintains forwarding interest in both `(S1,G)` and `(S2,G)`.

However, **SSM Mapping does not perform source election or active/standby selection**.

If both sources transmit simultaneously, the receiver may receive both streams.

For active/standby operation, the multicast applications or source systems must coordinate transmission so that only the intended source is active.

## SSM Mapping vs. Native SSM

Although both use PIM-SSM forwarding, native SSM and SSM Mapping differ in how the source is selected.

| Feature | Native SSM | SSM Mapping |
|---|---|---|
| IPv4 receiver | IGMPv3 | IGMPv1/v2 |
| IPv6 receiver | MLDv2 | MLDv1 |
| Source selection | Receiver/application | LHR mapping database |
| Source information | Included in membership Report | Static configuration or DNS |
| Multiple sources per group | Receiver selects desired sources | LHR joins mapped sources |
| PIM forwarding | `(S,G)` | `(S,G)` |
| RP required | No | No |
| Per-receiver source selection | Supported | Not supported by legacy receivers |

### Application-to-Group Limitation

A major limitation of SSM Mapping is that legacy receivers cannot distinguish between applications using the same multicast group.

Consider:

```text
Application A:
(S1,G)

Application B:
(S2,G)
```

With native SSM:

```text
Receiver 1 → Subscribe(S1,G)
Receiver 2 → Subscribe(S2,G)
```

Each receiver can request its preferred channel.

With SSM Mapping:

```text
Receiver 1 → Join G
Receiver 2 → Join G

Mapping Database:
G → S1, S2
```

Both receivers request the same group-only membership, so both receive traffic from the mapped source set.

**SSM Mapping effectively supports one application per group**, although that application may have multiple mapped sources.

Native SSM receivers can still use other `(S,G)` channels with the same group address independently.

## Membership Maintenance and Mapping Changes

SSM Mapping must maintain source-specific forwarding interest based on legacy group membership state.

For example, an IGMPv2 receiver periodically responds to Queries to maintain its membership in group G.

As long as receiver interest remains and the mapping is valid, the LHR maintains the corresponding source-specific forwarding state.

When the receiver leaves:

1. The router follows the normal IGMP or MLD leave procedure.
2. It determines whether other receivers still need the group.
3. If no receiver interest remains, the mapped `(S,G)` forwarding interest can be removed.
4. PIM Prunes are sent upstream when necessary.

IGMPv1 does not support explicit Leave messages, so its group membership typically expires through the membership timer.

### Changes to the Mapping Database

Mappings may also change while receivers remain subscribed.

For example:

```text
Original:
G → S1

Updated:
G → S2
```

The LHR must reconcile its source-specific forwarding state with the updated mapping.

It can remove forwarding interest in `(S1,G)` and establish forwarding interest in `(S2,G)`.

This can cause a temporary traffic interruption while the multicast distribution tree changes.

DNS-based mappings can also change when cached records expire and new DNS results are obtained.

## Static vs. DNS-Based SSM Mapping

IOS XE supports two methods for determining the mapped sources.

| Feature | Static Mapping | DNS-Based Mapping |
|---|---|---|
| Source information | Locally configured | Retrieved from DNS |
| Group matching | ACL-based | DNS lookup based on group address |
| Centralized administration | No | Yes |
| DNS server required | No | Yes |
| Source changes | Configuration changes | DNS record updates and cache refreshes |

### Static SSM Mapping

Static SSM Mapping uses ACLs to match multicast groups and associates those groups with source addresses.

It is useful when the number of groups and sources is relatively small or the mappings rarely change.

Multiple matching static entries can provide multiple source addresses for a group.

### DNS-Based SSM Mapping

DNS-Based SSM Mapping allows the LHR to look up sources using DNS.

This is useful for larger deployments because source mappings can be maintained centrally.

For IPv4, IOS XE constructs a DNS name based on the multicast group address and looks up the corresponding source addresses using DNS A records.

The IPv6 implementation also supports DNS-based source mapping for MLDv1 receivers.

> On IOS XE, enabling SSM Mapping automatically enables DNS-based mapping by default. DNS queries can be explicitly disabled when only static mappings are required.

For IPv4, locally configured static mappings take precedence over DNS-based mappings.

The exact configuration and verification commands are covered in [Static and DNS-Based SSM Mapping](static-ssm-mapping.md).

## IOS XE Configuration Overview

SSM Mapping is a separate feature from PIM-SSM itself.

Enabling SSM does not automatically enable SSM Mapping.

### IPv4

Enable the standard IPv4 SSM range:

```text
R1(config)# ip pim ssm default
```

Enable IGMP SSM Mapping:

```text
R1(config)# ip igmp ssm-map enable
```

A static mapping or valid DNS configuration is then needed to determine the sources associated with legacy group memberships.

### IPv6

Enable IPv6 multicast routing:

```text
R1(config)# ipv6 multicast-routing
```

Enable MLD SSM Mapping:

```text
R1(config)# ipv6 mld ssm-map enable
```

As with IPv4, the LHR must be able to determine the mapped sources.

IPv6 MLD SSM Mapping also supports VRF-specific configuration using the `vrf` keyword.

## SSM Mapping vs. Static Group Membership

Do not confuse **SSM Mapping** with configuring static source-specific multicast forwarding interest.

For IPv4:

```text
ip igmp static-group 232.1.1.1 source 10.1.1.10
```

This creates static forwarding interest for the explicitly specified `(S,G)` channel.

By contrast:

```text
ip igmp static-group 232.1.1.1 source ssm-map
```

Uses SSM Mapping to determine the source addresses and establishes static forwarding interest in the resulting channels.

The distinction is:

- **Static `(S,G)` membership:** The source address is specified directly in the interface configuration.
- **Static membership using SSM Mapping:** The group is specified, but the source addresses come from the SSM Mapping database.

For IPv6, `ipv6 mld static-group` can similarly create static source-specific forwarding interest without requiring a dynamic MLD Report.

## Verification

### IPv4

```text
show ip igmp ssm-mapping
show ip igmp ssm-mapping 232.1.1.1
show ip igmp groups detail
show ip mroute
```

| Command | Purpose |
|---|---|
| `show ip igmp ssm-mapping` | Displays SSM Mapping status |
| `show ip igmp ssm-mapping <group>` | Displays the mapping database and sources for a group |
| `show ip igmp groups detail` | Verifies receiver state and mapped sources |
| `show ip mroute` | Verifies resulting `(S,G)` routing state |

In detailed IGMP group output, IOS XE can display an **`M` flag** identifying sources learned through SSM Mapping.

### IPv6

```text
show ipv6 mld ssm-map
show ipv6 mld ssm-map ff35::fd00:101
show ipv6 mld groups detail
show ipv6 mroute
```

The IPv6 SSM Mapping output can identify:

```text
Group address
Group mode ssm
Database (STATIC or DNS)
Source list
```

To verify the complete forwarding process, confirm:

1. The legacy IGMP/MLD group membership is present.
2. The group has a valid SSM Mapping entry.
3. The expected source addresses appear in the mapped source list.
4. PIM has established the corresponding `(S,G)` forwarding state.
5. Multicast traffic follows the correct RPF path and reaches the receiver-facing interface.

## Key Points

- **SSM Mapping allows legacy IGMPv1/v2 and MLDv1 receivers to use PIM-SSM forwarding.**
- The **last-hop router** translates group-only membership reports into source-specific `(S,G)` memberships.
- Source addresses are obtained from **static mappings or DNS**.
- SSM Mapping does not change the PIM forwarding mechanism. The network still uses `(S,G)` source trees without an RP.
- One group can map to multiple sources, and the LHR establishes forwarding interest in all mapped channels.
- Cisco IOS XE supports up to **20 mapped sources per IPv4 group**.
- Unlike native SSM, legacy receivers cannot independently choose sources for the same group.
- SSM Mapping does not discover active sources automatically or provide active/standby source selection.
- Mapping changes can cause the LHR to update its source-specific forwarding state.
- **Enabling PIM-SSM does not automatically enable SSM Mapping.**
