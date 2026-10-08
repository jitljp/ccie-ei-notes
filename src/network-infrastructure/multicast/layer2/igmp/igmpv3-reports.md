# IGMPv3 Membership Reports

IGMPv3 uses **Membership Reports** to communicate a receiver's current multicast reception state and changes to that state.

The IGMPv3 Membership Report message type is:

```text
0x22
```

Unlike IGMPv1 and IGMPv2, an IGMPv3 Report can describe:

- Multiple multicast groups in a single Report
- The receiver's INCLUDE or EXCLUDE mode
- Source-specific receiver interest
- Changes to the receiver's group or source state

IGMPv3 Reports contain one or more **Group Records**, with each Group Record describing the state of one multicast group.

## Membership Report Format

IGMPv3 Membership Reports use the following format:

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Type = 0x22  |    Reserved   |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             Flags             |  Number of Group Records (M)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
.                                                               .
.                        Group Record [1]                       .
.                                                               .
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
.                                                               .
.                        Group Record [2]                       .
.                                                               .
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               .                               |
.                               .                               .
|                               .                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
.                        Group Record [M]                       .
.                                                               .
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The main fields are:

| Field | Purpose |
|---|---|
| **Type** | `0x22` for an IGMPv3 Membership Report |
| **Reserved** | Set to 0 when transmitted; ignored when received |
| **Checksum** | 16-bit one's-complement checksum over the complete IGMP message |
| **Flags** | Extensibility field |
| **Number of Group Records (M)** | Number of Group Records contained in the Report |

The checksum covers the **entire IGMP message**, including all Group Records.

## IP Encapsulation

IGMPv3 Reports are carried directly inside IPv4:

```text
IPv4
└── IGMP
```

with:

```text
IP Protocol = 2
TTL         = 1
```

They also carry the **IP Router Alert** option.

The destination of a native IGMPv3 Report is:

```text
224.0.0.22
```

`224.0.0.22` is the **IGMPv3 Routers** multicast address.

```text
Receiver --------------------> 224.0.0.22
          IGMPv3 Report
```

All IGMPv3-capable multicast routers on the subnet listen to this address.

This differs from IGMPv1 and IGMPv2, whose Reports are normally sent directly to the group being reported:

```text
IGMPv1/v2:
Report for 239.1.1.1 → 239.1.1.1

IGMPv3:
Report containing state for 239.1.1.1 → 224.0.0.22
```

The source address should normally be a valid unicast address belonging to the local subnet.

However, a system that has not yet acquired an IPv4 address may use `0.0.0.0` as the source of an IGMP Report, and routers must accept such Reports.

## Group Record Format

Each Group Record describes the sender's membership state for **one multicast group**.

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Record Type  |  Aux Data Len |     Number of Sources (N)     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Multicast Address                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address [1]                      |
+-                                                             -+
|                       Source Address [2]                      |
+-                              .                              -+
.                               .                               .
.                               .                               .
+-                                                             -+
|                       Source Address [N]                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
.                         Auxiliary Data                        .
.                                                               .
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The fields are:

| Field | Purpose |
|---|---|
| **Record Type** | Identifies the type of membership state being reported |
| **Aux Data Len** | Length of Auxiliary Data in 32-bit words |
| **Number of Sources (N)** | Number of source addresses in the Group Record |
| **Multicast Address** | Multicast group described by the record |
| **Source Address [i]** | Source addresses associated with the group |
| **Auxiliary Data** | Reserved for extensions |

IGMPv3 itself does not define any Auxiliary Data, so:

```text
Aux Data Len = 0
```

for normal IGMPv3 Group Records.

## Multiple Groups in One Report

One of the major differences from IGMPv1 and IGMPv2 is that a single IGMPv3 Report can contain **multiple Group Records**.

For example:

```text
IGMPv3 Membership Report
│
├── Group Record: 239.1.1.1
│     INCLUDE {10.1.1.1}
│
├── Group Record: 239.2.2.2
│     EXCLUDE {}
│
└── Group Record: 232.3.3.3
      INCLUDE {10.3.3.3}
```

This lets a host report the state of several groups in one packet.

For example, a response to a General Query might contain:

```text
Number of Group Records = 3
```

rather than requiring three separate IGMP packets.

This helps offset the additional Report traffic created by IGMPv3's removal of host Report suppression.

## Current-State Reports

When a host responds to a Query, it sends **Current-State Group Records**.

There are two Current-State Record types:

```text
Type 1: MODE_IS_INCLUDE
Type 2: MODE_IS_EXCLUDE
```

For example:

```text
Group: 239.1.1.1
Mode:  INCLUDE
Sources:
  10.1.1.1
  10.2.2.2
```

is reported as:

```text
MODE_IS_INCLUDE
Group: 239.1.1.1
Sources:
  10.1.1.1
  10.2.2.2
```

A traditional ASM-style membership can be represented as:

```text
MODE_IS_EXCLUDE
Group: 239.1.1.1
Sources: {}
```

meaning:

```text
Receive 239.1.1.1 from all sources
```

The detailed semantics of INCLUDE and EXCLUDE mode are covered on the **IGMPv3 Source Filtering and INCLUDE/EXCLUDE Modes** page.

## Reports in Response to Queries

As covered in the IGMPv3 Queries section, a host waits a random period up to the Query's **Max Response Time** before sending a response.

The contents depend on the Query type.

### General Query

A General Query asks for the host's current reception state.

The host can include multiple Group Records in a single Report:

```text
General Query
      ↓
Host waits random delay
      ↓
IGMPv3 Report
├── MODE_IS_INCLUDE for G1
├── MODE_IS_EXCLUDE for G2
└── MODE_IS_INCLUDE for G3
```

### Group-Specific Query

A Group-Specific Query causes the host to report its current state for that group:

```text
Query:
G = 239.1.1.1

Host state:
INCLUDE {10.1.1.1, 10.2.2.2}

Response:
MODE_IS_INCLUDE
239.1.1.1
{10.1.1.1, 10.2.2.2}
```

### Group-and-Source-Specific Query

A Group-and-Source-Specific Query asks specifically about selected sources.

For example:

```text
Host state:
INCLUDE {S1, S2}

Query:
G, {S1, S3}
```

The host reports only the queried sources that it is interested in:

```text
MODE_IS_INCLUDE {S1}
```

The response rules for the three Query types are covered in detail on the **IGMPv3 Queries** page.

## State-Change Reports

IGMPv3 also sends Reports **without first receiving a Query** when the host's multicast reception state changes.

For example:

```text
Join a group
Add a source
Remove a source
Change INCLUDE → EXCLUDE
Change EXCLUDE → INCLUDE
Leave a group
```

A change causes an immediate **State-Change Report**.

IGMPv3 therefore does not require the separate IGMPv2:

```text
Leave Group — Type 0x17
```

message for native IGMPv3 operation.

Instead, the change is represented with a State-Change Group Record.

The four State-Change Record types are:

```text
Type 3: CHANGE_TO_INCLUDE_MODE
Type 4: CHANGE_TO_EXCLUDE_MODE
Type 5: ALLOW_NEW_SOURCES
Type 6: BLOCK_OLD_SOURCES
```

For example:

```text
Old state:
INCLUDE {S1}

New state:
INCLUDE {S1, S2}
```

can result in:

```text
ALLOW_NEW_SOURCES {S2}
```

These records are covered in detail on the **IGMPv3 Group Record Types** and **IGMPv3 State Changes** pages.

## State-Change Report Retransmission

State-Change Reports are important because they may cause the router to start or stop forwarding particular multicast sources.

To protect against packet loss, they are retransmitted according to the **Robustness Variable**.

The initial State-Change Report is sent immediately, followed by:

```text
Robustness Variable - 1
```

additional transmissions.

With the default:

```text
Robustness Variable = 2
```

the host therefore sends:

```text
Initial State-Change Report
        ↓
One additional State-Change Report
```

The retransmission delay is randomly selected between:

```text
0 and Unsolicited Report Interval
```

The RFC default **Unsolicited Report Interval** for IGMPv3 is:

```text
1 second
```

Therefore, with the default Robustness Variable of 2, a state change is normally reported twice.

> Do not confuse the IGMPv3 default **1-second Unsolicited Report Interval** with IGMPv2's default 10-second Unsolicited Report Interval.

## No Host Report Suppression

IGMPv3 removes the host-based Report suppression used by IGMPv1 and IGMPv2.

With IGMPv2:

```text
PC1 sends Report
        ↓
PC2 and PC3 hear Report
        ↓
PC2 and PC3 cancel their pending Reports
```

With IGMPv3:

```text
PC1 sends its Report
PC2 sends its Report
PC3 sends its Report
```

An IGMPv3 host does **not** cancel its pending Report because another host has reported similar membership state.

Some benefits:

- Allows routers to potentially track membership per receiver
- Simplifies host processing
- Works better with IGMP snooping

The increased traffic is partly offset by allowing multiple Group Records in one Report

RFC 9776 explicitly identifies removal of Report suppression as one of IGMPv3's design changes.

