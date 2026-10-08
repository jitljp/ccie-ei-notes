# IGMPv3 Group Record Types

Each IGMPv3 Membership Report contains one or more **Group Records**.

A Group Record describes membership information for a single multicast group:

```text
Group Record
├── Record Type
├── Multicast Group
└── Source List
```

IGMPv3 defines **six Group Record Types**, divided into three classes:

| Class | Record Type | Value |
|---|---|---:|
| Current-State | `MODE_IS_INCLUDE` | 1 |
| Current-State | `MODE_IS_EXCLUDE` | 2 |
| Filter-Mode-Change | `CHANGE_TO_INCLUDE_MODE` | 3 |
| Filter-Mode-Change | `CHANGE_TO_EXCLUDE_MODE` | 4 |
| Source-List-Change | `ALLOW_NEW_SOURCES` | 5 |
| Source-List-Change | `BLOCK_OLD_SOURCES` | 6 |

The two change classes are collectively called **State-Change Records**:

```text
Group Records
├── Current-State Records
│   ├── MODE_IS_INCLUDE
│   └── MODE_IS_EXCLUDE
│
└── State-Change Records
    ├── Filter-Mode-Change Records
    │   ├── CHANGE_TO_INCLUDE_MODE
    │   └── CHANGE_TO_EXCLUDE_MODE
    │
    └── Source-List-Change Records
        ├── ALLOW_NEW_SOURCES
        └── BLOCK_OLD_SOURCES
```

## Group Record Format

Each Group Record uses the following format:

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Record Type  |  Aux Data Len |     Number of Sources (N)     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Multicast Address                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address [1]                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               .                               |
.                               .                               .
|                               .                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address [N]                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
.                         Auxiliary Data                        .
.                                                               .
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The **Record Type** determines how the source list should be interpreted.

This is important because the source list does **not always represent the complete desired source list**.

For example:

```text
MODE_IS_INCLUDE {S1,S2}
```

describes the receiver's complete current INCLUDE list.

But:

```text
ALLOW_NEW_SOURCES {S2}
```

describes only a **change**:

```text
S2 has become allowed
```

It does not describe the complete membership state.

## Current-State Records

Current-State Records report the receiver's **current interface state** for a group.

They are normally sent in response to Queries.

The two Current-State Record Types are:

```text
MODE_IS_INCLUDE
MODE_IS_EXCLUDE
```

### MODE_IS_INCLUDE

Record Type:

```text
1 = MODE_IS_INCLUDE
```

This means:

```text
The interface is currently in INCLUDE mode for this multicast group.
```

The source addresses in the Group Record contain the interface's **current INCLUDE source list**.

For example:

```text
MODE_IS_INCLUDE
Group: 239.1.1.1
Sources: {S1,S2}
```

means:

```text
Current state:

INCLUDE {S1,S2}
```

The receiver wants:

```text
(S1,239.1.1.1)
(S2,239.1.1.1)
```

but does not want the group from other sources.

An empty source list:

```text
MODE_IS_INCLUDE {}
```

means:

```text
INCLUDE {}
→ no sources are currently desired
```

### MODE_IS_EXCLUDE

Record Type:

```text
2 = MODE_IS_EXCLUDE
```

This means:

```text
The interface is currently in EXCLUDE mode for this multicast group.
```

The listed sources are the interface's **current EXCLUDE source list**.

For example:

```text
MODE_IS_EXCLUDE
Group: 239.1.1.1
Sources: {S1,S2}
```

means:

```text
Current state:

EXCLUDE {S1,S2}
```

The receiver wants the group from every source except `S1` and `S2`.

An empty source list:

```text
MODE_IS_EXCLUDE {}
```

means:

```text
EXCLUDE {}
→ receive from all sources
```

This is the normal IGMPv3 representation of traditional any-source ASM membership.

## Filter-Mode-Change Records

Filter-Mode-Change Records are generated when the interface changes between:

```text
INCLUDE ↔ EXCLUDE
```

The two Record Types are:

```text
CHANGE_TO_INCLUDE_MODE
CHANGE_TO_EXCLUDE_MODE
```

Unlike a Source-List-Change Record, a Filter-Mode-Change Record includes the **new complete source list** after the mode change.

### CHANGE_TO_INCLUDE_MODE

Record Type:

```text
3 = CHANGE_TO_INCLUDE_MODE
```

This means:

```text
The interface has changed to INCLUDE mode.
```

The Group Record contains the **new INCLUDE source list**.

For example:

```text
Old state:
EXCLUDE {S3}

New state:
INCLUDE {S1,S2}
```

The host sends:

```text
CHANGE_TO_INCLUDE_MODE {S1,S2}
```

The record describes the complete new state:

```text
INCLUDE {S1,S2}
```

It does not merely mean that `S1` and `S2` were newly added.

### CHANGE_TO_EXCLUDE_MODE

Record Type:

```text
4 = CHANGE_TO_EXCLUDE_MODE
```

This means:

```text
The interface has changed to EXCLUDE mode.
```

The Group Record contains the **new EXCLUDE source list**.

For example:

```text
Old state:
INCLUDE {S1}

New state:
EXCLUDE {S2,S3}
```

The host sends:

```text
CHANGE_TO_EXCLUDE_MODE {S2,S3}
```

The complete new interface state is:

```text
EXCLUDE {S2,S3}
```

## Source-List-Change Records

A Source-List-Change Record is generated when the source list changes,
but the filter mode stays the same

The two types are:

```text
ALLOW_NEW_SOURCES
BLOCK_OLD_SOURCES
```

These records contain only the **sources whose reception status changed**, not the complete source list.

This distinction is important:

```text
MODE_IS_*
CHANGE_TO_*_MODE
→ describe a complete source list

ALLOW_NEW_SOURCES
BLOCK_OLD_SOURCES
→ describe changes to source reception
```

## ALLOW_NEW_SOURCES

Record Type:

```text
5 = ALLOW_NEW_SOURCES
```

The listed sources are sources from which the receiver **previously did not want traffic but now does**.

The exact source-list operation depends on the filter mode.

### INCLUDE Mode

Suppose the state changes from:

```text
INCLUDE {S1}
```

to:

```text
INCLUDE {S1,S2}
```

`S2` was added to the INCLUDE list.

Therefore:

```text
ALLOW_NEW_SOURCES {S2}
```

is sent.

In INCLUDE mode:

```text
Adding source to INCLUDE list
→ ALLOW_NEW_SOURCES
```

### EXCLUDE Mode

Now suppose the state changes from:

```text
EXCLUDE {S1,S2}
```

to:

```text
EXCLUDE {S1}
```

`S2` was **removed from the EXCLUDE list**.

That means traffic from `S2` was previously blocked but is now wanted.

Therefore the host sends:

```text
ALLOW_NEW_SOURCES {S2}
```

In EXCLUDE mode:

```text
Removing source from EXCLUDE list
→ ALLOW_NEW_SOURCES
```

So `ALLOW_NEW_SOURCES` always has the same semantic meaning:

```text
Start allowing traffic from these sources
```

even though the actual list operation is opposite between INCLUDE and EXCLUDE mode.

## BLOCK_OLD_SOURCES

Record Type:

```text
6 = BLOCK_OLD_SOURCES
```

The listed sources are sources from which the receiver **previously wanted traffic but no longer does**.

Again, the source-list operation depends on the filter mode.

### INCLUDE Mode

Suppose:

```text
Old:
INCLUDE {S1,S2}

New:
INCLUDE {S1}
```

`S2` was removed from the INCLUDE list.

Therefore:

```text
BLOCK_OLD_SOURCES {S2}
```

In INCLUDE mode:

```text
Removing source from INCLUDE list
→ BLOCK_OLD_SOURCES
```

