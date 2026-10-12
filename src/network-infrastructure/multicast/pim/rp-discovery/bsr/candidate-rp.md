# Candidate RP and RP Advertisements

In the PIM Bootstrap Router (BSR) mechanism, a **Candidate Rendezvous Point (C-RP)** is a router configured to advertise its willingness to serve as an RP for particular multicast groups.

C-RPs send **Candidate-RP-Advertisement (C-RP-Adv)** messages to the elected BSR.

The BSR collects this information and distributes an **RP-Set** throughout the PIM domain using Bootstrap Messages (BSMs).

Each router then independently determines which RP to use for a given multicast group.

**Candidate RPs do not advertise directly to every PIM router. They advertise to the elected BSR.**

## Candidate RP Operation

Consider the following topology:

```text
                  R1 (BSR)
                    1.1.1.1
                       |
                       R2
                      /  \
                     /    \
                    R3    R4
                  (C-RP) (C-RP)
                 3.3.3.3 4.4.4.4
                      \  /
                       R5
```

Assume:

- R1 is the elected BSR.
- R3 and R4 are Candidate RPs for `239.0.0.0/8`.
- All routers have PIM enabled on the relevant interfaces.
- The Candidate RP addresses are reachable through unicast routing.

### Step 1: Candidate RPs Learn the Elected BSR

The elected BSR advertises itself through Bootstrap Messages.

For example:

```text
BSR Address: 1.1.1.1
BSR Priority: 10
```

R3 and R4 receive the BSMs and learn that R1 is the elected BSR.

The Candidate RPs now know where to send their advertisements.

### Step 2: Candidate RPs Send Advertisements

R3 and R4 each send a **unicast C-RP-Adv message** to R1.

```text
R3 (C-RP) ----------------\
                           \
                            > R1 (BSR)
                           /
R4 (C-RP) ----------------/
```

For example:

```text
R3 → R1:

PIM Candidate-RP-Advertisement
  RP Address: 3.3.3.3
  Group Range: 239.0.0.0/8
  RP Priority: 10
  Holdtime: 150 seconds
```

```text
R4 → R1:

PIM Candidate-RP-Advertisement
  RP Address: 4.4.4.4
  Group Range: 239.0.0.0/8
  RP Priority: 20
  Holdtime: 150 seconds
```

These messages inform the BSR which routers are available to act as RPs.

### Step 3: BSR Builds the Candidate RP Database

The BSR stores the received candidate information.

For example:

| Group Range | Candidate RP | Priority | Holdtime |
|---|---|---|---|
| `239.0.0.0/8` | `3.3.3.3` | 10 | 150 seconds |
| `239.0.0.0/8` | `4.4.4.4` | 20 | 150 seconds |

The BSR maintains an expiry timer for each group-to-C-RP mapping.

Advertisements received before the timer expires refresh the corresponding entry.

The BSR constructs the RP-Set from its candidate RP database and distributes that information through BSMs.

### Step 4: Routers Select the RP

When R5 receives the RP-Set, it determines which RP should serve `239.1.1.1`.

Both R3 and R4 advertise a matching group range.

However, R3 has the lower RP priority:

```text
R3: Priority 10
R4: Priority 20
```

Therefore, R5 selects:

```text
Group: 239.1.1.1
RP:    3.3.3.3
```

**Lower numeric RP priorities are preferred.**

The BSR distributes candidate information, but each receiving router performs the final group-to-RP selection.

The BSR can apply policies that limit which candidates are included in its advertised RP-Set.

> Reminder: In the BSR election, a **higher** priority is preferred. In the RP selection process, a **lower** priority is preferred.
> I wonder who was responsible for that decision.

## Candidate-RP-Advertisement Messages

C-RP-Adv messages have the following characteristics:

| Field | Value |
|---|---|
| PIM Version | 2 |
| PIM Message Type | **8 (Candidate-RP-Advertisement)** |
| Destination | Elected BSR address |
| Transport | **Unicast** |
| IP Protocol / Next Header | 103 |
| Advertisement Interval | 60 seconds by default |
| Holdtime | Typically 150 seconds by default |

