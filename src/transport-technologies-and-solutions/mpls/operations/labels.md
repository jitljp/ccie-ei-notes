# MPLS Labels

MPLS forwarding is based on **labels**, not normal destination IP lookups.

A label is a short value inserted between the Layer 2 header and the Layer 3 packet. Routers in the MPLS core use this label to make forwarding decisions.

In a basic MPLS network:

```text
CE sends unlabeled IP packet
  ↓
Ingress PE pushes label
  ↓
P routers swap labels
  ↓
Last P router often pops label due to PHP
  ↓
Egress PE forwards unlabeled IP packet to CE
```

In an MPLS L3VPN network, there are usually **two labels**:

```text
Outer/Top label = transport label
Inner/Bottom label = VPN label
```

The outer label, also called the **transport label** gets the packet across the provider core to the correct egress PE.

The inner label, also called the **VPN label** tells the egress PE which VRF/customer route to use.

---

## MPLS Label Stack Entry

An MPLS label is carried in a 32-bit **label stack entry**.

```text
  20 bits       3 bits   1 bit    8 bits
+-------------+--------+-------+---------+
|   Label     |  EXP   |   S   |   TTL   |
+-------------+--------+-------+---------+
```

| Field | Size    | Purpose                                 |
| :---- | :------ | :-------------------------------------- |
| Label | 20 bits | Label value used for forwarding         |
| EXP   | 3 bits  | Experimental field, often used for QoS |
| S     | 1 bit   | Bottom of Stack bit                     |
| TTL   | 8 bits  | Time to Live                            |

---

## Label Field

The **Label** field is the main forwarding value.

It is 20 bits long, so the theoretical label space is:

```text
0 to 1,048,575
```

However, some label values are reserved.

| Label | Meaning                                         |
| :---- | :---------------------------------------------- |
| 0     | IPv4 Explicit Null                              |
| 1     | Router Alert                                    |
| 2     | IPv6 Explicit Null                              |
| 3     | Implicit Null                                   |
| 4-15  | Reserved                                        |
| 16+   | Generally available for normal label assignment |

IOS-XE routers dynamically assign labels in the range 16 to 100,000 by default:
```text
PE1# show run all | i label range
mpls label range 16 100000

PE1# show mpls label range
Downstream Generic label region: Min/Max label: 16/100000
```

This range can be modified with:
```
PE1(config)# mpls label range <minimum> <maximum>
```


A label is not globally meaningful by itself. Labels have **local significance**.

Each router assigns a label to each FEC, and uses LDP to inform its neighbors about those labels.

---

## EXP Field

The **EXP** field is 3 bits long.

It is commonly used for QoS marking inside the MPLS network.

A common behavior is to copy the packet’s IP Precedence value into the MPLS EXP field when the packet enters the MPLS network.

IP Precedence is 3 bits. In modern IP QoS marking, those 3 bits correspond to the three most significant bits of the 6-bit DSCP value.

Example:

```text
IP packet enters MPLS network

DSCP value: 101000
IPP bits:   101
MPLS EXP:   101
```

So, in simple terms:

```text
IP Precedence / top 3 DSCP bits
  ↓
MPLS EXP value
  ↓
QoS treatment in the MPLS core
```

The EXP field is carried in each MPLS label stack entry. If a packet has multiple labels, each label has its own EXP field.

Example:

```text
Outer transport label, EXP=5
Inner VPN label,       EXP=0
Customer IP packet
```

---

## S Bit

The **S** bit means **Bottom of Stack**.

MPLS supports more than one label on a packet. This is called a **label stack**.

The S bit identifies the final label in the stack.

| S Bit | Meaning                            |
| :---- | :--------------------------------- |
| 0     | More labels exist below this label |
| 1     | This is the bottom label           |


Example with two labels:

```text
Label 200, S=0   ← outer/top label
Label 300, S=1   ← inner/bottom label
IP packet
```

The top label is processed first.

In an MPLS L3VPN packet, the label stack often looks like this:

```text
Outer transport label, S=0
Inner VPN label,       S=1
Customer IP packet
```

The P routers in the provider core only care about the **outer transport label**.

The egress PE uses the **inner VPN label** to determine how to forward the received packet.

---

## TTL Field

The MPLS label stack entry has its own **TTL** field.

This works similarly to the IP TTL field.