## Router Processing of Reports

An IGMPv3 router combines the Reports received from hosts into **aggregate reception state for the interface**.

For example, suppose:

```text
Rec1:
INCLUDE {10.1.1.1}

Rec2:
INCLUDE {10.2.2.2}
```

for group:

```text
239.1.1.1
```

The router can maintain aggregate interface state representing:

```text
Ethernet0/1

239.1.1.1
INCLUDE {
  10.1.1.1,
  10.2.2.2
}
```

The router therefore knows which groups and sources the attached network wants.

RFC 9776 describes router state conceptually as:

```text
(multicast address,
 Group Timer,
 Router Filter Mode,
 source records)
```

with each source record containing:

```text
(source address, Source Timer)
```

The router does not inherently need to retain a separate record for every individual host in order to calculate this aggregate forwarding state.

## IOS XE Verification

Enable IGMPv3 on the receiver-facing interface:

```text
R3(config)# interface Ethernet0/1
R3(config-if)# ip igmp version 3
```

Use:

```text
R3# show ip igmp groups                  
IGMP Connected Group Membership
Group Address    Interface                Uptime    Expires   Last Reporter   Group Accounted
239.1.1.1        Ethernet0/1              01:47:06  stopped   10.3.4.10       
224.0.1.40       Ethernet0/1              00:00:44  00:02:59  10.3.4.3  
```

to display directly connected group memberships.

For source-filter information:

```text
R3# show ip igmp groups 239.1.1.1 detail 

Flags: L - Local, U - User, SG - Static Group, VG - Virtual Group,
       SS - Static Source, VS - Virtual Source,
       Ac - Group accounted towards access control limit

Interface:      Ethernet0/1
Group:          239.1.1.1
Flags:
Uptime:         01:47:26
Group mode:     INCLUDE
Last reporter:  10.3.4.10
Group source list: (C - Cisco Src Report, U - URD, R - Remote, S - Static,
                    V - Virtual, M - SSM Mapping, L - Local,
                    Ac - Channel accounted towards access control limit)
  Source Address   Uptime    v3 Exp   CSR Exp   Fwd  Flags
  10.1.2.10        01:47:26  00:02:37  stopped   Yes  R
  10.1.2.11        00:00:26  00:02:33  stopped   Yes  R
```

This means the aggregate IGMP state on Ethernet0/1 includes receiver interest in:

```text
(10.1.2.10, 239.1.1.1)
and
(10.1.2.11, 239.1.1.1)
```

IOS XE also provides:

```text
R3# show ip igmp membership 239.1.1.1
Flags: A  - aggregate, T - tracked
       L  - Local, S - static, V - virtual, R - Reported through v3 
       I - v3lite, U - Urd, M - SSM (S,G) channel 
       1,2,3 - The version of IGMP, the group is in
Channel/Group-Flags: 
       / - Filtering entry (Exclude mode (S,G), Include mode (G))
Reporter:
       <mac-or-ip-address> - last reporter if group is not explicitly tracked
       <n>/<m>      - <n> reporter in include mode, <m> reporter in exclude

 Channel/Group                  Reporter        Uptime   Exp.  Flags  Interface 
 *,239.1.1.1                    2/0             01:47:57 stop  3AT    Et0/1
```

which displays IGMP membership information used for forwarding.

## Aggregate Membership State

Without explicit tracking, IOS XE can maintain the **aggregate group and source-filter state for the interface** without retaining a separate membership entry identifying exactly which receiver requested each `(S,G)` channel.

For example:

```text
R3# show ip igmp membership 239.1.1.1

 Channel/Group                  Reporter        Uptime   Exp.  Flags  Interface
 *,239.1.1.1                    10.3.4.10       01:30:52 00:54 3A     Et0/1
```

The:

```text
A
```

flag means:

```text
Aggregate
```

The `*,239.1.1.1` notation does **not** mean that receivers necessarily want traffic from all sources.

The actual source-filter state may simultaneously be:

```text
Group mode: INCLUDE

Source Address
10.1.2.10
10.1.2.11
```

meaning that the interface has interest in:

```text
(10.1.2.10, 239.1.1.1)
and
(10.1.2.11, 239.1.1.1)
```

The `*,G` entry is the aggregate **group-level membership entry**, not a statement that all sources are wanted.

Without explicit tracking, the `Reporter` field identifies the `last reporter` rather than listing every receiver:

```
R3(config-if)#do sh ip igmp membership 239.1.1.1
!
 Channel/Group                  Reporter        Uptime   Exp.  Flags  Interface 
 *,239.1.1.1                    10.3.4.11       02:01:38 stop  3A     Et0/1
 ```

