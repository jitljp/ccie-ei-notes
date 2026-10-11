# PIM Join and Prune Messages

**PIM Join/Prune messages** are used to establish, maintain, and remove multicast distribution-tree branches in PIM Sparse Mode (PIM-SM).

They are the primary mechanism through which downstream PIM routers communicate multicast forwarding requirements to upstream routers.

- **Join:** Requests that an upstream router forward multicast traffic toward the downstream router.
- **Prune:** Requests that an upstream router stop forwarding particular multicast traffic toward the downstream router.

Unlike PIM Dense Mode, which initially floods traffic, **PIM-SM relies on explicit Joins to establish multicast distribution trees**.

## PIM Join/Prune Message Characteristics

PIM Joins and Prunes are not separate PIM message types. They are carried within the same **PIM Join/Prune message (Type 3)**.

| Field | IPv4 | IPv6 |
|---|---|---|
| Destination | `224.0.0.13` | `FF02::D` |
| IP Protocol / Next Header | 103 | 103 |
| TTL / Hop Limit | 1 | 1 |
| PIM Version | 2 | 2 |
| PIM Message Type | 3 | 3 |

Join/Prune messages are multicast to all PIM routers on the local link, but contain an **Upstream Neighbor Address** identifying the router to which the message is directed.

Therefore, other PIM routers on a shared LAN can observe the message even though only one neighbor is its intended target.

This is important for Join suppression and Prune override.

## Join/Prune Message Format

The PIM Join/Prune message format is defined in RFC 7761.

```text
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |  Ver  | Type=3|   Reserved    |           Checksum            |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |             Upstream Neighbor Address (Encoded)               |
 |                             ...                               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |   Reserved    |  Num Groups   |           Holdtime            |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                  Multicast Group Address 1                    |
 |                             ...                               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |    Number of Joined Sources   |    Number of Pruned Sources   |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                    Joined Source Address 1                    |
 |                             ...                               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                    Pruned Source Address 1                    |
 |                             ...                               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                Additional Sources and Groups                  |
 |                             ...                               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The addresses are variable-length PIM encodings, depending on the address family.

Important fields include:

| Field | Description |
|---|---|
| Upstream Neighbor Address | Identifies the intended upstream PIM neighbor |
| Number of Groups | Number of multicast group records in the message |
| Holdtime | Lifetime of the advertised Join/Prune state |
| Multicast Group Address | Group to which the following entries apply |
| Number of Joined Sources | Number of source entries in the Join list |
| Number of Pruned Sources | Number of source entries in the Prune list |
| Joined Sources | State that the sender wishes to receive |
| Pruned Sources | State that the sender does not wish to receive |

A single Join/Prune message can contain multiple multicast groups, and each group can contain multiple Join and Prune entries.

For example, a single message might contain:

```text
Upstream Neighbor: 10.0.12.1
Holdtime:          210 seconds

Group: 239.1.1.1
  Joined:
    (10.10.10.10, 239.1.1.1)
  Pruned:
    (10.20.20.20, 239.1.1.1)

Group: 239.1.1.2
  Joined:
    (*, 239.1.1.2)
```

> The Holdtime applies to the Join/Prune message, rather than having a separate Holdtime for each individual source entry.

### Encoded Source Address Flags

The encoded source address contains three important flags:

```text
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | Addr Family   | Encoding Type | Rsrvd   |S|W|R|  Mask Len     |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                       Source Address                          |
 |                             ...                               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| Flag | Name | Meaning |
|---|---|---|
| S | Sparse | Identifies Sparse Mode encoding; used for PIMv1 compatibility |
| W | Wildcard (WC) | Identifies wildcard-source `(*,G)` state |
| R | RPT | Identifies state associated with the shared tree |

The **W and R flags distinguish the three important PIM-SM state types**.

