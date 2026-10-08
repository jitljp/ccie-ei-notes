# IGMP Snooping Report Suppression

**IGMP Snooping Report Suppression** reduces the number of IGMP Membership Reports forwarded from a Layer 2 switch toward multicast routers.

It is important to distinguish this from the **host-based Report Suppression** built into IGMPv1 and IGMPv2.

The two mechanisms have a similar goal:

```text
Prevent many duplicate Reports from reaching the multicast router
```

but they operate in different places:

```text
Without IGMP snooping
→ hosts suppress their own Reports (IGMPv1 and IGMPv2)

With IGMP snooping Report Suppression
→ the switch suppresses Reports toward the multicast routers
```

## Without IGMP Snooping

Consider three IGMPv2 receivers on the same VLAN:

```text
                R1
                 |
                SW1
             /   |   \
          Rec1  Rec2  Rec3
```

All three receivers have joined:

```text
239.1.1.1
```

R1 sends a General Query:

```text
      R1
       |
       | General Query
       v
      SW1
  /    |    \
  v    v    v
Rec1 Rec2 Rec3
```

Each receiver starts a random response timer for `239.1.1.1`.

For example:

```text
Rec1 → 3.2 seconds
Rec2 → 7.6 seconds
Rec3 → 8.4 seconds
```

Rec1's timer expires first, so Rec1 sends:

```text
IGMPv2 Membership Report
Group = 239.1.1.1
```

Without IGMP snooping, SW1 treats the multicast frame using normal Layer 2 multicast flooding behavior.

The Report is therefore flooded through the VLAN:

```text
                R1
                 ^
                 |
                SW1
             /   |   \
            ^    v     v
          Rec1  Rec2  Rec3
```

R1 receives the Report, but so do Rec2 and Rec3.

IGMPv1 and IGMPv2 implement **host Report Suppression**.

When Rec2 and Rec3 hear Rec1's Report for the same group, they cancel their pending Reports:

```text
Rec1 → sends Report

Rec2 → hears Report
     → cancels timer

Rec3 → hears Report
     → cancels timer
```

The result is approximately:

```text
3 receivers
     ↓
1 Membership Report
     ↓
multicast router
```

The suppression is being performed by the **hosts themselves**.

## Why Host Report Suppression Works

The multicast router does not normally need to know how many individual IGMPv1/v2 hosts have joined a group.

It only needs to know:

```text
Does at least one receiver for G exist on this interface?
```

Therefore, one Report for `239.1.1.1` is sufficient to refresh the router's group membership.

The other receivers do not need to send duplicate Reports.

## The Problem for IGMP Snooping

IGMP snooping introduces a different requirement.

The switch needs to know **which switch ports** contain receivers.

Suppose:

```text
                R1
                 |
                SW1
             /   |   \
          E0/1  E0/2 E0/3
            |     |    |
          Rec1  Rec2  Rec3
```

If only Rec1 sends a Report and Rec2 and Rec3 suppress theirs, the switch would not have received membership information directly from every listener port.

A snooping switch therefore intercepts IGMP Reports and builds its own listener database.

It does **not** forward one host's Reports to other hosts.

Conceptually:

```text
Rec1 ---- Report --->\
                      \
Rec2 ---- Report ----> SW1 --1 Report---> R1
                      /
Rec3 ---- Report --->/
```

SW1 can learn:

```text
239.1.1.1
→ E0/1
→ E0/2
→ E0/3
```

without requiring R1 to receive three duplicate Reports.

## With IGMP Snooping Report Suppression

With Report Suppression enabled, SW1 processes the Reports it receives from the hosts.

For example:

```text
Rec1 → Report for 239.1.1.1 on E0/1
Rec2 → Report for 239.1.1.1 on E0/2
Rec3 → Report for 239.1.1.1 on E0/3
```

The switch uses them to maintain its listener state:

```text
239.1.1.1
├── E0/1
├── E0/2
└── E0/3
```

However, it forwards only the **first Report for that group** toward the multicast-router ports for that multicast router Query.

For example:

```text
Rec1 ---- Report ----\
                      \
Rec2 ---- Report ------ SW1 ---- Report ---- R1
                      /
Rec3 ---- Report ----/
```

The remaining duplicate Reports are processed by SW1, but not forwarded to R1.

