# Control-Plane Traffic and Packet Punt Paths

Most traffic passing through a router or Layer 3 switch is forwarded by the **data plane**, without requiring processing by the device's CPU.

However, certain packets must be delivered to the **control plane** for processing.

Examples include:

- Routing protocol packets, such as OSPF Hellos and BGP messages.
- Management traffic, such as SSH and SNMP.
- Packets requiring special handling, such as packets with expired TTL values.
- Layer 2 control traffic, such as ARP and CDP.
- Transit packets that cannot be forwarded entirely by the normal forwarding path.

Understanding how these packets reach the CPU is essential for configuring **Control Plane Policing (CoPP)** and **Control Plane Protection (CPPr)**.

---

## Normal Data-Plane Forwarding

Modern Cisco IOS XE platforms commonly use dedicated forwarding hardware, such as ASICs, to forward packets.

For normal IP forwarding, the forwarding hardware uses information programmed into the **Forwarding Information Base (FIB)** and associated adjacency information.

Consider the following topology:

```text
PC1                                 PC2
 |                                    |
 +--- Ethernet0/0--R1--Ethernet0/1 ---+
```

PC1 sends a packet to PC2, and R1 routes the packet toward its destination.

When the necessary forwarding information is available in hardware, the packet is forwarded without involving the CPU.

```text
Incoming Packet
       |
       v
 Forwarding Hardware
       |
       v
    FIB Lookup
       |
       v
   Forward Packet
       |
       v
 Outgoing Interface
```

This is sometimes called **fast-path forwarding**.

> Normal hardware-forwarded transit packets do not need to pass through a CoPP policy because they do not require control-plane processing.

---

## CPU-Bound Traffic

Some packets cannot be handled entirely by the forwarding hardware.

These packets must be sent toward the device's CPU for additional processing.

On hardware-forwarding platforms, sending a packet from the forwarding hardware toward the CPU is commonly called **punting**.

```text
Incoming Packet
       |
       v
 Forwarding Hardware
       |
       +------ Normal Forwarding ------> Egress Interface
       |
       +------ CPU-Bound Traffic
                        |
                        v
                  CoPP Policing
                        |
                  +-----+-----+
                  |           |
                Permit       Drop
                  |
                  v
              CPU Processing
```

CoPP can police CPU-bound traffic before it consumes additional CPU processing resources.

A packet dropped by CoPP is not delivered to the corresponding control-plane process.

The exact position and implementation of policing depend on the platform. On hardware-based platforms, CoPP is commonly enforced in hardware before packets reach the CPU.

---

## Packet Punt vs. Packet Inject

Two terms describe the directions in which packets move between the forwarding hardware and CPU.

| Term | Direction | Purpose |
|---|---|---|
| **Punt** | Data plane → CPU | Sends a received packet for CPU processing |
| **Inject** | CPU → Data plane | Sends a CPU-generated packet toward an outgoing interface |

### Punt

A packet is punted when the forwarding path determines that CPU processing is required.

For example, an OSPF Hello received on an interface must be processed by the OSPF process.

```text
OSPF Neighbor
      |
      | OSPF Hello
      v
 Forwarding Hardware
      |
      | Punt
      v
     CPU
      |
      v
 OSPF Process
```

The forwarding hardware does not independently establish or maintain OSPF neighbor relationships.

### Inject

When the OSPF process generates its own Hello message, the packet must be sent from the CPU toward the forwarding hardware.

```text
 OSPF Process
      |
      v
     CPU
      |
      | Inject
      v
 Forwarding Hardware
      |
      v
 Outgoing Interface
      |
      v
 OSPF Neighbor
```

This is called **packet injection**.

The main purpose of ingress CoPP is to protect the CPU from received traffic. Some platforms also support output control-plane policies, but outbound policing capabilities are platform-dependent.

---

## Why Packets Are Punted

There are several reasons why a packet may require CPU processing.

### 1. Packets Destined for the Device

Packets addressed to the router or switch itself must generally be processed locally.

For example:

```text
R1 Ethernet0/0: 10.1.1.1

PC1: 10.1.1.10
```

PC1 initiates an SSH connection to R1.

```text
Source IP:      10.1.1.10
Destination IP: 10.1.1.1
Protocol:       TCP
Destination Port: 22
```

Because the destination is one of R1's own addresses, the packet is delivered to the CPU rather than forwarded toward another device.

Other examples include:

- BGP packets destined for a local BGP session.
- ICMP Echo Requests directed to a router interface.
- SNMP requests.
- Routing protocol messages that require local processing.
- Packets destined for a tunnel endpoint.

This traffic is often called **receive traffic** or **for-us traffic**.

### 2. Packets Requiring Exception Processing

Some packets require additional handling even though they are not ordinary management or routing protocol messages.

Examples include:

- Packets whose TTL expires during forwarding.
- Packets with certain IP options.
- Packets requiring processing by features not supported in the hardware forwarding path.
- Packets triggering certain exception or error-handling operations.

For example, R1 receives a transit packet with a TTL of 1.

