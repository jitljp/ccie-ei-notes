# QoS Models and DiffServ Architecture

There are three fundamental **Quality of Service (QoS) models**:

1. **Best Effort**
2. **Integrated Services (IntServ)**
3. **Differentiated Services (DiffServ)**

These models differ in how they allocate network resources and provide forwarding treatment to different types of traffic.

| Model | Approach | Resource reservation | Scalability |
|---|---|---|---|
| **Best Effort** | All traffic receives ordinary forwarding treatment. | None | High |
| **IntServ** | Reserves resources for individual traffic flows. | Per-flow | Limited |
| **DiffServ** | Groups traffic into classes and applies different forwarding behaviors. | No per-flow reservation required | High |

**DiffServ** is the primary QoS architecture used in modern enterprise networks because it provides traffic differentiation without requiring routers to maintain individual resource reservations for every flow.

---

## Best-Effort Model

The **Best-Effort model** is the traditional IP forwarding model.

Routers attempt to forward packets toward their destinations but do not provide guarantees regarding:
- **Bandwidth:** The amount of bandwidth available to a particular flow.
- **Delay:** The time required to deliver a packet.
- **Jitter:** Variation in packet delay.
- **Packet loss:** Whether packets will be delivered successfully.

All traffic competes for the available network resources without differentiated QoS treatment.

For example, consider a router forwarding voice traffic and file transfers over a congested WAN connection.

```text
       Voice Traffic -----+
                          |
                          v
                         R1 ---------- WAN
                          ^
                          |
       File Transfers ----+
```

Without a QoS policy, the router does not distinguish voice traffic from file transfers when allocating forwarding resources.

As a result, large file transfers may consume bandwidth and cause voice packets to experience increased delay, jitter, or packet loss.

Best Effort does not inherently require a particular queuing algorithm, although **First-In, First-Out (FIFO)** queuing is a common example of best-effort scheduling.

**Best Effort is suitable when network resources are sufficient to accommodate traffic demands without requiring differentiated treatment.**

---

## Integrated Services (IntServ)

**Integrated Services (IntServ)** is a QoS architecture designed to provide specific service characteristics to **individual traffic flows**.

Unlike Best Effort, IntServ allows applications to request network resources before transmitting traffic.

Routers along the forwarding path maintain information about these requests and allocate resources accordingly.

