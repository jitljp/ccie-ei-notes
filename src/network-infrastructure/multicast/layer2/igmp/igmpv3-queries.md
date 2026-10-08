# IGMPv3 Queries

IGMPv3 routers use **Membership Queries** to discover and verify both:

- Multicast group membership
- Source-specific receiver interest

IGMPv3 uses the same Query message type as earlier versions:

```text
Type = 0x11
```

However, IGMPv3 extends the Query format to carry additional parameters and an optional list of source addresses.

IGMPv3 defines three Query types:

```text
General Query
Group-Specific Query
Group-and-Source-Specific Query
```

## IGMPv3 Query Format

An IGMPv3 Membership Query uses the following format:

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Type = 0x11  | Max Resp Code |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Group Address                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Flags |S| QRV |     QQIC      |     Number of Sources (N)     |
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
```

The first 8 bytes are based on the IGMPv2 Query format.

IGMPv3 adds:

- Flags
- **S** — Suppress Router-Side Processing
- **QRV** — Querier's Robustness Variable
- **QQIC** — Querier's Query Interval Code
- Number of Sources
- Optional source addresses

A General or Group-Specific Query with no sources is **12 bytes** long.

A Group-and-Source-Specific Query adds 4 bytes for each listed IPv4 source address.

## General Queries

A **General Query** asks hosts to report their complete multicast reception state on the interface.

It contains:

```text
Type:              0x11
Destination:       224.0.0.1
Group Address:     0.0.0.0
Number of Sources: 0
```

For example:

```text
R3 -----------------------> 224.0.0.1
          General Query
```

Hosts respond with IGMPv3 Membership Reports containing **Current-State Group Records** for the groups they are interested in.

Unlike IGMPv2, a single IGMPv3 Report can contain Current-State Records for multiple groups.

General Queries are sent periodically by the elected IGMP querier.

## Group-Specific Queries

A **Group-Specific Query** asks whether receivers still have interest in a particular multicast group.

For example:

```text
Group: 239.1.1.1
```

The Query contains:

```text
Type:              0x11
Destination:       239.1.1.1
Group Address:     239.1.1.1
Number of Sources: 0
```

Conceptually:

```text
R3 -----------------------> 239.1.1.1
       Group-Specific Query
```

Hosts with reception state for the group respond with a Current-State Record describing their current filter mode and source list.

Group-Specific Queries are commonly generated when the router receives a State-Change Report indicating that group-level membership may have been removed.

## Group-and-Source-Specific Queries

IGMPv3 adds the **Group-and-Source-Specific Query**.

This asks whether any receivers still want traffic for a particular group from one or more particular sources.

For example:

```text
Group:   239.1.1.1
Sources:
  10.1.1.1
  10.2.2.2
```

The Query contains:

```text
Type:              0x11
Destination:       239.1.1.1
Group Address:     239.1.1.1
Number of Sources: 2
Source 1:          10.1.1.1
Source 2:          10.2.2.2
```

Conceptually:

```text
R3 -----------------------> 239.1.1.1

"Does anyone still want:

 (10.1.1.1, 239.1.1.1)
 (10.2.2.2, 239.1.1.1) ?"
```

This allows IGMPv3 to verify **source-specific interest** rather than checking only whether the group itself still has receivers.

Group-and-Source-Specific Queries are generated in response to relevant **State-Change Reports**, not normal Current-State Reports.

## Query Type Identification

The three IGMPv3 Query types can be identified from the **Group Address** and **Number of Sources** fields:

| Query | Group Address | Number of Sources |
|---|---|---:|
| General | `0.0.0.0` | 0 |
| Group-Specific | Group G | 0 |
| Group-and-Source-Specific | Group G | > 0 |

The IP destinations are:

| Query | IP Destination |
|---|---|
| General | `224.0.0.1` |
| Group-Specific | Group G |
| Group-and-Source-Specific | Group G |

## Max Resp Code

The **Max Resp Code (MRC)** specifies the maximum amount of time hosts may wait before responding to a Query.

As with IGMPv2, the actual **Max Response Time** is expressed in units of **1/10 second**.

However, IGMPv3 changes the encoding.

If:

```text
Max Resp Code < 128
```

the value is interpreted directly:

```text
Max Response Time = MRC × 0.1 seconds
```

For example:

```text
MRC = 100
```

means:

```text
10 seconds
```

### Floating-Point Encoding

Values of 128 or greater use a floating-point representation:

```text
0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+
|1| exp | mant  |
+-+-+-+-+-+-+-+-+
```

The value is calculated as:

```text
Max Response Time
= (mant | 0x10) << (exp + 3)
```

in units of **1/10 second**.

Therefore, IGMPv3 can represent much larger Max Response Times than IGMPv2's simple linear encoding.

For example:

```text
MRC = 0x96
```

gives:

```text
exp  = 1
mant = 6