| State | W | R | Address Encoded | Lookup Toward |
|---|---|---|---|---|
| `(*,G)` | 1 | 1 | RP | RP |
| `(S,G)` | 0 | 0 | Source S | Source S |
| `(S,G,rpt)` | 0 | 1 | Source S | RP |

For `(*,G)` entries, the encoded source address is actually the **RP address**, not `0.0.0.0`.

For example, a `(*,239.1.1.1)` Join with RP `1.1.1.1` uses:

```text
Group Address:   239.1.1.1
Source Address:  1.1.1.1
Wildcard:        1
RPT:             1
```

This is an important detail when interpreting PIM packet captures.

## Types of PIM Joins and Prunes

PIM-SM uses three types of Join/Prune state.

| Message | Purpose |
|---|---|
| `Join(*,G)` | Request shared-tree traffic for group G |
| `Prune(*,G)` | Remove shared-tree interest for group G |
| `Join(S,G)` | Request source-specific traffic along the SPT |
| `Prune(S,G)` | Remove source-specific SPT interest |
| `Join(S,G,rpt)` | Restore interest in source S through the shared tree |
| `Prune(S,G,rpt)` | Remove source S from shared-tree forwarding |

### (*,G) Join/Prune

A `(*,G)` Join requests traffic from the RP-rooted shared tree.

For example:

```text
Join(*, 239.1.1.1)
```

This is sent upstream toward the RP.

An upstream router receiving the Join adds the downstream interface to its shared-tree forwarding state.

A `Prune(*,G)` removes that shared-tree interest.

### (S,G) Join/Prune

An `(S,G)` Join requests traffic from a particular source along the source tree.

For example:

```text
Join(10.10.10.10, 239.1.1.1)
```

This is sent upstream toward `10.10.10.10`, rather than toward the RP.

An `(S,G)` Prune removes the corresponding source-tree interest.

### (S,G,rpt) Join/Prune

An `(S,G,rpt)` entry controls traffic from a specific source **on the shared tree**.

For example:

```text
Prune(10.10.10.10, 239.1.1.1, RPT)
```

This is sent toward the RP, commonly during SPT switchover.

It tells the upstream router that traffic from `10.10.10.10` is no longer needed through the shared tree.

However, traffic from other sources can continue along the RPT.

An `(S,G,rpt)` Join can reverse an earlier source-specific RPT Prune, restoring the source's eligibility for shared-tree forwarding.

> **Important:** `(S,G)` Prune and `(S,G,rpt)` Prune are different. The former concerns source-tree forwarding toward S, while the latter concerns shared-tree forwarding toward the RP.

## Periodic Join/Prune Messages

PIM-SM uses a **soft-state mechanism**.

A Join does not establish permanent multicast forwarding state. Instead, routers periodically refresh their upstream Join state.

RFC 7761 defines the following defaults:

| Timer | Default |
|---|---|
| Join/Prune Period | 60 seconds |
| Join/Prune Holdtime | 210 seconds |

The recommended Holdtime is:

```text
Holdtime = 3.5 × Join/Prune Period
         = 3.5 × 60
         = 210 seconds
```

### Example: Periodic Join Refresh

Suppose R2 joins group `239.1.1.1` through R1.

```text
R1 (Upstream)
 |
 |
R2 (Downstream)
```

R2 sends:

```text
Time 0:
  Join(*,239.1.1.1)
  Holdtime = 210 seconds

Time 60:
  Join(*,239.1.1.1)
  Holdtime = 210 seconds

Time 120:
  Join(*,239.1.1.1)
  Holdtime = 210 seconds
```

Each Join refreshes the downstream state maintained by R1.

If R2 stops sending Joins, the state eventually expires.

For example:

```text
Time 120:
  Last Join received.

Time 330:
  210 seconds have elapsed.
  R1 removes the expired downstream Join state.
```

If R1 no longer has any downstream interest, it can remove its own upstream Join state.

**Periodic Joins are refresh messages, not new membership requests.**

A router also sends triggered Join/Prune messages when forwarding requirements change, rather than always waiting for the next periodic interval.

