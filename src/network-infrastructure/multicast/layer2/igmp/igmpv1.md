# IGMPv1

**Internet Group Management Protocol (IGMP)** allows IPv4 hosts to tell multicast routers which multicast groups have receivers on a local network.

IGMP operates only on the local subnet between:

- **Multicast receivers** — hosts that want to receive traffic for a multicast group
- **Multicast routers** — routers that need to know whether multicast traffic should be forwarded onto the subnet

IGMP does not route multicast traffic through the network. Multicast routing protocols such as **PIM** perform that function.

IGMP is carried directly inside IPv4 using **IP protocol number 2**. It is not carried by TCP or UDP.

## IGMP Group Membership

A router does not need to know the individual IP addresses of every multicast receiver.

Instead, it only needs to know whether a particular multicast group has **at least one receiver** on the local subnet.

For example:

```text
                         239.1.1.1 Receiver
                               PC1
                                |
                                |
                              SW1
                            /     \
                          R1       PC2
                                   |
                            239.1.1.1 Receiver
```

If PC1 and PC2 both want to receive traffic for `239.1.1.1`, R1 only needs to maintain state indicating:

```text
239.1.1.1 has receivers on this interface
```

R1 does not need to maintain a list containing PC1 and PC2 individually.

## IGMPv1 Message Types

IGMPv1 uses two main message types:

| Message | Purpose |
|---|---|
| **Membership Query** | Sent by multicast routers to discover which multicast groups have receivers |
| **Membership Report** | Sent by hosts to indicate membership in a multicast group |

IGMPv1 does not have a Leave message.

## Membership Queries

Multicast routers periodically send **Membership Queries** onto the local network.

The Query is sent to:

```text
224.0.0.1
```

`224.0.0.1` is the **All Hosts** multicast address.

The Query asks, effectively:

> Which multicast groups currently have receivers on this subnet?

An IGMPv1 Membership Query applies to **all multicast groups**. IGMPv1 does not support group-specific queries.

The Group Address field in an IGMPv1 Query is set to:

```text
0.0.0.0
```

IGMP Queries are link-local and are not forwarded beyond the local subnet.

## Membership Reports

A host that belongs to a multicast group responds with a **Membership Report**.

For example, if PC1 is listening to `239.1.1.1`, it can send:

```text
Membership Report
Group: 239.1.1.1
Destination: 239.1.1.1
```

The report is sent to the multicast group being reported rather than directly to the router.

This means other hosts listening to the same group can also receive the report.

## Joining a Multicast Group

A host does not have to wait for the next Membership Query before announcing a new membership.

When a host joins a multicast group, it immediately sends a Membership Report for that group.

For example:

```text
PC1 joins 239.1.1.1

PC1 --------------------> R1
     Membership Report
     Group: 239.1.1.1
```

This allows the multicast router to learn about the new receiver without waiting for the next periodic Query.

The host may repeat the Report after a short delay in case the first Report was lost.

## Periodic Membership Checking

The router periodically sends Membership Queries to verify that receivers still exist.

For example:

```text
R1 --------------------> 224.0.0.1
        Membership Query

PC1 -------------------> 239.1.1.1
        Membership Report
        Group: 239.1.1.1
```

As long as the router continues receiving Reports for a group, it knows that multicast traffic for that group should continue to be forwarded onto the subnet.

## Report Delay

If many receivers belong to the same multicast group, having all of them respond to every Query would generate unnecessary traffic.

Therefore, when a host receives a Query, it starts a **random report delay timer** for each group it belongs to.

For IGMPv1, the random delay is between:

```text
0 and 10 seconds
```

For example:

```text
PC1 timer: 2.1 seconds
PC2 timer: 6.8 seconds
PC3 timer: 9.2 seconds
```

PC1's timer expires first, so PC1 sends the Membership Report.

## Report Suppression

Because Membership Reports are sent to the multicast group itself, other receivers for the same group can hear the Report.

If PC2 and PC3 hear PC1's Report before their own timers expire, they cancel their timers and do not send their own Reports.

```text
              Membership Report
PC1 -------------------------------->
                   239.1.1.1

PC2 hears report → suppresses its own report
PC3 hears report → suppresses its own report
```

This is called **report suppression**.

The router therefore usually receives only one Membership Report for a group, even if many hosts belong to that group.

This works because the router only needs to know that **at least one receiver exists**.

## Leaving a Group

IGMPv1 has no explicit Leave message.

When a host leaves a multicast group, it simply stops sending Membership Reports for that group.

For example:

```text
PC1 leaves 239.1.1.1

R1 --------------------> 224.0.0.1
        Membership Query

        No Report for 239.1.1.1
```

If the router stops receiving Membership Reports for the group, the membership state eventually expires.

The router can then stop forwarding traffic for that group onto the subnet.

