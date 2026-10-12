# Bootstrap Messages and RP-Set

In the PIM Bootstrap Router (BSR) mechanism, **Bootstrap Messages (BSMs)** distribute information about the elected BSR and available Rendezvous Points (RPs) throughout the PIM domain.

The elected BSR collects **Candidate-RP-Advertisements** from Candidate RPs (C-RPs), constructs an **RP-Set**, and periodically advertises that information in BSMs.

All PIM routers receiving the RP-Set can independently determine the RP for a particular multicast group.

Unlike Auto-RP, **BSR uses PIM control messages forwarded hop by hop**, so no preexisting multicast distribution tree or RP mapping is required.

## Bootstrap Message Operation

Consider the following network:

```text
                 R1 (BSR)
                  1.1.1.1
                     |
                     R2
                   /    \
                  /      \
             R3 (C-RP)  R4 (C-RP)
              3.3.3.3    4.4.4.4
                           |
                           R5
```

Assume:

- R1 is the elected BSR.
- R3 and R4 are Candidate RPs for `239.0.0.0/8`.
- R3 advertises RP priority 10.
- R4 advertises RP priority 20.

### Step 1: Elected BSR Advertises Its Address

Once R1 becomes the elected BSR, it originates **Bootstrap Messages (BSMs)** identifying itself as the BSR.

```text
R1 (BSR)
    |
    | BSM
    | BSR Address: 1.1.1.1
    v
    R2
   /  \
  R3  R4
(C-RP)(C-RP)
```

The BSMs are forwarded hop by hop throughout the PIM domain.

R3 and R4 receive the BSMs and learn that `1.1.1.1` is the elected BSR.

**The first BSM may contain an empty RP-Set**, because the newly elected BSR has not yet received any Candidate-RP-Advertisements.

### Step 2: Candidate RPs Advertise to the BSR

After learning the elected BSR's address, R3 and R4 unicast their **Candidate-RP-Advertisement (C-RP-Adv)** messages to `1.1.1.1`.

```text
R3 (C-RP) --------\
                   \
                    > R1 (BSR)
                   /
R4 (C-RP) --------/
```

The advertisements identify the Candidate RP addresses, group ranges, priorities, and Holdtimes.

When a new BSR is elected, Candidate RPs send **one to three triggered advertisements** after short randomized delays, rather than necessarily waiting for their next periodic advertisement.

### Step 3: BSR Constructs the RP-Set

R1 collects the Candidate RP information and constructs its RP-Set.

For example:

| Group Range | Candidate RP | RP Priority | Holdtime |
|---|---|---|---|
| `239.0.0.0/8` | `3.3.3.3` | 10 | 150 seconds |
| `239.0.0.0/8` | `4.4.4.4` | 20 | 150 seconds |

The BSR can apply local policies to restrict which Candidate RPs are accepted or included.

**The RP-Set contains candidate mappings, not just the winning RP for each group.**

### Step 4: BSR Distributes the RP-Set

R1 sends subsequent Bootstrap Messages containing the RP-Set.

```text
               R1 (BSR)
                   |
                   | BSM + RP-Set
                   v
                   R2
                  /  \
                 /    \
                R3    R4
                       |
                       v
                       R5
```

The BSMs are forwarded hop by hop, allowing all PIM routers in the domain to learn the RP-Set.

The BSR normally sends BSMs every **60 seconds**, but can send an updated BSM when the RP-Set changes, subject to the **10-second minimum interval**.

### Step 5: Routers Select an RP

After receiving the RP-Set, each router independently determines the RP for a multicast group.

For `239.1.1.1`, both advertised RPs match:

```text
3.3.3.3 - Priority 10
4.4.4.4 - Priority 20
```

R5 selects `3.3.3.3` because **lower RP priorities are preferred**.

If the priorities were equal, RP hashing would determine the selected RP.

See [BSR Hashing](hashing.md).

## Bootstrap Message Characteristics

