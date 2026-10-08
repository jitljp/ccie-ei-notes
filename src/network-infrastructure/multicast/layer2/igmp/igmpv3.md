# IGMPv3

**IGMPv3** extends IGMP by adding **source filtering**.

IGMPv1 and IGMPv2 allow a host to express interest in a multicast group:

```text
Receive group G
```

IGMPv3 allows a host to express interest in a multicast group **and specific sources**:

```text
Receive group G from source S
```

or:

```text
Receive group G from all sources except S
```

This gives multicast routers information about both:

- Which multicast **groups** have receivers
- Which multicast **sources** those receivers want or do not want

IGMPv3 is particularly important for **Source-Specific Multicast (SSM)**, but it also supports traditional **Any-Source Multicast (ASM)**.

> IGMPv3 is currently specified by **RFC 9776**, which obsoletes the original IGMPv3 specification, RFC 3376.

## Source Filtering

IGMPv3 maintains multicast reception state in the form:

```text
(Group, Filter Mode, Source List)
```

The two filter modes are:

```text
INCLUDE
EXCLUDE
```

### INCLUDE Mode

**INCLUDE mode** means:

```text
Receive traffic for group G
ONLY from the listed sources
```

For example:

```text
Group:       232.1.1.1
Mode:        INCLUDE
Source List: 10.1.1.1
```

means:

```text
Receive (10.1.1.1, 232.1.1.1)
```

but not traffic for `232.1.1.1` from other sources.

This is the model used by **SSM**.

### EXCLUDE Mode

**EXCLUDE mode** means:

```text
Receive traffic for group G
from all sources EXCEPT the listed sources
```

For example:

```text
Group:       239.1.1.1
Mode:        EXCLUDE
Source List: 10.1.1.1
```

means:

```text
Receive 239.1.1.1 from all sources
except 10.1.1.1
```

An empty EXCLUDE list:

```text
EXCLUDE {}
```

means:

```text
Receive group G from all sources
```

This corresponds closely to the traditional **ASM group join** used by IGMPv1 and IGMPv2.

Source filtering and the precise meaning of INCLUDE and EXCLUDE states are covered later.

## IGMPv3 and SSM

IGMPv3 enables a receiver to explicitly request an `(S,G)` channel.

For example:

```text
Source: 10.1.1.1
Group:  232.1.1.1
```

The receiver can signal:

```text
INCLUDE {10.1.1.1}
```

for `232.1.1.1`.

Conceptually:

```text
Receiver
   |
   | I want (10.1.1.1, 232.1.1.1)
   v
Local multicast router
   |
   | Build multicast state toward 10.1.1.1
   v
Source
```

Unlike ASM, the receiver already knows the desired source, so SSM does not require an RP or source-discovery mechanism.

IGMPv3 itself, however, is **not limited to SSM**. It support traditional ASM membership as well by using EXCLUDE mode with an empty source list,
in other words excluding no sources.

## IGMPv3 Queries

IGMPv3 continues to use:

```text
Type = 0x11
```

for Membership Queries.

It supports three Query types:

```text
General Query
Group-Specific Query
Group-and-Source-Specific Query
```

The **Group-and-Source-Specific Query** is new in IGMPv3 and allows a router to ask whether receivers still want traffic from particular sources for a group.

For example:

```text
Group:   232.1.1.1
Sources: 10.1.1.1, 10.2.2.2
```

An IGMPv3 Query also adds fields such as:

```text
S    = Suppress Router-Side Processing
QRV  = Querier's Robustness Variable
QQIC = Querier's Query Interval Code
```

and can carry a list of source addresses.

These fields allow IGMPv3 routers to communicate protocol parameters that IGMPv2 routers had to infer or configure independently.

IGMPv3 Query behavior and format are covered on the **IGMPv3 Queries** page.

## IGMPv3 Membership Reports

IGMPv3 introduces a completely redesigned Membership Report.

The IGMPv3 Report type is:

```text
0x22
```

Unlike IGMPv1 and IGMPv2 Reports, which are sent to the multicast group being reported, IGMPv3 Reports are sent to:

```text
224.0.0.22
```

`224.0.0.22` is the **IGMPv3 Routers** multicast address.

For example:

```text
Receiver --------------------> 224.0.0.22
          IGMPv3 Report
```

All multicast routers running IGMPv3 listen to this address.

## Group Records

An IGMPv3 Membership Report can describe multiple multicast groups in a **single packet**.

The Report contains one or more **Group Records**:

```text
IGMPv3 Membership Report
│
├── Group Record: 239.1.1.1
├── Group Record: 232.2.2.2
└── Group Record: 232.3.3.3
```

Each Group Record can describe:

- The multicast group
- The filter mode
- A list of sources
- A current state or state change

This is a major difference from IGMPv1 and IGMPv2, where each Membership Report represents a single multicast group.

The Group Record types are covered in detail later.

## No Host Report Suppression

IGMPv3 removes the host-based **Membership Report suppression** used by IGMPv1 and IGMPv2.

With IGMPv2:

```text
PC1 sends Report
        ↓
PC2 hears it
        ↓
PC2 suppresses its own Report
```

With IGMPv3:

```text
PC1 sends Report
PC2 sends Report
PC3 sends Report
```

Hosts do not cancel their pending Reports merely because they hear another host report similar membership state.

> Because only IGMPv3 **routers** listen to 224.0.0.22, an IGMPv3 host will discard other hosts' Reports even if they arrive on an interface.

This allows routers to potentially track membership on a per-host basis and avoids problems that IGMPv1/v2 report suppression caused for IGMP snooping switches.

IGMPv3 reduces the additional packet overhead by allowing a single Report to contain **multiple Group Records**.

## State Changes

IGMPv3 does not rely on a separate IGMPv2-style **Leave Group** message when operating in native IGMPv3 mode.

> The IGMPv1/IGMPv2 Leave Group message is not used in IGMPv3, except for backward compatibility.

Instead, changes in multicast reception state are communicated using **State-Change Group Records**.

For example, a receiver can report that:

```text
A new source should be allowed
A source should be blocked
The filter mode changed to INCLUDE
The filter mode changed to EXCLUDE
```

Therefore, IGMPv3 can signal changes at both the **group level** and the **source level**.

For example:

```text
Old state:
INCLUDE {10.1.1.1}

New state:
INCLUDE {10.1.1.1, 10.2.2.2}
```

The host can indicate that `10.2.2.2` is a newly desired source rather than simply reporting an undifferentiated group membership.

State changes are covered in detail on the **IGMPv3 State Changes** page.

## IGMPv3 Router State

IGMPv3 allows a multicast router to maintain both **group state** and **source state**.

Instead of knowing only:

```text
239.1.1.1 has receivers on Ethernet0/0
```

the router can know information such as:

```text
232.1.1.1 has receivers on Ethernet0/0 interested in sources:

10.1.1.1
10.2.2.2
```

Conceptually:

```text
IGMPv2:
(*,G) receiver interest

IGMPv3:
(*,G) and/or (S,G) receiver interest
```

The router combines the reception state reported by hosts on the interface into the multicast reception state required for that network.

A router is not inherently required to maintain a separate membership record for every individual host.

## Backward Compatibility

IGMPv3 is designed to interoperate with IGMPv1 and IGMPv2.

IGMPv3 hosts can detect older-version Queriers and enter the appropriate compatibility mode.

Likewise, IGMPv3 routers recognize older Membership Report and Leave message formats.

A useful way to distinguish Query versions is:

```text
IGMPv1 Query:
Length = 8 bytes
Max Resp Code = 0

IGMPv2 Query:
Length = 8 bytes
Max Resp Code != 0

IGMPv3 Query:
Length >= 12 bytes
```

When an IGMPv3 host detects an older-version Querier, it uses the corresponding older reporting behavior for compatibility.

## Cisco IOS XE

IOS XE uses IGMPv2 by default.

Configure IGMPv3 on an interface with:

```text
R3(config)# interface Ethernet0/1
R3(config-if)# ip igmp version 3
```

