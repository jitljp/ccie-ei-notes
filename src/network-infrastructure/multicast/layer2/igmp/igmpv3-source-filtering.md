# IGMPv3 Source Filtering and INCLUDE/EXCLUDE Modes

The major feature introduced by IGMPv3 is **source filtering**.

With IGMPv1 and IGMPv2, a receiver can request:

```text
Receive multicast group G
```

IGMPv3 allows a receiver to express:

```text
Receive group G only from these sources
```

or:

```text
Receive group G from all sources except these sources
```

IGMPv3 represents this reception state as:

```text
(Group, Filter Mode, Source List)
```

The two filter modes are:

```text
INCLUDE
EXCLUDE
```

## INCLUDE Mode

**INCLUDE mode** means:

```text
Receive traffic for group G
only from sources in the source list
```

For example:

```text
Group:       239.1.1.1
Mode:        INCLUDE
Source List: {10.1.1.1, 10.2.2.2}
```

means:

```text
Receive (10.1.1.1, 239.1.1.1)
Receive (10.2.2.2, 239.1.1.1)

Do not receive 239.1.1.1 from other sources
```

Conceptually:

```text
INCLUDE {S1, S2}

S1 → Forward
S2 → Forward
S3 → Do not forward
S4 → Do not forward
```

### Empty INCLUDE List

An empty INCLUDE list means:

```text
INCLUDE {}
```

The receiver wants traffic from **no sources** for the group.

Therefore:

```text
INCLUDE {}
= no multicast reception for G
```

At the host level, changing a group to `INCLUDE {}` effectively removes the multicast reception state for that group.

This is very different from:

```text
EXCLUDE {}
```

which means the opposite.

## EXCLUDE Mode

**EXCLUDE mode** means:

```text
Receive traffic for group G
from all sources except those in the source list
```

For example:

```text
Group:       239.1.1.1
Mode:        EXCLUDE
Source List: {10.1.1.1, 10.2.2.2}
```

means:

```text
Do not receive (10.1.1.1, 239.1.1.1)
Do not receive (10.2.2.2, 239.1.1.1)

Receive 239.1.1.1 from all other sources
```

Conceptually:

```text
EXCLUDE {S1, S2}

S1 → Do not forward
S2 → Do not forward
S3 → Forward
S4 → Forward
```

## Empty EXCLUDE List

An empty EXCLUDE list means:

```text
EXCLUDE {}
```

There are no excluded sources, so the receiver wants:

```text
Group G from all sources
```

This is the IGMPv3 equivalent of the traditional IGMPv1/IGMPv2 ASM group membership:

```text
IGMPv1/v2:
Join G

IGMPv3:
EXCLUDE {}
```

The two empty source lists therefore have completely opposite meanings:

| State | Meaning |
|---|---|
| `INCLUDE {}` | Receive G from no sources |
| `EXCLUDE {}` | Receive G from all sources |

This distinction is fundamental to IGMPv3.

## Source Filtering Applies to Multicast Data

INCLUDE and EXCLUDE filtering applies to **multicast data traffic**.

It does **not** filter IGMP control messages.

IGMP Queries and Reports must still be processed regardless of the source-filter state.

For example:

```text
Group state:
INCLUDE {10.1.1.1}
```

does not mean that IGMP control messages from other addresses should be discarded.

The source list describes the desired sources of **multicast traffic for the group**, not permitted IGMP speakers.

## Host Socket State

IGMPv3's source-filtering model begins with applications.

Conceptually, each application socket can request multicast reception in the form:

```text
(interface, group, filter mode, source list)
```

For example:

```text
Application A:
239.1.1.1 INCLUDE {S1, S2}

Application B:
239.1.1.1 INCLUDE {S2, S3}
```

Both applications are using the same interface and group, but they have different source requirements.

The host must combine these application-level requests into **one interface-level reception state** to advertise using IGMP.

## Combining Multiple INCLUDE Lists

If every application requesting a group is in INCLUDE mode, the host's interface-level source list is the **union** of all INCLUDE lists.

For example:

```text
Application A:
INCLUDE {S1, S2}

Application B:
INCLUDE {S2, S3}

Application C:
INCLUDE {S4}
```

The interface state becomes:

```text
INCLUDE {S1, S2, S3, S4}
```

This is necessary because the interface must receive every source requested by at least one application.

Conceptually:

```text
INCLUDE interface state
= all sources requested by any INCLUDE socket
```

## Combining EXCLUDE and INCLUDE State

The calculation is more interesting if one or more applications use EXCLUDE mode.

If **any** socket is in EXCLUDE mode, the resulting interface state is also EXCLUDE mode.

Consider:

```text
Application A:
EXCLUDE {S1, S2, S3, S4}

Application B:
EXCLUDE {S2, S3, S4, S5}

Application C:
INCLUDE {S4, S5, S6}
```

First consider what both EXCLUDE applications agree should be excluded:

```text
{S1,S2,S3,S4}
        ∩
{S2,S3,S4,S5}

= {S2,S3,S4}
```

However, Application C explicitly wants:

```text
{S4,S5,S6}
```

Therefore, `S4` cannot remain excluded.

The resulting interface state is:

```text
EXCLUDE {S2, S3}
```

In general:

```text
Interface EXCLUDE list
=
sources excluded by every EXCLUDE socket
minus
sources requested by any INCLUDE socket
```

This ensures that traffic required by **any application** is still received.

## EXCLUDE {} Dominates

Suppose another application requests:

```text
EXCLUDE {}
```

This means:

```text
Receive all sources
```

Since one application wants every source, the interface must receive every source.

The resulting interface state becomes:

```text
EXCLUDE {}
```

regardless of the more restrictive requirements of the other applications.

This illustrates an important source-filtering principle:

> The aggregate state must allow enough traffic to satisfy **all receivers**.

It is acceptable for an individual application to receive traffic at the interface that it does not want, because the host can perform additional filtering before delivering packets to that socket.

It is not acceptable to prevent traffic that another application requested from reaching the interface at all.

## Host State vs Router State

Do not confuse the reception state maintained by a host with the state maintained by the multicast router.

A host might advertise:

```text
239.1.1.1
INCLUDE {S1, S2}
```

but the router must combine that with the Reports from **all hosts on the LAN**.

For example:

```text
Rec1:
INCLUDE {S1}

Rec2:
INCLUDE {S2}
```

The router needs to forward:

```text
S1
S2
```

onto the LAN.

Its aggregate router state therefore represents:

```text
INCLUDE {S1, S2}
```

The router maintains this state **per group, per attached network**.

Conceptually:

```text
(Group,
 Group Timer,
 Router Filter Mode,
 Source Records)
```

Each source record has its own **Source Timer**.

## Router INCLUDE Mode

When the router's filter mode for a group is INCLUDE, its source records represent sources that receivers on the LAN want.

For example:

```text
Router Filter Mode: INCLUDE

S1 timer > 0
S2 timer > 0
```

means receivers currently want:

```text
(S1,G)
(S2,G)
```

The router therefore indicates to the multicast routing protocol (PIM) that these sources should be forwarded onto the network.

If the timer for `S1` expires:

```text
S1 timer → 0
```

the router concludes that no receiver still wants `S1`.

The source record can then be removed.

If there are no remaining source records in INCLUDE mode, there is no remaining receiver interest in the group.

## Router EXCLUDE Mode

If the router receives an EXCLUDE-mode membership for a group, its aggregate **Router Filter Mode** becomes EXCLUDE.

Unlike a host, however, the router does not represent EXCLUDE state with a single source list such as:

```text
EXCLUDE {S1,S2}
```

The router must combine the requirements of **all receivers on the LAN**. RFC 9776 therefore represents router EXCLUDE state as:

```text
EXCLUDE (X,Y)
```

where `X` and `Y` are sets of **source records**:

```text
X = sources with Source Timer > 0
    → IGMP state indicates at least one receiver still wants the source
    → forward

Y = sources with Source Timer = 0
    → no receiver is currently known to want the source
    → do not forward
```

