# IGMP Snooping and PIM Snooping

Multicast presents a challenge for Layer 2 switches.

A normal switch learns where to forward **unicast** traffic by examining source MAC addresses. However, multicast receivers do not send frames using the multicast destination MAC address as a source address, so the switch cannot learn multicast MAC addresses through normal MAC address learning.

Without multicast-specific forwarding intelligence, multicast traffic is therefore **flooded throughout the VLAN**.

```text
              SW1
             / | \
            /  |  \
          PC1 PC2 PC3

          Multicast
             ↓
        Flooded to all ports
```

**IGMP snooping** allows a Layer 2 switch to examine IGMP messages exchanged between multicast receivers and multicast routers and build a more selective multicast forwarding table.

## IGMP Snooping

IGMP snooping operates at **Layer 2**, but the switch examines Layer 3 **IGMP control messages** passing through it.

The switch does not participate in IGMP as a normal multicast router.

Instead, it "snoops" messages such as:

```text
IGMP Queries
IGMP Membership Reports
IGMP Leave / State-Change messages
```

and uses them to determine:

```text
Which switch ports lead to multicast receivers?

Which switch ports lead to multicast routers?
```

The resulting forwarding behavior can be represented as:

```text
(VLAN, Multicast Group)
        ↓
Layer 2 outgoing port list
```

For example:

```text
VLAN 10
239.1.1.1
        ↓
Ethernet0/1
Ethernet0/3
```

means that multicast traffic for `239.1.1.1` should be forwarded only through the appropriate ports rather than flooded throughout VLAN 10.

## Multicast Listener Ports

When the switch receives an IGMP Membership Report from a host, it can learn that the incoming Layer 2 port leads to a receiver interested in that multicast group.

For example:

```text
Rec1
 |
 | IGMP Report for 239.1.1.1
 v
Ethernet0/1
   SW1
```

SW1 can learn:

```text
239.1.1.1
→ Ethernet0/1
```

That interface becomes a **multicast listener port** for the group.

If another receiver joins through Ethernet0/2:

```text
239.1.1.1
→ Ethernet0/1
→ Ethernet0/2
```

multicast data for the group can be forwarded only to those listener ports rather than every port in the VLAN.

## Multicast Router Ports

The switch must also identify ports that lead toward **multicast routers**.

These are commonly called:

```text
multicast router ports
mrouter ports
```

A switch can dynamically identify an mrouter port by observing multicast routing/control traffic, such as IGMP Queries or PIM messages.

An mrouter port can also be configured statically.

For example:

```text
                 R1
                 |
                 | mrouter port
                 |
              Ethernet0/0
                 SW1
              /       \
             /         \
          Rec1         Rec2
```

IGMP Reports from receivers must be forwarded toward the multicast routers so that the routers can learn receiver interest.

Therefore, IGMP snooping commonly maintains two important types of ports:

```text
Listener ports
→ lead toward multicast receivers

Mrouter ports
→ lead toward multicast routers
```

## Multicast Data Forwarding

After snooping has learned the required ports, multicast data can be selectively forwarded.

For example:

```text
                 R1
                 |
                 |
                SW1
              /  |  \
             /   |   \
           PC1  PC2  PC3
```

Suppose:

```text
PC1 → joined 239.1.1.1
PC2 → joined 239.1.1.1
PC3 → not joined
```

Without IGMP snooping:

```text
239.1.1.1
→ PC1
→ PC2
→ PC3
```

With IGMP snooping:

```text
239.1.1.1
→ PC1
→ PC2
```

PC3's port does not need to receive the multicast traffic.

This is the primary purpose of IGMP snooping:

> Restrict Layer 2 multicast forwarding to ports where the traffic is actually required.

This improves the efficiency of multicast communication within a LAN, ensuring multicast messages only reach interested receivers.

## IGMP Snooping Does Not Replace IGMP

IGMP snooping and IGMP itself have different roles.

```text
IGMP
→ Host ↔ multicast router protocol
→ communicates receiver membership

IGMP snooping
→ Layer 2 switch function
→ observes IGMP to optimize Layer 2 forwarding
```

The multicast router still maintains the Layer 3 IGMP membership state.

The Layer 2 switch uses the same IGMP messages to maintain its own multicast forwarding information.

## IGMP Snooping Does Not Require the Switch to Route Multicast

A Layer 2 switch can perform IGMP snooping without acting as a multicast router.

Conceptually:

```text
Receiver
   |
   | IGMP
   v
+------------------+
| Layer 2 Switch   |
|                  |
| snoops IGMP      |
| but does not     |
| route multicast  |
+------------------+
   |
   v
Multicast Router
```

The switch examines the IGMP messages while bridging them between hosts and routers.

## Leave Processing

IGMP snooping must also determine when a port should **stop** receiving a multicast group.

For example:

```text
Ethernet0/1
239.1.1.1
    ↓
receiver leaves
```

The switch cannot always immediately remove Ethernet0/1 because multiple receivers could exist behind the same port:

```text
          SW2
         /   \
       Rec1  Rec2
         |
         +--------- SW1 Ethernet0/1
```

A Leave from Rec1 does not necessarily mean Rec2 has also left.

IGMP snooping can therefore use mechanisms such as:

```text
Last-member Queries
Immediate Leave
Explicit Tracking
```

to determine when a listener port can safely be removed.

These mechanisms are covered in the following sections.

## IGMP Snooping Report Suppression

A snooping switch may also reduce the number of IGMP Reports forwarded toward multicast routers.

For IGMPv1 and IGMPv2, multiple hosts may send equivalent Reports for the same group.

With **IGMP snooping Report Suppression**, the switch can forward a representative Report toward the multicast router while suppressing redundant Reports.

