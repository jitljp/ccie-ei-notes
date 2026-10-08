# IGMP Snooping Immediate Leave

By default, an IGMP snooping switch does **not** immediately remove a listener port when it receives an IGMPv2 Leave.

Instead, it first checks whether another receiver for the group still exists behind that port.

**IGMP Snooping Immediate Leave** skips that verification process and removes the port from the multicast forwarding entry immediately.

```text id="5a8crp"
Normal leave processing
→ verify whether another receiver remains
→ then remove port if necessary

Immediate Leave
→ remove port immediately
```

## Normal Leave Processing

Consider:

```text id="yvlg04"
                 R1
                  |
                 SW1
               /     \
            E0/1     E0/2
              |        |
            Rec1      Rec2
```

Both receivers have joined:

```text id="1nu6i6"
239.1.1.1
```

SW1 therefore has:

```text id="p8p8li"
239.1.1.1
→ E0/1
→ E0/2
```

Suppose Rec1 sends an IGMPv2 Leave:

```text id="cg9v5t"
Rec1
 |
 | Leave 239.1.1.1
 v
E0/1
 SW1
```

Without Immediate Leave, SW1 does not simply remove Ethernet0/1.

It first performs **last-member query processing**, similar to a multicat router.

Conceptually:

```text id="9tczcs"
Leave received on E0/1
        ↓
send Group-Specific Query for G out E0/1
        ↓
wait for any remaining receiver
        ↓
no Report received
        ↓
remove E0/1 from G
```

Cisco IOS XE uses the snooping last-member query parameters to control this process.

By default:

```text id="eyj5j0"
Last Member Query Count    = 2
Last Member Query Interval = 1000 ms
```

So the switch can send multiple Group-Specific Queries before deciding that the port no longer needs the group.

## Why the Switch Normally Verifies

A switch port does not necessarily lead to only one host.

For example:

```text id="y75eor"
                 SW1
                  |
                E0/1
                  |
                 SW2
                /   \
             Rec1   Rec2
```

Both receivers may be interested in:

```text id="htxsmy"
239.1.1.1
```

From SW1's perspective:

```text id="6rb1qe"
239.1.1.1
→ E0/1
```

If Rec1 sends a Leave, that does **not** necessarily mean Ethernet0/1 should be removed.

Rec2 might still want the traffic.

Therefore, normal processing asks:

```text id="cwzkpw"
"Is anyone else behind E0/1 still interested in G?"
```

The Group-Specific Query provides those remaining receivers an opportunity to respond.

## Immediate Leave

With Immediate Leave enabled, that verification step is skipped.

When SW1 receives an IGMPv2 Leave on Ethernet0/1:

```text id="qfsrsm"
Leave for 239.1.1.1 received on E0/1
        ↓
immediately remove E0/1 from the snooping entry
```

No Group-Specific Query is sent to determine whether another receiver remains behind the port.

For example:

```text id="s68sem"
Before:

239.1.1.1
→ E0/1
→ E0/2
```

Rec1 sends a Leave on E0/1:

```text id="awhpt7"
Rec1 → Leave 239.1.1.1
```

With Immediate Leave:

```text id="25s0k5"
Immediately:

239.1.1.1
→ E0/2
```

Ethernet0/1 is pruned immediately.

Cisco describes Immediate Leave as removing the interface from the multicast forwarding entry **without first sending Group-Specific Queries to that interface**.

## Why Immediate Leave Exists

Normal leave verification introduces some delay.

With the default values:

```text id="l7bjjb"
Last Member Query Count    = 2
Last Member Query Interval = 1000 ms
```

the switch can continue forwarding multicast traffic to the port while it verifies that no receiver remains.

That is safe, but it means unwanted traffic may continue briefly.

Immediate Leave eliminates this delay:

```text id="ukwq3m"
receiver leaves
      ↓
port immediately removed
      ↓
multicast traffic stops immediately
```

This can be useful in environments such as IPTV, where receivers frequently change multicast channels and unnecessary multicast traffic should stop as quickly as possible.

## The Single-Receiver Requirement

Immediate Leave should be enabled only when the switch can safely assume that **one receiver exists behind each listener port**.

For example:

```text id="cgcgfq"
        SW1
         |
       E0/1
         |
        Rec1
```

If Rec1 sends a Leave:

```text id="n6mjc6"
Rec1 leaves G
```

SW1 knows that no other receiver can still need `G` through Ethernet0/1.

Immediate removal is safe:

```text id="0fc2se"
G → E0/1
```

becomes:

```text id="fvm91t"
E0/1 removed
```

This is the intended use case.

Cisco specifically recommends Immediate Leave only on VLANs where a **single receiver is connected to each port**.

## Why Immediate Leave Can Be Dangerous

Consider:

```text id="6czl1q"
                 SW1
                  |
                E0/1
                  |
                 SW2
                /   \
             Rec1   Rec2
```

Both Rec1 and Rec2 want:

```text id="n4ibyu"
239.1.1.1
```

SW1 has:

```text id="77vbdz"
239.1.1.1
→ E0/1
```

Now Rec1 sends a Leave.

With normal processing:

```text id="428xwa"
Rec1 sends Leave
      ↓
SW1 sends Group-Specific Query
      ↓
Rec2 responds
      ↓
E0/1 remains in forwarding entry
```

With Immediate Leave:

```text id="ruttem"
Rec1 sends Leave
      ↓
SW1 immediately removes E0/1
      ↓
Rec2 stops receiving multicast
```

Rec2 is still interested, but SW1 has no opportunity to discover that before pruning the port.

Therefore:

> Do not enable Immediate Leave where multiple receivers may exist behind the same Layer 2 port.

## Configuration Scope

On Catalyst 9000, IGMP Snooping Immediate Leave is configured **per VLAN**.

For example:

```text id="ur0gch"
SW1(config)# ip igmp snooping vlan 10 immediate-leave
```

This enables Immediate Leave for listener ports in VLAN 10.

Disable it with:

```text id="qkcjz2"
SW1(config)# no ip igmp snooping vlan 10 immediate-leave
```

Because the feature is enabled for the VLAN rather than for an individual access port, the design assumption is important:

```
Every listener-facing port in the VLAN should effectively have only one receiver
```

## Interaction with Last-Member Query Timers

The normal leave process is controlled by parameters such as:

```text id="g47slh"
ip igmp snooping last-member-query-count
ip igmp snooping last-member-query-interval
```

For example:

```text id="3rx33c"
SW1(config)# ip igmp snooping vlan 10 last-member-query-count 3
SW1(config)# ip igmp snooping vlan 10 last-member-query-interval 500
```

Without Immediate Leave, those values determine how the switch verifies whether another receiver remains.

Conceptually:

```text id="da0lnm"
Leave
 ↓
Query #1
 ↓ 500 ms
Query #2
 ↓ 500 ms
Query #3
 ↓
no Report
 ↓
remove listener port
```

However, if Immediate Leave is enabled:

```text id="jc5q0t"
Leave
 ↓
remove listener port immediately
```

The last-member query process is bypassed for that leave.

Cisco explicitly states that when both Immediate Leave and last-member query processing are configured, **Immediate Leave takes precedence**.

## IGMPv1

IGMPv1 has no Leave Group message.

A host leaving a group simply stops sending Membership Reports.

Therefore, there is no explicit Leave for Immediate Leave to react to.

```text id="6f4gvb"
IGMPv1 host stops listening
        ↓
no Leave message
        ↓
membership eventually ages out
```

Immediate Leave therefore does not provide a leave optimization for IGMPv1.

## IGMPv2

IGMPv2 defines the explicit:

```text id="c670ci"
Leave Group
Type = 0x17
```

This is the message that triggers Catalyst IGMP Snooping Immediate Leave.

```text id="1im1mq"
IGMPv2 Leave received
        ↓
Immediate Leave enabled
        ↓
remove ingress listener port immediately
```

Catalyst 9000 documentation states that IGMP Snooping Immediate Leave is supported for **IGMPv2 hosts**.

## What About IGMPv3?

IGMPv3 does not use the IGMPv2 Leave Group message.

Instead, hosts communicate membership changes using IGMPv3 State-Change Reports such as:

```text id="m88ffp"
BLOCK_OLD_SOURCES
CHANGE_TO_INCLUDE_MODE
```

