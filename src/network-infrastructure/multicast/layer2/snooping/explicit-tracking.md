# IGMP Snooping Explicit Tracking

Normal IGMP snooping primarily needs to know:

```text
Which ports need multicast group G?
```

**IGMP Snooping Explicit Tracking**, also called **Explicit Host Tracking (EHT)**, goes further by tracking the individual hosts that reported membership in multicast groups or channels.

Conceptually:

```text
Normal snooping:
G → E0/1

Explicit Tracking:
Host A → G → E0/1
Host B → G → E0/1
```

This gives the switch more precise receiver state and can significantly reduce multicast leave latency.

Cisco identifies the main benefits as:

```text
minimal leave latency
faster channel changes
better diagnostics
```

Like router-side EHT, IGMP Snooping EHT is an **IGMPv3** feature.

## The Problem Without Explicit Tracking

Consider two receivers behind the same switch port:

```text
                 SW3
                  |
                E0/1
                  |
                 SW4
                /   \
             Rec1   Rec2
```

Both receivers are listening to:

```text
239.1.1.1
```

Ordinary snooping forwarding state only needs to know:

```text
239.1.1.1
→ E0/1
```

Now suppose Rec1 leaves.

SW3 cannot remove E0/1 immediately because Rec2 might still need the group.

Without reliable per-host membership information, the switch must verify whether another receiver remains:

```text
Rec1 leaves
    ↓
send Group-Specific Query
    ↓
wait for remaining receivers
    ↓
Rec2 responds
    ↓
keep E0/1
```

This creates **leave latency** if there are no other hosts connected to the port.

## With Explicit Tracking

Explicit Tracking allows the switch to maintain information about the individual receivers.

For example:

```text
239.1.1.1
├── 10.3.4.10 → E0/1
└── 10.3.4.11 → E0/1
```

Now if `10.3.4.10` leaves:

```text
239.1.1.1
└── 10.3.4.11 → E0/1
```

SW1 knows another receiver still exists behind Ethernet0/1, so the port must remain in the forwarding state.

If the final receiver leaves:

```text
10.3.4.11 leaves
       ↓
no tracked receivers remain for G on E0/1
       ↓
E0/1 can be removed immediately
```

With IGMPv3 and Explicit Tracking, Cisco states that the device can stop forwarding immediately when the **last tracked host** indicates that it no longer wants the multicast traffic.

## Explicit Tracking vs Immediate Leave

Both features can reduce leave latency, but they work very differently.

### Immediate Leave

Immediate Leave makes an assumption:

```text
one receiver per port
```

Therefore:

```text
Leave received on E0/1
→ immediately remove E0/1
```

It does not verify whether another receiver remains.

### Explicit Tracking

Explicit Tracking maintains actual host state:

```text
E0/1:
Host A → G
Host B → G
```

If Host A leaves:

```text
Host B still tracked
→ keep E0/1
```

If Host B also leaves:

```text
no hosts remain
→ remove E0/1
```

So:

```text
Immediate Leave
→ relies on topology assumption

Explicit Tracking
→ relies on per-host membership state
```

Explicit Tracking is therefore suitable for situations where **multiple receivers may exist behind the same port**
but you still want to reduce leave latency.

## Why Explicit Tracking Relies on IGMPv3

IGMPv3 allows the switch to maintain reliable **per-host membership state** because IGMPv3 hosts do **not** suppress their Reports after hearing Reports from other hosts.

Each host reports its own membership and source-filter state.

For example:

```text
Rec1:
Reporter 10.1.1.10
INCLUDE {10.10.10.1}
Group 239.1.1.1

Rec2:
Reporter 10.1.1.11
INCLUDE {10.10.10.1}
Group 239.1.1.1
```

The switch can therefore track:

```text
Host 10.1.1.10
→ (10.10.10.1,239.1.1.1)

Host 10.1.1.11
→ (10.10.10.1,239.1.1.1)
```

rather than only:

```text
239.1.1.1
→ E0/1
```

This lets the switch know exactly which tracked receivers remain when one host changes or leaves its membership.

### Why Not Reliably Track IGMPv1/v2 Hosts?

