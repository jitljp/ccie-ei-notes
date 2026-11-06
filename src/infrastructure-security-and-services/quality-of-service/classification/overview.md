# 4.4.b Classification and Trust Boundary

**Traffic classification** is the process of identifying packets and assigning them to traffic classes so that appropriate Quality of Service (QoS) policies can be applied.

Classification allows network devices to distinguish between traffic with different performance requirements, such as voice, video, business applications, and background traffic.

For example, a router might classify:
- **Voice traffic** into a class that receives low-latency forwarding.
- **Business applications** into a class with guaranteed bandwidth.
- **Backup traffic** into a class that receives lower priority.
- **Unclassified traffic** into the default class.

Classification is performed throughout a QoS-enabled network, but the classification methods used can differ.

At the **network edge**, devices typically perform detailed classification using packet characteristics such as IP addresses, protocols, and TCP/UDP ports. Packets are then **marked** with appropriate QoS values, such as DSCP.

Within the **network core**, devices typically perform simpler classification based on the existing DSCP markings, allowing them to apply the appropriate forwarding treatment without repeating detailed classification.

The **QoS trust boundary** determines where the network begins accepting QoS markings assigned by other devices.

---

## Classification vs. Marking

Although classification and marking are closely related, they are separate operations.

| Operation | Function |
|---|---|
| **Classification** | Identifies traffic and assigns packets to a traffic class |
| **Marking** | Sets or changes a field in the packet or frame to indicate the intended QoS treatment |

For example, a router might classify traffic by examining its TCP destination port and then mark matching packets with DSCP AF31.

```text
          Incoming Packet
                 |
                 v
       +------------------+
       |  Classification  |
       |                  |
       |  TCP port 443    |
       |  Server address  |
       +------------------+
                 |
                 v
          Business Class
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
```

Classification does not necessarily require marking.

For example, a router can classify traffic using an existing DSCP value and apply a queuing policy without changing that value.

Likewise, marking does not automatically give packets preferential forwarding treatment. The downstream network devices must implement the corresponding QoS policies.

---

## Classification Methods

Network devices can classify traffic using information from different layers of the network protocol stack.

| Layer | Classification criteria | Example |
|---|---|---|
| **Layer 2** | IEEE 802.1Q CoS, MAC addresses | Match CoS 5 |
| **Layer 3** | Source/destination IP, DSCP, IP Precedence | Match DSCP EF |
| **Layer 4** | TCP/UDP port numbers | Match TCP destination port 443 |
| **Layer 7** | Application recognition | Identify an application using NBAR |

### Layer 2 Classification

At Layer 2, classification commonly uses the **Class of Service (CoS)** field.

CoS is a 3-bit priority field within the IEEE 802.1Q tag, supporting values from **0 through 7**.

For example:
- CoS 0: Commonly used for best-effort traffic.
- CoS 3: Commonly used for call signaling.
- CoS 5: Commonly used for voice bearer traffic.

CoS is carried in tagged Ethernet frames. Untagged Ethernet frames do not contain a CoS field.

Because CoS is part of the Ethernet header, its value is not carried end-to-end across routed IP networks.

### Layer 3 Classification

Layer 3 classification uses information from the IP header.

Common criteria include:
- Source IP address.
- Destination IP address.
- Differentiated Services Code Point (DSCP).
- IP Precedence.

DSCP is particularly important because it allows downstream devices to classify packets using a single **6-bit field**.

Unlike CoS, DSCP is carried in the IPv4 or IPv6 header and can be preserved across routed hops.

### Layer 4 Classification

Layer 4 classification uses transport-layer information, particularly **TCP and UDP port numbers**.

For example, an ACL can identify packets with a particular destination IP address and TCP destination port.

Port-based classification is useful when applications use known ports, although it cannot reliably identify every application.

### Application-Based Classification

**Network Based Application Recognition (NBAR)** provides more advanced application classification.

Rather than relying exclusively on addresses and port numbers, NBAR can identify applications using protocol-specific characteristics and application signatures.

This is useful when applications:
- Use dynamic ports.
- Share well-known port numbers.
- Cannot be reliably distinguished by ordinary ACLs.

See [4.4.c Network Based Application Recognition (NBAR)](../nbar/overview.md).

For more information about the available classification methods, see [Traffic Classification Methods](methods.md).

---

## Behavior Aggregate and Multifield Classification

