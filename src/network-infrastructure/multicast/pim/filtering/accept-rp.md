# PIM RP Acceptance Filtering

**PIM RP acceptance filtering** restricts which Rendezvous Points (RPs) a router will accept for specific IPv4 multicast groups.

By default, IOS XE does not apply an RP acceptance filter. PIM Join/Prune messages are processed without this additional restriction.

The `ip pim accept-rp` command allows administrators to restrict acceptable RP/group combinations, preventing routers from processing shared-tree Joins associated with unauthorized RPs or multicast groups.

This is particularly useful for:

- Rejecting `(*,G)` Joins associated with unauthorized RPs.
- Restricting the multicast groups for which an RP can establish shared-tree state.
- Accepting only RPs learned through Auto-RP.
- Preventing incorrect or unauthorized RP configurations from affecting multicast routing.

Unlike RP announcement filtering, this mechanism operates during **PIM control-plane processing**, rather than filtering RP advertisements.

## Operation

Consider the following topology:

```text
              R1 (RP)
              3.3.3.3
                 |
                 |
                R2
               /  \
              /    \
             R3    R4
             |      |
           Rec1    Rec2
```

Assume R1 is the RP for the following multicast groups:

```text
239.1.1.1
239.1.1.2
239.2.2.2
```

R2 is configured to accept R1 as the RP only for groups in `239.1.1.0/24`.

When R2 receives PIM `(*,G)` Join messages:

1. R2 examines the RP address associated with the Join.
2. R2 checks whether the RP is permitted by its RP acceptance policy.
3. R2 checks whether the multicast group is permitted by the associated ACL.
4. If the RP/group combination is accepted, R2 processes the Join normally.
5. Otherwise, R2 rejects the Join.

For example:

| RP | Group | Result |
|---|---|---|
| `3.3.3.3` | `239.1.1.1` | Accepted |
| `3.3.3.3` | `239.1.1.2` | Accepted |
| `3.3.3.3` | `239.2.2.2` | Rejected |
| `4.4.4.4` | `239.1.1.1` | Rejected |

The filter evaluates the **RP address and multicast group**, not the address of the neighboring router that sent the Join.

> **Note:** PIM Join/Prune packets are normally sent to the link-local multicast destination `224.0.0.13`. The RP being validated is the RP identified in the `(*,G)` Join information, not the packet's IPv4 destination address.

## IOS XE Configuration

Configure RP acceptance filtering in global configuration mode:

```text
ip pim [vrf <vrf-name>] accept-rp {<rp-address> | auto-rp} [<acl>]
```

| Parameter | Purpose |
|---|---|
| `vrf <vrf-name>` | Optional multicast VRF to which the policy applies |
| `<rp-address>` | Specifies an acceptable RP address |
| `auto-rp` | Accepts RPs learned through Auto-RP |
| `<acl>` | Optional standard IPv4 ACL identifying permitted multicast groups |

The command supports two alternatives:

- **Explicit RP address:** Accept the specified RP for groups permitted by the optional ACL.
- **`auto-rp`:** Accept RPs present in the Auto-RP cache, optionally restricted to specific multicast groups.

If no ACL is specified, the rule applies to all multicast groups subject to RP acceptance filtering.

### Explicit RP Filtering

Suppose R2 should accept RP `3.3.3.3` only for `239.1.1.0/24`.

Configure:

```text
R2(config)# ip access-list standard RP-GROUPS
R2(config-std-nacl)# permit 239.1.1.0 0.0.0.255

R2(config)# ip pim accept-rp 3.3.3.3 RP-GROUPS
```

The ACL matches the **multicast group address (G)**, not the source address or the RP address.

The results are:

| RP | Group | Result |
|---|---|---|
| `3.3.3.3` | `239.1.1.1` | Accepted |
| `3.3.3.3` | `239.1.1.100` | Accepted |
| `3.3.3.3` | `239.2.2.2` | Rejected |
| `4.4.4.4` | `239.1.1.1` | Rejected |

The ACL's implicit deny rejects groups not explicitly permitted.

