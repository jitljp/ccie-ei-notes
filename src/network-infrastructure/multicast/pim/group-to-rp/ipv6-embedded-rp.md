
# IPv6 Embedded RP

**IPv6 Embedded RP**, defined in [RFC 3956](https://www.rfc-editor.org/rfc/rfc3956), allows an IPv6 PIM Sparse Mode (PIM-SM) router to **derive the Rendezvous Point (RP) address directly from the multicast group address**.

Unlike Static RP or BSR, Embedded RP does not require routers to receive RP advertisements or configure explicit Group-to-RP mappings.

Instead, the RP address is encoded into specific fields of the IPv6 multicast address.

## Embedded RP Operation

Consider an IPv6 multicast network using PIM-SM.

Normally, routers learn the RP for a multicast group through:

- Static RP configuration
- BSR advertisements

With Embedded RP, the multicast group address itself contains the information needed to calculate the RP address.

The process is:

1. A router encounters an IPv6 multicast group that uses the Embedded RP address format.
2. The router examines the fields in the multicast group address.
3. It extracts the IPv6 prefix and RP Interface ID (RIID).
4. It reconstructs the RP's IPv6 unicast address.
5. It uses the derived RP for PIM-SM operations.

For example:

```text
Multicast group: FF7E:140:2001:DB8:BEEF:FEED:1234:1234:1234

Embedded RP:     2001:DB8:BEEF:FEED::1
```

Routers can independently determine the same RP address without exchanging RP mapping information.

The derived RP is used for normal PIM-SM operations:

| Operation | Behavior |
|---|---|
| Receiver joins | The Last-Hop Router sends `(*,G)` Joins toward the embedded RP |
| Source registration | The source-facing DR sends PIM Register messages to the embedded RP |
| Shared-tree construction | Routers use the RPF path toward the embedded RP |

**Embedded RP changes how the RP address is discovered, not how PIM-SM constructs multicast distribution trees.**

## IPv6 Embedded RP Address Format

Embedded RP uses a modified version of the unicast-prefix-based IPv6 multicast address format defined in RFC 3306.

RFC 3956 defines the following 128-bit format:

```text
|  8 bits | 4 bits | 4 bits | 4 bits | 4 bits | 8 bits |     64 bits    |  32 bits |
+---------+--------+--------+--------+--------+--------+----------------+----------+
|11111111 | Flags  | Scope  |  Rsvd  |  RIID  |  Plen  | Network Prefix | Group ID |
+---------+--------+--------+--------+--------+--------+----------------+----------+
```

The fields are:

| Field | Length | Purpose |
|---|---|---|
| `11111111` | 8 bits | Identifies an IPv6 multicast address (`FF00::/8`) |
| Flags | 4 bits | Indicates the multicast address format |
| Scope | 4 bits | Identifies the multicast scope |
| Rsvd | 4 bits | Reserved, set to zero |
| RIID | 4 bits | Identifies the RP within the embedded unicast prefix |
| Plen | 8 bits | Number of unicast network-prefix bits used to construct the RP address |
| Network Prefix | 64 bits | Contains the unicast prefix used to derive the RP |
| Group ID | 32 bits | Identifies the multicast group |

### Flags

The four flag bits are:

```text
+---+---+---+---+
| 0 | R | P | T |
+---+---+---+---+
```

| Flag | Meaning |
|---|---|
| `0` | Reserved bit, set to zero in the original RFC 3956 format |
| `R` | RP address is embedded in the multicast address |
| `P` | Multicast address is based on a unicast prefix |
| `T` | Transient (dynamically assigned) multicast address |

For Embedded RP:

```text
R = 1
P = 1
T = 1
```

Therefore, the four-bit flag field is:

```text
0111 = 0x7
```

An Embedded RP address using the original RFC 3956 flag format consequently begins with:

```text
FF7x:
```

Here, `x` represents the multicast scope.

For example:

| Prefix | Scope |
|---|---|
| `FF72::/16` | Link-local |
| `FF75::/16` | Site-local |
| `FF78::/16` | Organization-local |
| `FF7E::/16` | Global |

Cisco IOS XE also recognizes Embedded RP addresses with the `FFFx::/16` format, where the highest-order flag bit is set.

**The important identifying bits are R=1, P=1, and T=1.**

### RP Interface ID (RIID)

The **RIID** field is four bits long.

It identifies the RP by specifying the value inserted into the **last four bits of the reconstructed RP address**.

Valid RIID values are:

```text
1 through F (hexadecimal)
```

An RIID of zero is not permitted for a valid Embedded RP address.

For example:

```text
Network prefix: 2001:DB8:BEEF:FEED::/64

RIID = 1
RP   = 2001:DB8:BEEF:FEED::1

RIID = 2
RP   = 2001:DB8:BEEF:FEED::2

RIID = A
RP   = 2001:DB8:BEEF:FEED::A
```

The RIID is **not a complete interface identifier**.

It provides only the final four bits of the RP address. All other bits that are not part of the extracted network prefix are set to zero.

### Prefix Length (Plen)

The Plen field specifies how many bits of the embedded Network Prefix are copied into the RP address.

It is an eight-bit field containing the prefix length as an integer.

For example:

| Prefix Length | Hexadecimal Plen |
|---|---|
| `/32` | `0x20` |
| `/48` | `0x30` |
| `/64` | `0x40` |

The prefix length must satisfy:

```text
1 <= Plen <= 64
```

A prefix length of zero is not permitted, and prefix lengths greater than 64 are invalid.

**The Plen field determines how much of the embedded network prefix is used to reconstruct the RP address.**

## Deriving the Embedded RP Address

To derive the RP address, a router performs two operations:

1. Copy the first **Plen bits** of the embedded Network Prefix into an otherwise zeroed 128-bit IPv6 address.
2. Replace the last four bits of that address with the **RIID** value.

### Example 1: /64 Prefix

Consider:

```text
Multicast group: FF7E:140:2001:DB8:BEEF:FEED:1234:1234
```

Break the multicast address into its fields:

| Field | Value |
|---|---|
| Flags | `7` |
| Scope | `E` (Global) |
| Reserved | `0` |
| RIID | `1` |
| Plen | `0x40` (64 bits) |
| Network Prefix | `2001:DB8:BEEF:FEED` |
| Group ID | `1234:1234` |

The router first extracts 64 bits of the network prefix:

```text
2001:DB8:BEEF:FEED::/64
```

It then sets the final four bits to the RIID value of `1`.

The resulting RP address is:

```text
2001:DB8:BEEF:FEED::1
```

Therefore:

```text
Group: FF7E:140:2001:DB8:BEEF:FEED::1234

RP:    2001:DB8:BEEF:FEED::1
```

### Example 2: /32 Prefix

Consider:

```text
Multicast group: FF7E:220:2001:DB8:1234:5678:1234:100
```

The relevant fields are:

| Field | Value |
|---|---|
| RIID | `2` |
| Plen | `0x20` (32 bits) |
| Network Prefix | `2001:DB8:1234:5678` |

Because Plen is 32, only the first 32 bits of the embedded network prefix are used:

```text
2001:DB8::/32
```

The remaining bits are set to zero, except for the last four bits, which receive the RIID value of `2`.

The resulting RP is:

```text
2001:DB8::2
```

Notice that `1234:5678` in the multicast address **does not become part of the RP address**.

Those bits lie beyond the specified 32-bit prefix length.

### Example 3: Multiple Groups Using the Same RP

Consider these multicast groups:

```text
FF7E:140:2001:DB8:BEEF:FEED:1234:100
FF7E:140:2001:DB8:BEEF:FEED:1234:200
FF7E:140:2001:DB8:BEEF:FEED:1234:300
```

They have different Group IDs, but identical values for:

- RIID
- Plen
- The extracted Network Prefix

Therefore, all three groups use:

```text
RP = 2001:DB8:BEEF:FEED::1
```

**Different multicast groups can share the same Embedded RP without requiring explicit Group-to-RP mappings.**

## Embedded RP and PIM-SM

Embedded RP does not introduce a different multicast forwarding protocol.

It uses standard PIM-SM procedures after the RP has been determined.

### Receiver Join Process

When a receiver joins an Embedded RP multicast group:

1. The receiver sends an MLD Report for the group.
2. The Last-Hop Router derives the RP address from the group.
3. The Last-Hop Router determines the RPF path toward the RP.
4. It sends a PIM `(*,G)` Join toward the RP.
5. Intermediate routers construct the shared tree using normal PIM-SM procedures.

### Source Registration

When a multicast source begins transmitting:

1. The source-facing DR identifies the Embedded RP multicast group.
2. It derives the RP address from the group.
3. It sends PIM Register messages to that RP.
4. The RP processes the Register messages using standard PIM-SM procedures.
5. The RP can join the source tree and deliver multicast traffic toward receivers.

SPT switchover, Register-Stop messages, RPF checks, and other PIM-SM functions continue to operate normally.

### RP Reachability

Embedded RP eliminates the need to distribute the RP mapping, but it does **not** eliminate the requirement for RP reachability.

For example:

```text
Group: FF7E:140:2001:DB8:BEEF:FEED:1234:1234

RP:    2001:DB8:BEEF:FEED::1
```

The router must have a valid RPF path toward:

```text
2001:DB8:BEEF:FEED::1
```

If the RP is unreachable, the router cannot successfully construct the shared tree toward it.

The RP must also be configured to perform the RP function for the relevant multicast group.

## IOS XE Configuration

On Cisco IOS XE, **IPv6 Embedded RP support is enabled by default**.

Routers automatically recognize supported Embedded RP group addresses and derive their RP addresses.

### Enable Embedded RP Support

The global configuration command is:

```text
R1(config)# ipv6 pim rp embedded
```

Because Embedded RP is enabled by default, this command is normally unnecessary.

For a VRF:

```text
R1(config)# ipv6 pim vrf BLUE rp embedded
```

### Disable Embedded RP Support

To disable automatic Embedded RP processing:

```text
R1(config)# no ipv6 pim rp embedded
```

For a VRF:

```text
R1(config)# no ipv6 pim vrf BLUE rp embedded
```

This disables Embedded RP processing without disabling IPv6 PIM itself.

For example, the following configuration disables Embedded RP while leaving IPv6 PIM enabled on Ethernet0/0:

```text
R1(config)# no ipv6 pim rp embedded
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 pim
```

The router can still use other supported RP mapping mechanisms.

### Configuring the RP Router

The RP router must own the IPv6 unicast address encoded in the multicast group.

For example:

```text
RP = 2001:DB8:BEEF:FEED::1
```

Configure the address on a loopback interface:

```text
RP(config)# interface Loopback0
RP(config-if)# ipv6 address 2001:DB8:BEEF:FEED::1/128
RP(config-if)# ipv6 pim
```

Ensure that the RP address is reachable through IPv6 unicast routing and that IPv6 PIM is configured on the relevant multicast-routing interfaces.

The Embedded RP mechanism does not require a static Group-to-RP mapping to be configured throughout the domain.

## IOS XE Verification

### Verify the Derived RP

Use:

```text
R1# show ipv6 pim group-map FF7E:140:2001:DB8:BEEF:FEED::1234
```

The mapping should identify:

```text
RP: 2001:DB8:BEEF:FEED::1
```

The information source should indicate that the mapping was derived from Embedded RP.

To display Embedded RP mappings:

```text
R1# show ipv6 pim group-map info-source embedded-rp
```

### Verify PIM State

Use:

```text
R1# show ipv6 pim topology
```

For the example group, the `(*,G)` state should use:

```text
RP = 2001:DB8:BEEF:FEED::1
```

The selected incoming interface and RPF neighbor depend on the unicast routing topology.

### Verify RP Reachability

Check the IPv6 route toward the derived RP:

```text
R1# show ipv6 route 2001:DB8:BEEF:FEED::1
```

Also verify that the necessary IPv6 PIM interfaces and neighbors are operational:

```text
R1# show ipv6 pim interface
R1# show ipv6 pim neighbor
```

These commands help distinguish RP address derivation problems from routing or PIM adjacency problems.

## Limitations and Considerations

### IPv6 Only

Embedded RP is defined for IPv6 multicast.

IPv4 does not provide the same RFC 3956 RP-encoding mechanism.

### PIM-SM ASM

Embedded RP is designed for ASM using PIM-SM.

SSM does not use an RP, so Embedded RP is unnecessary for SSM forwarding.

### RP Address Restrictions

The RP address must be constructible using:

- A unicast prefix of no more than 64 bits
- A four-bit nonzero RIID
- Zero bits in the other positions not covered by the prefix

For example, this RP address can be embedded:

```text
2001:DB8:BEEF:FEED::1
```

However, an arbitrary address such as:

```text
2001:DB8:BEEF:FEED:1234:5678:9ABC:DEF0
```

cannot be represented directly using the Embedded RP format.

### No Automatic RP Redundancy

Embedded RP encodes a particular RP address into each multicast group.

If that RP becomes unavailable, routers do not automatically select another RP using BSR hashing or an alternate mapping.

RP redundancy must be provided through a separate design, such as making the encoded RP address reachable through multiple RP routers.

### Security Considerations

Because the RP address is derived from the multicast group address, a sender or receiver can reference an Embedded RP group that identifies an unexpected RP.

Routers should validate derived RP addresses and apply appropriate multicast filtering and routing policies.

RFC 3956 specifically requires certain invalid RP address ranges to be rejected, including:

- `FE80::/10`
- `::/16`
- `FF00::/8`

## Key Points

- **IPv6 Embedded RP** is defined in RFC 3956.
- The RP address is derived directly from the IPv6 multicast group address.
- Embedded RP eliminates the need to distribute explicit Group-to-RP mappings for the applicable groups.
- The original Embedded RP multicast format uses **Flags = 0111 (0x7)**.
- Cisco IOS XE recognizes Embedded RP group ranges `FF7x::/16` and `FFFx::/16`.
- The **RIID** field contains the four-bit value inserted into the last four bits of the derived RP address.
- The **Plen** field identifies the number of unicast prefix bits used to construct the RP address.
- Valid Plen values are **1 through 64**.
- The RP is derived by copying Plen prefix bits into a zeroed IPv6 address and replacing the last four bits with RIID.
- Different multicast groups can encode the same RP address.
- Embedded RP uses normal PIM-SM procedures for receiver joins, source registration, and multicast forwarding.
- **Embedded RP support is enabled by default on IOS XE**.
- Use `no ipv6 pim rp embedded` to disable Embedded RP processing.
- Use `show ipv6 pim group-map` to verify RP mappings.
- Embedded RP does not eliminate RP reachability requirements or automatically provide RP redundancy.
- Embedded RP is not used for SSM.

## References

- [RFC 3956: Embedding the Rendezvous Point (RP) Address in an IPv6 Multicast Address](https://www.rfc-editor.org/rfc/rfc3956)
- [RFC 3306: Unicast-Prefix-based IPv6 Multicast Addresses](https://www.rfc-editor.org/rfc/rfc3306)
- [Cisco IOS XE: IPv6 Multicast PIM Sparse Mode](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipmulti_pim/configuration/imc-pim-xe-3s-book/ip6-mcast-pim-sm.html)
