# CoPP Traffic Classification

**CoPP traffic classification** determines which CPU-bound packets belong to each traffic class so that different policing actions can be applied.

CoPP uses the **Modular QoS CLI (MQC)** framework, which consists of three main components:

| Component | Function |
|---|---|
| **Class Map** | Identifies traffic using one or more match criteria |
| **Policy Map** | Defines the actions applied to each traffic class |
| **Service Policy** | Applies the policy map to the control plane |

For example, a CoPP policy might classify traffic into the following categories:

| Traffic Class | Examples | Typical Treatment |
|---|---|---|
| Routing | OSPF, EIGRP, BGP | High-priority protection with sufficient policing rates |
| Management | SSH, SNMP | Restrict to trusted sources and police |
| ICMP | Echo Requests, other ICMP messages | Rate-limit |
| Unwanted | Unauthorized management traffic | Drop or strictly police |
| Default | All remaining CPU-bound traffic | Apply a default policing policy |

Separating traffic into classes allows critical routing protocols to receive different treatment from less important traffic.

---

## CoPP Classification Process

CoPP classification is performed on traffic directed toward the control plane, not on all packets traversing the device.

For an input CoPP policy:

```text
Incoming Packet
       |
       v
 Forwarding Plane
       |
       | CPU-bound traffic
       v
  CoPP Policy
       |
       v
  Classify Packet
       |
       +---- Routing Class
       |
       +---- Management Class
       |
       +---- ICMP Class
       |
       +---- class-default
       |
       v
  Apply Class Action
       |
       +---- Transmit to CPU
       |
       +---- Drop
```

**Classification determines which policy-map class processes the packet.**

Classification itself does not necessarily permit, deny, or drop the packet. The configured policy action determines its treatment.

---

## Class Maps

A **class map** defines the criteria used to identify a particular category of traffic.

The basic MQC syntax is:

```text
class-map [match-all | match-any] <class-name>
```

Within the class map, one or more `match` commands specify the criteria.

For example:

```text
R1(config)# class-map match-all COPP-SSH
R1(config-cmap)# match access-group name SSH-ACL
```

This class map identifies packets permitted by the named ACL `SSH-ACL`.

Class maps are created in global configuration mode.

> Creating a class map does not activate CoPP. The class map must be referenced by a policy map, and that policy map must be applied to the control plane.

---

## match-all vs. match-any

Class maps support two matching modes.

| Mode | Behavior |
|---|---|
| `match-all` | All configured match criteria must match |
| `match-any` | At least one configured match criterion must match |

If neither keyword is specified, **`match-all` is the default**.

### match-all

For example:

```text
R1(config)# class-map match-all COPP-SSH
R1(config-cmap)# match access-group name SSH-ACL
R1(config-cmap)# match ip dscp cs3
```

A packet must satisfy both conditions:

1. Match a permitted entry in `SSH-ACL`.
2. Have a DSCP value of CS3.

```text
Matches SSH-ACL AND DSCP CS3
                |
                v
          COPP-SSH Class
```

A packet matching only one condition does not belong to this class.

### match-any

For example:

```text
R1(config)# class-map match-any COPP-ROUTING
R1(config-cmap)# match access-group name OSPF-ACL
R1(config-cmap)# match access-group name BGP-ACL
```

A packet must match at least one of the ACLs.

```text
Matches OSPF-ACL OR BGP-ACL
                |
                v
         COPP-ROUTING Class
```

This is useful when multiple routing protocols should receive the same CoPP treatment.

---

## ACL-Based Classification

**Access Control Lists (ACLs)** are commonly used to classify CoPP traffic.

Extended ACLs can identify traffic based on:

- Source IP address.
- Destination IP address.
- IP protocol.
- TCP or UDP source port.
- TCP or UDP destination port.
- ICMP message type.

Both numbered and named ACLs can be used on supported IOS XE platforms.

### Named ACL Example

Suppose R1 has the following addressing:

```text
R1: 10.1.1.1
PC1: 10.1.1.10
```

We want to classify SSH traffic sent from PC1 to R1.

Create an extended ACL:

```text
R1(config)# ip access-list extended COPP-SSH-ACL
R1(config-ext-nacl)# permit tcp host 10.1.1.10 host 10.1.1.1 eq 22
```

Then reference the ACL in a class map:

```text
R1(config)# class-map match-all COPP-SSH
R1(config-cmap)# match access-group name COPP-SSH-ACL
```

This classifies TCP packets with:

```text
Source IP:       10.1.1.10
Destination IP:  10.1.1.1
Destination Port: 22
```

Traffic matching the ACL belongs to the `COPP-SSH` class.

### Numbered ACL Example

A numbered extended ACL can also be used:

```text
R1(config)# access-list 101 permit tcp host 10.1.1.10 host 10.1.1.1 eq 22
```

Reference it using its number:

```text
R1(config)# class-map match-all COPP-SSH
R1(config-cmap)# match access-group 101
```

Both examples identify the same traffic.

---

## ACL Permit and Deny Behavior

An important distinction is that **ACL permit and deny statements behave differently when used for CoPP classification than when an ACL is applied directly to an interface for packet filtering**.

When an ACL is referenced by an MQC class map:

- **Permit:** The packet matches the ACL criterion and can be classified into the class.
- **Deny:** The packet does not match that ACL criterion.

An ACL `deny` does **not automatically drop the packet**.

The ACL is just a tool for matching traffic, like in a NAT configuration.

### Example

Consider the following ACL:

```text
R1(config)# ip access-list extended COPP-SSH-ACL
R1(config-ext-nacl)# deny tcp host 10.1.1.10 host 10.1.1.1 eq 22
R1(config-ext-nacl)# permit tcp any host 10.1.1.1 eq 22
```

Apply it to a class map:

```text
R1(config)# class-map match-all COPP-SSH
R1(config-cmap)# match access-group name COPP-SSH-ACL
```

The ACL is evaluated as follows:

| Source | Destination | CoPP-SSH Match? |
|---|---|---|
| 10.1.1.10 | 10.1.1.1 TCP/22 | No |
| 10.1.1.20 | 10.1.1.1 TCP/22 | Yes |
| 10.1.1.30 | 10.1.1.1 TCP/22 | Yes |
| 10.1.1.20 | 10.1.1.1 TCP/23 | No |

The first ACE excludes PC1 from the `COPP-SSH` class.

Other SSH traffic destined for R1 matches the class.

**The denied traffic is not automatically dropped.** It can match another policy-map class or be processed by `class-default`.

### Implicit Deny

As with normal ACLs, an implicit deny exists at the end.

For CoPP classification, this means that packets not explicitly permitted by the ACL **do not match that ACL criterion**.

```text
ACL Permit       -> Matches ACL criterion
ACL Deny         -> Does not match ACL criterion
ACL Implicit Deny -> Does not match ACL criterion
```

> An ACL used for CoPP classification identifies traffic. Actual packet dropping depends on the actions configured in the policy map.

---

## Classifying Routing Protocol Traffic

Routing protocol traffic is particularly important because excessive policing can cause neighbor relationships to fail.

Routing protocols should generally be classified separately from management and less critical traffic.

### OSPF

OSPF uses IP protocol number **89**.

Create an ACL to classify OSPF packets:

```text
R1(config)# ip access-list extended COPP-OSPF
R1(config-ext-nacl)# permit ospf any any
```

Create a class map:

```text
R1(config)# class-map match-all COPP-OSPF-CLASS
R1(config-cmap)# match access-group name COPP-OSPF
```

This identifies OSPF packets reaching the control plane, including packets sent to OSPF multicast destinations and unicast OSPF packets.

The ACL can be made more restrictive by specifying the expected source and destination addresses.

### EIGRP

EIGRP uses IP protocol number **88**.

```text
R1(config)# ip access-list extended COPP-EIGRP
R1(config-ext-nacl)# permit eigrp any any
```

```text
R1(config)# class-map match-all COPP-EIGRP-CLASS
R1(config-cmap)# match access-group name COPP-EIGRP
```

The class identifies EIGRP packets sent to the control plane.

### BGP

BGP uses **TCP port 179**.