Catalyst 9000 documentation specifically states that its **IGMP Snooping Immediate Leave feature is supported only for IGMPv2 hosts**.

Therefore, do not assume that:

```text id="vw10wb"
ip igmp snooping vlan 10 immediate-leave
```

provides the same fast-leave behavior for IGMPv3 State-Change Reports.

IGMPv3 leave/state-change processing instead follows the normal snooping verification mechanisms.

## Immediate Leave vs Explicit Tracking

Immediate Leave solves the multiple-host ambiguity by making a topology assumption:

```text id="86tr7v"
one host per port
```

If that assumption is true:

```text id="th91ut"
Leave received
→ no other host could still need G
→ port can be removed immediately
```

A more sophisticated alternative is **IGMP Snooping Explicit Tracking**, where the switch tracks individual receivers.

Conceptually:

```text id="4oo7kv"
Immediate Leave:
"Only one host can exist here,
so a Leave means the port is done."

Explicit Tracking:
"I know exactly which hosts are behind this port,
so I can determine whether another host remains."
```

Explicit Tracking is covered in the next section.

## Immediate Leave vs Router Immediate Leave

Do not confuse **IGMP Snooping Immediate Leave** with the Layer 3 IGMP feature configured on a multicast router.

Multicast router:

```
R1(config)# ip igmp immediate-leave group-list <standard-acl>
or
R1(config-if)# ip igmp immediate-leave group-list <standard-acl>
```

This affects the router's **IGMP membership state**, immediately removing it upon receiving a Leave Group message
for any group included in the ACL.

Snooping Immediate Leave:

```text id="xp3r53"
SW1(config)# ip igmp snooping vlan 10 immediate-leave
```

This affects the switch's **Layer 2 listener-port forwarding state**.

Its question is:

```text id="11gmhu"
Should this switch port remain in G's forwarding entry?
```


These are separate mechanisms operating at different layers.

## Verification

Use:

```
SW1# show ip igmp snooping vlan 10
```

The output identifies whether Immediate Leave is enabled for the VLAN.

For example:

```
SW3# how ip igmp snooping vlan 10
Global IGMP Snooping configuration:
-------------------------------------------
IGMP snooping Oper State     : Enabled
IGMPv3 snooping              : Enabled
Report suppression           : Enabled
TCN solicit query            : Disabled
Robustness variable          : 2
Last member query count      : 2
Last member query interval   : 1000
Check TTL=1                  : No
Check Router-Alert-Option    : No

Vlan 10:
--------
IGMP snooping Admin State           : Enabled
IGMP snooping Oper State            : Enabled
IGMPv2 immediate leave              : Disabled  <-- DISABLED BY DEFAULT
Explicit host tracking              : Enabled
Report suppression                  : Enabled
Robustness variable                 : 2
Last member query count             : 2
Last member query interval          : 1000
Check TTL=1                         : Yes
Check Router-Alert-Option           : Yes
Query Interval                      : 60
Max Response Time                   : 10000
```

## Observing Immediate Leave

I used the following debugs to view the process:

```
SW3# debug ip igmp snooping group
SW3# debug ip igmp snooping 239.1.1.1
```

Without `immediate-leave`:

```
*Oct  8 21:23:02.332: IGMPSN: Received IGMPv2 message for group 239.1.1.1 received on Vlan 10, port Et0/2
*Oct  8 21:23:02.332: IGMPSN: group:  Leave for group 239.1.1.1 received on Vlan 10, port Et0/2, mvr group (No)
*Oct  8 21:23:02.332: IGMPSN: group: Skip client info adding - ip 10.3.4.10, port_id Et0/2, on vlan 10
*Oct  8 21:23:02.332: IGMPSN: group: Group exist - Leave for group 239.1.1.1 received on Vlan 10, port Et0/2, group state (1)
*Oct  8 21:23:02.332: IGMPSN: group: Created v2 leave port on port Et0/2, for group 239.1.1.1 on Vlan 10 
*Oct  8 21:23:02.332: IGMPSN: group: Sending Group-Specific Query for group 239.1.1.1 on Vlan 10 , port Et0/2
*Oct  8 21:23:03.333: IGMPSN: group: Sending Group-Specific Query for group 239.1.1.1 on Vlan 10 , port Et0/2
*Oct  8 21:23:04.333: IGMPSN: group: Deleting leave port on port Et0/2, for 239.1.1.1 on Vlan 10 
*Oct  8 21:23:04.333: IGMPSN: Clearing all hosts on port Et0/2 for group 239.1.1.1
*Oct  8 21:23:04.333: IGMPSN: Delete group 239.1.1.1 member port Et0/2, on Vlan 10
```

