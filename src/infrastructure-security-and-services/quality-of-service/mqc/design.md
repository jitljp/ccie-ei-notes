
# End-to-End QoS Design

An **end-to-end Quality of Service (QoS) design** coordinates multiple QoS mechanisms across a network to provide consistent forwarding treatment for different traffic classes.

Rather than applying every QoS feature to every interface, the goal is to determine:

- **Where** traffic should be classified and marked.
- **Which** DSCP values should represent each traffic class.
- **Where** policing and shaping should be applied.
- **Which interfaces** require congestion management and avoidance.
- **How** traffic should be treated as it crosses different network devices and QoS domains.
- **How** to verify that the intended treatment is actually being provided.

Cisco's **Modular QoS CLI (MQC)** provides a common configuration framework for implementing these policies.

---

## Example Network

Consider a network with three routers connecting two sites.

```text
  Site A                                        Site B
+---------+                                   +---------+
| Hosts   |                                   | Hosts   |
| Voice   |                                   | Voice   |
| Data    |                                   | Data    |
+----+----+                                   +----+----+
     |                                             |
     | 1 Gbps                                      |
     |                                             |
    R1 ---------------- R2 --------------------- R3
          WAN Link 1            WAN Link 2
          20 Mbps CIR           50 Mbps CIR
          1 Gbps Physical       1 Gbps Physical
```

The WAN links have physical interface speeds of 1 Gbps, but their contracted service rates are lower.

| Link | Physical speed | Service rate |
|---|---|---|
| R1 to R2 | 1 Gbps | 20 Mbps |
| R2 to R3 | 1 Gbps | 50 Mbps |

The network carries four traffic classes:

| Class | DSCP | Requirement |
|---|---|---|
| VOICE | EF (46) | Low delay and jitter |
| BUSINESS | AF31 (26) | Minimum bandwidth allocation |
| BULK | CS1 (8) | Low-priority, delay-tolerant traffic |
| DEFAULT | CS0 (0) | Best-effort forwarding |

The objective is to provide appropriate forwarding treatment while preventing important applications from being overwhelmed by bulk transfers and ordinary data traffic.

---

## Step 1: Establish the Trust Boundary

The first design decision is determining **where the network should trust QoS markings**.

At Site A, hosts connect to R1 through an access network.

Some devices may generate appropriate DSCP markings, while others may send incorrect or unauthorized markings.

R1 must not automatically assume that every incoming marking is valid.

A suitable design is:

```text
        Untrusted Hosts
               |
               v
       +----------------+
       |       R1       |
       |                |
       | Classification |
       | DSCP Marking   |
       | Policing       |
       +----------------+
               |
               | Trusted QoS Domain
               v
              R2
               |
               v
              R3
```

At R1:

1. Identify traffic using appropriate classification criteria.
2. Preserve valid markings from trusted sources when appropriate.
3. Remark untrusted traffic according to the enterprise's QoS policy.
4. Optionally police traffic classes that require rate enforcement.

After R1 has assigned the appropriate markings, downstream devices can classify packets using DSCP.

**Detailed classification is generally performed near the network edge, while simpler DSCP-based classification is used throughout the trusted network.**

See [QoS Trust Boundaries](../classification/trust-boundaries.md).

---

## Step 2: Define a Consistent Marking Scheme

The network should use a consistent mapping between traffic types and DSCP values.

For example:

| Traffic | DSCP | Forwarding treatment |
|---|---|---|
| Voice bearer | EF | Strict priority |
| Business applications | AF31 | Bandwidth allocation |
| Bulk transfers | CS1 | Lower-priority bandwidth queue |
| Ordinary data | CS0 | Best effort |

R1 classifies and marks traffic entering from Site A.

R2 and R3 use these markings to identify the corresponding traffic classes.

### DSCP-to-PHB Mapping

The DSCP value identifies the intended **Per-Hop Behavior (PHB)**.

However, the marking does not independently implement the PHB.

For example:

```text
     R1                    R2                  R3
      |                     |                   |
      |                     |                   |
      | DSCP EF             | DSCP EF           |
      +-------------------->+------------------>+
      |                     |                   |
      |                     |                   |
      |             Priority Queue       Priority Queue
      |             (if configured)      (if configured)
```

R2 and R3 must have appropriate queuing policies if EF traffic is to receive strict priority treatment during congestion.

If R2 does not implement differentiated queuing, EF-marked packets do not automatically receive preferential scheduling.

### Crossing QoS Domains

Different administrative networks may use different DSCP-to-PHB mappings.

For example, a service provider may use different markings from those used by the enterprise.

At the boundary, traffic may need to be remarked to the codepoints used by the receiving domain.

When it returns to the enterprise network, another mapping may be necessary.

**End-to-end QoS requires consistent forwarding treatment, not necessarily identical DSCP values or configurations in every domain.**

See [DiffServ Domains and Traffic Conditioning](../diffserv/domains.md).

---

## Step 3: Identify Potential Congestion Points

Congestion management is most important at interfaces where the offered traffic can exceed the available transmission capacity.

