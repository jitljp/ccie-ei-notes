# MPLS Overview

**MPLS** (Multiprotocol Label Switching) is a forwarding technology that allows routers to forward packets based on **labels** instead of using the destination IP address in the IP header.

In normal IP routing, each router performs a FIB lookup based on the destination IP address.

```text
Normal IP forwarding:

Packet arrives
  ↓
Router checks destination IP
  ↓
Router performs FIB lookup
  ↓
Router forwards packet
```

With MPLS, packets are assigned short labels by a **PE** (Provider Edge) router at the edge of the MPLS network. Core routers then forward the packets by looking at the label, not by doing a normal IP route lookup.

```text
MPLS forwarding:

Labeled packet arrives
  ↓
Router checks MPLS label
  ↓
Router swaps/removes label according to LFIB
  ↓
Router forwards packet
```

For CCIE EI, the most important MPLS topic is **MPLS Layer 3 VPNs**.

---

## Why MPLS Exists

MPLS separates the forwarding decision from the original Layer 3 packet header.

Instead of every core router making a full IP routing decision, the MPLS edge router classifies the packet and applies a label. After that, MPLS routers in the core forward the packet based on labels.

This gives MPLS several advantages:

| Advantage                    | Explanation                                                |
| ---------------------------- | ---------------------------------------------------------- |
| Efficient forwarding         | Core routers forward based on labels.                      |
| VPN support                  | MPLS can keep customer routing tables separate using VRFs. |
| Protocol flexibility         | MPLS can carry IPv4, IPv6, Ethernet, and other payloads.   |
| Traffic engineering          | MPLS can steer traffic across specific paths.              |
| Service provider scalability | The core does not need to know every customer route.       |

> MPLS allows the provider core to forward customer traffic without the core routers running BGP or knowing the customer routes.

---

## Basic MPLS Terminology

| Term     | Meaning                                                       |
| -------- | ------------------------------------------------------------- |
| MPLS     | Multiprotocol Label Switching                                 |
| Label    | A short value used to forward traffic through an MPLS network |
| LSR      | Label Switch Router                                           |
| Edge LSR | MPLS router at the edge of the MPLS network                   |
| LER      | Label Edge Router                                             |
| PE       | Provider Edge router                                          |
| P        | Provider core router                                          |
| CE       | Customer Edge router                                          |
| LSP      | Label Switched Path                                           |
| FEC      | Forwarding Equivalence Class                                  |
| LIB      | Label Information Base                                        |
| LFIB     | Label Forwarding Information Base                             |
| LDP      | Label Distribution Protocol                                   |

In MPLS L3VPNs, the common device roles are:

```text
CE --- PE --- P --- P --- PE --- CE
       |                 |
       |<-- MPLS core -->|
```

| Device | Role                                                                       |
| ------ | -------------------------------------------------------------------------- |
| CE     | Customer router. Does not run MPLS.                                |
| PE     | Provider edge router. Runs MPLS and has VRFs.                              |
| P      | Provider core router. Runs MPLS but does not know customer routes. |

> Only the **PE** and **P** routers use MPLS. **CE** routers send and receive regular IP packets.

---

## MPLS Labels

An MPLS label is inserted between the Layer 2 header and the Layer 3 header.

```text
Ethernet Header | MPLS Label | IP Header | Payload
```

Because the MPLS header is inserted between Layer 2 and Layer 3, MPLS is often called a **Layer 2.5 protocol**. The header is often called a **shim header**.

The MPLS label header is 4 bytes long.

```text
+--------+-------+---+-----+
| Label  |  EXP  | S | TTL |
+--------+-------+---+-----+
 20 bits  3 bits   1  8 bits
```

| Field    |    Size | Purpose                         |
| -------- | ------ | ------------------------------- |
| Label    | 20 bits | Label value used for forwarding |
| EXP      |  3 bits | QoS marking                     |
| S        |   1 bit | Bottom of stack bit             |
| TTL      |  8 bits | Time to live                    |

The **S bit** (Bottom of Stack) identifies the bottom label in the label stack.

```text
S = 0 means another label follows.
S = 1 means this is the bottom label.
```

MPLS can use a **label stack**, meaning more than one label can be attached to a packet.

For example, in MPLS L3VPNs, a packet commonly has two labels:

```text
Outer label: transport label
Inner label: VPN label
```

The **outer label** gets the packet across the MPLS core to the correct PE router.

The **inner label** tells the egress PE what to do with the customer packet.

---

## MPLS Forwarding Process

MPLS forwarding uses three main actions regarding labels:

| Action | Meaning                              |
| ------ | ------------------------------------ |
| Push   | Add a label                          |
| Swap   | Replace one label with another label |
| Pop    | Remove a label                       |

A typical MPLS forwarding path looks like this:

```text
CE1 → PE1 → P1 → P2 → PE2 → CE2
```

The forwarding process is:

```text
1. CE1 sends an unlabeled IP packet to PE1.

2. PE1 receives the packet.
   PE1 performs an IP/VRF lookup.
   PE1 pushes one or more MPLS labels and forwards the packet.

3. P1 receives the labeled packet.
   P1 swaps the incoming label for a new outgoing label and forwards the packet.

4. P2 receives the labeled packet.
   P2 swaps the label and forwards the packet.

5. PE2 receives the labeled packet.
   PE2 pops the MPLS label.
   PE2 forwards the original customer packet toward CE2.
```

In simple form:

```text
Unlabeled packet
  ↓
PE pushes label
  ↓
P routers swap labels
  ↓
Egress PE pops label
  ↓
Unlabeled packet exits MPLS network
```

> Due to a behavior called **PHP** (**Penultimate Hop Popping**), the egress PE often does **not** pop the outer MPLS label itself. Instead, the penultimate router in the path does so.


---

## Label Switched Path

An **LSP**, or Label Switched Path, is the path that labeled traffic follows through the MPLS network.

Example:

```text
PE1 → P1 → P2 → PE2
```

The LSP is unidirectional.

That means traffic from PE1 to PE2 uses one LSP, and traffic from PE2 to PE1 uses another LSP.

```text
PE1 to PE2 LSP:

PE1 → P1 → P2 → PE2


PE2 to PE1 LSP:

PE2 → P2 → P1 → PE1
```

The forward and return paths do not have to be the same.

---

## Forwarding Equivalence Class

A **FEC**, or Forwarding Equivalence Class, is a group of packets that are forwarded in the same way.

For example, packets going toward the same destination prefix can belong to the same FEC.

```text
FEC: 10.1.1.0/24
Label: 100
Next hop: P1
```

Routers assign labels to FECs.

The idea is:

```text
Packets in the same FEC receive the same forwarding treatment.
```

A FEC is basically equivalent to an IP prefix. Packets within the same destination prefix are forwarded along the same LSP.

---

## MPLS Control Plane and Data Plane

MPLS has a control plane and a data plane.

| Plane         | Role                                 |
| ------------- | ------------------------------------ |
| Control plane | Builds routing and label information |
| Data plane    | Forwards labeled packets             |

The control plane uses protocols such as:

* OSPF
* IS-IS
* EIGRP
* BGP
* LDP
* RSVP-TE

The data plane uses the LFIB to forward labeled packets.

```text
Control plane:
IGP, BGP, LDP
  ↓
Builds routing and label information
  ↓
Data plane:
LFIB forwards labeled packets
```

---

## LIB and LFIB

The **LIB** (Label Information Base) stores label information learned from label distribution protocols such as LDP.

The **LFIB** (Label Forwarding Information Base), is used to actually forward labeled packets.

| Table | Purpose               |
| ----- | --------------------- |
| RIB   | IP routing table      |
| FIB   | IP forwarding table   |
| LIB   | MPLS label database   |
| LFIB  | MPLS forwarding table |


```text
RIB/FIB = IP forwarding
LIB/LFIB = MPLS forwarding
```

Useful verification commands:

```text
show ip route = displays the RIB
show ip cef = displays the FIB
show mpls ldp bindings = displays the LIB
show mpls forwarding-table = displays the LFIB
```

---

## Label Distribution

Routers need to learn which labels to use for each destination.

The most common label distribution protocol is **LDP** (Label Distribution Protocol).

LDP allows neighboring MPLS routers to exchange label bindings.

Example:

```text
P2 says:
"For my loopback 2.2.2.2/32, use label 200."

P1 stores that label binding.

When P1 forwards traffic toward 2.2.2.2/32, it can use label 200.
```

LDP depends on the IGP.

The IGP decides the next hop.

LDP provides the label for that next hop.

```text
IGP decides where to forward.
LDP decides which label to use.
```

---

## MPLS and the IGP

MPLS requires underlay reachability.

In a basic MPLS core, the provider routers usually run an IGP such as OSPF or IS-IS.

The IGP is used to advertise infrastructure routes, especially loopback addresses.

```text
PE1 loopback: 1.1.1.1/32
P1 loopback:  2.2.2.2/32
P2 loopback:  3.3.3.3/32
PE2 loopback: 4.4.4.4/32
```

The IGP must provide reachability between PE loopbacks.

This is especially important because MP-BGP VPN peering is commonly built between PE loopbacks.

```text
PE1 loopback reachability ⇄ PE2 loopback reachability
```

The MPLS control plane depends on this reachability.

---

## Penultimate Hop Popping

**Penultimate Hop Popping**, or **PHP**, is an MPLS behavior where the second-to-last router removes the outer MPLS label before forwarding the packet to the egress PE.

