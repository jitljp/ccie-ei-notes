
# 3.2.a Operations

This section covers the operation of **Multiprotocol Label Switching (MPLS)**, including label-based forwarding, label distribution using LDP, and MPLS connectivity verification.

MPLS separates the **control plane**, which determines reachability and distributes labels, from the **data plane**, which forwards packets using those labels.

In a typical LDP-based MPLS network:

1. An IGP, such as OSPF or IS-IS, establishes IP reachability between routers.
2. LDP distributes label bindings for forwarding equivalence classes (FECs).
3. Routers build their MPLS forwarding tables using routing and label information.
4. Packets travel along label-switched paths (LSPs), with routers performing label push, swap, and pop operations.
5. MPLS ping and traceroute can verify the resulting forwarding paths.

## 3.2.a (i) Label Stack, LSR, LSP

MPLS routers forward labeled packets along **Label Switched Paths (LSPs)**.

Each MPLS label stack entry is 32 bits and contains a label value, Traffic Class (TC), Bottom of Stack (S) bit, and TTL.

Routers perform three fundamental label operations:

- **Push:** Add a label to the packet.
- **Swap:** Replace the top label with another label.
- **Pop:** Remove the top label.

The MPLS forwarding process also includes **Penultimate Hop Popping (PHP)**, implicit-null labels, and explicit-null labels.

Pages:

- [MPLS Labels](labels.md)
- [Basic MPLS Forwarding](basic-mpls-forwarding.md)
- [PHP, Implicit Null, and Explicit Null](php-implicit-explicit-null.md)

## 3.2.a (ii) LDP

**Label Distribution Protocol (LDP)** distributes label bindings between MPLS routers.

LDP works alongside the IGP:

- The **IGP** determines the routing paths to destination prefixes.
- **LDP** distributes the labels associated with those prefixes.
- The **LFIB** determines how labeled packets are forwarded.

Topics include LDP discovery, sessions, label advertisement and retention, authentication, and mechanisms for improving MPLS convergence and reliability.

Pages:

- [LDP](ldp.md)
- [LDP Session Protection](ldp-session-protection.md)
- [LDP Authentication](ldp-authentication.md)
- [Label Filtering](label-filtering.md)
- [LDP-IGP Synchronization](ldp-igp-sync.md)

## 3.2.a (iii) MPLS Ping, MPLS Traceroute

**MPLS ping** and **MPLS traceroute** are MPLS Operations, Administration, and Maintenance (OAM) tools used to verify label-switched paths.

- **MPLS ping** verifies connectivity and forwarding consistency for an MPLS FEC.
- **MPLS traceroute** identifies the routers along an LSP and helps locate forwarding problems.

Unlike ordinary IP ping and traceroute, these tools can validate MPLS forwarding information rather than relying only on IP connectivity.

Page:

- [MPLS Ping and Traceroute](mpls-ping-traceroute.md)

## Key Points

- MPLS forwards packets using **labels** instead of performing an IP destination lookup at every hop.
- MPLS forwarding uses **push**, **swap**, and **pop** operations.
- An **LSP** is the path followed by labeled packets through an MPLS network.
- The **IGP** determines reachability, while **LDP** distributes label bindings.
- The **LFIB** contains the information required to forward labeled packets.
- **PHP** allows the penultimate router to remove the transport label before forwarding the packet to the egress router.
- **MPLS ping and traceroute** verify MPLS forwarding paths and help troubleshoot LSP problems.
