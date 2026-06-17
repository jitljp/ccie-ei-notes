# MPLS MTU

MPLS adds extra headers to a packet.

Each MPLS label is 4 bytes.

A normal MPLS L3VPN packet usually has two labels:

```text
Top label:    Transport label
Bottom label: VPN label
```

So a basic L3VPN packet usually adds 8 bytes of MPLS overhead.

```text
Customer IP packet: 1500 bytes
MPLS labels:           8 bytes
-----------------------------
Core requirement:   1508 bytes
```

This is why MTU matters in MPLS networks.

A packet that fits on the customer link may become too large after the ingress PE imposes MPLS labels.

---

## The Basic Problem

Assume CE1 sends a 1500-byte IP packet to CE2.

```text
CE1 --- PE1 --- P1 --- P2 --- PE2 --- CE2
```

CE1 sends a normal IP packet:

```text
Customer IP packet: 1500 bytes
```

PE1 receives the packet and imposes two MPLS labels:

```text
Transport label: 4 bytes
VPN label:       4 bytes
```

The packet now requires 1508 bytes across the MPLS core.

```text
PE1 → P1:
1500-byte IP packet + 8 bytes of MPLS labels
```

If the MPLS core link cannot carry a 1508-byte labeled packet, the packet can be dropped.

This often appears as a problem where small packets work, but larger packets fail.

Example:

```text
CE1# ping 11.11.11.1 source l0 size 1492 df-bit
Type escape sequence to abort.
Sending 5, 1492-byte ICMP Echos to 11.11.11.1, timeout is 2 seconds:
Packet sent with a source address of 10.10.10.1
Packet sent with the DF bit set
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/5 ms

CE1# ping 11.11.11.1 source l0 size 1493 df-bit
Type escape sequence to abort.
Sending 5, 1493-byte ICMP Echos to 11.11.11.1, timeout is 2 seconds:
Packet sent with a source address of 10.10.10.1
Packet sent with the DF bit set
M.M.M
Success rate is 0 percent (0/5)
```

In this example, the MPLS core can carry 1500 bytes total.

A 1492-byte customer packet plus two MPLS labels equals 1500 bytes.

```text
1492-byte customer packet
+ 8 bytes of MPLS labels
= 1500 bytes
```

A 1493-byte customer packet plus two MPLS labels equals 1501 bytes, so it fails.

```text
1493-byte customer packet
+ 8 bytes of MPLS labels
= 1501 bytes
```

The fix is to increase the MPLS MTU on the core-facing interfaces.

---

## Interface MTU, IP MTU, and MPLS MTU

There are three related MTU values to understand.

```text
Interface MTU:
Maximum frame payload size the interface can transmit.

IP MTU:
Maximum IP packet size allowed on the interface.

MPLS MTU:
Maximum labeled packet size allowed on the interface.
```

The important point is this:

```text
IP MTU controls unlabeled IP packets.
MPLS MTU controls labeled MPLS packets.
```

On a core-facing MPLS interface, you can keep the normal IP MTU at 1500 while allowing larger labeled packets.

Example:

```text
IP MTU:    1500
MPLS MTU:  1508
```

This means:

```text
Unlabeled IP packets up to 1500 bytes are allowed.
Labeled MPLS packets up to 1508 bytes are allowed.
```

For a basic L3VPN, this allows the PE to carry a normal 1500-byte customer IP packet after adding two MPLS labels.

---

## Why Configure MPLS MTU Instead of Lowering the Customer MTU?

One possible solution is to reduce the customer-facing MTU.

Example:

```text
CE-facing MTU: 1492
Core MTU:      1500
```

This works mathematically, but it is usually not ideal.

It forces the customer to send smaller packets because the provider core cannot carry the extra MPLS labels.

A better design is:

```text
CE-facing IP MTU:    1500
Core-facing MPLS MTU: 1508 or higher
```

The customer can continue sending normal 1500-byte IP packets.

The provider core carries the additional MPLS label overhead.

```text
CE1 → PE1:
1500-byte IP packet

PE1 → P1:
1508-byte labeled packet
```

