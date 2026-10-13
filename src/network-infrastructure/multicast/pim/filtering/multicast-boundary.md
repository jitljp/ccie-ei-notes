
# Multicast Boundaries and Administrative Scope

A **multicast boundary** restricts the propagation of multicast traffic between different parts of a network.

Multicast boundaries are commonly used to:

- Prevent multicast traffic from entering or leaving an administrative domain.
- Allow the same multicast group addresses to be reused in separate domains.
- Restrict particular multicast groups or `(S,G)` channels.
- Prevent unwanted multicast routing state and associated control-plane signaling.
- Prevent hosts on particular interfaces from acting as multicast sources.

Cisco IOS XE supports multicast boundaries for both IPv4 and IPv6, but the configuration mechanisms differ.

| Feature | IPv4 | IPv6 |
|---|---|---|
| Administrative scoping | `239.0.0.0/8` | Scope field within the multicast address |
| ACL-based interface boundary | `ip multicast boundary <acl>` | Not supported by `ipv6 multicast boundary` |
| Directional `(S,G)` filtering | `in` / `out` with extended ACLs | Use other mechanisms, such as `ipv6 multicast group-range` |
| Block incoming source traffic | `ip multicast boundary block source` | `ipv6 multicast boundary block source` |
| Scope-based boundary | ACL-based | `ipv6 multicast boundary scope` |
| Auto-RP group-range filtering | `filter-autorp` | Not applicable |
| BSR border | `ip pim bsr-border` | `ipv6 pim bsr border` |

## IPv4 Administrative Scope

RFC 2365 reserves **`239.0.0.0/8`** for administratively scoped IPv4 multicast.

Addresses in this range are intended for multicast communication within defined administrative regions.

Unlike globally scoped multicast addresses, administratively scoped addresses can be reused in separate domains.

RFC 2365 defines several address ranges:

| Range | Purpose |
|---|---|
| `239.0.0.0/8` | Entire administratively scoped multicast range |
| `239.255.0.0/16` | IPv4 Local Scope |
| `239.192.0.0/14` | Organization-Local Scope |

For example, two organizations can independently use `239.1.1.1`, provided multicast traffic for that group does not cross between their domains.

### Administrative Scope Boundaries

Consider the following topology:

```text
          Administrative Domain A
                  Source A
                 239.1.1.1
                     |
                    R1
                     |
                Ethernet0/0
                     |
             Multicast Boundary
                     X
                Ethernet0/0
                     |
                    R2
                     |
                  Source B
                 239.1.1.1
          Administrative Domain B
```

Both domains use `239.1.1.1`.

The multicast boundary prevents traffic for this group from crossing between the domains.

Without the boundary, PIM signaling could establish multicast forwarding state across the interconnection, allowing multicast traffic from one domain to reach the other.

**Administrative scope is not automatically enforced simply because an IPv4 group belongs to `239.0.0.0/8`.**

Routers must be configured to enforce the intended boundaries.

## IPv4 Multicast Boundary Configuration

IOS XE supports two main forms of the IPv4 multicast boundary command:

```text
ip multicast boundary <acl> [in | out | filter-autorp]
ip multicast boundary block source
```

Both are interface configuration commands, but they serve different purposes.

- **ACL-based boundaries** restrict particular multicast groups or `(S,G)` channels.
- **`block source`** blocks ordinary incoming multicast source traffic without requiring an ACL.

For ACL-based boundaries, the optional keywords are:

| Option | Behavior |
|---|---|
| No keyword | Bidirectional multicast boundary for the groups denied by a standard ACL |
| `in` | Filters incoming multicast source traffic and associated state |
| `out` | Filters outgoing multicast forwarding and associated downstream state |
| `filter-autorp` | Bidirectional boundary that additionally filters group-range information within Auto-RP messages |

**The ACL-based commands affect both multicast data forwarding and associated control-plane operations.**

The `filter-autorp` keyword is not required for multicast control-plane filtering in general. It adds specific processing of Auto-RP announcement and discovery information.

### ACL Matching

An ACL-based multicast boundary uses the following logic:

- **`permit`:** The group or channel is allowed by the boundary.
- **`deny`:** The group or channel is blocked by the boundary.
- **Implicit deny:** Groups or channels that do not match a permit entry are also blocked.