Bootstrap Messages use **PIM message type 4**.

| Field | IPv4 | IPv6 |
|---|---|---|
| Destination | `224.0.0.13` | `FF02::D` |
| IP Protocol / Next Header | 103 | 103 |
| PIM Version | 2 | 2 |
| PIM Message Type | **4 (Bootstrap)** | **4 (Bootstrap)** |
| TTL / Hop Limit | 1 | 1 |
| Transport | Normally multicast | Normally multicast |

BSMs are **not unicast from the BSR directly to every router**.

Instead, each intermediate PIM router creates a new IP packet carrying the received BSM and transmits it onto the appropriate PIM-enabled interfaces.

Therefore, the IP source address changes at every hop.

For example:

```text
R1 → R2:

IP Source:      R1's outgoing interface
IP Destination: 224.0.0.13
BSR Address:    1.1.1.1

R2 → R4:

IP Source:      R2's outgoing interface
IP Destination: 224.0.0.13
BSR Address:    1.1.1.1
```

Notice that the **BSR Address field remains `1.1.1.1`**, even though R2 is now transmitting the message.

For IPv6, the outer source address is normally the transmitting interface's IPv6 link-local address.

## Bootstrap Message Format

RFC 5059 defines the BSM format.

```text
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |PIM Ver| Type=4|N|  Reserved   |           Checksum            |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |         Fragment Tag          | Hash Mask Len | BSR Priority  |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |             BSR Address (Encoded-Unicast format)              |
 |                              ...                              |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |             Group Address (Encoded-Group format)              |
 |                              ...                              |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |   RP Count    |  Frag RP Cnt  |           Reserved            |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |             RP Address (Encoded-Unicast format)               |
 |                              ...                              |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |          RP Holdtime          |  RP Priority  |   Reserved    |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                     Additional RP entries                     |
 |                              ...                              |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                   Additional Group ranges                     |
 |                              ...                              |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The BSR address, group addresses, and RP addresses use variable-length PIM address encodings.

### Important Fields

| Field | Purpose |
|---|---|
| **N-bit** | No-Forward flag, preventing forwarding when set |
| **Fragment Tag** | Identifies fragments belonging to the same BSM |
| **Hash Mask Length** | Controls which group address bits are used during RP hashing |
| **BSR Priority** | Priority of the BSR identified in the message |
| **BSR Address** | Address of the BSR responsible for the advertised information |
| **Group Address** | Multicast group prefix associated with the candidate RPs |
| **RP Count** | Total number of RP addresses for the group range across the complete BSM |
| **Frag RP Cnt** | Number of RP addresses for that group range in this particular fragment |
| **RP Address** | Address of an advertised candidate RP |
| **RP Holdtime** | Lifetime of the advertised RP mapping |
| **RP Priority** | RP selection priority; lower values are preferred |

A BSM can include multiple group ranges, and each group range can have multiple RPs.

### BSR Priority vs. RP Priority

These are distinct fields within the BSM.

For example:

```text
BSR Priority:  20

Group: 239.0.0.0/8
  RP 3.3.3.3, Priority 10
  RP 4.4.4.4, Priority 20
```

- **BSR Priority:** Used for the BSR election. Higher wins.
- **RP Priority:** Used for RP selection. Lower wins.

The BSR priority does not determine which candidate RP is selected for a multicast group.

## Bootstrap Message Forwarding

BSMs are propagated hop by hop throughout the PIM domain.

Before forwarding a normal BSM, the receiving router performs several checks.

These include:

1. The message must be received from a valid PIM neighbor.
2. The message must pass the required **RPF neighbor check toward the BSR**.
3. The advertised BSR must be acceptable according to the BSR election state.
4. The message must not be blocked by a configured BSR boundary.

### BSM RPF Check

Consider:

```text
               R1 (BSR)
                   |
                   R2
                  /  \
                 R3  R4
