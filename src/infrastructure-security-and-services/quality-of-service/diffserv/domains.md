# DiffServ Domains and Traffic Conditioning

A **Differentiated Services (DiffServ) domain** is a contiguous collection of network devices that operate under a common QoS policy and support a common set of **Per-Hop Behaviors (PHBs)**.

Within a DiffServ domain, packets receive different forwarding treatments based on their **Differentiated Services Code Point (DSCP)** markings.

The DiffServ architecture separates two major functions:
- **Traffic classification and conditioning:** Primarily performed at the boundaries of the DiffServ domain.
- **PHB-based forwarding:** Performed throughout the domain, including by interior routers and switches.

This separation allows network devices in the core to provide differentiated forwarding treatment without maintaining resource reservations for individual traffic flows.

---

## DiffServ Domains

A **DiffServ domain** consists of two types of nodes:

| Node type | Function |
|---|---|
| **Boundary node** | Connects the DiffServ domain to another domain and performs classification and traffic conditioning as required. |
| **Interior node** | Forwards traffic within the DiffServ domain using the configured PHBs. |

A DiffServ domain typically belongs to a single administrative organization, such as an enterprise or service provider.

Consider the following topology:

```text
             Provider DiffServ Domain
        +--------------------------------+
        |                                |
LAN ---- PE1 ------- P1 ---- P2 ------ PE2 ---- LAN
        |            |       |           |
        |           Interior Nodes       |
        |                                |
        +--------------------------------+
```

In this example:
- **PE1 and PE2** are boundary nodes because they connect the provider's DiffServ domain to external networks.
- **P1 and P2** are interior nodes because they forward traffic within the provider's domain.

All four routers must apply the appropriate PHBs to traffic according to the domain's QoS policy.

### Boundary Nodes

A **DiffServ boundary node** connects a DiffServ domain to another DiffServ domain or a network that does not implement DiffServ.

A boundary node can perform:
- Traffic classification.
- DSCP marking and remarking.
- Traffic metering.
- Traffic policing.
- Traffic shaping.
- PHB-based forwarding.

Boundary nodes are responsible for ensuring that traffic entering or leaving a domain complies with the applicable traffic-conditioning policy.

Boundary nodes can operate in two roles:

| Role | Description |
|---|---|
| **Ingress node** | Handles traffic entering the DiffServ domain. |
| **Egress node** | Handles traffic leaving the DiffServ domain. |

These roles depend on the **direction of traffic**.

For example, when traffic travels from PE1 to PE2:
- PE1 is the ingress node.
- PE2 is the egress node.

When traffic travels in the opposite direction, their roles are reversed.

### Interior Nodes

A **DiffServ interior node** forwards traffic between other nodes within the same DiffServ domain.

Interior nodes generally use **Behavior Aggregate (BA) classification**, which examines the packet's DSCP value to select the appropriate PHB.

For example, an interior router might:
- Forward EF-marked packets using a low-latency priority queue.
- Allocate bandwidth to AF-marked packets.
- Forward BE-marked packets using best-effort treatment.

Unlike boundary nodes, interior nodes typically do not need to perform detailed application classification or enforce customer traffic agreements.

However, interior nodes can still perform additional classification or traffic conditioning if required by the network's QoS policy.

---

## DiffServ Regions

A **DiffServ region** consists of one or more contiguous DiffServ domains that can provide differentiated services across their combined networks.

For example:

```text
+----------------+    +----------------+    +----------------+
|   Enterprise   |    |    Provider    |    |   Enterprise   |
|   DS Domain    |----|    DS Domain   |----|    DS Domain   |
+----------------+    +----------------+    +----------------+
  DiffServ Region       DiffServ Region       DiffServ Region

```

Each domain can implement its own:
- DSCP-to-PHB mappings.
- Traffic classification policies.
- Traffic conditioning policies.
- Queuing and scheduling mechanisms.

The domains do not have to use identical internal QoS configurations.

However, they must coordinate how traffic is treated when it crosses domain boundaries if they intend to provide consistent end-to-end QoS.

For example, one domain may use DSCP AF31 for a particular application, while another domain uses a different DSCP value for the corresponding service class.

At the boundary, the packet can be **remarked** to the DSCP value used by the receiving domain.

If neighboring domains use compatible markings and forwarding policies, remarking may not be necessary.

---

## Traffic Classification at Domain Boundaries

Before traffic can receive the appropriate forwarding treatment, the network must determine which traffic class it belongs to.

DiffServ defines two fundamental classification methods:

| Classifier | Description |
|---|---|
| **Behavior Aggregate (BA)** | Classifies packets based only on their DSCP values. |
| **Multifield (MF)** | Classifies packets using one or more packet header fields, such as IP addresses, protocols, ports, and DSCP. |

**MF classification** is commonly used at domain boundaries because it can identify traffic using multiple packet characteristics.

For example, an ingress router might identify an application using its source and destination IP addresses and TCP port numbers.

After classification, the router can mark the packet with DSCP AF31.