For a standard ACL, the address being matched is the **multicast group address**, not the unicast source address of the packet.

> **Important:** To block only a particular multicast range while allowing other groups, configure an explicit `permit any` at the end of the standard ACL.

### Basic Bidirectional Boundary

Suppose R1 connects two multicast domains through `Ethernet0/0`.

The goal is to prevent multicast traffic for `239.1.0.0/16` from crossing the boundary in either direction.

Configure a standard ACL:

```text
R1(config)# ip access-list standard MCAST-BOUNDARY
R1(config-std-nacl)# deny 239.1.0.0 0.0.255.255
R1(config-std-nacl)# permit any
```

Apply it:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip multicast boundary MCAST-BOUNDARY
```

The result is:

| Multicast Group | Result |
|---|---|
| `239.1.1.1` | Blocked |
| `239.1.2.2` | Blocked |
| `239.2.1.1` | Permitted |
| `232.1.1.1` | Permitted |

For denied groups, the boundary prevents multicast data from crossing the interface in either direction.

It also restricts the related multicast forwarding state and PIM/IGMP operations that would establish forwarding across the interface.

For example, a downstream PIM Join for a denied group cannot establish a permitted outgoing branch across the boundary.

**The boundary is not merely a packet filter applied after multicast forwarding state has been created.**

However, it does not indiscriminately block all PIM or IGMP messages. Multicast boundary filtering is associated with the groups and channels identified by the ACL.

### Block the Entire Administratively Scoped Range

To block IPv4 administratively scoped multicast traffic:

```text
R1(config)# ip access-list standard ADMIN-SCOPE
R1(config-std-nacl)# deny 239.0.0.0 0.255.255.255
R1(config-std-nacl)# permit any

R1(config)# interface Ethernet0/0
R1(config-if)# ip multicast boundary ADMIN-SCOPE
```

This prevents multicast traffic addressed to `239.0.0.0/8` from crossing the interface through multicast routing.

Other groups are not denied by this ACL.

The command is not restricted to `239.0.0.0/8`. Other IPv4 multicast group ranges can also be filtered.

## IPv4 Directional Multicast Boundaries

The `in` and `out` options allow multicast boundaries to be applied in a particular direction.

They also support **extended ACLs**, allowing the router to filter individual `(S,G)` channels.

### Inbound Boundary

Configure an inbound boundary:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip multicast boundary MCAST-BOUNDARY in
```

The `in` keyword filters multicast traffic arriving at the router through the interface.

For denied groups or channels:

- Incoming multicast source traffic is blocked.
- The source/channel is not accepted for multicast forwarding through that interface.
- Associated multicast routing operations are restricted.

This is useful on interfaces facing multicast sources or upstream multicast routers.

For example:

```text
Source --- Ethernet0/0 [R1] --- Receivers
                  |
           Inbound boundary
```

The boundary prevents denied multicast source traffic from entering R1 through `Ethernet0/0`.

It does not automatically prevent the same multicast group from being received through another permitted interface.

### Outbound Boundary

Configure an outbound boundary:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip multicast boundary MCAST-BOUNDARY out
```

The `out` keyword filters multicast forwarding and associated state toward an interface.

For denied groups or channels, the boundary can prevent:

- Downstream PIM Join processing that would establish forwarding through the interface.
- IGMP membership processing from establishing permitted outgoing multicast state.
- The interface from becoming a forwarding OIL member for the denied traffic.
- Multicast data forwarding through the interface.

Consider:

```text
Source --- [R1] Ethernet0/0 --- Receivers
                    |
             Outbound boundary
```

The boundary prevents denied multicast groups or channels from being forwarded to receivers through `Ethernet0/0`.

**The `out` keyword is particularly useful when restricting which multicast channels downstream receivers may receive.**

### Directional Comparison

| Behavior | `in` | `out` |
|---|---|---|
| Traffic direction | Toward router | Away from router |
| Common placement | Source-facing interface | Receiver-facing interface |
| Primary purpose | Restrict accepted multicast sources/channels | Restrict forwarded multicast groups/channels |
| Control-plane impact | Incoming source/channel state | Downstream Join/IGMP-derived forwarding state |
| ACL types | Standard or extended | Standard or extended |

The keywords refer to the direction relative to the interface on the router where the boundary is configured.

They are not PIM Join/Prune message direction selectors.

## IPv4 Source-Group Filtering

Standard ACLs identify multicast groups without distinguishing between sources.

To restrict individual **`(S,G)` channels**, use an extended ACL with `in` or `out`.

Suppose two sources transmit to the same SSM group:

```text
(10.1.1.10, 232.1.1.1)
(10.1.1.20, 232.1.1.1)
```

The goal is to block the first channel while permitting the second.

Configure:

```text
R1(config)# ip access-list extended SSM-BOUNDARY
R1(config-ext-nacl)# deny ip host 10.1.1.10 host 232.1.1.1
R1(config-ext-nacl)# permit ip any any

