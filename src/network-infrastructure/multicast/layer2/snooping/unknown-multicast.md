# Unknown Multicast Forwarding

IGMP snooping restricts multicast forwarding only when the switch has learned where receivers for a multicast group exist.

If a multicast data packet arrives for a group that has **no matching IGMP snooping entry**, the traffic is **unknown multicast**.

For example:

```text
SW1# show ip igmp snooping groups vlan 10

Vlan      Group/source             Type        Version     Port List
-----------------------------------------------------------------------
10        239.1.1.1                I           v2          Et0/1
```

The switch knows how to forward:

```text
239.1.1.1
```

but if it receives traffic for:

```text
239.2.2.2
```

and no snooping entry exists for that group, `239.2.2.2` is unknown multicast.

## Known vs Unknown Multicast

With a known multicast group:

```text
239.1.1.1
→ listener ports
→ mrouter ports
```

the switch can selectively replicate the traffic.

For example:

```text
              SW1
        /      |      \
      E0/1    E0/2    E0/3
       |       |       |
     Rec1     PC2      R1
```

If the snooping state is:

```text
239.1.1.1
→ listener: E0/1
→ mrouter:  E0/3
```

traffic for `239.1.1.1` is forwarded to:

```text
E0/1
E0/3
```

but not E0/2.

If traffic instead arrives for an unknown group:

```text
239.2.2.2
```

the switch has no listener-port information for that group.

## Unknown Multicast Behavior

Unknown multicast traffic is **flooded through the VLAN**.

Conceptually:

```text
              SW1
        /      |      \
      E0/1    E0/2    E0/3
       |       |       |
     Rec1     PC2      R1
```

Suppose a source connected to E0/4 sends:

```text
239.2.2.2
```

but SW1 has no snooping entry for `239.2.2.2`.

The traffic is flooded:

```text
239.2.2.2
→ E0/1
→ E0/2
→ E0/3
```

and any other eligible forwarding ports in the VLAN, except the ingress port.

This resembles the behavior of a switch without IGMP snooping for that particular group.

## Why Flood Unknown Multicast?

The absence of a snooping entry does **not necessarily mean that no receiver exists**.

It only means:

```text
The switch currently has no snooping state for G.
```

For example:

```text
multicast source starts sending
        ↓
traffic for G reaches SW1
        ↓
SW1 has not yet learned any listeners for G
```

If SW1 simply dropped the traffic:

```text
no snooping entry
→ drop G
```

a legitimate receiver could be temporarily blackholed.

Flooding provides safe connectivity until the switch learns more precise membership information.

## Learning a Listener Stops the Flood

Suppose the switch initially receives unknown traffic for:

```text
239.2.2.2
```

and floods it throughout VLAN 10.

Then Rec1 sends an IGMP Membership Report:

```text
Rec1
 |
 | Report for 239.2.2.2
 v
E0/1
 SW1
```

SW1 learns:

```text
239.2.2.2
→ E0/1
```

The group is no longer unknown.

Forwarding can now become selective:

```text
Before membership learned:

239.2.2.2
→ flood VLAN


After membership learned:

239.2.2.2
→ listener ports
→ mrouter ports
```

So IGMP snooping does not inherently prevent all multicast flooding.

It changes traffic from:

```text
unknown
→ flood
```

to:

```text
known
→ selectively forward
```

once appropriate snooping state exists.

## An Entry Can Become Unknown Again

Dynamic IGMP snooping entries eventually age out when membership is no longer refreshed.

For example:

```text
239.1.1.1
→ E0/1
```

may disappear after the receiver stops reporting membership.

If multicast traffic for `239.1.1.1` subsequently arrives while no snooping entry exists:

```text
G no longer in snooping table
        ↓
traffic for G becomes unknown multicast
        ↓
traffic can be flooded again
```

Therefore, proper IGMP Query/Report operation is important not only for learning memberships but also for maintaining selective forwarding.

## No Querier Can Cause Unexpected Flooding

Dynamic snooping state normally depends on periodic IGMP Queries and Reports.

If a VLAN has no functioning IGMP querier:

```text
no periodic Queries
        ↓
membership Reports not refreshed
        ↓
dynamic snooping entries age out
        ↓
groups may become unknown
        ↓
multicast flooding can occur
```

This is one reason an IGMP snooping VLAN normally needs either:

```text
a multicast router acting as IGMP querier
```

or:

```text
an IGMP snooping querier
```

The snooping querier is covered separately in **1.6.a (iii) IGMP Querier**.

## Static Membership Can Make a Group Known

A static snooping membership can provide forwarding state even if no host dynamically sends IGMP Reports.

For example:

```text
SW1(config)# ip igmp snooping vlan 10 static 239.2.2.2 interface Ethernet0/1
```

creates static listener state:

```text
239.2.2.2
→ Ethernet0/1
```

The switch can therefore selectively forward that group rather than relying on dynamically learned listener membership.

