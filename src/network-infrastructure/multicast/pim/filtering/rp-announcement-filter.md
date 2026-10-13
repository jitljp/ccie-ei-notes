# Auto-RP Announcement Filtering

**Auto-RP announcement filtering** controls which Candidate RP (C-RP) advertisements an **Auto-RP Mapping Agent** accepts for particular multicast group ranges.

By default, a Mapping Agent accepts advertisements from any reachable C-RP. The `ip pim rp-announce-filter` command lets the Mapping Agent restrict the RP addresses and group ranges it accepts.

This feature applies to **IPv4 Auto-RP only**.

## Operation

Auto-RP uses two multicast groups:

| Message | Destination | Purpose |
|---|---|---|
| RP-Announce | `224.0.1.39` | C-RPs advertise their RP addresses and multicast group ranges |
| RP-Discovery | `224.0.1.40` | Mapping Agents distribute the selected group-to-RP mappings |

An RP announcement filter is applied **on the Mapping Agent**, after it receives an RP-Announce message but before the advertised C-RP information is accepted into its mapping database.

```text
     R1 (C-RP)           R2 (C-RP)
      3.3.3.3             4.4.4.4
          \                 /
           \ RP-Announce   /
            \             /
             R3 (Mapping Agent)
                     |
              RP-Announce Filter
                     |
                Accepted RPs
                     |
                 RP Selection
                     |
                RP-Discovery
                     |
                 PIM Routers
```

The filter controls **which RP mappings can be learned and advertised by the Mapping Agent**. It does not directly filter multicast data packets, PIM Joins, or the forwarding of RP-Announce packets across the network.

## IOS XE Configuration

Configure the command in **global configuration mode on the Mapping Agent**:

```text
ip pim [vrf <vrf-name>] rp-announce-filter [rp-list <rp-acl>] [group-list <group-acl>]
```

Specify at least one of `rp-list` or `group-list`.

| Parameter | Purpose |
|---|---|
| `vrf <vrf-name>` | Optional multicast VRF to which the filter applies |
| `rp-list <rp-acl>` | Standard IPv4 ACL selecting the **C-RP addresses** to which the filtering rule applies |
| `group-list <group-acl>` | Standard IPv4 ACL controlling the **multicast group ranges** accepted from the selected C-RPs |

Both ACLs are **standard IPv4 ACLs**, but they match different addresses:

- **`rp-list`:** The advertised Candidate RP address.
- **`group-list`:** The multicast group addresses the Candidate RP is advertising.

Neither ACL matches the outer multicast destination `224.0.1.39`. To filter the Auto-RP transport group itself, use a separate mechanism such as `ip multicast group-range` or an interface multicast boundary.

## RP-List and Group-List Evaluation

The most important distinction is that **`rp-list` selects which C-RPs are subject to the rule. It is not a conventional allowlist of accepted C-RPs.**

### `rp-list` Evaluation

- **Permit:** The C-RP is selected, and the associated filtering rule is applied to its advertisements.
- **Deny (including implicit deny):** The C-RP is **not selected** by that rule. Its advertisements bypass that rule, unless another rule filters them.

### `group-list` Evaluation

For a C-RP selected by the rule:

- **Permit:** The matching advertised group range is accepted.
- **Deny (including implicit deny):** The matching advertised group range is rejected.

### Omitted Options

- **`rp-list` omitted:** The `group-list` applies to **all Candidate RPs**.
- **`group-list` omitted:** **All advertised group ranges are rejected** for C-RPs selected by `rp-list`.

The resulting behavior is:

| `rp-list` | `group-list` | Result |
|---|---|---|
| Permits C-RP | Permits advertised group | Mapping accepted |
| Permits C-RP | Denies advertised group | Mapping rejected |
| Denies C-RP | Any result | This rule is bypassed; mapping can be accepted |
| Permits C-RP | Omitted | All of that C-RP's advertised groups rejected |
| Denies C-RP | Omitted | This rule is bypassed; mapping can be accepted |
| Omitted | Permits advertised group | Mapping accepted from any C-RP |
| Omitted | Denies advertised group | Mapping rejected from any C-RP |