IntServ is defined in [RFC 1633](https://www.rfc-editor.org/rfc/rfc1633).

### Per-Flow Resource Reservation

The defining characteristic of IntServ is **per-flow resource reservation**.

For example, suppose a voice application requires a certain amount of bandwidth.

```text
Voice Sender                           Receiver
     |                                    |
     R1 -------- R2 -------- R3 -------- R4
          WAN         WAN         WAN

       Resources reserved for the voice flow
       at each participating router
```

Each participating router along the path must have sufficient resources to support the requested service.

The reservation is associated with the particular flow rather than a general traffic class.

This requires routers to maintain **per-flow QoS state**, such as reservation information and traffic-control parameters.

### Resource Reservation Protocol (RSVP)

**Resource Reservation Protocol (RSVP)** is a signaling protocol commonly associated with IntServ.

RSVP allows applications to request resources along the forwarding path.

Two important RSVP messages are:

| Message | Direction | Purpose |
|---|---|---|
| **PATH** | Sender to receiver | Advertises the traffic flow and establishes path state. |
| **RESV** | Receiver to sender | Requests resource reservations along the reverse of the data path. |

The simplified process is:

1. The sender transmits an RSVP **PATH** message toward the receiver.
2. Routers along the path record information about the flow.
3. The receiver sends an RSVP **RESV** message upstream.
4. Each router performs **admission control** to determine whether sufficient resources are available.
5. Routers that accept the request establish the necessary reservation and traffic-control state.
6. Traffic receives the requested service along the successfully reserved path.

```text
            RSVP PATH
Sender -----------------------------> Receiver
       R1 ------- R2 ------- R3
          <----------------------
                RSVP RESV
```

**RSVP is receiver-oriented:** The receiver initiates the reservation request.

It is also important to distinguish RSVP from a routing protocol. RSVP does not calculate the forwarding path; it uses paths established by the network's routing mechanisms.

### Admission Control

**Admission control** determines whether a router can accept a new resource reservation without compromising existing reservations.

For example, suppose an interface has 10 Mbps of bandwidth available for reservations.

If existing flows have reserved 8 Mbps, a new request for 1 Mbps may be accepted.

A request for 4 Mbps would be rejected because insufficient reservable bandwidth remains.

Admission control therefore prevents new reservations from consuming resources already committed to other flows.

### IntServ Service Types

IntServ defines service models including:

| Service | Description |
|---|---|
| **Guaranteed Service** | Provides a mathematically bounded queuing delay for conforming traffic. |
| **Controlled-Load Service** | Provides service resembling that of a lightly loaded network, with low loss and delay under the specified conditions. |

**Guaranteed Service** is defined in [RFC 2212](https://www.rfc-editor.org/rfc/rfc2212).

**Controlled-Load Service** is defined in [RFC 2211](https://www.rfc-editor.org/rfc/rfc2211).

### IntServ Scalability Limitations

The main disadvantage of IntServ is its **limited scalability**.

Routers must maintain resource reservation state for individual flows.

For example:
- 100 flows may require 100 separate reservation states.
- 10,000 flows may require 10,000 separate reservation states.
- 1,000,000 flows may require up to 1,000,000 separate reservation states.

The number of active reservations increases the processing, memory, and signaling requirements of the routers.

RSVP also uses **soft state**, meaning reservation information must be periodically refreshed. Reservations expire if the required refresh messages are not received.

These requirements make traditional per-flow IntServ difficult to deploy across large networks.

---

## Differentiated Services (DiffServ)

**Differentiated Services (DiffServ)** addresses the scalability limitations of IntServ by providing QoS to **aggregates of traffic rather than individual flows**.

DiffServ is defined in [RFC 2475](https://www.rfc-editor.org/rfc/rfc2475).

Instead of reserving resources for each flow, DiffServ:
1. **Classifies** packets into traffic classes.
2. **Marks** packets using the Differentiated Services Code Point (DSCP).
3. **Applies forwarding behaviors** based on those markings.
4. **Allocates network resources** among the traffic classes.

For example, a network may divide traffic into three categories:

| Traffic | DSCP | Forwarding treatment |
|---|---|---|
| Voice | EF (46) | Low-delay forwarding |
| Business applications | AF31 (26) | Assured forwarding |
| File transfers | BE (0) | Best-effort forwarding |

Hundreds or thousands of individual flows can share the same class and forwarding treatment.

**DiffServ does not require routers to maintain resource reservations for individual flows.**

This makes it considerably more scalable than traditional IntServ.

---

## DiffServ Domains

A **DiffServ domain** is a contiguous collection of DiffServ-capable network devices operating under a common QoS service-provisioning policy.

A DiffServ domain contains two types of nodes:

| Node type | Primary responsibility |
|---|---|
| **Boundary node** | Connects the DiffServ domain to another domain and performs classification and traffic conditioning as required. |
| **Interior node** | Forwards packets within the DiffServ domain using the configured Per-Hop Behaviors (PHBs). |

A boundary node can act as an **ingress node** or **egress node**, depending on the direction of traffic.

### Boundary Nodes

**Boundary nodes** connect a DiffServ domain to another DiffServ domain or a network that does not support DiffServ.

At the ingress boundary, packets can undergo detailed classification using fields such as:
- Source and destination IP addresses.
- Layer 4 protocols and port numbers.
- Application information.
- Existing DSCP values.

Boundary nodes can also perform traffic conditioning, including marking, policing, and shaping.

The objective is to ensure that traffic entering the DiffServ domain receives the appropriate marking and complies with the domain's QoS policy.

### Interior Nodes

**Interior nodes** are located inside the DiffServ domain.

They typically perform simpler classification based on existing **DSCP markings**.

The DSCP value is mapped to a **Per-Hop Behavior (PHB)**, which determines how the packet is forwarded.

For example, the router may:
- Place voice traffic into a priority queue.
- Allocate a minimum bandwidth share to business applications.
- Forward file transfers using best-effort treatment.

Interior nodes generally do not need to repeat the detailed application classification performed at the network boundary.

This is a central reason for DiffServ's scalability.

### Example DiffServ Domain

```text
                    DiffServ Domain
         +-----------------------------------+
         |                                   |
         | Boundary    Interior    Boundary  |
         |   Node       Nodes        Node    |
         |                                   |
Sources ---- R1 ----- R2 ----- R3 ----- R4 --- Destination
         |    |        |        |       |    |
         | Classify    |        |       |    |
         | Mark        |        |       |    |
         | Condition   |        |       |    |
         |             |        |       |    |
         |             ----Apply PHBs----    |
         |                                   |
         +-----------------------------------+
```

In this example:

1. **R1** classifies incoming traffic and assigns the appropriate DSCP markings.
2. **R2 and R3** classify packets using their DSCP values and apply the corresponding PHBs.
3. **R4** forwards traffic out of the DiffServ domain, applying any required egress conditioning.

Both boundary and interior nodes must provide appropriate forwarding treatment. The difference is that detailed classification and conditioning are generally concentrated at the network boundaries.

For example, an interior router can still perform policing, remarking, or more detailed classification if required by the network design.

See [DiffServ Domains and Traffic Conditioning](domains.md).

---

## Behavior Aggregates

A fundamental concept in DiffServ is the **Behavior Aggregate (BA)**.

A Behavior Aggregate is a collection of packets with the same **DSCP value** crossing a link in a particular direction.

These packets are assigned the same PHB at the DiffServ node.

For example, suppose an enterprise has 500 IP phones.

Each phone generates a separate voice traffic flow, but their packets may all be marked **DSCP EF (46)**.

```text
Phone 1 ----+
Phone 2 ----+
Phone 3 ----+                       EF PHB
     ...    +---- R1 ---- R2 ---- Priority Queue
Phone 500 --+
             All marked DSCP 46
```

Rather than maintaining 500 individual resource reservations, R2 can classify these packets as a single EF behavior aggregate.

This reduces the amount of QoS classification and forwarding state required in the network core.

### Behavior Aggregate vs. Individual Flow

| Characteristic | Individual flow | Behavior aggregate |
|---|---|---|
| Identified by | Flow-specific information, such as a 5-tuple | DSCP |
| Number of packets | Packets belonging to one flow | Packets from potentially many flows |
| Primary association | IntServ | DiffServ |
| Resource management | Per-flow reservations | Class-based resource allocation |

---

## DiffServ Classification Methods

DiffServ defines two principal classification methods:

### Behavior Aggregate (BA) Classification

**BA classification** selects packets based only on their **DSCP value**.

For example:

```text
Packet DSCP = 46
        |
        v
    BA Classifier
        |
        v
      EF PHB
```

BA classification is particularly useful in interior nodes because the device does not need to inspect multiple packet fields to identify the traffic class.

### Multifield (MF) Classification

**MF classification** selects packets using the contents of one or more packet header fields.

For example:
- Source IP address.
- Destination IP address.
- IP protocol.
- TCP/UDP source or destination port.
- DSCP value.

MF classification can identify traffic more precisely than BA classification.

For example, an ingress router might classify packets sourced from a particular subnet and destined for TCP port 443 as business-application traffic.

It can then mark those packets with DSCP AF31.

Downstream interior routers can classify the packets using their DSCP markings alone.

### Comparison

| Classification | Fields examined | Typical location |
|---|---|---|
| **BA** | DSCP only | Interior nodes |
| **MF** | One or more packet header fields | Boundary nodes |

This separation of responsibilities is important:

**Complex classification is generally performed at the edge, while simpler DSCP-based classification is performed in the core.**

---

## Per-Hop Behaviors

A **Per-Hop Behavior (PHB)** describes the forwarding treatment applied by a DiffServ node to a behavior aggregate.

A PHB is not the same as a DSCP marking.

- **DSCP:** Identifies the intended forwarding treatment.
- **PHB:** Defines the forwarding behavior applied by a device.
- **Queuing and scheduling mechanisms:** Implement the forwarding behavior.

For example:

```text
Packet marked DSCP EF (46)
             |
             v
      DSCP Classification
             |
             v
           EF PHB
             |
             v
     Priority Queuing
```

The PHB defines the intended behavior, while queuing and scheduling mechanisms implement it.

The principal PHB categories are:

| PHB | Description |
|---|---|
| **Best Effort (BE)** | Ordinary forwarding without differentiated guarantees. |
| **Expedited Forwarding (EF)** | Designed to support low delay, low jitter, and low loss. |
| **Assured Forwarding (AF)** | Provides four forwarding classes with three levels of drop precedence each. |

The mapping between DSCP values and PHBs is determined by the device's QoS policy. Standardized DSCP values have recommended PHB mappings, but a network can also use local mappings.

**A DSCP marking does not automatically guarantee a particular forwarding treatment.**

See [Per-Hop Behaviors (BE, EF, AF)](phb.md) for the details of each PHB.

---

## IntServ vs. DiffServ

The most important distinction between IntServ and DiffServ is **how network resources are allocated**.

### IntServ: Per-Flow Resource Reservation

```text
Flow 1 ----+
           |
Flow 2 ----+---- R1 ---- R2 ---- R3
           |
Flow 3 ----+

Each router maintains separate
resource reservation state for each flow.
```

Each flow requests resources, and participating routers maintain the reservation state necessary to provide the requested service.

### DiffServ: Class-Based Resource Allocation

```text
Flow 1 ----+
Flow 2 ----+---- EF ----+
Flow 3 ----+            |
                        |
Flow 4 ----+            |
Flow 5 ----+---- AF ----+---- R1 ---- R2 ---- R3
                        |
Flow 6 --------- BE ----+
```

Multiple flows share the same QoS class.

Routers allocate bandwidth and buffer resources according to the PHBs configured for those classes.

### Detailed Comparison

| Characteristic | IntServ | DiffServ |
|---|---|---|
| **Granularity** | Individual flows | Traffic aggregates |
| **Resource reservation** | Per-flow | Not required per flow |
| **Signaling** | Typically RSVP | No per-flow signaling required |
| **Router state** | Reservation state for individual flows | Class-based QoS state |
| **Admission control** | Fundamental to resource reservation | Not inherently required |
| **QoS identification** | Flow-specific classification | DSCP-based classification in the core |
| **Scalability** | Limited by per-flow state | High due to aggregation |
| **Service assurance** | Can provide per-flow service guarantees | Depends on PHBs, resource provisioning, and traffic conditioning |
| **Typical application** | Specialized reservation-based services | Enterprise and service provider QoS |

### Does DiffServ Guarantee Bandwidth?

**DiffServ does not inherently guarantee bandwidth, delay, or packet delivery simply because packets carry a particular DSCP value.**

However, network administrators can configure queuing, scheduling, traffic conditioning, and resource allocation to provide specific service characteristics to traffic classes.

For example, a router may reserve a minimum bandwidth share for an AF class using a class-based queuing policy.

This does not create individual reservations for every flow in that class.

The guaranteed bandwidth share applies to the **class as a whole**, and the flows within that class share its resources.

---

## End-to-End QoS with DiffServ

A DiffServ deployment may involve multiple administrative domains, such as an enterprise network and a service provider network.

Different DiffServ domains can use different internal DSCP-to-PHB mappings.

Therefore, traffic entering another administrative domain may be **remarked** or assigned a different forwarding treatment.

End-to-end QoS requires coordination between domains, including appropriate resource provisioning and traffic-conditioning policies.

For example, a service provider may agree to carry a certain volume of customer traffic with preferential forwarding treatment.

That agreement can be enforced through classification, policing, shaping, and appropriate PHBs at the domain boundaries and within the provider network.

See [DiffServ Domains and Traffic Conditioning](domains.md).

---

## Key Points

- **Best Effort** provides ordinary packet forwarding without QoS guarantees.
- **IntServ** provides QoS through per-flow resource reservations, typically signaled using RSVP.
- **RSVP PATH** messages travel downstream, while **RESV** messages travel upstream to request reservations.
- **Admission control** determines whether sufficient resources are available to accept a reservation.
- **IntServ** has limited scalability because routers maintain per-flow reservation state.
- **DiffServ** groups packets into traffic classes rather than maintaining individual flow reservations.
- **Boundary nodes** typically perform detailed classification, marking, and traffic conditioning.
- **Interior nodes** typically use DSCP markings to select the appropriate PHB.
- A **Behavior Aggregate (BA)** consists of packets with the same DSCP value crossing a link in a particular direction.
- **BA classification** uses DSCP alone; **MF classification** examines one or more packet header fields.
- A **PHB** defines forwarding treatment, while queuing and scheduling mechanisms implement it.
- **DiffServ is highly scalable** because many flows can share the same forwarding treatment without requiring per-flow QoS state.
- **End-to-end QoS** depends on coordinated policies and appropriate resource allocation across the forwarding path.

---

## References

- [RFC 1633: Integrated Services in the Internet Architecture](https://www.rfc-editor.org/rfc/rfc1633)
- [RFC 2205: Resource ReSerVation Protocol (RSVP)](https://www.rfc-editor.org/rfc/rfc2205)
- [RFC 2211: Controlled-Load Network Element Service](https://www.rfc-editor.org/rfc/rfc2211)
- [RFC 2212: Guaranteed Quality of Service](https://www.rfc-editor.org/rfc/rfc2212)
- [RFC 2474: Definition of the Differentiated Services Field](https://www.rfc-editor.org/rfc/rfc2474)
- [RFC 2475: An Architecture for Differentiated Services](https://www.rfc-editor.org/rfc/rfc2475)