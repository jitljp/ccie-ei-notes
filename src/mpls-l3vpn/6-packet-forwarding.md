# L3VPN Packet Forwarding

This page explains how an MPLS L3VPN packet is forwarded from one customer site to another.

The previous pages covered the main control-plane pieces:

```text
VRF:
Customer routing table on the PE.

RD:
Makes VPN routes unique in MP-BGP.

RT:
Controls import and export.

VPNv4/VPNv6:
Carries VPN routes between PEs.

VPN label:
Advertised with the VPNv4/VPNv6 route.
```

This page focuses on the data plane.

---

## Basic Topology

Use the same topology:

```text
Customer A Site 1                    Customer A Site 2

10.10.10.0/24                        11.11.11.0/24
      |                                      |
     CE1                                    CE2
      |                                      |
     PE1 -------- P1 --------- P2 --------- PE2
    1.1.1.1     2.2.2.2      3.3.3.3      4.4.4.4
```

Assume CE1 sends traffic to CE2:

```text
Source:      10.10.10.1
Destination: 11.11.11.1
```

From the customer point of view, this is normal IP forwarding.

CE1 and CE2 do not know about MPLS.

The MPLS VPN forwarding happens inside the provider network.

---

## Control Plane Before Forwarding

Packet forwarding only works if the control plane has already built the correct state.

For traffic from CE1 to CE2, PE1 needs these things:

```text
1. A VRF route to 11.11.11.0/24.
2. A VPN label for 11.11.11.0/24.
3. A reachable BGP next hop for the VPNv4 route.
4. A transport label to the BGP next hop.
```

Example:

```text
VRF route:
11.11.11.0/24 via 4.4.4.4

VPNv4 next hop:
4.4.4.4

VPN label:
21

Transport label to 4.4.4.4:
17
```

The VRF route and VPN label come from MP-BGP VPNv4.

The transport label comes from LDP.

The route to the VPNv4 next hop comes from the IGP in the provider core.

```text
MP-BGP VPNv4:
Customer route + VPN label.

IGP:
Reachability to remote PE loopback.

LDP:
Transport label to remote PE loopback.
```

If any one of these pieces is missing, forwarding will fail.

---

## High-Level Process

Here is the forwarding process at a high level:

```text
1. CE1 sends an IP packet to PE1.

2. PE1 receives the packet on a VRF interface.

3. PE1 performs a lookup in the CUSTOMER_A VRF.

4. PE1 finds a remote VPN route learned from PE2.

5. PE1 imposes two labels:
   - Transport label to PE2
   - VPN label advertised by PE2

6. P1 swaps the transport label.

7. P2 swaps the transport label (or pops due to PHP).

8. PE2 receives the packet with the VPN label.

9. PE2 uses the VPN label to identify the correct VPN forwarding entry.

10. PE2 forwards the IP packet toward CE2.
```

The important point:

```text
The ingress PE does the VRF lookup.

The P routers do transport label switching.

The egress PE uses the VPN label.
```

---

## Label Stack

An MPLS L3VPN packet usually has two labels in the core:

```text
Top label:    Transport label
Bottom label: VPN label
Payload:      Customer IP packet
```

Example:

```text
Label 17      Transport label to PE2
Label 21      VPN label for 11.11.11.0/24
IP packet     10.10.10.1 → 11.11.11.1
```

In packet form:

```text
+------------------------------+
| MPLS label 17                |
| Transport label              |
| S bit = 0                    |
+------------------------------+
| MPLS label 21                |
| VPN label                    |
| S bit = 1                    |
+------------------------------+
| Customer IP packet           |
| 10.10.10.1 → 11.11.11.1      |
+------------------------------+
```

The top label has the bottom-of-stack bit unset.

The VPN label has the bottom-of-stack bit set.

```text
Transport label:
S bit = 0

VPN label:
S bit = 1
```

The P routers only make forwarding decisions based on the top label.

They do not need to understand the VPN label.

---

## Ingress PE Forwarding

PE1 receives the packet from CE1.

