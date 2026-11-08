# CoS, IP Precedence, and DSCP Mapping

**Class of Service (CoS)**, **IP Precedence**, and **Differentiated Services Code Point (DSCP)** are three QoS marking mechanisms used to identify the intended forwarding treatment of network traffic.

Although all three represent QoS classifications, they differ in their location, size, and available values.

| Marking | Size | Values | Location |
|---|---|---|---|
| **CoS** | 3 bits | 0–7 | IEEE 802.1Q VLAN tag |
| **IP Precedence** | 3 bits | 0–7 | First three bits of the DSCP field |
| **DSCP** | 6 bits | 0–63 | IPv4/IPv6 DS field |

The relationship between these markings is important when traffic crosses between Layer 2 and Layer 3 networks.

For example:
- A switch might receive a frame marked CoS 5 and assign DSCP EF (46).
- A router might receive a packet marked EF (46) and classify it as IP Precedence 5.
- A switch might receive an IP packet marked AF31 (26) and assign CoS 3 when forwarding it as a tagged Ethernet frame.

**CoS, IP Precedence, and DSCP do not have to contain equivalent values.** Devices can translate between them according to configured QoS policies.

---

## Class of Service (CoS)

**Class of Service (CoS)** is a Layer 2 QoS marking carried in the **Priority Code Point (PCP)** field of an IEEE 802.1Q VLAN tag.

The 802.1Q tag contains a 16-bit Tag Control Information (TCI) field:

```text
       IEEE 802.1Q Tag Control Information

       3 bits    1 bit       12 bits
     +---------+-------+------------------+
     |   PCP   |  DEI  |      VLAN ID     |
     +---------+-------+------------------+
         |
         +---- CoS (0-7)
```

The three PCP bits represent the CoS value.

CoS supports eight possible values:

| CoS | Binary |
|---|---|
| 0 | `000` |
| 1 | `001` |
| 2 | `010` |
| 3 | `011` |
| 4 | `100` |
| 5 | `101` |
| 6 | `110` |
| 7 | `111` |

Common enterprise QoS conventions include:
- **CoS 0:** Best Effort.
- **CoS 3:** Call signaling.
- **CoS 4:** Video traffic.
- **CoS 5:** Voice bearer traffic.
- **CoS 6:** Network control traffic.

These are common assignments rather than mandatory mappings.

### CoS and Tagged Ethernet Frames

CoS is carried in the 802.1Q tag, not in the IP header.

Therefore:
- **Tagged frames** can carry a CoS value.
- **Untagged frames** do not carry an explicit CoS value.
- **Priority-tagged frames** use VLAN ID 0 and can carry CoS without identifying a VLAN.

When a router forwards an IP packet, the original Ethernet header is removed and a new Layer 2 header is created for the outgoing interface.

Consequently, **CoS is not carried end-to-end across routed IP networks**.

The outgoing interface can assign a new CoS value based on the packet's DSCP or another QoS classification.

---

## IP Precedence

**IP Precedence** is an older Layer 3 QoS marking mechanism that uses the three most significant bits of the DS field.

It supports values from 0 through 7.

```text
             IPv4 / IPv6 DS Field

       IP Precedence
        3 bits
     +-----------+
     |           |
     +---+---+---+---+---+---+---+---+
     | P | P | P |   DSCP   |  ECN  |
     +---+---+---+---+---+---+---+---+
     |         6 bits      | 2 bits |
     +---------------------+--------+
```

IP Precedence uses the first three bits of the six-bit DSCP value.

| IP Precedence | Binary | Traditional name |
|---|---|---|
| 0 | `000` | Routine |
| 1 | `001` | Priority |
| 2 | `010` | Immediate |
| 3 | `011` | Flash |
| 4 | `100` | Flash Override |
| 5 | `101` | Critical |
| 6 | `110` | Internetwork Control |
| 7 | `111` | Network Control |

Values 6 and 7 are generally reserved for network control traffic.

### IP Precedence and DSCP

Because IP Precedence uses the first three DSCP bits, it can be calculated using:

`IP Precedence = DSCP >> 3`

Equivalently:

`IP Precedence = floor(DSCP / 8)`

For example, EF uses DSCP 46:

```text
DSCP EF:        101110
                ^^^
                 |
          IP Precedence 5
```

`floor(46 / 8) = 5`

Therefore, a packet marked EF also has IP Precedence 5.

This relationship applies to all DSCP values, not just Class Selector codepoints.

### Multiple DSCP Values Share the Same IP Precedence

Each IP Precedence value corresponds to **eight possible DSCP values**.

For example, IP Precedence 5 matches every DSCP value from 40 through 47.