Cisco describes the behavior as forwarding the first IGMPv1/IGMPv2 Report for a group to all multicast-router ports and suppressing the remaining Reports for that group.

## Switch Suppression vs Host Suppression

IGMP snooping changes where Report Suppression occurs.

Without IGMP snooping, IGMPv1 and IGMPv2 hosts can hear one another's Membership Reports:

```text
General Query
      ↓
Rec1, Rec2, Rec3 start random timers
      ↓
Rec1 sends Report first
      ↓
Report is flooded through the VLAN
      ↓
Rec2 and Rec3 hear Rec1's Report
      ↓
Rec2 and Rec3 cancel their pending Reports
```

Therefore:

```text
Without snooping:

Host Report Suppression
→ hosts suppress duplicate Reports themselves
```

With IGMP snooping, the switch needs to learn which individual Layer 2 ports lead toward receivers.

The switch therefore intercepts the hosts' Reports instead of forwarding each Report to the other host-facing ports:

```text
Rec1 → Report → SW1
Rec2 → Report → SW1
Rec3 → Report → SW1
```

Because the hosts do not hear one another's Reports, they do not suppress their own Reports.

SW1 can therefore learn:

```text
G
→ E0/1
→ E0/2
→ E0/3
```

The switch then performs Report Suppression **toward the multicast-router ports**:

```text
Rec1 ---- Report ----\
                      \
Rec2 ---- Report ------ SW1 ---- one Report ---- R1
                      /
Rec3 ---- Report ----/
```

So the suppression function effectively moves from the hosts to the switch:

```text
Without IGMP snooping:
hosts suppress duplicate IGMPv1/v2 Reports

With IGMP snooping Report Suppression:
switch receives Reports from the hosts
→ learns each listener port
→ suppresses duplicate Reports toward multicast routers
```

This behavior is relevant to **IGMPv1 and IGMPv2**, which implement host Report Suppression.

IGMPv3 does **not** use host Report Suppression, so IGMPv3 hosts send their own Reports regardless of Reports sent by other hosts.

## The Switch Still Learns Suppressed Reports

A Report being **suppressed** does not mean the switch ignores it.

Suppose:

```text
Rec1 → E0/1
Rec2 → E0/2
```

Both report membership in:

```text
239.1.1.1
```

SW1 might forward Rec1's Report to R1:

```text
E0/1 Report
→ process
→ forward to mrouter
```

and suppress Rec2's Report upstream:

```text
E0/2 Report
→ process
→ do not forward to mrouter
```

But SW1 still learns:

```text
239.1.1.1
→ E0/1
→ E0/2
```

Therefore:

> Report Suppression affects **Report forwarding toward multicast routers**, not the switch's ability to learn listener membership from the Report.

## Suppression Is Per Group

Report Suppression is performed independently for each multicast group.

Suppose the switch receives:

```text
Rec1 → Report for 239.1.1.1
Rec2 → Report for 239.1.1.1
Rec3 → Report for 239.2.2.2
```

The switch can forward:

```text
Report for 239.1.1.1
Report for 239.2.2.2
```

while suppressing the duplicate second Report for `239.1.1.1`.

So the goal is approximately:

```text
one Report
per group
per multicast router Query
```

not one Report for the entire VLAN.

## Multiple Mrouter Ports

If multiple multicast-router ports exist:

```text
          R1          R2
           \          /
            \        /
               SW1
             /     \
          Rec1     Rec2
```

and SW1 forwards the first Report for group `G`, that Report is forwarded toward the multicast routers.

Conceptually:

```text
Rec1/Rec2
    ↓
   SW1
  /   \
 v     v
R1     R2
```

The duplicate host Reports are suppressed from the multicast-router ports.

Cisco describes the forwarded Report as being sent to **all multicast routers**.

## IGMPv1 and IGMPv2

Catalyst IOS XE Report Suppression applies when the multicast Query involves only IGMPv1/IGMPv2 reporting.

Cisco's documented behavior is:

```text
first IGMPv1/v2 Report for G
→ forward to mrouter ports

remaining IGMPv1/v2 Reports for G
→ suppress toward mrouter ports
```

This mirrors the efficiency goal of native IGMPv1/v2 host Report Suppression, but the suppression decision is now performed by the switch.

