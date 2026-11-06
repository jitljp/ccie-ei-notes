# Traffic Classification Methods

**Traffic classification** is the process of identifying packets and assigning them to traffic classes so that appropriate Quality of Service (QoS) policies can be applied.

Classification can use information from:
- **Layer 2:** CoS, VLAN IDs, MAC addresses.
- **Layer 3:** IP addresses, DSCP, IP Precedence.
- **Layer 4:** TCP/UDP port numbers.
- **Layer 7:** Application identification using NBAR.
- **Device-specific information:** Input interfaces, internal QoS groups, and other locally assigned QoS identifiers.

Classification is performed throughout a QoS-enabled network.

However, the classification methods used at different points in the network can differ:
- **Network edge:** Detailed classification is commonly used to identify applications and assign appropriate QoS markings.
- **Network core:** Simpler classification based on existing markings, such as DSCP, is commonly used to select forwarding treatments.

---

## Behavior Aggregate (BA) vs. Multifield (MF) Classification

The DiffServ architecture defines two fundamental classification methods.

| Method | Description |
|---|---|
| **Behavior Aggregate (BA)** | Classifies packets based only on their DSCP values |
| **Multifield (MF)** | Classifies packets using one or more packet header fields, such as IP addresses, protocols, ports, and DSCP |

### Behavior Aggregate (BA) Classification

A **Behavior Aggregate (BA) classifier** examines a packet's DSCP value to determine which traffic class it belongs to.

For example, a router might classify packets into:
- **EF:** Voice traffic.
- **AF31:** Business-critical applications.
- **CS1:** Scavenger traffic.
- **CS0:** Best-effort traffic.

Cisco IOS XE uses the `match dscp` command for DSCP-based classification.

```text
R1(config)# class-map match-all VOICE
R1(config-cmap)# match dscp ef
```

This class map identifies packets marked with DSCP EF (46).

BA classification is commonly used within the network core because it does not require the device to examine multiple packet fields to identify an application.

### Multifield (MF) Classification

A **Multifield (MF) classifier** examines packet header fields other than, or in addition to, DSCP.

For example, a classifier could examine:
- Source IP address.
- Destination IP address.
- IP protocol.
- TCP source port.
- TCP destination port.
- Existing DSCP value.

MF classification can identify packets belonging to a particular application or traffic flow.

Consider a business application with the following characteristics:

| Field | Value |
|---|---|
| Source subnet | `10.10.10.0/24` |
| Destination | `192.0.2.10` |
| Protocol | TCP |
| Destination port | 443 |

An extended ACL can match this traffic:

```text
R1(config)# ip access-list extended BUSINESS-ACL
R1(config-ext-nacl)# permit tcp 10.10.10.0 0.0.0.255 host 192.0.2.10 eq 443
R1(config-ext-nacl)# exit
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match access-group name BUSINESS-ACL
```

This identifies packets that match the configured addresses, protocol, and destination port.

The traffic could then be marked AF31 using an ingress policy.

Downstream routers could classify the packets based only on their DSCP values.

```text
     Network Edge                     Network Core

+----------------------+      +----------------------+
| MF Classification    |      | BA Classification    |
|                      |      |                      |
| Source IP            |      |                      |
| Destination IP       |      |     DSCP AF31        |
| TCP destination port |      |                      |
+----------------------+      +----------------------+
            |                            ^
            v                            |
      Mark DSCP AF31 --------------------+
```

**MF classification is commonly used to establish the initial QoS classification, while BA classification uses the resulting DSCP markings throughout the network.**

---

## Layer 2 Classification

Layer 2 classification uses information from the Ethernet header or associated VLAN tag.

Common methods include:
- IEEE 802.1Q Class of Service (CoS).
- VLAN ID.
- Source and destination MAC addresses.

### Class of Service (CoS)

**Class of Service (CoS)** uses the 3-bit **Priority Code Point (PCP)** field in the IEEE 802.1Q tag.

Because the field is 3 bits long, it supports **8 values, from 0 through 7**.

```text
             IEEE 802.1Q Tag

+---------+-----+----------------+
|   PCP   | DEI |     VLAN ID    |
+---------+-----+----------------+
  3 bits   1 bit     12 bits
     ^
     |
    CoS
```

CoS is commonly used to classify traffic on Ethernet trunks and other links carrying tagged frames.

Some common enterprise markings are:

| CoS | Typical traffic |
|---|---|
| 0 | Best Effort |
| 1 | Scavenger |
| 3 | Call signaling |
| 4 | Real-time interactive traffic |
| 5 | Voice |
| 6 | Network control |

