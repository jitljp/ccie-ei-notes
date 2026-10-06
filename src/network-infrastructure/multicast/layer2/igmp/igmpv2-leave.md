# IGMPv2 Leave Process

IGMPv2 improves on IGMPv1 by allowing a host to explicitly indicate that it is leaving a multicast group.

This allows the router to determine much more quickly whether any receivers remain on the subnet.

## Leave Group Message

The IGMPv2 **Leave Group** message uses Type `0x17` and the standard IGMPv2 message format:

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      Type     | Max Resp Time |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Group Address                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

For a Leave Group message:

```text
Type:           0x17
Max Resp Time:  0
Group Address:  Group being left
```

For example, when leaving `239.1.1.1`:

```text
IP Destination:     224.0.0.2
IGMP Type:          0x17
IGMP Group Address: 239.1.1.1
TTL:                1
```

`224.0.0.2` is the **All Routers** multicast address.

Therefore, unlike a Membership Report:

```text
Membership Report → sent to the group
Leave Group       → sent to 224.0.0.2
```

The Leave is sent to the routers because the other receivers do not need to process it.

> For backward compatibility with early IGMPv2 implementations, routers should also accept Leave messages sent directly to the group being left. RFC-compliant hosts send them to `224.0.0.2`.

## When Does a Host Send a Leave?

An IGMPv2 host keeps track of whether it was the **last reporter** for each multicast group.

If the host was the last host to send a Membership Report for the group, it should send a Leave Group message when it leaves.

```text
Host was last reporter
        ↓
Host leaves group
        ↓
Send Leave Group
```

If the host heard another host report the group more recently, it may leave silently:

```text
PC1 sends Report
        ↓
PC2 hears Report
        ↓
PC2 clears its last-reporter flag
        ↓
PC2 leaves
        ↓
No Leave required
```

PC2 already knows that another receiver exists on the subnet, so triggering the leave process would be unnecessary.

This behavior is an optimization. A host that does not track the last-reporter state may simply send a Leave whenever it leaves a group.

> **Last reporter does not mean last remaining receiver in the LAN.** It only means that the host sent the most recently heard Membership Report.

## Querier Processing

When the **IGMP querier** receives a Leave Group message for an active group, it does not immediately remove the group.

Another receiver may still exist.

Instead, it sends a series of **Group-Specific Queries** to the group being left.

For example:

```text
PC1 -----------------------> 224.0.0.2
        Leave Group
        Group: 239.1.1.1

R1 ------------------------> 239.1.1.1
        Group-Specific Query

R1 ------------------------> 239.1.1.1
        Group-Specific Query
```

The number of Queries is the **Last Member Query Count (LMQC)**.

By default:

```text
Last Member Query Count = Robustness Variable = 2
```

The interval between them is the **Last Member Query Interval (LMQI)**:

```text
Default LMQI = 1 second
```

The Group-Specific Query also carries the LMQI as its **Max Resp Time**.

With the defaults:

```text
LMQC = 2
LMQI = 1 second
```

the router checks for remaining receivers over approximately:

```text
2 × 1 second = 2 seconds
```

This is the **Last Member Query Time**.

## If Another Receiver Remains

Suppose PC1 and PC2 have both joined the group `239.1.1.1`.

PC1 sends the Leave:

```text
PC1 -----------------------> R1
        Leave 239.1.1.1

R1 ------------------------> 239.1.1.1
        Group-Specific Query
```

PC2 is still a member, so it responds:

```text
PC2 -----------------------> 239.1.1.1
        Membership Report
```

The Report confirms that the group still has a receiver.

R1 retains the group membership and continues forwarding multicast traffic onto the subnet.

## If No Receiver Remains

If no Report is received after the response time for the final Group-Specific Query expires:

```text
Leave received
      ↓
Group-Specific Query
      ↓
Group-Specific Query
      ↓
No Membership Report
      ↓
Remove group membership
      ↓
Stop forwarding the group onto the subnet
```

The router can therefore remove the group in a few seconds instead of waiting for the normal membership timeout.

This is the major improvement over IGMPv1.

## Router State During Leave Processing

RFC 2236 refers to this temporary state as **Checking Membership**.

Conceptually:

```text
Members Present
      |
      | Leave received
      v
Checking Membership
      |
      +---- Report received ----> Members Present
      |
      +---- Timer expires ------> No Members Present
```

When checking membership, the group membership timer is shortened to:

```text
Last Member Query Count × Last Member Query Interval
```

rather than waiting for the normal Group Membership Interval.

## Querier vs Non-Querier Behavior

Only the **querier** acts directly on a Leave Group message.

RFC 2236 specifies:

```text
Querier     → Processes Leave and sends Group-Specific Queries
Non-Querier → Ignores the Leave
```