Because the router must wait for the state to time out, IGMPv1 can take relatively long to detect that the last receiver has left a group.

IGMPv2 improves this process by introducing explicit **Leave Group** messages and **Group-Specific Queries**.

## IGMPv1 Querier Behavior

IGMPv1 itself does not define an IGMP Querier election mechanism.

If multiple multicast routers are present on a subnet, IGMPv1 does not provide the lowest-IP-address querier election used by IGMPv2.

Instead, the PIM designated router (DR) operates as the IGMPv1 Querier.

This is one of the areas improved by IGMPv2.

## Cisco IOS XE IGMP CLI

IGMP operates on Layer 3 interfaces such as routed interfaces and SVIs.

There is no separate command required to enable IGMP on an interface. **Enabling PIM also enables IGMP** on that interface.

For example:

```text
R3(config)# ip multicast-routing

R3(config)# interface Ethernet0/1
R3(config-if)# ip pim dense-mode
```

`ip multicast-routing` enables multicast routing globally, while `ip pim dense-mode` enables PIM and, automatically, IGMP on the interface.

The IGMP version can then be configured with:

```text
R3(config-if)# ip igmp version 1
```

> The default IGMP version on IOS XE is **IGMPv2**.

Use `show ip igmp interface` to verify the IGMP configuration and operational state:

```text
R3# show ip igmp interface e0/1
Ethernet0/1 is up, line protocol is up
  Internet address is 10.3.4.3/24
  IGMP is enabled on interface
  Current IGMP host version is 1
  Current IGMP router version is 1
  IGMP query interval is 60 seconds
  IGMP configured query interval is 60 seconds
  Inbound IGMP access group is not set
  IGMP activity: 0 joins, 0 leaves
  Multicast routing is enabled on interface
  Multicast TTL threshold is 0
  Multicast designated router (DR) is 10.3.4.4  
  IGMP querying router is 10.3.4.4 (this system)
  No multicast groups joined by this system
```

This displays information such as:

- IGMP version
- Query interval
- Querier information
- Group membership timers

It also confirms that IGMP is enabled on the interface.

Use `show ip igmp groups` to view multicast groups with directly connected receivers learned through IGMP:

```text
R3# show ip igmp groups 
IGMP Connected Group Membership
Group Address    Interface                Uptime    Expires   Last Reporter   Group Accounted
239.1.1.1        Ethernet0/1              00:00:07  00:02:52  10.3.4.10       
224.0.1.40       Ethernet0/0              00:06:09  00:02:25  10.12.34.3  
```

### Joining a Group from a Router

For testing, a Cisco router can act as a multicast receiver by joining a group on an interface:

```text
Rec1(config)# interface 
Rec1(config-if)# ip igmp join-group 239.1.1.1
```

`ip igmp join-group` makes the router itself a member of the multicast group.

The router sends IGMP membership signaling for the group and accepts multicast packets destined for that group.

This is particularly useful in multicast labs because a router can act as a receiver without requiring a separate host.

The membership can be verified with:

```text
Rec1#show ip igmp interface e0/0
Ethernet0/0 is up, line protocol is up
  Internet address is 10.3.4.10/24
  IGMP is enabled on interface
  Current IGMP host version is 1
  Current IGMP router version is 2
  IGMP query interval is 60 seconds
  IGMP configured query interval is 60 seconds
  Inbound IGMP access group is not set
  IGMP activity: 1 joins, 0 leaves
  Multicast routing is disabled on interface
  Multicast TTL threshold is 0
  Multicast groups joined by this system (number of users):
      239.1.1.1(1)

Rec1# show ip igmp groups
IGMP Connected Group Membership
Group Address    Interface                Uptime    Expires   Last Reporter   Group Accounted
239.1.1.1        Ethernet0/0              00:01:27  never     10.3.4.10  
```

### Static Group Membership

Cisco IOS/IOS XE also supports:

```text
R3(config-if)# ip igmp static-group 239.1.1.1
```

This also causes multicast traffic for the group to be forwarded onto the interface regardless of whether there are interested receivers,
but the router itself does **not** become a receiver for the traffic.

The distinction is:

```text
ip igmp join-group   → Router joins and receives the multicast traffic
ip igmp static-group → Traffic is forwarded onto the interface, but the router does not receive it
```

For simple multicast labs, `ip igmp join-group` on a router acting as a receiver is often the easiest way to emulate a receiver.

## IGMPv1 Summary

IGMPv1 establishes the basic IGMP model:

```text
Router sends Membership Query
            ↓
Hosts start random report timers
            ↓
First host sends Membership Report
            ↓
Other hosts suppress their reports
            ↓
Router knows the group has local receivers
```

Its main limitations are:

- No explicit Leave message
- No Group-Specific Queries
- No IGMP querier election mechanism
- No source filtering

These limitations are addressed by **IGMPv2** and **IGMPv3**.