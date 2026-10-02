# Multicast Overview

**Multicast** is a method of delivering the same traffic from one or more sources to multiple receivers.

Instead of sending a separate copy of a packet to each receiver, the source sends one packet to a **multicast group address**.

The network replicates the packet only where necessary to reach interested receivers.

```text
Unicast:

Source
  |
  +---- copy 1 ----> Receiver 1
  |
  +---- copy 2 ----> Receiver 2
  |
  +---- copy 3 ----> Receiver 3
```

```text
Multicast:

Source
  |
  | one stream
  |
Router
  |
  |     +--------------> Receiver 1
  |     |
 Switch +--------------> Receiver 2
        |
        +--------------> Receiver 3
```

This allows multicast to efficiently deliver the same traffic to many receivers.

Common multicast applications include:

```text
Live video
IPTV
Market data
Audio/video conferencing
Software distribution
Routing protocols
```

---

## Unicast, Broadcast, and Multicast

IPv4 supports several methods of delivering packets.

| Type | Delivery |
| --- | --- |
| Unicast | One sender to one receiver |
| Broadcast | One sender to all hosts on a subnet |
| Multicast | One sender to a selected group of receivers |

### Unicast

With unicast, every packet has a destination address identifying a single receiver.

```text
Source -----------> Receiver
```

If the same data must be sent to three receivers, the source normally sends three independent streams.

```text
      |----------------> Receiver 1
      |       
Source-----------------> Receiver 2
      |
      |----------------> Receiver 3
```

This can waste bandwidth when the traffic is identical.

### Broadcast

A broadcast is delivered to all hosts in the local broadcast domain.

```text
              +----------> Host 1
              |
Source -------+----------> Host 2
              |
              +----------> Host 3
```

Every host must receive and process the broadcast even if it is not interested in the traffic.

Broadcasts are also generally limited to the local subnet.


> A directed broadcast can be used to send a message to all hosts in a remote subnet, 
> although its use is limited in modern networks.

### Multicast

Multicast sends traffic only toward network segments with interested receivers.

```text
                            Receiver 1
                               ^
                               |
Source ---- R1 ---- R2 --------+
            |
            |
            +---- R3 --------> Receiver 2
```

The source sends one stream.

R2 and R3 have interested receivers connected, so R1 duplicates the message and forwards it to R2 and R3,
which forward the message to the interested receivers in their LANs.

> The sender does not need to send a separate copy of every packet to every receiver. 
> It sends a single stream of packets, regardless of how many hosts are interested in receiving the stream.

---

## Multicast Groups

Multicast traffic is sent to a **multicast group**.

A multicast group is identified by a multicast IP address.

IPv4 multicast addresses are in **224.0.0.0/4**, the **class D** range.

This covers 224.0.0.0 to 239.255.255.255

The multicast group address identifies a group of receivers.

It does **not** identify a particular physical device.

For example:

```text
Multicast group: 239.1.1.1
```

Multiple receivers can request traffic for the same group.

```text
Host A joins 239.1.1.1
Host B joins 239.1.1.1
Host C joins 239.1.1.1
```

A source can then send traffic to 239.1.1.1,
and the multicast network delivers that traffic toward the interested receivers.

> IPv4 multicast addressing is covered in more detail on the next pages.

---

## Sources and Receivers

Multicast has two fundamental roles:

| Role | Function |
| --- | --- |
| Source | Sends multicast traffic |
| Receiver | Requests and receives multicast traffic |

For example:

```text
Source: 10.1.1.10
Group: 239.1.1.1

Receiver 1 wants 239.1.1.1
Receiver 2 wants 239.1.1.1
```

The traffic can be described as:

```text
Source = 10.1.1.10
Group  = 239.1.1.1
```

or:

```text
(S,G)

(10.1.1.10, 239.1.1.1)
```

where:

```text
S = Source
G = Group
```

A multicast source does not normally need to know which receivers exist.

It simply sends traffic to the multicast group.

The multicast network determines where copies of the traffic need to be delivered using protocols like IGMP and PIM.

---

## Joining a Multicast Group

Receivers indicate that they want multicast traffic by **joining a multicast group**.

For IPv4, hosts use **IGMP** (Internet Group Management Protocol).