IGMPv1 and IGMPv2 do not provide reliable end-to-end per-host visibility.

Even though a snooping switch may receive individual Reports from directly connected hosts, another snooping switch downstream can perform **IGMP Snooping Report Suppression** before those Reports reach the upstream switch.

For example:

```text
                 SW3
                  |
                E0/1
                  |
                 SW4
                /   \
             Rec1   Rec2
```

Suppose both Rec1 and Rec2 are IGMPv2 receivers for the same group:

```text
239.1.1.1
```

Because SW4 is performing IGMP snooping, it can receive a Report from each receiver:

```text
Rec1 → Report for G → SW4
Rec2 → Report for G → SW4
```

SW4 can therefore learn both listener ports locally.

However, with **IGMP Snooping Report Suppression** enabled, SW4 forwards only the first IGMPv2 Report for the group toward its mrouter/uplink port and suppresses the duplicate Report.

For example:

```text
Rec1 → Report for G ──┐
                      │
                      v
                     SW4 ───── Report for G ─────> SW3

Rec2 → Report for G ──┘
                      X
               duplicate Report
               suppressed upstream
```

SW4 may know about both receivers:

```text
SW4:

G
→ Rec1's port
→ Rec2's port
```

but SW3 may see only:

```text
one IGMPv2 Report for G
```

SW3 therefore cannot infer:

```text
"There is exactly one receiver behind E0/1."
```

There may actually be several receivers hidden behind SW4.

This is a fundamental limitation for reliable per-host tracking with IGMPv1/v2:

```text
Rec1 ──┐
       ├── SW4 ── one Report ──> SW3
Rec2 ──┘
```

The upstream switch sees group membership through E0/1, but it cannot reliably determine how many individual receivers exist behind that port.

IGMPv3 removes this ambiguity because **IGMPv3 Reports are not subject to IGMP Snooping Report Suppression**.

Therefore:

```text
Rec1 → IGMPv3 Report → SW4 → SW3
Rec2 → IGMPv3 Report → SW4 → SW3
```

SW3 can receive the individual membership information from both hosts and maintain separate tracked memberships.

Therefore:

```text
IGMPv1/v2
→ downstream snooping switches may suppress duplicate Reports
→ upstream devices may not see every receiver
→ reliable end-to-end per-host Explicit Tracking is not possible

IGMPv3
→ Reports are not snooping-suppressed
→ each host's membership can reach the upstream device
→ reliable per-host Explicit Tracking is possible
```

This is why IGMP Snooping Explicit Tracking relies on **IGMPv3 membership reporting**.

## Groups and Channels

Explicit Tracking can track both:

```text
group membership
(*,G)
```

and IGMPv3 source-specific:

```text
channel membership
(S,G)
```

For example:

```text
Host 10.3.4.10

Source: 10.1.2.10
Group:  239.1.1.1
Port:   E0/1
```

can be represented in the tracking database as:

```text
10.1.2.10/239.1.1.1
→ Reporter 10.3.4.10
→ E0/1
```

Cisco describes Explicit Tracking as tracking individual hosts joined to a particular **group or channel**.

## Tracking Source State Does Not Mean Source-Based Forwarding

Explicit Tracking can record source-specific IGMPv3 membership information such as:

```text
(S1,G)
Reporter: Host A
Port: E0/1
```

However, Catalyst 9000 still supports only **Basic IGMPv3 Snooping Support (BISS)** for Layer 2 forwarding.

Cisco states that Catalyst 9000 snooping forwarding is based only on the **destination multicast group address**, not the multicast source address.

Therefore:

```text
Explicit Tracking database:
(S1,G) → Host A → E0/1
```

does **not** mean the Layer 2 forwarding decision becomes:

```text
(S1,G) → E0/1
```

The forwarding decision remains effectively:

```text
G → E0/1
```

So:

```text
Explicit Tracking
→ can track S and G per receiver

BISS forwarding
→ still forwards based on G
→ does not filter based on S
```

## IGMP Version Compatibility

Suppose an IGMPv3 host joins a group but an IGMPv2 host is also present.