```

R3's RPF lookup toward BSR `1.1.1.1` identifies R2 as its upstream neighbor.

Therefore:

```text
BSM from R2 identifying BSR 1.1.1.1

RPF Neighbor Check: PASS
```

However, if R3 receives a normal BSM identifying the same BSR from another neighbor that is not its RPF neighbor:

```text
BSM from unexpected PIM neighbor

RPF Neighbor Check: FAIL
```

The message is rejected.

**The RPF check is toward the BSR address, not toward a Candidate RP address.**

This prevents incorrectly forwarded BSMs from being accepted and helps avoid loops.

### Forwarding on Shared LANs

An important RFC 5059 detail is that a router may forward an accepted BSM **back out the same multiaccess interface on which it was received**.

Consider:

```text
             Toward BSR
                 |
                 R1
                 |
          Shared Ethernet
            /        \
           R2        R3
```

Suppose R3's RPF neighbor toward the BSR is R2, rather than R1.

If R1 sends a BSM onto the LAN, R3 may reject it because R1 is not its expected RPF neighbor.

However, R2 can receive the BSM and retransmit it on the same LAN.

R3 can then accept the copy from R2.

This is why BSM forwarding on a shared LAN is not necessarily limited to interfaces other than the incoming interface.

On point-to-point links, forwarding a BSM back onto the incoming interface is normally unnecessary.

## RP-Set Maintenance and Expiration

RP mappings learned through BSMs are maintained as soft state.

Each group-to-RP mapping includes an **RP Holdtime**.

For example:

```text
Group: 239.0.0.0/8
RP:    3.3.3.3
Holdtime: 150 seconds
```

When a router receives a valid BSM advertising this mapping:

1. It adds the mapping if it does not already exist.
2. It records the advertised RP priority.
3. It starts or refreshes the **Group-to-RP Mapping Expiry Timer** using the advertised Holdtime.

If subsequent BSMs refresh the mapping before the timer expires, it remains valid.

If the mapping expires, it is removed.

### RP Holdtime vs. BSR Timeout

These timers serve different purposes.

| Timer | Default or Advertised Value | Purpose |
|---|---|---|
| **Bootstrap Period** | 60 seconds | Periodic BSM generation |
| **Bootstrap Timeout** | 130 seconds | Expiration of learned BSR information |
| **RP Holdtime** | Carried per RP entry, commonly 150 seconds | Expiration of the group-to-RP mapping |

The RP Holdtime carried in a BSM is derived from the Candidate RP information, but the BSR can adjust it to ensure that it is sufficiently long relative to the Bootstrap Period.

RFC 5059 requires the advertised RP Holdtime to be greater than the Bootstrap Period and recommends a value greater than 2.5 times that period, except for intentional withdrawal.

### RP Withdrawal

An **RP Holdtime of 0** in a BSM immediately withdraws the corresponding mapping.

For example:

```text
Group: 239.0.0.0/8
RP:    3.3.3.3
Holdtime: 0
```

Receiving routers remove that mapping immediately rather than waiting for a previously advertised Holdtime to expire.

This is useful when an RP is intentionally withdrawn.

If multiple candidate RPs remain for the group, the router can select another RP.

## Periodic and Triggered BSMs

The elected BSR periodically generates BSMs to refresh the RP-Set throughout the domain.

RFC 5059 defines:

| Timer | Default |
|---|---|
| Bootstrap Period | **60 seconds** |
| Minimum Bootstrap Interval | **10 seconds** |

The BSR can also generate a BSM when its RP-Set changes.

For example:

```text
Time 0:
  BSR advertises R3 and R4 as Candidate RPs.

Time 20:
  A new Candidate RP, R5, advertises itself.

Time 30:
  BSR can advertise an updated RP-Set.

Time 90:
  Next regular BSM.