### EXCLUDE Mode

Suppose:

```text
Old:
EXCLUDE {S1}

New:
EXCLUDE {S1,S2}
```

`S2` was added to the EXCLUDE list.

Therefore:

```text
BLOCK_OLD_SOURCES {S2}
```

In EXCLUDE mode:

```text
Adding source to EXCLUDE list
→ BLOCK_OLD_SOURCES
```

So the easiest way to understand the two Source-List-Change types is by their **traffic effect**, rather than by whether an address was added to or removed from a source list:

| Record | Meaning |
|---|---|
| `ALLOW_NEW_SOURCES` | Start receiving from these sources |
| `BLOCK_OLD_SOURCES` | Stop receiving from these sources |

## INCLUDE vs EXCLUDE Source-List Changes

The relationship can be summarized as:

| Filter Mode | Change | Record |
|---|---|---|
| INCLUDE | Add source | `ALLOW_NEW_SOURCES` |
| INCLUDE | Remove source | `BLOCK_OLD_SOURCES` |
| EXCLUDE | Remove source | `ALLOW_NEW_SOURCES` |
| EXCLUDE | Add source | `BLOCK_OLD_SOURCES` |

Notice that the operations are reversed in EXCLUDE mode:

```text
INCLUDE list
→ listed sources are wanted

EXCLUDE list
→ listed sources are not wanted
```

Therefore:

```text
Adding S to INCLUDE
= allow S

Removing S from INCLUDE
= block S

Removing S from EXCLUDE
= allow S

Adding S to EXCLUDE
= block S
```

## Allowing and Blocking Sources Simultaneously

A single source-list change can both allow some sources and block others.

For example:

```text
Old state:
INCLUDE {S1,S2}

New state:
INCLUDE {S2,S3}
```

Two things happened:

```text
S1 → no longer wanted
S3 → newly wanted
```

The host therefore sends **two Group Records for the same group**:

```text
ALLOW_NEW_SOURCES {S3}

BLOCK_OLD_SOURCES {S1}
```

Similarly, in EXCLUDE mode:

```text
Old:
EXCLUDE {S1,S2}

New:
EXCLUDE {S2,S3}
```

means:

```text
S1 was removed from EXCLUDE list
→ S1 is now allowed

S3 was added to EXCLUDE list
→ S3 is now blocked
```

so the host sends:

```text
ALLOW_NEW_SOURCES {S1}

BLOCK_OLD_SOURCES {S3}
```

A single IGMPv3 Membership Report can contain both Group Records.

## Joining a Group

A host with no membership state for a group is conceptually treated as:

```text
INCLUDE {}
```

because it wants the group from no sources.

This makes common join operations easy to understand.

### Source-Specific Join

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

The filter mode did not change; only the source list changed.

Therefore the host can send:

```text
ALLOW_NEW_SOURCES {S1}
```

to join (S1,G).

### Any-Source Join

Suppose the host performs an ordinary ASM join:

```text
Receive G from all sources
```

The state changes:

```text
INCLUDE {}
    ↓
EXCLUDE {}
```

The filter mode changed.

Therefore the host sends:

```text
CHANGE_TO_EXCLUDE_MODE {}
```

This distinction is useful when analyzing IGMPv3 packet captures:

```text
SSM-style join
→ typically ALLOW_NEW_SOURCES

ASM-style join
→ CHANGE_TO_EXCLUDE_MODE
```

## Leaving a Group

The same model explains native IGMPv3 leave behavior.

IGMPv3 does not require the separate IGMPv2:

```text
Leave Group
Type 0x17
```

message.

Instead, a host changes its source-filter state.

### Leaving an `(S,G)` Channel

Suppose:

```text
INCLUDE {S1}
    ↓
INCLUDE {}
```

The filter mode remains INCLUDE, but `S1` is no longer wanted.

The host sends:

```text
BLOCK_OLD_SOURCES {S1}
```

