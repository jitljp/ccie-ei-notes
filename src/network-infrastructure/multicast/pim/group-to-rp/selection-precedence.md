# RP Mapping Selection and Precedence

In PIM Sparse Mode (PIM-SM), multiple Rendezvous Points (RPs) can be associated with the same multicast group.

When a router has multiple matching **Group-to-RP mappings**, it must determine which RP to use for that group.

There are two distinct selection processes:

1. **RP selection for an advertised group range:** When multiple Candidate RPs advertise the same group range, a mechanism such as Auto-RP must determine which RP is associated with that range.
2. **Effective RP selection for a multicast group (G):** When multiple group ranges match a particular multicast group, the router must determine which mapping applies.

The exact rules depend on whether the mappings are configured statically or learned through BSR or Auto-RP.

## Overview of RP Selection Rules

The following table summarizes the primary IPv4 RP selection rules on Cisco IOS XE.

| Scenario | Selection Rule |
|---|---|
| Multiple static mappings match G | Highest RP IPv4 address |
| BSR Candidate RPs match G | Longest group prefix, lowest priority, highest hash, highest RP address |
| Multiple Auto-RP candidates advertise the same range | Mapping Agent selects the highest RP IPv4 address |
| Multiple positive Auto-RP group ranges match G | Longest matching group prefix, provided no negative mapping matches |
| Negative Auto-RP mapping matches G | No Auto-RP RP is selected; negative mappings take precedence over all positive mappings |
| Static and dynamic mappings both match G | Dynamic mapping wins unless static `override` is configured |

**Do not confuse selection among Candidate RPs with the final lookup of the RP for a multicast group.**

Also, longest-prefix matching is not a universal first step across every IOS XE RP mapping mechanism.

For example, overlapping static RP mappings use the highest RP address rather than selecting the most specific static group range.

## Auto-RP: Two Stages of RP Selection

Auto-RP provides a useful example of the distinction between selecting an RP for an advertised range and determining the effective RP for a specific multicast group.

Auto-RP uses two roles:

- **Candidate RP:** Advertises the multicast group ranges for which it is willing to act as an RP.
- **Mapping Agent (MA):** Receives Candidate RP announcements, selects RPs for advertised group ranges, and distributes the resulting mappings.

### Stage 1: Selecting an RP for Each Advertised Range

Suppose three Candidate RPs advertise the following information:

| Candidate RP | Advertised Group Range |
|---|---|
| `1.1.1.1` | `239.0.0.0/8` |
| `3.3.3.3` | `239.0.0.0/8` |
| `2.2.2.2` | `239.1.0.0/16` |

The Mapping Agent receives the RP-Announce messages and processes the advertised group ranges.

For `239.0.0.0/8`, two Candidate RPs are available:

```text
1.1.1.1
3.3.3.3
```

When multiple Candidate RPs advertise the **same group range**, the Mapping Agent selects the RP with the **highest IPv4 address**.

Therefore:

```text
Group range: 239.0.0.0/8
Selected RP: 3.3.3.3
```

For `239.1.0.0/16`, only one Candidate RP exists:

```text
Group range: 239.1.0.0/16
Selected RP: 2.2.2.2
```

The resulting mappings are:

| Group Range | Selected RP |
|---|---|
| `239.0.0.0/8` | `3.3.3.3` |
| `239.1.0.0/16` | `2.2.2.2` |

The Mapping Agent advertises these selected mappings in RP-Discovery messages sent to `224.0.1.40`.

**Stage 1 selects one RP for each advertised group range. It does not mean that every multicast group covered by a range must use that RP when a more specific positive mapping also exists. Negative Auto-RP mappings are a separate exception, discussed below.**

### Stage 2: Selecting the Effective RP for G

PIM routers receive the RP-Discovery messages and install the learned Group-to-RP mappings.

When a router needs an RP for a particular multicast group, it checks the group against the available mappings.

For example:

```text
G = 239.1.1.1
```

Both mappings match:

| Group Range | RP | Matches G? |
|---|---|---|
| `239.0.0.0/8` | `3.3.3.3` | Yes |
| `239.1.0.0/16` | `2.2.2.2` | Yes |

Because both entries are **positive mappings** and no negative mapping matches, the router uses **longest-prefix matching**, selecting the more specific `/16` mapping.

Therefore:

```text
Group:       239.1.1.1
Selected RP: 2.2.2.2
Reason:      Longest matching group prefix
```

Notice that `2.2.2.2` is selected even though `3.3.3.3` has a higher IP address.

The highest-address rule was applied during Stage 1 to choose between Candidate RPs advertising the **same group range**.

It does not override longest-prefix matching during the final lookup.

### Negative Auto-RP Mappings: Negative Before Positive

An Auto-RP Candidate RP can advertise both **positive** and **negative** group ranges through the standard IPv4 ACL referenced by `ip pim send-rp-announce ... group-list`:

- An explicit `permit` entry generates a **positive group announcement**, identifying a group range for which the Candidate RP is available.
- An explicit `deny` entry generates a **negative group announcement**, identifying a group range that should not use an Auto-RP RP.
- The **implicit deny** at the end of the ACL does **not** generate a negative announcement.

The Mapping Agent distributes learned negative mappings along with positive RP mappings. On receiving PIM routers, **negative Auto-RP mappings are checked before positive mappings**, regardless of prefix length or which Candidate RP advertised them.

For a particular group G, the Auto-RP lookup is:

1. Check the **negative mappings**. If any negative range matches G, **no Auto-RP RP is selected**.
2. If no negative range matches G, check the **positive mappings** and select the **longest matching prefix**.

**A negative `/8` mapping therefore takes precedence over a positive `/24` mapping**, even though `/24` is more specific. Longest-prefix matching applies only after the negative check succeeds (that is, no negative entry matches).

#### Example: Negative Mapping Overrides Multiple Positive Mappings

Suppose R1 and R2 advertise the following ranges. The examples use standard IPv4 ACL syntax, with contiguous wildcard masks:

**R1: Candidate RP `1.1.1.1`**

```text
R1(config)# ip access-list standard RP-GROUPS
R1(config-std-nacl)# permit 239.1.0.0 0.0.255.255
R1(config-std-nacl)# deny 239.0.0.0 0.255.255.255
R1(config)# ip pim send-rp-announce Loopback0 scope 16 group-list RP-GROUPS
```

**R2: Candidate RP `2.2.2.2`**

```text
R2(config)# ip access-list standard RP-GROUPS
R2(config-std-nacl)# permit 239.1.1.0 0.0.0.255
R2(config)# ip pim send-rp-announce Loopback0 scope 16 group-list RP-GROUPS
```

Assume the Mapping Agent distributes all three entries and the receiving router learns:

| Group Range | Type | Advertised By |
|---|---|---|
| `239.1.0.0/16` | Positive | `1.1.1.1` |
| `239.1.1.0/24` | Positive | `2.2.2.2` |
| `239.0.0.0/8` | **Negative** | `1.1.1.1` |

For group `239.1.1.1`, **all three mappings match**. Although R2's positive `/24` mapping is the most specific, the negative `/8` mapping takes precedence:

```text
Group:       239.1.1.1
Selected RP: None (negative Auto-RP mapping matches)
```

The result does **not** depend on whether the negative mapping and the positive mapping came from the same Candidate RP or different Candidate RPs.

Now consider `239.2.2.2`. It matches the negative `/8` but neither positive mapping, so no Auto-RP RP is selected for that group either.

#### Negative Mapping Does Not Automatically Enable Dense Mode

A negative mapping indicates that the matching group **has no Auto-RP RP** and is designated for Dense Mode operation. However, it does **not** by itself change the PIM mode configured on an interface.

- With `ip pim sparse-dense-mode`, the group can use Dense Mode forwarding when no RP is available, subject to the Dense Mode fallback configuration.
- With `ip pim sparse-mode`, a negative mapping does **not** automatically enable Dense Mode forwarding.

