# DSCP Values and Codepoint Encoding

The **Differentiated Services Code Point (DSCP)** is a 6-bit value carried in the **Differentiated Services (DS) field** of IPv4 and IPv6 packets.

DSCP identifies the intended **Per-Hop Behavior (PHB)** of a packet, allowing network devices to apply different forwarding treatments based on its marking.

Because DSCP is six bits long, it supports **64 possible values (0–63)**.

However, only certain values have standardized meanings. The most important DSCP categories are:
- **Best Effort (BE):** Default forwarding treatment.
- **Class Selector (CS):** Eight codepoints compatible with IP Precedence.
- **Assured Forwarding (AF):** Twelve codepoints representing four forwarding classes with three drop precedence levels each.
- **Expedited Forwarding (EF):** A codepoint intended for low-delay, low-jitter, low-loss forwarding.

Understanding the binary encoding of these codepoints is useful for configuring, interpreting, and troubleshooting QoS policies.

---

## DSCP Bit Structure

The DSCP occupies the six most significant bits of the 8-bit DS/ECN octet.

```text
 Bit:    0   1   2   3   4   5   6   7
       +---+---+---+---+---+---+---+---+
       |           DSCP        |  ECN  |
       +---+---+---+---+---+---+---+---+
                6 bits          2 bits

 DSCP bit:  5   4   3   2   1   0
 Weight:   32  16   8   4   2   1
```

Each DSCP bit contributes a particular weight to the decimal value.

For example, DSCP EF uses binary `101110`.

```text
 Bit:      5   4   3   2   1   0
 Value:    1   0   1   1   1   0
 Weight:  32  16   8   4   2   1

 Decimal = 32 + 8 + 4 + 2 = 46
```

Therefore, EF has a DSCP value of **46**.

The ECN bits are not included when calculating the DSCP value.

See [IPv4 and IPv6 DS Fields](ds-field.md) for the placement of DSCP and ECN in both IP headers.

---

## Best Effort (BE)

**Best Effort (BE)**, also called the **Default PHB**, uses DSCP 0.

| Property | Value |
|---|---|
| DSCP name | BE / Default / CS0 |
| Decimal | 0 |
| Binary | `000000` |
| IP Precedence | 0 |

BE traffic receives ordinary forwarding treatment without any special QoS assurance.

For example, ordinary web browsing and noncritical file transfers are commonly assigned to Best Effort.

Cisco IOS XE can mark packets with DSCP 0 using:

```text
R1(config-pmap-c)# set dscp default
```

The following commands are equivalent:

```text
R1(config-pmap-c)# set dscp cs0
R1(config-pmap-c)# set dscp 0
```

**Best Effort does not mean that packets must be dropped before all other traffic.** The actual treatment depends on the configured QoS policy.

---

## Class Selector (CS) Codepoints

**Class Selector (CS)** codepoints provide compatibility with the older **IP Precedence** marking system.

IP Precedence uses three bits, supporting values from 0 through 7.

Class Selector codepoints use the following six-bit format:

```text
      IP Precedence
        3 bits
     +-----------+
     |           |
     +---+---+---+---+---+---+
     | x | x | x | 0 | 0 | 0 |
     +---+---+---+---+---+---+
             DSCP
            6 bits
```

The first three bits contain the IP Precedence value, and the final three bits are always zero.

### Class Selector Values

| DSCP | Decimal | Binary | IP Precedence |
|---|---|---|---|
| CS0 | 0 | `000000` | 0 |
| CS1 | 8 | `001000` | 1 |
| CS2 | 16 | `010000` | 2 |
| CS3 | 24 | `011000` | 3 |
| CS4 | 32 | `100000` | 4 |
| CS5 | 40 | `101000` | 5 |
| CS6 | 48 | `110000` | 6 |
| CS7 | 56 | `111000` | 7 |

The decimal DSCP value can be calculated using:

`DSCP = CS number × 8`

For example, CS5:

`5 × 8 = 40`

CS5 therefore uses DSCP 40 (`101000`).

### Class Selector and IP Precedence Compatibility

The Class Selector codepoints preserve the first three bits used by IP Precedence.

For example:

```text
  IP Precedence 5:    101

  CS5:                101000
                      ^^^
                       |
                IP Precedence 5
```

A device performing IP Precedence-based classification interprets CS5 as IP Precedence 5.

