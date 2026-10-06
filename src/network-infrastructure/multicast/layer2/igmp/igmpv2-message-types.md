# IGMPv2 Message Types

IGMPv2 uses a small set of messages to communicate multicast group membership between hosts and routers.

The primary IGMPv2 message types are:

| Type | Message |
|---|---|
| `0x11` | Membership Query |
| `0x16` | IGMPv2 Membership Report |
| `0x17` | Leave Group |

IGMPv2 also recognizes the IGMPv1 Membership Report (`0x12`) for backward compatibility.

## IGMPv2 Message Format

IGMPv2 messages use the following 8-byte base format:

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      Type     | Max Resp Time |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Group Address                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The fields are:

| Field | Size | Purpose |
|---|---:|---|
| **Type** | 8 bits | Identifies the IGMP message type |
| **Max Resp Time** | 8 bits | Maximum time hosts may wait before responding to a Query |
| **Checksum** | 16 bits | Error detection for the IGMP message |
| **Group Address** | 32 bits | Multicast group associated with the message |

## Type

The **Type** field identifies the IGMP message.

```text
0x11 = Membership Query
0x12 = IGMPv1 Membership Report
0x16 = IGMPv2 Membership Report
0x17 = Leave Group
```

### Membership Query — `0x11`

Both **General Queries** and **Group-Specific Queries** use Type `0x11`.

They are distinguished by the **Group Address** field:

```text
General Query
Type:          0x11
Group Address: 0.0.0.0
```

```text
Group-Specific Query
Type:          0x11
Group Address: Multicast group being queried
```

For example:

```text
Type:          0x11
Group Address: 239.1.1.1
```

indicates a Group-Specific Query for `239.1.1.1`.

The details of General and Group-Specific Queries are covered on the **IGMPv2 Queries** page.

### IGMPv2 Membership Report — `0x16`

An IGMPv2 host sends a **Membership Report** to indicate that it wants to receive a multicast group.

```text
Type:          0x16
Group Address: Multicast group being reported
```

For example:

```text
Type:          0x16
Group Address: 239.1.1.1
```

The IP destination is also the multicast group being reported:

```text
Destination: 239.1.1.1
```

The Membership Report process is covered on the **IGMPv2 Membership Reports** page.

### Leave Group — `0x17`

An IGMPv2 host can explicitly indicate that it is leaving a multicast group by sending a **Leave Group** message.

```text
Type:          0x17
Group Address: Multicast group being left
```

For example:

```text
Type:          0x17
Group Address: 239.1.1.1
```

However, unlike a Membership Report, the Leave is sent to:

```text
224.0.0.2
```

`224.0.0.2` is the **All Routers** multicast address.

The complete Leave process is covered on the **IGMPv2 Leave Process** page.

### IGMPv1 Membership Report — `0x12`

IGMPv2 also recognizes:

```text
0x12 = IGMPv1 Membership Report
```

This allows IGMPv2 routers to operate with IGMPv1 hosts on the same subnet.

## Max Resp Time

The **Max Resp Time** field is used only in Membership Queries.

It tells hosts the maximum amount of time they may delay before responding with a Membership Report.

The value is encoded in units of **1/10 second**.

For example:

```text
Max Resp Time = 100
```

means:

```text
100 × 0.1 seconds = 10 seconds
```

A host chooses a random delay up to this value before sending its Report.

For messages other than Queries, the sender sets Max Resp Time to `0` and receivers ignore the field.

The Max Resp Time field is one of the important improvements IGMPv2 introduced over IGMPv1 because it allows the querier to control response timing and reduce leave latency.

## Checksum

The **Checksum** is a 16-bit one's-complement checksum covering the entire IGMP message.

When calculating the checksum:

```text
Checksum field = 0
```

The checksum is then calculated and inserted into the field.

A receiver verifies the checksum before processing the IGMP message.

## Group Address

The meaning of the **Group Address** field depends on the message type.

| Message | Group Address |
|---|---|
| General Query | `0.0.0.0` |
| Group-Specific Query | Group being queried |
| Membership Report | Group being reported |
| Leave Group | Group being left |

This field is particularly important because **General Queries and Group-Specific Queries use the same Type value (`0x11`)**.

The Group Address field tells the receiver which kind of Query it is.

## Message Destinations

The IP destination varies by message type:

| Message | IP Destination | Group Address Field |
|---|---|---|
| General Query | `224.0.0.1` | `0.0.0.0` |
| Group-Specific Query | Group being queried | Group being queried |
| Membership Report | Group being reported | Group being reported |
| Leave Group | `224.0.0.2` | Group being left |

For example, a host leaving `239.1.1.1` sends:

```text
IP Destination: 224.0.0.2
IGMP Type:      0x17
Group Address:  239.1.1.1
```

Notice that the **IP destination and IGMP Group Address fields do not necessarily contain the same address**.

## IP Encapsulation

IGMP is carried directly inside IPv4 rather than TCP or UDP.

```text
IPv4
└── IGMP
```

The IPv4 Protocol field is:

```text
Protocol = 2
```

IGMPv2 messages are sent with:

```text
TTL = 1
```

This keeps the messages on the local subnet.

IGMPv2 messages also include the **IP Router Alert option**, indicating that routers should examine the packet even when the packet is not addressed directly to the router.

> **IGMPv2 and IGMPv3** messages include the Router Alert option. **IGMPv1** messages do not.

## Message Length

The standard IGMPv2 host-router message format is `8 bytes`.

However, an IGMP message can technically contain additional data after the first 8 bytes.

For a recognized IGMPv2 message type, an IGMPv2 implementation processes the first 8 bytes and ignores additional data.

The **checksum still covers the entire IP payload**, including any additional bytes.

## Summary

A useful IGMPv2 packet-identification summary is:

```text
0x11 + Group 0.0.0.0    = General Query
0x11 + Group 239.x.x.x  = Group-Specific Query
0x16                    = IGMPv2 Membership Report
0x17                    = Leave Group
```