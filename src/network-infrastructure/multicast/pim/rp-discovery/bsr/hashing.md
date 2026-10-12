# BSR Hashing

The **PIM BSR hashing algorithm** allows routers to deterministically select a Rendezvous Point (RP) when multiple Candidate RPs are available for the same multicast group.

The elected BSR distributes an **RP-Set** containing candidate RP addresses, group ranges, and priorities. Each PIM router independently applies the same selection algorithm to determine the RP for a particular group.

**The BSR does not calculate or advertise a winning RP for each group.**

When multiple eligible RPs have equal priorities, hashing can distribute different multicast groups among them without requiring per-group RP configuration.

The RP selection algorithm is defined in **RFC 7761, Sections 4.7.1 and 4.7.2**.

## RP Selection Order

PIM-SM selects an RP using the following criteria, in order:

1. **Longest matching multicast group prefix wins.**
2. **Lowest numeric RP priority wins.**
3. If multiple RPs remain, the **highest hash value wins**.
4. If hash values are equal, the **highest RP IP address wins**.

| Criterion | Preferred Value |
|---|---|
| Group prefix length | Longest |
| RP priority | Lowest |
| RP hash value | Highest |
| RP address (final tie-breaker) | Highest |

The hashing algorithm is only needed when multiple Candidate RPs remain after group prefix and priority comparisons.

### Example 1: Longest Prefix Match

Consider these BSR-learned RP mappings:

| Candidate RP | Group Range | Priority |
|---|---|---|
| `3.3.3.3` | `239.0.0.0/8` | 0 |
| `4.4.4.4` | `239.1.0.0/16` | 100 |

For multicast group `239.1.1.1`, both ranges match.

However, `239.1.0.0/16` is more specific than `239.0.0.0/8`.

Therefore:

```text
Group:       239.1.1.1
Selected RP: 4.4.4.4
Reason:      Longest prefix match
```

**The longer matching group prefix wins even though its RP has a numerically higher (less preferred) priority.**

No hashing is necessary.

### Example 2: RP Priority

Assume the following mappings:

| Candidate RP | Group Range | Priority |
|---|---|---|
| `3.3.3.3` | `239.0.0.0/8` | 10 |
| `4.4.4.4` | `239.0.0.0/8` | 20 |

Both RPs match the same group range.

However, `3.3.3.3` has the lower RP priority.

```text
Group:       239.1.1.1
Selected RP: 3.3.3.3
Reason:      Lower RP priority
```

Again, no hashing is necessary.

### Example 3: Equal Priorities

Now suppose:

| Candidate RP | Group Range | Priority |
|---|---|---|
| `3.3.3.3` | `239.0.0.0/8` | 10 |
| `4.4.4.4` | `239.0.0.0/8` | 10 |

Both Candidate RPs have the same group prefix length and priority.

The router calculates a hash value for each Candidate RP using the multicast group address and the BSR's hash mask.

**The RP producing the highest hash value is selected.**

## PIM RP Hash Function

RFC 7761 defines the following hash function:

```text
Value(G, M, C) =

(1103515245 × ((1103515245 × (G & M) + 12345) XOR C)
 + 12345) mod 2^31
```

The inputs are:

| Parameter | Meaning |
|---|---|
| **G** | Multicast group address |
| **M** | Hash mask advertised by the BSR |
| **C** | Candidate RP address |
| **G & M** | Group address after applying the hash mask |

The formula produces a **31-bit unsigned hash value**, ranging from 0 to 2,147,483,647.

Each eligible Candidate RP produces its own hash value because its RP address is included in the calculation.

For example:

```text
                 Group: 239.1.1.1
                        |
                 Apply hash mask
                        |
              +---------+---------+
              |                   |
              v                   v
        RP 3.3.3.3           RP 4.4.4.4
              |                   |
         Calculate hash      Calculate hash
              |                   |
              +---------+---------+
                        |
                 Compare values
                        |
                 Highest wins
```

The hash function is deterministic.

Assuming they have the same RP-Set and hash mask, all PIM routers calculate identical values and select the same RP for a given group.

## Hash Mask Length

The **hash mask length** determines how many leading bits of the multicast group address are included in the hash calculation.

It is configured on the **Candidate BSR** and advertised by the elected BSR in Bootstrap Messages.

For example:

```text
R1(config)# ip pim bsr-candidate Loopback0 30 10
```

Here:

- `Loopback0` provides the BSR address.
- `30` is the hash mask length.
- `10` is the BSR election priority.

The hash mask length is **not the same as the multicast group prefix length advertised by a Candidate RP**.

The Candidate RP's group prefix determines *which groups it can serve*. The BSR's hash mask determines *which group address bits influence the hash calculation*.

### IPv4 Hash Mask Length

The valid IPv4 hash mask length is 0 to 32 bits.

| Hash Mask Length | IPv4 Mask | Effect |
|---|---|---|
| 0 | `0.0.0.0` | Group address does not influence the hash |
| 8 | `255.0.0.0` | First 8 group bits influence the hash |
| 24 | `255.255.255.0` | Groups within the same /24 have the same masked value |
| 30 | `255.255.255.252` | Groups within the same /30 have the same masked value |
| 32 | `255.255.255.255` | All 32 group bits influence the hash |

RFC 5059 recommends **30 bits** for IPv4.

### Example: 30-Bit Hash Mask

With a 30-bit hash mask:

```text
Hash Mask: 255.255.255.252
```

Consider the following groups:

```text
239.1.1.0
239.1.1.1
239.1.1.2
239.1.1.3
```

All four produce the same masked group value:

```text
239.1.1.0
```

Therefore, they generate identical hash values for each Candidate RP.

Assuming the eligible Candidate RP set remains unchanged, **all four groups select the same RP**.

The next block of four addresses uses a different masked value:

```text
239.1.1.4
239.1.1.5
239.1.1.6
239.1.1.7
```

These produce:

```text
239.1.1.4
```

They may select a different RP.

> **Important:** The hash mask groups multicast addresses into blocks that share the same hash input. It does not assign RPs in a round-robin sequence or guarantee that adjacent blocks select different RPs.

## Calculated IPv4 Hashing Example

Consider two Candidate RPs:

| Candidate RP | Group Range | Priority |
|---|---|---|
| `3.3.3.3` | `239.0.0.0/8` | 10 |
| `4.4.4.4` | `239.0.0.0/8` | 10 |

The BSR advertises a hash mask length of **30 bits**.

For group `239.1.1.1`:

```text
G: 239.1.1.1
M: 255.255.255.252

G & M = 239.1.1.0
```

Applying the RFC 7761 hash function gives:

| Candidate RP | Calculated Hash |
|---|---:|
| `3.3.3.3` | **1,840,069,355** |
| `4.4.4.4` | 192,087,346 |

Therefore:

```text
Group:       239.1.1.1
Selected RP: 3.3.3.3
```

The same RP is selected for `239.1.1.2` and `239.1.1.3`, because all three groups have the same masked value.

Now consider `239.1.1.13`:

```text
G: 239.1.1.13
M: 255.255.255.252

G & M = 239.1.1.12
```

The resulting values are:

| Candidate RP | Calculated Hash |
|---|---:|
| `3.3.3.3` | 883,924,439 |
| `4.4.4.4` | **1,542,141,622** |

Therefore:

```text
Group:       239.1.1.13
Selected RP: 4.4.4.4
```

The results demonstrate how equal-priority Candidate RPs can serve different multicast groups.

| Multicast Group | Selected RP |
|---|---|
| `239.1.1.1` | `3.3.3.3` |
| `239.1.1.2` | `3.3.3.3` |
| `239.1.1.3` | `3.3.3.3` |
| `239.1.1.13` | `4.4.4.4` |
| `239.1.1.14` | `4.4.4.4` |
| `239.1.1.15` | `4.4.4.4` |

**The hashing algorithm distributes groups, not individual multicast packets.**

All routers must select the same RP for each group, but different groups can select different RPs.

## Hash Mask Length 0

An important Cisco IOS XE detail is that the **default IPv4 BSR hash mask length is 0**.

With a hash mask length of 0:

```text
Hash Mask: 0.0.0.0
```

For every multicast group:

```text
G & M = 0.0.0.0
```

Therefore, the group address has no effect on the hash calculation.

Consider two Candidate RPs with equal priorities and identical group ranges:

```text
RP1: 3.3.3.3
RP2: 4.4.4.4
```

With a hash mask length of 0, their calculated hash values are:

| Candidate RP | Hash Value |
|---|---:|
| `3.3.3.3` | 450,145,259 |
| `4.4.4.4` | **1,642,267,698** |

Because the group address is excluded from the calculation, these values are identical for every multicast group.