R1(config)# interface Ethernet0/0
R1(config-if)# ip multicast boundary SSM-BOUNDARY in
```

The extended ACL fields have the following meaning:

| Field | Meaning |
|---|---|
| Source IPv4 address | Multicast source `(S)` |
| Destination IPv4 address | Multicast group `(G)` |
| Protocol | Protocol selector used by multicast boundary processing |

The result is:

| Channel | Result |
|---|---|
| `(10.1.1.10, 232.1.1.1)` | Blocked |
| `(10.1.1.20, 232.1.1.1)` | Permitted |
| `(10.1.1.10, 232.1.1.2)` | Permitted |

This applies channel-based filtering without blocking the entire multicast group.

### Extended ACL Protocol Matching

The protocol field of an extended ACL is significant.

Cisco's multicast boundary implementation supports protocol-specific entries, including `ip`, `udp`, `pim`, and `igmp`.

For example:

```text
deny ip host 10.1.1.10 host 232.1.1.1
```

This is the straightforward choice when intending to block the entire `(S,G)` channel rather than creating separate protocol-specific filtering policies.

Cisco documents that the protocol field is parsed for multicast control-plane consistency. However, an `(S,G)` operation is consistently blocked only when the applicable ACL policy denies that channel for all relevant protocols.

Therefore, **do not assume that a protocol-specific denial is equivalent to `deny ip` for the entire `(S,G)` channel**.

The effects of protocol-specific ACL entries should be verified on the target IOS XE platform.

> **Important:** Extended ACLs are supported with `in` and `out`. A boundary with no directional keyword, or one using `filter-autorp`, requires a standard ACL.

## IPv4 `block source`

In addition to ACL-based filtering, IOS XE supports:

```text
ip multicast boundary block source
```

This command blocks ordinary incoming multicast source traffic on the interface, regardless of its multicast group or source address.

No ACL is required.

### Configuration

For example:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip multicast boundary block source
```

This is particularly useful on **receiver-facing interfaces** where hosts should be allowed to receive multicast traffic but should not act as multicast sources.

Consider:

```text
            Multicast Network
                   |
                  R1
                   |
              Ethernet0/0
                   |
             Receiver LAN
              /       \
            PC1       PC2
```

Suppose PC1 and PC2 should only receive multicast streams.

Configuring `ip multicast boundary block source` on R1's `Ethernet0/0` prevents ordinary multicast source traffic originating from the LAN from being accepted for multicast forwarding.

However, R1 can still forward multicast traffic toward PC1 and PC2.

### Behavior

| Operation | Behavior |
|---|---|
| Incoming multicast source traffic | Blocked |
| Outgoing multicast data traffic | Permitted |
| Incoming reserved multicast control packets | Permitted as required for protocol operation |
| Requires an ACL | No |
| Filters selected groups only | No |

For example:

```text
PC1 --> R1: Multicast data to 239.1.1.1
           BLOCKED

R1 --> PC1: Multicast data to 239.1.1.1
           PERMITTED
```

**`block source` is not equivalent to an ACL-based bidirectional multicast boundary.**

It is a general inbound source restriction.

### Caveats

- It blocks ordinary incoming multicast source traffic, not all multicast-related packets indiscriminately.
- Reserved multicast packets used for protocol operation are exempted as appropriate.
- It does not prevent receivers from receiving multicast traffic forwarded out the interface.
- It does not identify individual `(S,G)` channels using an ACL.
- It is not a replacement for a multicast boundary that separates administrative domains in both directions.

Use an ACL-based boundary when selective filtering is required.

## Multiple IPv4 Boundaries on an Interface

IOS XE supports multiple ACL-based multicast boundary configurations on the same interface.

Cisco documents the following limits:

- One boundary with `in`.
- One boundary with `out`.
- One boundary with no keyword **or** `filter-autorp`.

Therefore, a maximum of three ACL-based boundary instances can be configured.

For example:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip multicast boundary SOURCE-FILTER in
R1(config-if)# ip multicast boundary RECEIVER-FILTER out
R1(config-if)# ip multicast boundary ADMIN-SCOPE filter-autorp
```

Each boundary can reference a different ACL.

The rules are **cumulative**. Multiple applicable boundary policies are evaluated rather than one replacing all the others.

A permitted match in one policy should not be treated as a blanket override of a denial in another applicable boundary.

Cisco specifically documents that a boundary without a directional keyword and an inbound boundary are both evaluated for the incoming direction.

### Important Restrictions

| Combination | Supported? |
|---|---|
| Standard ACL without keyword | Yes |
| Extended ACL without keyword | No |
| Standard ACL with `in` | Yes |
| Extended ACL with `in` | Yes |
| Standard ACL with `out` | Yes |
| Extended ACL with `out` | Yes |
| Standard ACL with `filter-autorp` | Yes |
| Extended ACL with `filter-autorp` | No |
| Separate `in` and `out` boundaries | Yes |
| No-keyword and `filter-autorp` instances simultaneously | No, they occupy the same ACL-based instance category |

The `block source` form is a separate source-blocking function, not another ACL direction keyword.

Feature interactions and configuration limits can vary with IOS XE platform support.

## Auto-RP Filtering at Multicast Boundaries

Auto-RP uses two well-known multicast addresses:

| Group | Function |
|---|---|
| `224.0.1.39` | RP-Announce |
| `224.0.1.40` | RP-Discovery |

An ordinary multicast boundary can block packets addressed to these groups if the ACL denies them.

However, that is different from examining the **group-to-RP mappings carried inside Auto-RP messages**.

The `filter-autorp` keyword adds this capability.

### Configuration

For example:

```text
R1(config)# ip access-list standard RP-BOUNDARY
R1(config-std-nacl)# deny 239.1.0.0 0.0.255.255
R1(config-std-nacl)# permit any

R1(config)# interface Ethernet0/0
R1(config-if)# ip multicast boundary RP-BOUNDARY filter-autorp
```

This configuration:

1. Enforces a bidirectional multicast boundary for groups denied by the ACL.
2. Examines Auto-RP RP-Announce and RP-Discovery messages.
3. Removes advertised group-to-RP ranges that are not fully permitted by the boundary ACL.

The Auto-RP message itself may be forwarded with its permitted mappings retained.

**`filter-autorp` does not merely block packets addressed to `224.0.1.39` or `224.0.1.40`. It examines the multicast group ranges being advertised within them.**

### Entire Advertised Ranges Are Evaluated

Suppose an Auto-RP message advertises:

```text
RP:          3.3.3.3
Group Range: 239.1.0.0/16
```

However, the boundary ACL denies:

```text
239.1.1.0/24
```

The advertised `/16` includes addresses denied by the ACL.

Therefore, **the entire `/16` mapping is removed**, even though most addresses within that range are permitted.

IOS XE does not automatically split the advertised range into smaller permitted ranges.

A range is accepted only if all addresses within that advertised range are permitted.

### Blocking Auto-RP Packets Entirely

If the goal is to prevent Auto-RP control packets themselves from crossing the boundary, deny their destination multicast groups.

For example:

```text
R1(config)# ip access-list standard BLOCK-AUTORP
R1(config-std-nacl)# deny host 224.0.1.39
R1(config-std-nacl)# deny host 224.0.1.40
R1(config-std-nacl)# permit any