In the example network:

```text
Site A                                             Site B
   |                                                  |
   R1 ---------------- R2 -------------------------- R3
        20 Mbps              50 Mbps
        Service              Service
          ^
          |
     Bottleneck for
     traffic from R1
```

For traffic traveling from Site A to Site B, WAN Link 1 has the lower service rate.

Assuming no other restrictions or competing traffic, it is the primary bottleneck.

For example:

```text
Offered traffic from Site A:

VOICE       1.5 Mbps
BUSINESS   10.0 Mbps
BULK       12.0 Mbps
DEFAULT     5.0 Mbps
---------------------
TOTAL      28.5 Mbps

WAN Link 1 service rate = 20 Mbps
```

The offered traffic exceeds the available 20 Mbps by 8.5 Mbps.

Therefore, R1 needs to control its transmission rate and determine how traffic classes share the available capacity.

### Not Every Interface Needs the Same Policy

WAN Link 2 supports 50 Mbps.

If it receives only the 20 Mbps of traffic forwarded by R1, it is unlikely to experience sustained congestion.

However, if R2 also forwards traffic from other sources onto WAN Link 2, the total offered traffic may exceed 50 Mbps.

In that case, QoS on R2's egress interface becomes important.

**Apply congestion management where traffic can compete for limited bandwidth, rather than automatically applying identical policies to every interface.**

---

## Step 4: Design the WAN Egress Policy

R1 must control traffic transmitted toward R2.

Because the physical interface operates at 1 Gbps while the contracted service rate is 20 Mbps, **Hierarchical QoS (HQoS)** is appropriate.

The design uses:

- **Parent policy:** Shape aggregate traffic to 20 Mbps.
- **Child policy:** Apply LLQ, CBWFQ, and congestion avoidance.

```text
                R1
                 |
                 v
       +---------------------+
       |    Child Policy     |
       |                     |
       |  VOICE    -> LLQ    |
       |  BUSINESS -> CBWFQ  |
       |  BULK     -> CBWFQ  |
       |  DEFAULT  -> CBWFQ  |
       |                     |
       |  WRED where useful  |
       +---------------------+
                 |
                 v
       +---------------------+
       |    Parent Policy    |
       |                     |
       |  Shape: 20 Mbps     |
       +---------------------+
                 |
                 v
           WAN Link 1
```

### Priority Traffic

Voice traffic should receive strict priority treatment to minimize queuing delay and jitter.

For example:

```text
VOICE priority allocation = 10%
Parent shaping rate       = 20 Mbps

Configured priority rate  = 2 Mbps
```

On IOS XE platforms using conditional priority policing, this rate is enforced during congestion.

If the voice traffic requires only 1.5 Mbps, it can be transmitted within its priority allocation.

However, the priority rate must be selected according to the expected voice traffic load.

Allocating too little priority bandwidth can result in voice packet drops during congestion.

Allocating too much can reduce the resources available to other traffic classes.

### Business Applications

Business applications should receive a minimum bandwidth allocation.

For example:

```text
BUSINESS minimum allocation = 30%
Parent shaping rate         = 20 Mbps

Minimum allocation          = 6 Mbps
```

This establishes a minimum scheduling allocation of 6 Mbps within the child policy, subject to the platform's scheduler behavior.

**A bandwidth allocation is not a maximum rate.**

BUSINESS may receive more than 6 Mbps when additional capacity is available.

### Bulk and Best-Effort Traffic

Bulk transfers and ordinary data can use regular bandwidth queues.

Bulk traffic can receive a lower scheduling weight than business traffic.

WRED may be appropriate for eligible TCP-dominated queues to signal congestion before their queues become full.

WRED is generally not appropriate for a strict priority voice queue.

See [Hierarchical Shaping and Queuing](../hqos/shaping-queuing.md).

---

## Step 5: Decide Where to Police Traffic

**Policing** enforces traffic profiles by applying immediate actions to packets that exceed configured limits.

It is often useful at trust boundaries, customer boundaries, or other points where traffic must comply with a defined rate.

For example, R1 might enforce a maximum rate for a specific type of traffic entering from Site A.

```text
    Site A
      |
      v
     R1
      |
   Ingress
   Policer
      |
      v
   Egress
   HQoS
      |
      v
     R2
```

The two mechanisms have different objectives:

| Mechanism | Design objective |
|---|---|
| Ingress policing | Enforce a traffic profile before forwarding traffic |
| Egress shaping | Control transmission rate to a downstream connection |
| Egress queuing | Determine which traffic receives transmission opportunities |

Policing and shaping are not interchangeable.

For example, policing all incoming traffic to 20 Mbps could cause immediate packet loss even if the excess traffic could have been buffered and transmitted later.

Shaping is generally preferred when controlling traffic toward a lower-rate WAN service, while policing is appropriate when a traffic contract or other requirement demands immediate enforcement.

### Provider Policing

The service provider may also police traffic to the contracted rate.

Shaping enterprise traffic before it enters the provider network can reduce the likelihood of provider policer drops.

