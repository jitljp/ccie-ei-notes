# Multicast Routing State

Multicast routers maintain forwarding state for multicast groups in the **multicast routing table**.

The two main types of state are:

```text
(*,G) = shared-tree state
(S,G) = source-specific state
```

For each entry, the router tracks:

```text
Incoming Interface (IIF)
RPF Neighbor
Outgoing Interface List (OIL)
```

## (*,G) State

`(*,G)` represents traffic for group `G` from **any source**.

For example:

```text
(*, 239.1.1.1)
```

In PIM Sparse Mode, this represents the **shared tree toward the RP**.

The incoming interface points toward the RP:

```text
                    RP
                    |
                    |
                   R1
                 /    \
               R2      R3
               |       |
            Receiver Receiver
```

On `R1`:

```text
(*,239.1.1.1)

IIF: toward RP
OIL: toward R2, R3
```

The OIL contains interfaces that lead toward interested receivers.

---

## (S,G) State

`(S,G)` represents traffic from a **specific source** to a specific group.

For example:

```text
(10.1.1.10, 239.1.1.1)
```

The incoming interface points toward the source:

```text
              Source
                 |
                R1
              /    \
            R2      R3
            |       |
         Receiver Receiver
```

On `R1`:

```text
(10.1.1.10,239.1.1.1)

IIF: toward 10.1.1.10
OIL: toward R2, R3
```

This is the forwarding state used by a source-specific tree.

---

## Incoming Interface

Each multicast routing entry has one **Incoming Interface (IIF)**.

The IIF is the interface on which multicast traffic is expected to arrive.

Its meaning depends on the type of state:

```text
(*,G)
IIF points toward the RP

(S,G)
IIF points toward the source
```

The router determines this using the **Reverse Path Forwarding (RPF)** process.

For example:

```text
Source 10.1.1.10
        |
       R1
       |
       R2
```

If `R2`'s best path toward `10.1.1.10` is through `Gi0/0`:

```text
(S,G)

Incoming interface: Gi0/0
```

Multicast packets from that source are expected to arrive on `Gi0/0`.

If they arrive on another interface, they fail the RPF check and are dropped.

---

## RPF Neighbor

The multicast routing entry also identifies the **RPF neighbor**.

This is the upstream router toward the root of the tree.

For:

```text
(*,G)
```

the RPF neighbor is toward the **RP**.

For:

```text
(S,G)
```

the RPF neighbor is toward the **source**.

Example:

```text
Source ---- R1 ---- R2 ---- R3
```

On `R3`, for `(S,G)`:

```text
Incoming Interface: toward R2
RPF Neighbor:       R2
```

The RPF process is covered in detail later.

---

## Outgoing Interface List

The **Outgoing Interface List (OIL)** identifies the interfaces on which multicast traffic should be forwarded.

For example:

```text
                 R1
               /    \
             R2      R3
             |       |
          Receiver Receiver
```

A multicast entry on `R1` might contain:

```text
Incoming Interface:
Gi0/0

Outgoing Interface List:
Gi0/1
Gi0/2
```

When a multicast packet passes the RPF check, the router replicates the packet and forwards a copy out each appropriate interface in the OIL.

```text
                  packet
                     |
                     v
                    R1
                  /    \
             copy      copy
               |        |
               v        v
              R2        R3
```

This replication is one of the fundamental differences between multicast and unicast forwarding.

---

## Building the OIL

Interfaces can be added to the OIL because of downstream multicast interest.

Common reasons include:

- a directly connected receiver joined the group
- a downstream PIM router sent a Join

If there is no downstream interest on an interface, that interface does not need to remain in the OIL.

An entry with no usable outgoing interfaces is not useful for forwarding multicast traffic downstream.

---

## Example Multicast Routing Entry

Cisco IOS XE displays multicast routing state with:

```text
show ip mroute
```

A simplified entry might look like:

```text
(*, 239.1.1.1)

Incoming interface: GigabitEthernet0/0
RPF nbr: 10.0.12.1

Outgoing interface list:
  GigabitEthernet0/1
  GigabitEthernet0/2
```

This means:

```text
                    RP
                     |
                     |
                 Gi0/0
                     |
                   Router
                  /      \
             Gi0/1      Gi0/2
                |          |
           Receiver     Receiver
```

For source-specific state:

```text
(10.1.1.10, 239.1.1.1)

Incoming interface: GigabitEthernet0/3
RPF nbr: 10.0.34.3

Outgoing interface list:
  GigabitEthernet0/1
  GigabitEthernet0/2
```

The important difference is what the upstream direction represents:

```text
(*,G)  -> toward RP
(S,G)  -> toward source
```

---

## (*,G) and (S,G) Can Exist Together

A router can maintain both entries for the same group:

```text
(*,239.1.1.1)

(10.1.1.10,239.1.1.1)
```

The `(*,G)` entry represents shared-tree state.

The `(S,G)` entry represents forwarding state for a particular source.

This commonly occurs during PIM Sparse Mode operation and SPT switchover.

---

## Key Point

The essential fields of a multicast routing entry are:

```text
(*,G) or (S,G)
        +
Incoming Interface
        +
RPF Neighbor
        +
Outgoing Interface List
```

The **IIF identifies where traffic should arrive**, while the **OIL identifies where matching traffic should be replicated and forwarded**.