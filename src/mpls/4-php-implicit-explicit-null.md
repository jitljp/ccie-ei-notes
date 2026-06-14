# PHP, Implicit Null, and Explicit Null

This page explains three closely related MPLS concepts:

```text
PHP
Implicit Null
Explicit Null
```

These concepts control what happens at the end of an LSP.

In basic MPLS forwarding (ignoring L3VPN), the packet has one label while it crosses the core:

```text
Layer 2 header
MPLS label
IP packet
```

The important question is:  Who removes the final MPLS label?

There are two main possibilities:

| Behavior | Who Removes the Label? | Packet Sent to Egress PE |
| :--- | :--- | :--- |
| PHP / Implicit Null | Penultimate router | Unlabeled IP packet |
| Explicit Null | Egress router | MPLS packet with label 0 or 2 |

---

## Reserved Null Labels

MPLS labels 0-15 are reserved.

The three null-related labels are:

| Label | Name | Meaning |
| :--- | :--- | :--- |
| 0 | IPv4 Explicit Null | Send packet with label 0; egress router pops it and forwards as IPv4 |
| 2 | IPv6 Explicit Null | Send packet with label 2; egress router pops it and forwards as IPv6 |
| 3 | Implicit Null | Do not actually send label 3; pop the label before forwarding |

The key difference is:

```text
Explicit Null is carried in the packet.
Implicit Null is not carried in the packet.
```

Label 3 is used in the control plane, but it does not appear in the data-plane packet.

---

## Review: Normal Single-Label Forwarding

Example topology:

```text
CE1 --- PE1 --- P1 --- P2 --- PE2 --- CE2
```

Assume PE1 sends traffic toward PE2's loopback:

```text
PE2 loopback: 4.4.4.4/32
```

The IGP provides reachability to `4.4.4.4/32`.

LDP provides labels for `4.4.4.4/32`.

Example label flow:

```text
PE1 pushes label 100.
P1 swaps label 100 for label 200.
P2 pops label 200 because PE2 advertised implicit null.
PE2 receives an unlabeled IP packet.
```

Packet format:

```text
PE1 to P1:

Label 100
IP packet


P1 to P2:

Label 200
IP packet


P2 to PE2:

IP packet
```

This is the default PHP behavior.

---

## PHP

**PHP** stands for **Penultimate Hop Popping**.

The **penultimate hop** is the second-to-last MPLS router in the LSP.

In this path:

```text
PE1 → P1 → P2 → PE2
```

P2 is the penultimate hop.

With PHP, P2 removes the top label before forwarding the packet to PE2.

```text
PE1 pushes label
  ↓
P1 swaps label
  ↓
P2 pops label
  ↓
PE2 receives unlabeled IP packet
```

In basic single-label forwarding, this means the egress PE receives a normal IP packet.

---

## Why PHP Is Useful

Without PHP, the egress PE would receive a labeled packet.

It would need to:

```text
1. Look at the MPLS label.
2. Pop the MPLS label.
3. Look at the exposed IP packet.
4. Perform the IP forwarding lookup.
```

With PHP, the penultimate router removes the label before the packet reaches the egress PE.

So the egress PE receives a normal IP packet and performs normal IP forwarding.

```text
P2 does the label pop.
PE2 does the IP lookup.
```

This reduces the amount of label processing required on the egress PE.

Hence PHP is the default behavior.

---

## Implicit Null

PHP is caused by **Implicit Null**.

Implicit Null is reserved label value **3**.

However, label 3 is not actually sent in the packet.

Instead, the egress LSR advertises label 3 to its upstream LDP neighbor.

This tells the upstream router:

```text
Do not send me a real label for this FEC.
Pop the label before sending the packet to me.
```

Example:

```text
PE2 advertises implicit null (label 3) for 4.4.4.4/32 to P2.
P2 installs a pop operation for that FEC.
P2 pops the label before forwarding traffic to PE2.
```

Important point:

```text
Implicit Null = label 3
But label 3 is never carried in the actual packet.
```

---

## Why It Is Called "Implicit"

It is called **implicit** because label 3 is not actually sent in the data plane.

The egress router advertises label 3 to its upstream neighbor in the control plane.
This tells the upstream router to pop the label before forwarding the packet.


