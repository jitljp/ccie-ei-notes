
# 4.4.h End-to-End Layer 3 QoS Using MQC

**End-to-end Layer 3 Quality of Service (QoS)** combines classification, marking, policing, shaping, congestion management, and congestion avoidance to provide consistent traffic treatment across a network.

The preceding sections examined these mechanisms individually. In practice, an effective QoS deployment requires them to work together.

Cisco IOS XE uses the **Modular QoS Command-Line Interface (MQC)** to configure these mechanisms through a common policy framework.

The objective is to ensure that applications receive appropriate forwarding treatment throughout the network, particularly at points of congestion.

---

## Combining QoS Mechanisms

An end-to-end QoS design typically involves several stages.

| Stage | Mechanism | Purpose |
|---|---|---|
| 1 | Classification | Identify traffic based on applications, protocols, addresses, or existing markings |
| 2 | Marking | Assign DSCP values to indicate the intended forwarding treatment |
| 3 | Policing | Enforce traffic rate limits, potentially dropping or remarking excess traffic |
| 4 | Shaping | Control transmission rates to prevent overwhelming downstream links |
| 5 | Congestion management | Use LLQ and CBWFQ to allocate bandwidth and prioritize traffic |
| 6 | Congestion avoidance | Use WRED or ECN to manage congestion before queues become full |

These stages do not necessarily occur on the same device.

For example:

- An access router may **classify and mark** traffic.
- A WAN edge router may **shape and queue** traffic.
- A service provider may **police** traffic according to a service agreement.
- Downstream routers may use **DSCP markings** to determine the appropriate forwarding treatment.

**QoS policies do not need to be identical on every device.** However, the network must consistently interpret traffic classes and their intended forwarding treatment.

---

## Example End-to-End QoS Deployment

Consider the following network:

```text
 Voice / Data
    Hosts
      |
      |
     R1
      |
      | Classification
      | DSCP Marking
      | Trust Boundary
      |
      v
     R2
      |
      | HQoS
      | - Parent Shaping: 20 Mbps
      | - LLQ: Voice (EF)
      | - CBWFQ: Business (AF31)
      | - WRED: Eligible Data
      |
      | WAN Connection
      v
     R3
      |
      | DSCP-Based Classification
      | Additional QoS if Required
      |
      v
   Remote LAN
```

### R1: Classification and Marking

R1 identifies different traffic types and assigns appropriate DSCP values.

For example:

| Traffic | DSCP | Intended treatment |
|---|---|---|
| Voice | EF (46) | Low delay and jitter |
| Business applications | AF31 (26) | Assured forwarding |
| Best-effort data | CS0 (0) | Ordinary forwarding |

R1 establishes a **trust boundary**, ensuring that untrusted hosts cannot arbitrarily assign high-priority markings.

After marking, downstream routers can classify traffic using DSCP rather than repeating complex application classification.

See [Classification and Trust Boundary](../classification/overview.md) and [Marking DSCP Values](../marking/overview.md).

### R2: Shaping and Congestion Management

R2 connects to a WAN service with a committed rate of 20 Mbps.

It uses **Hierarchical QoS (HQoS)**:

- **Parent policy:** Shapes aggregate traffic to 20 Mbps.
- **Child policy:** Uses LLQ to prioritize voice traffic.
- **Child policy:** Uses CBWFQ to allocate bandwidth to business applications.
- **Child policy:** Uses WRED for congestion avoidance in eligible nonpriority queues.

When the total offered traffic exceeds 20 Mbps, the parent shaper creates a controlled scheduling bottleneck.

The child policy determines how the available transmission opportunities are distributed among traffic classes.

Voice receives preferential scheduling, while other classes receive bandwidth according to their configured scheduling parameters.

See [Congestion Management and Avoidance](../congestion/overview.md) and [Hierarchical QoS](../hqos/overview.md).

### R3: Consistent Forwarding Treatment