These are common QoS design conventions, not forwarding behaviors automatically guaranteed by the CoS values.

#### CoS Classification with MQC

The `match cos` command classifies packets based on their CoS markings.

For example:

```text
SW1(config)# class-map match-all VOICE
SW1(config-cmap)# match cos 5
```

This matches traffic carrying CoS 5.

Multiple values can also be specified:

```text
SW1(config)# class-map match-all REALTIME
SW1(config-cmap)# match cos 4 5
```

This matches traffic with CoS 4 **OR** CoS 5.

**CoS is a Layer 2 marking**, so it is not preserved automatically across routed IP hops.

In addition, ordinary untagged Ethernet frames do not carry a CoS field.

Devices can map CoS values to DSCP values, allowing traffic to receive appropriate Layer 3 QoS treatment.

See [CoS, IP Precedence, and DSCP Mapping](../marking/cos-dscp-mapping.md).

### VLAN-Based Classification

Traffic can also be classified based on the **VLAN ID**.

This is useful when different VLANs carry different types of traffic.

For example:

| VLAN | Traffic |
|---|---|
| 10 | User data |
| 20 | IP phones |
| 30 | Video conferencing |

On supported Cisco IOS XE switches, the `match vlan` command identifies traffic belonging to a particular VLAN.

```text
SW1(config)# class-map match-all VOICE-VLAN
SW1(config-cmap)# match vlan 20
```

This classifies traffic based on VLAN 20, regardless of its IP address or DSCP value.

VLAN classification is useful when the VLAN itself identifies a traffic category.

However, **VLAN membership does not necessarily identify the application**.

For example, a PC in VLAN 10 could generate voice, video, web, and backup traffic.

VLAN classification alone would not distinguish between those applications.

### MAC Address Classification

Some Cisco platforms support classifying packets by their source or destination MAC addresses.

Relevant MQC commands include:

```text
match source-address mac
match destination-address mac
```

MAC-based classification can identify traffic associated with particular Layer 2 devices.

However, MAC addresses are local to a Layer 2 network.

When a router forwards a packet onto another Ethernet link, it constructs a new Ethernet header with different source and destination MAC addresses.

Therefore, MAC-based classification is primarily useful near the network edge rather than for end-to-end application identification.

---

## Layer 3 Classification

Layer 3 classification uses fields in the IPv4 or IPv6 header.

The most important classification criteria are:
- DSCP.
- IP Precedence.
- Source and destination IP addresses.

### DSCP Classification

**Differentiated Services Code Point (DSCP)** is a 6-bit field in the IPv4 and IPv6 headers.

Because it contains 6 bits, DSCP supports **64 values, from 0 through 63**.

Each DSCP value can be mapped to an appropriate **Per-Hop Behavior (PHB)**.

For example:

| DSCP | Decimal | Typical use |
|---|---:|---|
| CS0 | 0 | Best Effort |
| CS1 | 8 | Scavenger |
| AF21 | 18 | Transactional data |
| AF31 | 26 | Business applications |
| CS3 | 24 | Call signaling |
| EF | 46 | Voice |
| CS6 | 48 | Network control |

DSCP is a common classification method in the network core because it allows packets to be grouped into forwarding classes without inspecting application-specific fields.

#### Matching Multiple DSCP Values

The `match dscp` command accepts multiple values.

For example:

```text
R1(config)# class-map match-all AF3
R1(config-cmap)# match dscp af31 af32 af33
```

This matches packets marked AF31, AF32, **or** AF33.

Although the class map is configured with `match-all`, the DSCP values specified within a single `match dscp` statement are alternatives.

The packets belong to the same AF forwarding class but have different drop precedence levels.

### IP Precedence Classification

**IP Precedence** uses the three most significant bits of the IPv4 ToS field, or equivalently the three most significant bits of the modern DSCP field.

IP Precedence supports **8 values, from 0 through 7**.

```text
             IPv4 DS Field

+-------------+-------------+-------+
| Precedence  |             |       |
|   3 bits    |   3 bits    |  ECN  |
+-------------+-------------+-------+
|<------ DSCP (6 bits) ---->|2 bits |
```

The `match precedence` or `match ip precedence` command can classify traffic based on these three bits.

For example:

```text
R1(config)# class-map match-all HIGH-PREC
R1(config-cmap)# match ip precedence 5
```

This matches packets whose three most significant DSCP bits are `101`.

#### IP Precedence vs. DSCP Matching