### Leaving an ASM Group

Suppose:

```text
EXCLUDE {}
    ↓
INCLUDE {}
```

The host changes from receiving all sources to receiving no sources.

The filter mode changes, so the host sends:

```text
CHANGE_TO_INCLUDE_MODE {}
```

Thus, `CHANGE_TO_INCLUDE_MODE {}` is effectively an important IGMPv3 representation of leaving an any-source membership.

The detailed retransmission and router-query behavior following these changes is covered on the **IGMPv3 State Changes** page.

## Current-State vs State-Change Records

Current-State and State-Change Records serve different purposes.

### Current-State

Current-State Records answer:

```text
"What do you want right now?"
```

For example:

```text
MODE_IS_INCLUDE {S1,S2}
```

means:

```text
My current state is INCLUDE {S1,S2}
```

They are normally generated in response to Queries.

### State-Change

State-Change Records communicate:

```text
"What just changed?"
```

For example:

```text
ALLOW_NEW_SOURCES {S3}
```

means:

```text
I now want S3
```

and:

```text
BLOCK_OLD_SOURCES {S1}
```

means:

```text
I no longer want S1
```

State-Change Records are transmitted when the host's interface state changes and are retransmitted for robustness.

## SSM and Group Record Types

SSM uses INCLUDE-mode source filtering.

Under normal operation, the most relevant Group Record Types for an SSM group are:

```text
MODE_IS_INCLUDE
ALLOW_NEW_SOURCES
BLOCK_OLD_SOURCES
```

An SSM-aware host should not send:

```text
MODE_IS_EXCLUDE
CHANGE_TO_EXCLUDE_MODE
```

for groups in the SSM range.

An SSM-aware router should ignore those EXCLUDE-mode records for SSM groups.

`CHANGE_TO_INCLUDE_MODE` can still occur in special cases, such as a change to the configured SSM address range.

## Unknown Record Types

If an IGMPv3 implementation receives an unrecognized Group Record Type, it must **silently ignore that Group Record**.

It does not discard the entire Membership Report merely because one Group Record has an unknown Record Type.

## Packet-Capture Interpretation

When analyzing an IGMPv3 Membership Report, do not interpret the source list until you first check the **Record Type**.

For example:

```text
Sources: {S1,S2}
```

has completely different meanings depending on the type:

```text
MODE_IS_INCLUDE {S1,S2}
→ Current state wants only S1 and S2

MODE_IS_EXCLUDE {S1,S2}
→ Current state wants everything except S1 and S2

CHANGE_TO_INCLUDE_MODE {S1,S2}
→ New complete state is INCLUDE {S1,S2}

CHANGE_TO_EXCLUDE_MODE {S1,S2}
→ New complete state is EXCLUDE {S1,S2}

ALLOW_NEW_SOURCES {S1,S2}
→ S1 and S2 have become wanted

BLOCK_OLD_SOURCES {S1,S2}
→ S1 and S2 are no longer wanted
```

The source addresses alone are therefore insufficient to determine the receiver's state.

## Summary

The six IGMPv3 Group Record Types are:

```text
Current-State
├── 1 MODE_IS_INCLUDE
└── 2 MODE_IS_EXCLUDE

Filter-Mode-Change
├── 3 CHANGE_TO_INCLUDE_MODE
└── 4 CHANGE_TO_EXCLUDE_MODE

Source-List-Change
├── 5 ALLOW_NEW_SOURCES
└── 6 BLOCK_OLD_SOURCES
```

The key distinction is:

```text
MODE_IS_*
→ current complete state

CHANGE_TO_*_MODE
→ new complete state after a filter-mode change

ALLOW_NEW_SOURCES / BLOCK_OLD_SOURCES
→ only the sources whose reception status changed
```

The next section examines exactly how routers process these records, update **Group Timers** and **Source Timers**, and generate Group-Specific and Group-and-Source-Specific Queries during **IGMPv3 state changes**.