| DSCP | Binary | IP Precedence |
|---|---|---|
| CS5 (40) | `101000` | 5 |
| 41 | `101001` | 5 |
| 42 | `101010` | 5 |
| 43 | `101011` | 5 |
| VOICE-ADMIT (44) | `101100` | 5 |
| NQB (45) | `101101` | 5 |
| EF (46) | `101110` | 5 |
| 47 | `101111` | 5 |

Therefore, the following MQC commands are not equivalent:

```text
R1(config-cmap)# match ip precedence 5
```

Matches DSCP values 40 through 47.

```text
R1(config-cmap)# match dscp cs5
```

Matches only DSCP 40.

```text
R1(config-cmap)# match dscp ef
```

Matches only DSCP 46.

**IP Precedence classification is less granular than DSCP classification because it ignores the final three DSCP bits.**

---

## Class Selector (CS) and IP Precedence Mapping

The **Class Selector (CS)** codepoints provide compatibility between DSCP and the older IP Precedence model.

They use the following format:

```text
        IP Precedence
         3 bits
     +-----------+
     |           |
     +---+---+---+---+---+---+
     | x | x | x | 0 | 0 | 0 |
     +---+---+---+---+---+---+
             DSCP
```

The first three bits represent IP Precedence, and the final three bits are zero.

Therefore:

`DSCP = IP Precedence × 8`

| IP Precedence | Class Selector | DSCP |
|---|---|---|
| 0 | CS0 | 0 |
| 1 | CS1 | 8 |
| 2 | CS2 | 16 |
| 3 | CS3 | 24 |
| 4 | CS4 | 32 |
| 5 | CS5 | 40 |
| 6 | CS6 | 48 |
| 7 | CS7 | 56 |

For example, IP Precedence 3 maps to CS3:

`3 × 8 = 24`

**The reverse relationship is not one-to-one.** IP Precedence 3 could represent any DSCP value from 24 through 31.

When converting IP Precedence to DSCP using Class Selector encoding, the final three bits are set to zero.

---

## CoS-to-DSCP Mapping

CoS and DSCP are separate QoS fields:
- **CoS:** Three bits in an Ethernet VLAN tag.
- **DSCP:** Six bits in an IP header.

Because CoS has only eight possible values, it cannot represent all 64 DSCP values.

A common mapping convention aligns the three CoS bits with the three most significant DSCP bits.

The remaining three DSCP bits are set to zero.

```text
        CoS 5
        1 0 1
          |
          v
       +-----------+
DSCP:  |1 0 1 0 0 0|
       +-----------+

          CS5 (40)
```

This mapping can be calculated using:

`DSCP = CoS × 8`

### Class Selector-Aligned CoS-to-DSCP Mapping

| CoS | Binary CoS | DSCP | Class Selector |
|---|---|---|---|
| 0 | `000` | 0 | CS0 |
| 1 | `001` | 8 | CS1 |
| 2 | `010` | 16 | CS2 |
| 3 | `011` | 24 | CS3 |
| 4 | `100` | 32 | CS4 |
| 5 | `101` | 40 | CS5 |
| 6 | `110` | 48 | CS6 |
| 7 | `111` | 56 | CS7 |

This is a common default mapping on Cisco switches, but it is not a mandatory conversion performed by every network device.

### CoS 5 Does Not Automatically Mean EF

An important distinction is that **CoS 5 maps to CS5 (40) when using Class Selector-aligned mapping, not EF (46)**.

Compare:

```text
CoS 5:     101

CS5:       101000 = 40
EF:        101110 = 46
```

Both CS5 and EF have the same first three bits.

However, the lower three bits of their DSCP values differ.

Therefore, to mark CoS 5 traffic EF, the device needs an appropriate mapping or marking policy.

For example:

| CoS | Class Selector mapping | Custom voice mapping |
|---|---|---|
| 0 | CS0 (0) | CS0 (0) |
| 3 | CS3 (24) | CS3 (24) |
| 5 | CS5 (40) | EF (46) |

This distinction is particularly relevant when integrating Layer 2 voice QoS with Layer 3 DiffServ.

---

## DSCP-to-CoS Mapping

The reverse operation maps a six-bit DSCP value to a three-bit CoS value.

One common approach is to use the three most significant DSCP bits.

The remaining three bits are discarded.

```text
       DSCP EF (46)

       1 0 1 1 1 0
       +---+
          |
          v

       CoS = 101 = 5
```

This can be calculated using:

`CoS = DSCP >> 3`

Or:

`CoS = floor(DSCP / 8)`

For example, AF31 uses DSCP 26:

`floor(26 / 8) = 3`

Therefore, a Class Selector-aligned mapping assigns CoS 3.

### DSCP-to-CoS Mapping Table

| DSCP range | CoS |
|---|---|
| 0–7 | 0 |
| 8–15 | 1 |
| 16–23 | 2 |
| 24–31 | 3 |
| 32–39 | 4 |
| 40–47 | 5 |
| 48–55 | 6 |
| 56–63 | 7 |

