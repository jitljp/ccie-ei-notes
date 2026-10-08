# IGMPv3 State Changes

IGMPv3 uses **State-Change Reports** whenever a host's multicast reception state changes.

Examples include:

```text
Joining a new (S,G)
Leaving an (S,G)
Joining an ASM group
Leaving an ASM group
Adding or removing sources
Changing between INCLUDE and EXCLUDE mode
```

Unlike Current-State Records, which answer Queries and describe the host's current state, State-Change Records communicate:

```text
"What just changed?"
```

The four State-Change Record types are:

| Class | Record Type |
|---|---|
| Filter-Mode-Change | `CHANGE_TO_INCLUDE_MODE` |
| Filter-Mode-Change | `CHANGE_TO_EXCLUDE_MODE` |
| Source-List-Change | `ALLOW_NEW_SOURCES` |
| Source-List-Change | `BLOCK_OLD_SOURCES` |

## Determining the State-Change Record

The host compares its **old interface state** with its **new interface state**.

If no state previously existed for the group, the old state is treated as:

```text
INCLUDE {}
```

Likewise, deleting the final membership for a group results conceptually in:

```text
INCLUDE {}
```

This allows joins and leaves to use the same state-transition rules as other changes.

### Set Notation Used Below

The IGMPv3 state-transition rules use **sets of source addresses**.

For example:

```text
A = {S1,S2}
B = {S2,S3}
```

The following set operations are used:

```text
A ∪ B = union
        → sources that are in A, B, or both
        → {S1,S2,S3}

A ∩ B = intersection
        → sources that are in both A and B
        → {S2}

A - B = difference
        → sources that are in A but not B
        → {S1}

B - A = difference in the other direction
        → sources that are in B but not A
        → {S3}
```

So if:

```text
Old state: INCLUDE(A)
New state: INCLUDE(B)
```

then:

```text
B - A
```

means:

```text
sources that are in the new INCLUDE list
but were not in the old INCLUDE list
```

These are newly wanted sources, so the host sends:

```text
ALLOW(B-A)
```

Likewise:

```text
A - B
```

means:

```text
sources that were in the old INCLUDE list
but are not in the new INCLUDE list
```

These sources are no longer wanted, so the host sends:

```text
BLOCK(A-B)
```

RFC 9776 defines the basic transitions as:

| Old State | New State | Record Sent |
|---|---|---|
| `INCLUDE(A)` | `INCLUDE(B)` | `ALLOW(B-A)` and/or `BLOCK(A-B)` |
| `EXCLUDE(A)` | `EXCLUDE(B)` | `ALLOW(A-B)` and/or `BLOCK(B-A)` |
| `INCLUDE(A)` | `EXCLUDE(B)` | `CHANGE_TO_EXCLUDE_MODE(B)` |
| `EXCLUDE(A)` | `INCLUDE(B)` | `CHANGE_TO_INCLUDE_MODE(B)` |

If an `ALLOW` or `BLOCK` source set is empty, that Group Record is omitted.

## INCLUDE Source Changes

Suppose the host changes from:

```text
INCLUDE {S1}
```

to:

```text
INCLUDE {S1,S2}
```

`S2` has become wanted, so the host sends:

```text
ALLOW_NEW_SOURCES {S2}
```

If the state instead changes from:

```text
INCLUDE {S1,S2}
```

to:

```text
INCLUDE {S1}
```

`S2` is no longer wanted:

```text
BLOCK_OLD_SOURCES {S2}
```

If both occur simultaneously:

```text
Old:
INCLUDE {S1,S2}

New:
INCLUDE {S2,S3}
```

the host sends:

```text
ALLOW_NEW_SOURCES {S3}
BLOCK_OLD_SOURCES {S1}
```

in the same Membership Report.

## EXCLUDE Source Changes

The meaning of adding or removing an address from the source list is reversed in EXCLUDE mode.

Suppose:

```text
Old:
EXCLUDE {S1,S2}

New:
EXCLUDE {S1}
```

`S2` has been removed from the exclusion list, so traffic from `S2` is now wanted:

```text
ALLOW_NEW_SOURCES {S2}
```

Conversely:

```text
Old:
EXCLUDE {S1}

New:
EXCLUDE {S1,S2}
```

means `S2` has become blocked:

```text
BLOCK_OLD_SOURCES {S2}
```

Therefore:

```text
INCLUDE:
add source    → ALLOW
remove source → BLOCK

EXCLUDE:
remove source → ALLOW
add source    → BLOCK
```

## Filter-Mode Changes

If the interface changes between INCLUDE and EXCLUDE mode, a **Filter-Mode-Change Record** is used instead of `ALLOW`/`BLOCK`.

For example:

```text
Old:
INCLUDE {S1}

New:
EXCLUDE {S2}
```

results in:

```text
CHANGE_TO_EXCLUDE_MODE {S2}
```

The listed sources represent the **complete new EXCLUDE list**.

Similarly:

```text
Old:
EXCLUDE {S1}

New:
INCLUDE {S2,S3}
```

results in:

```text
CHANGE_TO_INCLUDE_MODE {S2,S3}
```

The source list is the complete new INCLUDE list.

## Source-Specific Join

A host with no previous membership for a group is treated as:

```text
INCLUDE {}
```

Suppose the host joins:

```text
(S1,G)
```

Its state changes:

```text
INCLUDE {}
    ↓
INCLUDE {S1}
```

The filter mode remains INCLUDE, so the host sends:

```text
ALLOW_NEW_SOURCES {S1}
```

Therefore, a new source-specific join typically appears as:

```text
ALLOW_NEW_SOURCES
```

## Source-Specific Leave

Suppose the host's only source-specific membership is:

```text
INCLUDE {S1}
```

and it leaves `(S1,G)`.

Its new state becomes:

```text
INCLUDE {}
```

so:

```text
INCLUDE {S1}
    ↓
INCLUDE {}
```

results in:

```text
BLOCK_OLD_SOURCES {S1}
```

The multicast router may then verify that no other receiver still wants `(S1,G)` before stopping the traffic.

## ASM Join

An ordinary any-source membership is represented by:

```text
EXCLUDE {}
```

A new ASM join therefore changes:

```text
INCLUDE {}
    ↓
EXCLUDE {}
```

The host sends:

```text
CHANGE_TO_EXCLUDE_MODE {}
```

So in a packet capture, an IGMPv3 ASM join commonly appears as:

```text
CHANGE_TO_EXCLUDE_MODE
Number of Sources = 0
```

## ASM Leave

Leaving an ASM membership reverses the state:

```text
EXCLUDE {}
    ↓
INCLUDE {}
```

The host therefore sends:

```text
CHANGE_TO_INCLUDE_MODE {}
```

IGMPv3 does not need the separate IGMPv2:

```text
Leave Group
Type = 0x17
```

message for native IGMPv3 operation.

The leave is expressed through the change in source-filter state.

## State-Change Reports Are Unsolicited

A State-Change Report is sent **immediately** when the host's interface state changes.

It is not sent in response to a Query.

For example:

```text
Application joins (S1,G)
        ↓
Host state changes
        ↓
ALLOW_NEW_SOURCES {S1}
        ↓
IGMPv3 Membership Report sent immediately
```

This differs from Current-State Records:

```text
Query received
      ↓
Random response delay
      ↓
MODE_IS_INCLUDE / MODE_IS_EXCLUDE
```

## State-Change Report Retransmission

Because an unsolicited State-Change Report could be lost, the host retransmits the state change.

The total number of transmissions is controlled by the **Robustness Variable**.

With the default:

```text
Robustness Variable = 2
```

the host sends:

```text
Initial State-Change Report
        ↓
One retransmission
```

In general:

```text
Total State-Change Reports
= Robustness Variable
```

or equivalently:

```text
Initial transmission
+
(Robustness Variable - 1) retransmissions
```

The retransmission delay is randomly selected from:

```text
0 to Unsolicited Report Interval
```

The RFC default IGMPv3 Unsolicited Report Interval is:

```text
1 second
```

This provides robustness against lost State-Change Reports.

## Changes During Retransmission

A host does not simply continue retransmitting an obsolete state change if the membership changes again.

Suppose:

```text
1. Host joins S1
   → ALLOW {S1}

2. Before retransmission finishes,
   host also joins S2
```

The host immediately sends another State-Change Report and merges the new information with the still-pending retransmission state.

The new Report becomes the start of a new Robustness Variable sequence.

This prevents rapid application changes from defeating IGMP's packet-loss protection.

## Filter-Mode Changes Take Priority

Filter-mode changes require special handling during retransmission.

Suppose the host changes:

```text
INCLUDE → EXCLUDE
```

and then its source list changes again before all retransmissions of the `CHANGE_TO_EXCLUDE_MODE` record have completed.

The host continues sending a Filter-Mode-Change Record for the required Robustness Variable transmissions.

The record reflects the **current** EXCLUDE source list.

Conceptually:

```text
CHANGE_TO_EXCLUDE_MODE
        ↓
must be transmitted RV times
        ↓
later source-list changes are incorporated
into the current state
```

Only after the filter-mode change has been transmitted robustly can pending source-list changes be represented purely with `ALLOW_NEW_SOURCES` or `BLOCK_OLD_SOURCES`.

## Router Processing of State Changes

A State-Change Report tells the multicast router that some previously desired traffic may no longer be wanted.

The router cannot immediately stop forwarding that traffic because **other receivers on the same LAN may still want it**.

For example:

```text
Rec1 leaves (S1,G)

Rec2 may still want (S1,G)
```

The router therefore uses IGMP Queries to verify whether receiver interest remains.

The two relevant Query types are:

```text
Q(G)
→ Group-Specific Query

Q(G,{S})
→ Group-and-Source-Specific Query
```

Group-and-Source-Specific Queries are generated in response to State-Change Records when the router needs to verify particular sources. They are not generated in response to ordinary Current-State Reports.

## Traffic Continues During Verification

When a router receives a state change indicating that traffic might no longer be wanted, the router does **not immediately prune that traffic**.

Instead:

```text
State-Change Report
        ↓
Send specific Queries
        ↓
Wait LMQT
        ↓
No remaining interest?
        ↓
Stop forwarding
```

During the Last Member Query period, the router continues suggesting that the multicast routing protocol forward the affected traffic.

Only after the verification period expires without a Report showing continued interest may the traffic be pruned.

## Last Member Query Time

The verification period is the **Last Member Query Time (LMQT)**:

```text
LMQT = LMQI × LMQC
```

where:

```text
LMQI = Last Member Query Interval
LMQC = Last Member Query Count
```

RFC defaults are:

```text
LMQI = 1 second
LMQC = Robustness Variable = 2

LMQT = 2 seconds
```

The specific Query is therefore normally transmitted twice, one second apart.

IOS XE exposes these parameters with:

```text
R1(config-if)# ip igmp last-member-query-interval 500
R1(config-if)# ip igmp last-member-query-count 3
```

The interval is configured in milliseconds.

## Source Timer Verification

Suppose the router is in:

```text
INCLUDE {S1,S2}
```

and receives:

```text
BLOCK_OLD_SOURCES {S1}
```

The router cannot assume that `S1` is no longer wanted, because another receiver may still want it.

The querier sends:

```text
Q(G,{S1})
```

and the Source Timer for `S1` is lowered to:

```text
LMQT
```

If another host responds indicating interest in `S1`, its Source Timer is refreshed.

If nobody responds before the timer expires:

```text
S1 Source Timer → 0
```

the router concludes that `(S1,G)` is no longer wanted and can stop forwarding it.

Conceptually:

```text
BLOCK {S1}
     ↓
Q(G,{S1})
     ↓
Source Timer → LMQT
     ↓
Report for S1?
   /        \
 yes        no
  |          |
refresh    timer expires
timer        |
          stop forwarding S1
```

## Group Timer Verification

The same principle applies to EXCLUDE-mode group membership.

