# IGMPv2

**IGMPv2** improves the basic group membership mechanism introduced by IGMPv1.

The core purpose remains the same:

- Hosts use IGMP to indicate which IPv4 multicast groups they want to receive.
- Multicast routers use IGMP to determine whether a subnet has receivers for a multicast group.
- IGMP operates only between hosts and multicast routers on the local subnet.

The major improvement in IGMPv2 is **faster detection when the last receiver leaves a multicast group**.

## IGMPv2 Improvements

IGMPv2 adds several features that were not present in IGMPv1:

| Feature | Purpose |
|---|---|
| **Leave Group message** | Allows a host to explicitly indicate that it is leaving a group |
| **Group-Specific Query** | Allows the router to check whether any receivers remain for a particular group |
| **Maximum Response Time** | Allows the querier to control how quickly hosts should respond to Queries |
| **Querier election** | Allows multiple IGMPv2 routers on a subnet to determine which router sends Queries |

IGMPv2 also continues to use **Membership Reports** and **report suppression**.

## Basic IGMPv2 Operation

Normal IGMPv2 operation can be summarized as:

```text
Router sends General Query
          ↓
Hosts start random response timers
          ↓
Host sends Membership Report
          ↓
Other members of the same group suppress their Reports
          ↓
Router knows that the group has receivers
```

When a host wants to join a new group, it does not wait for a Query. It sends an unsolicited **Membership Report** immediately.

For example:

```text
PC1 joins 239.1.1.1

PC1 --------------------> 239.1.1.1
        Membership Report
```

## Leaving a Group

IGMPv2 significantly improves the leave process.

When a host leaves a multicast group, it can send a **Leave Group** message to:

```text
224.0.0.2
```

`224.0.0.2` is the **All Routers** multicast address.

The querier then sends **Group-Specific Queries** for that multicast group.

```text
PC1 --------------------> 224.0.0.2
        Leave Group
        Group: 239.1.1.1

R1 ---------------------> 239.1.1.1
        Group-Specific Query
```

If another receiver still belongs to the group, it responds with a Membership Report.

If no receiver responds, the router can remove the group membership state and stop forwarding traffic for that group onto the subnet.

This is much faster than IGMPv1, where the router generally has to wait for the membership state to expire.

## IGMPv2 Querier

Only one IGMP router normally sends periodic Queries on a subnet.

If multiple IGMPv2 routers are present, they elect a **querier**.

The router with the **lowest IPv4 address** becomes the querier.

```text
R1: 10.0.0.1  ← Querier
R2: 10.0.0.2
R3: 10.0.0.3
```

Querier behavior and configuration are covered separately in the **IGMP Querier** section.

## IGMPv2 and IGMPv1

IGMPv2 was designed to coexist with IGMPv1 hosts and routers.

The protocols share the same basic Query and Report model, but IGMPv2 adds the additional mechanisms required for faster and more controlled membership management.

The most important differences are:

```text
IGMPv1
- General Queries
- Membership Reports
- No Leave message
- No Group-Specific Query

IGMPv2
- General Queries
- Group-Specific Queries
- Membership Reports
- Leave Group messages
- Querier election
```

## Cisco IOS XE

IGMPv2 is the default IGMP version on IOS XE.

It can be explicitly configured on an interface with:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip igmp version 2
```

Verify the version and IGMP parameters with:

```text
R3# show ip igmp interface e0/0
Ethernet0/0 is up, line protocol is up
  Internet address is 10.12.34.3/24
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
  IGMP activity: 1 joins, 0 leaves
  Multicast routing is enabled on interface
  Multicast TTL threshold is 0
  Multicast designated router (DR) is 10.12.34.4  
  IGMP querying router is 10.12.34.1  
  Multicast groups joined by this system (number of users):
      224.0.1.40(1)
```

View learned group memberships with:

```text
R3# show ip igmp groups 
IGMP Connected Group Membership
Group Address    Interface                Uptime    Expires   Last Reporter   Group Accounted
239.1.1.1        Ethernet0/1              00:08:52  00:02:42  10.3.4.10       
224.0.1.40       Ethernet0/0              00:14:54  00:02:46  10.12.34.3   
```

The following sections examine IGMPv2 in more detail:

- IGMPv2 Message Types
- IGMPv2 Queries
- IGMPv2 Membership Reports
- IGMPv2 Report Suppression
- IGMPv2 Leave Process