Examples:

| DSCP | Decimal | CoS |
|---|---|---|
| CS0 | 0 | 0 |
| CS3 | 24 | 3 |
| AF31 | 26 | 3 |
| AF41 | 34 | 4 |
| CS5 | 40 | 5 |
| EF | 46 | 5 |
| CS6 | 48 | 6 |

Notice that CS5 and EF both map to CoS 5.

Likewise, AF31, AF32, and AF33 all map to CoS 3.

### Loss of DSCP Granularity

Mapping DSCP to CoS reduces the available QoS information from six bits to three bits.

For example:

```text
AF31: 011010
AF32: 011100
AF33: 011110
      ^^^
       |
    CoS 3
```

All three AF3 codepoints share the same first three bits.

Therefore, a three-bit CoS value cannot distinguish between their different drop precedence levels.

**A DSCP-to-CoS mapping can lose information, but it does not inherently change the packet's DSCP value.**

A device can retain DSCP AF33 in the IP header while assigning CoS 3 to the outgoing Ethernet frame.

---

## CoS and DSCP Across Routed Hops

Consider an IP packet traveling between two VLAN-based networks through a router.

```text
   SW1                  R1                  SW2
    |                   |                   |
    +-------------------+-------------------+
       Tagged Ethernet        Tagged Ethernet

       CoS 5                  CoS 3
       DSCP EF                DSCP EF
```

When R1 receives the packet:
1. It removes the incoming Ethernet header.
2. The IP packet retains its DSCP EF marking, unless a QoS policy changes it.
3. R1 creates a new Layer 2 header for the outgoing interface.
4. The outgoing CoS value is assigned according to the egress QoS policy.

The outgoing CoS value does not need to match the incoming CoS value.

In this example:
- Incoming CoS = 5.
- Outgoing CoS = 3.
- DSCP remains EF (46).

CoS and DSCP can therefore be manipulated independently.

This is why **DSCP is generally the primary marking mechanism for end-to-end Layer 3 QoS**, while CoS is useful for QoS over individual Ethernet links.

---

## Marking CoS and IP Precedence with MQC

Cisco IOS XE supports MQC commands for marking CoS, IP Precedence, and DSCP.

| Command | Purpose |
|---|---|
| `set cos <value>` | Set the Layer 2 CoS value |
| `set precedence <value>` | Set IP Precedence |
| `set dscp <value>` | Set the complete six-bit DSCP value |

### Setting CoS

To mark traffic with CoS 5:

```text
SW1(config)# policy-map MARK-COS
SW1(config-pmap)# class class-default
SW1(config-pmap-c)# set cos 5
```

When applied at a supported attachment point, this policy assigns CoS 5 to matching traffic.

CoS marking is relevant to tagged Ethernet frames. An untagged frame has no PCP field in which to carry the marking.

### Setting IP Precedence

To set IP Precedence 5:

```text
R1(config)# policy-map MARK-PREC
R1(config-pmap)# class class-default
R1(config-pmap-c)# set precedence 5
```

This marks the packet with IP Precedence 5.

When the objective is to assign a specific DSCP codepoint, such as EF (46), use `set dscp` instead.

```text
R1(config-pmap-c)# set dscp ef
```

Setting IP Precedence 5 is not equivalent to explicitly setting DSCP EF, because IP Precedence does not identify the lower three DSCP bits.

---

## CoS and DSCP Mapping with Table Maps

Cisco IOS XE supports **QoS table maps**, which define how existing QoS values are translated into new values.

For example, a table map can translate CoS 5 to DSCP EF rather than CS5.

### Example: CoS to DSCP

Create a table map:

```text
SW1(config)# table-map COS-TO-DSCP
SW1(config-tablemap)# map from 5 to 46
SW1(config-tablemap)# map from 3 to 24
SW1(config-tablemap)# default 0
```

The table defines:

| Incoming CoS | Resulting DSCP |
|---|---|
| 5 | EF (46) |
| 3 | CS3 (24) |
| Other values | CS0 (0) |

Apply the table map using MQC on a platform supporting this cross-field operation:

```text
SW1(config)# policy-map MARK-DSCP
SW1(config-pmap)# class class-default
SW1(config-pmap-c)# set dscp cos table COS-TO-DSCP
SW1(config-pmap-c)# exit
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# service-policy input MARK-DSCP
```

The command:

`set dscp cos table COS-TO-DSCP`

means:
- **Destination field:** DSCP.
- **Source field:** CoS.
- **Mapping:** The values defined in `COS-TO-DSCP`.

This allows a three-bit CoS marking to select any supported six-bit DSCP value.

### Mapping DSCP to CoS

The corresponding table-map syntax on platforms supporting DSCP-to-CoS translation is:

```text
SW1(config-pmap-c)# set cos dscp table DSCP-TO-COS
```

