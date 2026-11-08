# IPv4 and IPv6 DS Fields
The **Differentiated Services (DS) field** is a 6-bit field in both IPv4 and IPv6 headers. It carries a **Differentiated Services Code Point (DSCP)**, a value from 0 to 63 used to select the intended **Per-Hop Behavior (PHB)** of an IP packet.

RFC 2474 originally introduced the name **DS field** for the 8-bit IPv4 Type of Service (ToS) and IPv6 Traffic Class octets. RFC 3260 subsequently clarified that the **DS field consists of the six most significant bits** of either octet. The remaining two bits are used for **Explicit Congestion Notification (ECN)**.

Although IPv4 and IPv6 use different header formats, **the DS field and its DSCP encoding are identical in both protocols**. Network devices can classify packets and apply QoS policies based on their DSCP values.

---

## IPv4 DS Field
The IPv4 header contains an 8-bit octet originally called **Type of Service (ToS)**. Its six most significant bits form the **DS field**, while its two least significant bits carry ECN.

The resulting format is:

```text
        IPv4 ToS Octet (DS Field + ECN)
Bit:   0   1   2   3   4   5   6   7
     +---+---+---+---+---+---+---+---+
     |       DS (DSCP)       |  ECN  |
     +---+---+---+---+---+---+---+---+
              6 bits          2 bits
```

| Field | Bits | Purpose |
|---|---|---|
| **DS (DSCP)** | 0–5 | Encodes the DSCP value selecting a Per-Hop Behavior |
| **ECN** | 6–7 | Communicates congestion information |

The DS field encodes **64 possible DSCP values (0–63)**, while the ECN bits support four values (0–3).

### Position in the IPv4 Header
The former ToS octet is the **second byte of the IPv4 header**, immediately following the Version and Internet Header Length (IHL) fields. The DS field occupies its upper six bits.

```text
                  IPv4 Header
  0                   1                   2                   3
  +---------+---------+-------------------+-------------------+
  | Version |   IHL   |     DSCP + ECN    |    Total Length   |
  +---------+---------+-------------------+-------------------+
  |             Identification            | Flags / Frag Off  |
  +---------------------------------------+-------------------+
  |     TTL     |   Protocol  |       Header Checksum         |
  +-----------------------------------------------------------+
  |                      Source Address                       |
  +-----------------------------------------------------------+
  |                    Destination Address                    |
  +-----------------------------------------------------------+
```

The IPv4 header must be updated when a router remarks the DS field.

Because IPv4 includes a **header checksum**, changing the DSCP or ECN bits also requires the router to update the IPv4 header checksum.

---

## IPv6 DS Field
The IPv6 header contains an 8-bit **Traffic Class** octet. As with IPv4, its six most significant bits form the **DS field** and the remaining two bits carry ECN.

Thus, the IPv6 Traffic Class octet and the IPv4 former ToS octet use the same DSCP and ECN encoding.

```text
             IPv6 Traffic Class Octet (DS Field + ECN)
Bit:   0   1   2   3   4   5   6   7
     +---+---+---+---+---+---+---+---+
     |       DS (DSCP)       |  ECN  |
     +---+---+---+---+---+---+---+---+
              6 bits          2 bits
```

The 8-bit Traffic Class octet occupies bits **4–11** of the IPv6 header, following the 4-bit Version field. The DS field itself occupies **bits 4–9**; ECN occupies bits **10–11**.

### Position in the IPv6 Header
The first 32 bits of the IPv6 header contain three fields:

```text
                 IPv6 Header
     4 bits       8 bits              20 bits
  +---------+-----------------+-----------------------+
  | Version |Traffic Class|         Flow Label        |
  +---------+-----------------+-----------------------+
  |       Payload Length      | Next Header| Hop Limit|
  +---------------------------------------------------+
  |                                                   |
  |                  Source Address                   |
  |                     128 bits                      |
  |                                                   |
  +---------------------------------------------------+
  |                                                   |
  |                Destination Address                |
  |                     128 bits                      |
  |                                                   |
  +---------------------------------------------------+
```

Unlike the IPv4 former ToS octet, the IPv6 Traffic Class octet does **not** occupy a complete, independently aligned byte in the packet. It spans portions of the first two bytes.

However, its logical 8-bit value is interpreted in exactly the same way.

### Traffic Class vs. Flow Label
The IPv6 **Traffic Class** octet (which contains the DS field) and **Flow Label** field serve different purposes.

| Field | Size | Purpose |
|---|---|---|
| **Traffic Class** | 8 bits | Contains the 6-bit DS field and 2-bit ECN value |
| **Flow Label** | 20 bits | Identifies packets belonging to the same flow |

The Flow Label is not a DSCP marking and does not directly identify a DiffServ PHB.