Unlike Bootstrap Messages, which are multicast and forwarded hop by hop, **C-RP-Adv messages are unicast directly to the elected BSR** using ordinary unicast routing.

Therefore, intermediate routers do not forward them using the special BSM flooding mechanism.

### Message Format

RFC 5059 defines the following C-RP-Adv message format:

```text
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |  Ver  | Type=8|   Reserved    |           Checksum            |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | Prefix Count  |   Priority    |           Holdtime            |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |              RP Address (Encoded-Unicast format)              |
 |                              ...                              |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |             Group Address 1 (Encoded-Group format)            |
 |                              ...                              |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |             Additional Encoded-Group Addresses                |
 |                              ...                              |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The important fields are:

| Field | Description |
|---|---|
| **Prefix Count** | Number of multicast group ranges advertised |
| **Priority** | Candidate RP priority; lower values are preferred |
| **Holdtime** | How long the BSR considers the advertisement valid |
| **RP Address** | Address the router is advertising as its RP address |
| **Group Addresses** | Multicast group prefixes for which the router is a candidate RP |

A single C-RP-Adv can advertise multiple multicast group ranges.

RFC 5059 requires C-RP-Adv messages to contain at least one group prefix.

> **Important:** The hash mask length is not carried in C-RP-Adv messages. It is a property advertised by the elected BSR in Bootstrap Messages.

## RP Priority

Each Candidate RP advertises a priority used during RP selection.

The priority range is **0 to 255** in the PIM C-RP advertisement format.

- **0:** Highest priority.
- **255:** Lowest priority.

For example:

| Candidate RP | Priority | Preference |
|---|---|---|
| R3 | 0 | Highest |
| R4 | 10 | Medium |
| R5 | 192 | Lower |

When multiple candidates advertise the same matching group range, the lowest priority value is preferred.

If the priorities are equal, routers use the **BSR hashing algorithm** to select an RP.

### Candidate RP Priority vs. Candidate BSR Priority

The direction of the priority comparison is different.

| Election or Selection | Preferred Priority |
|---|---|
| **BSR election** | Highest numeric value |
| **RP selection** | Lowest numeric value |

For example:

```text
BSR Priority 20 > BSR Priority 10

RP Priority 10 > RP Priority 20
```

In the second example, `10` represents the higher RP preference even though its numeric value is lower.

### IOS XE Default Priorities

An important IPv4/IPv6 difference exists in Cisco IOS XE:

| | IPv4 | IPv6 |
|---|---|---|
| Default Candidate RP priority | **0** | **192** |
| Highest possible preference | 0 | 0 |
| Preferred direction | Lower value | Lower value |

RFC 5059 recommends a default Candidate RP priority of **192**.

Cisco's IPv4 implementation uses 0 by default, whereas its documented IPv6 implementation uses 192.

Therefore, explicitly configured and default Candidate RPs may have different priorities, even when the configurations otherwise appear similar.

## Candidate RP Advertisement Timers

RFC 5059 defines the following default values:

| Parameter | Default |
|---|---|
| C-RP Advertisement Period | **60 seconds** |
| C-RP Advertisement Holdtime | **150 seconds** |
| Triggered Advertisement Backoff | **0 to 3 seconds** |

### Advertisement Interval

Candidate RPs periodically refresh their advertisements to the elected BSR.

By default:

```text
C-RP → BSR: Advertisement

       60 seconds

C-RP → BSR: Advertisement

       60 seconds

C-RP → BSR: Advertisement
```

These advertisements maintain the BSR's knowledge of the available RPs.

### Advertisement Holdtime

The **Holdtime** tells the BSR how long a particular C-RP advertisement remains valid without being refreshed.

RFC 5059 recommends:

```text
Holdtime = 2.5 × C-RP Advertisement Period
```

With the defaults:

```text
Holdtime = 2.5 × 60
         = 150 seconds