For example, a table could explicitly map:
- EF (46) to CoS 5.
- AF41 (34) to CoS 4.
- AF31 (26) to CoS 3.

However, **direct DSCP-to-CoS table maps are not supported on all Catalyst 9000 platforms**. Cisco's current Catalyst 9000 documentation explicitly lists this restriction.

Where direct translation is unavailable, an alternative is to classify traffic by DSCP and set CoS explicitly:

```text
SW1(config)# class-map match-all VOICE
SW1(config-cmap)# match dscp ef
SW1(config-cmap)# exit
SW1(config)# policy-map EGRESS-COS
SW1(config-pmap)# class VOICE
SW1(config-pmap-c)# set cos 5
```

Another supported design on some platforms uses internal QoS groups to preserve the classification between ingress and egress.

See [DSCP Marking and Remarking with MQC](mqc-marking.md) for more information about table maps and marking policies.

---

## Trust Boundaries and CoS/DSCP Mapping

The **QoS trust boundary** determines which markings a network device accepts from incoming traffic.

For example, a switch might receive:
- CoS 5 from a trusted IP phone.
- DSCP EF from the phone's voice traffic.
- DSCP EF from an untrusted PC attempting to obtain priority treatment.

The switch must decide whether to trust or replace those markings.

Possible policies include:

| Trust policy | Behavior |
|---|---|
| **Trust CoS** | Accept incoming CoS and use it for QoS classification |
| **Trust DSCP** | Accept incoming DSCP for QoS classification |
| **Remark** | Replace existing markings based on the configured policy |

Trusting CoS does not necessarily mean that the original DSCP value must be preserved.

Likewise, trusting DSCP does not necessarily mean that the original CoS value must be preserved.

An access switch can use a mapping policy to establish consistent DSCP and CoS markings before forwarding traffic into the network.

See [QoS Trust Boundaries](../classification/trust-boundaries.md) for details.

---

## Verification

The following commands are useful for verifying CoS, IP Precedence, and DSCP marking policies.

| Command | Purpose |
|---|---|
| `show class-map` | Display configured traffic classes |
| `show policy-map` | Display marking policies |
| `show policy-map interface <interface>` | Verify attached policies and class statistics |
| `show table-map` | Display configured QoS table maps |
| `show running-config interface <interface>` | Verify interface QoS configuration |

For example:

```text
SW1# show policy-map interface GigabitEthernet1/0/1
```

Use a packet capture to verify the actual markings carried in forwarded frames and packets.

In Wireshark, the relevant fields include:
- **IEEE 802.1Q:** Priority (PCP/CoS).
- **IPv4:** Differentiated Services Codepoint.
- **IPv6:** Differentiated Services Codepoint within Traffic Class.

For example, a capture of a tagged Ethernet frame carrying an IPv4 packet might show:

```text
802.1Q VLAN Tag
    Priority: 5

Internet Protocol Version 4
    DSCP: EF (46)
```

This demonstrates that CoS 5 and DSCP EF can coexist in the same packet, despite having different numerical values.

---

## Key Points

- CoS is a **3-bit Layer 2 marking** carried in the IEEE 802.1Q PCP field.
- IP Precedence is a **3-bit Layer 3 marking** represented by the first three DSCP bits.
- DSCP is a **6-bit Layer 3 marking**, supporting values from 0 through 63.
- IP Precedence can be calculated using `floor(DSCP / 8)`.
- Class Selector codepoints are calculated using `IP Precedence × 8`.
- A common CoS-to-DSCP mapping uses `CoS × 8`.
- A common DSCP-to-CoS mapping uses `floor(DSCP / 8)`.
- **CoS 5 maps to CS5 (40), not EF (46), when using Class Selector-aligned mapping.**
- Mapping DSCP to CoS loses the information contained in the lower three DSCP bits.
- CoS is not preserved across routed hops because the Ethernet header is replaced.
- DSCP can be preserved end-to-end across routed IP networks.
- MQC supports explicit CoS, IP Precedence, and DSCP marking, with table maps providing additional translation capabilities.

---

## References

- [RFC 2474: Definition of the Differentiated Services Field (DS Field) in the IPv4 and IPv6 Headers](https://www.rfc-editor.org/rfc/rfc2474)
- [RFC 9435: Considerations for Assigning a New Recommended Differentiated Services Code Point (DSCP)](https://www.rfc-editor.org/rfc/rfc9435)
- [Cisco: Implement QoS Policies with Differentiated Services Code Point](https://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-packet-marking/10103-dscpvalues.html)
- [Cisco IOS XE 17.17: Catalyst 9300 QoS Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-17/configuration_guide/qos/b_1717_qos_9300_cg/configuring_qos.html)
- [Cisco IOS XE 17: Quality of Service Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/qos/quality-of-service-configuration-guide/m-quality-of-service.html)