# IGMP Snooping

Without IGMP snooping, a Layer 2 switch has no normal MAC-learning mechanism for discovering which ports contain multicast receivers.

Multicast traffic is therefore flooded throughout the VLAN:

```text
                SW1
          /      |      \
         /       |       \
      Rec1      PC2      PC3

239.1.1.1
→ Rec1
→ PC2
→ PC3
```

**IGMP snooping** allows the switch to inspect IGMP control traffic and build multicast forwarding state.

With snooping:

```text
239.1.1.1
→ Rec1
```

while ports with no interested receivers can be excluded.

## Layer 2 Snooping of Layer 3 Control Traffic

IGMP snooping is a **Layer 2 switching feature**, but it examines Layer 3 IGMP messages.

The switch observes traffic such as:

```text
IGMP Queries
IGMP Membership Reports
IGMP Leave / State-Change messages
```

and uses that information to determine:

```text
Which ports lead toward receivers?

Which ports lead toward multicast routers?
```

The switch itself does not need to act as an IGMP router.

Conceptually:

```text
Host
 |
 | IGMP Report
 v
+----------------+
|      SW1       |
|                |
| Snoops IGMP    |
+----------------+
 |
 | IGMP Report
 v
Router
```

SW1 examines the Report while forwarding it toward the multicast router.

## Snooping Forwarding State

The switch builds multicast forwarding state per VLAN and multicast group.

For example:

```text
VLAN 10
239.1.1.1
→ Ethernet0/1
→ Ethernet0/2
```

means that Ethernet0/1 and Ethernet0/2 lead toward receivers for `239.1.1.1`.

The forwarding state therefore resembles:

```text
(VLAN, Group)
     ↓
Outgoing Layer 2 ports
```

Catalyst 9000 switches support **IGMPv3 Basic IGMPv3 Snooping Support (BISS)**.

Although IGMPv3 Reports can contain source-specific INCLUDE/EXCLUDE information, Catalyst 9000 IGMP snooping is based only on the **destination multicast group address**.

It does **not** maintain source-specific `(S,G)` snooping entries based on the multicast source IP address.

Therefore, at Layer 2:

```text
IGMPv3 snooping on Catalyst 9000

(VLAN, G)
→ listener ports
```

rather than:

```text
(VLAN, S, G)
→ listener ports
```

Do not confuse this with **Layer 3 multicast routing**, where Catalyst 9000 switches (and other devices supporting multicast routing)
can maintain normal `(*,G)` and `(S,G)` multicast routing entries.

## Basic Learning Process

Consider:

```text
                 R3
                 |
              E0/0
                 |
                SW1
              /     \
           E0/2     E0/3
             |        |
           Rec1      Rec1
```

R3 is the multicast router.

Rec1 wants to receive:

```text
239.1.1.1
```

### Step 1: IGMP Query

R1 sends an IGMP General Query.

```text
R1
 |
 | General Query
 v
SW1
```

SW1 observes the Query and can learn that:

```text
Ethernet0/0
→ multicast router port
```

A multicast-router-facing port is commonly called an:

```text
mrouter port
```

The General Query is forwarded throughout the VLAN so that receivers can respond.

```text
              R1
              |
          General Query
              |
             SW1
            /   \
           v     v
        Rec1     PC2
```

The details of mrouter-port learning are covered in **Multicast Router Ports**.

## Step 2: Membership Report

Rec1 responds with an IGMP Membership Report for:

```text
239.1.1.1
```

SW1 receives the Report on Ethernet0/1:

```text
Rec1
 |
 | Membership Report
 v
E0/1
 SW1
```

SW1 adds Ethernet0/1 to its snooping state for the group:

```text
VLAN 10
239.1.1.1
→ Ethernet0/1
```

Ethernet0/1 is now a **listener port** for `239.1.1.1`.

The Report is also delivered toward the multicast router through the appropriate mrouter port, subject to features such as **IGMP Snooping Report Suppression**.

## Step 3: Multicast Data Arrives

R1 later forwards multicast data for `239.1.1.1` to SW1.

Because SW1 has learned the interested port:

```text
239.1.1.1
→ Ethernet0/1
```

it can forward the multicast traffic only where required:

```text
                 R1
                 |
          239.1.1.1
                 |
                SW1
              /     \
             v       X
           Rec1     PC2
```

PC2 does not receive the multicast data.

## Listener Ports and Mrouter Ports

The two most important port types in the snooping forwarding process are:

```text
Listener port
→ leads toward one or more receivers

Mrouter port
→ leads toward a multicast router
```

Known multicast traffic is generally forwarded toward:

```text
interested listener ports
+
appropriate mrouter ports
```