The DiffServ architecture defines two fundamental types of classifiers.

| Classification type | Description |
|---|---|
| **Behavior Aggregate (BA)** | Classifies packets based on their DSCP values |
| **Multifield (MF)** | Classifies packets using one or more packet header fields |

### Multifield Classification

**Multifield classification** examines packet characteristics such as IP addresses, protocol numbers, TCP/UDP ports, and existing markings.

For example, an edge router might classify traffic based on:
- Source subnet: `10.10.10.0/24`
- Destination server: `192.0.2.10`
- TCP destination port: `443`

After matching these criteria, the router could mark the packets with DSCP AF31.

MF classification is particularly useful at the network edge, where packets may not already have trustworthy QoS markings.

### Behavior Aggregate Classification

**Behavior Aggregate classification** identifies traffic based on its existing DSCP marking.

For example, a core router might recognize all packets marked EF as belonging to the voice traffic class.

It does not need to examine the packets' IP addresses or TCP/UDP ports to make that classification.

```text
      Network Edge                  Network Core

+---------------------+       +------------------+
| MF Classification   |       | BA Classification|
|                     |       |                  |
| IP addresses        |       | Match DSCP EF    |
| Protocols           |       | Match DSCP AF31  |
| TCP/UDP ports       |       | Match DSCP CS0   |
+---------------------+       +------------------+
           |                            ^
           v                            |
     DSCP Marking ----------------------+
```

This is a key advantage of DiffServ: **detailed classification can be performed at the network edge, while the network core uses simpler DSCP-based classification**.

---

## Modular QoS CLI Classification

Cisco IOS XE commonly uses the **Modular QoS Command-Line Interface (MQC)** to define traffic classes and apply QoS policies.

MQC has three primary components:

| Component | Function |
|---|---|
| `class-map` | Defines the criteria used to identify a traffic class |
| `policy-map` | Specifies the QoS actions applied to each class |
| `service-policy` | Attaches the policy to an interface or another supported target |

### Class Maps

A **class map** defines the matching criteria for a particular traffic class.

For example:

```text
R1(config)# class-map match-all VOICE
R1(config-cmap)# match dscp ef
```

This class map identifies packets with DSCP EF.

A class map can contain multiple matching criteria.

Cisco IOS XE supports two main matching modes:

| Mode | Behavior |
|---|---|
| `match-all` | All configured match statements must match |
| `match-any` | At least one configured match statement must match |

**`match-all` is the default** if neither mode is specified.

For example:

```text
R1(config)# class-map match-any CRITICAL
R1(config-cmap)# match dscp ef
R1(config-cmap)# match dscp cs6
```

This class map matches packets marked **EF OR CS6**.

By comparison, `match-all` requires the configured match conditions to be satisfied together.

### ACL-Based Classification

Access control lists can also be used for classification.

For example, suppose traffic going to server `192.0.2.10` on TCP port 443 should belong to a business application class.

```text
R1(config)# ip access-list extended BUSINESS-ACL
R1(config-ext-nacl)# permit tcp any host 192.0.2.10 eq 443
R1(config-ext-nacl)# exit
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match access-group name BUSINESS-ACL
```

The ACL identifies the packets, and the class map assigns matching packets to the `BUSINESS` traffic class.

An important distinction from security ACLs is that **a deny statement in a QoS classification ACL does not mean the packet is dropped**.

Instead, it means the packet does not match that ACL for the relevant QoS classification.

### Policy Maps and Class Selection

A policy map associates traffic classes with QoS actions.

For example:

```text
R1(config)# policy-map EDGE-MARKING
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# set dscp af31
R1(config-pmap-c)# exit
R1(config-pmap)# class class-default
R1(config-pmap-c)# set dscp default
```

This policy:
- Marks packets matching the `BUSINESS` class with DSCP AF31.
- Marks all remaining packets with DSCP 0.

The predefined **`class-default`** class matches traffic that does not match an earlier class in the policy.

Policy classes are evaluated in order. Once a packet matches a class, the configured actions for that class are applied.

