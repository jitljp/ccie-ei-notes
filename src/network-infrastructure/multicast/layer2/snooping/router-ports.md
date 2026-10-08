# Multicast Router Ports

An IGMP snooping switch must identify not only ports that lead toward multicast receivers, but also ports that lead toward **multicast routers**.

These router-facing ports are called:

```text
multicast router ports
mrouter ports
```

For example:

```text
                 R1
                 |
               E0/0
                 |
                SW1
              /     \
           E0/1     E0/2
             |        |
           Rec1      Rec2
```

If Ethernet0/0 leads toward R1, SW1 can treat it as an **mrouter port**.

This is important because multicast routers must receive IGMP membership information and may also need multicast data traffic.

## Why Mrouter Ports Are Needed

Suppose Rec1 sends an IGMP Membership Report:

```text
Rec1
 |
 | Membership Report
 v
SW1
 |
 | ?
 v
R1
```

The switch must know which port leads toward the multicast router so that the Report can be delivered appropriately.

Likewise, when multicast data arrives:

```text
Multicast G
     |
    SW1
```

the switch may need to forward it toward:

```text
listener ports
+
mrouter ports
```

Catalyst IOS XE adds multicast-router ports to the forwarding state for Layer 2 multicast entries.

Conceptually:

```text
G
├── listener port E0/1
├── listener port E0/2
└── mrouter port   E0/3
```

## Dynamic Mrouter-Port Learning

Mrouter ports can be learned dynamically.

Catalyst IOS XE learns multicast-router ports by snooping multicast control traffic such as:

```text
IGMP Queries
PIM packets
```

For example:

```text
R1
 |
 | IGMP General Query
 v
E0/0
 SW1
```

SW1 sees the Query arrive on Ethernet0/0 and can infer:

```text
Ethernet0/0
→ multicast router reachable here
```

Ethernet0/0 therefore becomes a dynamically learned mrouter port.

## Learning from IGMP Queries

An IGMP Query provides strong evidence that a multicast router is reachable through the ingress port.

For example:

```text
R1
 |
 | General Query
 v
SW1 E0/0
```

results conceptually in:

```text
VLAN 10
mrouter port
→ Ethernet0/0
```

The same principle applies to other IGMP Queries received from a multicast router.

This allows IGMP snooping to learn the port toward the **IGMP querier** automatically.

You can view dynamically learned (and statically configured) mrouter ports with:

```
SW3# show ip igmp snooping mrouter 
Vlan    ports
----    -----
  10    Et0/0(dynamic)
```

## Why PIM Traffic Is Also Used

Learning only from IGMP Queries would not be sufficient on a LAN containing multiple multicast routers.

Consider:

```text
  R1
   |
   |
  SW1----Receivers
   |
   |
  R2
```

Suppose both R1 and R2 run multicast routing.

Only one router becomes the **IGMP querier**.

For example:

```text
R1 → IGMP querier
R2 → non-querier
```

R1 sends periodic IGMP Queries:

```text
R1
 |
 | IGMP Query
 v
SW1
```

so SW1 can easily discover R1's port as an mrouter port.

However, R2 normally does not send periodic Queries while it is the non-querier.

If IGMP Query snooping were the only discovery mechanism:

```text
R1 → discovered
R2 → potentially not discovered
```

This is why PIM traffic is also useful for mrouter discovery.

If R2 sends PIM control traffic:

```text
R2
 |
 | PIM
 v
SW1
```

SW1 can identify R2's port as another mrouter port.

Therefore:

```text
IGMP Query
→ discovers the IGMP querier

PIM traffic
→ can also identify other multicast routers
```

This allows multiple multicast-router ports to be learned on the same VLAN.

## PIM-Based Mrouter Discovery Is Not PIM Snooping

Do not confuse IGMP snooping observing PIM to discover an mrouter port with the separate feature:

```text
PIM snooping
```

They serve different purposes.

Basic mrouter discovery uses PIM traffic simply to determine:

```text
"A multicast router is reachable through this port."
```

It does not necessarily determine which multicast groups that router actually wants.

For example:

```text
            SW1
          /  |  \
        R1  R2  R3
```

Basic IGMP snooping may learn:

```text
E0/1 → mrouter
E0/2 → mrouter
E0/3 → mrouter
```

and multicast traffic can therefore be forwarded to all three mrouter ports.

**PIM snooping**, covered later, goes further by examining PIM Join/Prune state to determine which mrouter ports actually need particular multicast groups.

Conceptually:

```text
Mrouter discovery:
E0/1 → router
E0/2 → router
E0/3 → router
```

versus:

```text
PIM snooping:
G → E0/1, E0/3
```

because perhaps R2 has no downstream interest in `G`.

## Multiple Mrouter Ports

A VLAN can have multiple multicast-router ports.

For example:

```text
            R1
             |
           E0/1
             |
            SW1
          /     \
       E0/2     E0/3
        |         |
       R2         R3
```

SW1 may learn:

```text
VLAN 10 mrouter ports:
E0/1
E0/2
E0/3
```

Under basic IGMP snooping, multicast-router ports are included in multicast forwarding state.

For example:

```text
Group 239.1.1.1

Listener ports:
E0/4
E0/5

Mrouter ports:
E0/1
E0/2
E0/3
```

Known multicast traffic can therefore be forwarded to:

```text
E0/4
E0/5
E0/1
E0/2
E0/3
```

unless another feature, such as **PIM snooping**, further restricts router-facing forwarding.

## IGMP Reports and Mrouter Ports

Membership Reports originate from receivers but need to reach multicast routers.

For example:

```text
Rec1
 |
 | Report for 239.1.1.1
 v
SW1
 |
 | mrouter port
 v
R1
```

The snooping switch therefore forwards IGMP membership information toward its learned mrouter ports.

Conceptually:

```text
Report received on listener port
        ↓
process snooping membership
        ↓
forward toward mrouter ports
```

However, features such as **IGMP Snooping Report Suppression **can change exactly how many Reports are forwarded.

Report Suppression is covered separately.

## Queries from Mrouter Ports

IGMP Queries received from the multicast-router side must reach receivers.

For example:

```text
R1
 |
 | General Query
 v
SW1
/   \
v   v
Rec1 Rec2
```

A General Query is intended to discover or refresh membership across the VLAN, so it must reach potential receivers.

The switch also observes the Query for snooping purposes:

```text
Query received on E0/0
        ↓
learn/refresh E0/0 as mrouter port
```

Thus, the same Query serves both:

```text
IGMP function
→ ask receivers for membership state

IGMP snooping function
→ identify/refresh router-facing port
```

## Static Mrouter Ports

An mrouter port can also be configured statically.

For VLAN 10:

```text
SW1(config)# ip igmp snooping vlan 10 mrouter interface Ethernet0/1
```

This tells the switch:

```text
VLAN 10
Ethernet0/0
→ multicast router port
```

View the result with:

```
SW3# show ip igmp snooping mrouter 
Vlan    ports
----    -----
  10    Et0/0(dynamic), Et0/1(static)
```

The switch does not need to dynamically discover multicast control traffic on the port first.

Remove the configuration with:

```text
SW1(config)# no ip igmp snooping vlan 10 mrouter interface Ethernet0/1
```

Static mrouter configuration is useful when:

```text
dynamic discovery is unreliable
the connected device does not generate expected discovery traffic
deterministic forwarding is desired
```

Static mrouter ports remain configured until the administrator removes them.

## Dynamic vs Static Mrouter Ports

The two methods can be summarized as:

```text
Dynamic mrouter port
→ learned from multicast control traffic
→ IGMP Queries / PIM packets

Static mrouter port
→ administratively configured
→ ip igmp snooping vlan ... mrouter interface ...
```

A static mrouter port does not depend on the switch continuing to observe control-plane traffic in order to remain configured.

## Static Mrouter vs Static Listener Port

Do not confuse:

```text
SW1(config)# ip igmp snooping vlan 10 mrouter interface Ethernet0/1
```

with:

```text
SW1(config)# ip igmp snooping vlan 10 static 239.1.1.1 interface Ethernet0/1
```

The first means:

```text
A multicast router is reachable through Ethernet0/1.
```

The second means:

```text
A receiver for 239.1.1.1 is reachable through Ethernet0/1.
```

Therefore:

```text
mrouter
→ router-facing state

static group
→ receiver-facing state
```

They influence multicast forwarding for different reasons.

## Mrouter Ports and Unknown Multicast

Mrouter ports are also important when the switch receives multicast traffic for a group for which no listener membership has been learned.

For example:

```text
239.9.9.9 arrives
        ↓
no listener ports known
```

Even if no host-facing listener exists, a multicast router may still need to receive the traffic.

The handling of this case depends on the switch's **unknown multicast forwarding** behavior and is covered separately.

## Key Points

```text
Mrouter port
→ Layer 2 port leading toward a multicast router

Dynamic learning
→ IGMP Queries
→ PIM packets

IGMP Queries
→ identify the querier's port

PIM traffic
→ can identify other multicast routers that are not the IGMP querier

Static configuration
→ ip igmp snooping vlan <vlan> mrouter interface <interface>

IGMP Reports
→ forwarded toward mrouter ports

Known multicast
→ forwarded to listener ports
→ also forwarded toward mrouter ports

Basic mrouter discovery
→ learns that a router exists on a port

PIM snooping
→ can further learn whether that router
   actually needs a particular multicast group
```

The next sections examine features that modify IGMP snooping behavior, beginning with **IGMP Snooping Report Suppression**.