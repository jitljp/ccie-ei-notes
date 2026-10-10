# MLDv1

**Multicast Listener Discovery version 1 (MLDv1)** is defined in [RFC 2710](https://www.rfc-editor.org/rfc/rfc2710.html).

MLDv1 is the IPv6 equivalent of **IGMPv2** and uses essentially the same membership management mechanisms:

- General and Group-Specific Queries
- Membership Reports and report suppression
- Explicit leave notification using Done messages
- Querier election
- Last Listener Queries to verify whether receivers remain

The major differences are that **MLDv1 uses ICMPv6**, rather than IGMP, and operates with IPv6 multicast addresses.

## MLDv1 Message Format

MLDv1 messages are carried within **ICMPv6 (Next Header 58)**.

All three MLDv1 message types use the same 24-byte format:

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      Type     |      Code     |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      Maximum Response Delay   |           Reserved            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                                                               |
+                       Multicast Address                       +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| Field | Size | Description |
|---|---|---|
| Type | 8 bits | Identifies the MLD message type |
| Code | 8 bits | Set to 0 |
| Checksum | 16 bits | ICMPv6 checksum |
| Maximum Response Delay | 16 bits | Maximum time allowed for a response, in milliseconds |
| Reserved | 16 bits | Set to 0 |
| Multicast Address | 128 bits | Multicast group being queried, joined, or left |

The **Maximum Response Delay** field differs from IGMPv2:

- **IGMPv2:** Units of 1/10 second.
- **MLDv1:** Units of 1 millisecond.

For example, an MLDv1 Query with a Maximum Response Delay of `10000` allows hosts up to **10 seconds** to respond.

## MLDv1 Message Types

MLDv1 defines three ICMPv6 message types:

| Message | ICMPv6 Type | IPv6 Destination |
|---|---:|---|
| Multicast Listener Query | 130 | `ff02::1` (General) or the group being queried |
| Multicast Listener Report | 131 | Group being reported |
| Multicast Listener Done | 132 | `ff02::2` |

### Multicast Listener Queries

MLDv1 supports two Query types:

**General Query:**

```text
ICMPv6 Type:       130
IPv6 Destination:  ff02::1
Multicast Address: ::
```

**Multicast-Address-Specific Query:**

```text
ICMPv6 Type:       130
IPv6 Destination:  ff15::1234
Multicast Address: ff15::1234
```

The Multicast Address field distinguishes a General Query from a Group-Specific Query.

Like IGMPv2, receivers select random response delays based on the Query's Maximum Response Delay.

### Multicast Listener Reports

A host sends a **Multicast Listener Report** when joining a group or responding to a Query.

For example:

```text
ICMPv6 Type:       131
IPv6 Destination:  ff15::1234
Multicast Address: ff15::1234
```

MLDv1 also uses **report suppression**.

If a host hears another host report the same group while its response timer is running, it suppresses its own Report.

### Multicast Listener Done

A host can send a **Multicast Listener Done** message when leaving a group.

For example:

```text
ICMPv6 Type:       132
IPv6 Destination:  ff02::2
Multicast Address: ff15::1234
```

`ff02::2` is the **All Routers** multicast address.

As in IGMPv2, a host that knows another listener recently reported the group may leave without sending a Done message.

When the querier receives a Done message, it sends **Group-Specific Queries** to determine whether any listeners remain.

If no Reports are received, the router removes the membership state after the Last Listener Query procedure completes.

## IPv6-Specific MLD Requirements

MLDv1 has several important IPv6-specific requirements.

### Link-Local Source Address

MLD Queries must use a **link-local IPv6 source address**.

For example:

```text
Source:      fe80::1
Destination: ff02::1
ICMPv6:     MLD General Query
```

MLD Reports and Done messages normally also use link-local source addresses.

However, [RFC 3590](https://www.rfc-editor.org/rfc/rfc3590.html) permits the unspecified address (`::`) as the source of Reports and Done messages when a valid link-local address is not yet available, such as during Duplicate Address Detection (DAD).

### Hop Limit and Router Alert

MLDv1 packets use:

```text
IPv6 Hop Limit: 1
Hop-by-Hop Options Header: Router Alert
Router Alert Value: 0 (MLD)
```

The **Router Alert** option indicates that routers should examine the packet even if they are not ordinary listeners for its multicast destination.

MLD operates only on the local link and does not communicate multicast membership information between routed network segments.

## MLDv1 Querier Election

As in IGMPv2, only one multicast router normally sends periodic MLD Queries on a link.

The router with the **lowest link-local IPv6 address** becomes the querier.

For example:

```text
R1: fe80::1  ← Querier
R2: fe80::2
R3: fe80::3
```

This differs from IGMPv2, which compares IPv4 interface addresses.

The **MLD querier election is separate from the PIM Designated Router (DR) election**.

## MLDv1 Timers

MLDv1 uses essentially the same timers as IGMPv2.

RFC 2710 defines the following defaults:

| Timer / Parameter | Default |
|---|---:|
| Robustness Variable | 2 |
| Query Interval | 125 seconds |
| Query Response Interval | 10 seconds |
| Startup Query Interval | 1/4 Query Interval |
| Startup Query Count | 2 |
| Last Listener Query Interval | 1 second |
| Last Listener Query Count | 2 |

The default **Multicast Listener Interval** is:

```text
(Robustness Variable × Query Interval)
+ Query Response Interval

= (2 × 125) + 10
= 260 seconds
```

The default **Other Querier Present Interval** is:

```text
(Robustness Variable × Query Interval)
+ (Query Response Interval / 2)

= (2 × 125) + 5
= 255 seconds
```

With the default Last Listener Query Count and Interval, the querier can determine that the final listener has left in approximately **2 seconds**.

## Cisco IOS XE Configuration

Cisco IOS XE supports MLDv1 and MLDv2, with MLDv2 normally used for router-side processing and backward compatibility with MLDv1 receivers.

Enable IPv6 multicast routing globally:

```text
R1(config)# ipv6 multicast-routing
```

**MLDv2** is the default, but you can enable MLDv1 with:

```
R1(config-if)# ipv6 mld version 1
```

However, in modern IOS XE you will get a warning message:

```
*Oct 10 05:42:11.926: %PARSER-5-HIDDEN: Warning!!! ' ipv6 mld version 1' is a hidden command. Use of this command is not recommended/supported and will be removed in future.
```

### MLD Query Timers

The default MLD Query Interval on IOS XE is **125 seconds**, matching RFC 2710.

It can be changed on an interface:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 mld query-interval 60
```

The default Query Maximum Response Time is **10 seconds**.

To change it to 5 seconds:

```text
R1(config-if)# ipv6 mld query-max-response-time 5
```

Unlike the MLDv1 message field, which uses milliseconds, the IOS XE command accepts a value in **seconds**.

The Other Querier Present timeout can also be configured:

```text
R1(config-if)# ipv6 mld query-timeout 130
```

### MLD Group Membership

A router can be configured to join a multicast group on an interface:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 mld join-group ff15::1234
```

This causes the router itself to become a listener for the multicast group.

To create multicast forwarding interest on an interface without making the router itself a local receiver, use:

```text
R1(config-if)# ipv6 mld static-group ff15::1234
```

### Disabling MLD Router-Side Processing

By default, when IPv6 multicast routing is enabled, the router performs **MLD router-side processing** on its interfaces.

This includes:

- Sending MLD Queries when acting as the querier
- Processing MLD Reports and Done messages
- Maintaining multicast listener membership state
- Forwarding multicast traffic onto interfaces with interested receivers

To disable these functions on an interface:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# no ipv6 mld router
```

This has three main effects:

1. **Disables MLD router-side processing**: The router stops sending MLD Queries and tracking multicast group memberships on the interface.
2. **Disables multicast routing toward the interface**: The router no longer forwards routed IPv6 multicast traffic out the interface.
3. **Disables static MLD group configuration**: The `ipv6 mld static-group` feature cannot be used to establish multicast forwarding toward that interface.

However, the router can still function as an **MLD host** if `ipv6 mld join-group` is configured:

```text
R1(config-if)# no ipv6 mld router
R1(config-if)# ipv6 mld join-group ff15::1234
```

In this case, R1 does not act as an MLD router on Ethernet0/0, but it can still report its own membership in `ff15::1234` when another MLD querier sends a Query.

To restore router-side MLD processing:

```text
R1(config-if)# ipv6 mld router
```

> **Important:** This command applies only to the specified interface. It does not globally disable IPv6 multicast routing or Layer 2 MLD snooping.

## Verification

Display MLD information for an interface:

```text
R1# show ipv6 mld interface e0/1
Ethernet0/1 is up, line protocol is up
  Internet address is FE80::1/10
  MLD is enabled on interface
  Current MLD version is 1
  MLD query robustness value is 2
  MLD query interval is 125 seconds
  MLD querier timeout is 255 seconds
  MLD max query response time is 10 seconds
  Last member query response interval is 1 seconds
  MLD activity: 10 joins, 5 leaves
  MLD querying router is FE80::1 (this system)
```

Display multicast listener memberships:

```text
R1# show ipv6 mld groups 
MLD Connected Group Membership
Group Address                           Interface                                             Uptime    Expires
FF15::1234                              Ethernet0/1                                          00:00:21  00:03:58
```

Display MLD message counters:

```text
R1# show ipv6 mld traffic 
MLD Traffic Counters
Elapsed time since counters cleared: 00:18:32

                              Received     Sent
Valid MLD Packets               39          76        
Queries                         0           11        
Reports                         39          65        
Leaves                          0           0         
Mtrace packets                  0           0         

Errors:
Malformed Packets                           0         
Martian source                              0         
Non link-local source                       0         
Hop limit is not equal to 1                 0   
```

## Key Points

- MLDv1 is the IPv6 equivalent of **IGMPv2**, defined in RFC 2710.
- It uses **ICMPv6 Types 130 (Query), 131 (Report), and 132 (Done)**.
- Maximum Response Delay is encoded in **milliseconds**, unlike IGMPv2's 1/10-second units.
- MLD packets use a Hop Limit of 1 and the IPv6 Router Alert option.
- MLD querier election uses the lowest link-local IPv6 address, like in IGMPv2/v3.
- MLDv1 supports group membership only, not source-specific filtering.
- **MLDv2** adds source filtering and SSM support.