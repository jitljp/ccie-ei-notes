# 4.4.a Differentiated Services Architecture

**Differentiated Services (DiffServ)** is a Quality of Service (QoS) architecture that provides different forwarding treatments to different classes of network traffic.

Rather than treating every packet equally, DiffServ groups traffic into **classes** based on application requirements and network policies. Network devices can then allocate resources differently to each class.

For example:
- **Voice traffic** can receive preferential treatment to minimize delay and jitter.
- **Business-critical applications** can receive a guaranteed share of available bandwidth.
- **Bulk transfers and backups** can receive best-effort treatment.

DiffServ uses the **Differentiated Services Code Point (DSCP)** field in the IP header to identify the forwarding treatment a packet should receive.

The forwarding treatment applied by an individual network device is called a **Per-Hop Behavior (PHB)**.

The key principle of DiffServ is to perform detailed traffic classification and conditioning at the **network edge**, while allowing devices in the **network core** to make relatively simple forwarding decisions based on packet markings.

---

## QoS Models

Three fundamental QoS models are commonly discussed:

| Model | Description |
|---|---|
| **Best Effort** | All traffic receives ordinary forwarding treatment without QoS guarantees. |
| **Integrated Services (IntServ)** | Uses per-flow resource reservation, typically through RSVP, to provide QoS for individual flows. |
| **Differentiated Services (DiffServ)** | Groups traffic into classes and provides forwarding treatment based on those classes. |

**IntServ** maintains resource reservation state for individual flows, which can become difficult to scale in large networks.

**DiffServ** provides a more scalable approach because routers do not need to maintain separate QoS reservation state for every individual flow.

Instead, traffic from many individual flows is aggregated into a smaller number of classes.

DiffServ is the primary QoS architecture used in modern enterprise IP networks.

See [QoS Models and DiffServ Architecture](qos-models.md) for a more detailed comparison.

---

## DiffServ Architecture

A DiffServ network generally consists of three components:

1. **Boundary nodes:** Classify, mark, and condition traffic entering or leaving the network.
2. **Interior nodes:** Forward packets according to their assigned PHBs.
3. **DiffServ domain:** A collection of network devices operating under a coordinated QoS policy.

Consider the following network:

```text
             DiffServ Domain
    +------------------------------------+
    |                                    |
    |   Ingress       Interior   Egress  |
    |   Boundary       Nodes     Boundary|
    |     R1           R2   R3      R4   |
----+-----|------------|----|-------|----+----
    |     |            |    |       |    |
    | Classify         |    |       |    |
    | Mark             |    |       |    |
    | Police/Shape     |    |       |    |
    |     |            |    |       |    |
    |     +------------+----+-------+    |
    |       PHB-based forwarding         |
    +------------------------------------+
```

### Boundary Nodes

At the network boundary, traffic can be classified based on fields such as:
- Source and destination IP addresses.
- TCP or UDP port numbers.
- Protocol or application type.
- Existing DSCP markings.

After classification, the device can **mark** packets with an appropriate DSCP value.

Boundary nodes can also perform **traffic conditioning**, including policing, shaping, and remarking, to ensure that traffic complies with the network's QoS policy.

### Interior Nodes

Inside the DiffServ domain, routers and switches can classify packets using their existing DSCP values.

Each device maps the DSCP value to a **PHB**, which determines how the packet is treated.

For example, a device may place packets into different output queues and allocate different amounts of bandwidth to those queues.

This separation between detailed edge classification and simpler core forwarding is fundamental to DiffServ's scalability.

See [DiffServ Domains and Traffic Conditioning](domains.md) for more information.

---

## Differentiated Services Code Point (DSCP)

**DSCP** is a 6-bit field used to identify the intended forwarding treatment for an IP packet.

It occupies the six most significant bits of:
- The former **Type of Service (ToS)** octet in the IPv4 header.
- The **Traffic Class** octet in the IPv6 header.

The remaining two bits are used for **Explicit Congestion Notification (ECN)**.