For example:

```text
Host
 |
 | "I want traffic for 239.1.1.1"
 | IGMP Membership Report
 v
Router
```

The router now knows that the local network has a receiver interested in:

```text
239.1.1.1
```

The router can then build multicast forwarding state so that traffic for that group reaches the receiver.

For IPv6, the equivalent protocol is **MLD** (Multicast Listener Discovery).

```text
IPv4 receiver membership = IGMP
IPv6 receiver membership = MLD
```

> IGMP does not route multicast traffic between routers. It communicates receiver membership information between hosts and their local multicast router.

---

## IGMP, PIM, and IGMP Snooping

Three important technologies have different roles in an IPv4 multicast network.

| Technology | Main Role |
| --- | --- |
| IGMP | Communicates group membership between hosts and routers |
| IGMP Snooping | Controls Layer 2 multicast forwarding on switches |
| PIM | Builds multicast forwarding paths between routers |

The basic relationship is:

```text
Receiver
   |
   | IGMP
   |
Switch <-- IGMP Snooping runs on the switch
   |
   | IGMP
   |
Router
   |
   | PIM neighbors
   |
Router
   |
   | PIM neighbors
   |
Router
   |
   |
   |
Source
```

### IGMP

IGMP tells the local router which multicast groups have interested receivers.

```text
Receiver ---- IGMP ---- Router
```

### IGMP Snooping

A Layer 2 switch can inspect IGMP messages to determine which switch ports have interested multicast receivers.

Without IGMP snooping, multicast traffic is flooded throughout the VLAN.

With IGMP snooping, the switch can forward multicast traffic only toward the ports where it is needed.

```text
            Host A
              |
              | interested
              |
Source ---- Switch
              |
              | X not interested
              |
            Host B
```

> Even without IGMP Snooping, multicast is more efficient in a LAN than broadcast.
> This is because a host's NIC will discard multicast frames for groups
> the host is not interested in receiving, with no further processing needed.

### PIM

**PIM** stands for **Protocol Independent Multicast**.

PIM operates between multicast routers.

```text
Router ---- PIM ---- Router ---- PIM ---- Router
```

PIM creates the multicast distribution tree that connects sources and receivers.

PIM is called **protocol independent** because it is not tied to one particular unicast routing protocol.

It can use routes learned through protocols such as:

```text
OSPF
EIGRP
IS-IS
BGP
Static routes
```

PIM uses the existing unicast routing information when determining the upstream direction for multicast traffic.

> PIM does not replace the unicast routing table. It is **dependent** on the unicast routing table,
> but **independent** of any particular unicast routing protocol.

---

## First-Hop and Last-Hop Routers

Two important multicast router roles are the **First-Hop Router** and **Last-Hop Router**.

### First-Hop Router

The **First-Hop Router (FHR)** is the multicast router directly connected to the source.

```text
Source ---- FHR ---- multicast network
```

The FHR receives multicast traffic from the source and introduces it into the multicast routing domain.

### Last-Hop Router

The **Last-Hop Router (LHR)** is the multicast router directly connected to the receiver network.

```text
multicast network ---- LHR ---- Receiver
```

The LHR learns that receivers are interested in multicast groups, normally through IGMP.

For a simple topology:

```text
Source ---- R1 ---- R2 ---- R3 ---- Receiver
            ^               ^
            |               |
           FHR             LHR
```

R1 is the FHR. R3 is the LHR.

---

## Multicast Distribution Trees

Multicast routers build **distribution trees** connecting multicast sources to receivers.

Traffic flows from the source down the tree toward the receivers.

```text
                   R3 ---- Receiver 1
                   /
Source ---- R1 -- R2
                   \
                   R4 ---- Receiver 2
```

The routers do not simply flood multicast traffic everywhere.

They maintain multicast state describing where traffic should arrive and where it should be forwarded.

Two important kinds of multicast trees are:

```text
Source tree
Shared tree
```

### Source Tree

A source tree is rooted at the multicast source.

It is often called a **Shortest Path Tree (SPT)**.

```text
         Source
           |
           R1
          /  \
        R2    R3
        |      |
      Rec1    Rec2
```

Source-tree state is represented as:

```text
(S,G)
```

For example:

```text
(10.1.1.10, 239.1.1.1)
```

### Shared Tree

A shared tree is built around a common router called the **Rendezvous Point (RP)**.

```text
Source
  |
  R1
  |
  RP
 /  \
R2   R3
|     |
Rec1 Rec2
```

Shared-tree state is represented as:

```text
(*,G)
```

For example:

```text
(*,239.1.1.1)
```

The `*` means:

```text
Any source
```

Distribution trees and multicast routing state are covered in detail on later pages.

---

## Multicast Routing State

A normal unicast routing decision is primarily based on the destination address.

```text
Destination address
        ↓
Routing/FIB lookup
        ↓
Outgoing interface
```

Multicast forwarding is different.

A multicast router must consider both:

```text
Where did the traffic come from?
Where does the traffic need to go?
```

A multicast routing entry therefore includes information such as:

```text
Source
Group
Incoming interface (IIF)
Outgoing interface list (OIL)
```

For example:

```text
(10.1.1.10, 239.1.1.1)

Incoming interface:
GigabitEthernet0/0

Outgoing interface list:
GigabitEthernet0/1
GigabitEthernet0/2
```

This means:

```text
Traffic from 10.1.1.10
to multicast group 239.1.1.1

should arrive on Gi0/0

and should be replicated out:
Gi0/1
Gi0/2
```

The outgoing interfaces are collectively called the **Outgoing Interface List (OIL)**.

```text
IIF = Incoming Interface
OIL = Outgoing Interface List
```

---

## Reverse Path Forwarding

One of the most important concepts in multicast routing is **Reverse Path Forwarding (RPF)**.

Multicast routers must ensure that multicast traffic arrives from the expected upstream direction.

Suppose R1 receives multicast traffic from this source:

```text
10.1.1.10
```

R1 checks how it would route a **unicast** packet toward 10.1.1.10.

```text
Route to 10.1.1.10
        ↓
Expected upstream interface
        ↓
RPF interface
```

If the multicast packet arrives on the expected interface, the RPF check succeeds.

```text
Packet arrives on RPF interface

RPF check = PASS
```

If it arrives on another interface:

```text
Packet arrives on wrong interface

RPF check = FAIL
```

In this case, the multicast packet is normally dropped. This prevents multicast forwarding loops.

```text
Unicast routing tells multicast:

"Which direction leads back toward the source?"
```

This is why the unicast routing table is so important to PIM.

> Multicast traffic flows away from the source, but the router checks the path in the reverse direction to verify that the packet arrived from the correct upstream path.

RPF is covered in detail in its own section.

---

## Upstream and Downstream

Multicast uses the terms **upstream** and **downstream** frequently.

**Upstream** means toward the source or toward the root of the current multicast tree.

**Downstream** means toward the receivers.

```text
Source
  |
  | downstream traffic
  v
 R1
  |
  v
 R2
  |
  v
Receiver
```

From R2's perspective:

```text
R1 = upstream
Receiver = downstream
```

Control-plane messages can travel in the opposite direction from the multicast data.

For example, PIM Join messages generally travel **upstream**.

```text
 Source
   |
 Router
   |
   ^ PIM Join
   |
 Router
   |
   ^ toward source/RP
```

The resulting multicast traffic travels **downstream**.

```text
Source
   |
   v multicast traffic
 Router
   |
   v
Receiver
```

---

## PIM Modes

PIM has multiple operating modes.

The most important are:

```text
PIM Dense Mode
PIM Sparse Mode
Bidirectional PIM
```

### PIM Dense Mode

**PIM Dense Mode (PIM-DM)** initially assumes receivers may exist throughout the network.

It uses a **push model**.

The source traffic is initially **pushed** throughout the multicast network, and routers then prune branches where the traffic is not needed.

```text
Source sends multicast traffic
        ↓
Traffic is flooded throughout the network
        ↓
Routers discover where no receivers exist
        ↓
Those branches are pruned
```

This is called a **flood-and-prune** model.

```text
PIM-DM = push traffic first, prune later
```

PIM-DM is useful for understanding multicast fundamentals but is not commonly used in modern large networks.

### PIM Sparse Mode

**PIM Sparse Mode (PIM-SM)** assumes receivers are sparsely populated throughout the network.