```text
Rec1 ---- Report ----\
                      \
Rec2 ---- Report ------ SW1 ---- one Report ---- R1
                      /
Rec3 ---- Report ----/
```

IGMPv3 is different because its Reports can contain receiver-specific source-filter information.

IGMP snooping Report Suppression is therefore not used for IGMPv3 Reports.

## IGMP Snooping Explicit Tracking

IGMP snooping can optionally maintain more detailed membership information about the hosts behind individual Layer 2 interfaces.

This is **IGMP snooping explicit tracking**, a Layer 2 feature.

Do not confuse it with:

```text
ip igmp explicit-tracking
```

which is a Layer 3 IGMP router feature.

Snooping explicit tracking can improve leave processing because the switch has more precise knowledge of which receivers remain behind a port.

## Unknown Multicast Traffic

IGMP snooping can create forwarding entries only after it has learned useful receiver state.

The switch must therefore also have a policy for multicast traffic whose destination group has **no known snooping entry**.

This is commonly called **unknown multicast**.

Depending on the platform and configuration, unknown multicast may be flooded or subject to more restrictive forwarding behavior.

This is covered separately in **Unknown Multicast Forwarding**.

## Link-Local and Reserved Multicast

Not all IPv4 multicast traffic represents ordinary receiver-driven application groups.

Protocols such as routing protocols use addresses from:

```text
224.0.0.0/24
```

for local control traffic.

Examples include:

```text
224.0.0.5  → OSPF AllSPFRouters
224.0.0.6  → OSPF AllDRouters
224.0.0.22 → IGMPv3 Routers
```

These groups require special consideration because devices generally do not use ordinary IGMP membership signaling to join them.

Their Layer 2 handling is covered in **Reserved and Link-Local Multicast Groups**.

## Topology Changes

Layer 2 topology changes can invalidate multicast forwarding information just as they can affect normal MAC address learning.

For example:

```text
STP topology changes
        ↓
previous multicast path may no longer be valid
        ↓
temporary flooding / relearning may be required
```

IOS XE provides IGMP snooping behavior and tuning related to STP Topology Change Notifications (TCNs).

This is covered with the core IGMP snooping operation and configuration.

## PIM Snooping

IGMP snooping primarily solves the problem of forwarding multicast traffic toward **hosts**.

However, consider a Layer 2 network interconnecting several multicast routers:

```text
             R1
              \
               \
                SW1
              /  |  \
            R2   R3   R4
```

IGMP snooping may identify all of these interfaces as **multicast router ports**.

Without additional intelligence, multicast traffic can be forwarded to all mrouter ports even when only some routers have downstream receivers for a particular group.

**PIM snooping** extends the snooping concept to PIM control traffic.

The Layer 2 switch examines messages such as:

```text
PIM Hellos
PIM Joins
PIM Prunes
```

and can determine which multicast router ports actually need traffic for a particular group.

Conceptually:

```text
Without PIM snooping:

Multicast G
→ R2
→ R3
→ R4


With PIM snooping:

R2 has downstream interest
R3 has no downstream interest
R4 has downstream interest

Multicast G
→ R2
→ R4
```

PIM snooping is especially useful when a Layer 2 switch interconnects multiple multicast routers, such as in an **Internet Exchange Point (IXP)** or another shared Layer 2 multicast segment.

## IGMP Snooping vs PIM Snooping

The two features optimize different parts of Layer 2 multicast forwarding:

| Feature | Snoops | Primarily Optimizes |
|---|---|---|
| IGMP snooping | IGMP | Traffic toward multicast receivers |
| PIM snooping | PIM | Traffic toward multicast routers |

Consider a Layer 2 switch interconnecting several multicast routers:

```text
                         Multicast Source
                               |
                              R1
                               |
                         mrouter port
                               |
                              SW1
                     /          |          \
                    /           |           \
             mrouter port  mrouter port  mrouter port
                  |             |             |
                 R2            R3            R4
                  |                           |
             receivers                   receivers
              for G                        for G
```

IGMP snooping can identify the router-facing interfaces as **mrouter ports**, but multicast traffic is normally forwarded to all mrouter ports.

Therefore, without PIM snooping:

```text
Traffic for G
→ R2
→ R3
→ R4
```

even if R3 has no downstream receivers for `G`.

With PIM snooping, SW1 examines PIM control traffic and can learn:

```text
R2 → downstream interest in G
R3 → no downstream interest in G
R4 → downstream interest in G
```

SW1 can then restrict multicast data for `G` to:

```text
Traffic for G
→ R2
→ R4
```

Therefore:

```text
IGMP snooping
→ restricts multicast toward host/listener ports

PIM snooping
→ further restricts multicast toward mrouter ports
```

PIM snooping is particularly useful when a Layer 2 switch interconnects multiple multicast routers, such as on a shared Layer 2 multicast segment or an Internet Exchange Point (IXP).

## Section Overview

The following sections examine:

```text
IGMP Snooping
→ Core Layer 2 snooping behavior and configuration

Multicast Listener Ports
→ How receiver-facing ports are learned and maintained

Multicast Router Ports
→ How router-facing ports are learned and configured

IGMP Snooping Report Suppression
→ Suppressing redundant IGMPv1/v2 Reports

IGMP Snooping Immediate Leave
→ Removing listener ports without Last Member Queries

IGMP Snooping Explicit Tracking
→ Tracking receiver membership more precisely

Unknown Multicast Forwarding
→ Handling groups with no learned forwarding entry

Reserved and Link-Local Multicast Groups
→ Special handling of control multicast groups

PIM Snooping
→ Restricting multicast forwarding among multicast-router ports
```