# MLDv2 Source Filtering

**MLDv2 source filtering** allows an IPv6 multicast listener to specify which sources it wants (or does not want) to receive traffic from for a particular multicast group.

It is the IPv6 equivalent of **IGMPv3 source filtering**. The INCLUDE/EXCLUDE model, host-state aggregation, and six Multicast Address Record types work essentially the same way, so this page focuses on the IPv6-specific behavior and IOS XE commands.

MLDv1 expresses interest only in a group (`G`). MLDv2 can express interest in individual **source/group channels (`S,G`)**.

## INCLUDE and EXCLUDE Modes

A listener's reception state is described by:

```text
(Multicast Group, Filter Mode, Source List)
```

| Filter State | Meaning |
|---|---|
| `INCLUDE {S1,S2}` | Receive group G only from S1 and S2 |
| `INCLUDE {}` | Receive G from no sources; effectively no listening state |
| `EXCLUDE {S1,S2}` | Receive G from every source except S1 and S2 |
| `EXCLUDE {}` | Receive G from every source |

For example, a listener interested in one IPv6 SSM channel could have:

```text
Group:    ff35::fd00:1234
Mode:     INCLUDE
Sources:  {2001:db8:10::10}

Requested channel:
(2001:db8:10::10, ff35::fd00:1234)
```

The source addresses are the IPv6 addresses of the multicast senders from which the listeners wants to receive the multicast stream.

An any-source membership, such as one for `ff15::1234`, can instead be represented as `EXCLUDE {}`.

### Relationship to MLDv1

An MLDv1 listener can join or leave an IPv6 multicast group, but it cannot communicate a source list.

Conceptually:

```text
MLDv1 join G   = MLDv2 EXCLUDE {}
MLDv1 leave G  = MLDv2 INCLUDE {}
```

These are equivalent **reception intentions**, not identical message formats. MLDv1 uses Reports and Done messages; MLDv2 uses Multicast Address Records in Type 143 Reports.

## Listener State vs. Router State

MLDv2 distinguishes between the source-filtering state requested by applications and the **aggregate state maintained by the multicast router**.

A host combines the requests of all applications on an interface. For example:

```text
Application A: INCLUDE {S1}
Application B: INCLUDE {S2}

Host interface: INCLUDE {S1,S2}
```

If any application uses EXCLUDE mode, the interface uses EXCLUDE mode and calculates a source list that satisfies *all* applications. The host can filter unwanted packets further before delivering them to individual sockets.

The router must similarly combine Reports from **all listeners on the link**. For example:

```text
Rec1: INCLUDE {S1}
Rec2: INCLUDE {S2}

Router's aggregate listener state:
INCLUDE {S1,S2}
```

The router does **not** automatically maintain one membership record per host. It maintains group and source state for the attached link; optional explicit tracking can provide per-listener information.

### Router INCLUDE and EXCLUDE State

RFC 9777 represents router state as:

```text
INCLUDE (A)

or

EXCLUDE (X,Y)
```

- `A` is the **Include List**: sources for which listeners have requested reception.
- `X` is the **Requested List**: sources the router tracks with running Source Timers, including interest that may need to survive an EXCLUDE-to-INCLUDE transition.
- `Y` is the **Exclude List**: sources with Source Timers of zero that should not be forwarded to the receiver link based on listener interest.

For example, suppose two receivers want group `ff15::1234`:

```text
Rec1: EXCLUDE {S1}  (wants all sources except S1)
Rec2: INCLUDE {S1}  (explicitly wants S1)
```

The aggregate router state can be:

```text
EXCLUDE (
  X = {S1},
  Y = {}
)
```

**All sources are currently wanted on the link**: Rec1 wants everything except S1, and Rec2 wants S1. The router keeps S1 in the Requested List so it can preserve that interest if the EXCLUDE-mode receiver disappears.

When the **Filter Timer** expires, indicating that no EXCLUDE-mode listener remains, the router can transition to:

```text
INCLUDE {S1}
```

MLDv2 uses a **Filter Timer** for the group in EXCLUDE mode and individual **Source Timers** for source records. Source-specific Queries can shorten the relevant timer to the Last Listener Query Time when the querier needs to verify that a source is no longer wanted.

> MLDv2 provides forwarding recommendations to PIM; it does not override multicast routing decisions, and EXCLUDE lists are not data-plane security ACLs.

## MLDv2 Multicast Address Records

The same six Record Types used by IGMPv3 appear in MLDv2 **Multicast Address Records**:

| Type | Record | Meaning |
|---|---|---|
| 1 | `MODE_IS_INCLUDE` | Current INCLUDE state |
| 2 | `MODE_IS_EXCLUDE` | Current EXCLUDE state |
| 3 | `CHANGE_TO_INCLUDE_MODE` | Entire new INCLUDE list after a mode change |
| 4 | `CHANGE_TO_EXCLUDE_MODE` | Entire new EXCLUDE list after a mode change |
| 5 | `ALLOW_NEW_SOURCES` | Sources newly wanted |
| 6 | `BLOCK_OLD_SOURCES` | Sources no longer wanted |

**Current-State Records** (1 and 2) respond to Queries. **State-Change Records** (3–6) communicate changes without waiting for a Query.

For example, a listener adding S2:

```text
Old: INCLUDE {S1}
New: INCLUDE {S1,S2}

Report: ALLOW_NEW_SOURCES {S2}
```

Leaving its only `(S,G)` channel:

```text
Old: INCLUDE {S1}
New: INCLUDE {}

Report: BLOCK_OLD_SOURCES {S1}
```

Joining a traditional any-source group:

```text
Old: INCLUDE {}
New: EXCLUDE {}

Report: CHANGE_TO_EXCLUDE_MODE {}
```