Verify it with:

```text
R3# show ip igmp interface e0/1 | i Current
  Current IGMP host version is 3
  Current IGMP router version is 3
```

View learned memberships with:

```text
R3# show ip igmp groups 
IGMP Connected Group Membership
Group Address    Interface                Uptime    Expires   Last Reporter   Group Accounted
239.1.1.1        Ethernet0/1              01:24:27  00:02:56  10.3.4.10       
224.0.1.40       Ethernet0/0              02:06:32  00:01:56  10.12.34.3  
```

or for additional source information:

```text
R3# show ip igmp groups 239.1.1.1 detail 

Flags: L - Local, U - User, SG - Static Group, VG - Virtual Group,
       SS - Static Source, VS - Virtual Source,
       Ac - Group accounted towards access control limit

Interface:      Ethernet0/1
Group:          239.1.1.1
Flags:
Uptime:         01:32:22
Group mode:     INCLUDE
Last reporter:  10.3.4.10
Group source list: (C - Cisco Src Report, U - URD, R - Remote, S - Static,
                    V - Virtual, M - SSM Mapping, L - Local,
                    Ac - Channel accounted towards access control limit)
  Source Address   Uptime    v3 Exp   CSR Exp   Fwd  Flags
  10.1.2.10        00:02:58  00:02:30  stopped   Yes  R
```

> Note the INCLUDE mode with 10.1.2.10 in the source list. This means that there is a receiver interested in receiving (10.1.2.10, 239.1.1.1).

> IOS XE can optionally maintain IGMP membership state for individual receivers using **IGMP explicit tracking**. This is covered on the **IGMPv3 Membership Reports** page.

## Joining an (S,G) Channel

IOS XE can act as an IGMPv3 receiver for a specific source and group:

```text
Rec1(config)# interface Ethernet0/0
Rec1(config-if)# ip igmp version 3
Rec1(config-if)# ip igmp join-group 232.1.1.1 source 10.1.1.1
```

This requests:

```text
(10.1.1.1, 232.1.1.1)
```

using **INCLUDE mode**.

Verify it with:

```text
Rec1# show ip igmp groups detail 

Flags: L - Local, U - User, SG - Static Group, VG - Virtual Group,
       SS - Static Source, VS - Virtual Source,
       Ac - Group accounted towards access control limit

Interface:      Ethernet0/0
Group:          239.1.1.1
Flags:          L 
Uptime:         00:20:45
Group mode:     INCLUDE
Last reporter:  10.3.4.10
Group source list: (C - Cisco Src Report, U - URD, R - Remote, S - Static,
                    V - Virtual, M - SSM Mapping, L - Local,
                    Ac - Channel accounted towards access control limit)
  Source Address   Uptime    v3 Exp   CSR Exp   Fwd  Flags
  10.1.2.10        00:20:45  stopped   stopped   Yes  L
```

> When using `ip igmp join-group <group> source <source>`, configure `ip igmp version 3` on the interface. Without IGMPv3 enabled, IOS XE can create local `(S,G)` state but does not send the corresponding IGMPv3 Membership Report.

For SSM forwarding through the multicast network, SSM must also be enabled on the routers, for example:

```text
R1(config)# ip pim ssm default
```

This enables the default SSM range:

```text
232.0.0.0/8
```

> More on PIM-SSM in the PIM section.

## IGMPv2 vs IGMPv3

| Feature | IGMPv2 | IGMPv3 |
|---|---|---|
| Group membership | Yes | Yes |
| Source filtering | No | Yes |
| INCLUDE / EXCLUDE modes | No | Yes |
| `(S,G)` receiver signaling | No | Yes |
| SSM support | No native source signaling | Yes |
| Group-and-Source-Specific Query | No | Yes |
| Report destination | Group address | `224.0.0.22` |
| Multiple groups per Report | No | Yes |
| Host Report suppression | Yes | No |
| Separate Leave Group message | Yes | No in native v3 operation |

The following sections examine IGMPv3 Queries, Membership Reports, source filtering, Group Record types, and state changes in detail.