Suppose a router has an EXCLUDE-mode membership:

```text
EXCLUDE (X,Y)
```

Here, `X` and `Y` are **router-internal source sets** used while the router is in EXCLUDE mode. They are not source lists copied directly from a single host Report.

```text
X = sources with Source Timer > 0
    → IGMP state indicates the source is still desired
    → forward

Y = sources with Source Timer = 0
    → no receiver is currently known to want the source
    → do not forward
```

Any source that is in **neither `X` nor `Y`** is also forwarded while the router is in EXCLUDE mode.

For example:

```text
EXCLUDE (
  X = {S1},
  Y = {S2}
)
```

means:

```text
S1       → Forward        (Source Timer > 0)
S2       → Do not forward (Source Timer = 0)
S3, S4…  → Forward        (not excluded)
```

The router also maintains a separate **Group Timer**. While the Group Timer is running, the router believes that at least one EXCLUDE-mode receiver is still present on the LAN.

If the router receives a change indicating that an EXCLUDE-mode receiver may have left, the querier may send:

```text
Q(G)
```

When a router sends or receives a Group-Specific Query with `S = 0`, the Group Timer is lowered to:

```text
LMQT
```

If another EXCLUDE-mode receiver responds, the Group Timer is refreshed.

If no EXCLUDE-mode receiver responds before the Group Timer expires:

```text
Group Timer → 0
```

the router concludes:

```text
No EXCLUDE-mode receivers remain
```

Therefore, the router no longer needs to remain in EXCLUDE mode.

However, sources in `X` still have running Source Timers, meaning IGMP state indicates that those sources are still wanted by receivers. The router therefore keeps `X` and changes to INCLUDE mode:

```text
EXCLUDE (X,Y)
      ↓
Group Timer expires
      ↓
INCLUDE (X)
```

Sources in `Y` are discarded because their Source Timers are already `0`.

Sources in `X` remain because their Source Timers are still running.

For example:

```text
Before:

EXCLUDE (
  X = {S1},
  Y = {S2}
)

Group Timer > 0
```

If the Group Timer expires:

```text
After:

INCLUDE {S1}
```

The forwarding behavior changes from:

```text
S1       → Forward
S2       → Do not forward
S3, S4…  → Forward
```

to:

```text
S1       → Forward
S2, S3…  → Do not forward
```

because the EXCLUDE-mode receiver that implicitly wanted the unlisted sources is no longer present. Only the sources with still-running Source Timers in `X` remain wanted.

## The S Flag and Timer Updates

Specific Queries also carry the IGMPv3 **S flag**:

```text
S = Suppress Router-Side Processing
```

When a router sends or receives:

```text
Q(G)
```

with:

```text
S = 0
```

the Group Timer is lowered to LMQT.

When it sends or receives:

```text
Q(G,A)
```

with:

```text
S = 0
```

the Source Timers for the queried sources are lowered to LMQT.

However:

```text
S = 1
```

means:

```text
Hosts still process and answer the Query

Other routers do not modify their
Group/Source Timers because of it
```

This allows the querier to ask receivers about membership without unnecessarily shortening timer state maintained by other routers.

## Router State Notation

The router state-transition rules use:

```text
INCLUDE (A)
```

where:

```text
A = sources with running Source Timers
```

In EXCLUDE mode:

```text
EXCLUDE (X,Y)
```

where:

```text
X = sources with Source Timer > 0
    → IGMP state indicates the source is still desired
    → forward

Y = sources with Source Timer = 0
    → no receiver is currently known to want the source
    → do not forward
```

Sources not represented in either set are also forwarded while the group is in EXCLUDE mode.

The same set notation introduced earlier is used in the router state-transition rules:

```text
A ∪ B = all sources in either set
A ∩ B = only sources present in both sets
A - B = sources in A but not in B
```

For example, if:

```text
A = {S1,S2}
B = {S2,S3}
```

then:

```text
A ∪ B = {S1,S2,S3}
A ∩ B = {S2}
A - B = {S1}
```

## Router in INCLUDE Mode

