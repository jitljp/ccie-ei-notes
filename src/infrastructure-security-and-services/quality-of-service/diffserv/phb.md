# Per-Hop Behaviors (BE, EF, AF, CS)

A **Per-Hop Behavior (PHB)** defines the forwarding treatment that a DiffServ node applies to a **Behavior Aggregate (BA)**.

A Behavior Aggregate is a collection of packets with the same DSCP value crossing a link in a particular direction.

When a router receives a packet, it can examine the packet's **Differentiated Services Code Point (DSCP)** and use that value to select the appropriate PHB.

A PHB can influence:
- **Queuing:** Which queue receives the packet.
- **Scheduling:** How much forwarding service the queue receives.
- **Buffer allocation:** How much buffering is available to the traffic class.
- **Congestion avoidance:** How likely a packet is to be dropped when congestion develops.

A PHB describes the forwarding behavior, not the specific mechanism used to implement it.

For example, a router might implement a low-delay PHB using a **strict-priority queue**, while another PHB receives a guaranteed bandwidth allocation through **Class-Based Weighted Fair Queuing (CBWFQ)**.

The four principal PHB categories covered in this section are:

| PHB | Description | DSCP |
|---|---|---|
| **Best Effort (BE)** | Ordinary forwarding without preferential QoS treatment. | 0 |
| **Expedited Forwarding (EF)** | Designed for low delay, low jitter, and low packet loss. | 46 |
| **Assured Forwarding (AF)** | Four forwarding classes with three drop precedence levels each. | 12 standardized values |
| **Class Selector (CS)** | Provides forwarding classes compatible with the original IP Precedence model. | CS0 through CS7 |

---

## DSCP and PHB Mapping

The **DSCP** field is six bits long and is carried in the IPv4 and IPv6 headers.

Each DSCP value can be mapped to a specific PHB.

```text
        IP Packet
            |
            v
      Examine DSCP
            |
            v
       Select PHB
            |
            v
    Queuing / Scheduling
    Congestion Avoidance
            |
            v
      Forward Packet
```

**The DSCP value does not directly specify which queue or scheduling algorithm the router must use.**

Instead, the device maps the DSCP to an appropriate forwarding behavior according to its QoS policy.

For example:
- DSCP 46 (EF) may be mapped to a strict-priority queue.
- DSCP 26 (AF31) may be mapped to a bandwidth-allocated queue.
- DSCP 48 (CS6) may be mapped to a queue reserved for network-control traffic.
- DSCP 0 (BE) may be mapped to an ordinary best-effort queue.

A packet's DSCP marking does not automatically guarantee that the device provides the intended forwarding treatment.

---

## Best Effort (BE)

**Best Effort (BE)**, also called the **Default PHB**, provides ordinary IP forwarding without special QoS treatment.