This is separate from `ip pim autorp listener`, which enables Dense Mode forwarding specifically for the Auto-RP control groups `224.0.1.39` and `224.0.1.40`.

#### IOS XE Verification of Negative Mappings

Use:

```text
R1# show ip pim rp mapping
```

A negative range is marked with **`(-)`** beside the group prefix. For example, this abbreviated output illustrates the notation:

```text
Group(s) 239.1.1.0/24
  RP 2.2.2.2 (?), v2v1
    Info source: 2.2.2.2 (?), elected via Auto-RP
Group(s) (-)239.0.0.0/8
  RP 1.1.1.1 (?), v2v1
    Info source: 1.1.1.1 (?), elected via Auto-RP
```

**`(-)` means a negative group mapping**, not a lower-priority RP. Although an RP address still appears in the entry, that address identifies the advertising Candidate RP; it is **not** the RP selected for groups covered by the negative mapping.

> **Important:** This negative-before-positive rule describes lookup within **Auto-RP-learned mappings**. Selection between static and dynamic configurations follows the separate precedence rules later in this page.

See [Candidate RP Announcements](../rp-discovery/auto-rp/rp-announcements.md) for `group-list` configuration and more detail about positive and negative announcements.

### Additional Group Lookup Examples

Assuming the same **positive-only** mapping database (no matching negative entries):

| Group Range | Selected RP |
|---|---|
| `239.0.0.0/8` | `3.3.3.3` |
| `239.1.0.0/16` | `2.2.2.2` |

Here are some example group lookups:

| Multicast Group | Effective RP | Reason |
|---|---|---|
| `239.1.1.1` | `2.2.2.2` | `/16` is more specific |
| `239.1.100.10` | `2.2.2.2` | `/16` is more specific |
| `239.2.2.2` | `3.3.3.3` | Only `/8` matches |
| `239.255.1.1` | `3.3.3.3` | Only `/8` matches |

**The Mapping Agent resolves competing candidates for each advertised range, while the receiving PIM router determines which learned range applies to G.**

> **Important:** Auto-RP does not use BSR Candidate RP priorities or the BSR hashing algorithm on IOS XE. Candidate RP selection for an identical group range is based on the highest RP IPv4 address.

See [Mapping Agent and Discovery Messages](../rp-discovery/auto-rp/mapping-agent.md) for detailed Auto-RP operation.

## BSR RP Selection

With BSR, the selection process is different.

The elected Bootstrap Router distributes an **RP-Set**, which can contain multiple Candidate RPs for the same group range.

Unlike Auto-RP, **the BSR does not select one winning RP for every advertised group range before distributing the information**.

Instead, each receiving PIM router independently determines the effective RP for G.

The selection algorithm follows this order:

1. **Longest matching group prefix**
2. **Lowest numerical RP priority**
3. **Highest calculated RP hash value**
4. **Highest RP IP address**, if the hash values are equal

### Example 1: Longest Prefix Match

Suppose the BSR advertises:

| Candidate RP | Group Range | RP Priority |
|---|---|---|
| `1.1.1.1` | `239.0.0.0/8` | 0 |
| `2.2.2.2` | `239.1.0.0/16` | 100 |

For:

```text
G = 239.1.1.1
```

Both group ranges match, but `/16` is more specific.

Therefore:

```text
Selected RP: 2.2.2.2
```

The lower numerical priority of `1.1.1.1` does not matter because the longest matching group prefix is evaluated first.

### Example 2: RP Priority

Suppose two Candidate RPs advertise the same group range:

| Candidate RP | Group Range | RP Priority |
|---|---|---|
| `1.1.1.1` | `239.1.0.0/16` | 10 |
| `2.2.2.2` | `239.1.0.0/16` | 20 |

Both candidates have the same matching group-prefix length.

The router next compares their RP priorities.

**Lower numerical priority values are preferred.**

Therefore:

```text
Selected RP: 1.1.1.1
```

### Example 3: RP Hashing