The packet enters PE1 on an interface assigned to `CUSTOMER_A`.

```text
interface Ethernet0/1
 vrf forwarding CUSTOMER_A
 ip address 192.168.1.1 255.255.255.0
```

Because the ingress interface is in a VRF, PE1 does not use the global routing table for the customer destination.

It uses the VRF routing table.

```text
Packet arrives on CUSTOMER_A interface.
Destination: 11.11.11.1

Lookup table:
VRF CUSTOMER_A
```

PE1 finds a route like this:

```text
PE1# show ip route vrf CUSTOMER_A 11.11.11.0

Routing Table: CUSTOMER_A
Routing entry for 11.11.11.0/24
  Known via "bgp 65000", distance 200, metric 0, type internal
  Last update from 4.4.4.4 00:03:47 ago
  Routing Descriptor Blocks:
  * 4.4.4.4 (default), from 4.4.4.4, 00:03:47 ago
      opaque_ptr 0x794E53DEBCF0 
      Route metric is 0, traffic share count is 1
      AS Hops 0
      MPLS label: 21
      MPLS Flags: MPLS Required
```

---

## Recursive Resolution

A remote VPN route normally has the remote PE loopback as its BGP next hop.

Example:

```text
Customer route:
11.11.11.0/24

VPNv4 next hop:
4.4.4.4
```

PE1 now needs to resolve `4.4.4.4`.

That resolution happens in the global routing table, not in the customer VRF.

```text
VRF lookup:
Find the VPN route to 11.11.11.0/24.

Global lookup:
Resolve the next hop 4.4.4.4.
```

Example:

```text
PE1# show ip cef 4.4.4.4
4.4.4.4/32
  nexthop 10.1.1.2 Ethernet0/0 label 17-(local:16)
```

As the CEF entry shows, PE1 should use label `17` to reach 4.4.4.4.

You can also view that in the LFIB:

```text
PE1# show mpls forwarding-table 4.4.4.4

Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop    
Label      Label      or Tunnel Id     Switched      interface              
16         17         4.4.4.4/32       0             Et0/0      10.1.1.2  
```

So the result is:

```text
To reach customer prefix 11.11.11.0/24:
  Use VPN label 21.
  Send toward BGP next hop 4.4.4.4.

To reach BGP next hop 4.4.4.4:
  Use transport label 17.
  Send to P1 at 10.1.1.2.
```

CEF combines these into one forwarding action.

---

## CEF Imposition Entry

The ingress PE imposes the label stack based on the VRF CEF entry.

Example:

```text
PE1# show ip cef vrf CUSTOMER_A 11.11.11.0/24 detail

11.11.11.0/24, epoch 0, flags [rib defined all labels]
  recursive via 4.4.4.4 label 21
    nexthop 10.1.1.2 Ethernet0/0 label 17-(local:16)
```

Read this from top to bottom:

```text
11.11.11.0/24:
Customer prefix in the VRF.

recursive via 4.4.4.4:
The VPNv4 next hop is PE2.

label 21:
VPN label learned from PE2.

nexthop 10.1.1.2 Ethernet0/0:
Send the packet to P1.

label 17:
Transport label used to reach PE2.
```

The imposed stack is:

```text
Top label:    17
Bottom label: 21
```

---

## Why the VRF Route Points to the PE Loopback

A common point of confusion is this output:

```text
B        11.11.11.0 [200/0] via 4.4.4.4
```

At first, it looks like PE1 is routing the customer packet to `4.4.4.4`.

That is not exactly what happens.

The route is a VPN route learned from PE2.

The next hop identifies the egress PE that advertised the route.

The real forwarding action is built from two pieces:

```text
VPN route:
Destination customer prefix is behind PE2.
Use VPN label 24.

Transport route:
PE2 is reachable through P1.
Use transport label 20.
```

So the BGP next hop is used for transport resolution. It is not the final customer destination.

This is why the provider core routers don't need to know customer routes.
They just need to be able to forward labeled packets between the PE loopbacks using the transport label.

---

## Forwarding Across the Core

