# 4.4.d Marking DSCP Values in IPv4 and IPv6 Headers

**Differentiated Services Code Point (DSCP) marking** assigns a value to the DSCP field in an IPv4 or IPv6 packet to indicate the forwarding treatment the packet should receive.

DSCP marking is a fundamental part of the **Differentiated Services (DiffServ)** architecture.

Rather than repeatedly classifying traffic using IP addresses, transport-layer ports, or application signatures, network devices can use the existing DSCP value to determine how packets should be handled.

For example:
- **Voice traffic** might be marked EF (46) for low-latency forwarding.
- **Business applications** might be marked AF31 (26) for assured forwarding.
- **Ordinary traffic** might be marked CS0 (0) for best-effort forwarding.
- **Network control traffic** might be marked CS6 (48).

Marking does not independently prioritize packets. Network devices must implement QoS policies that interpret the markings and provide the appropriate forwarding treatment.

---

## DSCP in IPv4 and IPv6

DSCP is a **6-bit field** in both IPv4 and IPv6, supporting values from **0 through 63**.

Although DSCP has the same meaning in both protocols, it is carried in differently named header fields.

| Protocol | Header field | Size |
|---|---|---|
| **IPv4** | Differentiated Services (formerly Type of Service) | 8 bits |
| **IPv6** | Traffic Class | 8 bits |

Both fields have the same structure:

```text
 Bit:   0   1   2   3   4   5   6   7
      +---+---+---+---+---+---+---+---+
      |           DSCP        |  ECN  |
      +---+---+---+---+---+---+---+---+
               6 bits          2 bits
```

The eight bits are divided into:
- **DSCP (6 bits):** Identifies the intended forwarding behavior.
- **Explicit Congestion Notification (ECN, 2 bits):** Allows ECN-capable network devices to indicate congestion without necessarily dropping packets.

The DSCP and ECN fields are logically independent.

For example, setting DSCP to EF (46) does not require changing the packet's ECN bits.

See [IPv4 and IPv6 DS Fields](ds-field.md) for details about their placement and interpretation in both IP headers.

---

## DSCP Values

The six DSCP bits allow 64 possible codepoints.

However, several standardized DSCP values are particularly important for enterprise QoS.

| DSCP | Decimal | Binary | Typical use |
|---|---|---|---|
| **CS0 / BE** | 0 | `000000` | Best-effort traffic |
| **CS1** | 8 | `001000` | Lower-priority traffic |
| **AF21** | 18 | `010010` | Assured forwarding |
| **AF31** | 26 | `011010` | Assured forwarding |
| **AF32** | 28 | `011100` | AF3, medium drop precedence |
| **AF33** | 30 | `011110` | AF3, high drop precedence |
| **AF41** | 34 | `100010` | Assured forwarding |
| **CS5** | 40 | `101000` | Commonly used for signaling |
| **EF** | 46 | `101110` | Voice bearer traffic |
| **CS6** | 48 | `110000` | Network control traffic |

The DSCP values identify codepoints associated with forwarding behaviors called **Per-Hop Behaviors (PHBs)**.

The principal PHBs include:
- **Best Effort (BE):** Ordinary forwarding without preferential treatment.
- **Expedited Forwarding (EF):** Low-delay, low-jitter forwarding.
- **Assured Forwarding (AF):** Four forwarding classes, each with three drop precedence levels.
- **Class Selector (CS):** Codepoints compatible with the older IP Precedence model.

For example, AF31 and AF33 belong to the same AF forwarding class but have different drop precedence levels.

AF33 is intended to have a higher drop probability than AF31 under congestion, assuming the network implements the appropriate congestion-avoidance behavior.

See [DSCP Values and Codepoint Encoding](dscp-values.md) for the complete codepoint table and encoding rules.

---

## Classification and Marking

**Classification** and **marking** are related but separate operations.

| Operation | Function |
|---|---|
| **Classification** | Identifies traffic using packet characteristics |
| **Marking** | Writes a QoS value into the packet header |
| **Remarking** | Replaces an existing QoS marking with a different value |

For example, a router might identify traffic going to a business application server using its destination IP address and TCP port.

After identifying the traffic, it marks the packets with DSCP AF31.