An important difference is that **IP Precedence classification is less specific than DSCP classification**.

For example:

| DSCP | Decimal | Binary | IP Precedence |
|---|---:|---|---:|
| CS5 | 40 | `101000` | 5 |
| 42 | 42 | `101010` | 5 |
| 44 | 44 | `101100` | 5 |
| EF | 46 | `101110` | 5 |

All four values have the same three most significant bits: `101`.

Therefore:

```text
match ip precedence 5
```

Matches packets with any of these DSCP values.

However:

```text
match dscp ef
```

Matches only packets with DSCP 46.

More generally, each IP Precedence value corresponds to **eight possible DSCP values**.

DSCP provides more granular classification than IP Precedence.

IP Precedence-based classification also applies to IPv6 traffic through the corresponding high-order bits of the Traffic Class field.

---

## IP Address and Protocol Classification Using ACLs

**Access Control Lists (ACLs)** provide flexible traffic classification using Layer 3 and Layer 4 information.

ACLs can match:
- Source IP addresses.
- Destination IP addresses.
- IP protocols.
- TCP/UDP source ports.
- TCP/UDP destination ports.
- Other supported packet characteristics.

ACL-based classification is particularly useful when packets are not already marked with trustworthy DSCP values.

### Standard ACL Classification

A standard IPv4 ACL matches packets based on their **source IPv4 address**.

For example:

```text
R1(config)# ip access-list standard USERS
R1(config-std-nacl)# permit 10.10.10.0 0.0.0.255
R1(config-std-nacl)# exit
R1(config)# class-map match-all USER-TRAFFIC
R1(config-cmap)# match access-group name USERS
```

This classifies packets sourced from `10.10.10.0/24`.

The destination address and Layer 4 port numbers do not affect the match.

### Extended ACL Classification

Extended ACLs can match multiple packet fields.

For example:

```text
R1(config)# ip access-list extended BUSINESS-APP
R1(config-ext-nacl)# permit tcp 10.10.10.0 0.0.0.255 host 192.0.2.10 eq 443
R1(config-ext-nacl)# exit
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match access-group name BUSINESS-APP
```

This matches TCP packets:
- Sourced from `10.10.10.0/24`.
- Destined for `192.0.2.10`.
- Using destination port 443.

Extended ACLs can distinguish applications more precisely than standard ACLs.

However, TCP/UDP port numbers do not always uniquely identify an application.

For example, many different applications use TCP port 443.

### IPv6 ACL Classification

IPv6 ACLs can also be used for QoS classification.

For example:

```text
R1(config)# ipv6 access-list IPV6-BUSINESS
R1(config-ipv6-acl)# permit tcp 2001:DB8:10::/64 host 2001:DB8:20::10 eq 443
R1(config-ipv6-acl)# exit
R1(config)# class-map match-all IPV6-BUSINESS
R1(config-cmap)# match access-group name IPV6-BUSINESS
```

This identifies TCP traffic:
- Sourced from `2001:DB8:10::/64`.
- Destined for `2001:DB8:20::10`.
- Using destination port 443.

IPv4 and IPv6 classification can therefore use equivalent packet characteristics, even though the ACL syntax differs.

### ACL Permit and Deny Behavior

When an ACL is used for QoS classification, its `permit` and `deny` statements have different meanings than when the ACL is applied as a security filter.

| ACL result | QoS classification behavior |
|---|---|
| **Permit** | Packet matches the ACL-based classification criterion |
| **Deny** | Packet does not match the ACL-based classification criterion |
| **Implicit deny** | Packet does not match the ACL-based classification criterion |

For example:

```text
R1(config)# ip access-list extended WEB-ACL
R1(config-ext-nacl)# deny tcp 10.10.10.0 0.0.0.255 any eq 443
R1(config-ext-nacl)# permit tcp any any eq 443
R1(config-ext-nacl)# exit
R1(config)# class-map match-all WEB
R1(config-cmap)# match access-group name WEB-ACL
```

This class map matches TCP packets destined for port 443 **except** packets sourced from `10.10.10.0/24`.

The ACL's deny statement does **not** cause those packets to be dropped.

Instead, the packets fail to match this particular class and can be evaluated against other classes in the QoS policy.

**An ACL used for QoS classification identifies traffic. It does not directly permit or deny forwarding.**

---

## Layer 4 Classification

Layer 4 classification uses fields in transport-layer headers, especially TCP and UDP port numbers.

For example:

| Application or protocol | Common port |
|---|---|
| SSH | TCP 22 |
| DNS | TCP/UDP 53 |
| HTTPS | TCP 443 |
| QUIC / HTTP/3 | UDP 443 |
| NTP | UDP 123 |
| SIP | TCP/UDP 5060 |

TCP/UDP port classification is typically implemented using extended ACLs.

For example:

```text
R1(config)# ip access-list extended DNS-TRAFFIC
R1(config-ext-nacl)# permit udp any any eq 53
R1(config-ext-nacl)# permit tcp any any eq 53
R1(config-ext-nacl)# exit
R1(config)# class-map match-all DNS
R1(config-cmap)# match access-group name DNS-TRAFFIC
```

This classifies TCP and UDP packets destined for port 53.

### Limitations of Port-Based Classification

Port-based classification has several limitations:

1. Applications can use dynamically negotiated ports.
2. Different applications can share the same port.
3. An application can use nonstandard ports.
4. Some applications use encryption or tunneling, making identification more difficult.
5. A packet's port number does not necessarily reveal the application generating it.

For example, identifying all UDP 443 traffic as QUIC would be an assumption rather than reliable application identification.

For more accurate application classification, Cisco supports **Network Based Application Recognition (NBAR)**.

---

## Application-Based Classification with NBAR

**Network Based Application Recognition (NBAR)** allows Cisco devices to classify traffic based on recognized applications and protocols.

Unlike ordinary ACL classification, NBAR is not limited to matching static IP addresses and TCP/UDP port numbers.

Depending on the recognized application, it can examine protocol characteristics, application signatures, and other supported fields.

This allows NBAR to identify applications that:
- Use dynamically assigned ports.
- Share common port numbers.
- Cannot be reliably identified using basic header fields.

NBAR integrates with MQC through the `match protocol` command.

For example:

```text
R1(config)# class-map match-all HTTP
R1(config-cmap)# match protocol http
```

This class map uses NBAR to identify HTTP traffic.

NBAR can then be used as part of a QoS policy to classify, mark, or police the identified application traffic.

The details of application recognition, protocol discovery, NBAR2, and protocol packs are covered in [4.4.c Network Based Application Recognition (NBAR)](../nbar/overview.md).

---

## MPLS Traffic Class Classification

When traffic is forwarded through an MPLS network, QoS classification can use the **Traffic Class (TC)** field in the MPLS label header.

The TC field was formerly called the **Experimental (EXP)** field.

It is 3 bits long, supporting **8 values, from 0 through 7**.

```text
             MPLS Label Stack Entry

+------------------------+-----+---+--------+
|         Label          | TC  | S |  TTL   |
+------------------------+-----+---+--------+
         20 bits            3    1      8
```

MPLS routers can use the TC value to determine the forwarding treatment applied to labeled packets.

For example, a provider might map:
- Voice traffic to TC 5.
- Business traffic to TC 3.
- Best-effort traffic to TC 0.

### MPLS TC Classification with MQC

Cisco IOS XE supports matching the MPLS TC field using the `match mpls experimental` command.

For example:

```text
R1(config)# class-map match-all MPLS-VOICE
R1(config-cmap)# match mpls experimental topmost 5
```

This matches packets whose **topmost MPLS label** has TC 5.

The `topmost` keyword is important because MPLS packets can carry multiple labels.

For example:

```text
+-------------------------+
| Transport Label: TC 5   | <- Topmost
+-------------------------+
| VPN Label: TC 3         |
+-------------------------+
| Customer IP Packet      |
+-------------------------+
```

A match on the topmost label examines TC 5, not TC 3.

The relationship between IP DSCP and MPLS TC depends on the configured MPLS QoS model and label-marking policies.

Because MPLS TC contains only 3 bits, it cannot directly represent all 64 possible DSCP values.

---

## Classification Using Device-Specific Information

Not all classification methods examine fields carried in packet headers.

Cisco devices can also classify traffic based on locally available information.

Examples include:
- Input interface.
- Internal QoS group.
- Internal traffic class.

### Input Interface Classification

Some Cisco IOS XE platforms support the `match input-interface` command.

For example:

```text
R1(config)# class-map match-all CUSTOMER-A
R1(config-cmap)# match input-interface GigabitEthernet1
```

This classifies traffic based on the interface through which it entered the router.

Input-interface classification can be useful when different interfaces connect to different customers or network segments.

For example, an ISP might classify traffic arriving from a particular customer-facing interface without examining its IP addresses.

### Internal QoS Groups

