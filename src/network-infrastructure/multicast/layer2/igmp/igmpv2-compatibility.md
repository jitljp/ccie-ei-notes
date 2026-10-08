# IGMPv1/IGMPv2 Compatibility

IGMPv2 was designed to coexist with IGMPv1 devices.

Compatibility has two distinct cases:

- An **IGMPv2 host** operating with an **IGMPv1 router**
- An **IGMPv2 router** operating with **IGMPv1 hosts**

These mechanisms work differently and maintain compatibility state at different scopes.

## Identifying an IGMPv1 Query

IGMPv1 and IGMPv2 Queries both use:

```text
Type = 0x11
```

An IGMPv1 General Query can be identified by its **Max Resp Time** field:

```text
IGMPv1 Query
Type:          0x11
Max Resp Time: 0
Group Address: 0.0.0.0
```

An IGMPv2 host interprets:

```text
Max Resp Time = 0
```

in an IGMPv1 Query as:

```text
Max Resp Time = 10 seconds
```

rather than as an immediate-response requirement.

> There is no version field in the IGMP message format.
>
> IGMPv1 Queries are identified by `MRT = 0`.

## IGMPv2 Host with an IGMPv1 Querier

An IGMPv2 host maintains compatibility state **per interface**.

When it receives an IGMPv1 Query, it enters an **IGMPv1 Router Present** state for that interface.

```text
IGMPv1 Query received
        ↓
Max Resp Time = 0
        ↓
IGMPv1 Router Present
```

While an IGMPv1 router is considered present, the IGMPv2 host sends **IGMPv1 Membership Reports**:

```text
Type = 0x12
```

rather than IGMPv2 Reports:

```text
Type = 0x16
```

This applies to both:

- Reports sent in response to Queries
- Unsolicited Reports when joining a group

The reason is simple: an IGMPv1 router may not recognize an IGMPv2 Membership Report.

As the following `debug ip igmp` output shows, an IOS XE router using IGMPv1 discards IGMPv2 Membership Reports:

```
*Oct  6 23:13:42.758: IGMP(0)[default]: v2 report received on Ethernet0/1 while operating in v1 mode, discarding!
```

In this example, I have configured R3 to use **IGMPv1**:

```
R3# show run int e0/1
Building configuration...

Current configuration : 120 bytes
!
interface Ethernet0/1
 ip address 10.3.4.3 255.255.255.0
 ip pim dense-mode
 ip igmp version 1
 ip ospf 1 area 0
end
```

The connected receiver, Rec1, is configured top use the default **IGMPv2**:

```
Rec1#sh run int e0/0 
Building configuration...

Current configuration : 95 bytes
!
interface Ethernet0/0
 ip address 10.3.4.10 255.255.255.0
 ip igmp join-group 239.1.1.1
end

Rec1# show ip igmp interface e0/0 | i Current
  Current IGMP host version is 1
  Current IGMP router version is 2
```

The key line is `Current IGMP host version is 1`.
Because Rec1 received Queries from R3 with `MRT = 0`, Rec1 switched to operating as an IGMPv1 host.

You can view this with `debug ip igmp`:

```
Rec1# debug ip igmp
*Oct  6 22:48:00.549: IGMP(0)[default]: Received v1 Query on Ethernet0/0 from 10.3.4.3
*Oct  6 22:48:00.549: IGMP(0)[default]: Starting v1 router present timer on Ethernet0/0, 120 secs
*Oct  6 22:48:00.549: IGMP(0)[default]: Set report delay time to 0.8 seconds for 239.1.1.1 on Ethernet0/0
*Oct  6 22:48:01.368: IGMP(0)[default]: Send v1 Report for 239.1.1.1 on Ethernet0/0
```


### Version 1 Router Present Timeout

If an IGMPv2 host in the **IGMPv1 Router Present** state receives an IGMPv2 Query,
it does not immediately return to IGMPv2 operation.