Each MPLS router decrements the TTL as the packet is forwarded.

If the TTL reaches 0, the packet is discarded, preventing infinite loops in the MPLS network.

> More on the MPLS TTL & TTL propagation in another page.

---

## Label Stack

MPLS packets can carry one label or multiple labels.

This is called a **label stack**.

```text
Layer 2 header
MPLS label
IP packet
```

Or:

```text
Layer 2 header
Outer MPLS label
Inner MPLS label
IP packet
```

The label closest to the Layer 2 header is the **top label**.

The top label is the label that is examined by the next MPLS router.

```text
Layer 2 header
[ Top label    ] ← processed first
[ Bottom label ]
IP packet
```

For MPLS L3VPN, this is very important.

```text
Layer 2 header
[ Transport label ] ← used to reach the egress PE
[ VPN label       ] ← used by the egress PE for VRF forwarding
Customer IP packet
```

The P routers do not need customer routes.

They only need to know how to forward the packet based on the outer transport label.

---

## Label Operations

MPLS routers perform three main label operations:

```text
Push = add a label on top of the stack
Swap = change the top label for another label
Pop = remove the top label
```

---

## Push

A **push** operation adds a label to a packet.

The ingress PE usually performs the push operation.

Example:

```text
Unlabeled IP packet
  ↓
PE pushes label 100
  ↓
Labeled packet
```

In MPLS L3VPN, the **ingress PE** usually pushes two labels:

```text
Customer IP packet
  ↓
Push VPN label
  ↓
Push transport label
  ↓
Send packet into MPLS core
```

Result:

```text
Transport label
VPN label
Customer IP packet
```

The transport label is on top, and is used to direct the packet through the MPLS core
to the **egress PE**.

---

## Swap

A **swap** operation replaces the incoming label with a different outgoing label.

P routers usually perform swap operations.

Example:

```text
P1 receives packet with label 100.
P1 swaps label 100 for label 200.
P1 forwards the packet to the next router.
```

In simple form:

```text
In label 100
  ↓
Swap
  ↓
Out label 200
```

This is the normal forwarding behavior inside the MPLS core.

The router does not need to perform a destination IP lookup. It checks the incoming label,
looks in the LFIB, swaps the label, and forwards the packet.

---

## Pop

A **pop** operation removes a label from the packet.

Example:

```text
Labeled packet
  ↓
Pop label
  ↓
Packet without that label
```

If there is only one label, popping the label reveals the original IP packet.

If there are multiple labels, popping the top label reveals the next label.

Example:

```text
Before pop:

Transport label
VPN label
Customer IP packet

After pop:

VPN label
Customer IP packet
```

This is important in MPLS L3VPN.

The P router may pop the outer transport label, but the inner VPN label remains.

The egress PE then forwards the packet according to the VPN label.

---

## PHP

**PHP** stands for **Penultimate Hop Popping**.

The **penultimate hop** is the second-to-last MPLS router in the LSP.

In this path:

```text
CE1 --- PE1 --- P1 --- P2 --- PE2 --- CE2
```

P2 is the penultimate hop.

With PHP, the P2 pops the label before forwarding the packet to the egress PE.

```text
PE1 pushes label
  ↓
P1 swaps label
  ↓
P2 pops label
  ↓
PE2 receives packet
```

This reduces work for the egress PE.

Without PHP, the egress PE would receive a labeled packet, pop the label, and then perform the final lookup.

With PHP, the egress PE receives the packet after the outer label has already been removed.

A PE router advertises **label 3** (implicit null) to tell the penultimate LSR to perform PHP.

> More on PHP, implicit null, and explicit null in another page.

---

## MPLS Labels in L3VPN

In MPLS L3VPN, the packet usually has two labels.

```text
Outer transport label
Inner VPN label
Customer IP packet
```

The two labels have different purposes.

| Label           | Purpose                                            | Used By   |
| :-------------- | :------------------------------------------------- | :-------- |
| Transport label | Gets the packet to the egress PE                   | P routers |
| VPN label       | Identifies the VRF/customer route on the egress PE | Egress PE |

The transport label is learned from the MPLS transport control plane, usually LDP in basic L3VPN labs.

The VPN label is learned from MP-BGP VPNv4 or VPNv6 routes.

Example:

```text
PE1 learns VPNv4 route from PE2.

Route:
10.1.1.0/24

Next hop:
PE2 loopback

VPN label:
300

Transport label to reach PE2:
200
```

When PE1 forwards traffic to 10.1.1.0/24, it pushes both labels:

```text
Transport label 200
VPN label 300
Customer IP packet
```

The neighboring P router forwards based on label 200.

The egress PE uses label 300 to identify the correct VRF/customer route.

---

## Example L3VPN Forwarding

Topology:

```text
CE1 --- PE1 --- P1 --- P2 --- PE2 --- CE2
```

Customer route:

```text
CE2 LAN: 10.2.2.0/24
```

PE2 advertises the VPNv4 route to PE1 using MP-BGP:

```text
Route: 10.2.2.0/24
Next hop: PE2 loopback
VPN label: 300
```

LDP provides a transport label to reach PE2:

```text
Transport label: 200
```

When CE1 sends a packet to 10.2.2.1:

```text
1. CE1 sends an unlabeled IP packet to PE1.

2. PE1 performs a VRF lookup.
   Destination 10.2.2.1 matches a VPN route learned from PE2.

3. PE1 pushes two labels:
   Outer transport label = 200
   Inner VPN label = 300

4. P1 receives the packet.
   P1 swaps the outer transport label.

5. P2 receives the packet.
   P2 is the penultimate hop.
   Due to PHP, P2 pops the outer transport label.

6. PE2 receives the packet with the inner VPN label still present.

7. PE2 uses the VPN label to identify the correct VRF/customer route.

8. PE2 forwards the original customer packet to CE2.
```

Packet format as it crosses the network:

```text
CE1 to PE1:

Customer IP packet


PE1 to P1:

Transport label
VPN label
Customer IP packet


P1 to P2:

New transport label
VPN label
Customer IP packet


P2 to PE2, after PHP:

VPN label
Customer IP packet


PE2 to CE2:

Customer IP packet
```

Important point:

```text
PHP removes the outer transport label.
It does not remove the inner VPN label. 
```

> The egress PE still needs the VPN label so it knows which VRF or forwarding entry to use.

---

## Local Label and Remote Label

IOS output uses terms like:

```text
local label
outgoing label
remote label
```

The idea is:

```text
Local label = label this router advertises to neighbors
Remote label = label learned from a neighbor
Outgoing label = label this router uses when forwarding to that neighbor
```

Example:

```text
R1 advertises label 100 for 10.10.10.10/32.
R2 learns from R1 that label 100 reaches 10.10.10.10/32.
R2 uses label 100 as the outgoing label when sending traffic to R1.
```

From R1's perspective, label 100 is a **local label**.

From R2's perspective, label 100 is a **remote/outgoing label** learned from R1.

---

## Downstream Label Assignment

In normal MPLS forwarding, the downstream router assigns the label.

Example:

```text
R1 --- R2 --- R3
```

R3 owns this prefix (its loopback IP):

```text
3.3.3.3/32
```

R3 advertises label 300 for 3.3.3.3/32 to R2.

R2 advertises label 200 for 3.3.3.3/32 to R1.

Forwarding from R1 to R3, ignoring PHP:

```text
R1 uses label 200.
R2 swaps to label 300.
R3 receives the packet.
```

> The next-hop/downstream router tells the upstream router what label to use.

That is why labels are **locally significant**. Each router controls the labels that its neighbors use when sending traffic to it.

---

## Key Points

* MPLS labels are locally significant.
* An MPLS label stack entry is 32 bits.
* The 20-bit Label field is used to determine how the packet is forwarded.
* The 3-bit EXP field is used for QoS.
* The S-bit (Bottom of Stack) indicates the bottom label of the stack.
* The 8-bit TTL field is used for the same purpose as the IP TTL field.
* MPLS L3VPNs typically use a two-label stack: transport label and VPN label
* P routers forward based on the outer transport label.
* The egress PE uses the VPN label to correctly forward the packet to the customer.
* A **push** operation adds a label to the top of the stack.
* A **swap** operation changes the top label for another label.
* A **pop** operation removes the top label of the stack.
* **PHP** causes the penultimate LSR to pop the top label before forwarding to the egress PE.
* A **local label** is a label the local router advertises to neighbors.
* A **remote label** is learned from a neighbor.