However, IP Precedence examines only the first three bits, so multiple DSCP values can correspond to the same IP Precedence value.

For example:

| DSCP | Binary | IP Precedence |
|---|---|---|
| CS5 (40) | `101000` | 5 |
| Voice-Admit (44) | `101100` | 5 |
| EF (46) | `101110` | 5 |

Although these DSCP values are different, all three have the same IP Precedence value.

### Class Selector Usage

Common enterprise QoS conventions include:

| Codepoint | Typical use |
|---|---|
| CS0 | Best Effort |
| CS1 | Low-priority or scavenger traffic |
| CS2 | Organizationally defined service classes |
| CS3 | Call signaling |
| CS4 | Video or other important application traffic |
| CS5 | High-priority application traffic |
| CS6 | Network control traffic |
| CS7 | Network control traffic |

These are examples of common marking conventions, not mandatory application-to-DSCP mappings.

**A higher Class Selector number does not automatically provide a higher-priority queue.** The actual forwarding treatment depends on the device's QoS configuration.

---

## Assured Forwarding (AF) Codepoints

**Assured Forwarding (AF)** defines 12 standard codepoints, organized into:
- **Four forwarding classes**, numbered 1 through 4.
- **Three drop precedence levels**, numbered 1 through 3.

AF codepoints use the format:

`AFxy`

Where:
- **x:** Forwarding class (1–4).
- **y:** Drop precedence (1–3).

For example:
- AF31: Class 3, low drop precedence.
- AF32: Class 3, medium drop precedence.
- AF33: Class 3, high drop precedence.

### AF Binary Encoding

AF uses the following six-bit format:

```text
       Class       Drop      0
     +---------+---------+---+
     | x x x   |  y y    | 0 |
     +---------+---------+---+
       3 bits    2 bits    1
```

The first three bits encode the forwarding class.

| AF class | First three bits |
|---|---|
| AF1x | `001` |
| AF2x | `010` |
| AF3x | `011` |
| AF4x | `100` |

The next two bits encode drop precedence.

| Drop precedence | Two-bit value |
|---|---|
| Low (1) | `01` |
| Medium (2) | `10` |
| High (3) | `11` |

The final bit is always `0`.

For example, AF32:

```text
    Class 3     Drop 2    0
     +-----+    +---+    +-+
     | 011 |    | 10|    |0|
     +-----+    +---+    +-+

     Binary:   011100
     Decimal:  28
```

### Calculating AF DSCP Values

The decimal value of an AF codepoint can be calculated using:

`DSCP = (8 × x) + (2 × y)`

Where `x` is the forwarding class and `y` is the drop precedence.

For example, AF43:

`(8 × 4) + (2 × 3) = 38`

Therefore, AF43 uses:
- Decimal: 38
- Binary: `100110`

### Complete AF Codepoint Table

| DSCP | Decimal | Binary | Drop precedence |
|---|---|---|---|
| AF11 | 10 | `001010` | Low |
| AF12 | 12 | `001100` | Medium |
| AF13 | 14 | `001110` | High |
| AF21 | 18 | `010010` | Low |
| AF22 | 20 | `010100` | Medium |
| AF23 | 22 | `010110` | High |
| AF31 | 26 | `011010` | Low |
| AF32 | 28 | `011100` | Medium |
| AF33 | 30 | `011110` | High |
| AF41 | 34 | `100010` | Low |
| AF42 | 36 | `100100` | Medium |
| AF43 | 38 | `100110` | High |

The same values can be organized by forwarding class:

| Drop precedence | AF1 | AF2 | AF3 | AF4 |
|---|---|---|---|---|
| **Low (1)** | AF11 (10) | AF21 (18) | AF31 (26) | AF41 (34) |
| **Medium (2)** | AF12 (12) | AF22 (20) | AF32 (28) | AF42 (36) |
| **High (3)** | AF13 (14) | AF23 (22) | AF33 (30) | AF43 (38) |

### AF Drop Precedence

The three drop precedence levels are used for congestion avoidance **within the same AF class**.

For example, AF31, AF32, and AF33 belong to AF Class 3.

When congestion occurs, a device can use different drop probabilities for these packets.

The intended relationship is:

`P(drop AF33) ≥ P(drop AF32) ≥ P(drop AF31)`

This behavior can be implemented using mechanisms such as **Weighted Random Early Detection (WRED)**.

Drop precedence does not determine transmission priority within the class.