(6 | 0x10) << (1 + 3)
= 22 << 4
= 352
```

Therefore:

```text
Max Response Time = 35.2 seconds
```

Values from `0` through `127` therefore directly represent up to:

```text
12.7 seconds
```

while larger values use the exponential encoding.

> The maximum IGMPv3 Max Response Time that can be encoded in the 8-bit MRC field is 3174.4 seconds (~52.9 minutes).

## Host Response Delay

When a host receives a Query for which it has state to report, it does not normally respond immediately.

It selects a random delay between:

```text
0 and Max Response Time
```

before sending its response.

For example:

```text
Max Response Time = 10 seconds

Rec1 → 2.4 seconds
Rec2 → 7.1 seconds
Rec3 → 8.6 seconds
```

Unlike IGMPv2, however, **IGMPv3 hosts do not suppress their Reports when another host reports first**.

Each host reports its own reception state.

## Responses to the Three Query Types

The content of the response depends on the Query.

### General Query

A host reports its current reception state for **all applicable groups**.

For example:

```text
239.1.1.1 → INCLUDE {10.1.1.1}
239.2.2.2 → EXCLUDE {}
```

Both can be included as Group Records in the same IGMPv3 Membership Report.

### Group-Specific Query

A host reports its complete current reception state for the specified group.

For example:

```text
Query:
239.1.1.1

Host state:
INCLUDE {10.1.1.1, 10.2.2.2}
```

The host responds with a Current-State Record representing:

```text
MODE_IS_INCLUDE {
  10.1.1.1,
  10.2.2.2
}
```

### Group-and-Source-Specific Query

A Group-and-Source-Specific Query is more selective.

Assume the host has:

```text
INCLUDE {10.1.1.1, 10.2.2.2}
```

and receives a Query for:

```text
{10.1.1.1, 10.3.3.3}
```

The host is interested only in the intersection:

```text
{10.1.1.1}
```

so it responds with:

```text
MODE_IS_INCLUDE {10.1.1.1}
```

This is an important difference between a Group-Specific Query and a Group-and-Source-Specific Query: the latter asks about **only the listed sources**.

## QRV — Querier's Robustness Variable

The 3-bit **QRV** field advertises the querier's **Robustness Variable**.

For example, with:

```text
Robustness Variable = 2
```

the querier sends:

```text
QRV = 2
```

Other IGMPv3 routers on the subnet can learn and use this value.

This is an improvement over IGMPv2, where the Robustness Variable is not carried in the Query.

Because QRV is only 3 bits, it can represent:

```text
0–7
```

If the querier's Robustness Variable is greater than 7:

```text
QRV = 0
```

A received QRV of `0` means that routers use their statically configured Robustness Variable, or the protocol default if none is configured.

On IOS XE:

```text
R3(config)# interface Ethernet0/1
R3(config-if)# ip igmp robustness-variable 3
```

changes the value advertised in IGMPv3 Queries to:

```text
QRV = 3
```

and also affects RV-derived IGMP behavior.

## QQIC — Querier's Query Interval Code

The **QQIC** advertises the querier's Query Interval.

This lets non-querier IGMPv3 routers learn the Query Interval directly from the elected querier.

For values below 128:

```text
QQIC = Query Interval in seconds
```

For example:

```text
QQIC = 60
```

means:

```text
Query Interval = 60 seconds
```

Values of 128 or greater use the same floating-point encoding as the Max Resp Code:

```text
0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+
|1| exp | mant  |
+-+-+-+-+-+-+-+-+
```

with:

```text
QQI = (mant | 0x10) << (exp + 3)
```

This time, the result is measured in **seconds**, not tenths of a second.

On IOS XE, the default Query Interval is:

```text
60 seconds
```

and can be changed with:

```text
R3(config)# interface Ethernet0/1
R3(config-if)# ip igmp query-interval 30
```

An IGMPv3 General Query sent by R3 then advertises:

```text
QQIC = 30
```

## S — Suppress Router-Side Processing

The **S flag** means:

```text
Suppress Router-Side Processing
```

When:

```text
S = 1
```

other multicast routers receiving the Query do **not** perform the normal group/source timer updates caused by that Query.

However, the S flag does **not** suppress:

- Host processing of the Query
- Host responses
- IGMP querier election

Hosts still process and respond to the Query normally.

The S flag is particularly important during Group-Specific and Group-and-Source-Specific Query processing, where routers may be retransmitting Queries while different group or source timers are already running.

Conceptually:

```text
S = 0
→ Hosts process Query
→ Routers process Query and adjust applicable timers