However, the non-querier hears the Group-Specific Queries sent by the querier.

When it receives one, it also shortens its local membership timer so that its state remains synchronized with the querier.

For a non-querier:

```text
Membership timer
= Last Member Query Count × Max Resp Time
```

where the Max Resp Time comes from the received Group-Specific Query.

### Querier Change During Leave Processing

If the current querier begins the Last Member Query process and then would otherwise lose the querier election, it continues sending the remaining Group-Specific Queries.

This prevents the leave procedure from being interrupted halfway through.

## IGMPv1 Compatibility

If the router has detected an **IGMPv1 host** for a particular group, it must ignore IGMPv2 Leave Group messages for that group.

The reason is that **IGMPv1 hosts do not support Leave messages**.

For example:

```text
PC1 = IGMPv2
PC2 = IGMPv1
```

If PC1 leaves and sends:

```text
Leave 239.1.1.1
```

PC2 might still be receiving the group.

The router therefore cannot safely use the normal IGMPv2 fast leave process while an IGMPv1 receiver is known to exist.

## IOS XE Leave Timers

The most important IOS XE leave-related timer is the **Last Member Query Interval**.

The default is `1000 ms` (1 second):

```
R3# show ip igmp interface e0/1 | i Last member
  Last member query count is 2
  Last member query response interval is 1000 ms
```

It can be changed with:

```text
R3(config-if)# ip igmp last-member-query-interval 500

R3# show ip igmp interface e0/1 | i Last member
  Last member query count is 2
  Last member query response interval is 500 ms
```

This reduces the interval to `500 ms`, and therefore reduces leave latency.

The **Last Member Query Count** defaults to the Robustness Variable.

Therefore:

```text
R1(config-if)# ip igmp robustness-variable 3
```

also changes the default Last Member Query Count:

```text
Robustness Variable = 3
Last Member Query Count = 3
```

Manually configured dependent values override values that would otherwise be derived from the Robustness Variable.

As the following example shows, a manually configured LMQC overrides the RV:

```text
R3(config-if)# ip igmp last-member-query-count 4

R3# show ip igmp interface e0/1
Ethernet0/1 is up, line protocol is up
  Internet address is 10.3.4.3/24
  IGMP is enabled on interface
  Current IGMP host version is 2
  Current IGMP router version is 2
  IGMP query interval is 60 seconds
  IGMP configured query interval is 60 seconds
  IGMP robustness-variable is 3        <--- RV = 4
  IGMP querier timeout is 180 seconds
  IGMP configured querier timeout is 120 seconds
  IGMP max query response time is 10 seconds
  Last member query count is 4        <--- LMQC = 4
  Last member query response interval is 1000 ms
  Inbound IGMP access group is not set
  IGMP activity: 3 joins, 2 leaves
  Multicast routing is enabled on interface
  Multicast TTL threshold is 0
  Multicast designated router (DR) is 10.3.4.4  
  IGMP querying router is 10.3.4.3 (this system)
  No multicast groups joined by this system
```

## Immediate Leave

IOS XE can also be configured to process selected IGMPv2 Leaves **immediately**, without performing the normal Group-Specific Query process.

`ip igmp immediate-leave group-list` can be configured **globally** or **per interface**.

Globally:

```text
R1(config)# access-list 10 permit 239.1.1.0 0.0.0.255
R1(config)# ip igmp immediate-leave group-list 10
```

Or on a specific interface:

```text
R1(config)# access-list 10 permit 239.1.1.0 0.0.0.255
R1(config)# interface Ethernet0/0
R1(config-if)# ip igmp immediate-leave group-list 10
```

The global configuration applies Immediate Leave generally, while the interface configuration limits it to the selected interface.

> Configure Immediate Leave either globally or at the interface level, not in both modes at the same time.

For groups permitted by the ACL:

```text
Leave received
      ↓
Immediately remove membership
      ↓
No Group-Specific Queries
```

Groups not permitted by the ACL use the normal IGMPv2 leave process.

Immediate Leave reduces leave latency but must be used carefully.

If multiple receivers exist behind the same interface, removing the group immediately when one receiver leaves can interrupt multicast traffic for the remaining receivers.

## Leave Process Summary

The normal IGMPv2 leave process is:

```text
          Host leaves group
                   ↓
 If last reporter, send Leave to 224.0.0.2
                   ↓
   Querier sends Group-Specific Queries
                   ↓
       ┌─────────────────────────┐
       │                         │
  Report received           No Report
       │                         │
       v                         v
 Keep membership          Remove membership
 Continue forwarding      Stop forwarding
```

With the default values:

```text
Last Member Query Count    = 2
Last Member Query Interval = 1 second
```

IGMPv2 can usually determine that the final receiver has left in only a few seconds, rather than waiting for membership timeout (up to 3 minutes) used by IGMPv1.