> **CCIE note:** A `deny` in an **`rp-list`** does not reject that Candidate RP. To reject a C-RP's announcements with an RP-only filter, **permit its address in `rp-list` and omit `group-list`**.

## Filter Specific Candidate RPs and Groups

Suppose R3 is the Mapping Agent and should accept advertisements from C-RP `3.3.3.3` for `239.1.0.0/16`, but reject its advertisements for other ranges.

```text
R3(config)# ip access-list standard RP-LIST
R3(config-std-nacl)# permit host 3.3.3.3

R3(config)# ip access-list standard GROUP-LIST
R3(config-std-nacl)# permit 239.1.0.0 0.0.255.255

R3(config)# ip pim rp-announce-filter rp-list RP-LIST group-list GROUP-LIST
```

The behavior is:

| Candidate RP | Advertised group | Result |
|---|---|---|
| `3.3.3.3` | `239.1.0.0/16` | Accepted |
| `3.3.3.3` | `239.2.0.0/16` | Rejected by the group ACL's implicit deny |
| `4.4.4.4` | `239.1.0.0/16` | Bypasses this rule because it is not selected by `RP-LIST` |

**This configuration does not reject other Candidate RPs.** An unmatched C-RP is not selected by this rule.

## Filter Candidate RPs Only

To reject **all advertisements** from Candidate RP `4.4.4.4`, use `rp-list` without `group-list`:

```text
R3(config)# ip access-list standard BLOCKED-RPS
R3(config-std-nacl)# permit host 4.4.4.4

R3(config)# ip pim rp-announce-filter rp-list BLOCKED-RPS
```

Because `4.4.4.4` is **permitted by `rp-list`**, it is selected for the filter. With no `group-list`, all of its advertised groups are rejected.

Other Candidate RPs bypass this rule because they do not match the `rp-list` permit.

**Do not use `deny host 4.4.4.4` to block this RP.** That would cause it to bypass the rule instead.

## Filter Multicast Groups from All Candidate RPs

To reject advertisements for `239.1.0.0/16`, regardless of which C-RP sends them, omit `rp-list`:

```text
R3(config)# ip access-list standard GROUP-FILTER
R3(config-std-nacl)# deny 239.1.0.0 0.0.255.255
R3(config-std-nacl)# permit any

R3(config)# ip pim rp-announce-filter group-list GROUP-FILTER
```

The behavior is:

| Candidate RP | Advertised group | Result |
|---|---|---|
| `3.3.3.3` | `239.1.0.0/16` | Rejected |
| `4.4.4.4` | `239.1.0.0/16` | Rejected |
| Either RP | `239.2.0.0/16` | Accepted |

The final `permit any` allows group ranges not explicitly denied. Without it, the ACL's implicit deny would reject all other advertised groups too.

## Permit One Candidate RP and Reject All Others

Suppose the desired policy is:

| Candidate RP | Permitted group ranges |
|---|---|
| `3.3.3.3` | `239.1.0.0/16` |
| All other Candidate RPs | None |

Use **two filtering rules**: one for the trusted RP's permitted groups, and another to reject all other C-RPs.

**Rule 1: Restrict the trusted RP to its authorized groups.**

```text
R3(config)# ip access-list standard TRUSTED-RP
R3(config-std-nacl)# permit host 3.3.3.3

R3(config)# ip access-list standard TRUSTED-GROUPS
R3(config-std-nacl)# permit 239.1.0.0 0.0.255.255

R3(config)# ip pim rp-announce-filter rp-list TRUSTED-RP group-list TRUSTED-GROUPS
```

**Rule 2: Select every other RP and reject all its groups.**

```text
R3(config)# ip access-list standard OTHER-RPS
R3(config-std-nacl)# deny host 3.3.3.3
R3(config-std-nacl)# permit any

R3(config)# ip pim rp-announce-filter rp-list OTHER-RPS
```