Interior routers can then identify the packet's forwarding class using BA classification, without repeating the detailed classification.

```text
         Ingress Node                Interior Nodes

Packet --> MF Classifier --> DSCP --> BA --> PHB --> BA --> PHB
                |                      |            |
             Detailed               Simple       Simple
           Classification        Classification Classification
```

The classification process selects traffic for the appropriate forwarding and conditioning policies.

See [Traffic Classification Methods](../classification/methods.md) for more information.

---

## Traffic Conditioning

**Traffic conditioning** controls how traffic is marked and how its transmission characteristics are regulated according to a defined policy.

Traffic conditioning is commonly performed at DiffServ domain boundaries to ensure that incoming or outgoing traffic complies with the applicable service agreement.

A **traffic conditioner** can contain four components:

| Component | Function |
|---|---|
| **Meter** | Measures traffic against a defined traffic profile. |
| **Marker** | Sets or changes a packet's DSCP value. |
| **Shaper** | Buffers and delays packets to bring traffic into compliance with a traffic profile. |
| **Dropper** | Discards packets according to the traffic-conditioning policy. |

A classifier selects packets for processing by the appropriate traffic conditioner.

The following diagram illustrates the logical processing sequence:

```text
Packets
   |
   v
+------------+
| Classifier |
+------------+
   |
   v
+------------+
|   Meter    | ---> Profile conformance result
+------------+
   |
   v
+------------+
|   Marker   |
+------------+
   |
   v
+------------+
|  Shaper /  |
|  Dropper   |
+------------+
   |
   v
Forwarded Packets
```

The meter's result can influence the actions taken by the marker, shaper, or dropper.

**Not every traffic conditioner uses all four components.** For example, a policy that simply marks incoming traffic with a particular DSCP value may require only classification and marking.

### 1. Meter

A **meter** measures traffic against a configured **traffic profile**.

A traffic profile commonly specifies:
- An allowed traffic rate.
- A permitted burst size.

For example, a profile may allow traffic at an average rate of 10 Mbps with a specified burst allowance.

The meter determines whether traffic complies with the profile.

Two fundamental outcomes are:

| Result | Meaning |
|---|---|
| **In-profile (conforming)** | Traffic complies with the configured traffic profile. |
| **Out-of-profile (nonconforming)** | Traffic exceeds the limits defined by the profile. |

The meter reports this information to other components of the traffic conditioner.

**The meter itself does not determine whether a packet is forwarded, remarked, delayed, or dropped.** Those actions depend on the configured traffic-conditioning policy.

More advanced meters can distinguish between multiple levels of conformance, such as conforming, exceeding, and violating.

### 2. Marker

A **marker** sets or changes the DSCP value in a packet's IP header.

Marking can be performed:
- When a packet first enters a DiffServ domain.
- When traffic must be mapped to a different service class.
- In response to a meter determining that traffic exceeds its profile.

For example, a router may classify an application and mark its packets with DSCP AF31.

If a meter determines that some packets exceed the permitted traffic profile, the policy may remark those packets to AF33.

The packets remain in the same AF forwarding class but receive a higher drop precedence.

Alternatively, excess packets might be remarked to DSCP 0 for best-effort treatment.

**Remarking allows excess traffic to be forwarded with different QoS treatment rather than necessarily being dropped.**

See [DSCP Marking and Remarking with MQC](../marking/mqc-marking.md).

### 3. Shaper

A **shaper** delays packets to bring traffic into compliance with a configured traffic profile.

When traffic temporarily exceeds the configured rate, the shaper stores excess packets in a buffer and transmits them later.

For example, suppose traffic arrives at 15 Mbps, but the configured shaping rate is 10 Mbps.

The shaper can temporarily buffer excess traffic and transmit packets according to the 10 Mbps shaping policy.

```text
Incoming Traffic
     15 Mbps
        |
        v
   +----------+
   |  Shaper  |
   |  Buffer  |
   +----------+
        |
        v
  Shaped Output
     10 Mbps
```

Shaping reduces the rate at which traffic is transmitted, but it can introduce additional delay.

If the buffer becomes full, packets may still be dropped.

**Shaping generally controls the traffic departure rate by buffering excess packets instead of immediately discarding them.**

### 4. Dropper

A **dropper** discards packets according to the configured traffic-conditioning policy.

For example, a meter might determine that certain packets exceed a permitted traffic profile.

The dropper can discard those packets to prevent excess traffic from entering the DiffServ domain.

This is a common implementation of **traffic policing**.

```text
 Incoming Packets
        |
        v
   +---------+
   |  Meter  |
   +---------+
        |
   +----+----+
   |         |
Conform    Exceed
   |         |
   v         v
Forward     Drop
```

Unlike shaping, policing does not ordinarily buffer excess packets for later transmission.

However, policing policies can also be configured to **remark** excess packets instead of dropping them.

| Mechanism | Typical treatment of excess traffic |
|---|---|
| **Policing** | Drops or remarks excess packets. |
| **Shaping** | Buffers and delays excess packets. |