The **Source Timer is how the router remembers receiver interest in a particular source**. When Reports indicate that a source should be received, IGMP processing starts or refreshes that source's timer. As long as the timer is running, the router treats that source as still desired.

In EXCLUDE mode, a source can therefore be in one of three states:

```text
Source in X
→ Source Timer > 0
→ Forward

Source in Y
→ Source Timer = 0
→ Do not forward

Source not in X or Y
→ Forward, because the group is in EXCLUDE mode
```

For example:

```text
EXCLUDE (
  X = {S1},
  Y = {S2,S3}
)
```

means:

```text
S1       → Forward        (running Source Timer)
S2       → Do not forward (Source Timer = 0)
S3       → Do not forward (Source Timer = 0)
S4, S5…  → Forward        (not excluded)
```

`X` is particularly important when INCLUDE-mode and EXCLUDE-mode receivers coexist on the same LAN.

### Example: INCLUDE and EXCLUDE Receivers

Suppose the router initially receives:

```text
Rec1:
EXCLUDE {S1}
```

Rec1 wants:

```text
Every source except S1
```

If Rec1 is the only receiver, the router can represent the aggregate state as:

```text
EXCLUDE (
  X = {},
  Y = {S1}
)
```

`S1` has a Source Timer of `0`, so:

```text
S1       → Do not forward
S2, S3…  → Forward
```

Now another receiver reports:

```text
Rec2:
INCLUDE {S1}
```

Rec2 explicitly wants `S1`.

When the router processes this Report, it starts or refreshes the Source Timer for `S1`. `S1` therefore moves from `Y` to `X`:

```text
EXCLUDE (
  X = {S1},
  Y = {}
)

S1 Source Timer > 0
```

The resulting forwarding behavior is:

```text
S1       → Forward   (receiver interest represented by its running Source Timer)
S2, S3…  → Forward   (permitted by the EXCLUDE state)
```

Therefore, **all sources are currently forwarded onto the LAN**.

However, the router still keeps a source record for `S1` with a running timer. This records the fact that receiver interest in `S1` must survive even if the EXCLUDE-mode receiver later disappears.

Thus:

```text
EXCLUDE (
  X = {S1},
  Y = {}
)
```

currently results in all sources being forwarded, just as an unrestricted EXCLUDE membership would.

However, the router's state contains additional information:

```text
S1 Source Timer > 0
```

indicating that `S1` must continue to be forwarded independently of the EXCLUDE-mode membership.

### Transition Back to INCLUDE Mode

While the router is in EXCLUDE mode, it also maintains a **Group Timer**.

A running Group Timer indicates that at least one EXCLUDE-mode receiver is still present.

In the previous example:

```text
Router Filter Mode: EXCLUDE
Group Timer > 0

X = {S1}
Y = {}
```

If Rec1 disappears and no EXCLUDE-mode receiver refreshes the Group Timer, the Group Timer eventually expires.

The router can then conclude that:

```text
No EXCLUDE-mode receivers remain
```

However, `S1` still has a running Source Timer, indicating that a receiver still wants that source.

The router therefore transitions from:

```text
EXCLUDE (
  X = {S1},
  Y = {}
)
```

to:

```text
INCLUDE {S1}
```

In general:

```text
EXCLUDE (X,Y)
      |
      | Group Timer expires
      v
INCLUDE (X)
```

Sources in `X` are retained because their Source Timers are still running.

Sources in `Y` are discarded because their Source Timers are `0`.

This is the main reason the router keeps `X` separately while in EXCLUDE mode: it preserves source-specific receiver interest so that the router can transition from EXCLUDE mode back to INCLUDE mode without interrupting traffic still requested by receivers.

The exact Source Timer updates and state transitions caused by each IGMPv3 Group Record type are covered on the **IGMPv3 State Changes** page.

## Forwarding Summary

The basic forwarding interpretation of router state is:

| Router Filter Mode | Source State | IGMP Recommendation |
|---|---|---|
| INCLUDE | Source Timer > 0 | Forward source |
| INCLUDE | No source record | Do not forward source |
| EXCLUDE | Source Timer > 0 | Forward source |
| EXCLUDE | Source Timer = 0 | Do not forward source |
| EXCLUDE | No source record | Forward source |