```

The BSR must observe the minimum interval between originated BSMs.

An interesting consequence is that **the first BSM after a new BSR is elected may contain an empty RP-Set**, because the new BSR has not yet received Candidate RP advertisements.

After it learns the candidates, it can generate an updated BSM without waiting for the full 60-second period, subject to the minimum interval.

## Bootstrap Message Fragmentation

An RP-Set may contain many multicast group ranges and candidate RPs.

For example:

```text
Group: 239.1.0.0/16
  RP1
  RP2
  RP3

Group: 239.2.0.0/16
  RP4
  RP5

Group: 239.3.0.0/16
  RP6
  RP7
  RP8
```

If the resulting BSM would exceed the packet size limit, RFC 5059 supports **semantic fragmentation**.

Unlike ordinary IP fragmentation, semantic fragmentation divides the RP-Set across multiple independently formatted PIM Bootstrap Message fragments.

### Fragment Tag

All fragments belonging to the same BSM carry the same **Fragment Tag**.

For example:

```text
BSM Fragment 1:
  Fragment Tag: 1234
  Group: 239.1.0.0/16
  RPs: RP1, RP2

BSM Fragment 2:
  Fragment Tag: 1234
  Group: 239.1.0.0/16
  RPs: RP3
```

The receiving router recognizes that the fragments belong to the same BSM.

### RP Count vs. Frag RP Cnt

Two fields are particularly important.

Suppose three RPs are advertised for `239.1.0.0/16`, but they cannot all fit into one fragment.

```text
Fragment 1:

Group:        239.1.0.0/16
RP Count:     3
Frag RP Cnt:  2
RP Addresses: RP1, RP2
```

```text
Fragment 2:

Group:        239.1.0.0/16
RP Count:     3
Frag RP Cnt:  1
RP Addresses: RP3
```

The fields mean:

- **RP Count = 3:** Three RPs are advertised for this group range across the complete BSM.
- **Frag RP Cnt = 2 or 1:** Number of RPs carried in that particular fragment.

The receiving router waits until it has received the advertised number of RP addresses before replacing its existing RP-Set information for that group range.

**If only part of a group range's advertised RP list arrives, the incomplete replacement is discarded.**

This prevents the receiver from replacing a previously complete RP list with a partial list caused by lost fragments.

Semantic fragmentation is especially relevant in large PIM domains with numerous Candidate RPs.

## No-Forward Bootstrap Messages

RFC 5059 defines a special Bootstrap Message with the **No-Forward (N) bit** set.

These messages are primarily used to help newly discovered or rebooting PIM neighbors learn BSR information quickly.

### Example

Consider:

```text
             R1 (BSR)
                 |
                 R2
                 |
                 R3 (New Router)
```

R3 establishes a new PIM neighbor relationship with R2.

Without additional signaling, R3 might have to wait until the next periodic BSM before learning the BSR and RP-Set.

Instead, R2 can send a stored copy of the BSM to R3 when it detects the new neighbor.

```text
R2 → R3:

PIM Bootstrap Message
N-bit: 1
BSR: 1.1.1.1
RP-Set: ...
```

R3 processes the BSM and learns the BSR and RP information.

However, because the N-bit is set, **R3 does not forward the BSM to other routers**.

### No-Forward BSM Characteristics

| Property | Normal BSM | No-Forward BSM |
|---|---|---|
| PIM Type | 4 | 4 |
| N-bit | 0 | 1 |
| Destination | All PIM Routers | All PIM Routers |
| Forwarded hop by hop | Yes | No |
| BSM RPF neighbor check | Required | Not performed |
| Main purpose | RP-Set distribution | Inform new or rebooting neighbors |

The **No-Forward bit is not a separate PIM message type**. It is a flag in the normal BSM format.

On a multiaccess LAN, the PIM Designated Router normally sends the stored BSM to a newly discovered neighbor.

RFC 5059 also allows **unicast BSMs** for backward compatibility.

These are sent directly to the new neighbor rather than to the All PIM Routers group.

> **Note:** No-Forward and unicast BSMs have special acceptance rules and are intended for startup synchronization. They are not replacements for normal periodic BSM distribution.

## IPv6 Scoped BSMs

IPv6 supports administratively scoped multicast groups.

For example:

| Multicast Range | Scope |
|---|---|
| `FF15::/16` | Site-local |
| `FF18::/16` | Organization-local |
| `FF1E::/16` | Global |

The BSR mechanism can operate independently for different multicast scope zones.

A router can therefore learn different BSRs and RP-Sets for different scopes.

For example:

```text
Site-local scope (5):
  BSR: 2001:DB8:1::1
  RP:  2001:DB8:3::3