Instead, receiving an IGMPv1 Query starts or refreshes the:

```text
Version 1 Router Present Timeout
```

RFC 2236 defines this as:

```text
400 seconds
```

Therefore:

```text
IGMPv1 Query received
        ↓
Start 400-second timer
        ↓
Use IGMPv1 Reports
        ↓
Another IGMPv1 Query?
   ↓ Yes       ↓ No
Reset timer    Timer expires
                  ↓
            Return to IGMPv2
```

As the debug above showed, IOS XE uses a 120-second timer, not 400 seconds:
```
*Oct  6 22:48:00.549: IGMP(0)[default]: Starting v1 router present timer on Ethernet0/0, 120 secs
```

> Each subsequent IGMPv1 Query restarts the timer.
> Therefore, as long as R3 continues sending IGMPv1 Queries every 60 seconds, Rec1 remains in IGMPv1 compatibility mode indefinitely.
> If those Queries stop, Rec1 returns to IGMPv2 host operation when the 120-second timer expires.

## Leave Behavior with an IGMPv1 Querier

IGMPv1 does not support Leave Group messages.

Therefore, while operating with an IGMPv1 querier, an IGMPv2 host may suppress its normal IGMPv2 Leave messages.

Instead, group membership expires using normal IGMPv1 timeout behavior.

Conceptually:

```text
IGMPv2 operation:
Host leaves → Host sends Leave Group message → Router sends Group-Specific Queries

IGMPv1 compatibility:
Host leaves → Host doesn't send Leave Group message → Membership eventually times out
```

> RFC 2236 says that an IGMPv2 host **MAY** suppress Leave Group messages if an IGMPv1 querier is present, so it is optional.
> IOS XE does suppress Leave Group messages while the **v1 Router Present** timer is active, at least on the devices I tested.

## IGMPv2 Router with IGMPv1 Hosts

An IGMPv2 router can also have IGMPv1 and IGMPv2 receivers on the same subnet.

When the router receives an IGMPv1 Membership Report:

```text
Type = 0x12
```

it records that an **IGMPv1 host is present for that specific group**.

Unlike IGMPv1-router detection by a host, this compatibility state is maintained **per group**.

For example:

```text
239.1.1.1 → IGMPv1 host present
239.2.2.2 → IGMPv2 hosts only
```

The presence of an IGMPv1 receiver for `239.1.1.1` does not require IGMPv1 compatibility behavior for `239.2.2.2`.

## Version 1 Host Present Timer

When an IGMPv2 router receives an IGMPv1 Membership Report, it starts a timer for that group indicating that IGMPv1 receivers are present.

RFC 2236 specifies that this timer should use the `Group Membership Interval`.

Each subsequent IGMPv1 Report for the group refreshes the timer.

Conceptually:

```text
IGMPv1 Report received for 239.1.1.1
        ↓
Mark IGMPv1 host present
        ↓
Start / refresh v1 host timer
        ↓
Timer expires without another v1 Report
        ↓
No IGMPv1 host considered present
```

This timer can be verified with `debug ip igmp`:

```
*Oct  6 22:59:42.368: IGMP(0)[default]: Received v1 Report on Ethernet0/1 from 10.3.4.10 for 239.1.1.1
*Oct  6 22:59:42.368: IGMP(0)[default]: Received Group record for group 239.1.1.1, mode 2 from 10.3.4.10 for 0 sources
*Oct  6 22:59:42.368: IGMP(0)[default]: Setting v1 old host timer for 239.1.1.1 on Ethernet0/1, for 120 secs
```

Like the `Version 1 Router Present` timer, the `Version 1 Host Present` timer is `120 seconds` in IOS XE.

## Leave Messages Are Ignored

This per-group state is particularly important for IGMPv2 Leave processing.

If an IGMPv1 host is considered present for a group, the router **MUST ignore Leave Group messages for that group**.