The `deny host 3.3.3.3` in `OTHER-RPS` **excludes the trusted RP from Rule 2**. The `permit any` selects all other Candidate RPs, and the absence of `group-list` rejects all their advertisements.

| Candidate RP | Advertised group | Applied rule | Result |
|---|---|---|---|
| `3.3.3.3` | `239.1.0.0/16` | Rule 1 | Accepted |
| `3.3.3.3` | `239.2.0.0/16` | Rule 1 | Rejected |
| `4.4.4.4` | Any group | Rule 2 | Rejected |
| Any other C-RP | Any group | Rule 2 | Rejected |

## VRF-Specific Filtering

The same feature can be applied to a multicast VRF:

```text
R3(config)# ip pim vrf BLUE rp-announce-filter group-list GROUP-FILTER
```

This applies the group filtering policy to Candidate RP announcements associated with VRF `BLUE`, rather than the default multicast routing context.

The ACL remains a standard IPv4 ACL; the VRF is identified by the `ip pim vrf ... rp-announce-filter` command.

## Relationship to Other Multicast Filters

| Mechanism | Configured on | Function |
|---|---|---|
| `ip pim send-rp-announce ... group-list` | Candidate RP | Limits the groups the C-RP advertises |
| `ip pim rp-announce-filter` | Mapping Agent | Controls which advertised RP mappings the Mapping Agent accepts |
| `ip multicast boundary ... filter-autorp` | Interface | Filters Auto-RP mapping information crossing a multicast boundary |
| `ip multicast group-range` | Router or VRF | Restricts multicast group/channel processing, including the Auto-RP transport groups when denied |

These mechanisms operate at different points. In particular, `ip pim rp-announce-filter` evaluates **the C-RP address and advertised group ranges inside received RP-Announce messages**. It does not itself block forwarding of the RP-Announce or RP-Discovery multicast packets.

See [Multicast Boundaries and Administrative Scope](multicast-boundary.md) and [Multicast Group-Range Filtering](ipv6-group-range.md) for the other filtering mechanisms.

## IOS XE Verification

Check the configured filters and their ACLs:

```text
R3# show running-config | include rp-announce-filter
R3# show ip access-lists TRUSTED-RP
R3# show ip access-lists TRUSTED-GROUPS
R3# show ip access-lists OTHER-RPS
```

Verify the Mapping Agent and accepted RP mappings:

```text
R3# show ip pim autorp
R3# show ip pim rp mapping
```

Check the RP mapping learned by another router:

```text
R4# show ip pim rp mapping
R4# show ip pim rp 239.1.1.1
```

For additional detail, enable Auto-RP debugging temporarily:

```text
R3# debug ip pim auto-rp
R3# undebug all
```

**An increasing RP-Announce received counter does not prove that the advertised mapping was accepted.** Check `show ip pim rp mapping` to verify the resulting group-to-RP mappings.

RP mappings are maintained as soft state. After a rule starts rejecting an announcement, an existing mapping may remain visible until its **holdtime expires**. If the mapping is still accepted, the expiration timer continues to refresh.

If multiple Mapping Agents are configured, use consistent filters to avoid conflicting group-to-RP advertisements.

## Key Points

- **`ip pim rp-announce-filter` is configured on an Auto-RP Mapping Agent** and controls which C-RP advertisements are accepted.
- **`rp-list` selects Candidate RPs for filtering.** A permitted RP is subject to the rule; a denied RP bypasses that rule.
- **`group-list` determines which advertised group ranges are accepted** from selected RPs. An omitted `rp-list` makes the group filter apply to all Candidate RPs.
- **Omitting `group-list` rejects all advertised groups** from Candidate RPs selected by `rp-list`.
- A C-RP not selected by one rule is **not automatically rejected**. Use another rule if it must be blocked.
- Announcement filtering changes the mappings distributed through Auto-RP; it does not directly filter multicast data packets.
- Verify filtering with `show ip pim rp mapping`, and account for existing RP mapping holdtimes when changing the policy.