```text
        Incoming Packet
               |
               v
      +------------------+
      |  Classification  |
      |                  |
      |  Destination IP  |
      |  TCP/UDP Port    |
      +------------------+
               |
               v
       Business Traffic
               |
               v
      +------------------+
      |     Marking      |
      |                  |
      |  Set DSCP AF31   |
      +------------------+
               |
               v
        Forward Packet
         (DSCP = 26)
```

Downstream routers can then identify the traffic using its DSCP marking rather than performing the original detailed classification again.

**Marking does not create a bandwidth guarantee or priority queue.** Those forwarding behaviors must be configured separately.

---

## Marking and Remarking at the Trust Boundary

DSCP marking is commonly performed near the **network edge**, where traffic first enters a trusted QoS domain.

Packets can arrive with:
- **No special marking:** Typically DSCP 0.
- **Valid markings:** Assigned by trusted devices or applications.
- **Untrusted markings:** Values that must be validated or replaced.

A QoS policy can perform three operations:

| Operation | Result |
|---|---|
| **Mark** | Assign a DSCP value to traffic |
| **Remark** | Replace an existing DSCP value |
| **Preserve** | Forward the packet without changing its DSCP value |

For example, a PC might mark its backup traffic with DSCP EF to obtain preferential forwarding treatment.

An access switch can remark that traffic to CS0 before it enters the network.

Likewise, traffic from a trusted IP phone might already be marked EF and can retain that marking.

The **QoS trust boundary** determines where the network accepts markings from connected devices.

See [QoS Trust Boundaries](../classification/trust-boundaries.md) for trust boundary configuration and behavior.

---

## DSCP Marking with MQC

Cisco IOS XE uses the **Modular QoS Command-Line Interface (MQC)** to classify and mark traffic.

The basic configuration process is:

1. Create a `class-map` to identify the traffic.
2. Create a `policy-map` and configure a DSCP marking action.
3. Apply the policy using `service-policy`.

### Example: Marking Business Traffic

Suppose R1 receives business application traffic destined for `192.0.2.10` on TCP port 443.

The objective is to mark the traffic with DSCP AF31.

**Step 1: Create an ACL to identify the traffic.**

```text
R1(config)# ip access-list extended BUSINESS-ACL
R1(config-ext-nacl)# permit tcp any host 192.0.2.10 eq 443
```

**Step 2: Create a class map.**

```text
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match access-group name BUSINESS-ACL
```

**Step 3: Create the marking policy.**

```text
R1(config)# policy-map INGRESS-MARKING
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# set dscp af31
```

The `set dscp af31` command changes the DSCP field of matching packets to decimal 26.

The command also accepts numerical values:

```text
R1(config-pmap-c)# set dscp 26
```

Both commands produce the same marking.