```text
PE1 → P1 → P2 → PE2
           ↑
           Penultimate hop
```

Instead of PE2 receiving a packet with the outer transport label, P2 removes the label first.

```text
Without PHP:

P2 → PE2: packet with outer label


With PHP:

P2 → PE2: packet with outer label already removed
```

PHP reduces the amount of label processing required on the egress PE.

The label value used for PHP is **implicit null**, label 3.

```text
Implicit null = label 3
```

A router advertising implicit null is telling its upstream neighbor:

```text
"Do not send me this label. Pop the outer label before sending the packet to me."
```

---

## MPLS L3VPN Big Picture

MPLS L3VPNs use MPLS to connect customer sites through a provider network.

Example:

```text
Customer A Site 1                 Customer A Site 2

CE1 --- PE1 --- P --- P --- PE2 --- CE2
```

The provider uses VRFs on PE routers to keep customer routing tables separate.

```text
PE router:

Global routing table
VRF CUSTOMER_A
VRF CUSTOMER_B
VRF CUSTOMER_C
```

MPLS L3VPNs use several important pieces together:

| Component            | Role                                                             |
| -------------------- | ---------------------------------------------------------------- |
| VRF                  | Separate customer routing table                                  |
| RD                   | Makes customer routes unique in MP-BGP                           |
| RT                   | Controls route import and export                                 |
| MP-BGP               | Carries customer VPN routes between PE routers                   |
| MPLS transport label | Gets traffic across the provider core                            |
| VPN label            | Identifies the correct VRF or forwarding action on the egress PE |

A typical MPLS L3VPN packet crossing the core has two labels:

```text
Outer/Top label: reach the egress PE
Inner/Bottom label: identify the VPN route/VRF
```

Example:

```text
CE1 sends packet to CE2
  ↓
PE1 pushes two labels
  ↓
P routers forward based on outer label
  ↓
PE2 uses inner label to forward into the correct VRF
  ↓
Packet reaches CE2
```

The P routers in the core do not need the customer routes.

They only need labels for reaching PE router loopbacks.

---

## Why the MPLS Core Does Not Need Customer Routes

One of the most important MPLS L3VPN concepts is that the provider core is separated from customer routing.

```text
P routers know:
- Provider loopbacks
- Provider infrastructure links
- MPLS labels for provider routes


P routers do not need to know:
- Customer A routes
- Customer B routes
- Customer C routes
```

This makes the network more scalable.

Only PE routers need customer routing information.

```text
CE routes live in VRFs on PE routers.
VPN routes are exchanged between PE routers using MP-BGP.
P routers forward labeled packets through the core using the top label.
```

---

## Basic MPLS Configuration

A minimal MPLS core needs:

1. An IGP for provider reachability
2. CEF enabled
3. MPLS enabled globally
4. MPLS enabled on core-facing interfaces
5. LDP neighbor relationships

Enabling MPLS:

```text
ip cef

mpls ip

interface GigabitEthernet0/0
 mpls ip
```

> **ip cef** and **mpls ip** (global config mode command) are enabled by default. 
> To enable basic MPLS, simply configure **mpls ip** on interfaces connecting P and PE routers.

---

## Basic Verification

Check whether MPLS is enabled on interfaces:

```text
show mpls interfaces
```

Check LDP neighbors:

```text
show mpls ldp neighbor
```

Check label bindings (LIB):

```text
show mpls ldp bindings
```

Check the MPLS forwarding table (LFIB):

```text
show mpls forwarding-table
```

Check CEF:

```text
show ip cef
```

Check the IGP:

```text
show ip route
show ip ospf neighbor
show clns neighbors
```
---

## Core MPLS Logic

For basic MPLS forwarding:

```text
The IGP gives the next hop.
LDP gives the label.
The LFIB forwards the labeled packet.
```

For MPLS L3VPNs:

```text
The outer/top label gets the packet to the egress PE.
The inner/bottom label identifies the forwarding action on the egress PE.
The P routers do not need customer routes.
```

---

## Key Points

* MPLS forwards packets using labels.
* MPLS labels are inserted between the Layer 2 and Layer 3 headers ("Layer 2.5").
* LSRs forward labeled packets.
* PE routers sit at the edge of the provider MPLS network and connect to CE routers.
* P routers sit in the provider core.
* CE routers do not run MPLS.
* LDP is commonly used to distribute labels.
* The IGP provides underlay reachability.
* The LFIB is used to forward labeled packets.
* MPLS supports label stacking.
* MPLS L3VPNs commonly use an outer transport label and an inner VPN label.
* P routers do not need to know customer routes in an MPLS L3VPN.
* PHP removes the outer label on the penultimate hop.
* Implicit null is label 3.

---