The policy can then be attached to an interface:

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input EDGE-MARKING
```

This applies the classification and marking policy to packets entering GigabitEthernet1.

See [MQC Class Maps and Match Criteria](class-maps.md) for detailed configuration and matching behavior.

---

## QoS Trust Boundaries

A **QoS trust boundary** defines the point at which the network begins accepting QoS markings assigned by another device.

For example, a PC could mark all its traffic with DSCP EF, attempting to receive the same treatment as voice traffic.

If the network blindly accepts these markings, the PC's traffic could consume resources intended for legitimate high-priority applications.

A trust boundary prevents this by controlling which devices are permitted to supply authoritative QoS markings.

### Trusted and Untrusted Traffic

| Traffic source | Typical treatment |
|---|---|
| **Trusted IP phone** | Accept its QoS markings, subject to network policy |
| **Untrusted PC** | Classify and remark traffic according to network policy |
| **Trusted network switch** | Preserve markings already validated by the network |
| **External network** | Validate, remark, or police traffic according to the interconnection agreement |

Traffic from an untrusted source can be:
- Classified based on addresses, protocols, or applications.
- Marked with an appropriate DSCP value.
- Remarked to DSCP 0.
- Policed to control the amount of traffic entering a particular service class.

**Trusting a marking means accepting its QoS classification, not authenticating the packet or its source.**

### Trust Boundary Placement

Consider the following enterprise network:

```text
   PC
    |
    |
 IP Phone  <--- QoS Trust Boundary
    |
    |  Voice VLAN / Data VLAN
    |
+-------------------+
|   Access Switch   |
+-------------------+
         |
         |
+-------------------+
| Distribution SW   |
+-------------------+
         |
         |
+-------------------+
|    Core Router    |
+-------------------+
         |
         |
      WAN
```

In this design:
1. The IP phone marks its voice traffic with an appropriate QoS value, commonly DSCP EF.
2. The PC sends ordinary data traffic through the phone's integrated switch.
3. The access switch accepts the phone's authorized markings while ensuring that PC traffic receives appropriate markings.
4. Distribution and core devices classify packets based on the DSCP values established at the network edge.
5. The WAN router applies the appropriate queuing and congestion-management policies.

This design avoids repeatedly performing expensive application classification throughout the network.

### Moving the Trust Boundary

The trust boundary can be located at different points depending on the endpoint.

**Trusted endpoint:** A managed IP phone may be allowed to mark its own traffic.

**Untrusted endpoint:** A PC's markings may be ignored or overwritten by the access switch.

**Trusted network device:** A downstream switch or router can accept markings established by another trusted network device.

The general design principle is to **classify and mark traffic as close to its source as practical, while trusting only devices that are authorized to assign those markings**.

---

## Trust Behavior on Cisco Catalyst Switches

Cisco Catalyst switches support several ways to control the trust boundary.

### Trusting DSCP or CoS

On older Catalyst switches, commands such as `mls qos trust dscp` and `mls qos trust cos` explicitly configured which markings an interface accepted.

On **Catalyst 9000 switches**, incoming DSCP markings are preserved by default unless a policy changes them.

Therefore, an interface without an explicit trust command is not necessarily an untrusted interface.

An ingress MQC policy can remark traffic according to the network's requirements.

Catalyst 9000 switches also support commands such as:

```text
SW1(config-if)# auto qos trust dscp
```

This applies an AutoQoS configuration that trusts incoming DSCP markings.

Alternatively:

```text
SW1(config-if)# auto qos trust cos
```

This applies an AutoQoS configuration based on incoming CoS markings.

AutoQoS can also generate additional classification and queuing configuration.

### Conditional Trust

**Conditional trust** allows a switch to trust QoS markings only when an authorized device is detected.

For example:

```text
SW1(config)# interface GigabitEthernet1/0/10
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# switchport voice vlan 20
SW1(config-if)# trust device cisco-phone
```

The `trust device cisco-phone` command enables conditional trust based on the detection of a Cisco IP phone.

Cisco Discovery Protocol (CDP) is used to identify the connected phone.

If the expected phone is detected, the switch can accept its markings according to the configured trust policy.

If the phone is not detected, incoming QoS markings are not trusted and are reset to 0.

This prevents someone from simply disconnecting the phone, connecting an ordinary PC, and receiving the same trusted treatment.

Conditional trust can also be configured using AutoQoS:

```text
SW1(config-if)# auto qos voip cisco-phone
```

This automatically generates QoS configuration for a Cisco IP phone deployment, including conditional trust and other QoS policy elements.

See [QoS Trust Boundaries](trust-boundaries.md) for detailed examples of static trust, conditional trust, and untrusted endpoint policies.

---

## End-to-End Classification Example

Consider a network carrying three traffic types:

| Traffic | Classification at edge | DSCP marking |
|---|---|---|
| Voice | Trusted IP phone | EF (46) |
| Business application | Source/destination addresses and TCP port | AF31 (26) |
| General data | Default class | CS0 (0) |

At the access layer, the network performs detailed classification and establishes the appropriate DSCP markings.

The distribution and core devices can then classify packets using their DSCP values.

For example, a WAN router could define:

```text
R1(config)# class-map match-all VOICE
R1(config-cmap)# match dscp ef
R1(config-cmap)# exit
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match dscp af31
```

The router does not need to know which application generated the AF31 packets.

It only needs to know the forwarding treatment assigned to that class.

An outbound QoS policy can use these classes to allocate resources:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class VOICE
R1(config-pmap-c)# priority percent 10
R1(config-pmap-c)# exit
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth percent 30
R1(config-pmap-c)# exit
R1(config-pmap)# class class-default
R1(config-pmap-c)# bandwidth percent 20
R1(config-pmap-c)# exit
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output WAN-QOS
```