> **Note:** The 210-second Join/Prune Holdtime is different from the default 105-second PIM Hello Holdtime. They maintain different protocol state.

### Join/Prune Holdtime 0xFFFF

The Join/Prune Holdtime field supports the special value `0xFFFF` (65535).

This requests that the receiver retain the Join/Prune state until it is explicitly canceled or removed according to local policy.

The purpose is to avoid periodic refresh traffic on certain types of links, such as dial-on-demand connections.

## Join Suppression

On multiaccess networks, multiple downstream routers may request the same multicast traffic from the same upstream router.

Sending identical periodic Join messages from every downstream router can be unnecessary.

**PIM Join suppression** allows one router to delay its own Join refresh when it observes another router sending an equivalent Join.

### Example: Join Suppression

Consider:

```text
              R1 (Upstream)
                   |
                   |
                  SW1
                 /   \
                /     \
               R2     R3
               |       |
              PC1     PC2
```

Both R2 and R3 require traffic for `239.1.1.1`.

Both use R1 as their upstream PIM neighbor.

Suppose R2 sends:

```text
R2 → All PIM Routers:
  Upstream Neighbor: R1
  Join(*,239.1.1.1)
```

R3 also receives this message because it is multicast onto the shared LAN.

R3 can recognize that R2 has already requested the same shared-tree traffic from the same upstream router.

Instead of sending an immediate redundant Join refresh, R3 can delay its own next Join.

The RFC suppression interval is randomized between:

```text
1.1 × Join/Prune Period
```

and:

```text
1.4 × Join/Prune Period
```

With the 60-second default:

```text
Suppression Interval = 66–84 seconds
```

This reduces unnecessary periodic Join transmissions on shared networks.

**Join suppression does not remove R3's multicast interest.** It only suppresses redundant signaling while the upstream router continues forwarding onto the shared LAN.

## Prune Override on Multiaccess Networks

A Prune from one downstream router must not cause an upstream router to stop forwarding traffic needed by another router on the same LAN.

PIM handles this using **Prune Pending** and **Join override** mechanisms.

### Example: One Router Leaves, Another Still Needs Traffic

Consider:

```text
              R1 (Upstream)
                   |
                   |
                  SW1
                 /   \
                /     \
               R2     R3
               |       |
              PC1     PC2
```

Both R2 and R3 initially require `239.1.1.1`.

Suppose PC1 leaves the group, but PC2 remains interested.

R2 sends:

```text
R2 → R1: Prune(*,239.1.1.1)
```

If R1 immediately stopped forwarding onto SW1, R3 would lose the traffic needed by PC2.

Instead, R1 enters **Prune-Pending** state for the affected group and outgoing interface.

R3 hears R2's Prune and sends a Join override:

```text
R3 → R1: Join(*,239.1.1.1)
```

R1 cancels the pending Prune.

Multicast forwarding continues onto SW1.

```text
R2 sends Prune
       |
       v
R1 enters Prune-Pending
       |
       v
R3 sends Join override
       |
       v
R1 cancels pending Prune
       |
       v
Forwarding continues
```

If no router sends a Join override before the Prune-Pending Timer expires, R1 can remove the outgoing interface from the affected forwarding state.

### Join/Prune Override Timer

RFC 7761 defines the default parameters:

| Parameter | Default |
|---|---|
| Propagation Delay | 500 ms |
| Override Interval | 2500 ms |
| Join/Prune Override Interval | 3000 ms |

The effective delay is:

```text
Join/Prune Override Interval
    = Propagation Delay + Override Interval

    = 500 ms + 2500 ms
    = 3000 ms
```

The upstream router waits up to approximately **3 seconds** before completing the Prune on a shared LAN.

A downstream router that still requires the traffic schedules its Join override after a randomized delay between 0 and the effective Override Interval (2.5 seconds by default).

The delay allows other interested routers to prevent unnecessary traffic interruption.