However, matching the configured shaper rate exactly to the provider CIR does not always guarantee that no packets will be dropped.

Differences in traffic accounting, encapsulation overhead, token bucket parameters, and burst allowances can affect the result.

These factors should be considered when choosing the shaping rate and burst parameters.

---

## Step 6: Consider Both Traffic Directions

QoS policies are directional.

A policy applied to R1's WAN egress interface controls traffic leaving R1 toward R2.

It does not automatically control traffic traveling in the reverse direction.

```text
                Forward Direction

    R1 ---------------------------------> R2
       R1 Egress QoS

                Reverse Direction

    R1 <--------------------------------- R2
                         R2 Egress QoS
```

For example:

- R1's egress policy controls traffic from Site A toward Site B.
- R2's egress policy toward R1 controls traffic traveling toward Site A.
- R3's egress policy may be required for traffic traveling from Site B toward R2.

If the WAN service provides 20 Mbps in each direction, the appropriate transmitting devices must independently enforce that rate.

The presence of an output QoS policy on one router does not control the remote router's transmission behavior.

**End-to-end QoS must account for both forward and reverse traffic paths.**

This is particularly important when asymmetric routing causes packets in opposite directions to traverse different devices.

---

## Step 7: Verify End-to-End Behavior

A successful QoS design must be verified under realistic traffic conditions.

The following checks are particularly important.

### Classification and Marking

Verify that traffic matches the intended classes and receives the correct DSCP values.

```text
R1# show policy-map interface
```

Check:

- Class match counters.
- Marking actions.
- Unexpected matches in `class-default`.

If traffic matches the wrong class, downstream QoS treatment may also be incorrect.

### Shaping and Queuing

Verify the egress HQoS policy:

```text
R1# show policy-map interface GigabitEthernet0/1
```

Check:

- Parent shaping rate.
- Child class bandwidth allocations.
- Priority queue statistics.
- Queue depth and drops.
- WRED statistics, where applicable.

The exact counters depend on the IOS XE platform.

### Verify Under Congestion

QoS behavior should be tested when the offered load exceeds the available bandwidth.

For example, generate more than 20 Mbps of aggregate traffic toward WAN Link 1.

Then verify that:

1. R1's aggregate transmission rate is controlled by the parent shaper.
2. VOICE receives the intended priority treatment.
3. BUSINESS receives its configured scheduling allocation.
4. BULK and DEFAULT share available capacity according to the policy.
5. Queue drops and congestion-avoidance behavior are consistent with the design.

Class matching counters alone do not prove that congestion management is operating correctly.

For example, a VOICE class may match thousands of packets without the priority scheduler ever having to resolve contention between queues.

Testing during congestion is necessary to observe that behavior.

### Verify Across Multiple Devices

On each relevant device, verify:

- Traffic retains the intended DSCP marking, or is correctly remarked at a domain boundary.
- Ingress and egress policies are attached to the correct interfaces.
- Traffic is classified into the intended forwarding classes.
- Shaping and queuing policies operate at the intended congestion points.
- There are no unexpected policing or queue drops.

Packet captures can help confirm DSCP values at different locations.

Actual application performance should also be considered, particularly latency, jitter, and loss for real-time traffic.

---

## Common End-to-End QoS Design Mistakes

| Mistake | Consequence |
|---|---|
| Trusting all endpoint DSCP markings | Untrusted applications may receive unintended priority |
| Marking traffic without configuring appropriate PHBs | Markings may not produce the intended forwarding treatment |
| Applying QoS only on ingress | The intended egress scheduling behavior may be missing |
| Shaping without differentiated queuing | The aggregate rate is controlled, but important applications may experience excessive delay |
| Allocating excessive bandwidth to the priority queue | Other classes may receive insufficient resources |
| Using policing when shaping is appropriate | Excess traffic may be dropped unnecessarily |
| Assuming QoS is automatically bidirectional | Reverse-direction traffic may remain unprotected |
| Ignoring provider QoS policies | Packets may be remarked or dropped by the provider |
| Testing without congestion | Queuing behavior may never be exercised |

---

## Design Summary

An end-to-end MQC deployment typically follows this process:

```text
1. Identify Application Requirements
                  |
                  v
2. Define Traffic Classes and DSCP Values
                  |
                  v
3. Establish Trust Boundaries
                  |
                  v
4. Classify and Mark Traffic
                  |
                  v
5. Identify Bottlenecks
                  |
                  v
6. Design Policing and Shaping
                  |
                  v
7. Design LLQ, CBWFQ, and WRED
                  |
                  v
8. Apply Policies at Appropriate Interfaces
                  |
                  v
9. Verify Under Realistic Congestion
```

The central principle is that **QoS must be designed across the entire forwarding path, but each device only needs the mechanisms appropriate to its role**.

Classification and traffic conditioning are typically concentrated at network boundaries, while congestion management is deployed where competing traffic requires differentiated forwarding treatment.

The next page, [Complete End-to-End MQC Configuration](configuration.md), combines these principles into a practical multi-router configuration and verification lab.
