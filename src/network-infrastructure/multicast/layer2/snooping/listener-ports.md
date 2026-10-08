# Multicast Listener Ports

An IGMP snooping switch learns which Layer 2 ports lead toward multicast receivers.

These ports are commonly called:

```text
listener ports
member ports
host ports
```

For example:

```text
              SW1
          /         \
       E0/1         E0/2
        |             |
      Rec1           PC2
```

If Rec1 joins:

```text
239.1.1.1
```

SW1 can learn:

```text
VLAN 10
239.1.1.1
→ Ethernet0/1
```

Ethernet0/1 is now a listener port for that group.

Cisco describes IGMP snooping as adding the ingress port of a received IGMP Membership Report to the multicast forwarding-table entry for that group.

## Dynamic Listener-Port Learning

Listener ports are normally learned dynamically from IGMP Membership Reports.

For example:

```text
Rec1
 |
 | IGMP Report for 239.1.1.1
 v
E0/1
 SW1
```

SW1 processes the Report and creates or updates:

```text
VLAN 10
239.1.1.1
→ Ethernet0/1
```

If another receiver joins the same group through Ethernet0/2:

```text
VLAN 10
239.1.1.1
→ Ethernet0/1
→ Ethernet0/2
```

Multicast data for the group is then replicated only onto the required listener ports, plus any router-facing ports that must also receive it.

### Verifying Dynamically Learned Listener Ports

Use:

```text
SW3# show ip igmp snooping groups
Flags: I -- IGMP snooping, S -- Static, P -- PIM snooping, A -- ASM mode
       E -- EVPN sync

Vlan      Group/source             Type        Version     Port List
-----------------------------------------------------------------------
10        224.0.1.40               I           v3          Et0/0
10        239.1.1.1                I           v3          Et0/2
```

The entry:

```text
10        239.1.1.1                I           v3          Et0/2
```

shows that SW3 dynamically learned:

```text
VLAN 10
239.1.1.1
→ Ethernet0/2
```

The `I` flag indicates an **IGMP snooping** entry, and `v3` indicates that the membership was learned using IGMPv3.

For more detailed membership information, use:

```text
SW3# show ip igmp snooping membership
Snooping Membership Summary for Vlan 10
------------------------------------------
Total number of channels: 2
Total number of hosts   : 2

Source/Group                    Interface Reporter                        Uptime   Last-Join/
                                                                                  Last-Leave
-----------------------------------------------------------------------------------------------

0.0.0.0/224.0.1.40              Et0/0     10.3.4.3                        00:00:27 00:00:27 /
                                                                                      -

0.0.0.0/239.1.1.1               Et0/2     10.3.4.10                       00:00:31 00:00:31 /
                                                                                      -
```

This provides additional per-membership information.

For `239.1.1.1`:

```text
0.0.0.0/239.1.1.1
Et0/2
Reporter: 10.3.4.10
```

means that host `10.3.4.10`, reachable through Ethernet0/2, reported membership in `239.1.1.1`.

The source is shown as:

```text
0.0.0.0
```

because the snooping entry is **group-based**, not a source-specific `(S,G)` forwarding entry.

So the two commands provide complementary views:

```text
show ip igmp snooping groups
→ Which ports should receive each multicast group?

show ip igmp snooping membership
→ Which host/report caused that listener membership?
```

## A Listener Port Does Not Necessarily Mean One Host

A listener port means:

> At least one receiver for the group is reachable through this Layer 2 port.

It does not necessarily identify one individual receiver.

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

If either Rec1 or Rec2 joins `239.1.1.1`, SW1 may maintain:

```text
239.1.1.1
→ Ethernet0/1
```

If both hosts join, the forwarding entry on SW1 still contains the same outgoing Layer 2 port:

```text
239.1.1.1
→ Ethernet0/1
```

This becomes important when one receiver leaves.

SW1 cannot automatically assume that Ethernet0/1 should be removed, because another receiver may still exist behind that port.

## Listener-Port State Is Per VLAN and Group

IGMP snooping membership is associated with:

```text
VLAN
+
multicast group
+
Layer 2 port
```

For example:

```text
VLAN 10
239.1.1.1
→ E0/1

VLAN 10
239.2.2.2
→ E0/2

VLAN 20
239.1.1.1
→ E0/3
```

The same physical port can therefore be a listener port for many different groups.

Likewise, the same multicast group can have different listener ports in different VLANs.

## IGMPv1 and IGMPv2 Listener Learning

With IGMPv1 or IGMPv2, the listener-port relationship is group-based.

For example:

```text
IGMPv2 Membership Report
Group = 239.1.1.1
Ingress = E0/1
```

causes the switch to learn:

```text
239.1.1.1
→ E0/1
```

The switch does not need to know which application on the host joined the group.

It only needs to know that multicast traffic for the group must continue to be forwarded through that port.

## IGMPv3 Listener Learning

IGMPv3 Reports can contain source-filter information such as:

```text
MODE_IS_INCLUDE
Group: 239.1.1.1
Sources:
  10.1.1.1
```

At the IGMP protocol level, this means the receiver wants:

```text
(10.1.1.1,239.1.1.1)
```

However, Catalyst 9000 switches support only **Basic IGMPv3 Snooping Support (BISS)**.

Their snooping forwarding state is based on the **destination multicast group address**, not the source IP address. Cisco explicitly states that Catalyst 9000 IGMPv3 snooping does not support source-IP-based snooping.

Therefore:

```text
IGMPv3 Report:
INCLUDE {10.1.1.1}
Group 239.1.1.1
received on E0/1
```

results conceptually in:

```text
239.1.1.1
→ E0/1
```

not:

```text
(10.1.1.1,239.1.1.1)
→ E0/1
```

At Layer 2, different sources sending to the same group are not distinguished by Cat 9K IGMP snooping.

### Source Information in the Snooping Database

Although Catalyst 9000 does not perform **source-based snooping forwarding**, it can still parse and track the source information contained in IGMPv3 Membership Reports.

For example, suppose host `10.3.4.10` on Ethernet0/2 reports:

```text
INCLUDE {10.1.2.10}
Group: 239.1.1.1
```

The snooping group table shows both the group and the reported source:

```text
SW3# show ip igmp snooping groups
Flags: I -- IGMP snooping, S -- Static, P -- PIM snooping, A -- ASM mode
       E -- EVPN sync

Vlan      Group/source             Type        Version     Port List
-----------------------------------------------------------------------
10        224.0.1.40               I           v3          Et0/0
10        239.1.1.1                I           v3          Et0/2
           /10.1.2.10              I                       Et0/2
```

The indented source entry:

```text
239.1.1.1
 /10.1.2.10
```

shows that the switch has learned source-specific membership information for:

```text
(10.1.2.10,239.1.1.1)
```

The more detailed membership database shows the same information:

```text
SW3# show ip igmp snooping membership
Snooping Membership Summary for Vlan 10
------------------------------------------
Total number of channels: 2
Total number of hosts   : 2

Source/Group                    Interface Reporter                        Uptime   Last-Join/
                                                                                  Last-Leave
-----------------------------------------------------------------------------------------------

0.0.0.0/224.0.1.40              Et0/0     10.3.4.3                        00:00:54 00:00:54 /
                                                                                      -

10.1.2.10/239.1.1.1             Et0/2     10.3.4.10                       00:00:54 00:00:54 /
                                                                                      -
```

For the second entry:

```text
Source:    10.1.2.10
Group:     239.1.1.1
Interface: Ethernet0/2
Reporter:  10.3.4.10
```

the switch therefore knows that host `10.3.4.10` reported interest in the `(10.1.2.10,239.1.1.1)` channel through Ethernet0/2.

However, this does **not** mean Catalyst 9000 performs `(S,G)` snooping-based forwarding.

With Basic IGMPv3 Snooping Support (BISS), the source information can be retained in the snooping membership database, but Layer 2 multicast forwarding is still constrained based on the **destination multicast group**.

Conceptually:

```text
Snooping membership database:

(S1,G) → E0/2
```

but the forwarding decision remains effectively:

```text
G → E0/2
```

Therefore, if both:

```text
(S1,G)
(S2,G)
```