The following transitions show how a router with:

```text
INCLUDE (A)
```

processes State-Change Records.

### Receiving ALLOW(B)

```text
INCLUDE (A)
+
ALLOW (B)
```

becomes:

```text
INCLUDE (A ∪ B)
```

and the Source Timers for `B` are set to the **Group Membership Interval (GMI)**.

There is no need to verify these sources because the Report explicitly says that they are wanted.

### Receiving BLOCK(B)

The router retains:

```text
INCLUDE (A)
```

because another receiver may still want the blocked sources.

The querier checks only sources that are both:

```text
currently being forwarded
AND
listed in BLOCK
```

so it sends:

```text
Q(G, A ∩ B)
```

Those Source Timers are reduced through the Last Member Query process.

### Receiving CHANGE_TO_INCLUDE_MODE(B)

The state becomes:

```text
INCLUDE (A ∪ B)
```

and Source Timers for `B` are refreshed.

However, sources that were previously wanted but are missing from the new INCLUDE list might have belonged only to the receiver that changed mode.

The querier therefore verifies:

```text
A - B
```

with:

```text
Q(G, A-B)
```

### Receiving CHANGE_TO_EXCLUDE_MODE(B)

The router must immediately change its Router Filter Mode to EXCLUDE because at least one receiver now wants a potentially unrestricted set of sources.

The state becomes conceptually:

```text
EXCLUDE (
  X = A ∩ B,
  Y = B - A
)
```

Why?

Sources in:

```text
A ∩ B
```

were already being requested by some receiver but are now listed as excluded by the new receiver.

They therefore remain in `X` with running timers until the router determines whether other receivers still want them.

Sources in:

```text
B - A
```

were not previously being requested and are excluded by the new receiver, so they enter `Y`.

The querier verifies:

```text
A ∩ B
```

with a Group-and-Source-Specific Query.

The Group Timer is set to GMI because an EXCLUDE-mode receiver is now known to exist.

## Router in EXCLUDE Mode

Now consider a router already maintaining:

```text
EXCLUDE (X,Y)
```

### Receiving ALLOW(A)

The sources in `A` are now explicitly wanted.

The router moves them into the running-timer set:

```text
EXCLUDE (
  X = X ∪ A,
  Y = Y - A
)
```

and sets their Source Timers to GMI.

No verification Query is needed because the new Report explicitly confirms interest.

### Receiving BLOCK(A)

A receiver says that sources in `A` are no longer wanted.

The router cannot immediately stop those sources because other receivers may still want them.

The relevant sources are queried with:

```text
Q(G, A-Y)
```

Sources already in `Y` have timer 0 and are already considered unwanted, so there is no reason to query them again.

### Receiving CHANGE_TO_EXCLUDE_MODE(A)

Another receiver has entered EXCLUDE mode.

The Group Timer is refreshed to GMI.

Sources that the new EXCLUDE receiver allows may cause old exclusion state to be removed, while sources it excludes must be merged with the existing state.

Potentially affected sources are verified with:

```text
Q(G, A-Y)
```

The exact set calculations are defined by the RFC router state machine; the important operational point is that another EXCLUDE receiver **refreshes the Group Timer** rather than causing the router to leave EXCLUDE mode.

### Receiving CHANGE_TO_INCLUDE_MODE(A)

One receiver has switched from EXCLUDE to INCLUDE.

However, the router cannot immediately change its aggregate mode to INCLUDE because **another receiver on the LAN may still be in EXCLUDE mode**.

The router therefore remains:

```text
EXCLUDE
```

temporarily.

Sources in `A` are explicitly wanted, so their timers are refreshed.

The querier also sends:

```text
Q(G)
```

to determine whether any EXCLUDE-mode receiver remains.

It may additionally send Group-and-Source-Specific Queries for sources whose continued interest must be verified.

If no EXCLUDE-mode receiver responds:

```text
Group Timer expires
        ↓
EXCLUDE (X,Y)
        ↓
INCLUDE (X)
```

This is how the router safely returns from EXCLUDE mode to INCLUDE mode.