Control plane:

```text
PE2 advertises label 3 for 4.4.4.4/32.
```

Data plane:

```text
P2 pops the label before sending the packet to PE2.
```

So label 3 is a **control-plane signal**, not a data-plane label.

---

## LFIB Behavior with Implicit Null

With implicit null, the penultimate router's LFIB shows a pop operation.

Example on P2:

```text
P2# show mpls forwarding-table 4.4.4.4
Local  Outgoing    Prefix       Outgoing   Next Hop
label  label       or Tunnel Id interface
200    Pop Label   4.4.4.4/32   Gi0/1      10.0.23.4
```

---

## Forwarding with Implicit Null

Topology:

```text
PE1 --- P1 --- P2 --- PE2
```

Destination FEC:

```text
PE2 loopback: 4.4.4.4/32
```

Example labels:

| Router | Incoming Label | Outgoing Label | Action |
| :--- | :--- | :--- | :--- |
| PE1 | unlabeled IP packet | 100 | Push label 100 |
| P1 | 100 | 200 | Swap 100 for 200 |
| P2 | 200 | implicit null | Pop label |
| PE2 | unlabeled IP packet | unlabeled IP packet | IP lookup |

Packet flow:

```text
PE1 to P1:

Label 100
IP packet


P1 to P2:

Label 200
IP packet


P2 to PE2:

IP packet
```

The final hop is unlabeled, so PE2 simply performs an IP lookup and forwards the packet to CE2.

---

## Explicit Null

**Explicit Null** keeps an MPLS label on the packet until it reaches the egress LSR.

For IPv4, explicit null uses label **0**.

For IPv6, explicit null uses label **2**.

In this page, the examples use IPv4 explicit null:

```text
IPv4 Explicit Null = label 0
```

With explicit null, the egress router advertises label 0 instead of label 3.

This tells the penultimate router:

```text
Send me the packet with label 0.
Do not pop the MPLS header completely.
```

So the packet from the penultimate router to the egress PE still has an MPLS header:

```text
P2 to PE2:

Label 0
IP packet
```

Then PE2 receives label 0, pops it, and forwards the internal IPv4 packet.

---

## UHP

Explicit null causes **UHP**, or **Ultimate Hop Popping**.

This means the final MPLS label is popped by the **ultimate hop**, not the penultimate hop.

In this path:

```text
PE1 → P1 → P2 → PE2
```

PE2 is the ultimate hop.

With UHP:

```text
PE1 pushes label
  ↓
P1 swaps label
  ↓
P2 swaps to explicit null label 0
  ↓
PE2 pops label 0
  ↓
PE2 forwards the IP packet
```

Compare that with PHP:

```text
PE1 pushes label
  ↓
P1 swaps label
  ↓
P2 pops the label
  ↓
PE2 receives an unlabeled IP packet
```

---

## Why Explicit Null Is Useful

The main reason to use explicit null is to preserve MPLS QoS information all the way to the egress LSR.

The MPLS label stack entry includes the **EXP** field.

```text
Label | EXP | S | TTL
```

The EXP field is commonly used for QoS treatment in the MPLS core.

With PHP, the penultimate router removes the MPLS header before sending the packet to the egress router.

That means the egress PE receives an unlabeled packet.

```text
P2 to PE2 with PHP:

IP packet
```

If the egress PE needs to apply QoS based on the MPLS EXP value, the MPLS EXP field is no longer present.

Explicit null solves this by keeping an MPLS header on the packet until the egress PE.

```text
P2 to PE2 with explicit null:

Label 0, EXP value preserved
IP packet
```

Now PE2 can still see the MPLS EXP value on the received packet.

Then PE2 pops label 0 and forwards the IP packet.

---

## Implicit Null vs Explicit Null

| Feature | Implicit Null | Explicit Null |
| :--- | :--- | :--- |
| Reserved label | 3 | 0 for IPv4, 2 for IPv6 |
| Carried in packet? | No | Yes |
| Final action done by | Penultimate router | Egress router |
| Behavior | PHP | UHP |
| Packet sent to egress PE | Unlabeled IP packet | MPLS packet with label 0 or 2 |
| Main benefit | Reduces work on egress PE | Preserves MPLS EXP/QoS to egress PE |
| Default? | Yes | No |