In the above output, `10.3.4.11` is the last reporter for the `239.1.1.1` group.

## IGMP Explicit Tracking

IOS XE can additionally maintain membership state for **individual receivers** using:

```text
R3(config)# interface Ethernet0/1
R3(config-if)# ip igmp explicit-tracking
```

Cisco describes this feature as explicit tracking of IGMPv3 **hosts, groups, and channels**.

Without explicit tracking, IOS XE might know:

```text
Ethernet0/1

239.1.1.1
INCLUDE {
  10.1.2.10,
  10.1.2.11
}
```

With explicit tracking, it can additionally maintain information such as:

```text
10.3.4.10 wants (10.1.2.10, 239.1.1.1)
10.3.4.11 wants (10.1.2.11, 239.1.1.1)
```

The aggregate state is still maintained for multicast forwarding.

Use the following command to display explicitly tracked memberships:

```text
R3# show ip igmp membership 239.1.1.1 tracked 
Flags: A  - aggregate, T - tracked
       L  - Local, S - static, V - virtual, R - Reported through v3 
       I - v3lite, U - Urd, M - SSM (S,G) channel 
       1,2,3 - The version of IGMP, the group is in
Channel/Group-Flags: 
       / - Filtering entry (Exclude mode (S,G), Include mode (G))
Reporter:
       <mac-or-ip-address> - last reporter if group is not explicitly tracked
       <n>/<m>      - <n> reporter in include mode, <m> reporter in exclude

 Channel/Group                  Reporter        Uptime   Exp.  Flags  Interface 
 *,239.1.1.1                    2/0             00:00:38 stop  3AT    Et0/1
 10.1.2.10,239.1.1.1            10.3.4.10       00:00:38 02:21 T      Et0/1
 10.1.2.11,239.1.1.1            10.3.4.11       00:00:29 02:30 T      Et0/1
```

Important flags include:

```text
A = Aggregate
T = Tracked
```

So:

```text
3AT
```

indicates:

```text
IGMPv3
Aggregate
Tracked
```

while the individual `(S,G)` entry is marked as tracked.

> Explicit tracking does **not** enable IGMPv3 source filtering itself.
> IOS XE already maintains aggregate source-filter state without explicit tracking.
> Explicit tracking adds the ability to associate that membership state with individual receivers.

Explicit tracking consumes additional memory because the router must retain per-host membership state.

## Generating IGMPv3 Reports in a Lab

An IOS XE device can act as an IGMPv3 receiver for an `(S,G)` channel.

For example:

```text
Rec1(config)# interface Ethernet0/0
Rec1(config-if)# ip igmp version 3
Rec1(config-if)# ip igmp join-group 239.1.1.1 source 10.1.2.1
```

This creates receiver interest in:

```text
(10.1.2.1, 239.1.1.1)
```

and signals that interest using IGMPv3 INCLUDE-mode reporting.

## Report Size and MTU

Because an IGMPv3 Report may contain many groups and source addresses, it may not always fit in one packet.

If all required Group Records do not fit within the network MTU:

```text
Report
├── Group Records that fit
└── Remaining Group Records → additional Report(s)
```

If an individual Group Record contains too many sources to fit, most Record Types can be split across multiple Group Records in separate Reports.

`MODE_IS_EXCLUDE` and `CHANGE_TO_EXCLUDE_MODE` are exceptions: they are not split this way; the Report includes as many source addresses as fit.

This prevents IGMPv3's potentially large source lists from requiring IP fragmentation as the normal reporting mechanism.

## Report Summary

```text
IGMPv3 Membership Report
Type = 0x22
Destination = 224.0.0.22
        ↓
One or more Group Records
        ↓
Each Group Record describes:
Group + Filter Mode/State Change + Source List
```

The two broad classes of Group Records are:

```text
Group Records
├── Current-State Records
│   ├── Type 1: MODE_IS_INCLUDE
│   └── Type 2: MODE_IS_EXCLUDE
│
└── State-Change Records
    ├── Filter-Mode-Change Records
    │   ├── Type 3: CHANGE_TO_INCLUDE_MODE
    │   └── Type 4: CHANGE_TO_EXCLUDE_MODE
    │
    └── Source-List-Change Records
        ├── Type 5: ALLOW_NEW_SOURCES
        └── Type 6: BLOCK_OLD_SOURCES
```

The following sections examine **source filtering**, the individual **Group Record Types**, and **IGMPv3 state changes** in detail.