Organization-local scope (8):
  BSR: 2001:DB8:2::2
  RP:  2001:DB8:4::4
```

A **scoped BSM** identifies the scope zone to which its RP information applies.

The first group range in a scoped BSM identifies the scope zone. Additional group ranges in that BSM must belong to the same zone.

Scoped BSMs must not cross the corresponding administrative scope boundaries.

This allows separate regions to maintain independent RP mappings for scoped multicast groups.

## IOS XE Configuration

BSM generation and forwarding operate automatically when BSR is configured and PIM is operational.

There is no separate command required to enable ordinary Bootstrap Message generation.

### IPv4 Example

Configure R1 as the Candidate BSR:

```text
R1(config)# interface Loopback0
R1(config-if)# ip address 1.1.1.1 255.255.255.255
R1(config-if)# ip pim sparse-mode

R1(config)# ip pim bsr-candidate Loopback0 30 10
```

Configure R3 as a Candidate RP:

```text
R3(config)# ip pim rp-candidate Loopback0
```

Once R1 becomes the elected BSR and receives R3's advertisement, it can include R3 in its Bootstrap Messages.

### IPv6 Example

Configure an IPv6 Candidate BSR:

```text
R1(config)# ipv6 pim bsr candidate bsr 2001:DB8:1::1 126 priority 10
```

Configure an IPv6 Candidate RP:

```text
R3(config)# ipv6 pim bsr candidate rp 2001:DB8:3::3 priority 10
```

The BSR distributes the learned RP information through IPv6 Bootstrap Messages.

### IPv6 BSR-Announced RP Mappings

IOS XE also allows the elected IPv6 BSR to advertise a statically configured RP mapping without first learning it through a C-RP advertisement.

For example, on the Candidate BSR:

```text
R1(config)# ipv6 pim bsr announced rp 2001:DB8:3::3 priority 10
```

This allows the BSR to include the RP in its advertised mappings even if that RP does not participate in BSR Candidate RP signaling.

A group list can restrict the advertised mapping:

```text
R1(config)# ipv6 access-list RP-GROUPS-V6
R1(config-ipv6-acl)# permit ipv6 any FF15::/16

R1(config)# ipv6 pim bsr announced rp 2001:DB8:3::3 group-list RP-GROUPS-V6 priority 10
```

**This is different from `ipv6 pim rp-address`**, which configures a static RP mapping locally.

The `ipv6 pim bsr announced rp` command causes the elected BSR to **distribute the mapping to other routers through BSMs**.

## BSR Borders

To prevent Bootstrap Messages from crossing an interface, configure a BSR border.

IPv4:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip pim bsr-border
```