Suppose both Candidate RPs have the same group-prefix length and priority:

| Candidate RP | Group Range | RP Priority |
|---|---|---|
| `1.1.1.1` | `239.1.0.0/16` | 0 |
| `2.2.2.2` | `239.1.0.0/16` | 0 |

The router calculates a hash value for each Candidate RP.

The calculation uses:

- The multicast group address
- The RP address
- The hash mask advertised by the BSR

The Candidate RP with the **highest resulting hash value** wins.

If the calculated hash values are equal, the RP with the highest IP address is selected.

Hashing allows different multicast groups to be assigned to different RPs, depending on the configured hash mask and available candidates.

> **Note:** The BSR hash mask length is different from the group-prefix length. The group-prefix length determines which Candidate RPs are eligible, while the hash mask controls which group address bits influence the hash calculation.

The algorithm is covered in detail in [BSR Hashing](../rp-discovery/bsr/hashing.md).

### BSR vs. Auto-RP Selection

| Feature | BSR | Auto-RP |
|---|---|---|
| Information distributed | Candidate RP set | Selected RP mappings |
| Where Candidate RP selection occurs | Each PIM router | Mapping Agent |
| Overlapping group ranges | Longest prefix match | Negative mappings first; otherwise longest matching positive prefix |
| RP priority | Used | Not used for IOS XE Auto-RP selection |
| RP hashing | Used when needed | Not used |
| Highest RP address | Final hash tie-breaker | Selects among candidates for the same advertised range |

The essential difference is that **BSR distributes the candidates for routers to evaluate, whereas Auto-RP distributes mappings already selected by the Mapping Agent**.

## Multiple Static RP Mappings

Static RP mappings use different selection rules from BSR and Auto-RP.

IOS XE permits multiple `ip pim rp-address` commands, allowing different RPs to serve different multicast group ranges.

### Nonoverlapping Static Mappings

For example:

```text
R1(config)# ip access-list standard RP1-GROUPS
R1(config-std-nacl)# permit 239.1.0.0 0.0.255.255
R1(config)# ip access-list standard RP2-GROUPS
R1(config-std-nacl)# permit 239.2.0.0 0.0.255.255
R1(config)# ip pim rp-address 1.1.1.1 RP1-GROUPS
R1(config)# ip pim rp-address 2.2.2.2 RP2-GROUPS
```

The resulting selections are straightforward:

| Multicast Group | Selected RP |
|---|---|
| `239.1.1.1` | `1.1.1.1` |
| `239.2.2.2` | `2.2.2.2` |

Each multicast group matches only one static RP mapping.

### Overlapping Static Mappings

If multiple static RP mappings match the same multicast group, **IOS XE selects the RP with the highest IPv4 address**.

For example:

```text
R1(config)# ip pim rp-address 1.1.1.1
R1(config)# ip pim rp-address 2.2.2.2
```

Both configurations match group `239.1.1.1`.

The selected RP is:

```text
2.2.2.2
```

The selection does not depend on the unicast reachability of either RP.

### Static Group-Range Specificity

A less obvious behavior occurs when multiple static mappings have overlapping group ranges of different lengths.

Consider:

| Static RP | Group Range |
|---|---|
| `1.1.1.1` | `239.1.1.0/24` |
| `2.2.2.2` | `239.0.0.0/8` |

For group `239.1.1.10`, both ranges match.

Although `/24` is more specific than `/8`, IOS XE's documented rule for overlapping static `ip pim rp-address` configurations selects the **highest RP address**:

```text
Selected RP: 2.2.2.2
```

**Do not apply the BSR longest-prefix-first algorithm to overlapping static RP configurations.**

> **Important:** Each static IPv4 RP address can be used in only one `ip pim rp-address` configuration entry. To associate one RP with multiple group ranges, combine the group ranges in a single ACL.

See [Static RP](../rp-discovery/static-rp.md) for the complete configuration details.

## Static vs. Dynamic RP Precedence

When a router has matching static and dynamic Group-to-RP mappings, another selection rule applies.