R1(config)# interface Ethernet0/0
R1(config-if)# ip multicast boundary BLOCK-AUTORP
```

This blocks the two Auto-RP multicast groups at the boundary rather than selectively removing advertised group ranges.

### Auto-RP Filtering Comparison

| Goal | Mechanism |
|---|---|
| Block multicast data for specified groups | `ip multicast boundary <acl>` |
| Block Auto-RP packets by their destination group | ACL denying `224.0.1.39` and `224.0.1.40` |
| Filter advertised group-to-RP ranges within Auto-RP messages | `ip multicast boundary <acl> filter-autorp` |
| Reject Candidate RP advertisements at a Mapping Agent | `ip pim rp-announce-filter` |

For more information about filtering Candidate RP advertisements at the Mapping Agent, see [Auto-RP Announcement Filtering](rp-announcement-filter.md).

## IPv6 Multicast Administrative Scope

Unlike IPv4, IPv6 incorporates multicast scope directly into the multicast address.

IPv6 multicast addresses use `FF00::/8`.

The multicast address format includes a **4-bit Scope field**:

```text
| 8 bits | 4 bits | 4 bits |       112 bits       |
+--------+--------+--------+----------------------+
|  0xFF  | Flags  | Scope  |       Group ID       |
+--------+--------+--------+----------------------+
```

The Scope field determines the intended reach of the multicast group.

Common values are:

| Scope | Name | Example |
|---|---|---|
| `1` | Interface-Local | `FF11::101` |
| `2` | Link-Local | `FF12::101` |
| `3` | Realm-Local | `FF13::101` |
| `4` | Admin-Local | `FF14::101` |
| `5` | Site-Local | `FF15::101` |
| `8` | Organization-Local | `FF18::101` |
| `E` | Global | `FF1E::101` |

The scope is represented by the low-order four bits of the second octet.

For example:

```text
FF15::101
  |
  +-- Flags = 1
      Scope = 5 (Site-Local)
```

IPv6 distinguishes the intended scope of a multicast address directly, whereas IPv4 administrative scoping relies on reserved group ranges and explicitly configured boundaries.

> **Note:** The IPv6 Scope field identifies a scope category, not a unique administrative zone. The actual boundaries of site-local, organization-local, and other administratively defined zones must still be enforced by the network.

## IPv6 Multicast Boundary Configuration

IOS XE supports two primary forms of the IPv6 multicast boundary command:

```text
ipv6 multicast boundary scope <scope-value>
ipv6 multicast boundary block source
```

Both are interface configuration commands.

| Command | Purpose |
|---|---|
| `scope` | Enforces a multicast boundary based on the IPv6 multicast scope field |
| `block source` | Blocks ordinary incoming multicast source traffic |

Unlike IPv4 `ip multicast boundary <acl>`, neither command accepts an ACL for selectively permitting or denying arbitrary `(S,G)` channels.

For more granular IPv6 group/channel filtering, see [IPv6 Multicast Group-Range Filtering](ipv6-group-range.md).

## IPv6 Scope Boundaries

The command is:

```text
ipv6 multicast boundary scope <scope-value>
```

The scope value may be specified numerically or using supported named scope values.

Cisco documents numeric scope values from **2 through 15**, as well as named scopes such as Link-Local, Admin-Local, Site-Local, and Organization-Local.

### Configuration

For example:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 multicast boundary scope 5
```

**The boundary blocks multicast groups whose scope values are less than or equal to the configured value.**

For example:

| Group | Scope | Boundary Result |
|---|---|---|
| `FF12::101` | 2, Link-Local | Blocked |
| `FF14::101` | 4, Admin-Local | Blocked |
| `FF15::101` | 5, Site-Local | Blocked |
| `FF18::101` | 8, Organization-Local | Permitted |
| `FF1E::101` | E, Global | Permitted |

These results describe the scope boundary comparison. Link-local multicast is inherently restricted to the local link even without an administratively configured scope boundary.

### IPv6 Scope Boundary Example

Consider the following topology:

```text
               Site A
                 |
                R1
                 |
            Ethernet0/0
                 |
         IPv6 Scope Boundary
              Scope 5
                 X
            Ethernet0/0
                 |
                R2
                 |
               Site B
```

Both sites use Site-Local multicast groups.