Mrouter ports are important because other multicast routers may also need to receive multicast traffic or control-plane information.

The following pages examine both port types in detail.

## Reports Do Not Use Normal MAC Learning

Normal Ethernet learning works by examining the `Source MAC Address` of received frames.

That mechanism cannot learn multicast receivers because multicast MAC addresses appear as **destination** addresses, not as source addresses identifying the receiver.

IGMP snooping instead learns receiver location from:

```text
IGMP Membership Report
        +
ingress switch port
```

For example:

```text
Report for 239.1.1.1
received on Ethernet0/1

→ Ethernet0/1 has receiver interest in 239.1.1.1
```

This is control-plane-assisted Layer 2 forwarding rather than normal source-MAC learning.

## Multiple Receivers

Multiple switch ports can be associated with the same group:

```text
              SW1
          /    |    \
       E0/1  E0/2  E0/3
        |      |      |
      Rec1   Rec2    PC3
```

If Rec1 and Rec2 both join `239.1.1.1`:

```text
239.1.1.1
→ Ethernet0/1
→ Ethernet0/2
```

multicast data is replicated only onto those required outgoing interfaces.

```text
239.1.1.1
     |
    SW1
   /   \
  v     v
Rec1   Rec2
```

## Receivers Behind Another Switch

A snooping port does not necessarily correspond to a single receiver.

For example:

```text
             SW1
              |
            E0/1
              |
             SW2
            /   \
         Rec1   Rec2
```

If either Rec1 or Rec2 wants `239.1.1.1`, SW1 needs:

```text
239.1.1.1
→ Ethernet0/1
```

Therefore, an IGMP snooping entry fundamentally means:

> At least one receiver for this group is reachable through this port.

This is why a Leave received on a port cannot always cause the port to be removed immediately: another receiver may still exist behind it.

That behavior is covered in **IGMP Snooping Immediate Leave** and **IGMP Snooping Explicit Tracking**.

## IGMP Queries

IGMP Queries are important to snooping state maintenance.

A General Query allows receivers to refresh their memberships:

```text
General Query
      ↓
receivers send Reports
      ↓
switch refreshes snooping state
```

If membership is no longer refreshed, dynamic snooping state eventually ages out.

Queries therefore allow the snooping table to recover from situations such as:

```text
host disappears without sending a Leave
link fails
IGMP control packet is lost
topology changes
```

A network using dynamic IGMP snooping normally needs an IGMP querier somewhere in the VLAN so that memberships continue to be refreshed.

The querier itself is covered in **1.6.a (iii) IGMP Querier**.

## Leave Processing

A Leave or IGMPv3 state change can indicate that a receiver no longer wants particular multicast traffic.

However, the switch cannot always remove the listener port immediately.

For example:

```text
               SW1
                |
              E0/1
                |
               SW2
              /   \
           Rec1   Rec2
```

If Rec1 leaves group `G`, Rec2 may still want it.

The switch may therefore need to verify whether another listener remains before removing Ethernet0/1 from the forwarding entry.

IOS XE supports mechanisms including:

```text
Last Member Query processing
Immediate Leave
Explicit Tracking
```

These are covered in their dedicated sections.

## Enabling and Disabling IGMP Snooping

On current Catalyst IOS XE platforms, IGMP snooping is enabled by default:

```text
Globally  → Enabled
Per VLAN  → Enabled
```

Globally enable it with:

```text
SW1(config)# ip igmp snooping
```

Disable it globally with:

```text
SW1(config)# no ip igmp snooping
```

Per-VLAN control uses:

```text
SW1(config)# ip igmp snooping vlan 10
```

or:

```text
SW1(config)# no ip igmp snooping vlan 10
```

Global IGMP snooping must be enabled for per-VLAN snooping to operate.

## Static Group Membership

A Layer 2 port can be statically added to the snooping forwarding entry for a group:

```text
SW1(config)# ip igmp snooping vlan 10 static 239.1.1.1 interface Ethernet0/1
```

This creates static listener state:

```text
VLAN 10
239.1.1.1
→ Ethernet0/1
```

The switch forwards traffic for the group through Ethernet0/1 without requiring a dynamically learned Membership Report on that port.

Remove it with:

```text
SW1(config)# no ip igmp snooping vlan 10 static 239.1.1.1 interface Ethernet0/1
```

Static group membership is useful when a receiver cannot generate normal IGMP signaling or when deterministic forwarding is required.

Do not confuse a **static listener port** with a **static mrouter port**.

They perform different roles:

```text
static group
→ "a receiver is reachable through this port"

static mrouter
→ "a multicast router is reachable through this port"
```

## IGMP Snooping Timers

IOS XE maintains its own snooping parameters for membership processing.