Compatibility rules may cause the IGMPv3 host to use IGMPv2-style reporting for that group.

Conceptually:

```text
IGMPv3 host
+
IGMPv2 compatibility required
        ↓
v2-style Report
        ↓
per-host IGMPv3 tracking information unavailable
```

Cisco notes that this can disable Explicit Tracking for those host memberships.

Therefore, the minimal-leave benefit of Explicit Tracking is fundamentally associated with **IGMPv3 membership reporting**.

## Faster Leave Processing

Without Explicit Tracking:

```text
last host appears to leave
        ↓
send specific Query
        ↓
wait through last-member query process
        ↓
remove forwarding state
```

With IGMPv3 Explicit Tracking:

```text
tracked host leaves
        ↓
check tracking database
        ↓
other tracked receivers remain?
        |
   +----+----+
   |         |
  Yes        No
   |         |
keep port   remove port
```

If no other tracked receiver remains, the switch does not need to wait through the normal query/response verification process.

This is the main reason Explicit Tracking provides **minimal leave latency**.

## Faster Channel Changes

Fast leave processing is particularly useful for applications such as IPTV.

Suppose a receiver changes:

```text
Channel A
→
Channel B
```

Without fast leave processing:

```text
old multicast stream continues
+
new multicast stream starts
```

for a short period.

On a bandwidth-constrained access link, this temporary overlap may be undesirable.

Explicit Tracking lets the switch remove the old membership quickly when it knows the last receiver has left:

```text
leave Channel A
→ immediately remove old state

join Channel B
→ forward new stream
```

Cisco specifically lists faster channel changing as one of the benefits of Explicit Tracking.

## Improved Diagnostics

Without Explicit Tracking, the snooping table might show:

```text
239.1.1.1
→ E0/1
```

This tells you that at least one receiver exists behind Ethernet0/1.

With Explicit Tracking, the switch can maintain information such as:

```text
Source/Group: 10.1.2.10/239.1.1.1
Interface:    E0/1
Reporter:     10.3.4.10
```

This makes troubleshooting easier because the switch can identify:

```text
which host joined
which group/channel it joined
which interface it is reachable through
when it joined or left
```

Cisco identifies improved diagnostic capability as another major benefit of Explicit Tracking.

## Configuration

IGMP Snooping EHT is enabled on each VLAN by default:

```
SW3#show ip igmp snooping vlan 10
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
IGMPv2 immediate leave              : Enabled
Explicit host tracking              : Enabled  <-- ENABLED BY DEFAULT
Report suppression                  : Enabled
Robustness variable                 : 2
Last member query count             : 2
Last member query interval          : 1000
Check TTL=1                         : Yes
Check Router-Alert-Option           : Yes
Query Interval                      : 60
Max Response Time                   : 10000
```

Disable it with:

```text
SW1(config)# no ip igmp snooping vlan 10 explicit-tracking
```

The feature is configured **per VLAN**, not per physical Layer 2 interface.

## Layer 2 vs Layer 3 Explicit Tracking

Do not confuse:

```text
ip igmp snooping vlan 10 explicit-tracking
```

with:

```text
interface Vlan10
 ip igmp explicit-tracking
```

They apply at different layers.

### IGMP Snooping Explicit Tracking

```text
SW1(config)# ip igmp snooping vlan 10 explicit-tracking
```

tracks receiver membership for **Layer 2 IGMP snooping**.

Its purpose is to improve:

```text
listener-port state
leave processing
host diagnostics
```

### Layer 3 IGMP Explicit Tracking

```text
SW1(config-if)# ip igmp explicit-tracking
```

tracks individual receivers in the router's **Layer 3 IGMP membership state**.

Cisco documents both forms separately.

For this section, the focus is:

```text
ip igmp snooping vlan <vlan> explicit-tracking
```

## Viewing Tracked Host Membership

Use the following command to verify EHT:

```
SW3#show ip igmp snooping membership 
Snooping Membership Summary for Vlan 10
------------------------------------------
Total number of channels: 2
Total number of hosts   : 3

Source/Group                    Interface Reporter                        Uptime   Last-Join/
                                                                                   Last-Leave
-----------------------------------------------------------------------------------------------

0.0.0.0/224.0.1.40              Et0/0     10.3.4.3                        00:12:00 00:12:09 /
                                                                                   00:00:08  

0.0.0.0/239.1.1.1               Et0/1     10.3.4.10                       00:12:06 00:12:13 /
                                                                                   00:00:07  

0.0.0.0/239.1.1.1               Et0/1     10.3.4.11                       00:12:02 00:12:10 /
                                                                                   00:00:07  
```

This shows two individually tracked receivers for the same multicast channel behind Ethernet0/1.

Cisco documents `show ip igmp snooping membership` specifically for displaying Explicit Host Tracking membership information.

## Filtering the Membership Display

The command supports filters such as:

```text
SW1# show ip igmp snooping membership vlan 10
SW1# show ip igmp snooping membership interface Ethernet0/2
SW1# show ip igmp snooping membership reporter 10.3.4.10
SW1# show ip igmp snooping membership source 10.1.2.10 group 239.1.1.1
```

This makes Explicit Tracking particularly useful when troubleshooting large multicast access networks.

## Explicit Tracking and the Snooping Group Table

Do not confuse the detailed host tracking database with the normal snooping forwarding table.

For example:

```text
SW3# show ip igmp snooping groups 
Flags: I -- IGMP snooping, S -- Static, P -- PIM snooping, A -- ASM mode
       E -- EVPN sync

Vlan      Group/source             Type        Version     Port List
-----------------------------------------------------------------------
10        224.0.1.40               I           v3          Et0/0 
10        239.1.1.1                I           v3          Et0/1 
```

This answers:

```text
Where should multicast traffic be forwarded?
```

Whereas the EHT membership database:

```text
SW3#show ip igmp snooping membership 
Snooping Membership Summary for Vlan 10
------------------------------------------
Total number of channels: 2
Total number of hosts   : 3

Source/Group                    Interface Reporter                        Uptime   Last-Join/
                                                                                   Last-Leave
-----------------------------------------------------------------------------------------------

0.0.0.0/224.0.1.40              Et0/0     10.3.4.3                        00:12:00 00:12:09 /
                                                                                   00:00:08  

0.0.0.0/239.1.1.1               Et0/1     10.3.4.10                       00:12:06 00:12:13 /
                                                                                   00:00:07  

0.0.0.0/239.1.1.1               Et0/1     10.3.4.11                       00:12:02 00:12:10 /
                                                                                   00:00:07  
```


answers:

```text
Which receiver(s) reported this membership?
```

## Explicit Tracking vs Normal Snooping

The difference can be summarized as:

| Feature | Normal IGMP Snooping | Explicit Tracking |
|---|---|---|
| Tracks listener ports | Yes | Yes |
| Tracks individual receivers | Limited | Yes |
| Tracks IGMPv3 channel membership | Can parse/store source state | Per-host/channel tracking |
| Layer 2 forwarding based on source S on Cat 9K | No | No |
| Minimal IGMPv3 leave latency | No | Yes |
| Improved host-level diagnostics | Limited | Yes |

Explicit Tracking adds **receiver identity and membership state**; it does not change Catalyst 9000 BISS into source-aware Layer 2 forwarding.

## Key Points

```text
IGMP Snooping Explicit Tracking
→ also called Explicit Host Tracking (EHT)

Normal snooping
→ primarily tracks which ports need G

Explicit Tracking
→ tracks individual hosts, groups, and channels

IGMPv3
→ provides the detailed per-host reporting needed for reliable Explicit Tracking

Last tracked host leaves
→ forwarding can stop immediately
→ no last-member verification delay required

Multiple hosts behind one port
→ safe, because the switch knows which hosts remain

Immediate Leave
→ assumes one receiver per port

Explicit Tracking
→ actually tracks the receivers

Catalyst 9000 BISS
→ source information can be tracked
→ Layer 2 forwarding is still based on G, not S

Configuration:
ip igmp snooping vlan <vlan> explicit-tracking

Verification:
show ip igmp snooping vlan <vlan>
show ip igmp snooping membership
```