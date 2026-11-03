# Quality of Service

**Quality of Service (QoS)** is a collection of mechanisms used to control how network devices handle different types of traffic.

By default, IP networks generally use **best-effort delivery**, meaning packets are forwarded without guarantees regarding bandwidth, delay, or packet loss.

When a network becomes congested, applications compete for limited resources. Without QoS, delay-sensitive traffic such as voice and video may experience degraded performance.

QoS allows network administrators to **classify, prioritize, mark, rate-limit, and queue traffic** according to application requirements.

---

## Why QoS Is Needed

Different applications have different network performance requirements.

For example:

- **Voice:** Sensitive to delay, jitter, and packet loss.
- **Interactive video:** Sensitive to delay, jitter, and packet loss, and often requires substantial bandwidth.
- **Business applications:** May require guaranteed bandwidth or preferential treatment during congestion.
- **File transfers and backups:** Usually tolerate delay but can consume large amounts of bandwidth.

The primary performance characteristics addressed by QoS are:

| Characteristic | Description |
|---|---|
| **Bandwidth** | The capacity available to transmit traffic |
| **Delay (Latency)** | The time required for a packet to travel from source to destination |
| **Jitter** | Variation in packet delay |
| **Packet Loss** | Packets dropped due to congestion or other network conditions |

**QoS does not increase the physical bandwidth of a link.** Instead, it controls how the available bandwidth and other resources are allocated among different traffic classes.

QoS is especially important at network congestion points, such as WAN links where traffic from high-speed LAN interfaces is forwarded over a lower-bandwidth connection.

In an ideal scenario, QoS wouldn't be necessary - there would be enough bandwidth for all applications to run on the network without having to compete for resources.

But in reality, budgets aren't infinite, and there likely will be congestion at certain points in the network. 
QoS helps to ensure that certain applications sensitive to the negative effects of congestion receive proper treatment.

---

## Fundamental QoS Mechanisms

QoS uses several mechanisms to identify traffic and determine how it should be handled.

| Mechanism | Purpose |
|---|---|
| **Classification** | Identifies traffic and assigns packets to classes based on characteristics such as IP addresses, protocols, ports, or existing markings |
| **Marking** | Sets or modifies fields such as DSCP to indicate the desired forwarding treatment, aiding in classification |
| **Policing** | Enforces traffic rate limits, typically by dropping or remarking traffic that exceeds a configured rate |
| **Shaping** | Buffers excess traffic and delays its transmission to maintain a configured average rate |
| **Congestion Management** | Uses queuing and scheduling mechanisms to determine which packets are transmitted when an interface is congested |
| **Congestion Avoidance** | Attempts to prevent excessive queue buildup by selectively dropping or marking packets before queues become full |

These mechanisms are often combined into a single QoS policy.

For example, a router might classify voice traffic, mark it with a particular DSCP value, and place it into a priority queue when transmitting packets over a congested WAN link.

---

## Differentiated Services Architecture

The **Differentiated Services (DiffServ)** architecture is the primary QoS model used in modern IP networks.

Rather than maintaining QoS state for individual flows throughout the network, DiffServ groups traffic into classes.

Packets are typically marked using the **Differentiated Services Code Point (DSCP)** field in the IPv4 or IPv6 header.

Each router can then use the DSCP value to determine the appropriate **Per-Hop Behavior (PHB)**.

Common PHBs include:

- **Best Effort (BE):** Standard forwarding without special preferential treatment.
- **Expedited Forwarding (EF):** Intended for traffic requiring low delay, low jitter, and low loss, such as voice.
- **Assured Forwarding (AF):** Provides different classes of forwarding treatment, with multiple drop precedence levels.

A consistent DiffServ deployment allows traffic to receive appropriate treatment as it passes through multiple routers and switches.

---

## QoS Trust Boundaries

A **trust boundary** determines where the network begins trusting existing QoS markings.

Packets received from an untrusted device may be classified and remarked before entering the trusted portion of the network.

For example, an access switch might trust DSCP markings received from an IP phone but not those received from a connected PC.

Establishing appropriate trust boundaries prevents endpoints from marking all their traffic as high priority and consuming resources intended for critical applications.

Ideally, traffic is classified and marked near its source, allowing downstream devices to use those markings without repeatedly performing detailed classification.

---

## Modular QoS CLI (MQC)

Cisco IOS XE uses the **Modular QoS Command-Line Interface (MQC)** framework to configure many QoS features.

MQC consists of three main components:

| Component | Purpose |
|---|---|
| **Class map** (`class-map`) | Defines the criteria used to classify traffic |
| **Policy map** (`policy-map`) | Defines the QoS actions applied to each traffic class |
| **Service policy** (`service-policy`) | Attaches a policy map to an interface or another supported attachment point |

For example, an MQC policy can classify voice traffic using DSCP EF, allocate it to a priority queue, and assign bandwidth to other traffic classes.

MQC also supports **hierarchical policies**, allowing multiple levels of QoS treatment. A parent policy might shape traffic to a specified rate while a child policy distributes bandwidth among individual traffic classes.

---

## End-to-End QoS

Effective QoS requires a coordinated design across the network.

A typical end-to-end QoS deployment involves:

1. **Classifying** traffic at the network edge.
2. **Marking** packets with appropriate DSCP values.
3. **Establishing trust boundaries** to control which markings are accepted.
4. **Applying policing or shaping** where traffic rates must be controlled.
5. **Configuring queuing and congestion avoidance** at potential congestion points.
6. **Preserving and interpreting markings consistently** throughout the network.

QoS policies do not necessarily need to be identical on every device. However, the classification and forwarding treatment should remain consistent with the intended end-to-end QoS design.

---

## Topics Covered in This Section

This section covers the QoS technologies included in the CCIE Enterprise Infrastructure blueprint.

| Section | Topic |
|---|---|
| **4.4.a** | Differentiated Services architecture |
| **4.4.b** | Classification and trust boundaries |
| **4.4.c** | Network Based Application Recognition (NBAR) |
| **4.4.d** | Marking DSCP values in IPv4 and IPv6 headers |
| **4.4.e** | Policing and shaping |
| **4.4.f** | Congestion management and avoidance |
| **4.4.g** | Hierarchical QoS (HQoS) |
| **4.4.h** | End-to-end Layer 3 QoS using MQC |

The focus is on understanding how QoS mechanisms operate, configuring them on Cisco IOS XE devices, and verifying their behavior in practical network scenarios.