After PE1 imposes the labels, the packet enters the MPLS core.

```text
PE1 → P1:

Top label:    17
Bottom label: 21
Payload:      Customer IP packet
```

P1 performs an LFIB lookup on the top label.

- P1 does not perform an IP lookup on `11.11.11.1`.
- P1 does not look in a VRF.
- P1 does not use MP-BGP VPNv4.

P1 only uses the top label.

Example P1 LFIB entry:

```text
P1# show mpls forwarding-table 4.4.4.4

Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop    
Label      Label      or Tunnel Id     Switched      interface              
17         16         4.4.4.4/32       5705          Et0/1      10.1.2.2 
```

P1 swaps the transport label:

```text
Incoming top label: 17
Outgoing top label: 16

VPN label: unchanged
Customer IP packet: unchanged
```

After P1 forwards the packet:

```text
P1 → P2:

Top label:    16
Bottom label: 21
Payload:      Customer IP packet
```

The transport label can change at each hop (or remain the same, depending on the labels each router independently assigned).

The VPN label stays the same until the packet reaches the egress PE.

---

## Penultimate Hop Popping

P2 is the penultimate router before PE2.

If PE2 advertises implicit null for its loopback FEC (default), P2 pops the transport label before forwarding the packet to PE2.

P2 LFIB entry:

```text
P2# show mpls forwarding-table 4.4.4.4

Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop    
Label      Label      or Tunnel Id     Switched      interface              
16         Pop Label  4.4.4.4/32       5523          Et0/0      10.2.2.2  
```

So P2 pops the transport label and forwards the packet like this:

```text
Top label (VPN):    21
Payload:            Customer IP packet
```

The VPN label is now the only remaining label.

That is normal.

The transport label did its job: it got the packet to the egress PE.

The VPN label still needs to reach the egress PE.

```text
Transport label:
Can be popped by the penultimate hop.

VPN label:
Must be processed by the egress PE.
```

The penultimate P router must not pop the VPN label.

It does not own the VPN label and does not know what that VPN label means.

---

## Egress PE Forwarding

PE2 receives the packet from P2.

Depending on PHP, PE2 may receive:

```text
VPN label only
```

or:

```text
Transport label + VPN label
```

In the common PHP case, PE2 receives only the VPN label:

```text
Top label: 21
Payload:   Customer IP packet
```

PE2 looks up label `21` in its LFIB.

Example:

```text
PE2# show mpls forwarding-table labels 21

Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop    
Label      Label      or Tunnel Id     Switched      interface              
21         No Label   11.11.11.0/24[V] 0             Et0/1      192.168.2.2 
```

This tells PE2:

```text
If a packet arrives with local label 21:
  Pop the label.
  Forward the IP packet toward 192.168.2.2.
  Use the VPN forwarding entry for 11.11.11.0/24.
```

The `[V]` indicates a VPN route.

After PE2 removes the VPN label, the packet is a normal IP packet again:

```text
Source:      10.10.10.1
Destination: 11.11.11.1
```

PE2 sends it to CE2.

---

## Per-Prefix VPN Label Forwarding

With per-prefix label allocation, the VPN label identifies a specific VPN route or forwarding entry.

Example:

```text
11.11.11.0/24 → label 21
11.11.12.0/24 → label 22
11.11.13.0/24 → label 23
```

If PE2 receives label `21`, it already knows the forwarding entry is for `11.11.11.0/24`.

Conceptually:

```text
Incoming VPN label 21
  ↓
Forward toward CE2 for 11.11.11.0/24
```

This avoids a second IP lookup in the VRF for the customer destination.

The label identifies the route.

That is why per-prefix allocation uses more labels but provides very direct forwarding.

---

## Per-VRF VPN Label Forwarding

With per-VRF label allocation, the VPN label identifies the VRF, not the specific customer prefix.

Example:

```text
CUSTOMER_A VRF → label 33
```

Routes in that VRF may all use the same VPN label:

```text
11.11.11.0/24 → label 33
11.11.12.0/24 → label 33
11.11.13.0/24 → label 33
```