It uses a **pull model**.

Multicast traffic is not forwarded throughout the network by default. Instead, routers explicitly **pull** traffic toward interested receivers by sending PIM Join messages upstream.

```text
No receiver
    ↓
No multicast forwarding
```

And then:

```
Receiver joins
    ↓
Router sends PIM Join upstream
    ↓
Multicast tree is built
    ↓
Traffic is forwarded toward the receiver
```

> PIM-SM = pull traffic only where it is requested

PIM-SM uses a **Rendezvous Point (RP)** for Any-Source Multicast operation.

Once the LHR receives a multicast stream from a source, it can then build an SPT toward the source.

PIM-SM is the primary PIM mode for CCIE Enterprise Infrastructure.

### Bidirectional PIM

**Bidirectional PIM (PIM-Bidir)** uses a shared tree in both directions.

Sources and receivers use the same RP-rooted tree.

Unlike normal PIM-SM, PIM-Bidir does not switch to source-specific shortest-path trees.

PIM-DM and PIM-Bidir are covered briefly in later pages, but PIM-SM is the main focus for CCIE-EI.

---

## Rendezvous Point

PIM Sparse Mode uses a **Rendezvous Point (RP)** for **Any-Source Multicast (ASM)**.

The RP acts as a common meeting point between multicast sources and receivers.

Conceptually:

```text
Source side
     |
     v
     RP
     ^
     |
Receiver side
```

The receiver side initially builds a shared tree toward the RP.

The source's First-Hop Router also informs the RP about active sources.

This allows the multicast network to connect sources and receivers.

```text
Source ---- FHR ---- RP ---- LHR ---- Receiver
```

There are several ways routers can learn the RP:

```text
Static RP
Auto-RP
Bootstrap Router (BSR)
```

Group-to-RP mappings determine which RP is responsible for which multicast groups.

More advanced designs can use multiple RPs for redundancy, including **Anycast RP**.

---

## Any-Source Multicast

**Any-Source Multicast (ASM)** allows receivers to request traffic for a multicast group without specifying a particular source.

The receiver effectively says:

```text
"I want group 239.1.1.1."
```

It does not need to say:

```text
"I want group 239.1.1.1 specifically from source 10.1.1.10."
```

ASM therefore uses:

```text
(*,G)
```

state as part of the shared-tree process.

> (*,G) is pronounced **star comma G**.

An RP is used to help sources and receivers discover each other.

---

## Source-Specific Multicast

**Source-Specific Multicast (SSM)** allows a receiver to request multicast traffic from a specific source.

Instead of requesting only:

```text
G
```

the receiver requests:

```text
(S,G)
```

> (S,G) is pronounced **S comma G**.

For example:

```text
Source: 10.1.1.10
Group: 232.1.1.1
```

The receiver requests:

```text
(10.1.1.10, 232.1.1.1)
```

SSM does not require an RP.

```text
ASM:

LHR
   ↓ build tree to RP
RP
   ↓ build tree to FHR
FHR


SSM:

LHR already knows source
   ↓
Build tree directly toward source
```

IGMPv3 provides the source-filtering capabilities required for native IPv4 SSM receiver signaling.

SSM is simpler than ASM in several ways because the RP and source-registration mechanisms are not needed.

---

## Layer 2 Multicast

Multicast also has important Layer 2 behavior.

When an IPv4 multicast packet is sent over Ethernet, the multicast IPv4 destination is mapped to a multicast Ethernet MAC address.

```text
IPv4 multicast address
        ↓
Multicast Ethernet MAC address
```

A normal Layer 2 switch does not run PIM to determine where multicast receivers exist.

Without multicast-aware Layer 2 features, multicast frames are flooded in the VLAN.

**IGMP snooping** allows the switch to inspect IGMP messages and build Layer 2 multicast forwarding information.

```text
IGMP Membership Report
        ↓
Switch snoops message
        ↓
Switch learns which port wants group
        ↓
Multicast traffic forwarded to that port
```

Other Layer 2 multicast mechanisms include:

```text
IGMP snooping querier
PIM snooping
IGMP filtering
MLD snooping
```

These are covered in the Layer 2 multicast section.

---

## IPv4 and IPv6 Multicast

Both IPv4 and IPv6 support multicast.