A **QoS group** is an internal identifier used by a Cisco device to classify traffic.

Unlike DSCP or CoS, a QoS group is **not carried in a packet header**.

It exists only within the device.

For example, an ingress QoS policy might assign packets to QoS group 5:

```text
R1(config)# class-map match-all VOICE
R1(config-cmap)# match dscp ef
R1(config-cmap)# exit
R1(config)# policy-map INGRESS
R1(config-pmap)# class VOICE
R1(config-pmap-c)# set qos-group 5
```

Another policy on the same device could classify those packets using their internal QoS group:

```text
R1(config)# class-map match-all INTERNAL-VOICE
R1(config-cmap)# match qos-group 5
```

The conceptual forwarding process is:

```text
 Incoming Packet
        |
        v
   Match DSCP EF
        |
        v
 Set QoS Group 5
        |
        v
 Internal Forwarding
        |
        v
 Match QoS Group 5
        |
        v
 Apply QoS Treatment
```

This is useful when the device needs to preserve an internal traffic classification independently of the packet's external markings.

Because QoS groups are internal, **their values are not transmitted to the next router or switch**.

### Internal Traffic Classes

Some Catalyst switches also use an internal **traffic-class** value to select the appropriate hardware forwarding queue.

For example:

```text
SW1(config)# class-map match-all VOICE
SW1(config-cmap)# match dscp ef
SW1(config-cmap)# exit
SW1(config)# policy-map INGRESS
SW1(config-pmap)# class VOICE
SW1(config-pmap-c)# set traffic-class 5
```

The internal traffic class can then influence output queue selection and scheduling.

The exact mapping between DSCP, internal traffic classes, and hardware queues depends on the switch's QoS architecture.

Like QoS groups, internal traffic classes are **device-local identifiers**, not fields transmitted in ordinary IP or Ethernet headers.

---

## Other Classification Criteria

Cisco IOS XE also supports additional classification criteria on certain platforms.

Examples include:

| Classification criterion | MQC command | Purpose |
|---|---|---|
| Packet length | `match packet length` | Match packets based on their Layer 3 length |
| RTP port range | `match ip rtp` | Match UDP traffic using specified RTP port ranges |
| Inverted match | `match not` | Exclude packets matching a particular criterion |
| Nested class | `match class-map` | Use another class map as a match criterion |

### Packet Length Classification

Packet length classification can distinguish between packets of different sizes.

For example, a QoS policy might identify small packets separately from large data-transfer packets.

However, packet size alone does not reliably identify an application.

### RTP Port Classification

The `match ip rtp` command allows classification based on UDP port numbers commonly used for RTP traffic.

This is useful for voice and multimedia applications using known RTP port ranges.

However, an RTP port match does not necessarily verify that the traffic actually contains RTP.

### Negative Matching

The `match not` command allows a class map to exclude a particular matching criterion.

For example, a class could be defined to match traffic that does not belong to a specified QoS group.

These more advanced class-map operations are covered in [MQC Class Maps and Match Criteria](class-maps.md).

---

## Classification of Encapsulated Traffic

Encapsulation can affect which packet fields are available for classification.

For example, a GRE packet contains both an inner IP header and an outer IP header.

```text
+-----------------------+
|     Outer IP Header   |
+-----------------------+
|      GRE Header       |
+-----------------------+
|     Inner IP Header   |
+-----------------------+
|    TCP/UDP Header     |
+-----------------------+
|     Application Data  |
+-----------------------+
```

If a router applies a QoS policy to the physical interface carrying the encapsulated traffic, it may only be able to classify packets based on fields in the outer headers.

For example, every packet within a particular GRE tunnel could have the same outer source and destination addresses.

This makes it difficult to distinguish the original applications using the outer headers.

### QoS Pre-Classification

Cisco IOS XE supports **QoS pre-classification** to preserve original packet information for QoS classification after encapsulation.

For example:

```text
R1(config)# interface Tunnel0
R1(config-if)# qos pre-classify
```

This allows a supported QoS policy to classify encapsulated traffic using information from the original IP packet.

For example, a WAN QoS policy might need to distinguish voice and business traffic based on their original IP addresses or TCP/UDP ports.

Without pre-classification, those original fields may not be available to the physical interface's classification process.

**QoS pre-classification is generally unnecessary when classification can use the outer DSCP marking**, because the appropriate DSCP can be copied or mapped into the outer IP header.

QoS pre-classification does not itself mark packets or allocate bandwidth. It makes the original packet information available for classification.

