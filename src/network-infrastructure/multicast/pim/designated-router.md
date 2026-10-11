# PIM Designated Router

The **PIM Designated Router (DR)** is the router elected to perform certain multicast routing responsibilities on behalf of hosts connected to a shared network segment.

When multiple PIM routers are connected to the same LAN, a DR election determines which router acts on behalf of the locally connected multicast sources and receivers.

In PIM Sparse Mode, the DR has two primary responsibilities:

- **Source-facing DR:** Sends PIM Register messages to the Rendezvous Point (RP) on behalf of directly connected multicast sources.
- **Receiver-facing DR:** Sends upstream PIM Join/Prune messages on behalf of directly connected multicast receivers.

The DR election prevents multiple routers on the same LAN from independently performing these responsibilities for the same hosts.

## PIM DR Operation

Consider the following topology:

```text
                  Multicast Network
                    /         \
                   /           \
                  R1           R2
             10.0.12.1      10.0.12.2
                   \           /
                    \         /
                       SW1
                        |
                        |
                      PC1
                IGMP Membership
               Report for 239.1.1.1
```

Both R1 and R2 have PIM enabled on their interfaces connected to SW1.

When PC1 sends an IGMP Membership Report for `239.1.1.1`, both routers may receive it.

However, only the elected PIM DR is responsible for originating the upstream PIM Join on behalf of PC1.

For example, if R2 is elected DR:

```text
PC1 sends IGMP Report
         |
         v
    R1 and R2
         |
         v
    R2 is PIM DR
         |
         v
R2 sends (*,G) Join toward RP
```

This ensures that the receiver's local subnet is represented by a single DR when establishing the multicast distribution tree.

### Source-Facing DR

The DR also plays an important role when a multicast source is directly connected.

For example:

```text
             Multicast Source
                10.10.10.10
                     |
                    SW1
                   /   \
                  /     \
                 R1     R2
                  \     /
                   \   /
              Multicast Network
                     |
                     RP
                   1.1.1.1
```

Suppose R2 is elected DR on the source's LAN.

When the source begins sending traffic to an ASM group, R2 is responsible for encapsulating the initial multicast packets in **PIM Register messages** and sending them to the RP.

R1 does not independently register the same source.

This allows the RP to learn about the active multicast source.

> **Note:** PIM Register messages are used for traditional PIM-SM ASM operation. They are not required for Source-Specific Multicast (SSM).

## PIM DR Election

PIM routers elect a DR using information exchanged in **PIM Hello messages**.

The election is performed independently on each PIM-enabled network segment.

By default, all routers advertise a DR priority of **1** using the DR Priority Hello option (Type 19).

The election uses the following rules:

1. The router advertising the **highest DR priority** wins.
2. If multiple routers have the same priority, the router with the **highest primary IP address** wins.

Unlike elections in some other routing protocols, **a higher PIM DR priority is better**.

### Example 1: Default DR Election

Consider the following routers:

| Router | Interface IP | DR Priority |
|---|---|---|
| R1 | 10.0.12.1 | 1 |
| R2 | 10.0.12.2 | 1 |

Both routers have the default priority.

Therefore, R2 wins because `10.0.12.2` is higher than `10.0.12.1`.

```text
R1: 10.0.12.1, Priority 1
R2: 10.0.12.2, Priority 1

DR = R2
```

### Example 2: Configured DR Priority

Suppose R1 is configured with a higher DR priority:

| Router | Interface IP | DR Priority |
|---|---|---|
| R1 | 10.0.12.1 | 100 |
| R2 | 10.0.12.2 | 1 |

R1 now wins the election despite having the lower IP address.

```text
R1: 10.0.12.1, Priority 100  <-- DR
R2: 10.0.12.2, Priority 1
```

The IP address is considered only when the highest priorities are equal.

### DR Priority Option Not Supported

An important compatibility exception occurs when a PIM router does not advertise the **DR Priority Hello option**.

According to RFC 7761, if any PIM router on the link does not advertise this option, priority-based election is disabled for that link.

Instead, the router with the **highest primary IP address** becomes the DR, regardless of the advertised priorities.