If PE2 receives label `33`, it knows the packet belongs to `CUSTOMER_A`.

Then PE2 performs an IP lookup in the `CUSTOMER_A` VRF using the destination IP address.

Conceptually:

```text
Incoming VPN label 33
  ↓
Select VRF CUSTOMER_A
  ↓
Lookup destination 11.11.11.1 in CUSTOMER_A
  ↓
Forward toward CE2
```

Per-VRF labels use fewer labels.

The tradeoff is that the egress PE performs an additional VRF IP lookup.

---

## Disposition vs Imposition

It helps to separate the forwarding roles.

The ingress PE performs label **imposition**, pushing the transport and VPN labels onto the customer packet.

The P routers perform label swapping.

The egress PE performs label **disposition**, removing all remaining labels and forwarding the customer packet.

```text
Ingress PE:
IP packet in, labeled packet out.

P router:
Labeled packet in, labeled packet out.

Egress PE:
Labeled packet in, IP packet out.
```

Example:

```text
CE1 → PE1:
IP packet

PE1 → P1:
Transport label + VPN label + IP packet

P1 → P2:
New transport label + VPN label + IP packet

P2 → PE2:
VPN label + IP packet

PE2 → CE2:
IP packet
```

Only the PE routers understand the VPN context.

The P routers only understand the transport label.

---

## Forwarding in the Return Direction

L3VPN forwarding is independent in each direction.

For traffic from CE2 back to CE1, the roles reverse:

```text
CE2 → PE2 → P2 → P1 → PE1 → CE1
```

Now:

```text
PE2:
Ingress PE.

PE1:
Egress PE.
```

PE1 allocates a VPN label for its local route `10.10.10.0/24`.

PE1 advertises that label to PE2 using MP-BGP VPNv4.

PE2 uses that label when sending return traffic.

Example:

```text
PE1 advertises:
VPN route: 65000:1:10.10.10.0/24
Next hop:  1.1.1.1
VPN label: 21

PE2 sends return traffic using:
Transport label to 1.1.1.1 (18)
VPN label 21
```

Here is PE2's CEF entry:

```
PE2#show ip cef vrf CUSTOMER_A 10.10.10.0/24 detail
10.10.10.0/24, epoch 0, flags [rib defined all labels]
  recursive via 1.1.1.1 label 21
    nexthop 10.2.2.1 Ethernet0/0 label 18-(local:19)
```

Forward and reverse traffic do not have to use the same labels, although they can if both PEs happen to assign the same labels.

> In this case PE1 assigned label `21` for `10.10.10.0/24`, and PE2 also assigned label `21` for `11.11.11.0/24`.

But they are separately allocated and locally significant.

---

## What Changes Hop by Hop

In a normal L3VPN packet, not every field changes at every hop.

### Transport Label

The transport label is changed hop by hop.

```text
PE1 → P1:
Transport label 17

P1 → P2:
Transport label 16

P2 → PE2:
Transport label popped by PHP
```

### VPN Label

The VPN label normally stays the same across the provider core.

```text
PE1 → P1:
VPN label 21

P1 → P2:
VPN label 21

P2 → PE2:
VPN label 21
```

The VPN label is meaningful to the egress PE that advertised it.

### Customer IP Header

The customer source and destination IP addresses do not change.

```text
Source:      10.10.10.1
Destination: 11.11.11.1
```

The customer packet is simply carried as the payload of the MPLS packet.

---

## MPLS Header Fields

Each MPLS label entry contains more than just the label value.

Important fields:

```text
Label:
20-bit label value.

Traffic Class:
Used for QoS marking.

S bit:
Bottom-of-stack bit.

TTL:
MPLS TTL.
```

For a two-label L3VPN packet:

```text
Transport label:
Label = transport label value
S bit = 0

VPN label:
Label = VPN label value
S bit = 1
```

The Traffic Class field may be copied or set according to QoS policy.

The MPLS TTL behavior depends on the TTL propagation configuration.

But the key point is the label stack:

```text
Top label:
Used by P routers.

Bottom label:
Used by egress PE.
```

---

## Load Sharing and Label Forwarding

The ingress PE may have multiple equal-cost paths to the remote PE loopback.

Example:

```text
PE1 has two equal-cost IGP paths to 4.4.4.4.
```

In that case, CEF can load-share labeled traffic across multiple next hops.

The VPN route still points to the same BGP next hop:

```text
11.11.11.0/24 via 4.4.4.4
```

But the transport resolution may produce multiple outgoing label operations.

Conceptually:

```text
VRF route:
11.11.11.0/24 via 4.4.4.4 label 21

Transport resolution:
Path 1 to 4.4.4.4 with transport label 17
Path 2 to 4.4.4.4 with transport label 18
```

The VPN label remains the label advertised by the egress PE.

The top transport label depends on which core next hop is selected.

---

## Route Reflectors and the Data Plane

A VPNv4 route reflector reflects control-plane routes.

It is usually not in the customer data path.

Example:

```text
        RR1
       /   \
     PE1   PE2
```

PE2 advertises a VPNv4 route to RR1.

RR1 reflects the route to PE1.

The BGP next hop usually remains PE2.

```text
Route received by PE1:
11.11.11.0/24 via 4.4.4.4
```

Traffic from PE1 goes toward PE2, not toward the route reflector.

The VPNv4 session may be PE1 to RR1, but the transport LSP must exist from PE1 to PE2.

---

## Forwarding Checklist

For traffic from CE1 to CE2, check the path in this order:

```text
1. Does PE1 have the customer route in the correct VRF?

2. Does the route have a remote VPN label?

3. Does PE1 have global reachability to the VPNv4 next hop?

4. Does PE1 have a transport label to the VPNv4 next hop?

5. Does the VRF CEF entry show the correct imposed label stack?

6. Do the P routers have LFIB entries for the transport label path?

7. Does the penultimate hop pop or forward the transport label correctly?

8. Does PE2 have a local VPN label entry for the bottom label?

9. Does PE2 have a route or adjacency toward CE2?

10. Does the return path work in the opposite direction?
```

Do not stop after checking the VRF route.

A correct route in the VRF does not automatically guarantee a working label stack.

---

## Common Misunderstandings

### The P routers need customer routes

They do not.

P routers only need provider reachability and transport labels.

They forward based on the top label.

### The VPN label gets the packet across the core

It does not.

The transport label gets the packet across the core.

The VPN label is for the egress PE.

### The RD or RT is carried in the packet

They are not.

RDs and RTs are control-plane values.

The data plane uses MPLS labels.

### The route reflector is the data-plane next hop

Usually, it is not.

The route reflector reflects VPNv4 routes and improves scalability.

The BGP next hop usually remains the originating PE.

### The VPN label is globally meaningful

It is not.

The VPN label is locally significant to the egress PE that advertised it.

Remote PEs use the label because the egress PE told them to.

### PHP removes all labels

It does not.

PHP normally removes the transport label.

The VPN label remains and is processed by the egress PE.

---

## Key Points

* L3VPN forwarding begins with a VRF lookup on the ingress PE.
* The ingress PE resolves the VPN route's BGP next hop through the global routing table.
* The ingress PE imposes a VPN label and a transport label.
* The VPN label is advertised by the egress PE using MP-BGP VPNv4.
* The transport label is learned through LDP.
* P routers forward using only the top transport label.
* P routers do not need VRFs, customer routes, RDs, RTs, or MP-BGP.
* The transport label can change at every hop.
* The VPN label stays the same until it reaches the egress PE.
* PHP may remove the transport label before the packet reaches the egress PE.
* The egress PE uses the VPN label to select the correct VPN forwarding entry.
* With per-prefix labels, the VPN label identifies a specific route or forwarding entry.
* With per-VRF labels, the VPN label identifies the VRF, and the egress PE performs a VRF IP lookup.
* The customer IP packet is carried as the payload across the MPLS core.
* A working L3VPN data plane requires both the VPN label and the transport label.
