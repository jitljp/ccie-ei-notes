# IGMPv2 Membership Reports

IGMPv2 hosts send **Membership Reports** to indicate that they want to receive traffic for an IPv4 multicast group.

A Report can be generated:

- In response to a General Query
- In response to a Group-Specific Query
- Unsolicited, when a host first joins a group

The IGMPv2 Membership Report message type is `0x16`.

## Membership Report Format

An IGMPv2 Membership Report uses the standard 8-byte IGMPv2 format:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      Type     | Max Resp Time |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Group Address                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

For an IGMPv2 Membership Report:

```text
Type:           0x16
Max Resp Time:  0
Group Address:  Group being reported
```

The **Max Resp Time** field is meaningful only in Queries. It is set to `0` in a Membership Report and ignored by receivers.

For example, a Report for `239.1.1.1` contains:

```text
Type:           0x16
Max Resp Time:  0
Group Address:  239.1.1.1
```

## IP Encapsulation

The Report is sent directly to the multicast group being reported.

For example:

```text
IP Destination:     239.1.1.1
IP Protocol:        2 (IGMP)
TTL:                1

IGMP Type:          0x16
IGMP Group Address: 239.1.1.1
```

Therefore, for an IGMPv2 Membership Report:

```text
IP Destination = IGMP Group Address
```

The Report is not unicast to the querier.

This allows other hosts listening to the same group to hear the Report, which enables **report suppression**.

## Reports in Response to Queries

When a host receives a Query, it starts a random **report delay timer** for each applicable group membership.

For a General Query:

```text
Query applies to all groups on the interface
```

For a Group-Specific Query:

```text
Query applies only to the specified group
```

The timer is randomly selected between `0` and `Max Response Time` (default 10) seconds.

For example, if a General Query specifies:

```text
Max Resp Time = 10 seconds
```

three hosts belonging to `239.1.1.1` might select:

```text
PC1 = 2.3 seconds
PC2 = 6.7 seconds
PC3 = 8.4 seconds
```

If PC1's timer expires first, it sends:

```text
PC1 --------------------> 239.1.1.1
       Membership Report
```

The other hosts can hear this Report and suppress their own Reports.

Report suppression is covered in detail on the next page.

## Unsolicited Membership Reports

A host does not wait for a Query when it first joins a multicast group.

It immediately sends an **unsolicited Membership Report**.

For example:

```text
PC1 joins 239.1.1.1

PC1 --------------------> 239.1.1.1
       Membership Report
```

This allows multicast forwarding to begin without waiting for the next periodic General Query.

Because the initial Report might be lost, RFC 2236 recommends repeating the Report **once or twice** after a short random delay.

The default **Unsolicited Report Interval** is:

```text
10 seconds
```

The delay before a repeated unsolicited Report is selected from between `0` and `Unsolicited Report Interval` seconds.

> The concept is the same as the Query Response Interval (between 0 and 10 seconds).

Therefore, a typical sequence is:

```text
Host joins group
      ↓
Immediate unsolicited Report
      ↓
Random delay up to 10 seconds
      ↓
Additional unsolicited Report
```

The additional Report improves reliability if the initial Report is lost.

## Router Processing of Reports

When a router receives a valid Membership Report, it creates or refreshes membership state for that group on the receiving interface.

Conceptually:

```text
Report received for 239.1.1.1 on Ethernet0/0
                    ↓
239.1.1.1 has receivers on Ethernet0/0
```

The router starts or refreshes the group's membership expiration timer.

If additional Reports are received:

```text
Report received
      ↓
Membership timer refreshed
```

The router does **not** normally need to maintain a list of every receiver.

It only needs to know that **at least one receiver exists for this group on this interface**.

> This is why IGMPv1 and IGMPv2 use **report suppression**.

If the membership timer eventually expires without another Report, the router can remove the group from that interface.

## Host Membership States

RFC 2236 defines three host states for each multicast group on each interface:

| State | Meaning |
|---|---|
| **Non-Member** | Host does not belong to the group |
| **Delaying Member** | Host belongs to the group and has a report timer running |
| **Idle Member** | Host belongs to the group but has no report timer running |

### Joining a Group

When a host joins a group:

```text
Non-Member
     |
     | Join group
     | Send Report
     | Set last-reporter flag
     | Start unsolicited Report timer
     v
Delaying Member
```

The first unsolicited Report is sent immediately.

The host then starts a timer for a possible repeated unsolicited Report.

### Receiving a Query

An Idle Member that receives an applicable Query starts a report delay timer:

```text
Idle Member
     |
     | Query received
     | Start report timer
     v
Delaying Member
```

### Report Timer Expiration

If the timer expires before another host reports:

```text
Delaying Member
     |
     | Timer expires
     | Send Report
     | Set last-reporter flag
     v
Idle Member
```

### Hearing Another Host's Report

If another host reports first:

```text
Delaying Member
     |
     | Report received
     | Stop timer
     | Clear last-reporter flag
     v
Idle Member
```

This is the basis of **IGMPv2 report suppression**.

## Last-Reporter Flag

An IGMPv2 host remembers whether it was the **last host to send a Membership Report** for a group.

When the host itself sends a Report:

```text
Last-reporter flag = Set
```

If it hears another host report that group while its own timer is running:

```text
Last-reporter flag = Cleared
```

This flag is important when the host later leaves the group.

If the host believes it was the last reporter, it normally sends a **Leave Group** message.

