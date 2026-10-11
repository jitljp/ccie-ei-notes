
# MLD Filtering and Limits

**MLD filtering** controls which IPv6 multicast groups and sources receivers are allowed to join.

**MLD limits** restrict the amount of multicast membership state that can be established.

These features are similar to the IPv4 mechanisms covered in the [IGMP Filter](../igmp-filter/overview.md) section.

The main distinction is between:

- **Layer 2 filtering:** Controls multicast memberships learned by MLD snooping on switch ports.
- **Layer 3 filtering:** Controls MLD memberships accepted by multicast routers or Layer 3 switches.

Cisco IOS XE provides MLD access control and state limits on routed interfaces. Layer 2 filtering and throttling capabilities depend on the switch platform.

## MLD Filtering on Routed Interfaces

The `ipv6 mld access-group` command applies an IPv6 ACL to MLD membership processing on a routed interface.

It is the IPv6 equivalent of:

```text
ip igmp access-group
```

The syntax is:

```text
ipv6 mld access-group <acl-name>
```

This feature supports:

- Filtering multicast group memberships `(G)`
- Filtering MLDv2 source/group memberships `(S,G)`
- Restricting which multicast state receivers can establish

### Filtering Multicast Groups

Suppose R1 should accept listener memberships for `ff15::1234` but reject other multicast groups.

Configure an IPv6 ACL:

```text
R1(config)# ipv6 access-list MLD-FILTER
R1(config-ipv6-acl)# permit ipv6 any host ff15::1234
```

Apply it to the receiver-facing interface:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 mld access-group MLD-FILTER
```

The resulting policy is:

| Multicast Group | Result |
|---|---|
| `ff15::1234` | Permit |
| `ff15::1235` | Deny |
| `ff15::1236` | Deny |

The ACL's implicit deny rejects membership requests for other groups.

To block only one group while allowing others:

```text
R1(config)# ipv6 access-list MLD-FILTER
R1(config-ipv6-acl)# deny ipv6 any host ff15::1234
R1(config-ipv6-acl)# permit ipv6 any any
```

The ACL is evaluated against MLD membership information, rather than being applied as an ordinary data-plane IPv6 ACL.

### Filtering MLDv2 Sources

MLDv2 also supports source-specific filtering.

This is equivalent to using an extended ACL with `ip igmp access-group`.

For example, suppose only source `2001:db8:1::10` should be permitted for multicast group `ff3e::1234`.

Cisco documents a **two-stage ACL evaluation** for MLDv2 source-specific Reports:

1. The router first checks the synthetic group-level tuple `(0,G)`.
2. If that check succeeds, it checks each requested `(S,G)` channel.

For IPv6, the group-level check uses the unspecified source address `::`.

Configure:

```text
R1(config)# ipv6 access-list MLD-SSM-FILTER
R1(config-ipv6-acl)# permit ipv6 host :: host ff3e::1234
R1(config-ipv6-acl)# permit ipv6 host 2001:db8:1::10 host ff3e::1234
```

Apply it:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 mld access-group MLD-SSM-FILTER
```

The intended ACL evaluation is:

| Lookup | Result |
|---|---|
| `( :: , ff3e::1234)` | Permit |
| `(2001:db8:1::10, ff3e::1234)` | Permit |
| `(2001:db8:1::20, ff3e::1234)` | Deny |

The first ACE permits the preliminary group-level lookup.

The second ACE permits the specific multicast source.

The implicit deny rejects other sources.

**Important:** The source address in these ACL lookups represents the **multicast source**, not the IPv6 address of the host sending the MLD Report.

This is the same important distinction covered in the [IGMP Filtering on Routed Interfaces](../igmp-filter/routed-interfaces.md) section.

> `ipv6 mld access-group` controls accepted listener state. It does not directly filter multicast data packets, and it cannot independently distinguish two receivers on the same interface requesting identical `(S,G)` channels.

## MLD State Limits on Routed Interfaces

Cisco IOS XE can limit **multicast state created through MLD listener memberships**, similar to the `ip igmp limit` feature covered in [IGMP Filtering on Routed Interfaces](../igmp-filter/routed-interfaces.md).

| Command | Scope |
|---|---|
| `ipv6 mld limit` | Per routed interface |
| `ipv6 mld state-limit` | Global or VRF-specific |

These commands limit **accounted MLD-derived multicast state**, not the number of MLD Reports, individual receivers, or all multicast routing entries.

### What Counts Toward the Limit?

MLD can account for group-level `(*,G)` state or source-specific `(S,G)` state, depending on the multicast group:

| Membership type | State counted toward the limit |
|---|---|
| **ASM** | Generally one state per multicast group, even when MLDv2 source lists result in multiple `(S,G)` forwarding entries |
| **SSM** | One state per admitted `(S,G)` channel |

For example, a limit of `2` can allow one ASM group with three requested sources, but permits only two individually accounted SSM channels.

**The number of entries in `show ipv6 mroute` does not necessarily match the number of MLD states counted against the limit.** Multicast routing entries created independently of MLD are not covered by these commands either.

### Per-Interface Limit

Configure the limit on the receiver-facing routed interface:

```text
R1(config)# interface Ethernet0/1
R1(config-if)# ipv6 mld limit 2
```

The interface can admit up to **two counted MLD states**. If accepting a new group or source would exceed the limit, IOS XE rejects the additional membership state rather than replacing existing state.

The supported range is **1 to 64000**. No user-configured limit is imposed by default.

#### Excluding Groups from the Limit

Use the optional `except` ACL to exempt selected multicast groups or `(S,G)` channels from the MLD state limit:

```text
R1(config)# ipv6 access-list MLD-EXCEPTIONS
R1(config-ipv6-acl)# permit ipv6 any host ff15::1234
!
R1(config)# interface Ethernet0/1
R1(config-if)# ipv6 mld limit 2 except MLD-EXCEPTIONS
```

The ACL determines which memberships count toward the limit:

- **Permit:** The membership is exempt and does not count toward the limit.
- **Deny:** The membership counts toward the limit and is rejected only if the limit has been reached.

In this example, `ff15::1234` is exempt from the limit of two states. Other multicast memberships remain subject to the limit.

This controls **limit accounting**, not whether the membership itself is permitted. Use `ipv6 mld access-group` for membership filtering.

> Groups or channels exempt from **interface-level** accounting still count toward the **global limit** (`ipv6 mld state-limit`).
> This is the same concept as interface- and global-level `ip igmp limit` in IPv4.

### Global and VRF Limits

Configure a global limit:

```text
R1(config)# ipv6 mld state-limit 500
```

Or a VRF-specific limit:

```text
R1(config)# ipv6 mld vrf BLUE state-limit 100
```

Global, VRF-specific, and per-interface limits restrict their respective scopes. A membership must stay within every applicable limit.

> Unlike IGMP, which uses `ip igmp limit` for both global and interface-level limits, MLD uses `ipv6 mld state-limit` globally and `ipv6 mld limit` on interfaces.

### Verification

```text
R1# show ipv6 mld interface Ethernet0/1
R1# show ipv6 mld groups detail
R1# show ipv6 mld groups summary
R1# show ipv6 mroute
```

- `show ipv6 mld interface` displays the configured limit and active state count.
- `show ipv6 mld groups detail` identifies **group-level accounting** with `State accounted` and **source-level accounting** with the `Ac` flag.
- `show ipv6 mld groups summary` and `show ipv6 mroute` display membership and routing entries, respectively, **not necessarily the number of states counted toward the limit**.

An exceeded limit can generate `%MLD_PROT-4-LIMIT_REACHED`.

For more information, see Cisco's [MLD Group Limits](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_ip6-mcast-mld-limits.html) documentation.

## Layer 2 MLD Filtering on Switches

Catalyst 9000 IOS XE switches support **MLD snooping**, allowing switches to dynamically learn IPv6 multicast listener ports.

Unlike IGMP snooping, MLD snooping is **globally disabled by default**, although it is enabled by default on individual VLANs.

Enable it globally:

```text
SW1(config)# ipv6 mld snooping
```

Verify:

```text
SW1# show ipv6 mld snooping
SW1# show ipv6 mld snooping vlan 10
SW1# show ipv6 mld snooping address
```

Unlike IGMP, Catalyst 9000 IOS XE does not document equivalent Layer 2 MLD profile filtering or per-port membership throttling features.

The MLD filtering and limit commands covered earlier in this section operate on **routed interfaces**, not Layer 2 switch ports.

Further details about Layer 2 multicast forwarding are covered in [MLD Snooping](mld-snooping.md).

## Filtering vs. Data-Plane Security

MLD membership filtering does not necessarily prevent multicast data from reaching a receiver through other forwarding mechanisms.

For example:

```text
MLD Report received
        |
        v
MLD membership filtering
        |
        v
Membership rejected
        |
        v
No dynamic listener state created
```

This prevents the rejected membership from establishing the corresponding dynamic multicast forwarding interest.

However, multicast packets could still be forwarded because of:

- Static multicast memberships
- Other permitted listeners
- Multicast flooding behavior
- Other forwarding configuration

Therefore, MLD filtering is primarily a **membership admission-control mechanism**.

It should not be treated as equivalent to an ordinary data-plane ACL.

## IGMP vs. MLD Filtering Commands

| Function | IPv4 / IGMP | IPv6 / MLD |
|---|---|---|
| Routed-interface membership filter | `ip igmp access-group` | `ipv6 mld access-group` |
| Per-interface state limit | `ip igmp limit` | `ipv6 mld limit` |
| Global state limit | `ip igmp limit` | `ipv6 mld state-limit` |
| Layer 2 profile definition | `ip igmp profile` | `ipv6 mld profile`* |
| Apply Layer 2 profile | `ip igmp filter` | `ipv6 mld filter`* |
| Layer 2 group throttling | `ip igmp max-groups` | `ipv6 mld max-groups`* |

\*The listed Layer 2 MLD profile and throttling commands are supported on certain Cisco Business switches, but are not documented as equivalent features on Catalyst 9000 IOS XE. They are not available in the IOS XE switches I tested.

The Layer 3 IOS XE commands are much more directly comparable across IGMP and MLD.

## Key Points

- **MLD filtering** controls IPv6 multicast listener memberships, similar to IGMP filtering.
- `ipv6 mld access-group` uses an IPv6 ACL to filter multicast groups and MLDv2 `(S,G)` channels on routed interfaces.
- Filtering with `ipv6 mld access-group` performs a preliminary `(0,G)` ACL lookup before evaluating individual `(S,G)` channels.
- `ipv6 mld limit` restricts **MLD-derived multicast states** on an interface, not Report packets, individual receivers, or all entries in the multicast routing table.
- The `except` ACL exempts permitted groups or channels from the **interface limit**, but they still count toward the global limit.
- **ASM** memberships generally count as one state per group, whereas **SSM** memberships count per admitted `(S,G)` channel.
- `ipv6 mld state-limit` configures global or VRF-specific MLD state limits.
- On Catalyst IOS XE, **MLD snooping is globally disabled by default**, unlike IGMP snooping.
- Catalyst 9000 IOS XE does not provide the same Layer 2 MLD profile filtering and per-port throttling features available for IGMP.
- MLD membership filtering controls listener state, not multicast data packets directly.