Static snooping membership was covered in **Multicast Listener Ports**.

## Unknown Multicast and Mrouter Ports

For a **known** multicast group, basic IGMP snooping normally forwards data toward:

```text
listener ports
+
mrouter ports
```

For an **unknown** group, the switch does not yet have a group-specific listener-port entry to use.

With default flooding behavior, traffic is flooded through the VLAN, which naturally includes eligible mrouter ports as well as host-facing ports.

So compare:

```text
Known G:

G
→ known listener ports
→ mrouter ports
```

with:

```text
Unknown G:

G
→ flooded across VLAN
```

The second case is less selective because the switch has no membership state for that group.

## Catalyst 9000 Uses the Multicast IP Group

On Catalyst 9000, IGMP snooping forwarding is based on the **destination multicast IP address**, not merely the destination multicast MAC address.

Therefore, whether IP multicast is known or unknown is conceptually based on whether the switch has forwarding state for:

```text
G
```

For example:

```text
239.1.1.1
```

rather than simply whether:

```text
01:00:5e:xx:xx:xx
```

exists in the ordinary MAC address table.

This also avoids ambiguity caused by IPv4 multicast MAC-address mapping, where multiple IP multicast groups can map to the same Ethernet multicast MAC address.

## Do Not Confuse Unknown IP Multicast with `switchport block multicast`

Catalyst IOS XE provides the interface command:

```text
SW1(config-if)# switchport block multicast
```

Despite its name, this is **not an IGMP snooping command for blocking unknown IPv4 multicast groups**.

Cisco documents this feature as blocking unknown **pure Layer 2 multicast** traffic.

It specifically does **not** block multicast packets containing IPv4 or IPv6 information.

Therefore:

```text
switchport block multicast
```

should not be interpreted as:

```text
"Drop unknown IGMP-snooped IP multicast groups."
```

The two concepts are different:

```text
IGMP snooping unknown multicast
→ IP multicast group has no snooping forwarding entry

switchport block multicast
→ generic Layer 2 port-blocking feature
→ does not block IPv4/IPv6 multicast traffic
```

## Unknown Multicast vs Unknown Unicast

The concepts are similar but use different forwarding databases.

### Unknown unicast

```text
destination unicast MAC
not in MAC address table
        ↓
flood VLAN
```

### Unknown multicast with IGMP snooping

```text
destination multicast group G
not in snooping forwarding state
        ↓
flood VLAN
```

Once state exists:

```text
Known unicast
→ forward to learned destination port

Known multicast
→ replicate to listener/mrouter ports
```

## Unknown Multicast vs TCN Flooding

Unknown multicast flooding should also not be confused with **TCN flood mode**.

Unknown multicast flooding occurs because:

```text
no snooping entry exists for G
```

TCN flooding occurs because:

```text
Layer 2 topology changed
        ↓
existing snooping state may be stale
        ↓
temporarily flood multicast
while membership is relearned
```

So even a previously known multicast group may temporarily be flooded during TCN recovery.

## Unknown Multicast vs Reserved Groups

Some multicast groups receive special treatment and should not be interpreted simply as ordinary unknown multicast.

In particular:

```text
224.0.0.0/24
```

contains IPv4 Local Network Control Block multicast addresses used by protocols such as:

```text
OSPF
PIM
IGMP
```

Switches may flood or otherwise specially handle these groups regardless of normal listener membership.

These are covered in **Reserved Multicast Groups**.

## Verification

Use:

```text
SW3# show ip igmp snooping groups 
```

to inspect the multicast groups for which the switch has snooping membership state.

For example:

```text
Vlan      Group/source             Type        Version     Port List
-----------------------------------------------------------------------
10        224.0.1.40               I           v3          Et0/0 
10        239.1.1.1                I           v3          Et0/1 
```

If traffic arrives for:

```text
239.2.2.2
```

but there is no corresponding entry:

```text
239.2.2.2
→ no snooping forwarding state
```

the group is unknown from the perspective of the snooping table.

A packet capture on other VLAN ports can then demonstrate the resulting flooding behavior.

## Key Points

```text
Unknown multicast
→ no matching IGMP snooping forwarding entry for G
→ flooded through the VLAN by default

Known multicast
→ forwarded to listener ports
→ and appropriate mrouter ports

Why flood?
→ absence of snooping state does not prove
  that no legitimate receiver exists

IGMP Report received
→ listener state created
→ G becomes known
→ forwarding becomes selective

Snooping state ages out
→ G can become unknown again
→ flooding can resume

No IGMP querier
→ memberships may not refresh
→ snooping entries may age out
→ unexpected flooding can result

Catalyst 9000
→ IGMP snooping forwarding is based on multicast IP group G

switchport block multicast
→ does NOT block IPv4/IPv6 unknown multicast
→ separate Layer 2 port-blocking feature

TCN flooding
→ separate mechanism used after topology changes

224.0.0.0/24
→ special/reserved multicast behavior
→ covered separately
```