Important global knobs include:

```text
SW1(config)# ip igmp snooping robustness-variable 3
SW1(config)# ip igmp snooping last-member-query-count 3
SW1(config)# ip igmp snooping last-member-query-interval 500
```

They can also be overridden for an individual VLAN.

For example:

```text
SW1(config)# ip igmp snooping vlan 10 robustness-variable 3
SW1(config)# ip igmp snooping vlan 10 last-member-query-count 3
SW1(config)# ip igmp snooping vlan 10 last-member-query-interval 500
```

The current Catalyst IOS XE defaults include:

```text
Robustness Variable           = 2
Last Member Query Count       = 2
Last Member Query Interval    = 1000 ms
```

These values influence how snooping membership is maintained and removed.

The detailed leave behavior is covered later.

## Spanning-Tree Topology Changes

A Layer 2 topology change can make existing snooping forwarding state temporarily inaccurate.

For example:

```text
Before:

SW1 ---- SW2
 |
receiver path


After STP convergence:

SW1 ---- SW3 ---- SW2
```

The previously learned multicast path may no longer reflect the new topology.

To avoid **blackholing** multicast traffic while receiver state is relearned, IOS XE can temporarily flood multicast traffic following an STP Topology Change Notification (TCN).

Conceptually:

```text
STP TCN
   ↓
temporarily flood multicast
   ↓
IGMP General Queries occur
   ↓
receivers send Reports
   ↓
snooping table relearned
   ↓
return to selective forwarding
```

## TCN Flood Query Count

The number of General Queries for which multicast remains in TCN flood mode is controlled by:

```text
SW1(config)# ip igmp snooping tcn flood query count 3
```

The default is:

```text
2 General Queries
```

So by default:

```text
TCN occurs
    ↓
multicast flooding
    ↓
General Query #1
    ↓
membership relearning
    ↓
General Query #2
    ↓
exit flood mode
```

This trades additional temporary flooding for protection against traffic loss while the topology and snooping state reconverge.

## TCN Query Solicitation

Waiting for the next periodic General Query could prolong TCN flood mode.

IOS XE can therefore solicit a new Query:

```text
SW1(config)# ip igmp snooping tcn query solicit
```

This causes the switch to send a special IGMP **global Leave** associated with group:

```text
0.0.0.0
```

to prompt the multicast router to send General Queries and speed snooping-table relearning.

By default:

```text
TCN query solicitation = Disabled
```

Disable the feature with:

```text
SW1(config)# no ip igmp snooping tcn query solicit
```

## Disabling TCN Flooding on a Port

TCN flooding is enabled on interfaces by default.

It can be disabled on a specific interface:

```text
SW1(config)# interface Ethernet0/1
SW1(config-if)# no ip igmp snooping tcn flood
```

Re-enable it with:

```text
SW1(config-if)# ip igmp snooping tcn flood
```

Disabling TCN flooding reduces unnecessary multicast traffic during topology changes, but it also removes the protection that temporary flooding provides against stale snooping state.

## Unknown Multicast

IGMP snooping can selectively forward traffic only when useful snooping state exists.

If multicast traffic arrives for a group with no learned forwarding entry:

```text
239.9.9.9
→ no snooping membership
```

it is treated as **unknown multicast**.

Its behavior is different from normal known multicast forwarding and is covered separately in **Unknown Multicast Forwarding**.

## Link-Local Multicast

Normal IGMP snooping membership learning should also not be blindly applied to all multicast destinations.

Protocols use the IPv4 Local Network Control Block:

```text
224.0.0.0/24
```

for multicast control traffic.

Examples include:

```text
224.0.0.5  → OSPF AllSPFRouters
224.0.0.6  → OSPF AllDRouters
224.0.0.22 → IGMPv3 Routers
```

These groups do not rely on ordinary receiver IGMP joins in the same way as application multicast groups.

Their handling is covered in **Reserved and Link-Local Multicast Groups**.

## Key Points

```text
IGMP snooping
→ Layer 2 feature
→ examines IGMP control traffic
→ does not replace IGMP routing

Membership Report
→ learns/refreshes listener-port state

IGMP Query / multicast-routing control traffic
→ helps identify mrouter ports and maintain state

Known multicast data
→ forwarded to interested listener ports
→ also forwarded as required toward mrouter ports

Leave / state change
→ may trigger verification before removing a port

STP topology change
→ can temporarily cause multicast flooding
→ memberships are relearned from IGMP signaling
```

The following sections examine **Multicast Listener Ports**, **Multicast Router Ports**, Report Suppression, Immediate Leave, Explicit Tracking, Unknown Multicast, and reserved multicast behavior in detail.