**All groups with this same eligible Candidate RP set select `4.4.4.4`.**

This means that with equal-priority Candidate RPs advertising identical group ranges, the default IPv4 hash mask length of 0 does **not distribute different multicast groups among the RPs**.

It consistently selects one RP.

A hash mask length of 0 does not disable hashing. It simply removes the group address from the hash input.

> **Important:** This behavior applies to a given set of eligible Candidate RPs. Different group ranges or priorities can still cause other groups to select different RPs.

## Hash Mask Length and RP Distribution

The hash mask length affects the granularity of group-to-RP selection.

| Hash Mask | Result |
|---|---|
| **0** | One deterministic RP for each eligible Candidate RP set, regardless of group address |
| **Shorter mask** | Larger blocks of groups share the same hash input |
| **Longer mask** | Smaller blocks of groups share the same hash input |
| **Full-length mask** | Each distinct group address can produce different hash values |

A longer hash mask provides finer-grained opportunities to distribute groups across Candidate RPs.

However, **hashing does not guarantee equal multicast traffic distribution**.

For example, one multicast group might generate 1 Gbps of traffic, while another generates only 1 Mbps.

Even if two RPs are selected for an equal number of groups, their forwarding and control-plane loads may be very different.

Hashing provides deterministic group assignment, not traffic-aware load balancing.

## IPv6 BSR Hashing

IPv6 uses the same overall RP selection rules:

1. Longest matching group prefix.
2. Lowest RP priority.
3. Highest hash value.
4. Highest RP IPv6 address as the final tie-breaker.

However, IPv6 addresses are 128 bits long, while the PIM hash function operates on 32-bit values.

RFC 7761 therefore recommends deriving a **32-bit digest** from each IPv6 address before applying the hash function.

For ordinary IPv6 addresses, the recommended digest is the bitwise XOR of the four 32-bit portions of the address.

For example:

```text
IPv6 Address: 2001:0DB8:0003:0000:0000:0000:0000:0003

32-bit segments:

20010DB8
00030000
00000000
00000003
```

Calculate:

```text
20010DB8
XOR 00030000
XOR 00000000
XOR 00000003
------------
20020DBB
```

The resulting 32-bit digest is used as the Candidate RP address input to the PIM hash function.

The multicast group address is first masked using the IPv6 hash mask, and a corresponding 32-bit digest is derived from that masked address.

**All routers must use a consistent digest method to calculate the same RP.**

### IPv6 Hash Mask Length

The valid IPv6 hash mask length is 0 to 128 bits.

RFC 5059 recommends **126 bits** for IPv6.

Cisco IOS XE also uses **126 bits as the default IPv6 BSR hash mask length**.

For example:

```text
R1(config)# ipv6 pim bsr candidate bsr 2001:DB8:1::1 126 priority 10
```

With a 126-bit hash mask, groups sharing the first 126 bits use the same masked group address.

For example:

```text
FF15::100
FF15::101
FF15::102
FF15::103
```

All four share the same masked address:

```text
FF15::100
```

Therefore, they produce identical hash values for each Candidate RP and select the same RP.

### Calculated IPv6 Example

Assume the following Candidate RPs, both with priority 10:

```text
2001:DB8:3::3
2001:DB8:4::4
```

Both advertise themselves as RPs for the site-local multicast group range `FF15::/16`.

The BSR advertises a hash mask length of 126 bits.

Applying the recommended IPv6 digest and RFC 7761 hash function produces:

| Group | Selected RP |
|---|---|
| `FF15::101` | `2001:DB8:3::3` |
| `FF15::102` | `2001:DB8:3::3` |
| `FF15::103` | `2001:DB8:3::3` |
| `FF15::104` | `2001:DB8:4::4` |
| `FF15::105` | `2001:DB8:4::4` |

As with IPv4, multiple Candidate RPs can be assigned different multicast groups while routers make consistent, independent decisions.

IPv6 administrative scope also determines which RP mappings are applicable to a group. RP hashing operates on the eligible mappings within the relevant scope zone.

## Hash Value Tie-Breaker

The PIM hash function produces a 31-bit value.

Although uncommon, two Candidate RPs can produce identical hash values for the same group.

If this occurs, the router selects the Candidate RP with the **highest IP address**.

For example:

| Candidate RP | Hash Value |
|---|---:|
| `3.3.3.3` | 1,000,000,000 |
| `4.4.4.4` | 1,000,000,000 |