```text
       IPv4 ToS / IPv6 Traffic Class

 Bit:   0   1   2   3   4   5   6   7
      +---+---+---+---+---+---+---+---+
      |          DSCP         |  ECN  |
      +---+---+---+---+---+---+---+---+
      |<------ 6 bits ------->|<- 2 ->|
```

Because DSCP is 6 bits long, it supports **64 possible values (0 through 63)**.

Some commonly used values are:

| DSCP | Decimal value | Typical use |
|---|---:|---|
| **Default (DF/BE)** | 0 | Best-effort traffic |
| **AF21** | 18 | Assured Forwarding class 2, low drop precedence |
| **AF31** | 26 | Assured Forwarding class 3, low drop precedence |
| **AF41** | 34 | Assured Forwarding class 4, low drop precedence |
| **EF** | 46 | Low-delay traffic, commonly voice |
| **CS6** | 48 | Network control traffic |

**DSCP is a marking, not a guarantee.** Simply setting a packet's DSCP to EF does not automatically guarantee low delay or preferential treatment throughout the network.

Network devices must be configured to provide the intended forwarding behavior, and sufficient resources must be available.

The full details of DSCP encoding and marking are covered in [4.4.d Marking DSCP Values in IPv4 and IPv6 Headers](../marking/overview.md).

---

## Per-Hop Behaviors (PHBs)

A **Per-Hop Behavior (PHB)** defines the forwarding treatment that a DiffServ device provides to a particular group of packets.

A PHB describes the **behavior**, not the specific implementation.

For example, a PHB may specify that a traffic class receives preferential forwarding treatment, but the device may implement that behavior using particular queuing and scheduling mechanisms.

The three main PHB categories are:

| PHB | Description |
|---|---|
| **Best Effort (BE)** | Ordinary forwarding without special QoS treatment. |
| **Expedited Forwarding (EF)** | Intended to support low delay, low jitter, and low packet loss. |
| **Assured Forwarding (AF)** | Provides four forwarding classes, each with three levels of drop precedence. |

### Best Effort (BE)

**Best Effort** is the default forwarding behavior.

Packets marked with DSCP 0 receive ordinary forwarding treatment without special QoS assurances.

### Expedited Forwarding (EF)

**Expedited Forwarding** is designed to support applications sensitive to delay, jitter, and packet loss.

Its recommended DSCP value is **46**.

EF traffic is commonly placed in a low-latency priority queue, with appropriate traffic-rate controls to prevent excessive EF traffic from consuming resources intended for other classes.

### Assured Forwarding (AF)

**Assured Forwarding** defines four classes, each containing three drop-precedence levels.

The codepoints follow the format `AFxy`:
- **x:** Forwarding class, from 1 to 4.
- **y:** Drop precedence, from 1 to 3.

For example:
- `AF31` belongs to class 3 with low drop precedence.
- `AF32` belongs to class 3 with medium drop precedence.
- `AF33` belongs to class 3 with high drop precedence.

When congestion occurs, packets with higher drop precedence are more likely to be dropped than packets with lower drop precedence **within the same AF class**.

The class number does not independently guarantee a particular priority relationship between different AF classes.

See [Per-Hop Behaviors (BE, EF, AF)](phb.md) for a detailed explanation.

---

## Behavior Aggregates

One of DiffServ's most important concepts is the **Behavior Aggregate (BA)**.

A Behavior Aggregate is a collection of packets that share the same DSCP value and therefore receive the same PHB at a DiffServ node.

For example, suppose 100 IP phones are sending voice traffic through a router.

Rather than maintaining a separate QoS reservation for each phone's voice flow, the router can treat their packets as part of a single **EF behavior aggregate**.

This allows DiffServ to scale to large networks without maintaining individual QoS reservation state for every flow.

### Behavior Aggregate vs. Multifield Classification

DiffServ defines two fundamental classification methods:

| Classifier | Operation |
|---|---|
| **Behavior Aggregate (BA)** | Classifies packets based only on DSCP. |
| **Multifield (MF)** | Classifies packets using one or more packet fields, such as addresses, protocols, ports, or DSCP. |