## IGMPv3 Is Different

IGMPv3 deliberately removes the host Report Suppression behavior used by IGMPv1 and IGMPv2.

Each IGMPv3 receiver reports its own membership and source-filter state.

For example:

```text
Rec1:
INCLUDE {S1}

Rec2:
INCLUDE {S2}
```

The Reports contain different information:

```text
Rec1 → (S1,G)
Rec2 → (S2,G)
```

Simply discarding one of these Reports could remove source-filter information needed by the multicast router.

Therefore, Catalyst 9000 IGMP Snooping Report Suppression is **not supported when the Query includes IGMPv3 Reports**.

Cisco states that in this situation the switch forwards all IGMPv1, IGMPv2, and IGMPv3 Reports toward the multicast routers.

Conceptually:

```text
IGMPv1/v2:

Rec1 ----\
Rec2 ----- SW1 ---- one Report ---- R1
Rec3 ----/


IGMPv3:

Rec1 -------- Report --------\
Rec2 -------- Report --------- R1
Rec3 -------- Report --------/
```

This allows the router to receive the full source-filter information supplied by each IGMPv3 receiver.

## Configuration

IGMP Snooping Report Suppression is enabled by default on Catalyst IOS XE.

Explicitly enable it with:

```text
SW1(config)# ip igmp snooping report-suppression
```

Disable it with:

```text
SW1(config)# no ip igmp snooping report-suppression
```

When disabled:

```text
all IGMP Reports
→ forwarded toward multicast-router ports
```

However, disabling Report Suppression does **not** disable IGMP snooping and does **not** cause host Reports to be flooded to other host-facing ports.

For example:

```text
Rec1 --- E0/1
Rec2 --- E0/2
R1   --- E0/3
```

If Rec1 and Rec2 both send IGMPv2 Reports:

```text
Report Suppression enabled:

Rec1 → SW1 → R1
Rec2 → SW1    [duplicate Report suppressed toward R1]
```

With Report Suppression disabled:

```text
Rec1 → SW1 → R1
Rec2 → SW1 → R1
```

In both cases, SW1 is still performing IGMP snooping:

```text
Rec1's Report
→ not forwarded to Rec2

Rec2's Report
→ not forwarded to Rec1
```

Therefore:

```text
no ip igmp snooping report-suppression
≠ disable IGMP snooping
≠ flood Reports to all ports

It only disables suppression
toward multicast-router ports.
```

Cisco documents the `no` form as forwarding all IGMP Reports to the multicast routers.

## Verification

Use:

```text
SW3# show ip igmp snooping 
Global IGMP Snooping configuration:
-------------------------------------------
IGMP snooping Oper State     : Enabled
IGMPv3 snooping              : Enabled
Report suppression           : Enabled   <--- SUPPRESSION ENABLED BY DEFAULT
TCN solicit query            : Disabled
Robustness variable          : 2
Last member query count      : 2
Last member query interval   : 1000
Check TTL=1                  : No
Check Router-Alert-Option    : No
. . . 
```

Packet captures on the receiver-facing and router-facing sides are also useful.

With IGMPv2 Report Suppression enabled, you may observe:

```text
Host-facing side:

Rec1 → Report G
Rec2 → Report G
Rec3 → Report G
```

while the mrouter-facing side shows:

```text
SW1 → Report G
```

The switch has processed the individual host membership information but prevented duplicate Reports from reaching the multicast router.

## Comparison

The key difference can be summarized as:

| Behavior | No IGMP Snooping | IGMP Snooping + Report Suppression |
|---|---|---|
| Who suppresses duplicate IGMPv1/v2 Reports? | Hosts | Switch |
| Are Reports used to learn listener ports? | No | Yes |
| Does router need every receiver's v1/v2 Report? | No | No |
| Typical Reports reaching router per group/query | One | One |
| IGMPv3 host Report Suppression | No | No |

The important conceptual difference is:

```text
Without snooping
→ first host Report suppresses other hosts

With snooping
→ switch learns receiver state
→ switch suppresses duplicate Reports toward mrouter ports
```

This lets the switch maintain accurate Layer 2 listener information without unnecessarily sending duplicate IGMPv1/v2 Reports to the multicast router.