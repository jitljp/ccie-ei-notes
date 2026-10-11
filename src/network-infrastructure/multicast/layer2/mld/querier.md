
# MLD Querier Operation

The **MLD querier** periodically sends Multicast Listener Queries to discover and maintain IPv6 multicast listener information on a local link.

Both **MLDv1 and MLDv2** use an independent querier election mechanism, similar to IGMPv2 and IGMPv3.

When multiple multicast routers are connected to the same link, only one router normally sends periodic MLD General Queries.

The other routers continue processing MLD messages and maintaining listener state.

## MLD Querier Election

The router with the **lowest link-local IPv6 address** becomes the MLD querier.

For example:

```text
             R1               R2
          fe80::1           fe80::2
              \              /
               \            /
                    SW1
                   /   \
                 PC1   PC2
```

Both routers initially consider themselves eligible to become the querier.

### Election Process

**Step 1:** Each router sends an MLD General Query.

```text
R1:
Source:      fe80::1
Destination: ff02::1

R2:
Source:      fe80::2
Destination: ff02::1
```

`ff02::1` is the **All Nodes** link-local multicast address.

**Step 2:** Each router compares the source address of received Queries with its own link-local address.

When R2 receives R1's Query:

```text
Received source: fe80::1
Local address:   fe80::2

fe80::1 < fe80::2

R2 becomes Non-Querier
```

When R1 receives R2's Query:

```text
Received source: fe80::2
Local address:   fe80::1

fe80::2 > fe80::1

R1 remains Querier
```

The result is:

```text
R1 → Querier
R2 → Non-Querier
```

Only R1 continues sending periodic General Queries.

> Unlike IGMPv2/v3, which compares IPv4 interface addresses, MLD compares link-local IPv6 addresses. Specifically, RFC 9777 defines the comparison using the 64-bit interface identifier portion of the `fe80::/64` address.

The election is **preemptive**. A router with a lower link-local address can take over when it begins sending Queries.

There is no configurable MLD querier priority.

## MLD Querier vs PIM Designated Router

The **MLD querier election** and **PIM Designated Router (DR) election** are independent.

| Role | Election |
|---|---|
| MLD Querier | Lowest link-local IPv6 address |
| PIM DR | Highest DR priority, then highest IPv6 address |

For example, assume both routers have the default PIM DR priority of `1`:

```text
R1: fe80::1
R2: fe80::2
```

The results can be:

```text
MLD Querier = R1
PIM DR      = R2
```

The MLD querier sends Queries to discover and maintain local receiver membership information.

The PIM DR performs multicast routing functions, such as sending PIM Joins toward the source or RP.

> A router does not need to be the PIM DR to act as the MLD querier.

## MLD Query Types

The querier can send different Queries depending on the MLD version.

| Query | MLDv1 | MLDv2 | Purpose |
|---|---|---|---|
| General | Yes | Yes | Discover or refresh all listener memberships |
| Multicast Address Specific | Yes | Yes | Check for remaining listeners of a particular group |
| Multicast Address and Source Specific | No | Yes | Check for remaining listeners interested in particular sources |

All three use **ICMPv6 Type 130**.

### General Queries

General Queries are sent periodically:

```text
Source:           fe80::1
Destination:      ff02::1
Multicast Address::: 
```

The Multicast Address field contains the unspecified address (`::`).

Hosts respond with Multicast Listener Reports according to their MLD version and membership state.

### Multicast Address Specific Queries

These Queries check whether listeners remain for a particular multicast group.

For example:

```text
Source:           fe80::1
Destination:      ff15::1234
Multicast Address:ff15::1234
```

They are commonly generated when a receiver leaves a group.

### Multicast Address and Source Specific Queries

MLDv2 additionally supports Queries targeting specific source/group combinations.

For example:

```text
Multicast Address: ff3e::1234
Sources:
  2001:db8::1
  2001:db8::2
```

The querier uses these Queries to determine whether receivers still want traffic from the specified multicast sources.

These source-specific Queries are particularly important for **Source-Specific Multicast (SSM)**.

All MLD Queries use:

```text
IPv6 Hop Limit: 1
IPv6 Source:    Link-local address
Router Alert:   0 (MLD)
```

## Non-Querier Operation

A router that loses the election enters the **Non-Querier** state.

It stops sending periodic General Queries but continues:

- Receiving and processing MLD Reports
- Maintaining multicast listener state
- Processing MLD Queries
- Tracking the active querier

This allows it to take over if the current querier fails.

### Other Querier Present Timer

A non-querier maintains an **Other Querier Present Timer**.

When it receives a Query from a router with a lower link-local IPv6 address, it refreshes this timer.

```text
R1 sends General Query
          |
          v
R2 receives Query
          |
          v
R2 refreshes
Other Querier Present Timer
          |
          v
R2 remains Non-Querier
```