Another important difference from IPv4 is that **IPv6 does not have a header checksum**. Therefore, changing the DS field does not require an IP header checksum update.

---

## DSCP and ECN Encoding
The 6-bit DS field and 2-bit ECN value share the former IPv4 ToS octet or IPv6 Traffic Class octet, but have different functions.

The DSCP value encoded in the DS field selects the PHB the network should apply, while the ECN bits indicate whether the packet is ECN-capable and whether congestion has been experienced.

### DSCP Encoding Example
Consider a packet marked **EF (46)**.

The decimal value 46 is represented by six binary bits:

```text
DSCP 46 = 101110
```

If the packet has ECN set to `00`, the complete 8-bit field becomes:

```text
  DSCP = 46    ECN = 0
  1 0 1 1 1 0   0 0
  +-----------+------+
  |  101110   |  00  |
  +-----------+------+
  Binary:      10111000
  Hexadecimal: 0xB8
  Decimal:     184
```

Therefore:

- **DSCP value:** 46
- **ECN value:** 0
- **Complete 8-bit octet:** 184 (`0xB8`)

**DSCP 46 and the complete octet value 184 are not the same thing.**

The DSCP value is encoded in the six-bit DS field, while the complete octet also includes ECN.

### Converting Between DSCP and the Full Field
The complete 8-bit octet value can be calculated using:

`Field value = (DSCP × 4) + ECN`

For example, with DSCP AF31 (26) and ECN 0:

`(26 × 4) + 0 = 104 = 0x68`

To extract the DSCP value from the complete octet:

`DSCP = Field value >> 2`

The right shift removes the two ECN bits.

Similarly:

`ECN = Field value & 0x03`

This extracts only the two least significant bits.

The following examples assume ECN is 0:

| DSCP | Decimal DSCP | Complete octet (binary) | Hexadecimal |
|---|---|---|---|
| CS0 | 0 | `00000000` | `0x00` |
| CS1 | 8 | `00100000` | `0x20` |
| AF21 | 18 | `01001000` | `0x48` |
| AF31 | 26 | `01101000` | `0x68` |
| AF41 | 34 | `10001000` | `0x88` |
| CS5 | 40 | `10100000` | `0xA0` |
| EF | 46 | `10111000` | `0xB8` |
| CS6 | 48 | `11000000` | `0xC0` |

---

## Explicit Congestion Notification (ECN)
**Explicit Congestion Notification (ECN)** uses the final two bits of the IPv4 former ToS octet or IPv6 Traffic Class octet.

ECN allows supporting routers to signal congestion by marking packets instead of necessarily dropping them.

The four ECN codepoints are:

| Binary | Decimal | Name | Meaning |
|---|---|---|---|
| `00` | 0 | Not-ECT | Packet is not ECN-capable |
| `01` | 1 | ECT(1) | ECN-capable transport |
| `10` | 2 | ECT(0) | ECN-capable transport |
| `11` | 3 | CE | Congestion Experienced |

For example, an ECN-capable packet might arrive at a congested router with ECN `10` (ECT(0)).

If the router supports ECN and its congestion management policy decides to mark the packet, it changes the ECN field to `11` (CE).

```text
        Before Congestion
  +-----------------+--------+
  |    DSCP = 46    | ECN 10 |
  +-----------------+--------+
            |
            v
     Congested Router
            |
            v
        After Marking
  +-----------------+--------+
  |    DSCP = 46    | ECN 11 |
  +-----------------+--------+
```

**The DSCP value remains unchanged.** Only the ECN field is modified.

The receiving transport can then use the congestion indication to adjust its sending behavior.

For TCP, this involves ECN feedback to the sender.

ECN is covered further in [Explicit Congestion Notification (ECN)](../congestion/ecn.md).

---

## DSCP vs. IP Precedence
Before DiffServ, IPv4 QoS classification used **IP Precedence**, which occupied the first three bits of the original ToS octet.

The original ToS format included precedence and service-indicator bits:

```text
          Original IPv4 ToS Octet
  +-----------+-----------+-------+
  | Precedence|  D  T  R  |  0  0 |
  +-----------+-----------+-------+
     3 bits      3 bits     2 bits
```

Here, D, T, and R represented delay, throughput, and reliability preferences.

DiffServ superseded this interpretation with the 6-bit DS field. The first three bits still align with the former IP Precedence bits.

```text
              Modern Format
  +-----------------------+-------+
  |          DSCP         |  ECN  |
  +-----------------------+-------+
             6 bits        2 bits
   ^---------^
   First 3 bits are also
   used by IP Precedence
```

This provides backward compatibility with IP Precedence-based classification.

For example:

| DSCP | Binary | First 3 bits | IP Precedence |
|---|---|---|---|
| CS0 (0) | `000000` | `000` | 0 |
| CS3 (24) | `011000` | `011` | 3 |
| CS5 (40) | `101000` | `101` | 5 |
| EF (46) | `101110` | `101` | 5 |
| CS6 (48) | `110000` | `110` | 6 |

Both CS5 and EF correspond to IP Precedence 5 because their first three bits are identical.

However, their full 6-bit DSCP values are different.

**IP Precedence cannot distinguish between DSCP values that share the same first three bits.**

See [CoS, IP Precedence, and DSCP Mapping](cos-dscp-mapping.md) for more details.

---

## DSCP Preservation Across Routed Hops
Because the DS field is part of the IP header, DSCP markings can be carried across multiple routed hops.

For example:

```text
          DSCP EF          DSCP EF
   PC -------------- R1 ------------ R2
                       \              \
                        \              \
                      Preserve       Preserve
                        DSCP           DSCP
```

A router does not need to rewrite DSCP when forwarding a packet.

However, a QoS policy may explicitly change the marking.

For example, a router at a DiffServ domain boundary might remark untrusted EF traffic to CS0.

```text
             DSCP EF                 DSCP CS0
   PC ---------------- R1 --------------------- R2
                        |
                        |
                   Remark DSCP
                    EF -> CS0
```

The marking can also change when traffic enters an IP tunnel, where an additional IP header is introduced.

The **inner and outer IP headers have separate DS fields**. How those markings are copied or mapped depends on the tunnel implementation and configured QoS behavior.

End-to-end QoS therefore requires consistent marking and remarking policies across the network.

---

## Verification with Packet Capture
Packet capture tools such as **Wireshark** can display the DSCP and ECN fields separately.

### IPv4 Packet
For an IPv4 packet marked EF, Wireshark may show:

```text
Internet Protocol Version 4
    Differentiated Services Field: 0xb8
        Differentiated Services Codepoint: EF (46)
        Explicit Congestion Notification: Not ECN-Capable Transport (0)
```

Useful Wireshark display filters include:

```text
ip.dsfield.dscp == 46
```

Matches IPv4 packets marked EF.

```text
ip.dsfield.ecn == 3
```

Matches IPv4 packets with ECN CE (Congestion Experienced).

### IPv6 Packet
For an IPv6 packet marked AF31, Wireshark may show:

```text
Internet Protocol Version 6
    Traffic Class: 0x68
        Differentiated Services Codepoint: AF31 (26)
        Explicit Congestion Notification: Not ECN-Capable Transport (0)
```

Useful filters include:

```text
ipv6.tclass.dscp == 26
```

Matches IPv6 packets marked AF31.

```text
ipv6.tclass.ecn == 3
```

Matches IPv6 packets with ECN CE.

### IOS XE Verification
On Cisco IOS XE, the following command can be used to verify the classification and marking actions of an applied QoS policy:

```text
R1# show policy-map interface GigabitEthernet1
```

The output can display matching counters and configured DSCP marking actions.

To confirm the actual DSCP and ECN bits in forwarded packets, a packet capture provides direct visibility into the IP headers.

---

## Key Points
- Both IPv4 and IPv6 contain a **6-bit DS field** carrying a DSCP value.
- RFC 2474 originally introduced the name DS field for the full 8-bit octet; RFC 3260 clarified that the DS field contains only its **six most significant bits**.
- The remaining two bits of each octet are used for **ECN**.
- In IPv4, the DS field occupies the first six bits of the second header byte. In IPv6, it occupies header bits 4–9.
- DSCP supports 64 codepoints, numbered 0 through 63.
- ECN uses four codepoints: Not-ECT, ECT(1), ECT(0), and CE.
- The complete 8-bit octet value is calculated using `(DSCP × 4) + ECN`.
- **IP Precedence uses the first three bits of the DS field**, allowing partial backward compatibility.
- IPv4 header checksum updates are required when DSCP changes; IPv6 has no header checksum.
- DSCP markings can be preserved or remarked as packets traverse a network.

---

## References
- [RFC 2474: Definition of the Differentiated Services Field (DS Field) in the IPv4 and IPv6 Headers](https://www.rfc-editor.org/rfc/rfc2474)
- [RFC 3260: New Terminology and Clarifications for Diffserv](https://www.rfc-editor.org/rfc/rfc3260)
- [RFC 3168: The Addition of Explicit Congestion Notification (ECN) to IP](https://www.rfc-editor.org/rfc/rfc3168)
- [RFC 8200: Internet Protocol, Version 6 (IPv6) Specification](https://www.rfc-editor.org/rfc/rfc8200)
- [Cisco: Implement QoS Policies with Differentiated Services Code Point](https://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-packet-marking/10103-dscpvalues.html)
