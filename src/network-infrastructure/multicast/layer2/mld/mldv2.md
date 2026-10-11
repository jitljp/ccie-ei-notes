# MLDv2

**Multicast Listener Discovery version 2 (MLDv2)** is the IPv6 equivalent of **IGMPv3**.

MLDv2 was originally defined in RFC 3810, which has been obsoleted by [RFC 9777](https://www.rfc-editor.org/rfc/rfc9777.html).

Like IGMPv3, MLDv2 allows IPv6 hosts to specify not only which multicast groups they want to receive, but also **which multicast sources** they want or do not want to receive traffic from.

The major improvements over MLDv1 are:

- **Source filtering** using INCLUDE and EXCLUDE modes
- **Source-Specific Multicast (SSM)** support
- **Multicast Address and Source Specific Queries**
- Membership Reports containing multiple Multicast Address Records
- State-change reporting instead of separate Done messages
- Elimination of normal host Report suppression

Because these mechanisms are nearly identical to IGMPv3, this section focuses primarily on their MLDv2-specific implementation.

Check out the IGMPv3 pages for more details.

## MLDv2 Message Types

MLDv2 uses **ICMPv6 (Next Header 58)**.

| Message | ICMPv6 Type | IPv6 Destination |
|---|---:|---|
| Multicast Listener Query | 130 | `ff02::1` or the group being queried |
| MLDv2 Multicast Listener Report | 143 | `ff02::16` |

MLDv2 retains ICMPv6 Type `130` for Queries, but introduces **Type 143** for its redesigned Reports.

For backward compatibility, MLDv2 implementations also recognize:

- Type `131`: MLDv1 Multicast Listener Report
- Type `132`: MLDv1 Multicast Listener Done

MLDv2 does not use a separate Done message in native MLDv2 operation. Leaves and other membership changes are signaled through **State-Change Records**.

## MLDv2 Query Format

MLDv2 extends the 24-byte MLDv1 Query format to include additional parameters and an optional list of source addresses.

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Type = 130  |      Code     |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Maximum Response Code     |           Reserved            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                                                               |
+                       Multicast Address                       +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Flags |S| QRV |      QQIC     |      Number of Sources (N)     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                                                               |
+                       Source Address [1]                      +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                              ...                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The important additions are:

| Field | Size | Purpose |
|---|---|---|
| Flags | 4 bits | Reserved for extensions |
| S | 1 bit | Suppress Router-Side Processing |
| QRV | 3 bits | Querier's Robustness Variable |
| QQIC | 8 bits | Querier's Query Interval Code |
| Number of Sources | 16 bits | Number of source addresses in the Query |
| Source Addresses | 128 bits each | IPv6 unicast sources being queried |

These fields function essentially the same way as their IGMPv3 equivalents.

An MLDv2 Query without source addresses is **28 bytes**, compared to the 24-byte MLDv1 Query.

### MLDv2 Query Types

MLDv2 supports three Query types:

| Query | Multicast Address | Sources |
|---|---|---|
| General Query | `::` | None |
| Multicast Address Specific Query | Group G | None |
| Multicast Address and Source Specific Query | Group G | One or more |

The corresponding IPv6 destinations are:

- **General Query:** `ff02::1` (All Nodes)
- **Multicast Address Specific Query:** The multicast group being queried
- **Multicast Address and Source Specific Query:** The multicast group being queried

For example, suppose a router wants to determine whether receivers still want:

```text
(2001:db8:10::10, ff35::fd00:1234)
```

It can send:

```text
ICMPv6 Type:       130
IPv6 Destination:  ff35::fd00:1234
Multicast Address: ff35::fd00:1234
Number of Sources: 1
Source Address:    2001:db8:10::10
```

This is equivalent to an IGMPv3 Group-and-Source-Specific Query, but uses 128-bit IPv6 addresses.

### Maximum Response Code

One important difference from IGMPv3 is the **Maximum Response Code (MRC)** field.

| Protocol | Field Size | Time Units |
|---|---:|---|
| IGMPv3 | 8 bits | 1/10 second |
| MLDv2 | 16 bits | 1 millisecond |

In MLDv2, values below `32768` are interpreted directly as milliseconds.

For example:

```text
Maximum Response Code = 10000

Maximum Response Delay = 10000 ms
                       = 10 seconds
```

Values of `32768` or greater use floating-point encoding:

```text
 0 1 2 3 4 5 6 7 8 9 A B C D E F
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1| exp |          mant         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The Maximum Response Delay is calculated as:

```text
(mant | 0x1000) << (exp + 3)
```

in milliseconds.

This permits substantially larger response delays than MLDv1's linear 16-bit field.

### S, QRV, and QQIC

These fields work like their IGMPv3 counterparts:

- **S (Suppress Router-Side Processing):** Prevents receiving routers from performing certain normal timer updates caused by the Query. Hosts still process the Query.
- **QRV:** Advertises the querier's Robustness Variable. The default is `2`.
- **QQIC:** Advertises the querier's Query Interval, in seconds. Values of 128 or greater use floating-point encoding.

As with IGMPv3, QRV and QQIC allow non-querier routers to learn relevant protocol parameters from the elected querier.

## MLDv2 Membership Reports

MLDv2 introduces the **Version 2 Multicast Listener Report**, using ICMPv6 Type `143`.

Unlike MLDv1 Reports, which are sent to the group being reported, MLDv2 Reports are sent to:

```text
ff02::16
```

This is the **All MLDv2-Capable Routers** multicast address.

### Report Format

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Type = 143  |    Reserved   |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             Flags             |  Number of Address Records (M)|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
.                  Multicast Address Record [1]                 .
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
.                  Multicast Address Record [2]                 .
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                              ...                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Each **Multicast Address Record** describes the receiver's membership state for one IPv6 multicast group.

The record format is:

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Record Type |  Aux Data Len |      Number of Sources (N)    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                       Multicast Address                       +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                       Source Address [1]                      +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                              ...                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Auxiliary Data                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The structure is equivalent to an IGMPv3 Group Record, but the Multicast Address and each Source Address are **128 bits instead of 32 bits**.

For normal MLDv2 operation, `Aux Data Len` is zero.

### Multicast Address Record Types

MLDv2 uses the same six Record Types as IGMPv3:

| Type | Record | Purpose |
|---|---|---|
| 1 | `MODE_IS_INCLUDE` | Current INCLUDE state |
| 2 | `MODE_IS_EXCLUDE` | Current EXCLUDE state |
| 3 | `CHANGE_TO_INCLUDE_MODE` | Changed to INCLUDE |
| 4 | `CHANGE_TO_EXCLUDE_MODE` | Changed to EXCLUDE |
| 5 | `ALLOW_NEW_SOURCES` | Sources newly permitted |
| 6 | `BLOCK_OLD_SOURCES` | Sources no longer wanted |

Types 1 and 2 are **Current-State Records**, typically sent in response to Queries.

Types 3 through 6 are **State-Change Records**, sent when a receiver's membership or source-filtering state changes.

As in IGMPv3, a receiver can include records for multiple multicast groups in a single Report.

## Source Filtering and SSM

MLDv2 supports **INCLUDE** and **EXCLUDE** source-filtering modes.

For example:

```text
Group:  ff35::fd00:1234
Mode:   INCLUDE
Source: 2001:db8:10::10
```

The receiver requests traffic for:

```text
(2001:db8:10::10, ff35::fd00:1234)
```

but not for the same group from other sources.

This is the mechanism used by IPv6 SSM.

An EXCLUDE-mode receiver instead requests traffic from all sources except those in its source list.

The detailed behavior of INCLUDE/EXCLUDE filtering, source-list changes, and router aggregation is covered in the following **MLDv2 Source Filtering** section.

## No Host Report Suppression

Like IGMPv3, **MLDv2 removes normal host-based Report suppression**.

In MLDv1, a host can suppress its Report after hearing another host report the same group.

In native MLDv2 operation, hosts report their own membership state even when another host reports similar state.

> Because MLDv2 Reports are sent to `224.0.0.16`, which only MLDv2 routers listen to, hosts don't hear each other's Reports.

This allows routers to potentially track individual receivers and avoids issues that Report suppression can cause for MLD snooping.

A single MLDv2 Report can contain multiple Multicast Address Records, reducing the overhead of independent Reports.

## IPv6 Packet Requirements

MLDv2 retains the same IPv6 packet requirements as MLDv1:

```text
Next Header:        58 (ICMPv6)
Hop Limit:          1
Router Alert:       0 (MLD)
Source Address:     Link-local IPv6 address
```

**Queries must use a valid link-local IPv6 source address.**

Reports normally use a link-local source address, but may use the unspecified address (`::`) when the sender has not yet acquired a valid link-local address, such as during DAD.

MLD messages are exchanged only on the local link.

## MLDv1 Compatibility

MLDv2 supports backward compatibility with MLDv1 hosts and routers.

MLD versions can be distinguished by Query length:

| Version | Query Length |
|---|---|
| MLDv1 | 24 bytes |
| MLDv2 | At least 28 bytes |

### MLDv1 Querier Present

When an MLDv2 host receives an MLDv1 General Query, it enters **MLDv1 host compatibility mode**.

While this mode is active, the host uses MLDv1 reporting behavior.

When the Older-Version-Querier-Present Timer expires, it can return to MLDv2 operation.

On a link containing an MLDv1 router, the querier must use MLDv1. The RFC requires administrators to ensure the routers use the appropriate version.

### MLDv1 Hosts Present

An MLDv2 router can also receive MLDv1 Reports from older hosts.

When this occurs, the router maintains MLDv1 compatibility state for the affected multicast group.

An MLDv1 Report is treated as equivalent to:

```text
MODE_IS_EXCLUDE {}
```

This represents interest in receiving the multicast group from **all sources**.

An MLDv1 Done message is treated as a transition to:

```text
CHANGE_TO_INCLUDE_MODE {}
```

which indicates that the receiver no longer wants traffic for that group.

The router still performs the necessary membership verification rather than immediately removing traffic.

> An important limitation is that **MLDv1 cannot signal source-specific membership**. Native IPv6 SSM therefore requires MLDv2, unless a separate mechanism such as SSM mapping is used.

## MLDv2 Timers

MLDv2 uses timers similar to IGMPv3.

RFC 9777 defines the following defaults:

| Parameter | Default |
|---|---:|
| Robustness Variable | 2 |
| Query Interval | 125 seconds |
| Query Response Interval | 10 seconds |
| Startup Query Interval | 1/4 Query Interval |
| Startup Query Count | 2 |
| Last Listener Query Interval | 1 second |
| Last Listener Query Count | 2 |
| Unsolicited Report Interval | 1 second |

### Multicast Address Listening Interval

The **Multicast Address Listening Interval (MALI)** determines how long an MLDv2 router can retain multicast listener state without receiving a Report that refreshes it.

MLDv2 maintains membership information for multicast groups and individual sources. The interval is used when managing the timers associated with that state.

For example, if a router has learned that receivers on Ethernet0/0 want:

```text
(2001:db8:10::10, ff35::fd00:1234)
```

and receives no further Reports refreshing that membership, the corresponding state can eventually expire. The router can then stop requesting or forwarding that multicast traffic toward the link if no other listener state requires it.

A normal membership refresh resets the applicable timer. MALI is therefore a **membership aging interval**, not the interval between Queries or the time required to process an explicit leave.

#### MLDv2 MALI Calculation

RFC 9777 defines:

```text
MALI = (Robustness Variable × Query Interval)
       + (2 × Query Response Interval)
```

The default parameters are:

| Parameter | Default |
|---|---:|
| Robustness Variable (RV) | 2 |
| Query Interval (QI) | 125 seconds |
| Query Response Interval (QRI) | 10 seconds |

Therefore:

```text
MALI = (2 × 125) + (2 × 10)
     = 270 seconds
```

The additional Query Response Intervals provide time for delayed Reports and help prevent membership state from expiring prematurely, particularly when the querier changes.

#### Comparison with MLDv1 and IGMP

The corresponding membership timer has different names:

- **MLDv1:** Multicast Listener Interval
- **MLDv2:** Multicast Address Listening Interval (MALI)
- **IGMPv2/v3:** Group Membership Interval (GMI)

The RFC-defined formulas are:

| Protocol | Formula | RFC Default |
|---|---|---:|
| IGMPv2 | RV × QI + QRI | 260 seconds |
| IGMPv3 | RV × QI + (2 × QRI) | 270 seconds |
| MLDv1 | RV × QI + QRI | 260 seconds |
| MLDv2 | RV × QI + (2 × QRI) | 270 seconds |

All four use the same default RV, QI, and QRI values.

The important difference is:

```text
IGMPv2 / MLDv1:
(2 × 125) + 10 = 260 seconds

IGMPv3 / MLDv2:
(2 × 125) + 20 = 270 seconds
```

MLDv2 and IGMPv3 also apply these timers to **source-specific receiver state**, whereas MLDv1 and IGMPv2 maintain only group-level membership information.

#### Why the Extra Query Response Interval?

Consider what happens when the current MLDv2 querier fails.

The **Other Querier Present Interval** defaults to:

```text
(RV × QI) + (QRI / 2)

= (2 × 125) + 5
= 255 seconds
```

A backup router can therefore become the querier after approximately 255 seconds without hearing the existing querier.

Suppose a membership timer has not been refreshed during that period:

```text
t = 0
  Last membership refresh

t = 255
  Other Querier Present Interval expires
  New querier sends General Query

t = 265
  Host may respond up to 10 seconds later

t = 270
  MALI expires if no refresh was received
```

With the older 260-second membership interval, the router could remove membership state **before a host's valid response to the new querier arrives**.

Extending the interval to 270 seconds provides an additional 10 seconds of margin.

This change helps prevent temporary multicast interruptions during querier failover.

#### RFC Version Difference

The original IGMPv3 specification (RFC 3376) and MLDv2 specification (RFC 3810) both defined the older formula:

```text
(RV × QI) + QRI
```

Their replacements, **RFC 9776 (IGMPv3)** and **RFC 9777 (MLDv2)**, published in March 2025, use:

```text
(RV × QI) + (2 × QRI)
```

This change addresses a potential premature membership expiration during querier failover.

> **IOS XE Note:** In my test using IOS XE `17.16.1a`, the `MALI` is 260 seconds. Newer versions may or may not use the updated timers.

### Last Listener Query Time

MLDv2 uses the **Last Listener Query Interval (LLQI)** and **Last Listener Query Count (LLQC)** to verify whether a group or source still has interested receivers.

With the defaults:

```text
LLQI = 1 second
LLQC = 2

Last Listener Query Time = 2 seconds
```

The same mechanism can verify whether receivers remain interested in a particular `(S,G)` channel, rather than only a group.

## Cisco IOS XE Configuration

**MLDv2 is the default MLD version on IOS XE** when IPv6 multicast routing is enabled.

Enable IPv6 multicast routing:

```text
R1(config)# ipv6 multicast-routing
```

MLD router-side processing is then enabled on the router's IPv6-enabled interfaces by default.

### MLDv2 Source-Specific Membership

IOS XE can act as an MLDv2 receiver for a particular `(S,G)` channel.

For example:

```text
Rec1(config)# interface Ethernet0/0
Rec1(config-if)# ipv6 mld join-group ff35::fd00:1234 include 2001:db8:10::10
```

This requests traffic for:

```text
(2001:db8:10::10, ff35::fd00:1234)
```

using INCLUDE-mode reporting.

The `join-group` command makes the router itself a multicast receiver.

To establish forwarding interest without making the router a local receiver, use:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 mld static-group ff35::fd00:1234 include 2001:db8:10::10
```

### MLD Query Configuration

The main IOS XE Query timers are:

```text
! General Query interval (seconds)
R1(config-if)# ipv6 mld query-interval 60

! Maximum response time in Queries (seconds)
R1(config-if)# ipv6 mld query-max-response-time 10

! Other querier timeout (seconds)
R1(config-if)# ipv6 mld query-timeout 130
```

The normal Query Interval is **125 seconds**, and the default Maximum Response Time is **10 seconds**.

Although MLDv2 encodes the Maximum Response Delay in milliseconds, the IOS XE `ipv6 mld query-max-response-time` command accepts seconds.

## MLDv1 vs. MLDv2

| Feature | MLDv1 | MLDv2 |
|---|---|---|
| IPv4 equivalent | IGMPv2 | IGMPv3 |
| Query ICMPv6 Type | 130 | 130 |
| Report ICMPv6 Type | 131 | 143 |
| Report destination | Group address | `ff02::16` |
| Source filtering | No | Yes |
| INCLUDE/EXCLUDE modes | No | Yes |
| Source-specific Queries | No | Yes |
| Multiple groups per Report | No | Yes |
| Host Report suppression | Yes | No |
| Separate Done message | Yes | No (native v2) |
| Native SSM support | No | Yes |

## Key Points

- **MLDv2 is the IPv6 equivalent of IGMPv3**, currently specified by RFC 9777.
- MLDv2 Queries use **ICMPv6 Type 130**, while MLDv2 Reports use **Type 143**.
- Reports are sent to **`ff02::16`**, rather than the multicast group being reported.
- MLDv2 adds source filtering, `(S,G)` membership signaling, and source-specific Queries.
- The Maximum Response Code is **16 bits**, with delays expressed in milliseconds.
- MLDv2 Reports contain one or more **Multicast Address Records**, using the same six Record Types as IGMPv3.
- Normal host Report suppression is eliminated.
- MLDv2 supports MLDv1 compatibility, but MLDv1 cannot provide native source-specific signaling.
- **MLDv2 is the default on Cisco IOS XE**.