```

Each new advertisement refreshes the BSR's expiry timer.

If the timer expires, the BSR removes the affected candidate mapping from its database.

The updated RP-Set can then be distributed to the remaining PIM routers.

**The C-RP Advertisement Holdtime is separate from the BSR Bootstrap Timeout.**

The former tracks candidate RP availability at the BSR. The latter tracks the elected BSR itself.

### Advertisements After a BSR Election

When a new BSR is elected, Candidate RPs do not necessarily wait for their next 60-second periodic advertisement.

RFC 5059 requires a C-RP to send **one to three triggered advertisements** after learning about the new BSR.

A randomized delay of **0 to 3 seconds** is used before each transmission.

This allows the new BSR to learn the active candidate RPs relatively quickly.

### Candidate RP Withdrawal

RFC 5059 also defines a mechanism for withdrawing a Candidate RP.

When a Candidate RP is being shut down, it should send a final C-RP-Adv with:

```text
Holdtime: 0
```

The BSR immediately removes the advertised candidate mapping and can distribute an updated RP-Set.

This avoids waiting for the normal Holdtime to expire.

> **Note:** Sending a zero-Holdtime advertisement is the protocol-defined withdrawal behavior. Whether it is sent during a particular IOS XE configuration change or interface failure depends on the implementation and failure circumstances.

## IPv4 IOS XE Configuration

### Basic Candidate RP

On R3:

```text
R3(config)# ip multicast-routing

R3(config)# interface Loopback0
R3(config-if)# ip address 3.3.3.3 255.255.255.255
R3(config-if)# ip pim sparse-mode

R3(config)# ip pim rp-candidate Loopback0
```

This advertises the address of Loopback0 as a Candidate RP.

Without a group list, the Candidate RP advertises itself for the applicable ASM group range.

The Candidate RP must learn an elected BSR and have unicast connectivity to it.

### Restrict Advertised Group Ranges

For example, advertise R3 as a Candidate RP only for:

```text
239.1.0.0/16
```

Configure a standard ACL:

```text
R3(config)# ip access-list standard RP-GROUPS
R3(config-std-nacl)# permit 239.1.0.0 0.0.255.255
```

Apply it:

```text
R3(config)# ip pim rp-candidate Loopback0 group-list RP-GROUPS
```

The ACL determines which multicast group ranges are included in the C-RP advertisements.

It does not filter multicast data packets.

A Candidate RP can advertise multiple group ranges by configuring multiple permit statements in the ACL.

### Configure Priority and Advertisement Interval

On IOS XE platforms that support these keywords:

```text
R3(config)# ip pim rp-candidate Loopback0 group-list RP-GROUPS interval 30 priority 10
```

This configures:

- **`interval 30`:** Send periodic C-RP advertisements every 30 seconds.
- **`priority 10`:** Advertise RP priority 10 instead of the IPv4 default of 0.

A shorter advertisement interval can reduce the time required to detect an unavailable Candidate RP, at the expense of additional control-plane traffic.

## IPv6 IOS XE Configuration

IPv6 uses the following command:

```text
R3(config)# ipv6 pim bsr candidate rp 2001:DB8:3::3
```

The address must belong to the local router and be reachable throughout the relevant PIM domain.

### Basic Candidate RP

For example:

```text
R3(config)# ipv6 unicast-routing
R3(config)# ipv6 multicast-routing

R3(config)# interface Loopback0
R3(config-if)# ipv6 address 2001:DB8:3::3/128

R3(config)# ipv6 pim bsr candidate rp 2001:DB8:3::3
```

By default, the Candidate RP advertises itself for applicable IPv6 non-SSM multicast ranges.

### IPv6 Group List

To advertise R3 as a Candidate RP for `FF15::/16`:

```text
R3(config)# ipv6 access-list RP-GROUPS-V6
R3(config-ipv6-acl)# permit ipv6 any FF15::/16