On Cisco IOS XE:

**Dynamically learned RP mappings take precedence over static mappings by default, unless the static configuration uses the `override` keyword.**

Dynamic mappings can be learned through BSR or Auto-RP.

### Default: Dynamic Mapping Wins

Suppose R1 has the following static mapping:

```text
R1(config)# ip pim rp-address 1.1.1.1
```

It also learns a dynamic BSR mapping:

```text
Group range: 239.1.0.0/16
RP:          2.2.2.2
```

For `239.1.1.1`, both mappings apply.

The selected RP is:

```text
2.2.2.2
```

The dynamically learned RP takes precedence.

This also illustrates that the dynamic mapping can override a statically configured mapping covering a broader group range.

### Static RP Override

To make a static mapping take precedence, configure `override`:

```text
R1(config)# ip pim rp-address 1.1.1.1 override
```

Now the static RP is preferred for matching groups, even when a conflicting dynamic mapping exists.

For example:

```text
Group:       239.1.1.1
Selected RP: 1.1.1.1
```

The `override` keyword can also be used with a group ACL.

```text
R1(config)# ip access-list standard STATIC-GROUPS
R1(config-std-nacl)# permit 239.1.0.0 0.0.255.255
R1(config)# ip pim rp-address 1.1.1.1 STATIC-GROUPS override
```

In this case, the static override applies only to `239.1.0.0/16`.

Other multicast groups can continue using dynamically learned RP mappings.

### Static RP as a Fallback

A static mapping without `override` can serve as a fallback when no applicable dynamic mapping exists.

For example:

```text
R1(config)# ip pim rp-address 1.1.1.1
```

Suppose BSR advertises:

```text
239.1.0.0/16 -> 2.2.2.2
```

The effective RP selections are:

| Multicast Group | Effective RP | Reason |
|---|---|---|
| `239.1.1.1` | `2.2.2.2` | Dynamic mapping takes precedence |
| `239.2.2.2` | `1.1.1.1` | No applicable dynamic mapping |

If the dynamic mapping for `239.1.0.0/16` expires or is withdrawn, the static mapping can become the effective mapping for groups previously covered by the dynamic mapping.

However, this does not guarantee uninterrupted multicast forwarding.

The router may need to rebuild RP-dependent multicast state toward the newly selected RP.

### Static vs. Dynamic Summary

| Scenario | Result |
|---|---|
| Only static mapping matches G | Static RP |
| Only dynamic mapping matches G | Dynamic RP |
| Static and dynamic mappings match, without `override` | Dynamic RP |
| Static and dynamic mappings match, with `override` | Static RP |
| Dynamic mapping expires, matching static mapping remains | Static RP can become effective |

The `override` keyword changes **mapping precedence**, not RP reachability or failure detection.

## BSR and Auto-RP Used Together

BSR and Auto-RP are both dynamic RP discovery mechanisms, but they use different protocols and selection procedures.

Cisco IOS XE documentation states that **simultaneous deployment of BSR and Auto-RP is not supported** in the described IOS XE multicast configurations.

They should therefore be treated as alternative dynamic RP discovery mechanisms rather than mechanisms with a universal BSR-versus-Auto-RP precedence rule.

If different parts of a multicast network learn inconsistent RP information, routers may select different RPs for the same group.

Such inconsistencies can prevent shared-tree construction or source registration from working correctly.

## IPv6 RP Selection

IPv6 PIM-SM also uses Group-to-RP mappings.

The principal mechanisms include:

- Static IPv6 RP configuration
- IPv6 BSR
- Embedded RP for applicable IPv6 multicast groups

Auto-RP is IPv4-only.

### IPv6 Static RP

Configure an IPv6 RP using:

```text
R1(config)# ipv6 pim rp-address 2001:DB8:1::1
```

An optional IPv6 ACL can restrict the configuration to particular multicast group ranges.

Unlike the IPv4 command, the standard IOS XE `ipv6 pim rp-address` command does not provide an `override` keyword.

