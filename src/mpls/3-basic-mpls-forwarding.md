# Basic MPLS Forwarding

This page focuses on basic **single-label MPLS unicast forwarding** without L3VPN.

That means the packet normally has only **one MPLS label** while it crosses the MPLS core.

```
Layer 2 header
MPLS label
IP packet
```

This is different from MPLS L3VPN forwarding, where packets usually have two labels:

```
Layer 2 header
Transport label
VPN label
IP packet
```

The previous pages mentioned L3VPNs as they are the primary MPLS use case in CCIE EI,
but I'll cover them in more detail in a separate section.

For this page, the focus is on the basic MPLS transport behavior:

```
Ingress PE pushes one label.
P routers swap the label.
The penultimate router often pops the label.
The egress PE forwards the IP packet normally.
```

---

## Basic Topology

Example topology:

```
CE1 --- PE1 --- P1 --- P2 --- PE2 --- CE2
```

Only the provider routers run MPLS:

```
PE1 --- P1 --- P2 --- PE2
```

The CE routers do not run MPLS. They send and receive normal IP packets.

```
CE1 → PE1: normal IP packet
PE1 → P1: labeled packet
P1  → P2: labeled packet
P2  → PE2: usually unlabeled because of PHP
PE2 → CE2: normal IP packet
```

---

## The Goal of Basic MPLS Forwarding

The goal of basic MPLS forwarding is to allow the provider core to forward traffic using labels instead of requiring every core router to perform an IP forwarding lookup for every packet.

Normal IP forwarding looks like this:

```
Packet arrives
  ↓
Router checks destination IP address
  ↓
Router performs FIB lookup
  ↓
Router forwards packet
```

MPLS forwarding looks like this:

```
Labeled packet arrives
  ↓
Router checks top MPLS label
  ↓
Router looks in the LFIB
  ↓
Router performs the appropriate label operation and forwards the packet
```

The important point is:

```
The ingress PE performs the original IP lookup and pushes the label.
The P routers forward based on the label, usually swapping for another label.
The egress PE performs normal IP forwarding again after the label is removed.
```

---

## BGP-Free Core

One of the most important benefits of MPLS is that the provider core can be **BGP-free**.

A **BGP-free core** means the P routers do not run BGP.
Only the PE routers run BGP and form IBGP neighbor relationships with each other.

```
PE routers run BGP.
P routers run only the provider IGP and MPLS/LDP.
```

Example:

```
      
PE1 < IBGP between PE loopbacks > PE2
 |                                |
 |                                |
 P1 ------------------------------P2

P1 and P2 do not run BGP.
```

The P routers only need reachability to provider infrastructure routes, especially PE loopbacks.

```
P routers know:
- PE loopbacks
- P router loopbacks
- Provider core links
- Labels for provider infrastructure prefixes

P routers do not need:
- Full Internet routes
- Customer routes
- External BGP prefixes
```

This is what makes MPLS useful in service provider cores.

The BGP table can be kept on the PE routers, while the P routers forward traffic across the core using labels.

---

## Why BGP-Free Core Needs MPLS

Consider this topology:

```
CE1 --- PE1 --- P1 --- P2 --- PE2 --- CE2
```

Assume PE1 learns this route from PE2 using BGP:

```
Destination: 203.0.113.0/24
BGP next hop: PE2 loopback, 4.4.4.4
```

PE1 knows the destination route because PE1 runs BGP.

But P1 and P2 do not run BGP, so they do not know the route `203.0.113.0/24`.

If PE1 sent a normal unlabeled IP packet into the core, P1 would need to perform an IP lookup for `203.0.113.0/24`.

But P1 does not have that route.

```
PE1 sends unlabeled packet to P1
  ↓
P1 checks destination 203.0.113.1
  ↓
P1 does not have 203.0.113.0/24
  ↓
Packet is dropped
```

MPLS solves this problem.

PE1 does not send a plain IP packet to P1. Instead, PE1 pushes an MPLS label that represents the path to PE2's loopback.

```
PE1 sends labeled packet to P1
  ↓
P1 checks MPLS label, not destination IP
  ↓
P1 swaps the label and forwards toward PE2
```

The P router does not need the BGP route. It only needs an MPLS label for the BGP next hop.