Both hashes are equal.

Therefore:

```text
Selected RP: 4.4.4.4
Reason:      Higher RP address
```

**The highest RP address is only the final tie-breaker.**

It does not automatically win whenever Candidate RP priorities are equal.

## IOS XE Configuration

There is no separate command required to enable BSR RP hashing.

Hashing is performed automatically when a router receives multiple eligible Candidate RP mappings.

The main configurable parameter is the **hash mask length**, which is configured on the Candidate BSR.

### IPv4 Example

Configure R1 as the Candidate BSR with a hash mask length of 30:

```text
R1(config)# ip pim bsr-candidate Loopback0 30 10
```

Configure two Candidate RPs with equal priorities:

On R3:

```text
R3(config)# ip access-list standard RP-GROUPS
R3(config-std-nacl)# permit 239.0.0.0 0.255.255.255

R3(config)# ip pim rp-candidate Loopback0 group-list RP-GROUPS priority 10
```

On R4:

```text
R4(config)# ip access-list standard RP-GROUPS
R4(config-std-nacl)# permit 239.0.0.0 0.255.255.255

R4(config)# ip pim rp-candidate Loopback0 group-list RP-GROUPS priority 10
```

Assuming their respective Loopback0 addresses are `3.3.3.3` and `4.4.4.4`, both advertise themselves for the same group range with the same priority.

Each receiving PIM router uses the BSR's 30-bit hash mask to select the RP for each group.

> **Note:** The `priority` keyword is supported on router-oriented IOS XE platforms, but some Catalyst IOS XE images expose a more limited `ip pim rp-candidate` command. Verify the syntax on the target platform.

### IPv6 Example

Configure the Candidate BSR:

```text
R1(config)# ipv6 pim bsr candidate bsr 2001:DB8:1::1 126 priority 10
```

Configure R3 as a Candidate RP:

```text
R3(config)# ipv6 access-list RP-GROUPS-V6
R3(config-ipv6-acl)# permit ipv6 any FF15::/16

R3(config)# ipv6 pim bsr candidate rp 2001:DB8:3::3 group-list RP-GROUPS-V6 priority 10
```

Configure R4:

```text
R4(config)# ipv6 access-list RP-GROUPS-V6
R4(config-ipv6-acl)# permit ipv6 any FF15::/16

R4(config)# ipv6 pim bsr candidate rp 2001:DB8:4::4 group-list RP-GROUPS-V6 priority 10
```

The two Candidate RPs advertise identical group ranges and priorities.

Routers can then use the BSR hashing algorithm to select between them for eligible IPv6 multicast groups.

## IOS XE Verification

### Verify the IPv4 Hash Mask

```text
R1# show ip pim bsr-router
```

Example:

```text
R1# show ip pim bsr-router
PIMv2 Bootstrap information
This system is the Bootstrap Router (BSR)
  BSR address: 1.1.1.1
  BSR Priority: 10, Hash mask length: 30
```

Verify the **Hash mask length** field.

All routers participating in the same BSR domain should learn the same hash mask length from the elected BSR.

### Verify Candidate RP Mappings

```text
R1# show ip pim rp mapping
```

This displays the learned group-to-RP mappings and associated priorities.

Verify that both Candidate RPs are advertised for the expected group range.

### Verify the Selected RP and Hash Values

Use:

```text
R1# show ip pim rp-hash 239.1.1.1
```

This command displays the RP selected for the specified multicast group.

On supported IOS XE images, it also displays the PIMv2 hash values calculated for the Candidate RPs.

For example, compare the results for:

```text
R1# show ip pim rp-hash 239.1.1.1
R1# show ip pim rp-hash 239.1.1.2
R1# show ip pim rp-hash 239.1.1.13
```

With the Candidate RPs and mask used in the earlier calculated example:

- `239.1.1.1` and `239.1.1.2` should select the same RP.
- `239.1.1.13` should select the other RP.

This is an effective way to verify the effect of the BSR hash mask length.

### IPv6 Verification

Verify the elected BSR and its hash mask:

```text
R1# show ipv6 pim bsr election
```

Verify Candidate RP mappings:

```text
R1# show ipv6 pim group-map info-source bsr
```

For a particular group:

```text
R1# show ipv6 pim group-map FF15::101
```

Repeat the group lookup on multiple PIM routers.

Routers with the same RP-Set, scope, and hash inputs should agree on the selected RP.