R3 receives packets that retain their DSCP markings, assuming intermediate devices have not modified them.

It can use these markings to classify traffic and apply appropriate QoS actions where necessary.

For example, if R3 forwards traffic over another congested connection, it can use LLQ and CBWFQ to provide differentiated forwarding treatment.

If there is no congestion or other requirement for traffic conditioning, additional queuing policies may be unnecessary.

**DSCP marking alone does not guarantee preferential treatment.** The forwarding devices must implement appropriate QoS mechanisms.

---

## MQC as a Common Configuration Framework

MQC provides a consistent way to combine these mechanisms.

| Component | Function |
|---|---|
| `class-map` | Defines traffic classification criteria |
| `policy-map` | Defines the actions applied to traffic classes |
| `service-policy` | Attaches a policy to an interface or other supported attachment point |

A policy map can perform different actions depending on its purpose and attachment direction.

For example:

**Ingress policies** commonly perform:

- Classification.
- DSCP marking or remarking.
- Traffic policing.

**Egress policies** commonly perform:

- Traffic shaping.
- LLQ and CBWFQ.
- Queue limit enforcement.
- WRED or ECN.

Hierarchical policies combine multiple policy maps, typically using parent shaping and child queuing.

Supported QoS actions, policy combinations, and attachment directions depend on the IOS XE platform.

---

## Important End-to-End QoS Principles

An effective end-to-end QoS design follows several principles.

1. **Classify and mark traffic near its source.** This reduces the need for complex classification throughout the network.

2. **Establish appropriate trust boundaries.** Do not automatically trust DSCP markings from untrusted endpoints.

3. **Maintain consistent DSCP interpretation.** Devices should provide forwarding treatment consistent with the network's QoS design.

4. **Apply congestion management at potential bottlenecks.** Prioritization matters most when traffic competes for limited transmission capacity.

5. **Use shaping and policing appropriately.** Shaping controls transmission rates by buffering traffic, while policing enforces traffic profiles through immediate actions.

6. **Protect nonpriority traffic.** Strict priority traffic should be appropriately controlled to prevent starvation of other queues.

7. **Consider both traffic directions.** QoS policies applied in one direction do not automatically control traffic in the reverse direction.

8. **Verify actual behavior.** Correct configuration does not necessarily mean traffic matches the intended classes or receives the expected treatment.

QoS cannot eliminate congestion or create additional bandwidth. It determines how network resources are allocated when traffic competes for them.

---

## End-to-End QoS Verification

Verification must confirm both the **classification decisions** and the **resulting forwarding behavior**.

Important IOS XE commands include:

```text
R1# show class-map
R1# show policy-map
R1# show policy-map interface
R1# show policy-map interface GigabitEthernet2
R1# show interfaces GigabitEthernet2
```

The most important command is:

```text
R1# show policy-map interface
```

Depending on the platform and configured policy, it can display:

- Packets and bytes matching each class.
- Configured marking actions.
- Policing statistics and drops.
- Shaping parameters and statistics.
- Queue depth and packet drops.
- Bandwidth allocations and scheduling information.
- WRED statistics.

Verification should be performed while generating traffic that exercises the relevant classes and creates congestion where necessary.

For example, configuring LLQ does not prove that voice receives priority. The policy must be tested under congestion to observe its scheduling behavior.

---

## Topics Covered in This Section

This final QoS section combines the mechanisms covered throughout Section 4.4 into a complete network design.

- **[End-to-End QoS Design](design.md):** Examines how classification, marking, trust boundaries, policing, shaping, and congestion management work together across a network.

- **[Complete End-to-End MQC Configuration](configuration.md):** Provides a practical configuration and verification lab using multiple routers, with cumulative tasks covering the major QoS mechanisms.

The goal is to move beyond configuring individual QoS features and understand how to **design, implement, verify, and troubleshoot a complete Layer 3 QoS deployment**.