R3(config)# ipv6 pim bsr candidate rp 2001:DB8:3::3 group-list RP-GROUPS-V6
```

The IPv6 ACL's **destination field** identifies the multicast group range.

Group ranges that overlap the reserved IPv6 SSM range are not advertised as ASM RP mappings.

### IPv6 Priority and Advertisement Interval

Configure a Candidate RP priority and advertisement interval:

```text
R3(config)# ipv6 pim bsr candidate rp 2001:DB8:3::3 group-list RP-GROUPS-V6 priority 10 interval 30
```

This advertises:

- RP address `2001:DB8:3::3`.
- Group range `FF15::/16`.
- Priority 10.
- Advertisement interval 30 seconds.

### IPv6 Scope

IOS XE also supports the `scope` keyword on platforms implementing scoped BSR.

For example:

```text
R3(config)# ipv6 pim bsr candidate rp 2001:DB8:3::3 group-list RP-GROUPS-V6 scope 5
```

This limits the C-RP advertisement to the BSR for scope 5, which corresponds to IPv6 site-local multicast scope.

The `scope` keyword restricts the Candidate RP to advertising itself to the BSR for the specified IPv6 multicast scope.

When combined with `group-list`, only group prefixes belonging to that scope are advertised.

For example, `scope 5` allows the router to advertise its RP candidacy for site-local multicast groups (`FFx5::/16`), but not global-scope groups (`FFxE::/16`).

## Candidate RP for Bidirectional PIM

BSR can also advertise RPs for Bidirectional PIM groups.

C-RP advertisements and Bootstrap Messages distinguish ordinary PIM-SM mappings from Bidirectional PIM mappings.

For IPv6, IOS XE supports the `bidir` keyword:

```text
R3(config)# ipv6 pim bsr candidate rp 2001:DB8:3::3 group-list RP-GROUPS-V6 bidir
```

This advertises the RP for Bidirectional PIM rather than traditional PIM-SM ASM.

The BSR must distribute a consistent multicast mode for each group range. Routers cannot safely use ordinary PIM-SM and Bidirectional PIM for the same group with conflicting RP information.

## IOS XE Verification

### IPv4 Candidate RP

On the Candidate RP, verify the configuration:

```text
R3# show running-config | include ip pim rp-candidate
```

Verify that an elected BSR is known:

```text
R3# show ip pim bsr-router
```

On routers throughout the domain, verify the learned mappings:

```text
R5# show ip pim rp mapping
```

Check that:

- The advertised RP address is correct.
- The expected multicast group range appears.
- The mapping is learned through BSR.
- The expected priority is advertised where displayed.

For a specific group:

```text
R5# show ip pim rp-hash 239.1.1.1
```

### IPv6 Candidate RP

On the Candidate RP:

```text
R3# show ipv6 pim bsr candidate-rp
```

Example output:

```text
PIMv2 C-RP information

  Candidate RP: 2001:DB8:3::3
    All Learnt Scoped Zones, Priority 192, Holdtime 150
    Advertisement interval 60 seconds
    Next advertisement in 00:00:33
```

This verifies the local Candidate RP configuration and the advertisement timer.

### IPv6 BSR RP Cache

On the **elected BSR**:

```text
R1# show ipv6 pim bsr rp-cache
```

Illustrative output:

```text
PIMv2 BSR C-RP Cache

BSR Candidate RP Cache

Group(s) FF15::/16, RP count 2

  RP 2001:DB8:3::3
    Priority 10, Holdtime 150
    Uptime: 00:12:36, expires: 00:01:55

  RP 2001:DB8:4::4
    Priority 20, Holdtime 150
    Uptime: 00:12:36, expires: 00:01:55