This is usually the goal in MPLS L3VPN designs:

```text
Do not reduce the customer MTU just because the provider adds labels.
Increase the MPLS core MTU instead.
```

---

## Required MPLS MTU

Use this formula:

```text
Required MPLS MTU = customer packet MTU + (4 bytes × maximum label stack depth)
```

For a basic LDP-based L3VPN:

```text
Customer packet MTU: 1500
Transport label:       4
VPN label:             4
-------------------------
Required MPLS MTU:  1508
```

So the minimum MPLS MTU for a basic L3VPN carrying 1500-byte customer packets is:

```text
mpls mtu 1508
```

If the network may use more labels, add 4 bytes per extra label.

Examples:

```text
1500 + 2 labels = 1508
1500 + 3 labels = 1512
1500 + 4 labels = 1516
1500 + 5 labels = 1520
```

For a simple CCIE lab with LDP transport and one VPN label, 1508 is the key value to understand.

---

## Configuring MPLS MTU with `override`

On Ethernet core links, use `mpls mtu override` to allow labeled packets larger than the normal 1500-byte interface MTU.

Example on PE1:

```text
PE1(config)# interface Ethernet0/0
PE1(config-if)# description Core link to P1
PE1(config-if)# mpls mtu override 1508
PE1(config-if)# mpls ip
```

Example on P1:

```text
P1(config)# interface Ethernet0/0
P1(config-if)# description Core link to PE1
P1(config-if)# mpls mtu override 1508
P1(config-if)# mpls ip
```

This keeps the normal interface/IP MTU at 1500, but allows MPLS-labeled packets up to 1508 bytes.

Conceptually:

```text
interface MTU: 1500
IP MTU:        1500
MPLS MTU:      1508
```

Without the `override` keyword, you cannot configure an MPLS MTU greater than the interface MTU:

```
PE1(config-if)#mpls mtu ?
  <64-1500>  MTU (bytes)
  override   Override mpls mtu maximum of interface mtu
```

---

## Configuring a Larger MPLS MTU

You do not have to configure exactly 1508.

A larger MPLS MTU gives room for additional labels.

Example:

```text
interface Ethernet0/0
 description Core link to P1
 mpls mtu override 1520
 mpls ip
```

This supports a 1500-byte customer packet with up to five MPLS labels.

```text
1500-byte customer packet
+ 20 bytes of MPLS labels
= 1520 bytes
```

But in a standard L3VPN with two labels, 1508 is sufficient.

---

## Where to Configure MPLS MTU

Configure MPLS MTU on MPLS-enabled core-facing interfaces.

Example:

```text
PE1 --- P1 --- P2 --- PE2
```

Configure MPLS MTU on these links:

```text
PE1 ↔ P1
P1  ↔ P2
P2  ↔ PE2
```

You do not need to configure MPLS MTU on CE-facing interfaces if they do not run MPLS.

Example:

```text
CE1 --- PE1
```

The CE-facing PE interface is usually a normal VRF interface:

```text
interface Ethernet0/1
 description Link to CE1
 vrf forwarding CUSTOMER_A
 ip address 192.168.1.1 255.255.255.0
```

No MPLS MTU is needed there because PE1 receives normal IP packets from CE1.

The labels are imposed on the core-facing interface toward P1.

---

## Configuration Example

Assume this topology:

```text
CE1 --- PE1 --- P1 --- P2 --- PE2 --- CE2
```

The customer uses a normal 1500-byte IP MTU.

The MPLS core should carry a basic two-label L3VPN packet.

Configure 1508 as the MPLS MTU on all core-facing Ethernet links.

PE1:

```text
interface Ethernet0/0
 description Core link to P1
 ip address 10.1.1.1 255.255.255.0
 mpls mtu override 1508
 mpls ip
```

P1 toward PE1:

```text
interface Ethernet0/0
 description Core link to PE1
 ip address 10.1.1.2 255.255.255.0
 mpls mtu override 1508
 mpls ip
```

P1 toward P2:

```text
interface Ethernet0/1
 description Core link to P2
 ip address 10.1.2.1 255.255.255.0
 mpls mtu override 1508
 mpls ip
```

P2 toward P1:

```text
interface Ethernet0/0
 description Core link to P1
 ip address 10.1.2.2 255.255.255.0
 mpls mtu override 1508
 mpls ip
```

P2 toward PE2:

```text
interface Ethernet0/1
 description Core link to PE2
 ip address 10.2.2.1 255.255.255.0
 mpls mtu override 1508
 mpls ip
```

PE2:

```text
interface Ethernet0/0
 description Core link to P2
 ip address 10.2.2.2 255.255.255.0
 mpls mtu override 1508
 mpls ip
```

Now the MPLS core can carry:

```text
1500-byte customer IP packet
+ transport label
+ VPN label
```

---

## Verification

Check the interface configuration:

```text
PE1# show running-config interface Ethernet0/0
Building configuration...

interface Ethernet0/0
 description Core link to P1
 ip address 10.1.1.1 255.255.255.0
 mpls mtu override 1508
 mpls ip
end
```

You can also view the MPLS MTU with this command:

```
PE1#show mpls interfaces e0/0 detail 
Interface Ethernet0/0:
        Type Unknown
        IP labeling enabled (ldp) :
          Interface config
        LSP Tunnel labeling not enabled
        IP FRR labeling not enabled
        BGP labeling not enabled
        MPLS operational
        MTU = 1508
```

---

## Testing with Ping

Now CE1 can send pings up to the standard 1500-byte MTU:

```
CE1#ping 11.11.11.1 source l0 size 1500 df-bit 
Type escape sequence to abort.
Sending 5, 1500-byte ICMP Echos to 11.11.11.1, timeout is 2 seconds:
Packet sent with a source address of 10.10.10.1 
Packet sent with the DF bit set
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/4 ms

CE1#ping 11.11.11.1 source l0 size 1501 df-bit  
Type escape sequence to abort.
Sending 5, 1501-byte ICMP Echos to 11.11.11.1, timeout is 2 seconds:
Packet sent with a source address of 10.10.10.1 
Packet sent with the DF bit set
.....
Success rate is 0 percent (0/5)
```

---

## Alternate Solution: Increase the Core Interface MTU

Another valid solution is to increase the regular interface MTU on all MPLS core links.

Instead of keeping the interface MTU at 1500 and configuring MPLS MTU separately, you can make every core link support a larger frame size.

Example:

```text
interface Ethernet0/0
 description Core link
 mtu 1600
 mpls ip
```

Or, in a jumbo-MTU core:

```text
interface Ethernet0/0
 description Core link
 mtu 9000
 mpls ip
```

This gives the core enough room to carry normal 1500-byte customer packets plus MPLS labels.

Example with a 1600-byte core MTU:

```text
Customer IP packet: 1500 bytes
MPLS labels:           8 bytes
-----------------------------
Required:           1508 bytes
Core MTU:           1600 bytes
```

This works, but configure it consistently on every core link.

```text
PE1 --- P1 --- P2 --- PE2

Every MPLS core-facing interface in the path must support the larger frame.
```

The advantage is simplicity.

```text
Raise the core interface MTU high enough.
Then MPLS-labeled packets fit without needing a separate MPLS MTU value.
```

The main rule is the same in both cases:

```text
Every core link must be able to carry the customer packet plus the MPLS label stack.
```

---

## Key Points

* Each MPLS label adds 4 bytes.
* A basic MPLS L3VPN packet usually has two labels.
* A 1500-byte customer packet needs 1508 bytes in the MPLS core.
* The required MPLS MTU depends on the maximum label stack depth.
* Use `mpls mtu override 1508` on Ethernet core-facing interfaces for a basic L3VPN with 1500-byte customer packets.
* The customer-facing MTU can stay at 1500.
* The regular IP MTU can stay at 1500.
* MPLS MTU applies to labeled packets.
* Configure MPLS MTU on every MPLS-enabled core-facing interface in the path.
* The `override` keyword is used when configuring an MPLS MTU above the normal Ethernet interface MTU.