> THIS IS IMPORTANT: The MPLS label represents the **BGP next hop** (the egress PE's loopback).
The P routers know how to reach the loopback thanks to the IGP and assign an MPLS label for it.
They don't need a route to the inner IP packet's destination IP.

---

## Core Logic

For basic MPLS unicast forwarding, the logic is:

```
BGP provides the destination route to the PE.
The BGP next hop is the remote PE loopback.
The IGP provides reachability to the remote PE loopback.
LDP provides a label for the remote PE loopback.
The LFIB forwards the labeled packet through the core.
```

This is the key to a BGP-free core.

The P routers do not need to know the final destination prefix. They only need to know how to reach the remote PE loopback.

---

## Control Plane vs Data Plane

MPLS forwarding depends on both the IP routing table and the label forwarding table.

| Plane         | Protocols / Tables | Purpose                              |
| :------------ | :----------------- | :----------------------------------- |
| Control plane | IGP, LDP, BGP      | Builds routing and label information |
| Data plane    | FIB, LFIB          | Actually forwards packets            |

The control plane builds the information.

The data plane forwards the traffic.

```
Control plane:
IGP + LDP + BGP
  ↓
Builds routing and label information
  ↓
Data plane:
FIB + LFIB
  ↓
Forwards packets
```

---

## What the IGP Does

The IGP (usually OSPF or IS-IS) provides provider underlay reachability.

The IGP advertises infrastructure prefixes, especially loopbacks.

Example:

```
PE1 loopback: 1.1.1.1/32
P1 loopback:  2.2.2.2/32
P2 loopback:  3.3.3.3/32
PE2 loopback: 4.4.4.4/32
```

The IGP tells each router how to reach these loopbacks.

Example from P1:

```
To reach 4.4.4.4/32, send traffic to P2.
```

LDP is then used to distribute labels for each of those loopback addresses.

---

## What LDP Does

LDP distributes labels for FECs.

In basic MPLS, a FEC is usually an IGP prefix.

Example FEC:

```
4.4.4.4/32
```

PE2 advertises a label for its loopback to P2.

P2 advertises a label for PE2's loopback to P1.

P1 advertises a label for PE2's loopback to PE1.

```
PE2 loopback FEC: 4.4.4.4/32

PE2 tells P2: use implicit null
P2 tells P1:  use label 200
P1 tells PE1: use label 100
```

Each router advertises its own **local label** for the FEC to its LDP neighbors.

The upstream router uses the downstream router's label as its outgoing label.

---

## IGP and LDP Together

The IGP and LDP work together.

```
IGP decides the next hop.
LDP provides the label for that next hop.
```

Example from PE1:

```
Destination FEC: 4.4.4.4/32
IGP next hop: P1
LDP label learned from P1: 100
```

So PE1 forwards traffic toward PE2's loopback by pushing label 100 and sending the packet to P1.

```
IP destination is behind PE2
  ↓
BGP next hop is PE2 loopback
  ↓
IGP says PE2 loopback is reachable via P1
  ↓
LDP says use label 100 when sending to P1
  ↓
PE1 pushes label 100
```

---

## Single-Label Forwarding Example

Topology:

```
CE1 --- PE1 --- P1 --- P2 --- PE2 --- CE2
```

Loopbacks:

```
PE1 loopback: 1.1.1.1/32
P1 loopback:  2.2.2.2/32
P2 loopback:  3.3.3.3/32
PE2 loopback: 4.4.4.4/32
```

Assume PE1 has a BGP route:

```
Destination: 203.0.113.0/24
Next hop: 4.4.4.4
```

PE1 must forward traffic to the BGP next hop, PE2's loopback.

The IGP provides this path:

```
PE1 → P1 → P2 → PE2
```

LDP provides labels for the FEC `4.4.4.4/32`.

Example labels:

| Router | Incoming Label      | Outgoing Label      | Action               |
| :----- | :------------------ | :------------------ | :------------------- |
| PE1    | unlabeled IP packet | 100                 | Push label 100       |
| P1     | 100                 | 200                 | Swap 100 for 200     |
| P2     | 200                 | implicit null       | Pop label due to PHP |
| PE2    | unlabeled IP packet | unlabeled IP packet | Normal IP forwarding |

The packet crosses the core like this:

```
CE1 to PE1:

IP packet to 203.0.113.1


PE1 to P1:

Label 100
IP packet to 203.0.113.1


P1 to P2:

Label 200
IP packet to 203.0.113.1


P2 to PE2:

IP packet to 203.0.113.1


PE2 to CE2:

IP packet to 203.0.113.1
```

Because of PHP, PE2 receives the packet as a normal unlabeled IP packet.

PE2 then performs a normal IP lookup and forwards the packet toward CE2.

---

## Step-by-Step Forwarding Process

Using the same topology:

```
CE1 --- PE1 --- P1 --- P2 --- PE2 --- CE2
```

The forwarding process is:

```
1. CE1 sends an unlabeled IP packet to PE1.

2. PE1 performs an IP lookup in the global routing table.
   The destination matches a BGP route.
   The BGP next hop is PE2's loopback.

3. PE1 recursively resolves the BGP next hop using the IGP.
   PE2's loopback is reachable via P1.

4. PE1 checks the label information learned from LDP.
   P1 advertised label 100 for PE2's loopback.

5. PE1 pushes label 100 and forwards the packet to P1.

6. P1 receives a labeled packet.
   P1 checks the incoming label in the LFIB.
   P1 swaps label 100 for label 200.
   P1 forwards the packet to P2.

7. P2 receives a labeled packet.
   P2 checks the incoming label in the LFIB.
   PE2 advertised implicit null for its loopback.
   P2 pops the label and forwards the unlabeled IP packet to PE2.

8. PE2 receives an unlabeled IP packet.
   PE2 performs a normal IP lookup.
   PE2 forwards the packet toward CE2.
```

This is basic MPLS unicast forwarding with a single label.

---

## FEC and LSP

A **FEC** is a Forwarding Equivalence Class.

In basic MPLS forwarding, the FEC is basically an IGP prefix.

Example:

```
FEC: 4.4.4.4/32
Meaning: traffic forwarded toward PE2's loopback
```

An **LSP** is a Label Switched Path.

The LSP is the path that labeled traffic follows through the MPLS network.

Example:

```
PE1 → P1 → P2 → PE2
```

This LSP is for traffic from PE1 to PE2.

The return path uses a separate LSP.

```
PE2 → P2 → P1 → PE1
```

LSPs are unidirectional.

That means the forward and return labels are different, and the forward and return paths can also be different.

---

## Local Label, Remote Label, and Outgoing Label

Labels are locally significant.

A label value only has meaning to the router that assigned it.

Useful terms:

| Term           | Meaning                                                      |
| :------------- | :----------------------------------------------------------- |
| Local label    | Label this router advertises to neighbors for a FEC          |
| Remote label   | Label learned from a neighbor for a FEC                      |
| Outgoing label | Label this router uses when forwarding traffic to a neighbor |

Example:

```
P1 advertises label 100 for PE2's loopback.
PE1 learns label 100 from P1.
PE1 uses label 100 as the outgoing label when sending traffic to P1.
```

From P1's perspective:

```
Label 100 is a local label.
```

From PE1's perspective:

```
Label 100 is a remote label learned from P1.
Label 100 is also PE1's outgoing label for traffic sent to P1 for that FEC.
```

Another router can use label 100 for something else.

```
P1 might use label 100 for PE2's loopback.
P2 might use label 100 for PE1's loopback.
```

That is not a conflict because MPLS labels are locally significant.

---

## LFIB Entries

The LFIB is the MPLS forwarding table.

A router uses the LFIB when it receives a labeled packet.

A basic LFIB entry tells the router:

```
Incoming label
Outgoing interface
Next hop
Outgoing label or action
```

Example on P1:

```
Incoming label: 100
Outgoing interface: Gi0/1
Next hop: P2
Outgoing label: 200
Action: swap 100 for 200
```

---

## Push, Swap, and Pop in Single-Label Forwarding

In basic unicast MPLS forwarding, the common actions are:

| Router               | Action    | Meaning                                           |
| :------------------- | :-------- | :------------------------------------------------ |
| Ingress PE           | Push      | Add one MPLS label to an unlabeled IP packet      |
| P router             | Swap      | Replace the incoming label with an outgoing label |
| Penultimate P router | Pop       | Remove the label before the egress PE             |
| Egress PE            | IP lookup | Forward the unlabeled packet normally             |


```
Ingress PE receives an IP packet and pushes a label.
P routers receive labeled packets and swap labels.
Penultimate router often pops the label.
Egress PE receives an IP packet and forwards normally.
```

---

## PHP in Basic MPLS Forwarding

**PHP** stands for **Penultimate Hop Popping**.

The penultimate hop is the second-to-last MPLS router in the LSP.

In this path:

```
PE1 → P1 → P2 → PE2
```

P2 is the penultimate hop.

With PHP, P2 removes the label before forwarding the packet to PE2.

```
PE1 pushes label
  ↓
P1 swaps label
  ↓
P2 pops label
  ↓
PE2 receives unlabeled IP packet
```

PHP is triggered by **implicit null**, reserved label 3.

PE2 advertises label 3 to P2 for its directly connected or local FEC.

This tells P2:

```
Do not send me a labeled packet for this FEC.
Pop the label before sending the packet to me.
```

Important point:

```
Implicit null is advertised in the control plane.
Label 3 is not carried in the data-plane packet (hence "null").
```

So, with PHP, the packet from P2 to PE2 is usually unlabeled.

---

## What Happens on the Egress PE

In this basic single-label example, the egress PE usually receives an unlabeled IP packet because of PHP.

Then the egress PE performs a normal IP lookup.

```
PE2 receives packet
  ↓
No MPLS label is present
  ↓
PE2 performs IP lookup
  ↓
PE2 forwards packet toward CE2
```

This is different from MPLS L3VPN forwarding.

In an MPLS L3VPN, PHP usually removes only the outer transport label via PHP.
The inner VPN label remains, and the egress PE uses that VPN label for VRF forwarding.

But in this basic single-label example, there is no VPN label.

So when PHP removes the only label, the egress PE receives a normal IP packet.

---

## Recursive Forwarding on the Ingress PE

The ingress PE often uses recursive forwarding.

Example BGP route:

```
Destination: 203.0.113.0/24
BGP next hop: 4.4.4.4
```

The destination route points to a BGP next hop.

The BGP next hop is not directly connected.

So the router recursively resolves the BGP next hop through the IGP.

```
203.0.113.0/24
  ↓
BGP next hop 4.4.4.4
  ↓
IGP route to 4.4.4.4 via P1
  ↓
LDP label for 4.4.4.4 learned from P1
  ↓
Push label and forward to P1
```

The packet is labeled based on the route to the BGP next hop, not necessarily based directly on the final destination prefix.

> This is a key part of BGP-free core operation.
As noted above, the label does not represent the destination prefix, but rather the BGP next hop (the egress PE's loopback).

---

## Why the P Routers Do Not Need the Final Route

P routers forward based on the MPLS label.

They do not need the final destination route as long as the packet remains labeled through the core.

Example:

```
Packet destination: 203.0.113.1
Top label: 100
```

P1 forwards the packet using label 100.

It does not need to know `203.0.113.0/24`.

```
P1 checks label 100.
P1 swaps label 100 for label 200.
P1 forwards to P2.
```

The IP destination is still inside the packet, but the P router does not use it for forwarding.

That is why the core can be BGP-free.

---

## What Breaks If MPLS Is Missing

If the core is BGP-free but MPLS is not working, traffic may fail.

Example:

```
PE1 has BGP route to 203.0.113.0/24.
P1 does not have BGP route to 203.0.113.0/24.
```

If PE1 sends the packet unlabeled, P1 must route based on the destination IP.

```
P1 receives unlabeled packet to 203.0.113.1
  ↓
P1 performs IP lookup
  ↓
P1 has no route
  ↓
Drop
```

For a BGP-free core to work, PE1 must send the packet into the core with a label that the P routers can use,
because they do not know the destination IP prefix.

---

## Common Misunderstandings

### Misunderstanding 1: MPLS replaces routing

MPLS does not replace routing.

The IGP is still needed to build underlay reachability.

BGP is needed on the PE routers to exchange external routes.

MPLS uses routing information to build label forwarding information.

```
Routing still decides reachability.
MPLS provides label forwarding across the core.
```

---

### Misunderstanding 2: P routers do not need any IP routes

P routers still need IP routes for provider infrastructure.

They need routes to PE loopbacks and core links.

They do not need the external BGP/customer routes carried between PE routers.

```
P routers need provider IGP routes.
P routers do not need the full BGP table.
```

---

### Misunderstanding 3: The label represents the final destination prefix

The label for the **BGP next hop**.

Example:

```
BGP route: 203.0.113.0/24
BGP next hop: 4.4.4.4
MPLS label: label for 4.4.4.4/32
```

The label gets the packet to PE2.

PE2 then performs normal IP forwarding toward the final destination.

> I'm repeating this point multiple times because it's important!

---

### Misunderstanding 4: The egress PE always pops the label

Because of PHP, the egress PE often does not pop the transport label.

The penultimate router usually pops it first.

```
P2 pops the label.
PE2 receives an unlabeled IP packet.
PE2 performs normal IP forwarding.
```

---

## Key Points

* Basic MPLS unicast forwarding usually uses a single label.
* The ingress PE pushes a label onto an unlabeled IP packet.
* P routers swap labels according to the LFIB.
* The penultimate router often pops the label due to PHP.
* The egress PE usually receives an unlabeled IP packet.
* The IGP provides reachability to PE loopbacks and other provider infrastructure routes.
* LDP provides labels for IGP prefixes/FECs.
* BGP runs only on PE routers.
* P routers do not need to know external routes.
* This is called a BGP-free core.
* For BGP-free core forwarding, the ingress PE pushes a label for the BGP next hop (the egress PE's loopback), not for the final destination prefix.
* If MPLS/LDP is broken in a BGP-free core, traffic may be dropped when a P router receives an unlabeled packet for a route it does not know.