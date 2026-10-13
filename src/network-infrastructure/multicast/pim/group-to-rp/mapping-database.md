# RP Mapping Database and Group Ranges

In **PIM Sparse Mode (PIM-SM)** using Any-Source Multicast (ASM), routers maintain **Group-to-RP mapping information** to determine which Rendezvous Point (RP) is responsible for a multicast group.

A Group-to-RP mapping associates an individual multicast group or a range of groups with an RP address.

Depending on the RP discovery mechanism, a router may maintain multiple mappings that match the same group.

This page examines how these mappings are represented, maintained, and verified on Cisco IOS XE.

## RP Mapping Database

The **RP Mapping Database** contains the Group-to-RP information available to a PIM router.

A mapping can contain the following information:

| Information | Purpose |
|---|---|
| **Multicast group range** | Identifies the groups covered by the mapping |
| **RP address** | Identifies the router acting as an RP |
| **Information source** | Indicates whether the mapping was configured statically or learned dynamically |
| **RP priority** | Used when selecting among BSR Candidate RPs |
| **Holdtime / expiration timer** | Determines how long a dynamically learned mapping remains valid without a refresh |
| **PIM mode** | Indicates whether a mapping applies to Sparse Mode or Bidirectional PIM, where supported |

Not every field is present for every type of mapping.

For example, statically configured RPs do not require advertisement Holdtimes, whereas BSR-learned mappings can contain RP priorities and expiration information.

### RP Mapping Database vs. Multicast Routing Table

The RP Mapping Database is distinct from the **multicast routing table (mroute table)**.

| RP Mapping Database | Multicast Routing Table |
|---|---|
| Associates multicast groups with RPs | Maintains multicast forwarding state |
| Contains multicast group ranges | Contains `(*,G)` and `(S,G)` entries |
| Used to determine the appropriate RP | Used to determine incoming and outgoing interfaces |
| Can contain entries for groups without active receivers or sources | Generally contains state created by multicast control-plane or forwarding activity |

For example, a router might know that `3.3.3.3` is the RP for `239.1.0.0/16` even though no receivers have joined any groups in that range.

In that case, the mapping exists, but the router does not necessarily have a corresponding `(*,G)` or `(S,G)` entry.

**An RP mapping does not indicate that multicast forwarding state exists.**

## Multicast Group Ranges

An RP can serve a single multicast group or an entire range of groups.

Group ranges are commonly represented using a network address and prefix length.

For example:

```text
239.1.0.0/16
```

This range includes:

```text
239.1.0.0 - 239.1.255.255
```

The prefix length determines how many bits of the multicast group address must match.

| Group Range | Matching Multicast Groups |
|---|---|
| `224.0.0.0/4` | Entire IPv4 multicast address space |
| `239.0.0.0/8` | `239.0.0.0` through `239.255.255.255` |
| `239.1.0.0/16` | `239.1.0.0` through `239.1.255.255` |
| `239.1.1.0/24` | `239.1.1.0` through `239.1.1.255` |
| `239.1.1.1/32` | Only `239.1.1.1` |

> **Note:** A group range can include addresses that are not used for ordinary ASM forwarding. For example, IPv4 SSM groups do not require an RP, and `224.0.0.0/24` contains link-local control groups that are not forwarded between routers.

### Group Ranges in Static RP Configuration

For IPv4 Static RP, IOS XE uses a standard IPv4 ACL to identify the multicast groups associated with an RP.

For example:

```text
R1(config)# ip access-list standard RP-GROUPS
R1(config-std-nacl)# permit 239.1.0.0 0.0.255.255

R1(config)# ip pim rp-address 3.3.3.3 RP-GROUPS
```

This associates RP `3.3.3.3` with:

```text
239.1.0.0/16
```

Multiple ACL entries can associate the same RP with multiple group ranges:

```text
R1(config)# ip access-list standard RP-GROUPS
R1(config-std-nacl)# permit 239.1.0.0 0.0.255.255
R1(config-std-nacl)# permit 239.2.0.0 0.0.255.255
```

The RP serves both ranges.

Unlike the multicast source address, the multicast **group address (G)** is what determines whether a static RP ACL matches.

### Group Ranges in BSR and Auto-RP

BSR and Auto-RP also advertise group ranges, but their mapping databases differ.

**BSR:**

- Candidate RPs advertise the group ranges they support.
- The elected BSR distributes this information in its RP-Set.
- Multiple Candidate RPs can be advertised for the same group range.
- Each receiving router independently selects an RP.

**Auto-RP:**

- Candidate RPs announce their supported group ranges.
- The Mapping Agent selects an RP for each advertised range.
- Receiving routers learn the selected mappings through RP-Discovery messages.

