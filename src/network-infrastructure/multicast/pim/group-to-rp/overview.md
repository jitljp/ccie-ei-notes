# 1.6.c (iii) Group-to-RP Mapping

In **PIM Sparse Mode (PIM-SM)** using Any-Source Multicast (ASM), routers must determine which **Rendezvous Point (RP)** is responsible for a particular multicast group.

This process is called **Group-to-RP Mapping**.

The previous section covered how RP information is configured or distributed using Static RP, BSR, and Auto-RP.

This section focuses on how routers **maintain Group-to-RP mappings and select the effective RP** when multiple mappings are available.

## Group-to-RP Mapping Operation

A Group-to-RP mapping associates a multicast group or range of groups with an RP address.

For example:

| Multicast Group Range | RP Address |
|---|---|
| `239.1.0.0/16` | `1.1.1.1` |
| `239.2.0.0/16` | `2.2.2.2` |
| `239.3.0.0/16` | `3.3.3.3` |

When a router needs an RP for `239.2.1.10`, it determines that the group belongs to `239.2.0.0/16` and selects `2.2.2.2`.

The selected RP is used for two important PIM-SM operations:

| Operation | Use of the RP Mapping |
|---|---|
| **Receiver joins** | The Last-Hop Router (LHR) sends `(*,G)` Joins toward the selected RP to establish the shared tree |
| **Source registration** | The source-facing Designated Router (DR) unicasts PIM Register messages to the selected RP |

Intermediate PIM routers also use the Group-to-RP mapping when processing shared-tree Join/Prune messages.

After selecting an RP, routers use their routing information to determine the RPF path toward that RP.

**Group-to-RP mapping determines the RP address, not the actual network path used to reach it.**

## Sources of RP Mapping Information

A router can obtain Group-to-RP mappings through several mechanisms:

| Mechanism | How RP Mapping Information Is Obtained |
|---|---|
| **Static RP** | Mappings are configured locally using `ip pim rp-address` or `ipv6 pim rp-address` |
| **BSR** | Routers receive an RP-Set and independently select the RP using group-range matching, priority, and hashing |
| **Auto-RP** | Routers receive mappings selected and advertised by an Auto-RP Mapping Agent |

Static RP and BSR support IPv4 and IPv6, whereas Auto-RP is IPv4-only.

Depending on the mechanism, a router may have multiple candidate RPs for a group or receive an already selected mapping.

## RP Mapping Selection and Precedence

A router may have multiple mappings that match the same multicast group.

For example:

| Group Range | RP | Source |
|---|---|---|
| `239.0.0.0/8` | `1.1.1.1` | Static RP |
| `239.1.0.0/16` | `2.2.2.2` | BSR |
| `239.1.1.0/24` | `3.3.3.3` | BSR |

All three ranges include `239.1.1.10`.

The router must determine which RP mapping takes precedence.

Two distinct decisions are involved:

1. **RP selection within a mechanism:** For example, BSR uses group-range specificity, RP priority, and hashing to select among Candidate RPs.
2. **Precedence between mapping sources:** For example, IOS XE can prefer a dynamically learned RP over a conflicting static RP unless the static mapping uses the `override` keyword.

These rules determine the **effective RP** that the router uses for a particular multicast group.

The selection rules are covered in detail in [RP Mapping Selection and Precedence](selection-precedence.md).

## Mapping Consistency and RP Reachability

PIM routers within the same multicast domain must agree on the effective RP for a given ASM group.

If routers select different RPs, source registration and shared-tree construction may become inconsistent, potentially preventing receivers from receiving multicast traffic.

An RP mapping also does not guarantee that the RP is reachable.

The router must have a valid RPF path toward the selected RP for normal shared-tree operation.

Additionally, dynamically learned mappings can expire or change when RP advertisements are withdrawn or stop arriving.

**RP mapping availability, RP selection, and RP reachability are separate considerations.**

> **Note:** Source-Specific Multicast (SSM) does not require Group-to-RP mapping. Receivers specify both the source and group, and routers build source-specific trees directly. See [Source-Specific Multicast](../ssm/overview.md).

## IOS XE Verification

For IPv4, display the Group-to-RP mapping information:

```text
R1# show ip pim rp mapping
```

Check the effective RP for a particular multicast group:

```text
R1# show ip pim rp 239.1.1.10
```

Verify the RPF path toward the selected RP:

```text
R1# show ip rpf 3.3.3.3
```

For IPv6, display the Group-to-RP mappings:

```text
R1# show ipv6 pim group-map
```

Check the mapping for a particular IPv6 group:

```text
R1# show ipv6 pim group-map FF15::101
```

These commands help distinguish missing mappings, unexpected RP selection, and RP reachability problems.

## Section Overview

The following pages examine Group-to-RP mapping in more detail:

- [RP Mapping Database and Group Ranges](mapping-database.md): How IOS XE maintains RP mapping information, handles multicast group ranges, and displays learned mappings.
- [RP Mapping Selection and Precedence](selection-precedence.md): How IOS XE determines the effective RP when multiple mappings or RP discovery mechanisms apply.

## Key Points

- **Group-to-RP mapping** associates ASM multicast groups with Rendezvous Points.
- PIM-SM routers use the effective RP for shared-tree construction and source registration.
- RP mapping information can come from **Static RP, BSR, or Auto-RP**.
- Multiple mappings can match the same group, requiring RP selection and precedence rules.
- BSR distributes candidate RP information, while Auto-RP distributes mappings selected by a Mapping Agent.
- PIM routers must agree on the effective RP, and the RP must be reachable.
- RP mappings can change or expire, potentially affecting multicast forwarding.
- **SSM does not require Group-to-RP mapping**.

## References

- [RFC 7761, Section 4.7.1: Group-to-RP Mapping](https://www.rfc-editor.org/rfc/rfc7761.html#section-4.7.1)
- [Cisco IOS XE: Verifying IP Multicast Operation](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-multicast/ip-multicast/m_imc_verify.html)