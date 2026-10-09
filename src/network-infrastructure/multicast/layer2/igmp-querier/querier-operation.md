# IGMP Querier Operation and Configuration

The **IGMP querier** periodically sends IGMP Queries to discover and maintain multicast receiver membership information on a subnet.

When multiple multicast routers are connected to the same subnet, the querier selection process depends on the IGMP version.

## IGMPv1 Querier Selection

IGMPv1 does **not** define its own querier election mechanism.

On Cisco routers running PIM, the **PIM Designated Router (DR)** performs the IGMPv1 querier function.

PIM DR election uses:

1. Highest PIM DR priority
2. Highest IP address (tie-breaker)

The default PIM DR priority is `1`.

For example:

```text
            R1               R2
        10.10.10.1       10.10.10.2
             \              /
              \            /
                   SW1
                  /   \
                PC1   PC2
```

Assuming both routers use the default PIM DR priority:

```text
PIM DR = R2 (10.10.10.2)

IGMPv1 Querier = R2
```

Because IGMPv1 has no independent querier election, the multicast routing protocol is used to coordinate the querier role.

## IGMPv2 and IGMPv3 Querier Election

IGMPv2 introduced an **independent querier election mechanism**, which IGMPv3 also uses.

The router with the **lowest IPv4 address** on the subnet becomes the IGMP querier.

The election takes place through IGMP General Query messages.

### Election Process

Consider:

```text
            R1               R2
        10.10.10.1       10.10.10.2
             \              /
              \            /
                   SW1
                  /   \
                PC1   PC2
```

Both routers initially consider themselves eligible to act as the querier.

**Step 1:** Each router sends an IGMP General Query to `224.0.0.1`.

```text
R1:
Source IP = 10.10.10.1
Destination = 224.0.0.1

R2:
Source IP = 10.10.10.2
Destination = 224.0.0.1
```

**Step 2:** Each router compares the source IP address of received General Queries with its own interface address.

When R2 receives R1's Query:

```text
Received source: 10.10.10.1
Local address:   10.10.10.2

10.10.10.1 < 10.10.10.2

R2 becomes Non-Querier
```

When R1 receives R2's Query:

```text
Received source: 10.10.10.2
Local address:   10.10.10.1

10.10.10.2 > 10.10.10.1

R1 remains Querier
```

The result is:

```text
R1 → Querier
R2 → Non-Querier
```

Only R1 sends periodic General Queries.

R2 continues maintaining IGMP membership information by processing received Membership Reports and relevant Queries.

## IGMP Querier vs PIM DR

The IGMPv2/v3 querier election and the PIM DR election are independent.

For example, assuming equal PIM DR priorities:

| Role | Election | Winner |
|---|---|---|
| IGMPv2/v3 Querier | Lowest IP address | R1 (10.10.10.1) |
| PIM DR | Highest priority, then highest IP | R2 (10.10.10.2) |

Therefore:

```text
R1 → IGMP Querier
R2 → PIM DR
```

The **IGMP querier** maintains receiver membership information by sending Queries.

The **PIM DR** is responsible for certain multicast routing functions, including sending PIM Joins toward the RP or source and registering directly connected sources in PIM Sparse Mode.

> A router does not have to be the PIM DR to act as the IGMPv2/v3 querier.

## Non-Querier Operation

A router that loses the querier election enters the **Non-Querier** state.

It stops sending periodic General Queries but continues processing IGMP messages and maintaining membership information.

It also maintains an **Other Querier Present Timer**.

Each time it receives a General Query from the active querier, the timer is refreshed.

```text
R1 sends General Query
          |
          v
R2 receives Query
          |
          v
Other Querier Present Timer resets
          |
          v
R2 remains Non-Querier
```

If the active querier stops sending Queries, the timer eventually expires.

The non-querier can then take over the querier role.

## Querier Failure and Takeover

Suppose R1 is the active querier:

```text
R1 = 10.10.10.1 → Querier
R2 = 10.10.10.2 → Non-Querier
```

R1 sends General Queries periodically.