## Router State-Change Summary

The most important logic is:

```text
ALLOW
→ positive evidence of receiver interest
→ start/refresh Source Timer

BLOCK
→ possible loss of source interest
→ verify with Group-and-Source-Specific Query

CHANGE_TO_EXCLUDE_MODE
→ an EXCLUDE receiver now exists
→ enter/refresh EXCLUDE state

CHANGE_TO_INCLUDE_MODE
→ an EXCLUDE receiver may have disappeared
→ verify group and/or source state before pruning
```

The core principle is:

> A positive membership indication can be acted on immediately, but a negative indication must usually be verified because other receivers may still want the traffic.

## Multiple Routers on the LAN

Only the elected IGMP **Querier** originates the specific Queries used for state-change verification.

However, all IGMPv3 multicast routers on the LAN maintain membership state.

Therefore, when the querier sends:

```text
Q(G)
```

or:

```text
Q(G,A)
```

the other routers hear those Queries as well.

When the Query has:

```text
S flag = 0
```

they lower the corresponding Group or Source Timers to LMQT just like the querier.

This keeps the routers' membership state synchronized even though only one router is responsible for transmitting Queries.

## IOS XE Lab Examples

### Source-Specific Join

Configure:

```text
Rec1(config)# interface Ethernet0/0
Rec1(config-if)# ip igmp version 3
Rec1(config-if)# ip igmp join-group 239.1.1.1 source 10.1.2.10
```

A new source-specific membership changes:

```text
INCLUDE {}
    ↓
INCLUDE {10.1.2.10}
```

so the initial State-Change Report can contain:

```text
ALLOW_NEW_SOURCES
Group: 239.1.1.1
Source: 10.1.2.10
```

Remove the membership:

```text
Rec1(config-if)# no ip igmp join-group 239.1.1.1 source 10.1.2.10
```

and the state becomes:

```text
INCLUDE {10.1.2.10}
    ↓
INCLUDE {}
```

resulting in:

```text
BLOCK_OLD_SOURCES {10.1.2.10}
```

### ASM Join

Configure:

```text
Rec1(config)# interface Ethernet0/0
Rec1(config-if)# ip igmp version 3
Rec1(config-if)# ip igmp join-group 239.1.1.1
```

The host changes:

```text
INCLUDE {}
    ↓
EXCLUDE {}
```

and sends:

```text
CHANGE_TO_EXCLUDE_MODE {}
```

Removing the membership:

```text
Rec1(config-if)# no ip igmp join-group 239.1.1.1
```

changes:

```text
EXCLUDE {}
    ↓
INCLUDE {}
```

and can generate:

```text
CHANGE_TO_INCLUDE_MODE {}
```

## Verification

Use:

```text
R1# show ip igmp groups
```

and:

```text
R1# show ip igmp groups 239.1.1.1 detail
```

to inspect the resulting router filter mode and source state.

For protocol-level observation:

```text
R1# debug ip igmp
```

A packet capture is especially useful because it lets you observe:

```text
CHANGE_TO_INCLUDE_MODE
CHANGE_TO_EXCLUDE_MODE
ALLOW_NEW_SOURCES
BLOCK_OLD_SOURCES

Group-Specific Queries

Group-and-Source-Specific Queries

S flag

Max Resp Code

Query source lists
```

## State-Change Process Summary

A complete IGMPv3 state change can be viewed as:

```text
Host reception state changes
        ↓
Host immediately sends State-Change Report
        ↓
Report retransmitted for robustness
        ↓
Router processes positive membership immediately
        ↓
Potential loss of membership?
        ↓
Querier sends specific Query
        ↓
Relevant timer lowered to LMQT
        ↓
Other receiver responds?
      /                 \
    Yes                  No
     |                    |
Refresh timer        Timer expires
     |                    |
Keep forwarding      Stop forwarding
                     or change filter mode
```

This provides IGMPv3 with both **fast leave behavior** and **source-specific membership tracking** without allowing one receiver's state change to accidentally interrupt multicast traffic still required by another receiver.