S = 1
→ Hosts process Query normally
→ Other routers do not adjust those timers
```

This prevents a specific Query from incorrectly shortening router-side state that should continue to use its existing timer.

## Number of Sources

The **Number of Sources (N)** field is 16 bits and identifies how many source addresses follow the fixed Query header.

For:

```text
General Query
Group-Specific Query
```

the value is:

```text
N = 0
```

For a Group-and-Source-Specific Query:

```text
N > 0
```

Each source address consumes another:

```text
4 bytes
```

The number of sources that can fit in one Query is therefore limited by the interface MTU.

With a 1500-byte Ethernet MTU, RFC 9776 gives a maximum of 366 source addresses in a single Query.

## Specific Queries and State Changes

IGMPv3 does not use an IGMPv2-style Leave Group message for native IGMPv3 operation.

Instead, hosts send **State-Change Group Records**.

If a State-Change Report indicates that a group or particular sources may no longer be wanted, the querier verifies whether other receivers still require them.

For group-level state:

```text
State-Change Report
        ↓
Group-Specific Query
        ↓
Does anyone still want G?
```

For source-specific state:

```text
State-Change Report
        ↓
Group-and-Source-Specific Query
        ↓
Does anyone still want (S,G)?
```

The router continues forwarding the queried group or sources during this verification period.

Only after the **Last Member Query Time** expires without a Report confirming continued interest can that state be removed.

## Last Member Query Parameters

As with IGMPv2, specific Queries use the **Last Member Query Interval (LMQI)** and **Last Member Query Count (LMQC)**.

By default:

```text
LMQI = 1 second
LMQC = Robustness Variable = 2
```

Therefore, a specific Query is transmitted **2 times** at approximately **1 second intervals**.

The total Last Member Query Time is:

```text
LMQT = LMQI × LMQC
```

With the defaults:

```text
1 × 2 = 2 seconds
```

For Group-Specific and Group-and-Source-Specific Queries, the LMQI is also used to calculate the **Max Resp Code**.

On IOS XE:

```text
R3(config)# interface Ethernet0/1
R3(config-if)# ip igmp last-member-query-interval 500
```

sets the interval to:

```text
500 ms
```

The query count can be changed with:

```text
R3(config-if)# ip igmp last-member-query-count 3
```

These knobs therefore affect both group-level and source-specific membership verification.

## Pending Query Responses

A host can receive multiple Queries before an earlier response timer expires.

IGMPv3 does not necessarily schedule a completely separate Report for every Query.

It can **merge pending responses**.

For example, if multiple Group-and-Source-Specific Queries arrive for the same group:

```text
Query 1:
G, {S1, S2}

Query 2:
G, {S3}
```

the host can maintain a combined pending source list:

```text
G, {S1, S2, S3}
```

and send a single response when the earliest applicable response timer expires.

Similarly, a pending General Query response can make a separate response to a narrower Query unnecessary because the General Query response will already report the required state.

This reduces unnecessary IGMPv3 Report traffic while still ensuring that the querier receives the required membership information.

## Query Version Identification

IGMPv1, IGMPv2, and IGMPv3 all use:

```text
Type = 0x11
```

for Membership Queries.

The version is identified from the Query length and Max Resp Code:

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

An IGMPv3 host that detects an older-version querier enters the appropriate compatibility mode.

## IP Encapsulation

Like all IGMPv3 messages, Queries are carried directly inside IPv4:

```text
IPv4
└── IGMP
```

with:

```text
IP Protocol = 2
TTL         = 1
```

IGMPv3 Queries also carry the **IP Router Alert option**.

This keeps IGMP signaling local to the subnet and causes routers to examine the packet as control traffic.

## IOS XE Query Configuration

Enable IGMPv3 on the interface with:

```text
R3(config)# interface Ethernet0/1
R3(config-if)# ip igmp version 3
```

Important Query-related knobs include:

```text
! Periodic General Query interval
R3(config-if)# ip igmp query-interval 60

! Max Response Time for General Queries
R3(config-if)# ip igmp query-max-response-time 10

! Robustness Variable advertised as QRV
R3(config-if)# ip igmp robustness-variable 2

! Interval / Max Response Time for specific Queries
R3(config-if)# ip igmp last-member-query-interval 1000

! Number of specific Query transmissions
R3(config-if)# ip igmp last-member-query-count 2
```

Important IOS XE defaults include:

| Parameter | Default |
|---|---:|
| IGMP version | 2 |
| Query Interval | 60 seconds |
| Maximum Query Response Time | 10 seconds |
| Robustness Variable | 2 |
| Last Member Query Interval | 1000 ms |
| Last Member Query Count | 2 |

> As with IGMPv2, IOS XE's default **60-second Query Interval** differs from the RFC default of **125 seconds**.

Verify the operational values with:

```text
R3# show ip igmp interface Ethernet0/1
```

and observe Query processing directly with:

```text
R3# debug ip igmp
```

A packet capture is particularly useful for viewing the IGMPv3-specific Query fields:

```text
MRC
S
QRV
QQIC
Number of Sources
Source Addresses
```

## Query Summary

```text
General Query
→ "Tell me your multicast reception state."

Group-Specific Query
→ "Does anyone still want group G?"

Group-and-Source-Specific Query
→ "Does anyone still want these sources for group G?"
```

The key IGMPv3 improvement is that the router can verify membership at the **source level**, not just the group level.