# 1.6.a (iv) IGMP Filter

**IGMP filtering** allows a switch or router to control multicast group memberships.

Normally, when **IGMP snooping** is enabled, a Layer 2 switch learns multicast group memberships by examining IGMP Membership Reports received from hosts.

However, an administrator might want to restrict multicast group membership.

For example:

- Allow a host to join certain multicast groups but not others.
- Limit the number of multicast groups a host can join.
- Prevent unauthorized subscriptions to multicast services.

Cisco IOS XE provides two related Layer 2 features:

| Feature | Purpose |
|---|---|
| **IGMP Filtering** | Controls which multicast groups a port can join |
| **IGMP Throttling** | Limits how many multicast groups a port can join |

Both operate on dynamically learned multicast group memberships.

## IGMP Filtering

IGMP filtering uses **IGMP profiles** to permit or deny membership in specific multicast groups.

An IGMP profile defines a set of multicast group addresses and whether membership in those groups is permitted or denied.

The profile is then applied to a Layer 2 switch port.

For example:

```text
                 SW1
              /       \
        Ethernet0/1  Ethernet0/2
             |            |
            PC1          PC2
```

Suppose PC1 is allowed to join `239.1.1.1` but not `239.1.1.2`.

```text
PC1 → IGMP Report for 239.1.1.1
      → Permitted

PC1 → IGMP Report for 239.1.1.2
      → Denied
```

When an IGMP Report is denied, the switch drops the Report and does not dynamically add the port as a listener for that group.

The primary configuration commands are:

```text
ip igmp profile <number>
ip igmp filter <number>
```

An IGMP profile can be applied to multiple Layer 2 ports, but each port can have only **one IGMP profile** applied.

## IGMP Maximum Groups and Throttling

IGMP throttling limits the number of multicast groups that can be dynamically learned on a Layer 2 port.

For example, suppose Ethernet0/1 has a maximum of two groups:

```text
Ethernet0/1

239.1.1.1 → Joined
239.1.1.2 → Joined
239.1.1.3 → New membership request
```

The port has already reached its limit.

IOS XE supports two actions when an additional group membership is requested:

| Action | Behavior |
|---|---|
| `deny` | Reject the new membership request |
| `replace` | Replace a randomly selected existing group entry with the new group |

The relevant commands are:

```text
ip igmp max-groups <number>
ip igmp max-groups action {deny | replace}
```

> By default, there is **no maximum group limit**.
> Once a limit is configured and reached, the default action is `deny`, which drops new IGMP membership reports for additional groups.
> The alternative `replace` action removes an existing group entry to accommodate a new one.

## IGMP Filtering on Routed Interfaces

IOS XE also supports IGMP filtering and membership limits on **Layer 3 interfaces**, using different commands.

These features operate on the router's IGMP membership state rather than the Layer 2 switch's IGMP snooping state.

| Command | Purpose |
|---|---|
| `ip igmp access-group` | Uses an ACL to control which multicast memberships the router accepts |
| `ip igmp limit` | Limits the number of IGMP membership states maintained by the router |

For example:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip igmp access-group 10
R1(config-if)# ip igmp limit 50
```

- `ip igmp access-group` can use standard ACLs to filter multicast groups or extended ACLs to filter IGMPv3 source/group `(S,G)` memberships.
- `ip igmp limit` can be configured globally or per interface. The interface-level command also supports an optional `except` ACL to exclude selected groups or channels from the limit.

These Layer 3 features are distinct from the Layer 2 `ip igmp filter` and `ip igmp max-groups` commands.

## IGMP Filtering vs. Multicast Traffic Filtering

IGMP filtering controls **group membership signaling**, rather than directly filtering multicast data packets.

For example:

```text
IGMP Membership Report
        ↓
IGMP Filtering
        ↓
Permit or deny membership
        ↓
IGMP Snooping Membership State
        ↓
Multicast Forwarding Decision
```

Denying membership prevents the switch from dynamically learning the port as a listener for that group.

However, IGMP filtering is **not equivalent to an ACL that drops multicast data traffic**.

Multicast packets might still reach a port through other forwarding mechanisms, such as unknown multicast flooding or statically configured multicast forwarding entries.

IGMP filtering also does not prevent general IGMP Queries from being forwarded.

## Important Limitations

On Catalyst 9000 IOS XE switches:

- IGMP filtering applies to dynamically learned multicast memberships, not statically configured memberships.
- IGMP profiles can be applied to Layer 2 physical interfaces, but not routed ports, SVIs, or physical EtherChannel member ports.
- IGMPv3 join and leave messages are not supported by the IGMP profile filtering feature.
- IGMP profile filtering is based on **multicast group addresses**, not the source-specific INCLUDE/EXCLUDE filtering supported by IGMPv3.

## Key Points

- **IGMP filtering** controls which multicast groups a Layer 2 port can join.
- **IGMP profiles** define permitted or denied multicast group addresses.
- **IGMP throttling** limits the number of groups dynamically learned on a port.
- The two throttling actions are `deny` (default) and `replace`.
- By default, no IGMP profiles are applied and no maximum group limit is configured.
- On routed interfaces, `ip igmp access-group` controls accepted group memberships and `ip igmp limit` limits IGMP membership state.
- These features control IGMP membership learning, not multicast data packets directly.

The following sections cover IGMP profiles, applying IGMP filters, IGMP maximum group limits and throttling, and IGMP filtering on routed interfaces.