arrive at the switch, the snooping state does not use the source IP address to forward `S1` to Ethernet0/2 while filtering `S2` from Ethernet0/2.

The distinction is:

```text
IGMPv3 source information
→ parsed and tracked in the snooping membership database

Layer 2 snooping forwarding
→ based on destination group G
→ not filtered based on source S
```

This is what Cisco means when it states that Catalyst 9000 supports IGMPv3 snooping based only on the **destination multicast IP address**, not the source IP address.

## Source Filtering Still Matters to the Router

Catalyst 9000 can parse and retain the **source-specific membership information** contained in IGMPv3 Reports.

For example, the snooping membership database can record:

```text
(S1,G) → Ethernet0/2
```

showing that a receiver on Ethernet0/2 reported interest in source `S1` for group `G`.

However, with Catalyst 9000 **Basic IGMPv3 Snooping Support (BISS)**, this source information is **not used for Layer 2 multicast forwarding**.

The snooping forwarding decision remains effectively:

```text
G → listener ports
```

rather than:

```text
(S,G) → listener ports
```

The multicast router, however, receives the full IGMPv3 Report and can use:

```text
INCLUDE
EXCLUDE
source lists
```

to maintain source-specific receiver state and make Layer 3 multicast routing decisions.

Therefore, the switch and router can both track source information, but use it differently:

```text
Layer 2 switch:
tracks IGMPv3 source membership information
but forwards based on G

Multicast router:
tracks source/filter state
and can make (S,G)-specific routing decisions
```

The key distinction is:

```text
Catalyst 9000 IGMPv3 snooping database
→ can contain source-specific membership information

Catalyst 9000 snooping forwarding
→ based on destination multicast group G

Multicast routing
→ can use full (S,G) receiver state
```

## Refreshing Listener-Port State

Dynamic listener-port entries are not permanent.

The multicast router periodically sends IGMP General Queries:

```text
General Query
      ↓
receivers respond
      ↓
snooping switch sees Reports
      ↓
listener state refreshed
```

For example:

```text
239.1.1.1
→ Ethernet0/2
```

remains active while the switch continues to observe membership information indicating that a receiver is reachable through Ethernet0/1.

Cisco notes that snooping-learned entries are periodically removed if Membership Reports are no longer received.

This allows stale state to disappear if a host:

```text
powers off
disconnects
fails
leaves without sending a usable Leave
```

## Listener State Depends on a Querier

Dynamic membership refresh normally depends on periodic IGMP Queries.

Without a querier:

```text
No periodic Queries
        ↓
No periodic membership refresh
        ↓
dynamic snooping state may eventually age out
```

Therefore, a VLAN using IGMP snooping normally needs either:

```text
a multicast router acting as IGMP querier
```

or:

```text
an IGMP snooping querier
```

The snooping querier is covered in **1.6.a (iii) IGMP Querier**.

## Leaving a Group

Listener-port removal is more complicated than listener-port learning.

Suppose:

```text
239.1.1.1
→ Ethernet0/1
```

and a host behind Ethernet0/1 sends a Leave.

The switch cannot always immediately conclude:

```text
remove Ethernet0/1
```

because multiple receivers may exist behind the port.

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

If Rec1 leaves:

```text
Rec1 → no longer wants G
Rec2 → may still want G
```

Therefore, normal leave processing may involve:

```text
Leave / state change
        ↓
Group-Specific or Group-and-Source-Specific Query
        ↓
wait for remaining receivers
        ↓
remove listener port only if no membership remains
```

Cisco IOS XE exposes snooping last-member query timers and counts for this process.

The details are covered in **IGMP Snooping Immediate Leave** and **IGMP Snooping Explicit Tracking**.

## IGMPv1 Listener Removal

IGMPv1 has no Leave Group message.

A v1 host simply stops sending Reports.

Therefore:

```text
listener leaves
      ↓
no explicit Leave
      ↓
membership stops being refreshed
      ↓
listener-port entry eventually ages out
```

This is inherently slower than explicit leave processing.

## IGMPv2 Listener Removal

IGMPv2 introduces the Leave Group message:

```text
Type 0x17
Destination 224.0.0.2
```