IPv6:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 pim bsr border
```

These commands prevent BSMs from being sent or accepted on the interface.

They do not directly block other PIM messages, Candidate-RP-Advertisements, or multicast data packets.

BSR borders are useful for keeping RP discovery information separate between PIM domains.

## IOS XE Verification

### IPv4

Verify the elected BSR:

```text
R1# show ip pim bsr-router
```

Verify the learned RP mappings:

```text
R1# show ip pim rp mapping
```

For a particular multicast group:

```text
R1# show ip pim rp-hash 239.1.1.1
```

The commands help answer three different questions:

| Command | Question |
|---|---|
| `show ip pim bsr-router` | Which router is the elected BSR? |
| `show ip pim rp mapping` | Which RPs and group ranges have been learned? |
| `show ip pim rp-hash` | Which RP is selected for this particular group? |

### IPv6

Verify BSR election and timers:

```text
R1# show ipv6 pim bsr election
```

On the elected BSR, verify Candidate RP advertisements received:

```text
R1# show ipv6 pim bsr rp-cache
```

Verify the learned group-to-RP mappings:

```text
R1# show ipv6 pim group-map
```

For a particular multicast group:

```text
R1# show ipv6 pim group-map FF15::101
```

The distinction between the two databases is important:

- **BSR RP cache:** Candidate RP information learned by the elected BSR.
- **Group-to-RP mapping database:** Information available to a PIM router after RP-Set distribution and selection.

### Troubleshooting BSM Propagation

If downstream routers do not learn the expected RP-Set:

1. Verify that a BSR has been elected.
2. Verify that the BSR has learned the Candidate RPs.
3. Verify PIM neighbors between the BSR and downstream routers.
4. Verify the RPF path toward the BSR.
5. Check for BSR border interfaces.
6. Check whether BSMs are reaching downstream interfaces.

Useful commands include:

```text
R1# show ip pim neighbor
R1# show ip rpf 1.1.1.1
R1# show ip pim bsr-router
R1# show ip pim rp mapping
```

For IPv6:

```text
R1# show ipv6 pim neighbor
R1# show ipv6 rpf 2001:DB8:1::1
R1# show ipv6 pim bsr election
R1# show ipv6 pim group-map
```

If necessary, capture PIM packets using IP protocol 103 and inspect Bootstrap Messages (Type 4).

## Key Points

- **Bootstrap Messages (PIM Type 4)** distribute the elected BSR's identity and RP-Set throughout the PIM domain.
- BSMs are normally sent to `224.0.0.13` for IPv4 or `FF02::D` for IPv6.
- They use **TTL/Hop Limit 1** and are forwarded hop by hop by PIM routers.
- The **outer IP source address changes at every hop**, but the BSR Address field continues identifying the elected BSR.
- The RP-Set contains **candidate RP addresses, group ranges, priorities, and Holdtimes**, not just the selected RP.
- Each receiving router independently selects its RP from the advertised mappings.
- Normal BSMs must pass an **RPF neighbor check toward the BSR address**.
- The default Bootstrap Period is **60 seconds** and the Minimum Bootstrap Interval is **10 seconds**.
- RP mappings have individual expiry timers refreshed by BSMs.
- An **RP Holdtime of 0** immediately withdraws a mapping.
- **Semantic fragmentation** allows large RP-Sets to be distributed across multiple BSM fragments.
- **RP Count** identifies the total number of RPs for a group range, while **Frag RP Cnt** identifies how many are included in the current fragment.
- The **No-Forward (N) bit** prevents a BSM from being forwarded and is useful for synchronizing newly discovered PIM neighbors.
- IPv6 supports **scoped BSMs**, allowing different scope zones to maintain separate RP-Sets.
- `ipv6 pim bsr announced rp` allows an IPv6 BSR to advertise an RP mapping without receiving a Candidate-RP-Advertisement.

## References

- [RFC 5059, Section 3.3: Creating the RP-Set at the BSR](https://www.rfc-editor.org/rfc/rfc5059.html#section-3.3)
- [RFC 5059, Section 3.4: Forwarding Bootstrap Messages](https://www.rfc-editor.org/rfc/rfc5059.html#section-3.4)
- [RFC 5059, Section 3.5: Bootstrap Messages to New and Rebooting Routers](https://www.rfc-editor.org/rfc/rfc5059.html#section-3.5)
- [RFC 5059, Section 4.1: Bootstrap Message Format](https://www.rfc-editor.org/rfc/rfc5059.html#section-4.1)
- [Cisco IOS XE: IPv6 Multicast Bootstrap Router](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-multicast/ip-multicast/m_ip6-mcast-bsr.html)