The concepts are similar, but some protocols differ.

| Function | IPv4 | IPv6 |
| --- | --- | --- |
| Receiver membership | IGMP | MLD |
| Router-to-router multicast | PIM | PIM for IPv6 |
| Multicast group addresses | 224.0.0.0/4 | FF00::/8 |
| RPF | Yes | Yes |
| SSM | Yes | Yes |

IPv6 makes extensive use of multicast.

IPv6 does not use broadcast.

MLD performs the receiver-membership role that IGMP performs for IPv4.

---

## Multicast Control Plane and Data Plane

Like other routing technologies, multicast has a control plane and a forwarding plane.

### Control Plane

The multicast control plane learns:

```text
Which groups have receivers
Where sources are located
Which routers are upstream
Which interfaces are downstream
Which RP is responsible for a group
```

Protocols and mechanisms involved include:

```text
IGMP
MLD
PIM
RPF
RP discovery
MSDP
```

### Data Plane

The data plane forwards multicast packets according to the multicast forwarding state.

Conceptually:

```text
IGMP/PIM/RPF
     ↓
Build multicast state
     ↓
Multicast forwarding table
     ↓
Forward and replicate multicast packets
```

On Cisco IOS XE, important multicast information can be viewed with commands such as:

```text
R1# show ip mroute
IP Multicast Routing Table
Flags: D - Dense, S - Sparse, B - Bidir Group, s - SSM Group, C - Connected,
       L - Local, P - Pruned, R - RP-bit set, F - Register flag,
       T - SPT-bit set, J - Join SPT, M - MSDP created entry, E - Extranet,
       X - Proxy Join Timer Running, A - Candidate for MSDP Advertisement,
       U - URD, I - Received Source Specific Host Report, 
       Z - Multicast Tunnel, z - MDT-data group sender, 
       Y - Joined MDT-data group, y - Sending to MDT-data group, 
       G - Received BGP C-Mroute, g - Sent BGP C-Mroute, 
       N - Received BGP Shared-Tree Prune, n - BGP C-Mroute suppressed, 
       Q - Received BGP S-A Route, q - Sent BGP S-A Route, 
       V - RD & Vector, v - Vector, p - PIM Joins on route, 
       x - VxLAN group, c - PFP-SA cache created entry, 
       * - determined by Assert, # - iif-starg configured on rpf intf, 
       e - encap-helper tunnel flag, l - LISP decap ref count contributor
Outgoing interface flags: H - Hardware switched, A - Assert winner, p - PIM Join
                          t - LISP transit group
 Timers: Uptime/Expires
 Interface state: Interface, Next-Hop or VCD, State/Mode

(*, 239.1.1.1), 00:00:05/stopped, RP 0.0.0.0, flags: D
  Incoming interface: Null, RPF nbr 0.0.0.0
  Outgoing interface list:
    Ethernet0/2, Forward/Dense, 00:00:05/stopped, flags: 
    Ethernet0/1, Forward/Dense, 00:00:05/stopped, flags: 
    Ethernet0/0, Forward/Dense, 00:00:05/stopped, flags: 

(10.1.2.10, 239.1.1.1), 00:00:05/00:02:54, flags: PT
  Incoming interface: Ethernet0/0, RPF nbr 0.0.0.0
  Outgoing interface list:
    Ethernet0/1, Prune/Dense, 00:00:05/00:02:54, flags: 
    Ethernet0/2, Prune/Dense, 00:00:05/00:02:54, flags: A

(*, 224.0.1.40), 02:19:42/00:02:23, RP 0.0.0.0, flags: DCL
  Incoming interface: Null, RPF nbr 0.0.0.0
  Outgoing interface list:
    Ethernet0/2, Forward/Dense, 00:03:21/stopped, flags: 
    Ethernet0/1, Forward/Dense, 02:19:02/stopped, flags: 
    Ethernet0/0, Forward/Dense, 02:19:42/stopped, flags: 



R1# show ip mfib
Entry Flags:    C - Directly Connected, S - Signal, IA - Inherit A flag,
                ET - Data Rate Exceeds Threshold, K - Keepalive
                DDE - Data Driven Event, HW - Hardware Installed
                ME - MoFRR ECMP entry, MNE - MoFRR Non-ECMP entry, MP - MFIB 
                MoFRR Primary, RP - MRIB MoFRR Primary, P - MoFRR Primary
                MS  - MoFRR  Entry in Sync, MC - MoFRR entry in MoFRR Client,
                e   - Encap helper tunnel flag.
I/O Item Flags: IC - Internal Copy, NP - Not platform switched,
                NS - Negate Signalling, SP - Signal Present,
                A - Accept, F - Forward, RA - MRIB Accept, RF - MRIB Forward,
                MA - MFIB Accept, A2 - Accept backup,
                RA2 - MRIB Accept backup, MA2 - MFIB Accept backup

Forwarding Counts: Pkt Count/Pkts per second/Avg Pkt Size/Kbits per second
Other counts:      Total/RPF failed/Other drops
I/O Item Counts:   HW Pkt Count/FS Pkt Count/PS Pkt Count   Egress Rate in pps
Default
 (*,224.0.0.0/4) Flags:
   SW Forwarding: 0/0/0/0, Other: 0/0/0
   Ethernet0/0 Flags: NS
   Ethernet0/1 Flags: NS
   Ethernet0/2 Flags: NS
 (*,224.0.1.40) Flags: C
   SW Forwarding: 0/0/0/0, Other: 0/0/0
   Ethernet0/0 Flags: F IC NS
     Pkts: 0/0/0    Rate: 0 pps
   Ethernet0/1 Flags: F NS
     Pkts: 0/0/0    Rate: 0 pps
   Ethernet0/2 Flags: F NS
     Pkts: 0/0/0    Rate: 0 pps
 (*,239.1.1.1) Flags: C
   SW Forwarding: 0/0/0/0, Other: 0/0/0
   Ethernet0/0 Flags: F NS
     Pkts: 0/0/0    Rate: 0 pps
   Ethernet0/1 Flags: F NS
     Pkts: 0/0/0    Rate: 0 pps
   Ethernet0/2 Flags: F NS
     Pkts: 0/0/0    Rate: 0 pps
 (10.1.2.10,239.1.1.1) Flags:
   SW Forwarding: 1/0/100/0, Other: 9/3/6
   Ethernet0/0 Flags: A
```