The important rule is that **ALLOW/BLOCK describe changes**, whereas `MODE_IS_*` and `CHANGE_TO_*_MODE` describe a complete source list. The detailed record transitions have already been covered in the IGMPv3 section.

## MLDv2 Source Filtering and IPv6 SSM

IPv6 Source-Specific Multicast uses the `ff3x:0000::/32` address space, with group allocation within `ff3x::/96`.

An SSM listener requests a specific `(S,G)` channel using **INCLUDE mode**:

```text
Group:  ff35::fd00:1234
Source: 2001:db8:10::10

INCLUDE {2001:db8:10::10}
```

An SSM-aware host should not send EXCLUDE-mode records for SSM groups, and SSM-aware routers ignore those records. MLDv2 also permits INCLUDE/EXCLUDE signaling for non-SSM groups, but an INCLUDE-mode Report **does not by itself turn an ASM group into an SSM-routed group**.

## Cisco IOS XE Configuration

MLDv2 is the default MLD version after IPv6 multicast routing is enabled:

```text
R1(config)# ipv6 multicast-routing
```

### Join One Source/Group Channel

Configure an IOS XE device to act as a receiver:

```text
Rec1(config)# interface Ethernet0/0
Rec1(config-if)# ipv6 mld join-group ff35::fd00:1234 include 2001:db8:10::10
```

This configures local listener interest in:

```text
(2001:db8:10::10, ff35::fd00:1234)
```

When a source address is supplied, **INCLUDE is the default** if neither mode is specified.

To receive traffic from all sources except one, use an ASM group:

```text
Rec1(config-if)# ipv6 mld join-group ff15::1234 exclude 2001:db8:10::10
```

To remove the configured receiver membership:

```text
Rec1(config-if)# no ipv6 mld join-group ff35::fd00:1234 include 2001:db8:10::10
```

> `ipv6 mld join-group` makes the router itself a receiver; it is not the same as adding a downstream multicast forwarding interface. Also, **repeating `ipv6 mld join-group` for the same group replaces the previous configuration** rather than adding another source. Use a source list when multiple sources are required.

You can confirm on the router with:

```
R1# show ipv6 mld groups detail 
Interface:      Ethernet0/1
Group:          FF35::FD00:1234
Uptime:         00:01:02
Router mode:    INCLUDE
Host mode:      INCLUDE
Last reporter:  FE80::A8BB:CCFF:FE00:5400
Group source list:
Source Address                          Uptime    Expires   Fwd  Flags
2001:DB8:10::10                         00:00:13  00:04:07  Yes  Remote 4
```

### Join Multiple Sources with an IPv6 ACL

The `source-list` option identifies multiple multicast senders using a named IPv6 ACL:

```text
Rec1(config)# ipv6 access-list MLD-SOURCES
Rec1(config-ipv6-acl)# permit ipv6 host 2001:db8:10::10 any
Rec1(config-ipv6-acl)# permit ipv6 host 2001:db8:10::20 any
Rec1(config-ipv6-acl)# exit
Rec1(config)# interface Ethernet0/0
Rec1(config-if)# ipv6 mld join-group ff35::fd00:1234 include source-list MLD-SOURCES
```

The ACL selects **multicast source IPv6 addresses** for the MLD source list. This is not an IPv6 packet-filter ACL applied to the interface.

Confirm on the router:

```
R1# show ipv6 mld groups detail 
Interface:      Ethernet0/1
Group:          FF35::FD00:1234
Uptime:         00:02:39
Router mode:    INCLUDE
Host mode:      INCLUDE
Last reporter:  FE80::A8BB:CCFF:FE00:5400
Group source list:
Source Address                          Uptime    Expires   Fwd  Flags
2001:DB8:10::10                         00:01:50  00:03:51  Yes  Remote 4
2001:DB8:10::20                         00:00:02  00:04:17  Yes  Remote 4
```

### Create Static Forwarding Interest

To make the router treat an interface as having a downstream listener **without making the router itself a local receiver**:

```text
R1(config)# interface Ethernet0/1
R1(config-if)# ipv6 mld static-group ff35::fd00:1234 include 2001:db8:10::10
```

`ipv6 mld static-group` also supports `exclude` and `source-list`. Actual forwarding still depends on multicast routing conditions, including the PIM forwarding state.

## Source Filtering vs. MLD Admission Filtering

Do not confuse these two features:

| Feature | Purpose |
|---|---|
| **MLDv2 INCLUDE/EXCLUDE** | A listener describes which multicast **sources** it wants for a group |
| **`ipv6 mld access-group`** | An administrator controls which received MLD memberships the router is allowed to accept |

INCLUDE/EXCLUDE state describes requested multicast **data** reception; it does not filter MLD Queries or Reports. Even if a router does not request a particular source for a receiver link, multicast routing or other Layer 2 forwarding behavior can still cause traffic to appear on that link.

MLD access control and state limits are covered separately in **MLD Filtering and Limits**.

## Key Points

- **MLDv2 source filtering is the IPv6 equivalent of IGMPv3 source filtering.**
- `INCLUDE {S}` requests only those sources; `EXCLUDE {S}` requests all sources except those listed.
- `INCLUDE {}` means no sources; `EXCLUDE {}` means all sources, equivalent to traditional MLDv1 group membership.
- Router state aggregates the interests of **all listeners on the link** and uses source and filter timers.
- IPv6 SSM uses **MLDv2 INCLUDE-mode `(S,G)` signaling**.
- `ipv6 mld join-group ... include <source>` makes the IOS XE device a listener; `ipv6 mld static-group` establishes downstream forwarding interest.
- `source-list` supports multiple sources; repeated `join-group` commands for the same group do **not** accumulate sources.
- **MLDv2 source filtering is not the same feature as `ipv6 mld access-group` admission filtering.**