If R1 fails:

```text
R1 fails
    |
    v
R2 stops receiving R1's Queries
    |
    v
Other Querier Present Timer expires
    |
    v
R2 becomes Querier
    |
    v
R2 begins sending General Queries
```

If R1 subsequently recovers, it begins sending Queries again.

Because R1 has the lower IP address, R2 recognizes it as the preferred querier and returns to the Non-Querier state.

> IGMPv2/v3 querier election is preemptive. If the current querier fails, another router takes over after the Other Querier Present Timer expires. When a router with a lower IP address returns and sends a General Query, the current querier immediately relinquishes the role.

## Other Querier Present Interval

RFC 2236 defines the **Other Querier Present Interval** as:

```text
Other Querier Present Interval
= (Robustness Variable × Query Interval)
  + (Query Response Interval / 2)
```

Using the RFC defaults:

```text
Robustness Variable     = 2
Query Interval          = 125 seconds
Query Response Interval = 10 seconds

(2 × 125) + (10 / 2)
= 255 seconds
```

However, **IOS XE uses different default values**.

On Cisco IOS XE, the default IGMP Query Interval is typically `60 seconds`, and the default configured querier timeout is twice the Query Interval:

```text
Querier Timeout = 2 × 60
                = 120 seconds
```

This timer can be configured on an interface:

```text
R2(config)# interface Ethernet0/0
R2(config-if)# ip igmp querier-timeout 180
```

R2 now uses a configured querier timeout of `180 seconds`.

**Important:** The operational timeout can also be affected by the configured Robustness Variable.

For example, with:

```text
ip igmp robustness-variable 3
```

the operational timeout increases from `120` to `180 seconds`, even though the timeout itself hasn't been configured.

Both values can be observed with:

```
R3(config-if)# ip igmp robustness-variable 3   
[Ethernet0/1] Warning: Please be aware of the other IGMP parameters that may
be affected by this configuration change.
R3(config-if)#do sh ip igmp int e0/1          
!
  IGMP robustness-variable is 3
  IGMP querier timeout is 180 seconds
  IGMP configured querier timeout is 120 seconds
!
```

Changing querier timeout values should be done carefully. Setting the timeout too low relative to the Query Interval can cause unnecessary querier takeovers.

Setting it too high can delay querier failure detection and takeover, leaving the subnet without an active querier.

## IGMP Querier Timers

Several IGMP timers affect querier operation.

| Parameter | Typical IOS XE Default | Purpose |
|---|---:|---|
| Query Interval | 60 seconds | Interval between General Queries |
| Query Response Interval | 10 seconds | Maximum time hosts can delay responses |
| Robustness Variable | 2 | Tolerance for lost IGMP messages |
| Configured Querier Timeout | 120 seconds | Timeout before a non-querier may take over |
| Last Member Query Interval | 1000 ms | Interval between specific Queries during leave processing |
| Last Member Query Count | 2 | Number of specific Queries during leave processing |

These timers have already been introduced in the IGMPv2 and IGMPv3 Query sections.

The key distinction is between:

```text
Query Interval
→ How frequently the querier sends Queries

Query Response Interval
→ How long receivers have to respond

Querier Timeout
→ How long a non-querier waits before takeover

Group Membership Timeout
→ How long receiver membership information is retained
```

The **querier timeout** and **group membership timeout** are separate mechanisms.

A router can become the new querier without deleting its existing group membership information.


## IGMPv3 Querier Parameters

IGMPv3 uses the same lowest-IP-address election mechanism as IGMPv2 but adds fields that allow non-querier routers to synchronize their operating parameters with the elected querier.

Two important fields are:

- **QRV (Querier's Robustness Variable)** — Advertises the querier's Robustness Variable.
- **QQIC (Querier's Query Interval Code)** — Advertises the querier's Query Interval.

### Non-Querier Synchronization

When a non-querier receives an IGMPv3 Query, it adopts the advertised values:

- **QRV** — The non-querier uses the received Robustness Variable instead of its own locally configured value, provided QRV is nonzero.
- **QQIC** — The non-querier uses the received Query Interval instead of its own local value, provided the advertised interval is nonzero.

For example:

```text
                    R3               R4
                 10.3.4.3         10.3.4.4
                  Querier        Non-Querier
                     \               /
                      \             /
                            SW3
```

Suppose the routers are initially configured with different parameters:

| Parameter | R3 | R4 |
|---|---:|---:|
| Robustness Variable | 3 | 2 |
| Query Interval | 45 seconds | 60 seconds |

R3 is elected querier and advertises:

```text
QRV  = 3
QQIC = 45
```

When R4 receives the Query, RFC 9776 says it should adopt these operational values:

```text
R4:

Robustness Variable = 3
Query Interval      = 45 seconds
```

R4 uses these learned parameters to calculate timers such as:

- **Other Querier Present Interval** — Determines when R4 can take over as querier if R3 stops sending Queries.
- **Group Membership Interval** — Determines when group or source membership state expires without being refreshed.

RFC 9776 defines:

```text
Other Querier Present Interval
= (Robustness Variable × Query Interval)
  + (Query Response Interval / 2)
```

However, IOS XE uses:

```text
Other Querier Present Interval
= (Robustness Variable × Query Interval)
```

Output on R4:

```
R4# show ip igmp int e0/1
Ethernet0/1 is up, line protocol is up
  Internet address is 10.3.4.4/24
  IGMP is enabled on interface
  Current IGMP host version is 3
  Current IGMP router version is 3
  IGMP query interval is 45 seconds
  IGMP configured query interval is 60 seconds
  IGMP robustness-variable is 2
  IGMP querier timeout is 90 seconds
  IGMP configured querier timeout is 120 seconds
  IGMP max query response time is 10 seconds
  Last member query count is 2
  Last member query response interval is 1000 ms
  Inbound IGMP access group is not set
  IGMP activity: 2 joins, 0 leaves
  Multicast routing is enabled on interface
  Multicast TTL threshold is 0
  Multicast designated router (DR) is 10.3.4.4 (this system)
  IGMP querying router is 10.3.4.3  
  No multicast groups joined by this system
```

There are a couple of takeaways:

```
  IGMP query interval is 45 seconds
  IGMP configured query interval is 60 seconds
```

In the above output, we can see that R4 has adopted R3's query interval of 45 seconds. However:

```
  IGMP robustness-variable is 2
```

As this output shows, it does **not** adopt the QRV as its own.

Let's look at R4's Other Querier Present Interval:

```
  IGMP querier timeout is 90 seconds
  IGMP configured querier timeout is 120 seconds
```

It has used the QQIC of `45 seconds` with its own RV of `2` to calculate `90 seconds` as the operational Other Querier Present Interval.

So at least on this IOS XE platform (IOL in CML), a non-Querier adopts the QQIC, but does **not** adopt the QRV.

This may or may not apply to other platforms and IOS XE versions. I can't find any clear Cisco documentation on it.

> R2 does not send periodic General Queries while it remains a non-Querier. It uses the learned Query Interval for timer calculations, not to transmit Queries itself.

### Zero Values

There are special rules for zero values:

- **QRV = 0** — The receiving router uses its own locally configured Robustness Variable, or the default of 2 if not configured.
- **QQIC = 0** — The receiving router uses the protocol's default Query Interval (60 seconds).

QRV is limited to 3 bits. If the querier's Robustness Variable exceeds 7, it advertises QRV = 0.

### IGMPv2 vs IGMPv3

IGMPv2 does not advertise QRV or QQIC, so routers must rely on locally configured parameters.

> In IOS XE, an IGMPv2 router can still learn and adopt the QQIC advertised in an IGMPv3 Query.

## Basic IOS XE Configuration

On a multicast router, IGMP is enabled on an interface when PIM is enabled.

For example:

```text
R1(config)# ip multicast-routing
R1(config)# interface Ethernet0/0
R1(config-if)# ip address 10.10.10.1 255.255.255.0
R1(config-if)# ip pim sparse-mode
```

There is no IGMP Querier priority value; the lowest IP address always wins (in IGMPv2 and IGMPv3).

Therefore, the only way to control which router becomes the Querier is through their IP addresses.

## Configuring the Query Interval

The default IOS XE Query Interval is `60 seconds`.

To change it to `30 seconds`:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip igmp query-interval 30
```

R1, when acting as the querier, now sends General Queries every `30 seconds`.

Verify:

```text
R1# show ip igmp interface Ethernet0/0
  IGMP query interval is 30 seconds
  IGMP configured query interval is 30 seconds
```

Shorter Query Intervals increase IGMP control traffic but allow membership information to be refreshed more frequently.

Longer intervals reduce IGMP traffic but can increase membership expiration and querier failure detection times.

## Configuring the Maximum Query Response Time

The maximum response time advertised in General Queries can be configured with:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip igmp query-max-response-time 5
```

This sets the maximum host response delay to `5 seconds`.

Verify:

```text
R1# show ip igmp interface Ethernet0/0
  IGMP max query response time is 5 seconds
```

Reducing this value causes hosts to respond within a shorter window, but concentrates IGMP Report traffic into a shorter period.

> The Query Response Interval should be shorter than the Query Interval.

## Configuring the Robustness Variable

The default Robustness Variable is `2`.

It can be modified with:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip igmp robustness-variable 3
```

This increases tolerance for lost IGMP messages.

The Robustness Variable can affect multiple parameters, including the Last Member Query Count and querier timeout.

Verify:

```text
R1# show ip igmp interface Ethernet0/0
  IGMP robustness-variable is 3
```

## Configuring Last Member Queries

The querier also generates Group-Specific Queries (IGMPv2/v3) and Group-and-Source-Specific Queries (IGMPv3) to verify whether receivers remain interested in a group or particular sources.

Two commands control the normal last-member verification process:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip igmp last-member-query-interval 500
R1(config-if)# ip igmp last-member-query-count 3
```

This configures:

```text
Last Member Query Interval = 500 ms
Last Member Query Count    = 3
```

The approximate verification period is:

```text
3 × 500 ms = 1500 ms
```

The complete leave procedure is covered in **IGMPv2 Leave Process** and **IGMPv3 State Changes**.

## Verifying the IGMP Querier

The primary IOS XE verification command is:

```text
R1# show ip igmp interface Ethernet0/0
```

Example output:

```text
Ethernet0/0 is up, line protocol is up
  Internet address is 10.10.10.1/24
  IGMP is enabled on interface
  Current IGMP host version is 2
  Current IGMP router version is 2
  IGMP query interval is 60 seconds
  IGMP configured query interval is 60 seconds
  IGMP robustness-variable is 2
  IGMP querier timeout is 120 seconds
  IGMP configured querier timeout is 120 seconds
  IGMP max query response time is 10 seconds
  Last member query count is 2
  Last member query response interval is 1000 ms
  Inbound IGMP access group is not set
  Multicast routing is enabled on interface
  Multicast designated router (DR) is 10.10.10.2
  IGMP querying router is 10.10.10.1 (this system)
  No multicast groups joined by this system
```

Notice the two separate fields:

```text
Multicast designated router (DR) is 10.10.10.2

IGMP querying router is 10.10.10.1 (this system)
```

R1 is the IGMP querier, but R2 is the PIM DR.

This confirms that the two roles are independent.

On R2, the output would instead identify R1 as the IGMP querier:

```text
IGMP querying router is 10.10.10.1
```

without `(this system)`.

## Key Points

- IGMPv1 has no independent querier election; the PIM DR performs the role.
- IGMPv2 and IGMPv3 elect the router with the **lowest IPv4 address**.
- Non-queriers continue processing IGMP messages and maintain a timer to detect querier failure.
- IOS XE uses a **60-second Query Interval** and **120-second configured Querier Timeout** by default.
- Querier election is independent of PIM DR election in IGMPv2/v3.
- The primary verification command is `show ip igmp interface`.