For example:

| Router | IP Address | Advertised DR Priority |
|---|---|---|
| R1 | 10.0.12.1 | 100 |
| R2 | 10.0.12.2 | Not advertised |
| R3 | 10.0.12.3 | 1 |

The winner is R3 because it has the highest IP address.

```text
DR = R3 (10.0.12.3)
```

This fallback provides compatibility with older PIM implementations that do not support the DR Priority option.

> **Note:** A missing DR Priority option is different from an advertised priority of `0`. Priority `0` is still a valid priority, and a router advertising it remains eligible to become DR.

## Configuring PIM DR Priority

On IOS XE, the DR priority is configured per interface using:

```text
R1(config-if)# ip pim dr-priority <priority>
```

The default priority is `1`.

The configurable IPv4 range is `0` through `4294967294` on IOS XE platforms supporting this command.

### IPv4 Example

Configure R1 to be preferred as DR:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip pim sparse-mode
R1(config-if)# ip pim dr-priority 100
```

R2 retains the default priority:

```text
R2(config)# interface Ethernet0/0
R2(config-if)# ip pim sparse-mode
```

The resulting election is:

```text
R1 Priority: 100  <-- DR
R2 Priority: 1
```

To restore the default priority:

```text
R1(config-if)# no ip pim dr-priority
```

Changing the DR priority does not require restarting PIM.

The updated priority is advertised through PIM Hello messages, allowing neighboring routers to recalculate the election.

### DR Priority 0

Setting the DR priority to `0` makes a router less preferred:

```text
R1(config-if)# ip pim dr-priority 0
```

However, **priority 0 does not disable DR eligibility**.

If R1 is the only PIM router on the segment, it can still become DR.

Similarly, if every router advertises priority 0, the router with the highest IP address wins.

## DR Election Changes

The elected DR can change when:

- A new PIM neighbor is discovered.
- An existing PIM neighbor expires or goes down.
- A router's DR priority changes.
- An interface's primary IP address changes.
- A neighbor's advertised election information changes.

For example:

```text
Initial state:

R1 Priority 100  <-- DR
R2 Priority 1

R1 fails.

New state:

R2 Priority 1    <-- DR
```

R2 assumes the DR responsibilities after it determines that R1 is no longer an eligible PIM neighbor.

If R1 subsequently returns with priority 100, it can reclaim the DR role.

**PIM DR election is preemptive by design.** There is no separate `preempt` command required.

Neighbor failure detection normally depends on the PIM Hello Holdtime, which defaults to 105 seconds when the Hello interval is 30 seconds.

However, a neighbor can be removed sooner when the interface goes down or a Hello with Holdtime 0 is received.

## PIM DR vs. IGMP Querier

The **PIM DR** and **IGMP Querier** are two different roles with independent election procedures.

| Feature | PIM DR | IGMPv2/v3 Querier |
|---|---|---|
| Election information | PIM Hello messages | IGMP Queries |
| Primary criterion | Highest DR priority | Lowest IPv4 address |
| Tie-breaker | Highest primary IP | Not applicable |
| Primary responsibility | PIM operations on behalf of local hosts | Sending IGMP Queries |

For example:

```text
R1: 10.0.12.1, PIM DR Priority 1
R2: 10.0.12.2, PIM DR Priority 1
```

The election results are:

```text
PIM DR:       R2 (highest IP address)
IGMP Querier: R1 (lowest IP address)
```

Therefore, **the PIM DR is not necessarily the IGMP Querier**.

R1 can send periodic IGMP Queries, while R2 sends upstream PIM Joins in response to receiver membership.

The same principle applies to IPv6, where PIM DR and MLD Querier elections are separate.

## PIM DR vs. PIM Assert Winner

The PIM DR must also be distinguished from the **PIM Assert winner**.

The DR election selects a router to perform specific PIM responsibilities on a shared segment.

A PIM Assert election selects the router that should forward a particular multicast stream onto a multiaccess network when competing routers would otherwise forward duplicate traffic.

| Feature | PIM DR | PIM Assert Winner |
|---|---|---|
| Scope | Network segment | Multicast tree/flow on an interface |
| Selection | DR priority, then IP address | RPF-related metric preference and metric, then IP address |
| Purpose | Local source and receiver PIM responsibilities | Prevent duplicate multicast forwarding |
| Trigger | PIM neighbor/election changes | Multicast forwarder conflict |

The DR is normally responsible for forwarding multicast traffic onto a directly connected receiver LAN in the absence of a competing Assert winner.

However, **the DR and Assert winner do not have to be the same router**.

Likewise, the **Designated Forwarder (DF)** in Bidirectional PIM is a separate role, elected using different rules.

PIM Assert and Bidirectional PIM are covered in their respective chapters.

## IPv6 PIM DR Election

IPv6 PIM uses the same general DR election principles as IPv4.

The router advertising the highest DR priority wins, with the highest IPv6 address used to break ties.

IPv6 PIM neighbors normally communicate using link-local IPv6 addresses, so these addresses are used for the election tie-breaker.

### Configuration

Configure the IPv6 PIM DR priority using:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 pim dr-priority 100
```