For example, packets marked AF33 do not automatically receive lower transmission priority than AF31 packets.

Also, **AF Class 4 is not inherently more important than AF Class 3**. The forwarding resources allocated to each class are determined by the QoS policy.

See [Per-Hop Behaviors (BE, EF, AF, CS)](../diffserv/phb.md) for detailed AF forwarding behavior.

---

## Expedited Forwarding (EF)

**Expedited Forwarding (EF)** uses DSCP 46, defined in RFC 3246.

| Property | Value |
|---|---|
| DSCP name | EF |
| Decimal | 46 |
| Binary | `101110` |
| IP Precedence | 5 |

EF is designed for traffic requiring:
- Low delay.
- Low jitter.
- Low packet loss.
- An assured forwarding rate.

EF is commonly used for **VoIP bearer traffic**.

### EF vs. CS5

EF and CS5 have the same first three bits:

```text
  CS5: 101000 = 40
  EF:  101110 = 46
       ^^^
        |
    IP Precedence 5
```

However, their complete DSCP values are different.

A router matching IP Precedence 5 can match either marking, while a router matching DSCP EF specifically identifies DSCP 46.

This distinction is important when configuring QoS policies.

**EF does not automatically enable strict-priority queuing.** The appropriate scheduling mechanism, such as Low Latency Queuing (LLQ), must be configured separately.

---

## Other Standardized DSCP Values

In addition to BE, CS, AF, and EF, the IETF has standardized several other DSCP values.

| Codepoint | Decimal | Binary | Purpose |
|---|---|---|---|
| **LE** | 1 | `000001` | Lower Effort |
| **VOICE-ADMIT** | 44 | `101100` | Admission-controlled voice |
| **NQB** | 45 | `101101` | Non-Queue-Building traffic |

### Lower Effort (LE)

**Lower Effort (LE)** is defined in RFC 8622 and uses DSCP 1.

It identifies traffic that should receive lower forwarding preference than ordinary Best Effort traffic when competing for network resources.

LE is distinct from CS1 (8).

Although CS1 is commonly used for scavenger traffic in enterprise QoS designs, LE is the standardized Lower Effort PHB.

### Voice-Admit

**Voice-Admit** is defined in RFC 5865 and uses DSCP 44.

It identifies voice traffic associated with an admission-controlled service.

Unlike ordinary EF traffic, Voice-Admit is specifically intended for environments where admission control determines which traffic is allowed to enter the service.

### Non-Queue-Building (NQB)

**Non-Queue-Building (NQB)** uses DSCP 45.

It identifies traffic intended for a forwarding service designed to avoid delays caused by competition with queue-building traffic.

NQB is not a replacement for EF and does not inherently require strict-priority queuing.

These additional codepoints are useful to recognize, but the principal CCIE EI focus is on **BE, CS, AF, and EF**.

---

## DSCP Codepoint Assignment

Although the DSCP field supports 64 values, they are not all assigned to standardized PHBs.

IANA divides the codepoint space into three pools.

| Pool | Binary format | Purpose |
|---|---|---|
| **Pool 1** | `xxxxx0` | Standards-based assignments |
| **Pool 2** | `xxxx11` | Experimental or local use |
| **Pool 3** | `xxxx01` | Standards-based assignments |

Here, `x` represents either 0 or 1.

Pool 1 includes the familiar CS, AF, and EF codepoints.

Pool 2 is reserved for experimental or local use, allowing administrators to define their own meanings within a DiffServ domain.

Pool 3 was originally available for experimental or local use but was subsequently reserved for standardized assignments by RFC 8436.

For example, LE (1) and NQB (45) belong to Pool 3.

**The DSCP value does not inherently determine how a packet is forwarded.** A network device maps the codepoint to a PHB according to its configured QoS policy.

---

## Common Enterprise DSCP Markings

The following table provides examples of DSCP markings frequently encountered in enterprise QoS designs.

| Traffic | Common DSCP | Decimal |
|---|---|---|
| Voice bearer | EF | 46 |
| Voice signaling | CS3 | 24 |
| Interactive video | AF41 | 34 |
| Business-critical data | AF31 | 26 |
| Standard application traffic | CS0 | 0 |
| Scavenger/background traffic | CS1 | 8 |
| Routing and network control | CS6 | 48 |

These markings are design conventions rather than mandatory mappings.

For example, an enterprise might use AF21 instead of AF31 for a particular business application, depending on its QoS model.