Therefore, do not assume that IPv4 static-versus-dynamic precedence configuration is directly applicable to IPv6.

### IPv6 BSR Selection

IPv6 BSR follows the same general selection order as IPv4 BSR:

1. Longest matching group prefix
2. Lowest numerical RP priority
3. Highest RP hash value
4. Highest RP IPv6 address as the final tie-breaker

The IPv6 hash calculation uses IPv6 address inputs and the BSR-advertised hash mask.

IPv6 administrative scope also affects which mappings are applicable to a particular group.

See [BSR Hashing](../rp-discovery/bsr/hashing.md) for details.

## RP Selection vs. RP Reachability

An important distinction is that **RP selection is independent of ordinary unicast routing metrics and reachability**.

For example, suppose two static RPs match the same group:

| RP | Reachability |
|---|---|
| `1.1.1.1` | Reachable |
| `2.2.2.2` | Unreachable |

IOS XE still selects:

```text
2.2.2.2
```

The RP with the highest IPv4 address wins, even though it is unreachable.

Similarly, BSR selection does not simply choose the Candidate RP with the lowest unicast routing metric.

An RP mapping can remain valid while the selected RP is unreachable.

When the selected RP is unreachable, PIM-SM operation can fail because routers cannot establish the necessary shared-tree paths or communicate with the RP.

**RP selection is not an automatic reachability-based failover mechanism.**

RP redundancy techniques such as IPv4 Anycast RP with MSDP and IPv6 PIM Anycast RP are covered in separate sections.

## IOS XE Verification

### Display Known RP Mappings

Use:

```text
R1# show ip pim rp mapping
```

This displays the router's known Group-to-RP mappings.

Check:

- Multicast group ranges, including **negative entries marked `(-)`**
- Candidate or selected RP addresses
- Mapping information sources
- RP priorities, where applicable
- Mapping expiration timers

On supported IOS XE releases, the following options can help filter the display:

```text
R1# show ip pim rp mapping elected
R1# show ip pim rp mapping in-use
```

The `elected` option displays elected Auto-RP mappings, while `in-use` filters mappings currently in use.

> **Note:** The `show ip pim rp` command displays active RP information. Its optional IPv4 address argument identifies an **RP address**, not a multicast group address.

### Verify BSR RP Selection for a Group

Use:

```text
R1# show ip pim rp-hash 239.1.1.1
```

This displays the BSR RP selection and hash information for the specified group.

It is particularly useful when multiple Candidate RPs advertise matching group ranges with identical priorities.

### Verify the Effective RP in Multicast State

For a group with multicast routing state, use:

```text
R1# show ip mroute 239.1.1.1
```

The `(*,G)` entry identifies the RP associated with that group's shared-tree state.

For example:

```text
(*, 239.1.1.1), 00:05:14/00:03:04, RP 2.2.2.2, flags: SJC
```

This confirms that the multicast routing entry references RP `2.2.2.2`.

Note that an existing mroute entry may reflect forwarding state that has not yet converged following an RP mapping change. Verify the mapping database as well.

### Verify Static RP Configuration

Use:

```text
R1# show running-config | include ^ip pim rp-address
```

Check for:

- Multiple static RP configurations
- Overlapping group ACLs
- The `override` keyword

To inspect an RP group ACL:

```text
R1# show ip access-lists STATIC-GROUPS
```

### Verify Dynamic RP Discovery

For BSR:

```text
R1# show ip pim bsr-router
R1# show ip pim rp mapping
```

For Auto-RP:

```text
R1# show ip pim autorp
R1# show ip pim rp mapping
```

The mappings learned by routers should be consistent throughout the PIM domain.

### IPv6 Verification

Display IPv6 group mappings:

```text
R1# show ipv6 pim group-map
```

Display static mappings:

```text
R1# show ipv6 pim group-map info-source static
```

Display BSR mappings:

```text
R1# show ipv6 pim group-map info-source bsr
```

Check a particular group:

```text
R1# show ipv6 pim group-map FF15::101
```