**Step 4: Apply the policy inbound.**

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input INGRESS-MARKING
```

Packets matching the business application class are marked AF31 as they enter R1.

Packets that do not match the class retain their existing DSCP values because no marking action has been configured for them.

### Remarking Traffic to Best Effort

To remark all traffic not matching an earlier class to DSCP 0, configure `class-default`:

```text
R1(config)# policy-map INGRESS-MARKING
R1(config-pmap)# class class-default
R1(config-pmap-c)# set dscp default
```

This is useful on an untrusted ingress interface where unmatched traffic should not retain arbitrary DSCP markings.

**The `class-default` class does not automatically reset DSCP values to 0.** An explicit marking action is necessary.

See [DSCP Marking and Remarking with MQC](mqc-marking.md) for additional configurations, including marking based on existing DSCP values.

---

## IPv4 and IPv6 Marking with MQC

Cisco IOS XE supports DSCP marking for both IPv4 and IPv6 using the same MQC commands.

For example:

```text
R1(config-pmap-c)# set dscp ef
```

This command marks matching IPv4 or IPv6 packets with DSCP EF (46).

The specific IP version affected depends on which packets match the class map.

Cisco IOS XE also supports:

```text
R1(config-pmap-c)# set ip dscp ef
```

**`set dscp` and `set ip dscp` both support IPv4 and IPv6 marking.**

The distinction is more important when configuring classification:

| Command | Matching behavior |
|---|---|
| `match dscp ef` | Matches IPv4 and IPv6 packets marked EF |
| `match ip dscp ef` | Matches only IPv4 packets marked EF |

To explicitly classify IPv6 traffic, a class map can use the `match protocol ipv6` command.

For example:

```text
R1(config)# class-map match-all IPV6-EF
R1(config-cmap)# match protocol ipv6
R1(config-cmap)# match dscp ef
```

This class matches IPv6 packets marked EF, but not IPv4 packets.

The same `set dscp` commands can then be used to remark the matching packets.

---

## CoS, IP Precedence, and DSCP

DSCP is not the only QoS marking mechanism used in enterprise networks.

| Marking | Bits | Range | Location |
|---|---|---|---|
| **CoS** | 3 | 0–7 | IEEE 802.1Q VLAN tag |
| **IP Precedence** | 3 | 0–7 | First three bits of the DSCP field |
| **DSCP** | 6 | 0–63 | IPv4/IPv6 header |
| **MPLS TC** | 3 | 0–7 | MPLS label stack entry |

**Class of Service (CoS)** operates at Layer 2 and is carried in the Priority Code Point (PCP) field of an IEEE 802.1Q tag.

**IP Precedence** is an older Layer 3 QoS mechanism that uses the first three bits of what is now the DSCP field.

DSCP extends this approach by using six bits instead of three.

For example:
- IP Precedence 5 corresponds to the first three DSCP bits being `101`.
- CS5 is DSCP 40 (`101000`).
- EF is DSCP 46 (`101110`), whose first three bits are also `101`.

Consequently, multiple DSCP values can correspond to the same IP Precedence value.

When traffic crosses between networks that use different marking mechanisms, devices may need to map values between CoS, IP Precedence, DSCP, and MPLS TC.

**CoS and DSCP are separate markings, and there is no universal automatic one-to-one conversion between them.** The mapping depends on the configured QoS policy and device behavior.

See [CoS, IP Precedence, and DSCP Mapping](cos-dscp-mapping.md) for mapping rules and configuration examples.

---

## Verification

The primary command for verifying MQC marking policies is:

```text
R1# show policy-map interface GigabitEthernet1
```

This displays the policy attached to the interface, including:
- Traffic classes and matching criteria.
- Packets and bytes matched by each class.
- Configured marking actions.
- Available per-class statistics.

Other useful commands include:

| Command | Purpose |
|---|---|
| `show policy-map` | Display configured policy maps and their actions |
| `show policy-map INGRESS-MARKING` | Display a specific policy map |
| `show policy-map interface GigabitEthernet1` | Verify the applied policy and matching statistics |
| `show running-config interface GigabitEthernet1` | Verify the service policy attachment |
| `show class-map` | Display configured class maps |

To verify the actual DSCP value in transmitted packets, use packet capture on the appropriate interface.

For example, Wireshark can display:
- **IPv4:** Differentiated Services Field.
- **IPv6:** Traffic Class field.
- **DSCP:** The six-bit codepoint carried in the field.

Checking the packet header is particularly useful when verifying whether markings are preserved or modified across multiple network devices.

---

## Key Points

- DSCP uses **6 bits** in the IPv4 and IPv6 headers, supporting values from 0 to 63.
- DSCP occupies the upper six bits of an 8-bit field; ECN occupies the lower two bits.
- **Classification** identifies traffic, while **marking** changes its QoS codepoint.
- Marking should normally occur near the network edge, with existing markings preserved where they are trusted.
- `set dscp` can mark both IPv4 and IPv6 traffic.
- `class-default` preserves existing DSCP values unless a marking action changes them.
- DSCP markings indicate intended forwarding treatment but do not automatically provide prioritization, bandwidth guarantees, or congestion avoidance.
- `show policy-map interface` is the primary IOS XE command for verifying marking policies.

---

## References

- [RFC 2474: Definition of the Differentiated Services Field (DS Field) in the IPv4 and IPv6 Headers](https://www.rfc-editor.org/rfc/rfc2474)
- [RFC 3168: The Addition of Explicit Congestion Notification (ECN) to IP](https://www.rfc-editor.org/rfc/rfc3168)
- [Cisco IOS XE 17.x: QoS Packet Marking](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-mrkg.html)
- [Cisco IOS XE 17.x: Applying QoS Features Using MQC](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-mqc-0.html)