Note the two Group-Specific Queries:

```
*Oct  8 21:23:02.332: IGMPSN: group: Sending Group-Specific Query for group 239.1.1.1 on Vlan 10 , port Et0/2
*Oct  8 21:23:03.333: IGMPSN: group: Sending Group-Specific Query for group 239.1.1.1 on Vlan 10 , port Et0/2
```

And now with `immediate-leave`:

```
SW3(config)# ip igmp snooping vlan 10 immediate-leave
```

```
*Oct  8 21:25:22.741: IGMPSN: Received IGMPv2 message for group 239.1.1.1 received on Vlan 10, port Et0/2
*Oct  8 21:25:22.741: IGMPSN: group:  Leave for group 239.1.1.1 received on Vlan 10, port Et0/2, mvr group (No)
*Oct  8 21:25:22.741: IGMPSN: group: Skip client info adding - ip 10.3.4.10, port_id Et0/2, on vlan 10
*Oct  8 21:25:22.741: IGMPSN: group: Group exist - Leave for group 239.1.1.1 received on Vlan 10, port Et0/2, group state (0)
*Oct  8 21:25:22.741: IGMPSN: Clearing all hosts on port Et0/2 for group 239.1.1.1
*Oct  8 21:25:22.741: IGMPSN: Delete group 239.1.1.1 member port Et0/2, on Vlan 10
*Oct  8 21:25:22.741: IGMPSN: mgt: deleted port Et0/2 on gce 0100.5e01.0101, on Vlan 10 
*Oct  8 21:25:22.741: IGMPSN: group: Deleting port Et0/2 from group 239.1.1.1 on Vlan 10
*Oct  8 21:25:22.741: IGMPSN: group: Deleting group 239.1.1.1
*Oct  8 21:25:22.741: IGMPSN: mgt: try to delete group 239.1.1.1, on Vlan 10 
*Oct  8 21:25:22.741: IGMPSN: group: purge group 239.1.1.1 in vlan 10
*Oct  8 21:25:22.741: IGMPSN: mgt: deleting group 239.1.1.1, on Vlan 10 
*Oct  8 21:25:22.741: IGMPSN: group: Deleting gce 0100.5e01.0101
*Oct  8 21:25:22.741: IGMPSN: mgt: deleting gce 0100.5e01.0101, on Vlan 10 
*Oct  8 21:25:22.741: IGMPSN: group: Sending Leave for group 239.1.1.1, w/ src_addr 10.3.4.10 on Vlan 10
*Oct  8 21:25:22.741: IGMPSN: group: Fast_leave enabled - Leave for group 239.1.1.1 received on Vlan 10, port Et0/2
```

Note the lack of Group-Specific Queries. It also states:

```
*Oct  8 21:25:22.741: IGMPSN: group: Fast_leave enabled - Leave for group 239.1.1.1 received on Vlan 10, port Et0/2
```

## Key Points

```text id="6tm3bu"
Normal IGMP snooping leave processing
→ send Group-Specific Queries
→ verify whether another receiver remains
→ remove port only if no receiver responds

Immediate Leave
→ skip Group-Specific Queries
→ immediately remove the listener port

Catalyst configuration
→ ip igmp snooping vlan <vlan> immediate-leave

Safe use
→ one receiver per listener port

Unsafe use
→ multiple receivers may exist behind one port

IGMPv1
→ no explicit Leave message
→ Immediate Leave does not apply

IGMPv2
→ explicitly supported
→ Leave Group can trigger Immediate Leave

IGMPv3
→ Catalyst Immediate Leave is not supported
→ normal IGMPv3 state-change processing applies

Immediate Leave
→ affects Layer 2 snooping state
→ not the same as Layer 3 IGMP immediate-leave
```