Therefore, **the mapping information available on a receiving router depends on the discovery mechanism**.

See [BSR Hashing](../rp-discovery/bsr/hashing.md) and [Mapping Agent and Discovery Messages](../rp-discovery/auto-rp/mapping-agent.md) for the respective selection mechanisms.

## Overlapping Group Ranges

Multiple RP mappings can match the same multicast group.

Consider the following entries:

| Group Range | RP |
|---|---|
| `239.0.0.0/8` | `1.1.1.1` |
| `239.1.0.0/16` | `2.2.2.2` |
| `239.1.1.0/24` | `3.3.3.3` |

For multicast group:

```text
239.1.1.10
```

All three ranges match.

Within the BSR RP selection algorithm, the longest matching group prefix is preferred. Therefore, if these entries belong to the same BSR RP-Set, the `/24` mapping is preferred over the `/16` and `/8` mappings.

However, **group-range specificity is not the only consideration when mappings come from different mechanisms**.

For example, the precedence of static and dynamically learned RP information can affect the final result.

The selection and precedence rules are covered in [RP Mapping Selection and Precedence](selection-precedence.md).

## Mapping Sources and Maintenance

Group-to-RP mappings can be installed through static configuration or learned through dynamic RP discovery.

### Static Mappings

Static mappings are maintained locally through configuration.

For example:

```text
R1(config)# ip pim rp-address 3.3.3.3
```

The mapping remains configured until it is modified or removed.

It does not require periodic RP advertisements or a mapping expiration timer.

**Static mappings do not automatically disappear when their RP becomes unreachable.**

### BSR-Learned Mappings

With BSR, routers receive an RP-Set containing Candidate RP information.

A typical RP-Set entry includes:

- Multicast group prefix
- Candidate RP address
- RP priority
- RP Holdtime

The router maintains the received information and uses it when determining the appropriate RP for a group.

BSR mappings are refreshed through Bootstrap Messages.

If a mapping is not refreshed before its expiration timer reaches zero, the router removes it.

A mapping can also be explicitly withdrawn using an advertised RP Holdtime of zero.

### Auto-RP-Learned Mappings

With Auto-RP, routers receive RP-Discovery messages from Mapping Agents.

These messages contain selected Group-to-RP mappings and their Holdtimes.

Receiving routers maintain the mappings as soft state.

If RP-Discovery messages stop arriving, the mappings eventually expire.

The Mapping Agent itself maintains Candidate RP information separately, based on received RP-Announce messages.

### Mapping Expiration vs. RP Reachability

Mapping expiration and RP reachability are separate mechanisms.

An RP mapping can remain valid even when its RP is unreachable through the unicast routing table.

Conversely, a perfectly reachable RP might not be selected if the router does not have a valid mapping for the multicast group.

| Condition | Result |
|---|---|
| Mapping exists and RP is reachable | Normal RP-dependent PIM-SM operation is possible |
| Mapping exists but RP is unreachable | Mapping can remain present, but shared-tree construction or source registration may fail |
| Dynamic mapping expires | Mapping is removed; RP selection must be reevaluated |
| Static RP becomes unreachable | Static mapping remains configured |
| No applicable RP mapping exists | RP-dependent Sparse Mode ASM operation cannot proceed normally |

On interfaces using `ip pim sparse-dense-mode`, missing RP information may instead trigger Dense Mode forwarding, depending on the applicable configuration.

## IOS XE Verification

### Display IPv4 RP Mappings

The primary verification command is:

```text
R1# show ip pim rp mapping
```

This displays the Group-to-RP mappings known to the router.

For example, a router learning a mapping through Auto-RP might display:

```text
R1# show ip pim rp mapping
PIM Group-to-RP Mappings

Group(s) 239.1.0.0/16
  RP 3.3.3.3 (?), v2v1
    Info source: 2.2.2.2 (?), elected via Auto-RP
         Uptime: 00:05:12, expires: 00:02:45
```

*Illustrative output; exact formatting and fields vary by IOS XE platform and release.*

Important fields:

| Field | Meaning |
|---|---|
| `Group(s)` | Multicast group range associated with the RP |
| `RP` | RP address for the mapping |
| `Info source` | Origin of the mapping information |
| `elected via Auto-RP` | Mapping was learned through Auto-RP |
| `Uptime` | Time since the mapping was installed |
| `expires` | Time remaining before the mapping expires without a refresh |
| `v2v1` | PIM version compatibility information |

With a statically configured RP, IOS XE identifies the entry as static rather than dynamically learned.

### Filter the RP Mapping Output

IOS XE also supports additional options on supported platforms and releases:

```text
R1# show ip pim rp mapping ?
```

Useful options include:

| Option | Purpose |
|---|---|
| `in-use` | Displays RP mappings currently in use |
| `elected` | Displays elected Auto-RP mappings |
| RP address | Filters mappings associated with a particular RP |

For example:

```text
R1# show ip pim rp mapping in-use
```

The available options can vary by IOS XE release.

### Determine the RP for a Specific Group

Use:

```text
R1# show ip pim rp 239.1.1.10
```

This checks the RP associated with multicast group `239.1.1.10`.

For BSR-selected RPs, the following command also displays RP hash information:

```text
R1# show ip pim rp-hash 239.1.1.10
```

This is particularly useful when multiple Candidate RPs advertise matching group ranges.

### Display IPv6 Group Mappings

IPv6 uses:

```text
R1# show ipv6 pim group-map
```

Unlike the IPv4 RP mapping display, the IPv6 group mapping table also contains information about multicast forwarding modes and group ranges that do not require RPs.

For example, entries can indicate:

- **SM:** PIM Sparse Mode with an RP
- **BIDIR:** Bidirectional PIM
- **SSM:** Source-Specific Multicast without an RP
- **LL:** Link-local multicast groups
- **NO / NOROUTE:** Nonroutable multicast ranges

Therefore, not every entry displayed by `show ipv6 pim group-map` represents a Group-to-RP mapping.

For an ASM group, a mapping can display an RP address, an information source, and an RPF path toward the RP.

### Filter IPv6 Mapping Sources

To display statically configured IPv6 mappings:

```text
R1# show ipv6 pim group-map info-source static
```

To display mappings learned through BSR:

```text
R1# show ipv6 pim group-map info-source bsr
```

To check a specific group:

```text
R1# show ipv6 pim group-map FF15::101
```

Cisco IOS XE may also display an asterisk (`*`) next to group ranges that are being used.

The `Groups` field indicates the number of multicast groups present in the topology table for the corresponding range, **not the number of possible multicast addresses covered by the prefix**.

### Verify RP Reachability

Once the RP is identified, check the RPF path toward it.

For IPv4:

```text
R1# show ip rpf 3.3.3.3
```

For IPv6:

```text
R1# show ipv6 rpf 2001:DB8:3::3
```

These commands help determine whether the router has a valid RPF path toward the RP.

A correct RP mapping without a valid RPF path can still prevent normal PIM-SM operation.

## Troubleshooting RP Mapping Problems

| Symptom | Possible Cause |
|---|---|
| Expected RP mapping missing | Static configuration missing, BSR advertisements not received, or Auto-RP discovery failure |
| Group not covered by an RP mapping | Incorrect ACL, group prefix, or advertised group range |
| Multiple mappings for one group | Overlapping group ranges or multiple RP discovery sources |
| Dynamic mapping disappears | Advertisements stopped, mapping expired, or mapping was withdrawn |
| Different routers have different mappings | Inconsistent static configuration, RP discovery propagation, or filtering |
| Mapping exists but multicast does not work | RP unreachable, RPF failure, PIM signaling issue, or another forwarding problem |
| IPv6 mapping has no RP address | Group may be classified as SSM, link-local, or another mode that does not use an RP |

When troubleshooting, first determine whether the expected mapping exists. Then verify which RP is selected and whether it is reachable.

## Key Points

- The **RP Mapping Database** maintains information associating multicast group ranges with Rendezvous Points.
- RP mapping information is separate from multicast forwarding state.
- Group ranges can cover individual groups or large blocks of multicast addresses.
- Multiple RP mappings can overlap and match the same multicast group.
- **Static mappings** persist until changed or removed from configuration.
- **Dynamic mappings** are refreshed periodically and can expire or be withdrawn.
- BSR distributes Candidate RP information, while Auto-RP distributes mappings selected by a Mapping Agent.
- `show ip pim rp mapping` displays known IPv4 RP mappings.
- `show ipv6 pim group-map` displays IPv6 group mappings, including modes that do not require an RP.
- A valid mapping does not guarantee RP reachability or successful multicast forwarding.

## References

- [RFC 7761: PIM Sparse Mode, Section 4.7.1](https://www.rfc-editor.org/rfc/rfc7761.html#section-4.7.1)
- [RFC 5059: Bootstrap Router Mechanism for PIM](https://www.rfc-editor.org/rfc/rfc5059.html)
- [Cisco IOS XE: Monitoring and Maintaining IP Multicast](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-multicast/ip-multicast/m_imc_maintain.html)
- [Cisco IOS XE: Configuring Basic IP Multicast](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-multicast/ip-multicast/m_imc_basic_cfg-0.html)
- [Cisco: IPv6 `show ipv6 pim group-map` Command Reference](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipv6/command/ipv6-cr-book/ipv6-s4.html)