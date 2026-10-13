
# ASM vs SSM and SSM Address Ranges

**Any-Source Multicast (ASM)** and **Source-Specific Multicast (SSM)** are two multicast service models that differ in how receivers identify the traffic they want and how the network establishes multicast distribution trees.

- **ASM:** Receivers join a multicast group `G` and can receive traffic from multiple sources sending to that group.
- **SSM:** Receivers subscribe to a specific source/group channel `(S,G)` and receive traffic only from the requested source.

Both models are supported by IPv4 and IPv6 multicast.

## ASM vs. SSM

The fundamental difference is whether the receiver must identify the multicast source.

### Any-Source Multicast (ASM)

In ASM, a receiver can request traffic for a group without knowing which sources are sending to it.

For example:

```text
IPv4 Group: 239.1.1.1
IPv6 Group: ff15::101
```

Multiple sources may send traffic to the same group.

Traditional PIM Sparse Mode uses a **Rendezvous Point (RP)** to connect sources and receivers.

The receiver initially joins a shared tree `(*,G)` toward the RP. The last-hop router may subsequently establish an `(S,G)` Shortest-Path Tree (SPT) toward an individual source.

### Source-Specific Multicast (SSM)

In SSM, the receiver must specify both the source and group.

For example:

```text
IPv4:
(10.1.1.10, 232.1.1.1)

IPv6:
(2001:db8:10::10, ff35::fd00:101)
```

Because the source is already known, the last-hop router can build an `(S,G)` tree directly toward the source.

SSM does not require an RP, shared tree, PIM Register messages, or SPT switchover.

### Comparison

| Feature | ASM | SSM |
|---|---|---|
| Receiver interest | Group `G` (optionally source-filtered) | Channel `(S,G)` |
| Source discovery | Network can discover sources | Source must already be known |
| RP | Required by traditional PIM-SM | Not required |
| Shared tree `(*,G)` | Supported | Not used |
| Source tree `(S,G)` | Supported | Used exclusively |
| PIM Register messages | Used for source discovery | Not used |
| SPT switchover | Possible | Not required |
| IPv4 receiver signaling | IGMPv1/v2/v3 | IGMPv3 |
| IPv6 receiver signaling | MLDv1/v2 | MLDv2 |

## Source-Specific Receiver Signaling Does Not Necessarily Mean SSM

**IGMPv3 and MLDv2 support source-specific membership requests for both ASM and SSM.**

For example, an IGMPv3 receiver can request:

```text
Group:   239.1.1.1
Mode:    INCLUDE
Sources: {10.1.1.10}
```

This is a source-specific request, but `239.1.1.1` is not part of the default IPv4 SSM range.

Therefore, PIM may still use traditional ASM mechanisms, including RP-based forwarding, depending on the router's configuration and implementation.

The same distinction applies to IPv6 MLDv2.

**PIM-SSM operation is determined by the multicast group being treated as an SSM group, not simply by the presence of a source list in an IGMPv3 or MLDv2 Report.**

For example, assuming the standard SSM ranges:

| Receiver Request | Multicast Service Model |
|---|---|
| IGMPv3 INCLUDE `10.1.1.10` for `232.1.1.1` | SSM |
| IGMPv3 INCLUDE `10.1.1.10` for `239.1.1.1` | ASM |
| MLDv2 INCLUDE `2001:db8:10::10` for `ff35::fd00:101` | SSM |
| MLDv2 INCLUDE `2001:db8:10::10` for `ff15::101` | ASM |

An ASM router may still establish `(S,G)` forwarding state. Therefore, **the presence of `(S,G)` state in the multicast routing table does not by itself prove that PIM is operating in SSM mode**.

## IPv4 SSM Address Range

IANA reserves the following IPv4 multicast range for SSM:

```text
232.0.0.0/8
```

This covers:

```text
232.0.0.0 - 232.255.255.255
```

RFC 4607 and the IANA registry define the following allocations:

| Address Range | Purpose |
|---|---|
| `232.0.0.0` | Reserved |
| `232.0.0.1 - 232.0.0.255` | Reserved for IANA allocation |
| `232.0.1.0 - 232.255.255.255` | Local allocation by hosts |

