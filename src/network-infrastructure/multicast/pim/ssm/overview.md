
# 1.6.c (iv) Source-Specific Multicast

**Source-Specific Multicast (SSM)** is a multicast service model in which receivers explicitly identify both the **multicast source (S)** and the **multicast group (G)** from which they want to receive traffic.

This combination is called an **SSM channel**, represented as:

```text
(S,G)
```

For example:

**IPv4:**

```text
Source (S): 10.1.1.10
Group (G):  232.1.1.1

Channel: (10.1.1.10, 232.1.1.1)
```

**IPv6:**

```text
Source (S): 2001:db8:10::10
Group (G):  ff35::fd00:1234

Channel: (2001:db8:10::10, ff35::fd00:1234)
```

Unlike traditional **Any-Source Multicast (ASM)**, receivers do not simply request traffic for a group from any available source. They specify exactly which sources they want to receive.

Because the sources are already known, SSM does **not** require a **Rendezvous Point (RP)**, shared tree, or source registration process.

Instead, PIM builds a **source-specific Shortest-Path Tree (SPT)** directly toward each requested source.

## ASM vs. SSM

Traditional PIM Sparse Mode supports ASM, in which a receiver joins a multicast group without necessarily knowing the sources.

SSM eliminates the need for the multicast network to discover sources on behalf of receivers.

| Feature | ASM | SSM |
|---|---|---|
| Receiver requests | Group `G` | Source/group channel `(S,G)` |
| Source selection | Any source sending to G | Only explicitly requested sources |
| RP required | Yes, for traditional PIM-SM | No |
| Shared tree `(*,G)` | Yes | No |
| Source tree `(S,G)` | Optional after SPT switchover | Used from the beginning |
| PIM Register messages | Yes | No |
| SPT switchover | Possible | Not required |
| IPv4 receiver signaling | IGMPv1/v2/v3 | IGMPv3 |
| IPv6 receiver signaling | MLDv1/v2 | MLDv2 |

SSM uses the source-tree mechanisms of PIM Sparse Mode, but removes the RP-dependent operations associated with ASM.

## SSM Address Ranges

IPv4 and IPv6 both define address space specifically for SSM.

| Protocol | SSM Address Range | Example |
|---|---|---|
| IPv4 | `232.0.0.0/8` | `232.1.1.1` |
| IPv6 | `ff3x::/32` | `ff35::fd00:1234` |

For IPv6, `x` identifies the multicast scope.

Common examples include:

```text
ff32::/16   Link-Local SSM
ff35::/16   Site-Local SSM
ff3e::/16   Global SSM
```

The IPv6 SSM format uses `ff3x::/32`, with actual group-address allocation normally occurring within `ff3x::/96`.

Cisco IOS XE also supports defining a **custom IPv4 SSM range** using an access list.

SSM address ranges and their configuration are covered in [ASM vs SSM and SSM Address Ranges](asm-vs-ssm.md).

## Basic SSM Operation

SSM uses a receiver-driven forwarding model.

Consider the following topology:

```text
             Multicast Source
                    |
                   R1
                    |
                   R2
                  /  \
                 /    \
                R3    R4
                 |     |
              Receiver Receiver
```

Assume both receivers want traffic from the same source.

The basic process is:

1. **Receiver signaling:** The receivers send IGMPv3 (IPv4) or MLDv2 (IPv6) Reports identifying the desired `(S,G)` channel.
2. **Last-hop router processing:** R3 and R4 learn the requested source and group.
3. **PIM Join messages:** Each last-hop router sends a PIM `(S,G)` Join toward the source, following its RPF path.
4. **Source-tree construction:** Intermediate routers establish `(S,G)` forwarding state and propagate the Joins upstream.
5. **Multicast forwarding:** Traffic flows from the source along the resulting source-specific distribution tree to the receivers.

There is no initial shared-tree construction or subsequent SPT switchover.

**PIM does not need to discover the source because its address was explicitly supplied by the receiver.**

SSM still relies on **Reverse Path Forwarding (RPF)** to identify the upstream interface toward the source and to validate incoming multicast traffic.

The complete process is covered in [SSM Receiver Signaling and PIM Forwarding](pim-forwarding.md).


## IGMPv3 and MLDv2 Requirements

Native SSM requires receiver signaling that can identify individual multicast sources.

- **IPv4:** IGMPv3
- **IPv6:** MLDv2

Both protocols support source filtering using **INCLUDE** mode.

For example:

```text
Group:   G
Mode:    INCLUDE
Sources: {S1, S2}
```

This indicates that the receiver wants traffic for group G only from S1 and S2.