> **Note:** `show ipv6 pim rp-hash` is available on some other Cisco operating systems, including NX-OS, but should not be assumed to be available on IOS XE. Use the supported IPv6 PIM group mapping commands for verification.

## RP Selection Changes

The selected RP can change when:

- A Candidate RP is added to or removed from the RP-Set.
- A Candidate RP priority changes.
- A multicast group range changes.
- The elected BSR advertises a different hash mask length.
- The BSR domain or applicable IPv6 scope changes.

For example, suppose:

```text
Group: 239.1.1.1

Candidate RPs:
  3.3.3.3
  4.4.4.4

Selected RP:
  3.3.3.3
```

If R3 stops advertising itself and its mapping is removed, the remaining Candidate RP is:

```text
4.4.4.4
```

The group-to-RP mapping changes accordingly.

Such changes can affect multicast forwarding state.

For example, a last-hop router may need to establish a new `(*,G)` tree toward the newly selected RP, while a first-hop router may need to send PIM Register messages to the new RP.

**BSR hashing does not provide RP state synchronization or seamless failover by itself.**

It provides a deterministic method for selecting an RP from the currently available mappings.

## Troubleshooting RP Hashing

| Problem | Possible Cause |
|---|---|
| All groups select one RP | IPv4 hash mask length 0, unequal RP priorities, or only one eligible RP |
| Different routers select different RPs | Inconsistent RP-Sets, transient BSM propagation, or incompatible hash processing |
| Expected RP never selected | Less specific group prefix, less preferred RP priority, or excluded from RP-Set |
| Selected RP changes unexpectedly | C-RP advertisement expiry, RP-Set changes, or BSR change |
| IPv6 group selects an unexpected RP | Scoped BSR mappings, group range mismatch, or RP priority differences |
| Changing the hash mask has no visible effect | Only one RP remains after prefix and priority comparisons, or tested groups still hash to the same RP |

When troubleshooting, verify these items in order:

1. The multicast group range matches the intended Candidate RPs.
2. The Candidate RPs have the expected priorities.
3. The elected BSR advertises the intended hash mask length.
4. All routers have learned the same RP-Set.
5. The hash calculation produces the expected selected RP.

## Key Points

- **BSR hashing selects an RP when multiple eligible Candidate RPs have equal group prefix lengths and priorities.**
- RP selection order is **longest group prefix, lowest RP priority, highest hash value, highest RP address**.
- The hashing algorithm is defined in **RFC 7761**.
- **Every PIM router performs the calculation independently** using its learned RP mappings.
- The elected BSR advertises the **hash mask length** in Bootstrap Messages.
- The hash mask length controls how many leading multicast group address bits influence the hash.
- Groups sharing the same masked address select the same RP when their eligible Candidate RP sets are identical.
- RFC 5059 recommends hash mask lengths of **30 bits for IPv4** and **126 bits for IPv6**.
- Cisco IOS XE defaults to **0 bits for IPv4** and **126 bits for IPv6**.
- With an IPv4 hash mask length of **0**, the group address does not influence the calculation, so identical eligible RP sets always select the same RP.
- A **higher hash value wins**; the highest RP address is only used when hash values are equal.
- IPv6 hashing requires deriving consistent **32-bit digests** from the 128-bit addresses.
- Hashing provides deterministic **per-group RP assignment**, not packet-level load balancing or guaranteed equal traffic distribution.
- `show ip pim rp-hash <group>` verifies IPv4 RP selection and hash values.
- BSR hashing does not synchronize multicast state between RPs or guarantee seamless RP failover.

## References

- [RFC 7761, Section 4.7.1: Group-to-RP Mapping](https://www.rfc-editor.org/rfc/rfc7761.html#section-4.7.1)
- [RFC 7761, Section 4.7.2: Hash Function](https://www.rfc-editor.org/rfc/rfc7761.html#section-4.7.2)
- [RFC 5059: Bootstrap Router Mechanism for PIM](https://www.rfc-editor.org/rfc/rfc5059.html)
- [Cisco IOS XE: Configuring Basic IP Multicast](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-multicast/ip-multicast/m_imc_basic_cfg-0.html)
- [Cisco IOS XE: IPv6 PIM Command Reference](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipv6/command/ipv6-cr-book/ipv6-i4.html)
- [Cisco IOS XE: IPv6 PIM Verification Commands](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipv6/command/ipv6-cr-book/ipv6-s4.html)