---

## Multicast Routing Table

The multicast routing table can be viewed with:

```text
show ip mroute
```

A simplified entry might look conceptually like:

```text
(10.1.1.10, 239.1.1.1)

Incoming interface:
GigabitEthernet0/0

Outgoing interface list:
GigabitEthernet0/1
GigabitEthernet0/2
```

This tells the router:

```text
If traffic from 10.1.1.10
for group 239.1.1.1

arrives on Gi0/0,

forward copies out:
Gi0/1
Gi0/2
```

If there are no downstream receivers, the outgoing interface list may be empty.

Multicast forwarding state is dynamic.

It can change as:

```text
Receivers join
Receivers leave
Sources become active
Unicast routes change
PIM state changes
RP information changes
```

---

## Basic End-to-End Multicast Operation

Consider this topology:

```text
Source                                        Receiver
10.1.1.10                                      Host A
   |                                              |
   R1 ------------ R2 ------------ R3 ------------+
   ^                               ^
   |                               |
  FHR                             LHR
```

Suppose the source sends traffic to:

```text
239.1.1.1
```

A simplified PIM-SM process is:

```text
1. Host A decides to receive traffic for 239.1.1.1.

2. Host A sends an IGMP Membership Report.

3. R3 learns that its local network has an interested receiver.

4. R3 uses PIM to join the multicast distribution tree.

5. PIM messages travel upstream toward the RP/source as appropriate.

6. R1 receives multicast traffic from the source.

7. Multicast forwarding state is created through the network.

8. Routers perform RPF checks on arriving multicast packets.

9. The traffic is forwarded down the multicast distribution tree.

10. R3 forwards the traffic onto the receiver LAN.

11. Host A receives the multicast stream.
```

The important relationship is:

```text
IGMP tells the LHR that a receiver exists.

PIM builds the router-to-router distribution tree.

The unicast routing table provides RPF information.

The multicast forwarding table determines where packets are replicated.
```

---

## Packet Replication

One of the major advantages of multicast is that packets are replicated inside the network only when required.