For normal SSM applications and labs, use addresses from the locally allocated range.

For example:

```text
232.1.1.1
232.10.10.10
232.100.1.1
```

Unlike ASM, the same SSM group address can be used by independent sources without creating the same multicast channel.

For example:

```text
(10.1.1.10, 232.1.1.1)
(10.2.2.20, 232.1.1.1)
```

These are two distinct SSM channels.

A receiver subscribed only to the first channel does not request traffic from the second source.

### IOS XE Default SSM Range

Enable the standard IPv4 SSM range using:

```text
R1(config)# ip pim ssm default
```

This configures PIM to treat `232.0.0.0/8` as SSM.

For groups in this range:

- IGMPv3 INCLUDE-mode reports can establish source-specific subscriptions.
- PIM generates source-specific `(S,G)` Joins and Prunes.
- PIM does not build an RP-based shared tree for the group.
- PIM Register messages are not used for source registration.
- MSDP Source-Active messages are not used for source discovery.

IPv4 multicast routing and PIM must still be enabled on the relevant interfaces.

### Custom IPv4 SSM Ranges

IOS XE allows a standard IPv4 ACL to define the multicast groups treated as SSM:

```text
ip pim ssm range <standard-acl>
```

For example, suppose we want both `232.0.0.0/8` and `239.10.10.0/24` to use SSM:

```text
R1(config)# ip access-list standard SSM-RANGE
R1(config-std-nacl)# permit 232.0.0.0 0.255.255.255
R1(config-std-nacl)# permit 239.10.10.0 0.0.0.255
R1(config-std-nacl)# exit
R1(config)# ip pim ssm range SSM-RANGE
```

The ACL identifies multicast **group addresses**, not source addresses.

Important points:

- Permitted groups are treated as SSM.
- Groups not permitted by the ACL are not included in this SSM range.
- The ACL defines the SSM range; it is not a data-plane packet-filtering ACL.
- A custom range should explicitly include `232.0.0.0/8` if the standard SSM range must remain included.

For example:

```text
Group          PIM Mode
-------------  ----------------
232.1.1.1      SSM
239.10.10.10   SSM
239.20.20.20   Not SSM
```

A group excluded from the SSM range may use ASM if appropriate multicast routing and RP configuration are available.

> **Important:** When customizing the SSM range, ensure that routers consistently classify the relevant groups. Overlapping or inconsistent ASM and SSM configurations can produce multicast forwarding failures.

## IPv6 SSM Address Range

IPv6 defines SSM addresses using:

```text
ff3x::/32
```

Here, `x` represents the multicast scope.

More precisely, the SSM prefix is:

```text
ff3x:0000::/32
```

The first 16 bits contain the multicast prefix, flags, and scope. The next 16 bits contain the reserved and prefix-length fields, which are zero for the standard SSM address format.

### IPv6 SSM Address Format

The standard IPv6 SSM address format is:

```text
| 8 bits | 4 bits | 4 bits | 8 bits | 8 bits | 64 bits | 32 bits |
+--------+--------+--------+--------+--------+---------+---------+
|   FF   | Flags  | Scope  |Reserved|  Plen  | Prefix  | Group ID|
+--------+--------+--------+--------+--------+---------+---------+
             0011             0        0         0
```

The multicast flags are:

```text
0 R P T = 0011
```

- `R = 0`: No Embedded-RP information.
- `P = 1`: Unicast-prefix-based multicast address format.
- `T = 1`: Transient multicast address.

For SSM, the prefix length (`Plen`) and network-prefix field are zero.

The resulting format is:

```text
ff3x:0000:0000:0000:0000:0000:GGGG:GGGG
```

The last 32 bits identify the group.

### IPv6 SSM Scopes

The fourth hexadecimal digit identifies the IPv6 multicast scope.

| Prefix | Scope |
|---|---|
| `ff32:0000::/32` | Link-Local SSM |
| `ff33:0000::/32` | Realm-Local SSM |
| `ff34:0000::/32` | Admin-Local SSM |
| `ff35:0000::/32` | Site-Local SSM |
| `ff38:0000::/32` | Organization-Local SSM |
| `ff3e:0000::/32` | Global SSM |

For example:

```text
ff35::fd00:101
```

is a site-local SSM group.