---

## Forwarding with Explicit Null

Topology:

```text
PE1 --- P1 --- P2 --- PE2
```

Destination FEC:

```text
PE2 loopback: 4.4.4.4/32
```

Assume PE2 advertises explicit null for `4.4.4.4/32`.

Example labels:

| Router | Incoming Label | Outgoing Label | Action |
| :--- | :--- | :--- | :--- |
| PE1 | unlabeled IP packet | 100 | Push label 100 |
| P1 | 100 | 200 | Swap 100 for 200 |
| P2 | 200 | 0 | Swap 200 for explicit null |
| PE2 | 0 | unlabeled IP packet | Pop label 0 and perform IP lookup |

Packet flow:

```text
PE1 to P1:

Label 100
IP packet


P1 to P2:

Label 200
IP packet


P2 to PE2:

Label 0
IP packet


PE2 after popping label 0:

IP packet
```

The difference is the final MPLS hop:

```text
With PHP:

P2 sends an unlabeled IP packet to PE2.


With explicit null:

P2 sends an MPLS packet with label 0 to PE2.
```

---

## Visual Comparison

### Default PHP / Implicit Null

```text
PE1              P1              P2              PE2
 |               |               |               |
 | Label 100     |               |               |
 |-------------->|               |               |
 |               | Label 200     |               |
 |               |-------------->|               |
 |               |               | IP packet     |
 |               |               |-------------->|
```

P2 pops the label before forwarding to PE2.

---

### Explicit Null

```text
PE1              P1              P2              PE2
 |               |               |               |
 | Label 100     |               |               |
 |-------------->|               |               |
 |               | Label 200     |               |
 |               |-------------->|               |
 |               |               | Label 0       |
 |               |               |-------------->|
```

P2 swaps the label to label 0.

PE2 receives label 0, pops it, and forwards the IP packet.

---

## Control-Plane Difference

The difference is caused by what PE2 advertises to P2.

### Default behavior

```text
PE2 advertises label 3 for 4.4.4.4/32.
```

P2 interprets label 3 as:

```text
Pop the label before forwarding to PE2.
```

### Explicit null behavior

```text
PE2 advertises label 0 for 4.4.4.4/32.
```

P2 interprets label 0 as:

```text
Swap the outgoing label to 0 and forward the packet to PE2.
```

So the control-plane advertisement changes the data-plane behavior.

---

## Where to Configure Explicit Null

Explicit null is configured on the **egress LSR**.

In this example, configure it on PE2:

```text
PE1 --- P1 --- P2 --- PE2
```

Why PE2?

Because PE2 is the router advertising the null label for its own local/connected FEC.

The penultimate router does not decide by itself whether to use PHP or explicit null.

It follows the label advertised by the downstream router.

```text
PE2 advertises implicit null → P2 pops the label.
PE2 advertises explicit null → P2 sends label 0.
```

---

## Configuring Explicit Null on IOS XE

Basic command:

```text
PE2(config)# mpls ldp explicit-null
```

This tells PE2 to advertise explicit null in situations where it would normally advertise implicit null.

In a simple lab, this is usually enough.

Example:

```text
PE2(config)# mpls ldp explicit-null
```

Then check the penultimate router.

Before explicit null, P2 may show:

```text
P2# show mpls forwarding-table 4.4.4.4
Local  Outgoing    Prefix       Outgoing   Next Hop
label  label       or Tunnel Id interface
200    Pop Label   4.4.4.4/32   Gi0/1      10.0.23.4
```

After explicit null, P2 should show outgoing label `0`:

```text
P2# show mpls forwarding-table 4.4.4.4
Local  Outgoing    Prefix       Outgoing   Next Hop
label  label       or Tunnel Id interface
200    0           4.4.4.4/32   Gi0/1      10.0.23.4
```

The important change is:

```text
Before: Pop Label
After:  0
```

That means P2 will forward the packet to PE2 with an explicit null label.

---

## Scoping Explicit Null with ACLs

On IOS XE, you can scope explicit null with ACLs.

The general syntax is:

```text
mpls ldp explicit-null [for <prefix-acl> | to <peer-acl> | for <prefix-acl> to <peer-acl>]
```