A typical design uses **MF classification near the network edge** to identify applications and assign appropriate markings.

Devices in the core can then use **BA classification** to apply the configured PHBs.

---

## Traffic Conditioning

**Traffic conditioning** controls how traffic is marked and how its transmission characteristics are regulated.

A DiffServ traffic conditioner can contain four components:

| Component | Function |
|---|---|
| **Meter** | Measures traffic against a defined traffic profile, such as a rate and burst allowance. |
| **Marker** | Sets or changes a packet's DSCP value. |
| **Shaper** | Delays packets to bring traffic into compliance with a specified rate or profile. |
| **Dropper** | Discards packets that violate the configured traffic policy. |

For example, a provider might allow a customer to transmit a certain amount of traffic as a preferred class.

Traffic within the agreed profile retains its preferred marking, while excess traffic may be remarked to a lower service class or dropped.

A traffic conditioner does not have to use all four components.

DiffServ domains can also establish **Traffic Conditioning Agreements (TCAs)** that specify classification rules, traffic profiles, and the actions applied to conforming and nonconforming traffic.

See [DiffServ Domains and Traffic Conditioning](domains.md).

---

## End-to-End DiffServ Operation

Consider an enterprise network carrying three traffic classes:

| Traffic | DSCP | Intended treatment |
|---|---|---|
| Voice | EF (46) | Low-delay forwarding |
| Business applications | AF31 (26) | Allocated bandwidth with assured forwarding |
| Backups | BE (0) | Best-effort forwarding |

A typical packet follows these steps:

1. **Classification:** The ingress switch or router identifies the application.
2. **Marking:** The packet receives the appropriate DSCP value.
3. **Traffic conditioning:** The device may police or shape traffic according to the policy.
4. **Core forwarding:** Each interior device maps the DSCP value to a configured PHB.
5. **Congestion handling:** Queuing and congestion-avoidance mechanisms allocate resources and determine which packets are transmitted or dropped.
6. **Domain boundary:** Another network may preserve the DSCP marking, remark it, or apply a different QoS policy.

Importantly, a packet does not carry a reserved amount of bandwidth or an end-to-end service guarantee.

**DiffServ provides hop-by-hop forwarding treatment.** End-to-end QoS depends on consistent policies, traffic conditioning, and adequate resources across the entire path.

---

## Key Points

- **DiffServ** is a scalable, class-based QoS architecture that avoids per-flow resource reservation in the network core.
- Traffic is typically **classified, marked, and conditioned at the network boundary**.
- **DSCP** is a 6-bit field used to identify the intended forwarding treatment in IPv4 and IPv6.
- A **PHB** defines how a network device treats packets belonging to a behavior aggregate.
- **BA classification** uses DSCP alone, while **MF classification** can inspect multiple packet fields.
- **BE, EF, and AF** are the fundamental PHB categories.
- **Traffic conditioning** includes metering, marking, shaping, and dropping.
- **DSCP markings do not guarantee QoS by themselves**. Network devices must implement the intended PHBs.
- Effective end-to-end QoS requires consistent policies and sufficient resources across all relevant network segments.

---

## References

- [Cisco: Quality of Service Configuration Guide, Cisco IOS XE 17](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/qos/quality-of-service-configuration-guide/m-quality-of-service.html)
- [Cisco: Implement QoS Policies with Differentiated Services Code Point](https://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-packet-marking/10103-dscpvalues.html)
- [RFC 2474: Definition of the Differentiated Services Field](https://www.rfc-editor.org/rfc/rfc2474)
- [RFC 2475: An Architecture for Differentiated Services](https://www.rfc-editor.org/rfc/rfc2475)
- [RFC 2597: Assured Forwarding PHB Group](https://www.rfc-editor.org/rfc/rfc2597)
- [RFC 3246: An Expedited Forwarding PHB](https://www.rfc-editor.org/rfc/rfc3246)