For example:

```text
                           R3 ---- Receiver 1
                          /
Source ---- R1 ---- R2 ---
                          \
                           R4 ---- Receiver 2
```

The source sends only one packet toward R1.

R1 sends only one packet toward R2.

R2 reaches a branching point and creates two copies.

```text
Source → R1 → R2
               |
               +→ R3 → Receiver 1
               |
               +→ R4 → Receiver 2
```

If ten receivers exist behind R3, the WAN path toward R3 still does not require ten separate copies of the stream.

Replication happens as close to the receivers as the topology allows.

---

## Basic IPv4 Multicast Configuration

IPv4 multicast routing is enabled globally with:

```text
ip multicast-routing
```

PIM is then enabled on the Layer 3 interfaces that participate in multicast routing.

For example:

```text
interface GigabitEthernet0/0
 ip pim sparse-mode
```

```text
interface GigabitEthernet0/1
 ip pim sparse-mode
```

Enabling PIM on an interface also enables IGMP operation on that interface.

For ASM using PIM Sparse Mode, the routers also need an RP mapping.

A simple static example is:

```text
ip pim rp-address 10.10.10.10
```

A minimal PIM-SM configuration therefore looks like:

```text
ip multicast-routing

ip pim rp-address 10.10.10.10

interface GigabitEthernet0/0
 ip pim sparse-mode

interface GigabitEthernet0/1
 ip pim sparse-mode
```

The actual configuration depends on the multicast design.

Later sections cover:

```text
Static RP
Auto-RP
BSR
SSM
Anycast RP
MSDP
PIMv6 Anycast RP
Multicast multipath
```

---

## Multicast Big Picture

The major multicast mechanisms fit together like this:

```text
Receiver wants multicast traffic
        ↓
IGMP/MLD signals receiver interest
        ↓
Layer 2 switch uses snooping to control local forwarding
        ↓
Last-Hop Router learns that a receiver exists
        ↓
PIM builds multicast state upstream
        ↓
RPF determines the correct upstream direction
        ↓
Multicast distribution tree is formed
        ↓
Source traffic enters the tree
        ↓
Routers replicate traffic at branching points
        ↓
Traffic reaches interested receivers
```

For ASM using PIM Sparse Mode:

```text
Receiver
   ↓
IGMP
   ↓
LHR
   ↓
PIM Join
   ↓
RP
   ↑
Source registration
   ↑
FHR
   ↑
Source
```

For SSM:

```text
Receiver
   ↓
IGMPv3 identifies (S,G)
   ↓
LHR
   ↓
PIM Join directly toward source
   ↓
FHR
   ↓
Source
```

---

## Key Points

* Multicast efficiently sends the same traffic to multiple receivers.
* A multicast source sends traffic to a multicast **group address** rather than to individual receivers.
* IPv4 multicast addresses use `224.0.0.0/4`.
* A multicast group represents a set of interested receivers, not a single device.
* A source does not need to know which receivers exist.
* Receivers use **IGMP** to communicate IPv4 multicast group membership to their local router.
* IPv6 uses **MLD** instead of IGMP.
* **IGMP snooping** controls multicast forwarding on Layer 2 switches.
* **PIM** is used between routers to build multicast distribution trees.
* PIM is protocol independent because it can use routing information learned through different unicast routing protocols.
* The **First-Hop Router (FHR)** is connected to the multicast source.
* The **Last-Hop Router (LHR)** is connected to the multicast receivers.
* **RPF** verifies that multicast traffic arrives from the correct upstream direction.
* RPF normally depends on unicast routing information.
* `(S,G)` identifies multicast state for a specific source and group.
* `(*,G)` identifies shared-tree state for any source sending to a group.
* The **IIF** is the expected incoming interface for multicast traffic.
* The **OIL** is the list of outgoing interfaces toward receivers.
* A source tree is rooted at the multicast source.
* A shared tree is rooted at an RP.
* PIM Sparse Mode uses an RP for ASM.
* **ASM** allows receivers to request a group without identifying a specific source.
* **SSM** allows receivers to request traffic from a specific source and does not require an RP.
* Multicast packets are replicated at branching points in the network rather than requiring the source to send a separate stream to each receiver.