### Verify RP Reachability

Once the RP has been identified, check the RPF path toward it:

```text
R1# show ip rpf 2.2.2.2
```

A valid RP mapping does not guarantee that the router can reach the RP.

## Troubleshooting RP Selection

| Symptom | Possible Cause |
|---|---|
| Unexpected static RP selected | Another overlapping static mapping uses a higher RP address |
| Static RP ignored | Dynamic mapping takes precedence without `override` |
| Dynamic RP ignored | Static mapping uses `override` |
| Unexpected BSR RP selected | More specific group range, RP priority, or hash calculation |
| Unexpected Auto-RP mapping advertised | Another Candidate RP advertises the same range with a higher address |
| Unexpected RP selected from Auto-RP mappings | Another **positive** learned group range is more specific |
| No Auto-RP RP selected despite positive mappings | A matching **negative Auto-RP mapping** takes precedence, even over more-specific positive mappings |
| Different routers use different RPs | Inconsistent static mappings, RP-Sets, or Auto-RP discovery information |
| RP mapping changes unexpectedly | Advertisement expiration or withdrawal |
| Correct RP selected, but multicast fails | RPF failure, RP reachability, or PIM forwarding issue |

When troubleshooting, first distinguish whether the problem involves **candidate selection for a group range**, **effective mapping selection for G**, or **precedence between static and dynamic mappings**.

## Key Points

- RP selection involves two distinct concepts: selecting an RP for a group range and determining the effective RP for a particular group G.
- **Auto-RP:** The Mapping Agent selects the highest RP IPv4 address when multiple candidates advertise the same group range.
- **Auto-RP:** Receiving routers check **negative group mappings first**; if any matches G, no Auto-RP RP is selected. Otherwise, the most specific matching positive mapping wins.
- **Auto-RP:** Explicit `deny` entries in a Candidate RP `group-list` generate negative announcements; the implicit deny does not.
- **Auto-RP:** A negative mapping is shown with `(-)` in `show ip pim rp mapping`. It does not by itself enable Dense Mode forwarding on an interface configured with `ip pim sparse-mode`.
- **BSR:** Each router independently selects the RP using longest prefix, lowest priority, highest hash, and highest RP address.
- **Static RP:** If multiple static IPv4 mappings match G, the highest RP address wins, even when another mapping is more specific.
- **Static vs. dynamic:** Dynamic mappings take precedence by default on IOS XE.
- The IPv4 `override` keyword gives a matching static RP precedence over dynamically learned RP mappings.
- BSR and Auto-RP should not normally be deployed simultaneously in the same IOS XE multicast domain.
- RP selection does not automatically account for RP reachability or unicast routing metrics.
- Use `show ip pim rp mapping` to inspect known mappings, `show ip pim rp-hash <group>` for BSR group selection, and `show ip mroute <group>` to verify the RP associated with existing multicast forwarding state.

## References

- [Cisco IOS XE 17.17: IP Multicast Routing Commands](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9600/software/release/17-17/command_reference/b_1717_9600_cr/ip_multicast_routing_commands.html)
- [Cisco IOS XE 17.x: Configuring Basic IP Multicast](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_imc_basic_cfg-0.html)
- [Cisco IOS XE 17.x: Multicast Configuration Guide, PIM](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/multicast/multicast-configuration-guide/pim.html)
- [Cisco: How to Implement a Filtering Policy for Rendezvous Points](https://www.cisco.com/c/en/us/support/docs/ip/ip-multicast/41680-rp-announce.pdf)
- [Cisco: PIM Auto-RP Behavior with Other RP Distribution Techniques](https://www.cisco.com/c/en/us/support/docs/ip/multicast/118405-config-rp-00.html)
- [RFC 7761, Section 4.7.1: Group-to-RP Mapping](https://www.rfc-editor.org/rfc/rfc7761.html#section-4.7.1)
- [RFC 5059: Bootstrap Router Mechanism for PIM](https://www.rfc-editor.org/rfc/rfc5059.html)