The default IPv6 DR priority is also `1`.

IPv4 and IPv6 PIM DR elections operate independently.

Therefore, the same router does not necessarily become DR for both address families on a dual-stack LAN.

## Verification

### `show ip pim interface`

The primary command for identifying the elected IPv4 PIM DR is:

```text
R1# show ip pim interface
```

Example:

```text
Address          Interface       Ver/Mode    Nbr   Query   DR
                                             Count Intvl
10.0.12.1        Ethernet0/0     v2/Sparse     1     30    10.0.12.2
```

In this example:

```text
Local Router: R1 (10.0.12.1)
PIM DR:       R2 (10.0.12.2)
```

R1 has one PIM neighbor, and R2 is the elected DR.

### `show ip pim neighbor`

Use:

```text
R1# show ip pim neighbor
```

Example:

```text
PIM Neighbor Table
Neighbor       Interface      Uptime/Expires   Ver  DR
Address                                               Prio/Mode
10.0.12.2      Ethernet0/0    00:05:30/00:01:34 v2   1 / DR
```

The `DR` flag indicates that the neighbor is the elected Designated Router.

The `DR Prio` field displays the advertised priority.

If an older PIM neighbor does not advertise the DR Priority option, IOS XE may display `N` instead of a numeric priority.

### Additional Verification

| Command | Purpose |
|---|---|
| `show ip pim interface` | Display the elected IPv4 PIM DR |
| `show ip pim interface Ethernet0/0 detail` | Display detailed IPv4 PIM interface information |
| `show ip pim neighbor` | Display neighbors, DR priorities, and DR flags |
| `show ip igmp interface Ethernet0/0` | Compare the PIM DR with the IGMP Querier |
| `show ipv6 pim interface` | Display IPv6 PIM interface and DR information |
| `show ipv6 pim neighbor` | Display IPv6 PIM neighbors |
| `show running-config interface Ethernet0/0` | Verify configured DR priority |

If the expected router is not becoming DR, verify that the PIM neighbor relationships are established, all routers advertise the DR Priority option, and the priorities are configured on the correct interfaces.

## Key Points

- The **PIM DR** performs multicast routing responsibilities on behalf of directly connected sources and receivers.
- On a source LAN, the DR sends PIM Register messages toward the RP for ASM traffic.
- On a receiver LAN, the DR originates upstream PIM Join/Prune messages based on local receiver interest.
- The **highest DR priority wins**; ties are broken by the highest primary IP address.
- The default DR priority is **1**.
- If any router does not advertise the DR Priority Hello option, RFC 7761 specifies an election based on the highest IP address.
- Configure IPv4 DR priority using `ip pim dr-priority` and IPv6 using `ipv6 pim dr-priority`.
- Priority `0` does not make a router ineligible to become DR.
- DR election is **preemptive**, and the election can change when neighbors or priorities change.
- The **PIM DR, IGMP Querier, PIM Assert winner, and Bidir-PIM Designated Forwarder are distinct roles**.
- Use `show ip pim interface` and `show ip pim neighbor` to verify DR election.