When the switch sees a Leave for a dynamically learned group, it must determine whether another receiver remains behind that port.

Under normal processing, the membership is not necessarily removed immediately.

The last-member query process allows remaining receivers to refresh the state before the port is deleted.

Immediate removal is possible with **IGMP Snooping Immediate Leave**, but should be used only where one receiver is expected behind the port.

## IGMPv3 Listener Removal

IGMPv3 does not use the separate IGMPv2 Leave Group message for native state changes.

Instead, the host sends State-Change Records such as:

```text
BLOCK_OLD_SOURCES
CHANGE_TO_INCLUDE_MODE
```

The snooping switch examines these IGMPv3 Reports and updates its group-level listener state.

On Catalyst 9000, because BISS forwarding is destination-group-based, source-specific changes do not create independent `(S,G)` Layer 2 listener entries.

The important Layer 2 question remains:

```text
Does this port still need traffic for G?
```

## Static Listener Ports

A port can also be configured as a static member of a multicast group.

For VLAN 10:

```text
SW1(config)# ip igmp snooping vlan 10 static 239.1.1.1 interface Ethernet0/1
```

This tells the switch:

```text
Always treat Ethernet0/1 as a listener port for 239.1.1.1 in VLAN 10
```

The switch therefore forwards traffic for the group through Ethernet0/1 without requiring dynamic IGMP membership signaling from a host on that port. Cisco documents static snooping membership as overriding dynamic manipulation for that configured port/group membership.

Remove it with:

```text
SW1(config)# no ip igmp snooping vlan 10 static 239.1.1.1 interface Ethernet0/1
```

## Static vs Dynamic Listener State

Dynamic listener state is learned from IGMP:

```text
Membership Report
      ↓
dynamic listener port
```

Static listener state is administratively configured:

```text
ip igmp snooping vlan ... static ...
      ↓
static listener port
```

Both can exist in the multicast forwarding table.

Static membership is useful when:

```text
a receiver does not send IGMP
deterministic forwarding is required
testing/lab behavior is needed
```

## Static Listener Port vs Mrouter Port

Do not confuse:

```text
ip igmp snooping vlan 10 static 239.1.1.1 interface Ethernet0/1
```

with:

```text
ip igmp snooping vlan 10 mrouter interface Ethernet0/1
```

They mean different things:

```text
static group membership
→ receiver for G is reachable through this port

static mrouter port
→ multicast router is reachable through this port
```

A single physical interface could potentially have different roles depending on the topology, but the forwarding semantics are distinct.

## Port or VLAN Changes

Dynamic snooping membership depends on the Layer 2 topology.

Cisco documents that snooping-learned multicast groups associated with a port are deleted when certain Layer 2 attributes change, including:

```text
Spanning Tree state
port group
VLAN ID
```

because the previously learned receiver reachability may no longer be valid.

The switch can then relearn listener state from subsequent IGMP signaling.

## Listener Ports and TCN Flood Mode

After an STP topology change, existing snooping state may temporarily be incomplete or stale.

During TCN flood mode, multicast may therefore be flooded even to ports that are not currently listed as listener ports.

This is deliberate:

```text
topology changed
      ↓
listener state may be stale
      ↓
temporarily flood
      ↓
Queries / Reports relearn membership
      ↓
return to listener-based forwarding
```

So a listener-port entry is normally what restricts multicast forwarding, but temporary topology-change behavior can override that selectivity.

## Listener-Port Summary

```text
IGMP Membership Report received on port
        ↓
Switch learns:
(VLAN, G) → port
        ↓
port becomes listener/member port
        ↓
known multicast for G is forwarded there
```

Dynamic listener state is refreshed by IGMP signaling and eventually removed when receiver interest disappears.

The key points are:

```text
Listener port
→ at least one receiver for G is reachable through the port

One port can represent multiple receivers
→ a single Leave does not always mean the port can be removed

Catalyst 9000 IGMPv3 snooping
→ can track source-specific membership information
→ forwarding decisions are still based on destination group G
→ source S is not used for Layer 2 snooping forwarding

Static group membership
→ permanently adds a listener port for that group until removed
```

The next section examines the complementary port type: **Multicast Router Ports**.