Because BGP runs over TCP, its packets can have port 179 as either the source or destination port.

Consider the following BGP peers:

```text
R1: 10.1.1.1
R2: 10.1.1.2
```

An ACL that matches both possibilities is:

```text
R1(config)# ip access-list extended COPP-BGP
R1(config-ext-nacl)# permit tcp host 10.1.1.2 host 10.1.1.1 eq 179
R1(config-ext-nacl)# permit tcp host 10.1.1.2 eq 179 host 10.1.1.1
```

The first entry matches incoming packets whose destination port is 179.

The second entry matches incoming packets whose source port is 179.

Both forms may occur depending on which peer actively establishes the TCP session.

Create the class map:

```text
R1(config)# class-map match-all COPP-BGP-CLASS
R1(config-cmap)# match access-group name COPP-BGP
```

> When classifying BGP using TCP ports, consider both connection-establishment directions. Matching only destination port 179 can miss incoming BGP packets from a peer using source port 179.

### Combining Routing Protocols

Different routing protocols can share one policing class.

For example:

```text
R1(config)# ip access-list extended COPP-ROUTING
R1(config-ext-nacl)# permit ospf any any
R1(config-ext-nacl)# permit eigrp any any
R1(config-ext-nacl)# permit tcp host 10.1.1.2 host 10.1.1.1 eq 179
R1(config-ext-nacl)# permit tcp host 10.1.1.2 eq 179 host 10.1.1.1
```

```text
R1(config)# class-map match-all COPP-ROUTING-CLASS
R1(config-cmap)# match access-group name COPP-ROUTING
```

This creates one class for OSPF, EIGRP, and the specified BGP peer.

All traffic matching this class receives the same policy-map treatment.

---

## Classifying Management Traffic

Management traffic can also be classified using extended ACLs.

Common examples include:

| Protocol | IP Protocol | Port |
|---|---|---|
| SSH | TCP | 22 |
| Telnet | TCP | 23 |
| SNMP Queries | UDP | 161 |
| NETCONF over SSH | TCP | 830 |
| HTTPS | TCP | 443 |

These port numbers identify commonly used services, but an ACL should also consider the destination and expected source addresses.

For example:

```text
R1(config)# ip access-list extended COPP-MGMT
R1(config-ext-nacl)# permit tcp 10.10.10.0 0.0.0.255 host 10.1.1.1 eq 22
R1(config-ext-nacl)# permit udp host 10.10.10.100 host 10.1.1.1 eq 161
```

This matches:

- SSH connections from the `10.10.10.0/24` management subnet.
- SNMP queries from `10.10.10.100`.

Create the class map:

```text
R1(config)# class-map match-all COPP-MGMT-CLASS
R1(config-cmap)# match access-group name COPP-MGMT
```

This provides a separate class for management traffic.

> A CoPP classification ACL does not replace VTY access restrictions, AAA, or management-plane access controls. Its primary purpose is to identify traffic for QoS treatment.

---

## Classifying ICMP Traffic

ICMP packets directed toward the device can consume CPU resources.

They are commonly classified separately from routing and management traffic.

For example:

```text
R1(config)# ip access-list extended COPP-ICMP
R1(config-ext-nacl)# permit icmp any any echo
```

```text
R1(config)# class-map match-all COPP-ICMP-CLASS
R1(config-cmap)# match access-group name COPP-ICMP
```

This class identifies ICMP Echo Requests reaching the control plane.

To classify all ICMP messages instead:

```text
R1(config)# ip access-list extended COPP-ICMP
R1(config-ext-nacl)# permit icmp any any
```

The distinction matters because ICMP includes more than just ping.

ICMP error messages can be important for normal network operation, including Path MTU Discovery.

Excessively restrictive policing can therefore disrupt legitimate traffic.

---

## IPv6 Traffic Classification

CoPP can also classify IPv6 traffic on supported IOS XE platforms.

IPv6 ACLs use the `ipv6 access-list` command.

For example, suppose R1 has the following IPv6 address:

```text
2001:DB8:1::1
```

Create an IPv6 ACL to match SSH traffic:

```text
R1(config)# ipv6 access-list COPP-SSH-V6
R1(config-ipv6-acl)# permit tcp 2001:DB8:10::/64 host 2001:DB8:1::1 eq 22
```

Reference the ACL in a class map:

```text
R1(config)# class-map match-all COPP-SSH-V6-CLASS
R1(config-cmap)# match access-group name COPP-SSH-V6
```

This identifies IPv6 SSH packets sent from the specified subnet to R1.

### IPv6 Routing and Neighbor Discovery Traffic

IPv6 control-plane traffic includes protocols such as:

- OSPFv3.
- EIGRP for IPv6.
- BGP over IPv6.
- ICMPv6.
- IPv6 Neighbor Discovery.

IPv6 Neighbor Discovery is particularly important because it provides essential functions such as address resolution and router discovery.

When designing CoPP policies, IPv6 control-plane traffic must be classified and protected appropriately.

**IPv4 ACLs do not automatically classify IPv6 traffic.**

On platforms supporting both IPv4 and IPv6 ACL classification, separate ACLs can be used and combined into an appropriate class map.

---

## Classification Using DSCP and IP Precedence

CoPP classification is not limited to ACLs.

On supported platforms, class maps can also match packet QoS markings.

Examples include:

```text
match ip dscp cs6
match ip precedence 6
```

DSCP and IP precedence matching can be useful when packets are consistently marked within the network.

For example:

```text
R1(config)# class-map match-all COPP-CS6
R1(config-cmap)# match ip dscp cs6
```

This class matches CPU-bound IP packets with DSCP CS6.

However, **DSCP markings alone do not reliably identify trusted traffic**.

An attacker may generate packets with the same markings.

For security-sensitive classification, matching source addresses, destination addresses, and protocol information using ACLs is generally more reliable.

---

## Classifying Non-IP Traffic

IP ACLs cannot match all types of CPU-bound traffic.

Some Layer 2 control protocols do not use IP.

For example, ARP uses EtherType `0x0806`, rather than an IP protocol number.

On supported IOS XE router platforms, ARP can be classified using:

```text
R1(config)# class-map match-all COPP-ARP
R1(config-cmap)# match protocol arp
```

This identifies ARP traffic subject to the control-plane policy.

Other supported protocol match criteria can include:

```text
match protocol ipv6
match protocol pppoe
match protocol pppoe-discovery
```

These commands are platform-dependent.

For example, `match protocol ipv6` provides broad matching of IPv6 packets rather than identifying one particular IPv6 application.

> Not all MQC match commands supported for ordinary interface QoS policies are available under CoPP. Platform-specific restrictions must be considered.

---

## Policy Map Class Selection

After classification, packets are processed by the appropriate class in the policy map.

**Policy-map classes are evaluated in order.**

The first matching class determines the treatment applied to the packet.

Consider the following conceptual policy:

```text
policy-map COPP-IN
 class COPP-ROUTING
 class COPP-MGMT
 class COPP-ICMP
 class class-default
```

The policy-map classes reference previously configured class maps.

For example, if an OSPF packet matches `COPP-ROUTING`, it is processed by that class.

It is not subsequently processed by `COPP-MGMT` or `COPP-ICMP`.

### Overlapping Match Criteria

Suppose one class matches all IP traffic:

```text
R1(config)# ip access-list extended COPP-ALL-IP
R1(config-ext-nacl)# permit ip any any
```

If this class appears before a more specific OSPF class, OSPF traffic may match the broad class first.

The specific OSPF class would then not receive those packets.

Therefore, **more specific classes should generally appear before broader classes**.

### class-default

The special **`class-default`** class identifies traffic that does not match any preceding class in the policy map.

For example:

```text
Routing traffic    -> COPP-ROUTING
Management traffic -> COPP-MGMT
ICMP traffic       -> COPP-ICMP
Other traffic      -> class-default
```

If `class-default` has no configured policing or dropping action, otherwise unmatched packets are not dropped merely because they fall into this class.

A policing or dropping action must be configured to restrict them.

This makes `class-default` important when designing a complete CoPP policy.

---

## Classification Restrictions

