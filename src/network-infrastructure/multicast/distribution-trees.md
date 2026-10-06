# Multicast Distribution Trees

Multicast routers build **distribution trees** to forward traffic from sources to receivers.

The two main tree types are:

```text
Shared Tree = Rendezvous Point Tree (RPT) = (*,G) forwarding state
Source Tree = Shortest Path Tree (SPT)    = (S,G) forwarding state
```

## Example Topology

The following diagram shows the **physical topology**, not a multicast forwarding tree:

```text
                              Source S
                                 |
                                R1
                              /    \
                            R2      R3
                           /  \      |
                         R6    R7    R4
                         |      |    |
                         R9    R10   RP
                         |      |
                      Recv A  Recv B
```

Assume both receivers want traffic for:

```text
239.1.1.1
```

The RP is located behind `R3` and `R4`, while the shortest path from the source to the receivers is through `R2`.

This is not an ideal RP placement, but it makes the difference between the RPT and SPT easy to see.

---

## Shared Tree — RPT

In PIM Sparse Mode, the receiver-side routers initially join toward the RP.

This builds the **Rendezvous Point Tree (RPT)** and creates `(S,G)` forwarding state.

For Receiver A, the shared-tree path from the RP is:

```text
RP -> R4 -> R3 -> R1 -> R2 -> R6 -> R9 -> Recv A
```

For Receiver B:

```text
RP -> R4 -> R3 -> R1 -> R2 -> R7 -> R10 -> Recv B
```

The RPT is therefore rooted at the **RP**:

```text
                              RP
                              |
                              R4
                              |
                              R3
                              |
                              R1
                              |
                              R2
                            /    \
                          R6      R7
                          |        |
                          R9      R10
                          |        |
                       Recv A   Recv B
```

The shared-tree state is:

```text
(*, 239.1.1.1)
```

where `*` means **any source**.

---

## The RP Builds an SPT Toward the Source

The receiver side builds an **RPT toward the RP**, but the RP must also receive traffic from the multicast source.

After learning about the source, the RP sends a PIM Join toward that specific source.

This builds an **SPT from the RP toward the source** using `(S,G)` state.

In this topology, the RP's SPT toward the source is:

```text
Source S -> R1 -> R3 -> R4 -> RP
```

So before the receiver switches to its own SPT, multicast traffic uses **two trees joined at the RP**:

```text
Source
   |
   | SPT — (S,G)
   v
   RP
   |
   | RPT — (*,G)
   v
Receiver
```

For Receiver A, the complete forwarding path is therefore:

```text
Source  ->  R1  ->  R3  ->   R4   ->   RP
                                        |
                                        v
Recv A <- R9 <- R6 <- R2 <- R1 <- R3 <- R4
```

Or:

```text
Source -> SPT -> RP -> RPT -> Receiver
```

The traffic travels toward the RP and then back toward the receiver.

> How the source is initially discovered by the RP using PIM Register messages is covered later.

---

## Source Tree — SPT

A **Shortest Path Tree (SPT)** is rooted at a multicast source.

Its forwarding state is:

```text
(S,G)
```

For example:

```text
(10.1.1.10, 239.1.1.1)
```

The RP already uses an SPT to reach the source.

A receiver-side router can also build its **own SPT directly toward the source**, bypassing the RP.

For Receiver A, the shortest path is:

```text
Source S -> R1 -> R2 -> R6 -> R9 -> Recv A
```

For Receiver B:

```text
Source S -> R1 -> R2 -> R7 -> R10 -> Recv B
```

The receiver-side SPT is therefore:

```text
                              Source S
                                 |
                                R1
                                 |
                                R2
                              /    \
                            R6      R7
                            |        |
                            R9      R10
                            |        |
                         Recv A   Recv B
```

The RP is no longer in the receiver's data path.

---

## RPT to SPT Switchover

Initially, Receiver A receives traffic using:

```text
Source -> SPT -> RP -> RPT -> Receiver
```

In this topology:

```text
Source  ->  R1  ->  R3  ->   R4   ->   RP
                                        |
                                        v
Recv A <- R9 <- R6 <- R2 <- R1 <- R3 <- R4
```

After a receiver's LHR receives multicast packets from the source,
the LHR can then join the source directly and build its own `(S,G)` SPT.

After the switchover:

```text
Source -> R1 -> R2 -> R6 -> R9 -> Recv A
```

The SPT switchover eliminates the detour through the RP:

```text
Before:

Source -> SPT -> RP -> RPT -> Receiver


After:

Source -------------> Receiver
          SPT
```

The exact SPT switchover process is covered later.

---

## RPT vs SPT

```text
RPT
Root:   RP
State:  (*,G)
Used:   Shared tree from the RP toward receivers

SPT
Root:   Source
State:  (S,G)
Used:   Source-specific shortest-path tree
```

In normal PIM Sparse Mode operation:

```text
Receiver side:
Builds an RPT toward the RP

RP:
Builds an SPT toward the source

After SPT switchover:
Receiver side builds an SPT directly toward the source
```

## Source-Specific Multicast

SSM does not use an RP or shared tree.

The receiver specifies both the source and group:

```text
(S,G)
```

Therefore, the LHRs build an SPT directly toward the source.

## Key Point

Before SPT switchover in PIM Sparse Mode:

```text
Source -> SPT -> RP -> RPT -> Receiver
```

After SPT switchover:

```text
Source -> SPT -> Receiver
```

The **receiver side initially builds an RPT toward the RP**, while the **RP builds an SPT toward the source**. The receiver side can then switch to its own SPT and remove the RP from the data path.