The last row explains why:

```text
EXCLUDE {}
```

means:

```text
Forward all sources
```

No source appears in the exclusion state, so all sources are permitted.

> IGMP provides receiver-interest information to the multicast routing protocol. It does not by itself override multicast routing behavior on transit networks.

## INCLUDE Mode and SSM

**Source-Specific Multicast (SSM)** uses INCLUDE-mode semantics.

For example:

```text
Group:  232.1.1.1
Source: 10.1.1.1
```

is expressed as:

```text
INCLUDE {10.1.1.1}
```

This directly identifies the desired `(S,G)` channel:

```text
(10.1.1.1, 232.1.1.1)
```

An SSM-aware host should not use EXCLUDE-mode records for groups in the SSM range, and an SSM-aware router should ignore EXCLUDE-mode records for SSM groups.

With:

```text
R1(config)# ip pim ssm default
```

the default IOS XE SSM range is:

```text
232.0.0.0/8
```

Therefore:

```text
232.1.1.1 → PIM-SSM by default
239.1.1.1 → not PIM-SSM by default
```

IGMPv3 source filtering itself is **not limited to the SSM range**.

For example, a receiver can still advertise:

```text
239.1.1.1
INCLUDE {10.1.1.1}
```

However, this does not automatically make `239.1.1.1` a PIM-SSM group.

Whether the multicast network uses the SSM forwarding model depends on whether the group falls within the configured **PIM SSM range**.

## IOS XE INCLUDE-Mode Receiver

Configure IGMPv3 on the receiver:

```text
Rec1(config)# interface Ethernet0/0
Rec1(config-if)# ip igmp version 3
```

Then configure source-specific membership:

```text
Rec1(config-if)# ip igmp join-group 239.1.1.1 source 10.1.2.10
```

This causes Rec1 to request:

```text
(10.1.2.10, 239.1.1.1)
```

using INCLUDE-mode IGMPv3 signaling.

On the multicast router:

```text
R3# show ip igmp groups 239.1.1.1 detail
```

might display:

```text
Interface:      Ethernet0/1
Group:          239.1.1.1
Group mode:     INCLUDE
Last reporter:  10.3.4.10

Group source list:
  Source Address
  10.1.2.10
```

This means the aggregate receiver state on Ethernet0/1 includes:

```text
INCLUDE {10.1.2.10}
```

for `239.1.1.1`.

> When using `ip igmp join-group <group> source <source>`, configure `ip igmp version 3`. Without IGMPv3 enabled, IOS XE can create local `(S,G)` state but does not send the corresponding IGMPv3 Membership Report.

## IOS XE ASM-Style Membership

A normal group membership without a source can be configured with:

```text
Rec1(config)# interface Ethernet0/0
Rec1(config-if)# ip igmp version 3
Rec1(config-if)# ip igmp join-group 239.1.1.1
```

For IGMPv3, ordinary any-source group membership is represented using:

```text
EXCLUDE {}
```

On the multicast router:

```text
R3# show ip igmp groups 239.1.1.1 detail
```

can therefore display:

```text
Group mode:     EXCLUDE
Source list is empty
```

which means:

```text
Receive 239.1.1.1 from all sources
```

## INCLUDE vs EXCLUDE Summary

| State | Meaning |
|---|---|
| `INCLUDE {S1,S2}` | Receive only S1 and S2 |
| `INCLUDE {}` | Receive no sources |
| `EXCLUDE {S1,S2}` | Receive all sources except S1 and S2 |
| `EXCLUDE {}` | Receive all sources |

The simplest mental model is:

```text
INCLUDE list
= sources I WANT

EXCLUDE list
= sources I DO NOT WANT
```

But remember that the router must aggregate the requirements of **all receivers** on the LAN.

Therefore:

```text
If any receiver needs a source,
the router must still forward that source.
```

The next sections examine how these filter states are represented by the individual **IGMPv3 Group Record Types** and how changes in filter mode or source lists are signaled.