See [4.4.e Policing and Shaping](../policing-shaping/overview.md) for token bucket operation, policing models, and Cisco IOS XE configuration.

---

## Traffic Profiles

A **traffic profile** defines the characteristics that traffic must satisfy to be considered conforming.

Traffic profiles commonly use a **token bucket** to measure traffic rates and permitted bursts.

For example, a traffic profile might specify:

| Parameter | Example |
|---|---|
| **Traffic class** | Business applications |
| **Committed rate** | 10 Mbps |
| **Permitted burst** | 1 MB |
| **Conforming action** | Forward with DSCP AF31 |
| **Exceeding action** | Remark to DSCP 0 |

In this example, the meter uses the configured rate and burst allowance to determine whether packets are in-profile.

Packets that conform to the traffic profile retain AF31 marking.

Packets that exceed the profile can be remarked to best effort.

The receiving network may therefore continue forwarding excess traffic, but without providing the same preferential treatment.

The configured average rate does not imply that every instantaneous traffic rate above 10 Mbps is nonconforming. The permitted burst size allows short bursts above the committed rate.

The detailed behavior of token bucket meters is covered in [Token Bucket Fundamentals](../policing-shaping/token-bucket.md).

---

## Service Agreements and Traffic Conditioning

DiffServ domains can coordinate how traffic is handled through service agreements.

The original DiffServ architecture defines two related concepts:

| Term | Description |
|---|---|
| **Service Level Agreement (SLA)** | An agreement describing the forwarding service provided to a customer or another network. |
| **Traffic Conditioning Agreement (TCA)** | Defines classification rules, traffic profiles, and the conditioning actions applied to selected traffic. |

An **SLA** describes the service a network agrees to provide.

A **TCA** describes the traffic-conditioning rules used to support that service.

For example, a customer may purchase a service that provides preferential treatment for a limited amount of voice traffic.

The SLA may define the service expectations, while the TCA specifies which packets qualify and how excess traffic is handled.

### Service Level Specification (SLS) and Traffic Conditioning Specification (TCS)

[RFC 3260](https://www.rfc-editor.org/rfc/rfc3260) introduces two additional terms to distinguish technical specifications from broader agreements.

| Term | Description |
|---|---|
| **Service Level Specification (SLS)** | Technical parameters defining the service offered to a traffic stream. |
| **Traffic Conditioning Specification (TCS)** | Technical parameters defining classification rules and a traffic profile. |

The **TCS is part of the SLS**.

An SLA or TCA may include contractual and business terms, whereas the SLS and TCS describe technical service parameters.

These distinctions are useful when interpreting the DiffServ architecture and service-provider QoS agreements.

---

## Example: Traffic Conditioning Between Domains

Consider an enterprise connected to a service provider:

```text
Enterprise                  Service Provider
DS Domain                     DS Domain

+----------+          +--------------------------+
|          |          |                          |
|   CE1 ---+----------+--- PE1 --- P1 --- PE2    |
|          |          |     |                    |
+----------+          +--------------------------+
                            |
                     Provider Ingress
                     Traffic Conditioning
```

Suppose the service agreement allows the customer to send **up to 2 Mbps of voice traffic** for preferential forwarding treatment, subject to an agreed burst allowance.

The provider might implement the following policy:

| Condition | Action |
|---|---|
| Voice traffic within the agreed profile | Forward with DSCP EF (46). |
| Voice traffic exceeding the agreed profile | Drop or remark to a lower service class. |
| Other traffic | Classify and process according to the agreed service policy. |

### Packet Processing

1. **CE1** sends voice traffic toward the service provider, marked DSCP EF.
2. **PE1** identifies traffic belonging to the customer's voice class.
3. **PE1** measures the traffic against the agreed 2 Mbps profile.
4. **Conforming packets** retain their EF treatment.
5. **Excess packets** are dropped or remarked according to the provider's policy.
6. **P1** forwards admitted traffic using the PHBs configured within the provider domain.
7. **PE2** forwards traffic toward the destination, performing any required egress conditioning.

This illustrates the distinction between **traffic conditioning** and **PHB-based forwarding**.

Traffic conditioning determines whether traffic complies with the network's policy and what markings or actions it receives.

PHB-based forwarding determines how admitted packets are treated as they traverse the network.

### Why Traffic Conditioning Matters

Suppose a customer is allowed to send 2 Mbps of EF traffic but instead sends 20 Mbps marked EF.

Without traffic conditioning, excessive EF traffic could consume resources intended for other customers or service classes.

An ingress policer can enforce the permitted traffic profile by dropping or remarking excess packets.

This protects the resources allocated to EF traffic and helps the provider deliver consistent service to other traffic classes.

---

## Traffic Conditioning and Trust Boundaries

A **QoS trust boundary** determines where the network begins accepting DSCP markings from another device or network.

Traffic conditioning and trust boundaries are closely related but serve different purposes.

| Concept | Purpose |
|---|---|
| **Trust