However, **a source-specific IGMPv3 or MLDv2 membership request does not necessarily mean PIM will operate in SSM mode**.

The distinction is:

- **IGMPv3/MLDv2:** Specify which multicast sources the receiver wants.
- **PIM:** Determines how multicast distribution trees are established, based on the multicast group and the router's multicast configuration.

For groups in the **SSM range**, the router uses source-specific `(S,G)` forwarding without an RP or shared tree.

For groups in the **ASM range**, PIM can still use traditional ASM mechanisms, including RP-based shared trees, even when the receiver has supplied a source list.

Therefore, **IGMPv3/MLDv2 source filtering and PIM-SSM are related but distinct mechanisms**.

An IGMPv3 or MLDv2 receiver can request specific sources for an ASM group without causing PIM to switch to SSM.


## SSM Mapping

A limitation of native SSM is that older receivers using **IGMPv1/IGMPv2** or **MLDv1** cannot include source addresses in their membership reports.

**SSM Mapping** allows these receivers to use an SSM-enabled network.

Instead of learning the source directly from the receiver, the last-hop router obtains the source address from a mapping database.

For example:

```text
Legacy Receiver
  |
  | Join G
  v
Last-Hop Router
  |
  | SSM Mapping: G → S
  |
  | PIM (S,G) Join
  v
Multicast Source
```

SSM mapping can obtain source information from:

- **Static mappings:** Source addresses configured locally on the router.
- **DNS-based mappings:** Source addresses retrieved using DNS queries.

Cisco IOS XE supports SSM mapping for IPv4 IGMP receivers and IPv6 MLD receivers.

SSM mapping does not change PIM's source-specific forwarding model. It simply allows the last-hop router to obtain source addresses that legacy receivers cannot supply themselves.

The mechanisms and configuration are covered in [SSM Mapping](ssm-mapping.md) and [Static and DNS-Based SSM Mapping](static-ssm-mapping.md).

## Basic IOS XE Configuration

### IPv4 SSM

IPv4 SSM must be enabled using the following global configuration command:

```text
R1(config)# ip pim ssm default
```

This enables SSM for the standard `232.0.0.0/8` range.

The router also requires IPv4 multicast routing and PIM on the appropriate interfaces. IGMPv3 is required on receiver-facing interfaces for native SSM.

### IPv6 SSM

IPv6 SSM is supported as part of IPv6 multicast routing using PIM and MLDv2.

Enable IPv6 multicast routing:

```text
R1(config)# ipv6 multicast-routing
```

On IOS XE platforms supporting IPv6 PIM, multicast routing enables PIM and MLD processing on eligible IPv6-enabled interfaces.

Unlike IPv4, **IPv6 does not use `ip pim ssm default`**. The standard IPv6 SSM address range is recognized through the IPv6 multicast implementation.

### Verification

Useful IPv4 commands include:

```text
show ip igmp groups detail
show ip pim neighbor
show ip mroute
```

Useful IPv6 commands include:

```text
show ipv6 mld groups detail
show ipv6 pim neighbor
show ipv6 mroute
```

The main item to verify is that source-specific receiver interest results in the expected `(S,G)` multicast routing state and forwarding interfaces.

## Section Overview

The following chapters examine SSM operation and configuration in greater detail.

- **[ASM vs SSM and SSM Address Ranges](asm-vs-ssm.md):** Differences between ASM and SSM, IPv4 and IPv6 SSM address ranges, and custom IPv4 SSM ranges.
- **[SSM Receiver Signaling and PIM Forwarding](pim-forwarding.md):** IGMPv3/MLDv2 source-specific membership, PIM `(S,G)` Join processing, and source-tree forwarding.
- **[SSM Mapping](ssm-mapping.md):** Supporting legacy IGMP and MLD receivers through group-to-source mapping.
- **[Static and DNS-Based SSM Mapping](static-ssm-mapping.md):** Configuring, verifying, and troubleshooting static and DNS-based SSM mappings for IPv4 and IPv6.

## Key Points

- **SSM identifies multicast channels using `(S,G)`**, rather than group membership alone.
- IPv4 reserves `232.0.0.0/8` for SSM; IPv6 uses `ff3x::/32`.
- Native SSM requires **IGMPv3 for IPv4** and **MLDv2 for IPv6**.
- SSM uses **source-specific distribution trees** without requiring an RP or shared tree.
- PIM `(S,G)` Joins follow the RPF path toward the specified source.
- No source registration or SPT switchover is required.
- **SSM Mapping** supports legacy receivers that cannot provide source-specific membership information.
- Static and DNS-based SSM Mapping are supported on Cisco IOS XE for both IPv4 and IPv6.