See [Cisco: Quality of Service Options on GRE Tunnel Interfaces](https://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-policing/10106-qos-tunnel.html).

---

## Comparing Classification Methods

| Method | Information examined | Typical use |
|---|---|---|
| **CoS** | 802.1Q PCP field | Ethernet QoS |
| **VLAN** | VLAN ID | Traffic separation at Layer 2 |
| **MAC address** | Source/destination MAC | Device-based classification |
| **DSCP** | 6-bit IP DSCP field | DiffServ BA classification |
| **IP Precedence** | Three highest-order DSCP bits | Legacy-compatible IP classification |
| **IPv4/IPv6 ACL** | IP addresses, protocols, ports | Detailed MF classification |
| **NBAR** | Application-specific characteristics | Application identification |
| **MPLS TC** | MPLS label TC field | MPLS forwarding treatment |
| **Input interface** | Interface receiving the packet | Customer or segment classification |
| **QoS group** | Internal device identifier | Internal classification |
| **Traffic class** | Internal forwarding class | Hardware queue selection |

### Choosing a Classification Method

The most appropriate classification method depends on where the device operates and what information is available.

**At the network edge:**
- Use ACLs when applications can be identified by addresses and ports.
- Use NBAR when more sophisticated application identification is required.
- Use existing markings from trusted devices.
- Mark packets consistently according to the network's QoS design.

**Within the network core:**
- Use DSCP-based classification for IP traffic.
- Use MPLS TC where appropriate in labeled networks.
- Use internal QoS identifiers when required by the forwarding architecture.

The objective is to avoid unnecessary repeated application classification while ensuring that each network device applies the appropriate QoS treatment.

---

## Verification

The following commands are useful for examining classification configuration and results.

| Command | Purpose |
|---|---|
| `show class-map` | Displays class maps and their matching criteria |
| `show class-map <name>` | Displays a specific class map |
| `show policy-map` | Displays configured QoS policies |
| `show policy-map interface` | Displays applied policies and class statistics |
| `show access-lists` | Displays ACL entries and available match counters |

For example:

```text
R1# show class-map
R1# show class-map VOICE
R1# show policy-map interface GigabitEthernet1
```

When verifying classification, check:
1. The expected classification criteria are configured.
2. The class map is referenced by a policy map.
3. The policy map is attached to the correct interface and direction.
4. Matching traffic increments the expected class counters.
5. Traffic that does not match a class receives the intended default treatment.

**Defining a class map alone does not affect packet forwarding.** The class map must be referenced by an active QoS policy.

---

## Key Points

- **Classification** identifies packets and assigns them to QoS traffic classes.
- Classification is performed throughout a QoS-enabled network, but detailed application classification is typically concentrated at the edge.
- **BA classification** uses only the DSCP field.
- **MF classification** uses one or more packet header fields and can identify traffic based on addresses, protocols, ports, and markings.
- **CoS** is a 3-bit Layer 2 marking carried in the IEEE 802.1Q tag.
- **DSCP** is a 6-bit Layer 3 marking carried in IPv4 and IPv6 headers.
- **IP Precedence** uses the three most significant DSCP bits, making it less granular than DSCP classification.
- **ACL-based classification** can match source/destination IP addresses and TCP/UDP ports.
- ACL `deny` statements used for QoS classification do not directly drop packets.
- **NBAR** provides more sophisticated application identification than ordinary port-based classification.
- **MPLS TC** is a 3-bit field in the MPLS label header used for QoS classification.
- **QoS groups and internal traffic classes** are device-local identifiers, not packet-header markings.
- **QoS pre-classification** can preserve original packet information for classifying encapsulated traffic.

---

## Further Reading

- [MQC Class Maps and Match Criteria](class-maps.md)
- [QoS Trust Boundaries](trust-boundaries.md)
- [4.4.c Network Based Application Recognition (NBAR)](../nbar/overview.md)
- [4.4.d Marking DSCP Values in IPv4 and IPv6 Headers](../marking/overview.md)

### Cisco Documentation

- [Cisco IOS XE 17.x: Applying QoS Features Using the MQC](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-apply.html)
- [Cisco Catalyst 9000: Quality of Service Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/qos/quality-of-service-configuration-guide/m-quality-of-service.html)
- [Cisco IOS XE 17: IPv6 Quality of Service](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_classn/configuration/xe-17/qos-classn-xe-17-book/ip6-qos-xe.html)
- [Cisco: Quality of Service Options on GRE Tunnel Interfaces](https://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-policing/10106-qos-tunnel.html)