If the timer expires without receiving another qualifying Query, R2 becomes the querier.

### Other Querier Present Interval

[RFC 2710](https://www.rfc-editor.org/rfc/rfc2710.html) and [RFC 9777](https://www.rfc-editor.org/rfc/rfc9777.html) define:

```text
Other Querier Present Interval
= (Robustness Variable × Query Interval)
  + (Query Response Interval / 2)
```

Using the default values:

```text
Robustness Variable     = 2
Query Interval          = 125 seconds
Query Response Interval = 10 seconds

(2 × 125) + (10 / 2)
= 255 seconds
```

Therefore, the default RFC-defined timeout is **255 seconds**.

This is separate from the Multicast Address Listening Interval, which controls the expiration of multicast listener state.

> The querier timeout determines when another router can take over. It does not directly determine when a multicast group or source membership expires.

## Querier Failure and Takeover

Suppose:

```text
R1 (fe80::1) → Querier
R2 (fe80::2) → Non-Querier
```

R1 sends periodic General Queries.

If R1 fails:

```text
R1 fails
    |
    v
R2 stops receiving Queries
    |
    v
Other Querier Present Timer expires
    |
    v
R2 becomes Querier
    |
    v
R2 starts sending General Queries
```

R2 does not need to wait for its existing MLD membership state to expire before becoming the querier.

If R1 subsequently recovers, it begins sending Queries again.

Because R1 has the lower link-local IPv6 address, R2 returns to the Non-Querier state.

## MLDv2 Querier Parameter Synchronization

MLDv2 introduces two fields in Query messages that allow routers to coordinate their operating parameters.

These are equivalent to the IGMPv3 fields:

- **QRV (Querier's Robustness Variable):** Advertises the querier's Robustness Variable.
- **QQIC (Querier's Query Interval Code):** Advertises the querier's Query Interval.

For example:

```text
R1 (Querier)
Robustness Variable = 3
Query Interval      = 60 seconds
```

R1 advertises:

```text
QRV  = 3
QQIC = 60
```

A non-querier receiving the MLDv2 Query can use these values for its operating parameters.

RFC 9777 specifies that:

- A nonzero QRV is adopted by receiving multicast routers.
- A nonzero QQIC is adopted by non-querier routers as their operational Query Interval.
- QRV = 0 causes the receiver to use its configured or default Robustness Variable.
- QQIC = 0 causes a non-querier to use the default Query Interval.

If the querier's Robustness Variable exceeds 7, it advertises QRV = 0 because the field is only 3 bits long.

**MLDv1 does not include QRV or QQIC**, so it cannot advertise these parameters.

> These are RFC-defined behaviors. Actual IOS XE parameter adoption should be verified on the platform, especially when routers have different locally configured values.

## MLD Querier Timers

RFC 2710 and RFC 9777 define the following default values:

| Parameter | Default | Purpose |
|---|---:|---|
| Robustness Variable | 2 | Tolerance for lost MLD messages |
| Query Interval | 125 seconds | Interval between General Queries |
| Query Response Interval | 10 seconds | Maximum delay for responses to General Queries |
| Other Querier Present Interval | 255 seconds | Timeout before querier takeover |
| Startup Query Interval | 31.25 seconds | Interval between startup Queries |
| Startup Query Count | 2 | Number of startup Queries |
| Last Listener Query Interval | 1 second | Interval used for leave-related Queries |
| Last Listener Query Count | 2 | Number of leave-related Queries |

### Startup Queries

When a router becomes the querier, it initially sends General Queries more frequently.

The RFC defaults are:

```text
Startup Query Interval = Query Interval / 4
Startup Query Count    = Robustness Variable
```

For example:

```text
Query Interval = 125 seconds

Startup Query Interval = 125 / 4
                       = 31.25 seconds

Startup Query Count = 2
```

The router sends its initial General Query, followed by another approximately 31.25 seconds later, before continuing its normal Query schedule.

This helps a newly elected querier discover listener state more quickly.

### Last Listener Queries

When the querier needs to verify whether listeners remain for a multicast address or source, it can send repeated specific Queries.

The **Last Listener Query Time (LLQT)** is:

```text
LLQT = Last Listener Query Interval
       × Last Listener Query Count
```

With default values:

```text
LLQT = 1 × 2
     = 2 seconds
```

During this period, the router checks for Reports indicating continued interest.

If no relevant listener interest is reported, the router can allow the corresponding membership state to expire and stop requesting the multicast traffic.

This applies to:

- Multicast Address Specific Queries
- Multicast Address and Source Specific Queries (MLDv2)

## Cisco IOS XE Configuration

On IOS XE, IPv6 multicast routing enables router-side MLD processing on IPv6-enabled interfaces.

Unlike typical IPv4 multicast configuration, which requires enabling PIM on an interface to activate router-side IGMP processing, **IPv6 MLD does not require a separate per-interface PIM command**.

Enable IPv6 multicast routing:

```text
R1(config)# ipv6 unicast-routing
R1(config)# ipv6 multicast-routing
```

For example:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 enable
```

The router automatically generates a link-local IPv6 address and can participate in MLD querier election.

### Controlling Querier Election

Because the lowest link-local address wins, you can influence the election by manually configuring link-local addresses.

On R1:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 address fe80::1 link-local
```

On R2:

```text
R2(config)# interface Ethernet0/0
R2(config-if)# ipv6 address fe80::2 link-local
```

R1 is the preferred querier.

> There is no `ipv6 mld querier-priority` command. The election is determined by link-local IPv6 addresses.

### Query Interval

The default IOS XE MLD Query Interval is **125 seconds**.

To modify it:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 mld query-interval 60
```

R1 now sends General Queries every 60 seconds while acting as the querier.

### Query Maximum Response Time

The default Query Maximum Response Time is **10 seconds**.

To change it to 5 seconds:

```text
R1(config-if)# ipv6 mld query-max-response-time 5
```

This value controls the maximum delay hosts may use when responding to General Queries.

The command accepts seconds, although the MLD message field represents milliseconds.

### Querier Timeout

Configure the time that a non-querier waits before taking over:

```text
R2(config)# interface Ethernet0/0
R2(config-if)# ipv6 mld query-timeout 130
```

R2 now uses a configured querier timeout of 130 seconds.

**Important:** This timer should be coordinated with the Query Interval and Query Maximum Response Time.

Cisco documents the standard relationship:

```text
Querier Timeout
= (2 × Query Interval)
  + (Query Max Response Time / 2)
```

For example:

```text
Query Interval          = 60 seconds
Query Max Response Time = 20 seconds

(2 × 60) + (20 / 2)
= 130 seconds
```

A timeout that is too short can cause unnecessary querier takeovers.

A timeout that is too long delays failure detection.

Cisco references differ on the exact default timeout, with some command references listing 250 seconds and others 255 seconds. Check the operational value on the specific IOS XE platform.

### Disabling MLD Router-Side Processing

To disable router-side MLD processing on an interface:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# no ipv6 mld router
```

The router stops acting as an MLD router on that interface.

This disables MLD querying and router-side listener processing, and prevents IPv6 multicast routing toward that interface.

Restore the function with:

```text
R1(config-if)# ipv6 mld router
```

This is independent of Layer 2 MLD snooping.

## Verification

Use the following command to verify the MLD Querier:

```
R1# show ipv6 mld interface e0/1
Ethernet0/1 is up, line protocol is up
  Internet address is FE80::1/10
  MLD is enabled on interface
  Current MLD version is 2
  MLD query robustness value is 2
  MLD query interval is 125 seconds
  MLD querier timeout is 255 seconds
  MLD max query response time is 10 seconds
  Last member query response interval is 1 seconds
  MLD activity: 5 joins, 0 leaves
  MLD querying router is FE80::1 (this system)   <-- THIS ROUTER IS THE QUERIER
```

## MLD Router Querier vs MLD Snooping Querier

An MLD router querier should not be confused with an **MLD snooping querier**.

| Feature | MLD Router Querier | MLD Snooping Querier |
|---|---|---|
| Sends MLD General Queries | Yes | Yes |
| Maintains router-side MLD state | Yes | No |
| Requires IPv6 multicast routing | Yes | No |
| Routes IPv6 multicast | Yes | No |
| Main purpose | Discover receiver interest for multicast routing | Refresh Layer 2 snooping memberships |

A Layer 2 switch can use an MLD snooping querier when there is no multicast router on the VLAN.

On supported Catalyst IOS XE platforms, this is a separate feature enabled with:

```text
SW1(config)# ipv6 mld snooping querier
```

Further details are covered in the MLD Snooping section.

## Key Points

- MLDv1 and MLDv2 elect the router with the **lowest link-local IPv6 address** as the querier.
- Querier election is independent of the PIM DR election.
- Non-querier routers stop sending periodic General Queries but continue maintaining listener state.
- The **Other Querier Present Timer** controls takeover when the active querier fails.
- The default RFC-defined Other Querier Present Interval is **255 seconds**.
- MLDv2 adds QRV and QQIC fields for advertising querier parameters.
- The querier sends General Queries periodically and specific Queries when checking whether multicast listeners remain.
- On IOS XE, `ipv6 multicast-routing` enables IPv6 multicast processing without requiring a separate per-interface PIM command.
- Use `show ipv6 mld interface` to identify the elected querier and verify timers.