A consistent mapping between DSCP values and forwarding treatments is more important than assigning every application a particular standardized codepoint.

---

## DSCP Values in Cisco IOS XE

Cisco IOS XE supports both named and numerical DSCP values in MQC configuration.

### Matching DSCP Values

To classify packets marked EF:

```text
R1(config)# class-map match-all VOICE
R1(config-cmap)# match dscp ef
```

The same matching criterion can be specified numerically:

```text
R1(config-cmap)# match dscp 46
```

To match multiple DSCP values:

```text
R1(config)# class-map match-any AF3
R1(config-cmap)# match dscp af31 af32 af33
```

This class matches packets marked AF31, AF32, or AF33.

### Setting DSCP Values

DSCP values can be assigned using the `set dscp` command within a policy map.

For example:

```text
R1(config)# policy-map MARKING
R1(config-pmap)# class VOICE
R1(config-pmap-c)# set dscp ef
```

The numerical equivalent is:

```text
R1(config-pmap-c)# set dscp 46
```

Both commands mark matching packets with DSCP 46.

For detailed MQC marking configurations, see [DSCP Marking and Remarking with MQC](mqc-marking.md).

---

## DSCP vs. Complete DS Field Value

The DSCP value occupies six bits, but the complete DS/ECN octet is eight bits long.

When ECN is `00`, the complete octet is calculated using:

`DS/ECN octet = DSCP × 4`

For example, EF:

`46 × 4 = 184 = 0xB8`

This gives:

```text
 DSCP:       101110    = 46
 ECN:        00        = 0

 Full octet: 10111000  = 184 (0xB8)
```

The following table assumes ECN is `00`:

| DSCP | DSCP decimal | Complete octet (hex) |
|---|---|---|
| CS0 | 0 | `0x00` |
| CS1 | 8 | `0x20` |
| AF21 | 18 | `0x48` |
| CS3 | 24 | `0x60` |
| AF31 | 26 | `0x68` |
| AF41 | 34 | `0x88` |
| CS5 | 40 | `0xA0` |
| EF | 46 | `0xB8` |
| CS6 | 48 | `0xC0` |

When examining packet captures, distinguish between:
- The **6-bit DSCP value**, such as EF (46).
- The **8-bit DS/ECN octet**, such as `0xB8`.

If ECN is nonzero, the complete octet value changes even when the DSCP remains the same.

---

## Key Points

- DSCP is a **6-bit value**, supporting 64 possible codepoints (0–63).
- BE uses DSCP 0 (`000000`).
- Class Selector codepoints use the format `xxx000`.
- Class Selector values are calculated using `CS number × 8`.
- AF codepoints use the format `xxxyy0`, with three bits for the class and two bits for drop precedence.
- AF values are calculated using `(8 × class) + (2 × drop precedence)`.
- AF provides four forwarding classes with three drop precedence levels each, producing 12 codepoints.
- EF uses DSCP 46 (`101110`).
- IP Precedence examines only the first three DSCP bits.
- DSCP values identify intended PHBs but do not independently enable queuing, prioritization, or congestion avoidance.
- Cisco IOS XE accepts both named and numerical DSCP values in MQC.

---

## References

- [RFC 2474: Definition of the Differentiated Services Field (DS Field) in the IPv4 and IPv6 Headers](https://www.rfc-editor.org/rfc/rfc2474)
- [RFC 2597: Assured Forwarding PHB Group](https://www.rfc-editor.org/rfc/rfc2597)
- [RFC 3246: An Expedited Forwarding PHB](https://www.rfc-editor.org/rfc/rfc3246)
- [RFC 5865: A Differentiated Services Code Point (DSCP) for Capacity-Admitted Traffic](https://www.rfc-editor.org/rfc/rfc5865)
- [RFC 8622: A Lower-Effort Per-Hop Behavior (LE PHB) for Differentiated Services](https://www.rfc-editor.org/rfc/rfc8622)
- [RFC 9956: A Non-Queue-Building Per-Hop Behavior (NQB PHB) for Differentiated Services](https://www.rfc-editor.org/rfc/rfc9956)
- [IANA: Differentiated Services Field Codepoints (DSCP)](https://www.iana.org/assignments/dscp-registry/dscp-registry.xhtml)
- [Cisco: Implement QoS Policies with Differentiated Services Code Point](https://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-packet-marking/10103-dscpvalues.html)