Configure the boundary on R1:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 multicast boundary scope 5
```

Site-Local multicast groups cannot cross the configured interface boundary.

However, Organization-Local and Global multicast groups are not blocked by this scope comparison.

### Control-Plane and Data-Plane Effects

The IPv6 scope boundary affects both multicast packet forwarding and associated multicast protocol signaling.

For groups whose scope is less than or equal to the configured boundary, the behavior depends on the role of the interface.

**If the interface is the RPF interface:**

- Multicast packets for the denied scopes are not accepted.
- PIM Join/Prune messages for those groups are not sent on the RPF interface.
- The MRIB does not mark the interface with the `Accept` flag for the denied multicast state.

**If the interface is an outgoing interface:**

- Multicast packets for the denied scopes are not forwarded through the interface.
- Received PIM Join/Prune messages for the denied groups are not processed to establish forwarding through the interface.

The scope boundary also filters applicable PIM Register and BSR messages.

Therefore, the command can enforce both multicast forwarding boundaries and scoped RP discovery boundaries.

### Caveats

- The command filters by scope value, not by an individual multicast group address.
- Specifying scope `5` blocks all groups with scope values of `5` or lower, not only Site-Local groups.
- A numeric scope boundary may affect several scope categories simultaneously.
- The Scope field does not automatically enforce site or organization boundaries.
- An IPv6 scope boundary is not equivalent to an IPv4 extended-ACL `(S,G)` boundary.
- Additional multicast policies may be needed when filtering specific groups or sources within the same scope.

## IPv6 `block source`

IOS XE also supports:

```text
ipv6 multicast boundary block source
```

This blocks ordinary incoming multicast source traffic on the interface while permitting multicast traffic to be forwarded out the interface.

### Configuration

For example:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 multicast boundary block source
```

This is useful on receiver-facing interfaces where hosts should receive multicast traffic but should not originate multicast streams.

### Behavior

| Operation | Behavior |
|---|---|
| Incoming multicast source traffic | Blocked |
| Outgoing multicast data traffic | Permitted |
| Incoming reserved multicast packets | Permitted |
| Requires an ACL | No |
| Filters by scope value | No |

Cisco explicitly documents that reserved multicast packets are still permitted to enter the interface.

This is important because IPv6 uses multicast for essential control-plane functions, such as Neighbor Discovery and certain routing protocol operations.

The command is intended to prevent hosts from functioning as multicast sources, not to disable IPv6 multicast-based control protocols on the link.

### `scope` vs. `block source`

| Feature | `scope` | `block source` |
|---|---|---|
| Filtering criterion | IPv6 multicast scope value | Incoming multicast source traffic |
| Inbound data filtering | Yes, for denied scopes | Yes |
| Outbound data filtering | Yes, for denied scopes | No |
| Associated PIM signaling restrictions | Yes | Does not generally prevent reserved protocol signaling |
| Common use | Separate multicast scope zones | Restrict sources on receiver-facing interfaces |

The two commands solve different problems.

A scope boundary controls which multicast groups may cross an administrative boundary.

A source-blocking boundary controls whether multicast source traffic may enter through a particular interface.

## IPv4 vs. IPv6 Source Blocking

Both address families support the same source-blocking concept.

**IPv4:**

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip multicast boundary block source
```

**IPv6:**

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 multicast boundary block source
```

In both cases:

- Ordinary incoming multicast source traffic is blocked.
- Outgoing multicast forwarding is permitted.
- Reserved multicast traffic required for protocol operation is not indiscriminately blocked.
- No ACL is required.

These commands are useful on interfaces where connected hosts should function only as multicast receivers.

They are **not equivalent to disabling multicast routing or PIM on the interface**.

## PIM BSR Borders

Multicast forwarding boundaries are different from **PIM BSR borders**.

BSR distributes group-to-RP mappings using PIM Bootstrap Messages (BSMs).

A BSR border prevents BSMs from crossing an interface, allowing separate PIM domains to maintain independent RP discovery information.

### IPv4 BSR Border

Configure:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip pim bsr-border
```

The router neither sends nor accepts IPv4 BSMs through the interface.

### IPv6 BSR Border

Configure:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 pim bsr border
```

This prevents IPv6 BSMs from crossing the interface regardless of their advertised scope.

IPv6 also supports scoped BSR operation, in which different RPs and BSRs serve different multicast scope zones.

The `ipv6 multicast boundary scope` command can help enforce such zones by filtering multicast forwarding and applicable BSR signaling.

### BSR Border vs. Multicast Boundary

| Feature | Multicast Boundary | BSR Border |
|---|---|---|
| Restricts ordinary multicast data | Yes | No |
| Restricts group/channel forwarding state | Yes | No |
| Filters BSMs | IPv6 scope boundaries can filter applicable scoped BSMs | Yes |
| Primary purpose | Control multicast forwarding and related signaling | Separate BSR discovery domains |

A BSR border alone does not prevent multicast data traffic from crossing an interface.