For example:

```text
PC1 = IGMPv1 receiver
PC2 = IGMPv2 receiver

Both receive 239.1.1.1
```

PC2 leaves and sends:

```text
Leave Group: 239.1.1.1
```

However, PC1 cannot send an IGMPv2 Leave and might still require the traffic.

Therefore:

```text
IGMPv1 host present for 239.1.1.1
        ↓
IGMPv2 Leave received for 239.1.1.1
        ↓
Ignore Leave
```

The router waits for the normal membership state to expire instead.

This prevents an IGMPv2 host from causing traffic to be prematurely removed from an IGMPv1 receiver.

## Report Suppression Between Versions

IGMPv1 and IGMPv2 hosts can also suppress each other's Reports.

An IGMPv2 host with a pending report timer must allow that Report to be suppressed by either:

```text
IGMPv1 Membership Report → 0x12
IGMPv2 Membership Report → 0x16
```

For example:

```text
PC1 (IGMPv1) → Report for 239.1.1.1

PC2 (IGMPv2)
   ↓
Hears PC1's Report
   ↓
Cancels its pending Report
```

The router still knows that at least one receiver exists, so another Report is unnecessary.

## IGMPv1 and IGMPv2 Routers on the Same Subnet

Compatibility between **routers** requires special care.

IGMPv1 does not have the IGMPv2 querier election mechanism, and an IGMPv1 router has no reliable way to detect that another router is running IGMPv2.

Therefore, RFC 2236 specifies that if IGMPv1 routers exist on the subnet, the routers must be **administratively configured to use IGMPv1**.

An IGMPv2 router does not dynamically downgrade its router operation simply because it hears an IGMPv1 Query.

On IOS XE, configure the interface as IGMPv1 with:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip igmp version 1
```

Verify the version with:

```text
R1# show ip igmp interface Ethernet0/0
```

IOS XE uses IGMPv2 by default, so IGMPv1 must be explicitly configured when required.

Otherwise, you could end up with both routers declaring themselves to be the Querier:

```
R3(config-if)# do sh ip igmp int e0/1
!
  Current IGMP router version is 2
!
  Multicast designated router (DR) is 10.3.4.4  
  IGMP querying router is 10.3.4.3 (this system)
```

```
R4(config-if)#do sh ip igmp int e0/1
!
  Current IGMP router version is 1
!
  Multicast designated router (DR) is 10.3.4.4 (this system)
  IGMP querying router is 10.3.4.4 (this system)
```

In this case, both routers consider themselves the IGMP Querier.

R3 is running IGMPv2 and does not dynamically downgrade to IGMPv1 when it hears R4's IGMPv1 Queries.

R4, running IGMPv1, does not implement the IGMPv2 lowest-IP querier election mechanism.

Therefore, the normal IGMPv2 election cannot resolve the conflict.

## Summary

The most important IGMPv1/IGMPv2 compatibility rules are:

| Situation | Behavior |
|---|---|
| IGMPv2 host receives Query with Max Resp Time `0` | Detects IGMPv1 querier |
| IGMPv2 host with IGMPv1 querier | Sends IGMPv1 Reports |
| Version 1 Router Present timer | 120 seconds in IOS XE |
| IGMPv2 host with IGMPv1 querier | May suppress Leave messages |
| IGMPv2 host hears v1 or v2 Report | Either can suppress its Report |
| IGMPv2 router receives v1 Report | Tracks v1-host presence for that group |
| Version 1 Host Present timer | 120 seconds in IOS XE |
| IGMPv1 host present for group | Ignore Leaves for that group |
| IGMPv1 and v2 routers coexist | Configure routers to use IGMPv1 |

The key principle is that **IGMPv2 devices adapt their behavior to operate witg older receivers and routers**:

```text
Older querier present
→ hosts use the older reporting format

Older receiver present
→ routers disable newer leave behavior for that group
```