It is defined in [RFC 2474](https://www.rfc-editor.org/rfc/rfc2474).

| Field | Value |
|---|---|
| **DSCP name** | BE / Default / CS0 |
| **Decimal** | 0 |
| **Binary** | `000000` |

Best-effort traffic receives ordinary forwarding service without specific guarantees regarding bandwidth, delay, jitter, or packet loss.

For example, ordinary web browsing and noncritical file transfers may be assigned to the best-effort class.

### Best-Effort Queuing

Best Effort does not require a particular queuing mechanism.

One possible implementation is **First-In, First-Out (FIFO)** queuing, where packets are transmitted in the order they enter the queue.

In a differentiated QoS policy, best-effort traffic can instead receive its own class-based queue.

```text
EF Traffic ---- Priority Queue --+
                                 |
AF Traffic ---- AF Queue --------+----> Output
                                 |
BE Traffic ---- BE Queue --------+
```

**Best Effort does not mean that packets must always be dropped first or that the class receives no guaranteed bandwidth.**

A network can allocate a minimum bandwidth share to the default class to prevent starvation.

---

## Expedited Forwarding (EF)

**Expedited Forwarding (EF)** is designed to provide forwarding treatment suitable for traffic requiring:
- **Low delay**
- **Low jitter**
- **Low packet loss**
- **An assured forwarding rate**

EF is defined in [RFC 3246](https://www.rfc-editor.org/rfc/rfc3246).

| Field | Value |
|---|---|
| **DSCP name** | EF |
| **Decimal** | 46 |
| **Binary** | `101110` |

EF is commonly used for **VoIP bearer traffic**, which is particularly sensitive to delay and jitter.

### EF Forwarding Behavior

The EF PHB requires packets to receive forwarding service at a configured rate, independently of the amount of non-EF traffic competing for the same output interface.

For example, a router may provide an EF service rate of 10 Mbps on a 100 Mbps WAN link.

The router must provide the specified EF forwarding service even when other traffic classes are heavily loaded.

This helps prevent competing traffic from causing excessive delay or jitter for EF packets.

**EF does not mean unlimited priority bandwidth.**

The network must provision sufficient resources for EF traffic and control the amount admitted into the class.

### Strict-Priority Queuing

A common implementation of EF is **Low Latency Queuing (LLQ)**, which uses a strict-priority queue.

```text
EF Traffic ---- Priority Queue --+
                                 |
AF Traffic ---- AF Queue --------+----> Output
                              |
BE Traffic ---- BE Queue --------+
```

When EF packets are waiting in the priority queue, the scheduler services them ahead of nonpriority queues, subject to the configured scheduling and rate controls.

This minimizes the time EF packets spend waiting behind other traffic.

### Why EF Traffic Must Be Controlled

Strict-priority queuing introduces a potential problem: **starvation**.

If a priority queue continuously consumes all available bandwidth, lower-priority queues may not receive sufficient forwarding opportunities.

Therefore, EF traffic is commonly controlled using traffic policing or admission control.

For example:

| Traffic class | Policy |
|---|---|
| EF voice | Priority service with a 10 Mbps allocation |
| AF business traffic | Guaranteed bandwidth allocation |
| BE traffic | Remaining bandwidth |

Cisco IOS XE can implement LLQ using the `priority` command:

```text
R1(config)# class-map match-all VOICE
R1(config-cmap)# match dscp ef
!
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class VOICE
R1(config-pmap-c)# priority percent 10
```

The `priority percent 10` command configures priority queuing with a bandwidth allocation of 10% of the interface bandwidth.

See [Low Latency Queuing (LLQ)](../congestion/llq.md) for detailed operation and configuration.

---

## Assured Forwarding (AF)

**Assured Forwarding (AF)** provides multiple levels of forwarding assurance by dividing traffic into separate forwarding classes.

AF is defined in [RFC 2597](https://www.rfc-editor.org/rfc/rfc2597).

The AF PHB group defines:
- **Four forwarding classes**, numbered 1 through 4.
- **Three drop precedence levels** within each class.

This produces **12 standardized AF codepoints**.

### AF Classes

The four AF classes are:

| Class | Codepoint group |
|---|---|
| **AF Class 1** | AF11, AF12, AF13 |
| **AF Class 2** | AF21, AF22, AF23 |
| **AF Class 3** | AF31, AF32, AF33 |
| **AF Class 4** | AF41, AF42, AF43 |

Each AF class receives an independently configured allocation of forwarding resources, such as bandwidth and buffer space.

For example, a network might allocate:
- AF Class 1: 10% of interface bandwidth.
- AF Class 2: 20% of interface bandwidth.
- AF Class 3: 30% of interface bandwidth.
- AF Class 4: 20% of interface bandwidth.

These are example allocations, not standardized requirements.

**A higher AF class number does not automatically mean higher forwarding priority.**

For example, AF41 is not inherently more important than AF31.

The actual forwarding treatment depends on the resources and scheduling behavior configured for each class.

### AF Drop Precedence

Within each AF class, packets are assigned one of three **drop precedence** levels.

| Drop precedence | Meaning |
|---|---|
| **1** | Low drop precedence |
| **2** | Medium drop precedence |
| **3** | High drop precedence |

When congestion occurs within an AF class, packets with higher drop precedence should have a greater probability of being dropped.

For example, within AF Class 3:

```text
AF31: Low drop precedence
AF32: Medium drop precedence
AF33: High drop precedence
```

The intended relationship is:

`P(drop AF33) ≥ P(drop AF32) ≥ P(drop AF31)`


These relationships apply **within the same AF class**.

Drop precedence does not independently establish a forwarding priority relationship between different AF classes.

For example, AF43 is not automatically more likely to be dropped than AF31 because those packets belong to different forwarding classes.

---

## AF Codepoint Format

AF codepoints use the format:

`AFxy`

Where:
- **x:** Forwarding class (1 through 4).
- **y:** Drop precedence (1 through 3).

For example, `AF32` identifies forwarding class 3 with medium drop precedence.

### AF DSCP Encoding

AF codepoints use the following 6-bit format:

```text
  Class     Drop      0
+---------+-------+---+
|  x x x  |  y y  | 0 |
+---------+-------+---+
  3 bits   2 bits  1
```

The first three bits identify the forwarding class, and the next two bits indicate drop precedence.

The final bit is always 0.

| AF class | First three bits |
|---|---|
| AF1x | `001` |
| AF2x | `010` |
| AF3x | `011` |
| AF4x | `100` |

| Drop precedence | Two-bit value |
|---|---|
| Low (1) | `01` |
| Medium (2) | `10` |
| High (3) | `11` |

### AF DSCP Values

The complete set of AF codepoints is:

| Drop precedence | AF1 | AF2 | AF3 | AF4 |
|---|---|---|---|---|
| **Low (1)** | AF11 (10) | AF21 (18) | AF31 (26) | AF41 (34) |
| **Medium (2)** | AF12 (12) | AF22 (20) | AF32 (28) | AF42 (36) |
| **High (3)** | AF13 (14) | AF23 (22) | AF33 (30) | AF43 (38) |

The numbers in parentheses represent decimal DSCP values.

The DSCP value can be calculated using:

`DSCP = (8 × x) + (2 × y)`

For example, AF32:

`(8 × 3) + (2 × 2) = 28`

AF32 therefore uses DSCP 28, or binary `011100`.

---

## AF Queuing and Drop Precedence

AF provides differentiated treatment through two mechanisms:

1. **Resource allocation between forwarding classes.**
2. **Drop precedence within each forwarding class.**

For example, suppose a router receives packets marked AF31, AF32, and AF33.

All three codepoints belong to AF Class 3, so they share the resources allocated to that class.

```text
AF31 ----+
         |
AF32 ----+---- AF Class 3 Queue ----> Output
         |
AF33 ----+
```

The three markings do not require three separate output queues.

Instead, the packets can share a single AF3 queue while receiving different drop probabilities.

### Weighted Random Early Detection (WRED)

**Weighted Random Early Detection (WRED)** can implement different drop probabilities within an AF class.

WRED monitors queue congestion and begins selectively dropping packets before the queue becomes full.

It can use different thresholds for different DSCP values.

| DSCP | Drop precedence | Intended WRED behavior |
|---|---|---|
| AF31 | Low | Lowest drop probability |
| AF32 | Medium | Intermediate drop probability |
| AF33 | High | Highest drop probability |

The thresholds can be configured so that AF33 packets become candidates for dropping earlier than AF32 and AF31 packets.

The actual drop behavior depends on queue occupancy and the configured WRED parameters.

**Drop precedence affects drop probability, not transmission priority within the same AF class.**

### Packet Ordering

RFC 2597 requires AF implementations to avoid reordering packets belonging to the **same microflow and AF class**, regardless of drop precedence.

For example, packets belonging to one TCP flow may be marked AF31 or AF32 according to whether they conform to an agreed traffic profile.

Changing their drop precedence must not cause them to be reordered within that AF class.

See [Weighted Random Early Detection (WRED)](../congestion/wred.md) for detailed operation and configuration.

---

## AF Traffic Conditioning Example

A common use of AF is to distinguish **in-profile** traffic from traffic that exceeds an agreed rate.

Suppose a service provider allows a customer to send 10 Mbps of traffic in AF Class 3.

A three-outcome traffic conditioner might apply the following markings:

| Traffic | Marking | Treatment |
|---|---|---|
| Conforming | AF31 | Low drop precedence |
| Exceeding | AF32 | Medium drop precedence |
| Violating | AF33 | High drop precedence |

```text
                Customer Traffic
                       |
                       v
                 Traffic Meter
                       |
          +------------+------------+
          |            |            |
       Conform       Exceed       Violate
          |            |            |
          v            v            v
        AF31         AF32         AF33
          |            |            |
          +------------+------------+
                       |
                       v
                  AF3 Queue
                       |
                       v
                     Output
```

All three codepoints belong to the same forwarding class, but their drop precedence differs.

During congestion, the router can preferentially protect packets with lower drop precedence.

The corresponding policing mechanisms are covered in [Single-Rate and Two-Rate Policing](../policing-shaping/single-two-rate.md).

---

## Class Selector (CS) PHBs

**Class Selector (CS) PHBs** provide forwarding treatment compatible with the older **IP Precedence** QoS model.

Before DiffServ introduced the 6-bit DSCP field, the IPv4 Type of Service (ToS) field contained a 3-bit IP Precedence field.

IP Precedence supported eight values, from 0 through 7.

DiffServ preserves compatibility with these values through eight **Class Selector codepoints**, CS0 through CS7.

The Class Selector PHB requirements are defined in [RFC 2474](https://www.rfc-editor.org/rfc/rfc2474).

### Class Selector Encoding

Class Selector codepoints use the following binary format:

`xxx000`

The first three bits correspond to the original IP Precedence value.

The final three bits are always 0.

```text
  IP Precedence
  3 bits     3 bits
+---------+---------+
|  x x x  |  0 0 0  |
+---------+---------+
       6-bit DSCP
```

For example:

- IP Precedence 3 corresponds to CS3.
- IP Precedence 5 corresponds to CS5.
- IP Precedence 6 corresponds to CS6.

The decimal DSCP value is calculated as:

`DSCP = CS nubmer * 8`

Therefore, CS6 has a decimal DSCP value of 48.

### Class Selector Codepoints

| Codepoint | Decimal DSCP | Binary | Common Cisco QoS use |
|---|---:|---|---|
| **CS0** | 0 | `000000` | Best Effort |
| **CS1** | 8 | `001000` | Scavenger / low-priority data |
| **CS2** | 16 | `010000` | Network management / OAM |
| **CS3** | 24 | `011000` | Call signaling |
| **CS4** | 32 | `100000` | Real-time interactive traffic |
| **CS5** | 40 | `101000` | Broadcast video |
| **CS6** | 48 | `110000` | Network control / routing protocols |
| **CS7** | 56 | `111000` | Reserved for high-priority network-control use |

These application mappings represent common Cisco QoS design recommendations, not mandatory mappings imposed by the CS codepoints themselves.

For example:
- **CS1** may be used for backups or other low-priority background traffic.
- **CS2** may be used for SNMP, SSH, and other network-management traffic.
- **CS3** may be used for SIP and other call-signaling traffic.
- **CS6** may be used for routing protocols such as OSPF, EIGRP, and BGP.

### Class Selector Forwarding Requirements

RFC 2474 defines several requirements for Class Selector PHBs.

**1. At least two independently forwarded classes**

The eight CS codepoints must collectively map to at least two independently forwarded classes of traffic.

This means a DiffServ implementation cannot simply map all eight codepoints to the same undifferentiated forwarding treatment.

**2. Relative forwarding precedence**

Under reasonable operating conditions, higher CS values should receive a probability of timely forwarding that is no worse than lower CS values.

For example, CS5 should generally receive forwarding treatment no worse than CS3.

However, **this does not mean that all eight CS values must have separate queues or strict-priority relationships**.

Multiple CS codepoints can map to the same PHB.

**3. CS6 and CS7 receive preferential treatment**

RFC 2474 explicitly requires the PHBs selected by **CS6 and CS7** to provide preferential forwarding treatment compared to the Default PHB (CS0).

This preserves the historical use of IP Precedence 6 and 7 for network-control traffic.

**4. Independent forwarding**

Different CS codepoints should be independently forwarded, although packets marked with distinct CS values may be reordered.

The forwarding behavior can be implemented using mechanisms such as weighted scheduling, class-based queuing, or strict-priority queuing.

### CS1 and Scavenger Traffic

A common enterprise QoS practice is to mark **scavenger traffic** with CS1.

Scavenger traffic is traffic that should receive a lower allocation of resources than ordinary best-effort traffic when the network is congested.

Examples include:
- Noncritical backups.
- Background file transfers.
- Other delay-tolerant, low-priority applications.

This deserves attention because CS1 has a numerically higher DSCP value than CS0.

**CS1 is commonly treated as lower priority than CS0**, despite its higher numerical value.

RFC 2474's relative-order recommendation is not an absolute requirement, and later QoS service-class recommendations use CS1 for low-priority traffic.

The network must be configured to provide this treatment. Simply marking a packet CS1 does not automatically assign it to a low-priority queue.

See [RFC 3662: A Lower Effort Per-Domain Behavior](https://www.rfc-editor.org/rfc/rfc3662) for the Lower Effort service concept.

### CS3 vs. CS5 for Call Signaling

Cisco commonly recommends **CS3 (24)** for call signaling, while [RFC 4594](https://www.rfc-editor.org/rfc/rfc4594) recommends **CS5 (40)** for its Signaling service class.

Cisco's QoS design recommendations continue to use CS3 for call signaling in many enterprise deployments.

For CCIE EI, the important point is to understand that the **actual traffic treatment follows the configured QoS policy**, not merely the application associated with a DSCP value.

### Class Selector vs. EF and AF

CS codepoints are distinct from the EF and AF codepoints, even when their binary values share the same first three bits.

For example:

| Codepoint | Decimal | Binary |
|---|---:|---|
| CS5 | 40 | `101000` |
| EF | 46 | `101110` |
| CS4 | 32 | `100000` |
| AF41 | 34 | `100010` |
| CS3 | 24 | `011000` |
| AF31 | 26 | `011010` |

**CS5 and EF are not equivalent.**

Likewise, CS4 and AF41 are not equivalent, even though they share the same three most significant DSCP bits.

Each codepoint selects a forwarding behavior according to the device's DSCP-to-PHB mapping.

---

## Cisco IOS XE PHB Implementation

Cisco IOS XE commonly implements DiffServ forwarding behavior using the **Modular QoS Command-Line Interface (MQC)**.

Typical implementations include:

| PHB | Common implementation |
|---|---|
| **BE** | Default class with ordinary queuing |
| **EF** | Low Latency Queuing (LLQ) |
| **AF** | CBWFQ with DSCP-based WRED |
| **CS** | Class-based queuing with bandwidth or priority allocations |

### Example Configuration

The following example implements separate forwarding classes for voice, network control, call signaling, business applications, scavenger traffic, and best-effort traffic.

```text
R1(config)# class-map match-all VOICE
R1(config-cmap)# match dscp ef
!
R1(config)# class-map match-all NETWORK_CONTROL
R1(config-cmap)# match dscp cs6
!
R1(config)# class-map match-all CALL_SIGNALING
R1(config-cmap)# match dscp cs3
!
R1(config)# class-map match-any AF3
R1(config-cmap)# match dscp af31 af32 af33
!
R1(config)# class-map match-all SCAVENGER
R1(config-cmap)# match dscp cs1
!
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class VOICE
R1(config-pmap-c)# priority percent 10
R1(config-pmap-c)# exit
R1(config-pmap)# class NETWORK_CONTROL
R1(config-pmap-c)# bandwidth percent 5
R1(config-pmap-c)# exit
R1(config-pmap)# class CALL_SIGNALING
R1(config-pmap-c)# bandwidth percent 5
R1(config-pmap-c)# exit
R1(config-pmap)# class AF3
R1(config-pmap-c)# bandwidth percent 25
R1(config-pmap-c)# random-detect dscp-based
R1(config-pmap-c)# exit
R1(config-pmap)# class SCAVENGER
R1(config-pmap-c)# bandwidth percent 1
R1(config-pmap-c)# exit
R1(config-pmap)# class class-default
R1(config-pmap-c)# bandwidth percent 25
!
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy output WAN-QOS
```

The policy implements the following example resource allocations:

| Class | Matching DSCP | Allocation |
|---|---|---|
| VOICE | EF | 10% priority |
| NETWORK_CONTROL | CS6 | 5% bandwidth |
| CALL_SIGNALING | CS3 | 5% bandwidth |
| AF3 | AF31, AF32, AF33 | 25% bandwidth |
| SCAVENGER | CS1 | 1% bandwidth |
| class-default | Remaining traffic | 25% bandwidth |

The bandwidth percentages are minimum allocations under congestion, not fixed transmission-rate limits. Unused bandwidth can be made available to other classes according to the scheduler.

The AF3 class uses DSCP-based WRED. Its drop thresholds should be configured appropriately to achieve the desired differences in drop precedence.

Notice that **CS6 receives a bandwidth allocation rather than strict-priority treatment**.

The purpose of its class-based allocation is to ensure that important network-control traffic receives adequate forwarding resources.

Also notice that CS1 receives a much smaller minimum bandwidth allocation than the default class. This supports its intended treatment as low-priority traffic.

### Verification

Display the policy applied to the interface:

```text
R1# show policy-map interface GigabitEthernet1
```

Useful information includes:
- Traffic classes and match counters.
- Queue bandwidth allocations.
- Priority queue statistics.
- WRED statistics.
- Queue drops.

To inspect the policy configuration independently of an interface:

```text
R1# show policy-map WAN-QOS
```

---

## Comparing BE, EF, AF, and CS

| Characteristic | BE | EF | AF | CS |
|---|---|---|---|---|
| **Purpose** | Ordinary forwarding | Low delay, jitter, loss | Assured forwarding and differentiated loss | IP Precedence compatibility |
| **DSCP values** | 0 | 46 | 12 AF codepoints | 0, 8, 16, 24, 32, 40, 48, 56 |
| **Forwarding classes** | Default | EF | Four | At least two independently forwarded classes |
| **Drop precedence** | None defined | None defined | Three per AF class | None defined |
| **Typical implementation** | Ordinary queue | Priority queue | Bandwidth queues with WRED | Class-based queues |
| **Common traffic** | General data | VoIP bearer | Business and multimedia applications | Control, signaling, management, low-priority traffic |

The key distinction between AF and CS is:

- **AF** defines independent forwarding classes and explicit drop precedence levels within each class.
- **CS** provides forwarding classes designed to preserve compatibility with IP Precedence, without defining AF-style drop precedence levels.

**EF** is different from both because its forwarding behavior is specifically designed to provide an assured service rate with low delay and jitter.

---

## Key Points

- A **PHB** defines the forwarding treatment applied to a Behavior Aggregate at a DiffServ node.
- **DSCP identifies the intended PHB**, but the device's QoS configuration determines how the packet is actually forwarded.
- **BE** uses DSCP 0 and provides ordinary best-effort forwarding.
- **EF** uses DSCP 46 and is intended for low-delay, low-jitter, low-loss traffic.
- **EF** is commonly implemented using strict-priority queuing, with controls to prevent starvation of other classes.
- **AF** defines four independent forwarding classes, each with three drop precedence levels.
- **AFxy:** `x` identifies the forwarding class; `y` identifies the drop precedence.
- **AF DSCP = (8 × class) + (2 × drop precedence)**.
- Higher AF drop precedence means greater drop probability under congestion within the same AF class.
- A higher AF class number does **not** automatically mean higher forwarding priority.
- **WRED** can provide different drop probabilities for AF packets sharing the same queue.
- **CS** provides compatibility with the older IP Precedence model.
- **CS codepoints** use the format `xxx000`, with decimal values equal to the CS number multiplied by 8.
- **CS6 and CS7** must receive preferential forwarding treatment compared to CS0.
- **CS1** is commonly used for low-priority scavenger traffic.
- **CS3** is commonly used for call signaling, while **CS6** is commonly used for network control.
- **CS5 (40) and EF (46)** are different codepoints with different PHB definitions.
- **The numeric DSCP value alone does not determine forwarding priority.**

---

## References

- [RFC 2474: Definition of the Differentiated Services Field](https://www.rfc-editor.org/rfc/rfc2474)
- [RFC 2597: Assured Forwarding PHB Group](https://www.rfc-editor.org/rfc/rfc2597)
- [RFC 3246: An Expedited Forwarding PHB](https://www.rfc-editor.org/rfc/rfc3246)
- [RFC 3662: A Lower Effort Per-Domain Behavior](https://www.rfc-editor.org/rfc/rfc3662)
- [RFC 4594: Configuration Guidelines for DiffServ Service Classes](https://www.rfc-editor.org/rfc/rfc4594)
- [Cisco: Implement QoS Policies with Differentiated Services Code Point](https://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-packet-marking/10103-dscpvalues.html)
- [Cisco: Quality of Service Considerations](https://www.cisco.com/c/en/us/td/docs/voice_ip_comm/hcs/14_0/HCS_Solution/SRND/hcs_14_0_srnd/chcs_m_quality-of-service-considerations.html)