This example provides:
- Low-latency priority service for EF traffic.
- A minimum bandwidth allocation for AF31 traffic.
- A minimum bandwidth allocation for remaining traffic.

The remaining bandwidth can be used by other classes according to the queuing scheduler.

Notice that the WAN router performs **BA classification** based on DSCP, while detailed MF classification was already performed at the network edge.

---

## Verification

Useful Cisco IOS XE commands include:

| Command | Purpose |
|---|---|
| `show class-map` | Displays configured classes and matching criteria |
| `show policy-map` | Displays policy maps and their configured actions |
| `show policy-map interface` | Displays interface QoS policies and statistics |
| `show running-config interface` | Checks QoS configuration applied to an interface |

For example:

```text
R1# show class-map
R1# show policy-map
R1# show policy-map interface GigabitEthernet1
```

When verifying classification, check:
1. Whether the expected class maps are configured.
2. Whether the correct service policy is attached to the interface.
3. Whether the policy is applied in the correct direction.
4. Whether traffic is matching the expected classes.
5. Whether the configured marking or forwarding actions are being applied.

A configured class map alone does not affect traffic. It must be used by an active QoS policy.

---

## Key Points

- **Classification** identifies traffic and assigns packets to QoS classes.
- **Marking** sets or changes the QoS information carried in packet headers or Ethernet tags.
- **Multifield (MF) classification** examines multiple packet fields and is commonly used at the network edge.
- **Behavior Aggregate (BA) classification** uses DSCP values and is commonly used in the network core.
- **MQC class maps** define classification criteria using commands such as `match dscp` and `match access-group`.
- **`match-all`** requires all match statements to match; **`match-any`** requires at least one.
- **A QoS trust boundary** determines where existing QoS markings begin to be accepted.
- **Untrusted traffic** should be classified and marked according to the network's QoS policy.
- **Conditional trust** allows a switch to accept QoS markings only when a trusted device, such as a Cisco IP phone, is detected.
- **Catalyst 9000 switches preserve incoming DSCP markings by default**, unless a policy changes them.
- The goal of an end-to-end QoS design is to classify and mark traffic near its source, then use those markings to apply consistent forwarding policies throughout the network.

---

## Further Reading

- [Traffic Classification Methods](methods.md)
- [MQC Class Maps and Match Criteria](class-maps.md)
- [QoS Trust Boundaries](trust-boundaries.md)
- [4.4.c Network Based Application Recognition (NBAR)](../nbar/overview.md)
- [4.4.d Marking DSCP Values in IPv4 and IPv6 Headers](../marking/overview.md)
- [4.4.h End-to-End Layer 3 QoS Using MQC](../mqc/overview.md)

### Cisco Documentation

- [Cisco Catalyst 9000 QoS and Queuing White Paper](https://www.cisco.com/c/en/us/products/collateral/switches/catalyst-9000/white-paper-c11-742388.html)
- [Cisco IOS XE Quality of Service Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/qos/quality-of-service-configuration-guide/m-quality-of-service.html)
- [Configuring AutoQoS on Catalyst 9000 Switches](https://www.cisco.com/c/en/us/support/docs/switches/catalyst-9200-series-switches/222225-configure-autoqos-on-catalyst-9000-switc.html)