### Accept an RP for All Groups

To accept RP `3.3.3.3` without restricting its multicast groups:

```text
R2(config)# ip pim accept-rp 3.3.3.3
```

Because no ACL is specified, the command allows the RP for all applicable multicast groups.

This does not automatically configure `3.3.3.3` as the RP for those groups. Group-to-RP mapping remains a separate process.

## ACL Permit and Deny Behavior

The ACL controls which multicast groups are acceptable for the specified RP.

- **Permit:** The group is accepted for the specified RP.
- **Deny:** The group is rejected for the specified RP.
- **Implicit deny:** Groups not matching a permit statement are rejected.

For example, suppose R1 should be accepted as the RP for all groups **except `239.1.0.0/16`**.

```text
R2(config)# ip access-list standard RP-GROUPS
R2(config-std-nacl)# deny 239.1.0.0 0.0.255.255
R2(config-std-nacl)# permit any

R2(config)# ip pim accept-rp 3.3.3.3 RP-GROUPS
```

The results are:

| RP | Group | Result |
|---|---|---|
| `3.3.3.3` | `239.1.1.1` | Rejected |
| `3.3.3.3` | `239.1.2.2` | Rejected |
| `3.3.3.3` | `239.2.2.2` | Accepted |
| `3.3.3.3` | `239.3.3.3` | Accepted |

The final `permit any` is important.

Without it, the ACL's implicit deny would reject every other group as well.

## Multiple Accepted RPs

Multiple `ip pim accept-rp` statements can be configured to permit different RP/group combinations.

For example, suppose the desired policy is:

| RP | Accepted Groups |
|---|---|
| `3.3.3.3` | `239.1.0.0/16` |
| `4.4.4.4` | `239.2.0.0/16` |

Configure:

```text
R2(config)# ip access-list standard RP1-GROUPS
R2(config-std-nacl)# permit 239.1.0.0 0.0.255.255

R2(config)# ip access-list standard RP2-GROUPS
R2(config-std-nacl)# permit 239.2.0.0 0.0.255.255

R2(config)# ip pim accept-rp 3.3.3.3 RP1-GROUPS
R2(config)# ip pim accept-rp 4.4.4.4 RP2-GROUPS
```

The results are:

| RP | Group | Result |
|---|---|---|
| `3.3.3.3` | `239.1.1.1` | Accepted |
| `3.3.3.3` | `239.2.2.2` | Rejected |
| `4.4.4.4` | `239.1.1.1` | Rejected |
| `4.4.4.4` | `239.2.2.2` | Accepted |
| `5.5.5.5` | Either group | Rejected |

These statements define acceptable RP/group combinations.

**They do not select which RP is used for a group.** Static RP configuration, BSR, or Auto-RP determines the effective group-to-RP mapping.

## Auto-RP Acceptance Filtering

Instead of explicitly configuring acceptable RP addresses, the router can use the **Auto-RP cache**.

Configure:

```text
R2(config)# ip pim accept-rp auto-rp
```

This allows the router to accept RPs learned through Auto-RP while rejecting RPs not present in its Auto-RP cache, unless another acceptance rule explicitly permits them.

For example, suppose the router has learned:

| Group Range | RP | Learned Via |
|---|---|---|
| `239.1.0.0/16` | `3.3.3.3` | Auto-RP |
| `239.2.0.0/16` | `4.4.4.4` | Auto-RP |

With `ip pim accept-rp auto-rp` configured:

- RPs learned through Auto-RP are acceptable for their corresponding multicast groups.
- An RP configured statically but not present in the Auto-RP cache is not accepted merely because it has a valid static mapping.
- RPs learned only through BSR are not included in the Auto-RP cache.

This provides a way to restrict RP acceptance to information learned through the Auto-RP mechanism.

### Auto-RP with Group Filtering

The `auto-rp` option can also reference an ACL.

For example, accept Auto-RP-learned RPs only for groups in `239.1.0.0/16`:

```text
R2(config)# ip access-list standard ALLOWED-GROUPS
R2(config-std-nacl)# permit 239.1.0.0 0.0.255.255

R2(config)# ip pim accept-rp auto-rp ALLOWED-GROUPS
```

The router now requires both conditions:

1. The RP must be present in the Auto-RP cache.
2. The multicast group must be permitted by `ALLOWED-GROUPS`.

An RP being present in the Auto-RP cache is not sufficient if the group is denied by the ACL.

### Auto-RP Bootstrap Exception

A special consideration applies when a network uses Sparse Mode with a **statically configured RP for the Auto-RP control groups**.

Auto-RP uses:

| Group | Purpose |
|---|---|
| `224.0.1.39` | RP-Announce |
| `224.0.1.40` | RP-Discovery |

Suppose a static RP at `3.3.3.3` is responsible for these control groups.

If the router uses:

```text
R2(config)# ip pim accept-rp auto-rp
```

The static RP may be rejected because it is not present in the Auto-RP cache.

To permit it specifically for the Auto-RP control groups, configure an additional acceptance rule:

```text
R2(config)# ip access-list standard AUTORP-GROUPS
R2(config-std-nacl)# permit host 224.0.1.39
R2(config-std-nacl)# permit host 224.0.1.40

R2(config)# ip pim accept-rp 3.3.3.3 AUTORP-GROUPS
```

The two rules work together:

```text
ip pim accept-rp auto-rp
ip pim accept-rp 3.3.3.3 AUTORP-GROUPS
```

This permits:

- RPs learned through Auto-RP.
- Static RP `3.3.3.3` for the two Auto-RP control groups.

This additional rule is relevant when the control groups actually use a static RP. It is not required simply because Auto-RP is enabled.

## RP Acceptance Filtering on the RP Itself

An important distinction is that `ip pim accept-rp` can also affect the router acting as the RP.

When the specified RP address is one of the local router's own addresses, the command restricts which multicast groups the router will serve as RP.

For example, on R3 (RP address `3.3.3.3`):

```text
R3(config)# ip access-list standard RP-GROUPS
R3(config-std-nacl)# permit 239.1.0.0 0.0.255.255

R3(config)# ip pim accept-rp 3.3.3.3 RP-GROUPS
```

R3 accepts RP-related PIM signaling for groups in `239.1.0.0/16`.

For groups outside that range:

- `(*,G)` Joins directed toward R3 as the RP are rejected.
- PIM Registers for denied groups are not accepted.
- R3 responds to rejected source registrations with **PIM Register-Stop** messages.

This allows the RP itself to enforce the group ranges it is authorized to serve.

> **Note:** `ip pim accept-rp` and `ip pim accept-register` are different commands. The latter specifically filters PIM Register messages using source/group criteria.

## Comparison with Other PIM Filtering Commands

| Command | Purpose | Configured On |
|---|---|---|
| `ip pim accept-rp` | Restricts acceptable RP/group combinations | Routers processing RP-related PIM signaling |
| `ip pim rp-announce-filter` | Filters Candidate RP advertisements | Auto-RP Mapping Agent |
| `ip pim accept-register` | Filters PIM source registration messages | RP |
| `ip multicast group-range` | Restricts multicast groups or source/group channels processed by the router | Multicast routers |
| `ip multicast boundary` | Restricts multicast forwarding and signaling across an interface | Multicast router interfaces |

These commands operate at different stages.

For example, `ip pim rp-announce-filter` can prevent an unauthorized Candidate RP from being accepted by a Mapping Agent.

However, `ip pim accept-rp` can reject an unacceptable RP/group combination during subsequent PIM processing, even when the router already has RP mapping information.

**Learning an RP mapping does not mean that all PIM Join messages associated with that RP will be accepted.**

## VRF-Specific RP Acceptance Filtering

RP acceptance filtering also supports multicast VRFs.

The syntax is:

```text
ip pim vrf <vrf-name> accept-rp {<rp-address> | auto-rp} [<acl>]
```

For example:

```text
R2(config)# ip access-list standard BLUE-GROUPS
R2(config-std-nacl)# permit 239.1.0.0 0.0.255.255

R2(config)# ip pim vrf BLUE accept-rp 3.3.3.3 BLUE-GROUPS
```

This applies the policy to multicast routing within VRF `BLUE`, rather than the global multicast routing context.

The ACL still matches IPv4 multicast group addresses.

## IOS XE Verification and Troubleshooting

### Verify RP Acceptance Configuration

```text
R2# show running-config | include accept-rp
```

Example:

```text
ip pim accept-rp 3.3.3.3 RP-GROUPS
```

Display the referenced ACL:

```text
R2# show ip access-lists RP-GROUPS
```

### Verify Group-to-RP Mapping

```text
R2# show ip pim rp mapping
```

This displays the learned or configured RP mappings.

When troubleshooting, compare the mapped RP address against the configured `ip pim accept-rp` statements.

### Verify Multicast Routing State

```text
R2# show ip mroute 239.1.1.1
```

Check whether the expected `(*,G)` state and outgoing interfaces have been established.

A rejected Join may prevent the corresponding shared-tree forwarding state from being created or maintained.

### INVALID_RP_JOIN Messages

A common symptom of RP acceptance filtering is:

```text
%PIM-6-INVALID_RP_JOIN: Received (*, 239.1.1.1) Join from 10.1.12.2 for invalid RP 3.3.3.3
```

This indicates that the router rejected a PIM `(*,G)` Join because the associated RP was considered invalid.

Possible causes include:

- The RP address is not permitted by the acceptance policy.
- The multicast group is denied by the associated ACL.
- The RP is not present in the Auto-RP cache when `auto-rp` acceptance is required.
- RP mappings have changed, but the acceptance configuration has not been updated.

**An `INVALID_RP_JOIN` message does not necessarily mean that the RP address itself is incorrect.** A denied multicast group can also cause the Join to be rejected.

For detailed troubleshooting:

```text
R2# debug ip pim
```

Use debugging carefully on production devices.

### Remove RP Acceptance Filtering

Remove an explicit RP acceptance rule:

```text
R2(config)# no ip pim accept-rp 3.3.3.3 RP-GROUPS
```

Remove Auto-RP-based acceptance filtering:

```text
R2(config)# no ip pim accept-rp auto-rp
```

When no RP acceptance rules remain, the additional RP acceptance check is disabled, returning to the default behavior.

## Key Points

- `ip pim accept-rp` restricts acceptable **RP/group combinations** during PIM processing.
- It is configured in global configuration mode, with optional VRF support.
- By default, RP acceptance filtering is disabled.
- An explicit RP address can be configured with an optional standard IPv4 ACL.
- The ACL matches the **multicast group address**, not the RP address or PIM neighbor address.
- **ACL permit** allows the group for the specified RP; **deny** rejects it.
- If the ACL is omitted, the rule applies to all applicable multicast groups.
- Multiple `ip pim accept-rp` rules can permit different RP/group combinations.
- `ip pim accept-rp auto-rp` accepts RPs present in the Auto-RP cache.
- The `auto-rp` option can also use a group ACL to restrict accepted multicast groups.
- When configured on the RP itself, the command can also restrict source registrations for denied groups.
- `ip pim accept-rp` does **not** configure or select the RP for a group.
- Rejected `(*,G)` Joins can generate `%PIM-6-INVALID_RP_JOIN` messages.
- `ip pim accept-rp` is an IPv4 PIM control-plane filtering mechanism, not a general multicast data-plane ACL.

## Further Reading

- [Cisco: Troubleshoot IP Multicast Guide](https://www.cisco.com/c/en/us/support/docs/ip/ip-multicast/16450-mcastguide0.html)
- [Cisco IOS XE 17.x: Configuring PIM](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/multicast/multicast-configuration-guide/pim.html)
- [Cisco: ip pim accept-rp Command Reference](https://www.cisco.com/c/en/us/td/docs/ios/12_2sb/feature/guide/sbb_mvpn.pdf)