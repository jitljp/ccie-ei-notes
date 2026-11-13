# 1.5.a (v) 4-Byte AS Numbers

Originally, BGP used **2-byte (16-bit) Autonomous System Numbers (ASNs)**, allowing values from 0 to 65535.

As the number of autonomous systems increased, this address space became insufficient.

**4-byte (32-bit) ASNs**, defined in [RFC 6793](https://www.rfc-editor.org/rfc/rfc6793.html), expand the ASN space to 32 bits.

| Characteristic | 2-Byte ASN | 4-Byte ASN |
|---|---|---|
| Size | 16 bits | 32 bits |
| Numeric range | 0 to 65535 | 0 to 4294967295 |

Not all values are available for public ASN assignment. Certain values and ranges are reserved.

**4-byte ASNs are backward compatible with 2-byte ASNs** through extensions to the BGP protocol.

Cisco IOS XE supports both 2-byte and 4-byte ASNs without requiring a separate BGP process type.

---

## ASN Notation

There are three formats for representing ASNs:

- **asplain:** A single decimal number.
- **asdot:** Uses decimal notation for 2-byte ASNs and dotted notation for ASNs of 65536 or greater.
- **asdot+:** Always uses dotted notation, including for 2-byte ASNs.

These are simply different textual representations of the same ASN.

### asplain

**asplain** represents an ASN as a single decimal number.

For example:

```text
65001
65546
100000
4200000001
```

**asplain is the default ASN notation on modern Cisco IOS XE.**

It is also the format recommended by RFC 5396.

### asdot

**asdot** represents a 4-byte ASN as two 16-bit decimal values separated by a period.

For example:

```text
1.10
1.100
2.0
```

The two values represent:

```text
<High 16 bits>.<Low 16 bits>
```

Each component ranges from 0 to 65535.

### Conversion Between asplain and asdot

Convert asdot to asplain using:

```text
asplain = (High × 65536) + Low
```

For example:

```text
1.10 = (1 × 65536) + 10
     = 65546
```

Another example:

```text
2.100 = (2 × 65536) + 100
      = 131172
```

To convert asplain to asdot:

```text
High = ASN // 65536
Low  = ASN % 65536
```

Here, `//` represents integer division, and `%` represents the remainder.

For example:

```text
ASN = 100000

High = 100000 // 65536 = 1
Low  = 100000 % 65536  = 34464

asdot = 1.34464
```

### Conversion Table

| asplain | asdot |
|---|---|
| 65000 | 65000 |
| 65535 | 65535 |
| 65536 | 1.0 |
| 65537 | 1.1 |
| 65546 | 1.10 |
| 65550 | 1.14 |
| 100000 | 1.34464 |
| 131072 | 2.0 |
| 4294967295 | 65535.65535 |

In asdot notation, values below 65536 are written without a period.

### asdot+

**asdot+** always uses the dotted format, even for 2-byte ASNs.

For example:

| ASN | asplain | asdot | asdot+ |
|---|---|---|---|
| 65000 | 65000 | 65000 | 0.65000 |
| 65546 | 65546 | 1.10 | 1.10 |
| 131072 | 131072 | 2.0 | 2.0 |

Cisco IOS XE supports **asplain and asdot**. The asdot+ distinction is primarily relevant when studying the RFC notation formats.

---

## Configuring 4-Byte ASNs

Cisco IOS XE supports configuring 4-byte ASNs using either asplain or asdot notation.

Consider two EBGP routers:

```text
         AS 65546                         AS 65547
          (1.10)                           (1.11)

            R1 ------------------------------ R2
           E0/0                              E0/0
       10.0.12.1/30                      10.0.12.2/30

                                         Lo0: 192.0.2.1/24
```

### R1 Configuration

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip address 10.0.12.1 255.255.255.252
R1(config-if)# no shutdown

R1(config)# router bgp 65546
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# neighbor 10.0.12.2 remote-as 65547
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 activate
```

### R2 Configuration

```text
R2(config)# interface Ethernet0/0
R2(config-if)# ip address 10.0.12.2 255.255.255.252
R2(config-if)# no shutdown

R2(config)# interface Loopback0
R2(config-if)# ip address 192.0.2.1 255.255.255.0

R2(config)# router bgp 65547
R2(config-router)# bgp router-id 2.2.2.2
R2(config-router)# neighbor 10.0.12.1 remote-as 65546
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.12.1 activate
R2(config-router-af)# network 192.0.2.0 mask 255.255.255.0
```

The configuration is identical to normal BGP peering, except that the ASNs exceed the 2-byte range.

No additional command is needed to enable 4-byte ASN support.

### Equivalent asdot Configuration

R1 could instead use:

```text
R1(config)# router bgp 1.10
R1(config-router)# neighbor 10.0.12.2 remote-as 1.11
```

This is equivalent to:

```text
R1(config)# router bgp 65546
R1(config-router)# neighbor 10.0.12.2 remote-as 65547
```

**The configured notation does not change the actual ASN.**

---

## Configuring the ASN Display Format

By default, modern Cisco IOS XE displays 4-byte ASNs in **asplain notation**.

To display them in asdot notation:

```text
R1(config)# router bgp 65546
R1(config-router)# bgp asnotation dot
```

Cisco documents a BGP hard reset to apply this change to existing sessions:

```text
R1# clear ip bgp *
```

**Warning:** This resets all BGP sessions in the process. Do not perform it casually on a production router.

### Default Output

```text
R1# show ip bgp summary
BGP router identifier 1.1.1.1, local AS number 65546

Neighbor        V         AS
10.0.12.2       4      65547
```

### With `bgp asnotation dot`

```text
R1# show ip bgp summary
BGP router identifier 1.1.1.1, local AS number 1.10

Neighbor        V         AS
10.0.12.2       4       1.11
```

The change affects:

1. ASN display in BGP show commands.
2. ASN matching in BGP AS-path regular expressions.

It does **not** change the ASN used in BGP messages.

Restore the default format:

```text
R1(config)# router bgp 65546
R1(config-router)# no bgp asnotation dot
```

Cisco also documents resetting BGP sessions after changing back to asplain.

---

## BGP 4-Byte ASN Capability

The original BGP OPEN message contains a **2-byte My Autonomous System field**.

However, 4-byte ASNs cannot fit in this field.

RFC 6793 introduces the **4-Octet AS Number Capability**, which is exchanged during BGP session establishment.

| Field | Value |
|---|---|
| Capability Code | 65 |
| Capability Length | 4 bytes |
| Capability Value | Local 4-byte ASN |

For example, R1 belongs to AS 65546:

```text
R1                                 R2
 |                                  |
 |------------ OPEN --------------->|
 |                                  |
 |  My AS: 23456                    |
 |  4-Byte ASN Capability:          |
 |    Code: 65                      |
 |    Value: 65546                  |
 |                                  |
 |<------------ OPEN ---------------|
 |                                  |
```

The 2-byte My AS field uses the reserved value **23456 (AS_TRANS)** because R1's actual ASN cannot fit in 16 bits.

The 4-byte ASN capability carries the actual ASN, 65546.

When both BGP speakers support 4-byte ASNs, they use the capability value to identify the peer's ASN.

### Important Distinction

**Using a 2-byte ASN does not mean a router lacks 4-byte ASN support.**

For example, a modern router configured in AS 65001 still advertises the 4-byte ASN capability.

Its capability value is simply 65001.

---

## AS_TRANS (23456)

**AS_TRANS** is the reserved ASN **23456**, introduced to support interoperability between 2-byte and 4-byte BGP implementations.

It acts as a placeholder for ASNs that cannot be represented in 16 bits.

For example:

```text
Actual ASN: 65546
AS_TRANS:   23456
```

When communicating using 2-byte ASN encoding, the 4-byte ASN may be represented by AS_TRANS.

### AS_TRANS in BGP OPEN

A 4-byte-capable router belonging to AS 65546 uses:

```text
BGP OPEN:

My Autonomous System: 23456
4-Byte ASN Capability: 65546
```

A 4-byte-capable router belonging to AS 65001 uses:

```text
BGP OPEN:

My Autonomous System: 65001
4-Byte ASN Capability: 65001
```

**AS_TRANS is used in the OPEN My AS field only when the actual ASN cannot be represented using 2 bytes.**

AS_TRANS is reserved for protocol compatibility and must not be used as a normal production ASN.

Cisco IOS XE does not permit configuring ASN 23456 as an ordinary BGP ASN.

---

## 4-Byte ASNs in AS_PATH

BGP uses the **AS_PATH** attribute to identify the autonomous systems traversed by a route.

The handling of 4-byte ASNs depends on whether the receiving peer supports the 4-byte ASN capability.

### Both Peers Support 4-Byte ASNs

When both neighbors support the capability, the existing `AS_PATH` attribute carries ASNs encoded as 4-byte values.

For example:

```text
       AS 65546                AS 65001              AS 65547

          R1 -------------------- R2 -------------------- R3
```

Suppose R1 originates `192.0.2.0/24`.

R2 receives:

```text
AS_PATH: 65546
```

R3 receives:

```text
AS_PATH: 65001 65546
```

Although AS 65001 fits in 2 bytes, the ASNs in the `AS_PATH` attribute are encoded using 4 bytes because the peers negotiated 4-byte ASN support.

**The AS_PATH attribute is not replaced by AS4_PATH when both peers support 4-byte ASNs.**

---

## AS4_PATH

The **AS4_PATH** attribute preserves 4-byte ASN information when a route traverses legacy BGP speakers that do not support 4-byte ASN capability.

| Property | Value |
|---|---|
| Attribute | AS4_PATH |
| Type Code | 17 |
| Category | Optional Transitive |
| Purpose | Preserve 4-byte AS_PATH information across legacy BGP speakers |

An older BGP speaker can understand the normal 2-byte-encoded AS_PATH but cannot fully interpret a 4-byte ASN.

To preserve the original information:

1. The 4-byte-capable router sends a 2-byte-encoded AS_PATH to the legacy neighbor.
2. Any ASN exceeding 65535 is replaced with AS_TRANS in that AS_PATH.
3. An AS4_PATH attribute carries the original 4-byte ASN information.
4. The legacy router propagates the optional transitive AS4_PATH attribute.
5. A downstream 4-byte-capable router can reconstruct the original AS path.

### Example

Consider:

```text
       4-Byte Capable        4-Byte Capable           Legacy
          AS 65546              AS 65010              AS 65020

             R1 ------------------- R2 ------------------- R3

                 4-byte AS_PATH          2-byte AS_PATH
                                         + AS4_PATH
```

R1 originates `192.0.2.0/24`.

R2 receives the route from R1:

```text
AS_PATH: 65546
```

When R2 advertises the route to legacy R3, it prepends its own ASN and uses compatibility encoding.

```text
AS_PATH:  65010 23456
AS4_PATH: 65010 65546
```

In the 2-byte AS_PATH, `23456` represents AS 65546.

The AS4_PATH attribute preserves the actual ASN.

### AS_PATH Reconstruction

Now suppose R3 advertises the route to R4, a 4-byte-capable router in AS 65030.

```text
        R1 -------- R2 -------- R3 -------- R4
      AS 65546    AS 65010    AS 65020    AS 65030
       4-byte     4-byte       Legacy      4-byte
```

R3 prepends its ASN to the normal AS_PATH:

```text
AS_PATH:  65020 65010 23456
AS4_PATH: 65010 65546
```

The legacy router leaves the AS4_PATH information unchanged.

R4 uses the two attributes to reconstruct:

```text
AS_PATH: 65020 65010 65546
```

The original 4-byte ASN is preserved.

**The reconstruction is not a simple replacement of every AS_TRANS with an ASN from AS4_PATH.**

RFC 6793 specifies how the leading AS_PATH segments and preserved AS4_PATH segments are combined. Some legacy route aggregation scenarios can prevent complete reconstruction.

### Important Compatibility Limitation

Although 4-byte ASN information can traverse legacy BGP speakers, RFC 6793 specifies that a 4-byte-capable speaker can directly peer with a truly legacy speaker only when the capable speaker has a 2-byte-representable ASN.

For example:

- Modern AS 65010 to legacy AS 65020: Supported.
- Modern AS 65546 to truly legacy AS 65020: Not supported as an ordinary direct peering relationship under RFC 6793.

A truly legacy speaker cannot negotiate the actual 4-byte ASN carried in Capability 65.

**All BGP speakers in an AS using a non-mappable 4-byte ASN should support 4-byte ASNs.**

---

## AS4_AGGREGATOR

The **AGGREGATOR** attribute identifies the ASN and router ID of the router that performed BGP route aggregation.

With 4-byte ASNs, legacy BGP speakers cannot represent an aggregating router's ASN when it exceeds 65535.

RFC 6793 introduces **AS4_AGGREGATOR**.

| Attribute | Type Code | Purpose |
|---|---:|---|
| AGGREGATOR | 7 | Identifies the aggregating ASN and router ID |
| AS4_AGGREGATOR | 18 | Preserves the aggregating 4-byte ASN for legacy interoperability |

AS4_AGGREGATOR is **optional transitive**.

For example, assume AS 65546 aggregates routes using router ID `1.1.1.1`.

When advertising to a legacy BGP peer:

```text
AGGREGATOR:
  ASN:       23456
  Router ID: 1.1.1.1

AS4_AGGREGATOR:
  ASN:       65546
  Router ID: 1.1.1.1
```

The legacy AGGREGATOR attribute uses AS_TRANS.

The AS4_AGGREGATOR attribute preserves the actual ASN.

When both peers support 4-byte ASNs, the standard AGGREGATOR attribute can carry the 4-byte ASN directly.

**AS4_PATH and AS4_AGGREGATOR are compatibility attributes, not replacements for AS_PATH and AGGREGATOR between modern peers.**

---

## 4-Byte ASNs and AS-Path Regular Expressions

Cisco IOS XE supports matching AS_PATH values using regular expressions.

However, **the active ASN display format also affects the representation used for AS-path regular expression matching**.

### Default asplain Format

Suppose the BGP path contains AS 65546.

With the default asplain notation:

```text
R1# show ip bgp regexp ^65546$
```

This matches paths consisting of exactly AS 65546.

### With `bgp asnotation dot`

After configuring asdot notation and applying the change, the same ASN appears as `1.10`.

The regular expression must use that format:

```text
R1# show ip bgp regexp ^1\.10$
```

Notice the backslash:

```text
\.
```

The period (`.`) is a regular expression metacharacter that matches any single character.

Escaping the period ensures that it matches a literal dot.

### Important Difference

| ASN Notation | Correct Exact AS_PATH Match |
|---|---|
| asplain | `^65546$` |
| asdot | `^1\.10$` |

For example, with `bgp asnotation dot` enabled:

```text
R1# show ip bgp regexp ^65546$
```

This does not match the route when its path is represented as `1.10`.

Use:

```text
R1# show ip bgp regexp ^1\.10$
```

**Changing ASN notation can affect existing AS-path filtering policies that use regular expressions.**

Review AS-path ACLs and route maps before changing the notation on a production router.

See [AS Path Manipulations](../as-path/overview.md) for additional AS_PATH information.

---

## Reserved and Private 4-Byte ASNs

The 4-byte ASN space includes reserved ranges.

Some important ranges and values are:

| ASN or Range | Purpose |
|---|---|
| 0 | Reserved |
| 23456 | AS_TRANS |
| 64496-64511 | 2-byte documentation ASNs |
| 64512-65534 | 2-byte private ASNs |
| 65535 | Reserved |
| 65536-65551 | 4-byte documentation ASNs |
| 4200000000-4294967294 | 4-byte private ASNs |
| 4294967295 | Reserved |

For example, `4200000001` is a private 4-byte ASN.

Private ASN handling, including removal from outbound AS_PATH attributes, is covered in [Private AS Numbers](private-asn.md).

### 4-Byte ASNs and Communities

Traditional standard BGP communities use a 32-bit value, commonly represented as two 16-bit fields:

```text
ASN:Value
```

The ASN portion of this traditional representation cannot contain a full 4-byte ASN.

For 4-byte-AS-specific information, BGP supports other encodings, including 4-byte-AS-specific extended communities and Large Communities.

Cisco IOS XE also supports 4-byte ASN formats in route distinguishers and route targets for VPN deployments.

---

## Verification

### Display Local and Neighbor ASNs

```text
R1# show ip bgp summary
```

Verify:

- Local ASN.
- Remote neighbor ASN.
- Session state.

### Display BGP Neighbor Capabilities

```text
R1# show ip bgp neighbors 10.0.12.2
```

Look for the 4-byte ASN capability in the neighbor capability information.

The detailed output also identifies the remote ASN and negotiated BGP session parameters.

### Display AS_PATH Information

```text
R1# show ip bgp
R1# show ip bgp 192.0.2.0
```

These commands display BGP path information, including AS_PATH values.

With asplain notation:

```text
AS_PATH: 65547
```

With asdot notation:

```text
AS_PATH: 1.11
```

### Verify ASN Notation

```text
R1# show running-config | section router bgp
R1# show ip bgp summary
```

Check whether the BGP process contains:

```text
bgp asnotation dot
```

### Test AS-Path Regular Expressions

```text
R1# show ip bgp regexp ^65546$
```

For asdot notation:

```text
R1# show ip bgp regexp ^1\.10$
```

---

## Troubleshooting

| Symptom | Possible Cause |
|---|---|
| BGP neighbor rejected with Bad Peer AS | Incorrect configured remote ASN |
| 4-byte ASN appears as 23456 | Legacy compatibility encoding or unsupported 4-byte ASN negotiation |
| AS-path regex no longer matches | ASN display/regex notation changed |
| Unexpected AS_PATH values through legacy peers | AS_TRANS substitution or AS4_PATH reconstruction |
| Aggregator ASN appears as 23456 | Legacy AGGREGATOR encoding |
| Direct peering fails with legacy BGP router | Legacy peer does not support the 4-byte ASN capability |

When troubleshooting, first verify that the two neighbors support the necessary ASN capabilities.

Then examine the BGP OPEN negotiation, configured ASN values, and received AS_PATH attributes.

---

## Command Summary

| Command | Purpose |
|---|---|
| `router bgp <4-byte-ASN>` | Configure BGP using a 4-byte ASN |
| `neighbor <IP> remote-as <4-byte-ASN>` | Configure a neighbor using a 4-byte ASN |
| `bgp asnotation dot` | Use asdot for BGP show output and AS-path regex matching |
| `no bgp asnotation dot` | Restore asplain notation |
| `show ip bgp summary` | Display local and neighbor ASNs |
| `show ip bgp neighbors <IP>` | Display neighbor capabilities and ASN information |
| `show ip bgp` | Display BGP routes and AS_PATH values |
| `show ip bgp regexp <REGEX>` | Match BGP routes using AS-path regular expressions |
| `clear ip bgp *` | Hard reset BGP sessions to apply notation changes as documented by Cisco |

---

## Key Points

- **2-byte ASNs** use 16 bits; **4-byte ASNs** use 32 bits.
- The maximum possible 4-byte ASN value is **4294967295**, although this value is reserved.
- **asplain** represents an ASN as a single decimal number.
- **asdot** represents ASNs of 65536 or greater as `High.Low`.
- Convert asdot to asplain using `(High × 65536) + Low`.
- Cisco IOS XE uses **asplain by default**.
- `bgp asnotation dot` changes show output and AS-path regex matching, not the actual ASN.
- The **4-Octet AS Number Capability (Code 65)** advertises support for 4-byte ASNs in BGP OPEN messages.
- **AS_TRANS (23456)** represents non-mappable 4-byte ASNs in legacy 2-byte fields.
- **AS4_PATH (Type 17)** preserves 4-byte path information across legacy BGP speakers.
- **AS4_AGGREGATOR (Type 18)** preserves 4-byte aggregator information.
- When both peers support 4-byte ASNs, the ordinary AS_PATH and AGGREGATOR attributes encode ASNs using 4 bytes.
- In asdot notation, escape the period when matching an ASN using AS-path regular expressions.

---

## Cisco Documentation

- [Cisco IOS XE 17.x: BGP Support for 4-byte ASN](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-4byte-asn.html)
- [RFC 6793: BGP Support for Four-Octet Autonomous System Number Space](https://www.rfc-editor.org/rfc/rfc6793.html)
- [RFC 5396: Textual Representation of Autonomous System Numbers](https://www.rfc-editor.org/rfc/rfc5396.html)
- [IANA: Autonomous System Numbers](https://www.iana.org/assignments/as-numbers/as-numbers.xhtml)
