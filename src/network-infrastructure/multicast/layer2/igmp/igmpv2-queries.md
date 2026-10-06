# IGMPv2 Queries

IGMPv2 routers use **Membership Queries** to determine whether multicast groups still have receivers on a local subnet.

IGMPv2 defines two kinds of Query:

- **General Query** — asks about all multicast groups
- **Group-Specific Query** — asks about one particular multicast group

Both use the IGMP Type value:

```text
0x11
```

They are distinguished by the **Group Address** field.

## General Queries

The IGMP querier periodically sends **General Queries** to discover which multicast groups have receivers on the subnet.

A General Query uses:

```text
Type:              0x11
Destination:       224.0.0.1
Group Address:     0.0.0.0
Max Resp Time:     Query Response Interval
TTL:               1
```

`224.0.0.1` is the **All Hosts** multicast address.

For example:

```text
R1 ------------------------> 224.0.0.1
          General Query

Type:              0x11
Group Address:     0.0.0.0
Max Resp Time:     10 seconds
```

The source IP address is the querier's interface address.

This source address is also used by IGMPv2 routers during **querier election**, which is covered separately in the IGMP Querier section.

## Query Interval

The **Query Interval** determines how frequently the querier sends General Queries.

RFC 2236 defines a default of **125 seconds**.

IOS XE instead uses a default of **60 seconds**:

```text
R3# show ip igmp interface e0/1 | include query interval
  IGMP query interval is 60 seconds
  IGMP configured query interval is 60 seconds
```

The IOS XE interval can be changed with:

```text
R3(config)# interface Ethernet0/1
R3(config-if)# ip igmp query-interval 30
```

This causes R3, when acting as the querier, to send a General Query every 30 seconds.

Verify the timer with:

```text
R3# show ip igmp interface e0/1 | include query interval
  IGMP query interval is 30 seconds
  IGMP configured query interval is 30 seconds
```

> IOS XE's default Query Interval of 60 seconds differs from the RFC 2236 default of 125 seconds.

## Max Response Time

Every IGMPv2 Query contains a **Max Resp Time** field.

This tells hosts how long they may wait before responding.

The field is encoded in units of:

```text
1/10 second
```

For example:

```text
Max Resp Time = 100
```

represents:

```text
10 seconds
```

For General Queries, this value is known as the **Query Response Interval**.

The RFC default and IOS XE default are both **10 seconds**:

```text
R3# show ip igmp interface e0/1 | i response time
  IGMP max query response time is 10 seconds
```

On IOS XE, you can modify the timer with this command:

```text
R3(config-if)# ip igmp query-max-response-time 5

R3# show ip igmp interface e0/1 | i response time
  IGMP max query response time is 5 seconds
```

## Host Response to a General Query

When a host receives a General Query, it does not immediately send Reports for every group it has joined.

Instead, it starts a separate **random response timer** for each group membership.

For example, assume PC1 belongs to:

```text
239.1.1.1
239.2.2.2
239.3.3.3
```

and receives a Query with:

```text
Max Resp Time = 10 seconds
```

PC1 might select:

```text
239.1.1.1 → 2.7 seconds
239.2.2.2 → 7.4 seconds
239.3.3.3 → 4.1 seconds
```

When each timer expires, the host sends a Membership Report for that group.

The random delays spread host responses over time rather than causing every receiver to respond simultaneously.

If the host hears another host's Membership Report for the same group before its timer expires, it suppresses its own Report.

## Receiving Another Query While a Timer Is Running

A host may receive another Query while its response timer for a group is already running.

If the new Query's requested Max Response Time is **less than the remaining time** on the existing timer, the host selects a new random delay within the shorter interval.

For example:

```text
Current timer remaining:     7 seconds
New Max Resp Time:           2 seconds
```

The host replaces the existing timer with a new random value:

```text
0 < timer <= 2 seconds
```

If the existing timer already expires sooner than the new Query requires, it is left unchanged.

This prevents a Query requesting a fast response from being delayed by an older, longer timer.

## Group-Specific Queries

A **Group-Specific Query** asks whether any receivers remain for one particular multicast group.

For example:

```text
Type:              0x11
Destination:       239.1.1.1
Group Address:     239.1.1.1
```

Unlike a General Query:

```text
General Query
Destination:       224.0.0.1
Group Address:     0.0.0.0
```

a Group-Specific Query is sent directly to the multicast group being queried:

```text
Group-Specific Query
Destination:       239.1.1.1
Group Address:     239.1.1.1
```

Only hosts that belong to that group need to respond.

## Group-Specific Queries and the Leave Process

Group-Specific Queries are particularly important when a host leaves a group.

For example:

```text
PC1 -----------------------> R1
          Leave Group
          Group 239.1.1.1

R1 ------------------------> 239.1.1.1
          Group-Specific Query
```

The querier does not immediately assume that the group has no receivers.

Another host on the subnet may still belong to the group.

Instead, the querier sends Group-Specific Queries to determine whether any receivers remain.

The complete process is covered on the **IGMPv2 Leave Process** page.

## Last Member Query Interval

The **Last Member Query Interval (LMQI)** controls the timing of Group-Specific Queries sent as part of the leave process.

The default is `1000 ms` (1 second):

```
R3# show ip igmp interface e0/1 | i Last member
  Last member query count is 2
  Last member query response interval is 1000 ms
```

The LMQI serves two purposes:

1. It is the interval between Group-Specific Queries.
2. It is placed in the Query's Max Resp Time field.

Therefore, with the default:

```text
Group-Specific Query

Max Resp Time = 1 second
```

On IOS XE, the LMQI is configured in **milliseconds**:

```text
R3(config-if)# ip igmp last-member-query-interval 500

R3# show ip igmp interface e0/1 | i Last member
  Last member query count is 2
  Last member query response interval is 500 ms
```

Lowering the LMQI reduces the time required to determine that the last receiver has left, but also gives remaining receivers less time to respond.

## Last Member Query Count

The **Last Member Query Count (LMQC)** determines how many Group-Specific Queries are sent during the leave process.

The default is **2**:

```
R3# show ip igmp interface e0/1 | i Last member
  Last member query count is 2
  Last member query response interval is 1000 ms
```

You can modify it with this command:

```text
R3(config-if)# ip igmp last-member-query-count 3

R3# show ip igmp interface e0/1 | i Last member
  Last member query count is 3
  Last member query response interval is 1000 ms
```

causes three Group-Specific Queries to be sent.

The approximate time spent checking for remaining members is therefore:

```text
Last Member Query Time = LMQC × LMQI
```

With the defaults:

```text
LMQC = 2
LMQI = 1 second

2 × 1 second = 2 seconds
```

Thus, IGMPv2 can normally determine that the last receiver has left much faster than IGMPv1,
which can take up to **3 minutes** to time out the group membership.

## General Query vs Group-Specific Query

| | General Query | Group-Specific Query |
|---|---|---|
| Type | `0x11` | `0x11` |
| IP destination | `224.0.0.1` | Group being queried |
| Group Address | `0.0.0.0` | Group being queried |
| Purpose | Discover/refresh all memberships | Check one specific group |
| Typical Max Resp Time | 10 seconds | Last Member Query Interval |
| Normally sent | Periodically | During leave processing |

## Important IGMPv2 Timers

RFC 2236 defines the following default values:

| Timer / Variable | RFC Default |
|---|---:|
| Robustness Variable | 2 |
| Query Interval | 125 seconds |
| Query Response Interval | 10 seconds |
| Startup Query Interval | 1/4 Query Interval |
| Startup Query Count | Robustness Variable |
| Last Member Query Interval | 1 second |
| Last Member Query Count | Robustness Variable |

### Robustness Variable

The **Robustness Variable (RV)** allows IGMP to be tuned for the expected amount of packet loss on a subnet.

The default is **2**:

```text
R3# show ip igmp interface e0/1 | i robustness
  IGMP robustness-variable is 2
```

IGMP is designed to tolerate `RV - 1` lost IGMP messages.

Therefore:

```text
RV = 2 → tolerate 1 lost message
RV = 3 → tolerate 2 lost messages
RV = 4 → tolerate 3 lost messages
```

Increasing the Robustness Variable improves tolerance of packet loss, but also causes some IGMP state to be retained longer and some messages to be transmitted more times.

```
R3(config-if)# ip igmp robustness-variable 3
[Ethernet0/1] Warning: Please be aware of the other IGMP parameters that may
be affected by this configuration change.
```

> RFC 2236 says the RV MUST NOT be zero, and SHOULD NOT be one.

The Robustness Variable influences several other IGMP values, including:

```text
Startup Query Count
Last Member Query Count
Group Membership Interval
Other Querier Present Interval
```

For example:

```text
R3# sh run int e0/1
Building configuration...

Current configuration : 132 bytes
!
interface Ethernet0/1
 ip address 10.3.4.3 255.255.255.0
 ip pim dense-mode
 ip igmp robustness-variable 3   <-- ONLY THIS HAS BEEN CONFIGURED
 ip ospf 1 area 0
end

R3# show ip igmp interface e0/1
Ethernet0/1 is up, line protocol is up
  Internet address is 10.3.4.3/24
  IGMP is enabled on interface
  Current IGMP host version is 2
  Current IGMP router version is 2
  IGMP query interval is 60 seconds
  IGMP configured query interval is 60 seconds
  IGMP robustness-variable is 3   <-- NEW RV OF 3
  IGMP querier timeout is 180 seconds   <--- MODIFIED BY RV
  IGMP configured querier timeout is 120 seconds
  IGMP max query response time is 10 seconds
  Last member query count is 3   <-- MODIFIED BY RV
  Last member query response interval is 1000 ms
  Inbound IGMP access group is not set
  IGMP activity: 3 joins, 2 leaves
  Multicast routing is enabled on interface
  Multicast TTL threshold is 0
  Multicast designated router (DR) is 10.3.4.4  
  IGMP querying router is 10.3.4.3 (this system)
  No multicast groups joined by this system
```

Notice that changing only the Robustness Variable to 3 also changes:

```
Last Member Query Count: 2 → 3
Operational querier timeout: 120 → 180 seconds
```

The configured querier timeout remains at its default of `120 seconds`,
but the operational querier timeout is adjusted to `180 seconds` based on the new Robustness Variable.

> Manually configuring an **RV-dependent timer or count** overrides the value that would otherwise be derived from the Robustness Variable.

> IGMPv2 does **not** advertise the Robustness Variable in its Query messages. Therefore, non-default values should be consistent among the multicast routers on the same subnet. IGMPv3 adds a Querier's Robustness Variable (QRV) field to its Query format.

### Group Membership Interval

An important derived timer is the **Group Membership Interval**:

```text
Group Membership Interval
= (Robustness Variable × Query Interval)
  + Query Response Interval
```

Using the RFC defaults:

```text
(2 × 125) + 10
= 260 seconds
```

The Group Membership Interval determines how long a router can go without receiving a Report before concluding that a group has no receivers.

**IMPORTANT**: IOS XE does **not** use the RFC-defined timer. Instead, it uses:

```
Group Membership Interval
= (Query Interval x 3)
```

For a default of `60 x 3 = 180 seconds`.

> This timer can be modified by configuring the Query Interval with `ip igmp query-interval`.

### Startup Queries

When a router becomes the querier, it initially sends Queries more rapidly so that it can quickly discover existing group memberships.

RFC 2236 defines:

```text
Startup Query Interval = Query Interval / 4
Startup Query Count    = Robustness Variable
```

Using the RFC defaults:

```text
Startup Query Interval = 125 / 4
                       = 31.25 seconds

Startup Query Count = 2
```

> I can't find documentation on IOS XE's behavior regarding Startup Queries, but it is safe to assume it is different than the RFC definition.

## IGMPv1 Query Compatibility

IGMPv1 and IGMPv2 both use Type `0x11` for Membership Queries.

An IGMPv1 Query can be identified because its **Max Resp Time field is 0**.

```text
IGMPv1 Query
Type:          0x11
Max Resp Time: 0
Group Address: 0.0.0.0
```

An IGMPv2 host that receives an IGMPv1 Query must behave compatibly with the older router for a period of time.

During this compatibility period, it sends **IGMPv1 Membership Reports** rather than IGMPv2 Reports and does not send IGMPv2 Leave Group messages.

## IOS XE Query Configuration

The most important IGMPv2 query-related knobs are (default values shown):

```text
R3(config)# interface Ethernet0/1

! General Query frequency
R3(config-if)# ip igmp query-interval 60

! Max Resp Time in General Queries
R3(config-if)# ip igmp query-max-response-time 10

! Group-Specific Query interval / Max Resp Time
R3(config-if)# ip igmp last-member-query-interval 1000

! Number of Group-Specific Queries during leave processing
R3(config-if)# ip igmp last-member-query-count 2

! Robustness variable that affects several other values
R3(config-if)# ip igmp robustness-variable 2
```

Verify the operational values with:

```text
R3# show ip igmp interface Ethernet0/0
```

For troubleshooting, IGMP events can also be observed with:

```text
R3# debug ip igmp
```