The values can be negotiated through the **LAN Prune Delay Hello option** when all PIM neighbors on the link support it.

> **Note:** This Join/Prune Override Interval is different from the **Assert Override Interval**. Both default to 3 seconds, but they serve different purposes.

### Prune Echo

If a Prune-Pending Timer expires and the upstream router stops forwarding, the router can transmit a **Prune Echo**.

A Prune Echo is a PIM Prune message sent by the upstream router with its own address in the Upstream Neighbor Address field.

This provides another opportunity for downstream routers to detect the Prune and send a Join override if needed.

It is particularly useful if a downstream router missed the original Prune.

## Join/Prune Processing on Point-to-Point Links

Join suppression and Prune override are primarily relevant to multiaccess networks.

On a point-to-point link with only one downstream PIM neighbor, an upstream router does not need to wait for another downstream router to override a Prune.

Therefore, Prune processing can occur immediately.

The main Join/Prune functions remain the same:

- Join requests downstream forwarding.
- Prune removes downstream forwarding interest.
- Periodic Joins refresh soft state.

### Join/Prune Interval Configuration

The default PIM Join/Prune interval is **60 seconds** for both IPv4 and IPv6.

On IOS XE, the interval can be configured per interface.

**IPv4:**

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip pim join-prune-interval 75
```

**IPv6:**

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 pim join-prune-interval 75
```

Both commands configure a periodic Join/Prune interval of 75 seconds.

> **Note:** These commands control periodic Join/Prune transmissions. Triggered Join/Prune messages can still be sent immediately when multicast forwarding requirements change.

### Verification Commands

| Command | Purpose |
|---|---|
| `show ip mroute` | Examine multicast forwarding state and OILs |
| `show ip pim neighbor` | Verify PIM neighbors |
| `show ip pim interface` | Verify PIM-enabled interfaces |
| `show ip rpf <address>` | Verify the selected upstream RPF path |
| `show ipv6 mroute` | Examine IPv6 multicast forwarding state |
| `show ipv6 pim join-prune statistic` | Display IPv6 Join/Prune aggregation statistics |
| `show ipv6 pim traffic` | Examine IPv6 PIM message counters |
| `debug ip pim` | Observe IPv4 PIM control-plane events |

For example:

```text
R1# show ip mroute 239.1.1.1
```

Illustrative output:

```text
(*, 239.1.1.1), 00:05:00/00:02:59, RP 1.1.1.1, flags: S
  Incoming interface: Ethernet0/0, RPF nbr 10.0.12.2
  Outgoing interface list:
    Ethernet0/1, Forward/Sparse, 00:05:00/00:02:59
```

The outgoing interface indicates downstream multicast interest, which can be maintained through received PIM Joins or local receiver memberships.

For detailed PIM signaling, use:

```text
R1# debug ip pim
```

Disable debugging when finished:

```text
R1# undebug all
```

**Packet captures are particularly useful for Join/Prune analysis**, because they reveal the Upstream Neighbor Address, Holdtime, Joined and Pruned source lists, and encoded source flags.

## Key Points

- **PIM Join/Prune is Message Type 3**, containing separate Joined and Pruned source lists.
- Join/Prune messages are multicast to `224.0.0.13` or `FF02::D`, but identify a specific upstream neighbor.
- **`(*,G)` Join/Prune** controls shared-tree interest toward the RP.
- **`(S,G)` Join/Prune** controls source-tree interest toward the source.
- **`(S,G,rpt)` Join/Prune** controls a particular source's traffic on the shared tree.
- The encoded source address's **W and R flags** distinguish the three state types.
- PIM-SM uses soft state with a default **60-second Join/Prune Period** and **210-second Holdtime**.
- **Join suppression** reduces redundant Join refreshes on shared LANs.
- **Prune override** allows another interested router to prevent a shared interface from being pruned.
- The default **Join/Prune Override Interval is 3 seconds**.
- PIM Join/Prune operation is automatic when PIM-SM is enabled.