CoPP does not support every classification feature available in MQC.

On some IOS XE router platforms:

- `match input-interface` is not supported for CoPP.
- Classification using NBAR-dependent features may not function as expected.
- Certain Layer 2 and hardware-specific classifications are unavailable.
- Supported IPv4, IPv6, and non-IP match criteria vary by platform.
- Hardware resources can limit the number and complexity of supported classes.

CoPP policies generally aggregate CPU-bound traffic from multiple forwarding interfaces.

This differs from an ordinary QoS service policy applied directly to an individual physical interface.

### Catalyst 9000 Restrictions

Modern Catalyst 9000 switches use predefined **System CoPP** classes.

On platforms such as the Catalyst 9300, administrators cannot create arbitrary user-defined CoPP classes or modify the predefined class definitions.

The class-map and ACL examples in this page therefore apply to **IOS XE platforms supporting user-defined CoPP policies**, such as supported router platforms.

Catalyst 9000 System CoPP classification is covered separately in the System CoPP chapter.

---

## IOS XE Verification

Use the following commands to inspect CoPP classification.

### Display Class Maps

```text
R1# show class-map
```

Displays configured class maps and their match criteria.

To inspect an individual class:

```text
R1# show class-map COPP-ROUTING-CLASS
```

### Display Classification ACLs

```text
R1# show ip access-lists COPP-ROUTING
```

For IPv6:

```text
R1# show ipv6 access-list COPP-SSH-V6
```

These commands display the ACL entries used for traffic classification.

ACL hit counters may not reflect all CoPP matches, particularly when classification is performed in hardware.

### Display CoPP Policy Statistics

After applying a CoPP policy:

```text
R1# show policy-map control-plane
```

This command can display:

- Applied control-plane policy.
- Matched traffic classes.
- Class-map match criteria.
- Packet and byte counters.
- Policing rates and actions.
- Packet drops.

For example, a class may be displayed with:

```text
Class-map: COPP-ROUTING-CLASS (match-all)
  100 packets, 8000 bytes
  Match: access-group name COPP-ROUTING
```

This indicates that the class has matched 100 packets.

Actual output and available counters depend on the platform.

> `show policy-map control-plane` is generally more useful than ACL hit counters for verifying that CPU-bound traffic is matching the intended CoPP class.

---

## Key Points

- CoPP uses **MQC class maps** to classify CPU-bound traffic.
- Class maps define match criteria; policy maps define the actions.
- **`match-all`** requires all criteria to match and is the default.
- **`match-any`** requires at least one criterion to match.
- ACLs are commonly used to classify traffic by IP addresses, protocols, and ports.
- In an ACL used for CoPP classification, **`permit` means match the traffic**, not automatically allow it.
- An ACL **`deny` excludes traffic from that ACL match criterion**, but does not automatically drop it.
- The ACL's implicit deny also means the traffic does not match that ACL criterion.
- BGP classification should account for TCP port 179 appearing as either the source or destination port.
- IPv4 and IPv6 traffic require appropriate classification criteria.
- Non-IP traffic, such as ARP, may require protocol-based matching.
- Policy-map classes are processed in order, and the **first matching class** determines the packet's treatment.
- **`class-default`** matches traffic that does not match any preceding class.
- CoPP class maps do not automatically restrict traffic. The configured policy actions determine whether traffic is transmitted or dropped.
- Modern Catalyst 9000 System CoPP uses predefined classes rather than arbitrary user-defined class maps.

---

## References

- [Cisco - Control Plane Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/quality-of-service/quality-of-service/m_qos-plcshp-ctrl-pln-plc-0.html)
- [Cisco - Applying QoS Features Using the MQC](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-apply.html)
- [Cisco ASR 1000 - Control Plane Policing](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_plcshp/configuration/xe-3s-asr1000/qos-plcshp-xe-3s-asr-1000-book/qos-plcshp-ctrl-pln-plc.html)
- [Cisco Catalyst 9300 IOS XE 17.18.x - Configuring Control Plane Policing](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-18/configuration_guide/sec/b_1718_sec_9300_cg/configuring_control_plane_policing.html)