```

The most important fields are:

| Field | Meaning |
|---|---|
| Group(s) | Advertised multicast group range |
| RP count | Number of candidate RPs for the range |
| Priority | Advertised Candidate RP priority |
| Holdtime | Holdtime carried in the C-RP advertisement |
| Expires | Remaining validity of the candidate mapping |

The `expires` timer is refreshed when another valid C-RP advertisement is received.

On other IPv6 PIM routers:

```text
R5# show ipv6 pim group-map FF15::101
```

This verifies the selected RP after the BSR distributes the RP-Set.

## Common Troubleshooting Problems

| Problem | Possible Cause |
|---|---|
| Candidate RP sends no advertisements | No elected BSR learned, invalid C-RP configuration, or PIM not operational |
| BSR does not learn Candidate RP | Unicast routing problem, filtering, or C-RP address not reachable |
| Candidate RP advertises unexpected groups | Incorrect or missing group list |
| Candidate RP expires from the BSR cache | Periodic advertisements are not reaching the BSR |
| Unexpected RP selected | RP priorities, overlapping group ranges, or RP hashing |
| IPv6 Candidate RP not advertised for a group | Incorrect IPv6 ACL, scope mismatch, or group in an SSM range |
| BSR knows the RP but other routers do not | BSM distribution or RPF problem |

A useful troubleshooting distinction is:

- **C-RP to BSR:** Unicast Candidate-RP-Advertisements.
- **BSR to PIM domain:** Bootstrap Messages containing RP-Set information.

Therefore, a router can successfully learn the elected BSR but still fail to advertise itself as a Candidate RP because of a unicast connectivity problem.

Likewise, the elected BSR can learn a Candidate RP successfully while other routers fail to learn the mapping because of a BSM forwarding problem.

## Key Points

- **Candidate RPs advertise themselves to the elected BSR**, not directly to all PIM routers.
- C-RP advertisements use **PIM Type 8** and are sent as unicast packets using IP protocol 103.
- Each advertisement identifies the **RP address, group ranges, priority, and Holdtime**.
- **Lower RP priority values are preferred**, opposite to the BSR election priority comparison.
- The default C-RP advertisement interval is **60 seconds**.
- The default advertisement Holdtime is typically **150 seconds**, calculated as 2.5 times the advertisement interval.
- A **Holdtime of 0** is used to withdraw a Candidate RP advertisement.
- Candidate RPs send triggered advertisements after learning a newly elected BSR.
- The IPv4 IOS XE default C-RP priority is **0**, while the documented IPv6 default is **192**.
- IPv4 uses `ip pim rp-candidate`; IPv6 uses `ipv6 pim bsr candidate rp`.
- Group lists restrict which multicast group ranges are advertised.
- IOS XE support for optional Candidate RP keywords varies by platform.
- Verify local IPv6 C-RP state using `show ipv6 pim bsr candidate-rp`, and received advertisements at the BSR using `show ipv6 pim bsr rp-cache`.

## References

- [RFC 5059, Section 3.2: Sending Candidate-RP-Advertisement Messages](https://www.rfc-editor.org/rfc/rfc5059.html#section-3.2)
- [RFC 5059, Section 3.3: Creating the RP-Set at the BSR](https://www.rfc-editor.org/rfc/rfc5059.html#section-3.3)
- [RFC 5059, Section 4.2: Candidate-RP-Advertisement Message Format](https://www.rfc-editor.org/rfc/rfc5059.html#section-4.2)
- [Cisco IOS XE: Configuring Basic IP Multicast](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-multicast/ip-multicast/m_imc_basic_cfg-0.html)
- [Cisco IOS XE Dublin 17.12: IP Multicast Routing Commands](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9400/software/release/17-12/command_reference/b_1712_9400_cr/ip_multicast_routing.html)
- [Cisco IOS XE: IPv6 PIM BSR Candidate RP Command](https://www.cisco.com/c/en/us/td/docs/ios/ipv6/command/reference/ipv6_book/ipv6_08.html)
- [Cisco IOS XE: IPv6 PIM BSR Verification Commands](https://www.cisco.com/c/en/us/td/docs/ios/ipv6/command/reference/ipv6_book/ipv6_15.html)