The keywords mean:

| Keyword | Meaning |
| :--- | :--- |
| `for` | Advertise explicit null only for specific prefixes/FECs |
| `to` | Advertise explicit null only to specific LDP peers |
| `for ... to ...` | Advertise explicit null for specific prefixes to specific peers |

---

## Explicit Null for Specific Prefixes

Use the `for` keyword to advertise explicit null only for selected prefixes.

Example:

```text
PE2(config)# access-list 44 permit host 4.4.4.4
PE2(config)# mpls ldp explicit-null for 44
```

This means:

```text
Advertise explicit null for 4.4.4.4/32.
```

In this example, PE2 advertises explicit null only for the FEC permitted by ACL 44.

Other FECs can still use the default implicit null behavior.

---

## Explicit Null to Specific Peers

Use the `to` keyword to advertise explicit null only to selected LDP peers.

The ACL should match the neighbor's LDP ID.

Example:

```text
PE2(config)# access-list 33 permit host 3.3.3.3
PE2(config)# mpls ldp explicit-null to 33
```

This means:

```text
Advertise explicit null only to the LDP peer with LDP ID 3.3.3.3.
```

This is useful if you want only a specific penultimate router to send explicit null.

---

## Explicit Null for Specific Prefixes to Specific Peers

You can combine both options.

Example:

```text
PE2(config)# access-list 44 permit host 4.4.4.4
PE2(config)# access-list 33 permit host 3.3.3.3
PE2(config)# mpls ldp explicit-null for 44 to 33
```

This means:

```text
For prefix 4.4.4.4/32,
advertise explicit null only to LDP peer 3.3.3.3.
```

This gives the most precise control.

---

## L3VPN Note

This page focuses on single-label forwarding, 
but it's worth pointing out one L3VPN detail.

In MPLS L3VPN, the packet usually has two labels:

```text
Transport label
VPN label
Customer IP packet
```

With normal PHP, the penultimate router pops the **outer transport label**.

The inner VPN label remains:

```text
VPN label
Customer IP packet
```

> MPLS L3VPNs are covered in their own section.

---

## Common Misunderstandings

### Misunderstanding 1: Label 3 is sent in the packet

Label 3 is not sent in the packet.

Implicit null is advertised in the control plane.

The upstream router responds by popping the label.

```text
Control plane: label 3 is advertised.
Data plane: label is popped.
```

---

### Misunderstanding 2: Explicit null means "no label"

Explicit null does not mean no label.

It means the packet is sent with a real reserved label.

For IPv4:

```text
Explicit Null = label 0
```

So the packet still has an MPLS header.

---

### Misunderstanding 3: The penultimate router chooses PHP by itself

The penultimate router does not simply decide to do PHP.

It does PHP because the downstream router advertised implicit null.

```text
Downstream advertises label 3.
Upstream installs pop operation.
```

---

### Misunderstanding 4: Explicit null changes the LSP path

Explicit null does not change the LSP path.

It only changes the final label operation.

The path is still:

```text
PE1 → P1 → P2 → PE2
```

The difference is:

```text
PHP: P2 pops the label.
Explicit null: P2 swaps to label 0, and PE2 pops label 0.
```

---

## Key Points

* PHP stands for Penultimate Hop Popping.
* PHP means the second-to-last MPLS router pops the top label.
* PHP is triggered by implicit null.
* Implicit null is reserved label 3.
* Label 3 is advertised in the control plane but not carried in the data plane.
* With PHP in a single-label LSP, the egress PE usually receives an unlabeled IP packet.
* Explicit null keeps an MPLS label on the packet until it reaches the egress LSR.
* IPv4 explicit null uses label 0.
* IPv6 explicit null uses label 2.
* Explicit null causes UHP, or Ultimate Hop Popping.
* With explicit null, the penultimate router swaps the outgoing label to 0 or 2.
* The egress router receives the explicit null label, pops it, and forwards the IP packet.
* The main benefit of explicit null is preserving the MPLS EXP value to the egress router.
* On IOS XE, configure explicit null with `mpls ldp explicit-null`.
* Configure explicit null on the egress LSR, not the penultimate LSR.
* Verify explicit null by checking that the penultimate router shows outgoing label `0` instead of `Pop Label`.