```text
PC1 -----> R1 -----> R2 -----> PC2

Packet TTL = 1
```

R1 cannot forward the packet with a TTL of 0.

The packet is discarded, and R1 may generate an ICMP Time Exceeded message.

Depending on the platform, the exception may require CPU involvement.

> Not every packet requiring special treatment is necessarily punted. Modern forwarding hardware can handle some exceptions directly.

### 3. Packets Requiring Software Forwarding

A packet may be destined for another device but require CPU assistance because it cannot be forwarded entirely through the normal hardware path.

Examples include certain unsupported forwarding features and tunnel-processing scenarios.

```text
PC1 ------> R1 ------> PC2
             |
             | Special processing required
             v
            CPU
             |
             v
        Forward Packet
```

These packets are different from packets directly destined for R1.

The CPU processes them to support **transit forwarding**, rather than consuming them as locally addressed traffic.

### 4. Layer 2 Control Traffic

Certain Layer 2 protocols require local device processing.

Examples include:

| Protocol | Purpose |
|---|---|
| ARP | IPv4 address-to-MAC resolution |
| CDP | Cisco neighbor discovery |
| LLDP | Link-layer neighbor discovery |
| STP | Layer 2 loop prevention |
| LACP | EtherChannel negotiation |

These protocols are not ordinary IP unicast traffic.

They may require delivery to the CPU or other specialized processing paths, depending on the platform.

---

## CEF Adjacencies and CPU Processing

Cisco Express Forwarding (CEF) uses adjacency information to determine how packets should be handled.

Some adjacency types indicate that normal forwarding is not possible or that a special action is required.

| Adjacency Type | Behavior |
|---|---|
| **Normal adjacency** | Provides the Layer 2 rewrite information needed for forwarding |
| **Receive adjacency** | Identifies a destination that belongs to the device itself |
| **Glean adjacency** | Indicates that Layer 2 resolution is required for a directly connected destination |
| **Punt adjacency** | Indicates that a packet requires additional processing outside the normal CEF forwarding path |
| **Drop/Discard adjacency** | Discards the packet rather than forwarding it |

### Receive Adjacency

A **receive adjacency** is used for traffic destined for one of the device's own IP addresses.

For example:

```text
R1 Ethernet0/0: 10.1.1.1/24
```

Traffic destined for `10.1.1.1` is locally received rather than forwarded through R1.

On a supported IOS XE router, CEF information can be inspected using:

```text
R1# show ip cef 10.1.1.1 detail
```

The route to the local address can be associated with a receive adjacency.

### Glean Adjacency

A **glean adjacency** represents a directly connected destination for which the device does not yet have the required Layer 2 rewrite information.

For example:

```text
R1 Ethernet0/0: 10.1.1.1/24

Destination: 10.1.1.100
ARP entry:   Not present
```

R1 needs to resolve the destination's MAC address.

A glean condition can trigger ARP resolution and may involve CPU processing.

Once the necessary adjacency information is installed, subsequent packets can normally be forwarded without additional CPU assistance.

> A glean adjacency does not mean that every packet for the destination must be software-forwarded. Hardware can perform resolution-related handling and rate limiting, depending on the platform.

### Punt Adjacency

A **punt adjacency** identifies packets requiring handling outside the normal CEF forwarding path.

The term *punt* is also used more broadly to describe any traffic sent from a forwarding ASIC to the CPU.

Therefore, **a packet can be punted to the CPU without matching a CEF adjacency explicitly named `punt`**.

For example, a packet matching a receive adjacency may also traverse the hardware-to-CPU punt path.

---

## CPPr Traffic Classification

**Control Plane Protection (CPPr)** divides CPU-bound traffic into three logical control-plane subinterfaces.

Each subinterface handles a different category of traffic.

| CPPr Subinterface | Traffic Category | Examples |
|---|---|---|
| **Host** | Traffic directly destined for the device | SSH, BGP, OSPF, SNMP |
| **Transit** | Transit IP traffic requiring software forwarding | Certain software-switched transit or tunnel traffic |
| **CEF-Exception** | Forwarding exceptions and certain non-IP control traffic | ARP, Layer 2 keepalives, packets with expired TTL |

The three categories are **mutually exclusive** for CPPr classification. A packet is assigned to one category, not all three.

### Host Traffic

The **host subinterface** receives control-plane IP traffic directly destined for the router.

For example:

```text
PC1 ---- SSH ----> R1
                   |
                   v
             CPPr Host
                   |
                   v
              SSH Process
```

This category includes most IP management and routing protocol traffic.

### Transit Traffic

The **transit subinterface** receives IP packets that are not destined for the router itself but require software forwarding.

```text
PC1 ---- Packet ----> R1 ----> PC2
                       |
                       v
                  CPPr Transit
                       |
                       v
                CPU Processing
                       |
                       v
                  Forwarding
```

The router processes the packet and then forwards it toward its destination, when appropriate.

### CEF-Exception Traffic

The **CEF-exception subinterface** handles traffic redirected for exception processing and certain non-IP traffic delivered to the control plane.

Examples include:

- ARP packets.
- Layer 2 keepalive packets.
- Certain IP packets with options.
- IP packets with TTL values of 0 or 1.

```text
Incoming Packet
       |
       v
 Forwarding Hardware
       |
       | Exception
       v
  CPPr CEF-Exception
       |
       v
   CPU Processing
```

An important distinction is that **CEF-exception traffic is not limited to packets destined for the router**.

It can also include packets that would otherwise be transit traffic.

> CPPr classification describes how the packet reaches the control plane. The exact classification behavior and supported subinterfaces depend on the platform and software release.

---

## Aggregate CoPP vs. CPPr

Traditional **aggregate CoPP** applies a policy to CPU-bound traffic without requiring separate policies for the host, transit, and CEF-exception categories.

CPPr provides additional granularity by allowing separate policies for these traffic paths.

```text
                   CPU-Bound Traffic
                           |
                           v
                   Aggregate CoPP
                           |
                           v
                    CPPr Classifier
                           |
               +-----------+-----------+
               |           |           |
               v           v           v
              Host       Transit   CEF-Exception
               |           |           |
               v           v           v
           Host CoPP   Transit CoPP  Exception CoPP
               |           |           |
               +-----------+-----------+
                           |
                           v
                     CPU Processing
```

This is a conceptual illustration of the CPPr framework. The actual hardware/software processing order can differ by platform.

An aggregate policy and CPPr-specific policies can coexist on platforms that support these features.

---

## Catalyst 9000 Punt Paths

Catalyst 9000 switches use hardware forwarding and a predefined **System CoPP** policy to protect the CPU.

CPU-bound packets are classified into hardware CPU queues and associated policers.

Examples of punt reasons include:

- Packets destined for the switch itself.
- Routing protocol packets.
- ARP packets.
- Layer 2 control protocol packets.
- Packets requiring software forwarding.
- Glean adjacency processing.
- Certain forwarding exceptions.

The switch uses the packet's **punt cause** to help identify why it was sent toward the CPU.

These punt causes and CPU queues are important when troubleshooting System CoPP drops.

Unlike CPPr, Catalyst 9000 System CoPP does not expose the host, transit, and CEF-exception subinterfaces as independently configurable CPPr policies.

---

## IOS XE Verification

The following commands help identify CPU-bound traffic and control-plane protection behavior.

### General CoPP Verification

```text
R1# show policy-map control-plane
```

Displays the configured control-plane policy, including traffic classes, policing rates, and available packet/drop counters.

On platforms supporting CPPr:

```text
R1# show control-plane host
R1# show control-plane transit
R1# show control-plane cef-exception
```

These commands can display information about the corresponding logical control-plane subinterfaces.

### Catalyst 9000 Punt Verification

On supported Catalyst 9000 platforms:

```text
SW1# show platform software fed switch active punt cause summary
```

Displays punt causes and their associated packet and drop counters.

```text
SW1# show platform software fed switch active punt cpuq all
```

Displays information about CPU punt queues.

```text
SW1# show platform hardware fed switch active qos queue stats internal cpu policer
```

Displays hardware CPU queue and policer statistics, including drops.

These commands are platform-specific and are not universally available on IOS XE routers or virtual images.

---

## Key Points

- Most transit traffic is **forwarded by the data plane without CPU involvement**.
- **CPU-bound traffic** includes locally destined packets, certain software-switched transit packets, Layer 2 control traffic, and forwarding exceptions.
- **Punt** means sending a packet from the forwarding path toward the CPU.
- **Inject** means sending a packet from the CPU toward the forwarding path.
- A **receive adjacency** identifies traffic destined for the device itself.
- A **glean adjacency** indicates that Layer 2 resolution information is required.
- A **punt adjacency** identifies traffic requiring special handling outside the normal CEF forwarding path.
- CPPr divides control-plane traffic into three categories: **host, transit, and CEF-exception**.
- **Host** traffic is directly destined for the device.
- **Transit** traffic requires software forwarding but is not destined for the device.
- **CEF-exception** traffic includes forwarding exceptions and certain non-IP control traffic.
- CoPP protects the CPU by policing traffic before it reaches control-plane processing.
- Catalyst 9000 switches use hardware CPU queues and **System CoPP** policers to protect the punt path.

---

## References

- [Cisco - Control Plane Policing Implementation Best Practices](https://www.cisco.com/c/en/us/about/security-center/copp-best-practices.html)
- [Cisco IOS XE 17.x - Control Plane Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-ctrl-pln-plc-0.html)
- [Cisco - Control Plane Protection](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_plcshp/configuration/15-mt/qos-plcshp-15-mt-book/qos-plcshp-cpp.html)
- [Cisco - Cisco Express Forwarding Overview](https://www.cisco.com/c/en/us/td/docs/ios/ipswitch/configuration/guide/convert/ips_cef/cef_overview.html)
- [Cisco - Troubleshoot Control Plane Operations on Catalyst 9000 Switches](https://www.cisco.com/c/en/us/support/docs/switches/catalyst-9300-switch/221841-troubleshoot-control-plane-operations-on.html)