A multicast boundary alone does not necessarily prevent every form of RP discovery information from crossing a domain boundary.

Therefore, multiple mechanisms may be necessary to fully separate multicast administrative domains.

## IOS XE Verification

### IPv4 Boundary Verification

Check the interface configuration:

```text
R1# show running-config interface Ethernet0/0
```

Inspect configured ACLs:

```text
R1# show ip access-lists
```

Check whether a particular `(S,G)` is blocked by an inbound boundary:

```text
R1# show ip pim boundary Ethernet0/0 10.1.1.10 232.1.1.1 in
```

Check the outbound boundary:

```text
R1# show ip pim boundary Ethernet0/0 10.1.1.10 232.1.1.1 out
```

These commands report whether the specified channel is blocked or unblocked by the applicable interface boundary.

Other useful commands include:

```text
R1# show ip mroute
R1# show ip pim rp mapping
R1# show ip pim bsr-router
R1# show ip pim autorp
```

Use them to verify multicast routing state, OIL membership, and RP discovery information.

For `block source`, confirm the interface configuration and verify actual ingress/egress behavior using multicast test traffic.

A normal multicast forwarding table does not necessarily provide a separate indicator for every packet dropped by a source-blocking boundary.

### IPv6 Boundary Verification

Check the interface configuration:

```text
R1# show running-config interface Ethernet0/0
```

Inspect multicast state:

```text
R1# show ipv6 mroute
R1# show ipv6 mrib route
R1# show ipv6 pim topology
```

The `show ipv6 mrib route` command is particularly useful for scope boundaries.

If the scope boundary prevents multicast traffic from being accepted on an RPF interface, the interface does not have the MRIB `Accept` flag for that denied multicast state.

Verify IPv6 BSR information:

```text
R1# show ipv6 pim bsr election
R1# show ipv6 pim bsr rp-cache
```

Use these commands to determine whether the expected BSR and Candidate RP information is present.

For `block source`, verify the configured command and test that incoming source traffic is blocked while outbound multicast forwarding remains permitted.

## Removing Multicast Boundaries

### IPv4

Remove ACL-based boundary configurations using the corresponding `no` forms:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# no ip multicast boundary MCAST-BOUNDARY
```

For directional boundaries, specify the configured option when required by the parser:

```text
R1(config-if)# no ip multicast boundary SOURCE-FILTER in
R1(config-if)# no ip multicast boundary RECEIVER-FILTER out
R1(config-if)# no ip multicast boundary RP-BOUNDARY filter-autorp
```

Remove source blocking:

```text
R1(config-if)# no ip multicast boundary block source
```

### IPv6

Remove a scope boundary:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# no ipv6 multicast boundary scope 5
```

Remove source blocking:

```text
R1(config-if)# no ipv6 multicast boundary block source
```

### BSR Borders

Remove the IPv4 BSR border:

```text
R1(config-if)# no ip pim bsr-border
```

Remove the IPv6 BSR border:

```text
R1(config-if)# no ipv6 pim bsr border
```

## Key Points

- IPv4 reserves **`239.0.0.0/8`** for administratively scoped multicast, but the intended boundaries must be enforced.
- **`ip multicast boundary <acl>`** creates an IPv4 multicast boundary affecting data forwarding and associated control-plane state.
- The default ACL-based boundary is bidirectional. The `in` and `out` keywords configure directional filtering.
- Standard ACLs filter multicast groups; extended ACLs can filter individual `(S,G)` channels with `in` or `out`.
- Multiple ACL-based boundaries can be configured on one interface, subject to the documented instance limits.
- **`filter-autorp`** additionally filters group-range information contained within Auto-RP messages. It does not merely enable generic control-plane filtering.
- **`ip multicast boundary block source`** blocks ordinary incoming IPv4 multicast source traffic while permitting outgoing multicast forwarding.
- IPv6 multicast addresses contain a **4-bit Scope field**.
- **`ipv6 multicast boundary scope <value>`** blocks multicast groups with scope values less than or equal to the configured value.
- IPv6 scope boundaries affect both multicast data forwarding and associated PIM signaling.
- **`ipv6 multicast boundary block source`** blocks ordinary incoming IPv6 multicast source traffic but permits outbound multicast and reserved control packets.
- **BSR borders** prevent PIM Bootstrap Messages from crossing an interface but do not directly block multicast data forwarding.