If another host reported more recently, the host may omit the Leave because it already knows that another receiver exists on the subnet.

The detailed behavior is covered on the **IGMPv2 Leave Process** page.

> **Last reporter does not mean last remaining receiver.** It means only that the host was the most recent host to send a Membership Report for that group.

## Link-Local Multicast Groups

The `224.0.0.0/24` range is the **Local Network Control Block**.

These multicast groups are used by protocols and control functions on the local subnet, for example:

```text
224.0.0.1  All Hosts
224.0.0.2  All Routers
224.0.0.5  All OSPF Routers
224.0.0.6  OSPF DR/BDR Routers
```

Traffic for these groups is link-local and is not forwarded by routers.

Devices generally do **not** use IGMP Membership Reports to announce membership in these control groups. Instead, the relevant protocol causes the device to listen to the required multicast address.

For example, an OSPF router listens to `224.0.0.5` as part of running OSPF; it does not first send an IGMP Membership Report for `224.0.0.5`.

### 224.0.0.1

Every IPv4 multicast-capable host implicitly belongs to:

```text
224.0.0.1
```

the **All Hosts** group.

Hosts do not send Membership Reports for this group.

RFC 2236 defines this membership as permanently remaining in the **Idle Member** state:

```text
224.0.0.1
    ↓
Implicit membership
    ↓
No Membership Reports
```

Routers therefore do not need IGMP to discover membership in the All Hosts group.

> This general behavior is important for IGMP snooping: switches cannot rely on IGMP membership state for `224.0.0.0/24` control groups such as OSPF multicast addresses.

## IGMPv1 Report Compatibility

IGMPv2 must coexist with IGMPv1 hosts.

An IGMPv2 Membership Report uses:

```text
0x16
```

while an IGMPv1 Membership Report uses:

```text
0x12
```

An IGMPv2 host must allow its Report to be suppressed by either:

```text
IGMPv1 Membership Report (0x12)
IGMPv2 Membership Report (0x16)
```

Therefore, an IGMPv1 host can suppress an IGMPv2 host's pending Report.

### IGMPv1 Hosts on an IGMPv2 Network

When a router receives an IGMPv1 Membership Report for a group, it records that an **IGMPv1 host is present** for that group.

That state is maintained for the Group Membership Interval.

While IGMPv1 hosts are considered present for the group, the router must **ignore IGMPv2 Leave Group messages for that group**.

This is necessary because IGMPv1 hosts have no Leave message and therefore might still be receiving the group.

## IOS XE Verification

Use `show ip igmp groups` to view directly connected multicast group memberships:

```text
R3# show ip igmp groups 
IGMP Connected Group Membership
Group Address    Interface                Uptime    Expires   Last Reporter   Group Accounted
239.1.1.1        Ethernet0/1              00:55:04  00:02:57  10.3.4.10       
224.0.1.40       Ethernet0/0              02:20:17  00:02:23  10.12.34.3  
```

Fields include:

- **Group Address** — multicast group with local receivers
- **Interface** — interface on which the Report was received
- **Uptime** — how long the membership has existed
- **Expires** — Group Membership Timer
- **Last Reporter** — source of the most recently received Report

The **Last Reporter** field does not identify every receiver.

For example:

```text
PC1 ──┐
PC2 ──┼── Ethernet0/0 ── R1
PC3 ──┘
```

all three hosts might receive `239.1.1.1`, but because of report suppression R1 might display only:

```text
Last Reporter: 10.1.1.10
```

That does **not** mean `10.1.1.10` is the only receiver.

For additional information:

```text
R3# show ip igmp groups detail 

Flags: L - Local, U - User, SG - Static Group, VG - Virtual Group,
       SS - Static Source, VS - Virtual Source,
       Ac - Group accounted towards access control limit

Interface:      Ethernet0/1
Group:          239.1.1.1
Flags:
Uptime:         00:56:20
Group mode:     EXCLUDE (Expires: 00:02:46)
Last reporter:  10.3.4.10
Source list is empty

Interface:      Ethernet0/0
Group:          224.0.1.40
Flags:          L U 
Uptime:         02:21:33
Group mode:     EXCLUDE (Expires: 00:02:05)
Last reporter:  10.12.34.4
Source list is empty
```

## Generating Reports in a Lab

An IOS XE router can act as an IGMP receiver with:

```text
Rec1(config)# interface Ethernet0/0
Rec1(config-if)# ip igmp join-group 239.1.1.1
```

This makes `Rec1` itself (an IOS XE router) a member of `239.1.1.1`.

`Rec1` therefore generates IGMP membership signaling for the group and accepts multicast traffic destined for the group.

Verify the membership with:

```text
Rec1# show ip igmp interface e0/0
Ethernet0/0 is up, line protocol is up
!output omitted
  Multicast groups joined by this system (number of users):
      239.1.1.1(1)

Rec1# show ip igmp groups        
IGMP Connected Group Membership
Group Address    Interface                Uptime    Expires   Last Reporter   Group Accounted
239.1.1.1        Ethernet0/0              01:09:25  never     10.3.4.10  
```

This makes `ip igmp join-group` useful for creating multicast receivers in labs without requiring separate host devices.

Do not confuse it with:

```text
R3(config-if)# ip igmp static-group 239.1.1.1
```

`ip igmp static-group` installs static group forwarding state on the interface of a multicast router, but does **not** make the router itself a multicast receiver.