```text
ff3e::fd00:101
```

is a global-scope SSM group.

The multicast scope still applies to SSM. For example, link-local SSM traffic cannot be routed beyond the local link.

> **Note:** Do not confuse `ff3x::/32` with the broader `ff3x::/16` range. The latter also includes other unicast-prefix-based multicast addresses. RFC 4607 specifies that routers treat the `ff3x:0000::/32` space as SSM, while standard SSM group allocation occurs within `ff3x::/96`.

### IPv6 SSM Group IDs

The standard SSM allocation space is:

```text
ff3x::/96
```

The final 32 bits contain the group ID.

RFC 10028 updates the dynamic IPv6 multicast group-ID allocations.

Important group-ID ranges include:

| Group ID | Purpose |
|---|---|
| `F0000000 - FCFFFFFF` | Host allocation for SSM |
| `FD000000 - FDFFFFFF` | Private Use |
| `FE000000 - FEFFFFFF` | Experimental Use |
| `FF000000 - FFFFFFFF` | Reserved for Solicited-Node multicast Group IDs |

For example:

```text
ff35::fd00:101
```

uses a private-use group ID.

This is a convenient choice for CCIE labs because the address is locally allocated and uses site-local scope.

### IPv6 SSM on IOS XE

IPv6 SSM is supported as part of IPv6 multicast routing.

Enable IPv6 multicast routing using:

```text
R1(config)# ipv6 multicast-routing
```

The standard IPv6 SSM ranges are recognized by the IPv6 PIM implementation. Unlike IPv4, the usual IOS XE configuration does not require an equivalent of `ip pim ssm default`.

Native IPv6 SSM requires MLDv2 source-specific receiver signaling.

The router uses `(S,G)` PIM forwarding for groups classified as SSM, without requiring an RP.

## SSM Group Address Reuse

Because SSM channels are identified by `(S,G)` rather than just `G`, different sources can reuse the same group address.

For example:

```text
Source S1 → Group G
Source S2 → Group G
```

These are independent channels:

```text
(S1,G)
(S2,G)
```

This reduces the need to allocate a globally unique group address to every application.

However, **Layer 2 forwarding can still introduce limitations**.

Switches use multicast snooping entries based on `(*,G)` or group-level forwarding rather than distinguishing every `(S,G)` channel.

If two receivers request different sources using the same group, a switch without source-specific forwarding may deliver both streams to their ports, leaving the hosts to filter unwanted packets.

For this reason, using different multicast group addresses for independent applications can still be beneficial.

## Verification

### IPv4

```text
R1# show ip igmp groups detail
R1# show ip mroute
R1# show ip pim rp mapping
R1# show running-config | include ip pim ssm
```

Verify that:

- The required groups fall within the configured SSM range.
- IGMPv3 reports identify the expected sources.
- SSM forwarding uses `(S,G)` state.
- The SSM groups do not depend on an RP.

### IPv6

```text
R1# show ipv6 mld groups detail
R1# show ipv6 mroute
R1# show ipv6 pim range-list
R1# show ipv6 pim group-map
```

`show ipv6 pim range-list` is particularly useful because it displays the IPv6 multicast ranges classified as SSM or associated with RP mappings.

Verify that SSM groups are classified correctly and that the multicast routing table contains the expected source-specific entries.

## Key Points

- **ASM identifies multicast reception primarily by group `G`**, while SSM identifies individual channels `(S,G)`.
- Traditional PIM-SM ASM uses an RP and shared tree. SSM does not.
- **IGMPv3 and MLDv2 source-specific Reports do not automatically imply PIM-SSM operation.** The group must be treated as SSM.
- IPv4 reserves `232.0.0.0/8` for SSM.
- IOS XE uses `ip pim ssm default` for the standard IPv4 range or `ip pim ssm range <acl>` for custom ranges.
- IPv6 SSM uses `ff3x:0000::/32`, with standard group allocation within `ff3x::/96`.
- IPv6 SSM addresses include a multicast scope that limits their distribution.
- Different SSM channels can reuse the same group address, although Layer 2 snooping behavior may limit the efficiency of that reuse.
- An `(S,G)` multicast routing entry alone does not prove the router is operating in SSM mode.
