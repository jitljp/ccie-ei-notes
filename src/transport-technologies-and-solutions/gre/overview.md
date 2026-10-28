# Static Point-to-Point GRE Tunnels

**GRE** (Generic Routing Encapsulation) is a tunneling protocol that encapsulates packets inside another IP packet so they can be transported across an IP network.

A **static point-to-point GRE tunnel** connects two routers using a logical tunnel interface on each router. The tunnel endpoints are explicitly configured, and the routers communicate through the existing IP network between them.

```text
LAN A                                               LAN B
10.1.1.0/24                                         10.2.2.0/24
     |                                                   |
     R1 ------------ IP Network / ISP ----------------- R2
192.0.2.1                                          198.51.100.2
     |                                                   |
Tunnel0                                             Tunnel0
10.0.12.1/30                                        10.0.12.2/30
     |                                                   |
     +=============== GRE Tunnel ========================+
```

The physical network forwards the encapsulated packets between R1 and R2. From the perspective of routing over the tunnel, R1 and R2 are directly connected through `Tunnel0`.

---

## Why GRE Is Used

GRE allows routers to carry traffic across a network without requiring the intermediate routers to understand the encapsulated traffic.

Common uses include:

- Connecting two private networks across an IP network.
- Carrying IPv6 traffic across an IPv4-only network.
- Running dynamic routing protocols between routers that are not physically adjacent.
- Transporting multicast or broadcast traffic across a unicast IP network.
- Providing a tunnel that can be protected with IPsec.

For example, an ISP might provide only IPv4 unicast connectivity between two sites. A GRE tunnel can carry IPv6 packets and routing-protocol traffic between the sites without requiring the ISP to participate in the overlay routing.

> GRE provides **encapsulation**, not encryption or authentication. GRE alone does not make traffic confidential or secure against modification. IPsec can be used to protect GRE traffic.

---

## Underlay and Overlay Networks

A GRE tunnel uses two separate routing contexts:

| Network | Purpose | Example |
| --- | --- | --- |
| **Underlay** | Delivers the outer IP packets between the tunnel endpoints. | R1 `192.0.2.1` to R2 `198.51.100.2` |
| **Overlay** | Carries traffic through the logical GRE tunnel. | R1 `10.0.12.1/30` to R2 `10.0.12.2/30` |

The **underlay** is the existing network between the two routers. It may contain many intermediate routers, but it must provide IP reachability between the tunnel endpoints.

The **overlay** is the logical point-to-point connection created by the tunnel interfaces.

```text
Underlay (physical forwarding path):

R1 ---- ISP Router ---- ISP Router ---- R2
192.0.2.1                          198.51.100.2

Overlay (logical routing adjacency):

R1 Tunnel0 ------------------ R2 Tunnel0
10.0.12.1                       10.0.12.2
```

Routes to the remote tunnel endpoint must resolve through the underlay, **not back through the GRE tunnel itself**. If a route to the tunnel destination points through the tunnel, recursive routing can cause the tunnel to fail.

The tunnel interface IP addresses belong to the overlay. They are not the addresses used in the outer IP header.

---

## GRE Encapsulation and Forwarding

Consider a host in LAN A sending a packet to a host in LAN B:

```text
Original packet:

Source IP:      10.1.1.10
Destination IP: 10.2.2.10
```

R1 has a route to `10.2.2.0/24` through `Tunnel0`.

R1 performs the following steps:

1. Receives the original IP packet.
2. Determines that the packet should be forwarded through `Tunnel0`.
3. Adds a **GRE header** and a new **outer IP header**.
4. Forwards the encapsulated packet toward the remote tunnel endpoint using the underlay routing table.

The encapsulated packet looks like this:

```text
+--------------------+------------+--------------------+---------+
| Outer IPv4 Header  | GRE Header | Inner IPv4 Header  | Payload |
+--------------------+------------+--------------------+---------+
  192.0.2.1              GRE        10.1.1.10
  -> 198.51.100.2                   -> 10.2.2.10
```

The intermediate routers use the **outer destination IP address**, `198.51.100.2`, to forward the packet. They do not need a route to `10.2.2.0/24`.

When R2 receives the packet, it removes the outer IP and GRE headers, then processes the original packet for forwarding toward LAN B.

This is called **encapsulation** at the sending tunnel endpoint and **decapsulation** at the receiving endpoint.

### GRE Protocol Number

When GRE is transported over IPv4, the outer IPv4 header uses **IP protocol number 47** to identify GRE.

```text
Outer IPv4 header:
  Source IP:      192.0.2.1
  Destination IP: 198.51.100.2
  Protocol:       47 (GRE)
```

**47 is an IP protocol number, not a TCP or UDP port number.** An underlay ACL or firewall must permit GRE (IP protocol 47) for this traffic to pass.

For basic GRE over IPv4, the additional headers are typically:

| Header | Size |
| --- | ---: |
| Outer IPv4 header (without options) | 20 bytes |
| Basic GRE header (without optional fields) | 4 bytes |
| **Total overhead** | **24 bytes** |

Optional GRE fields and different outer IP versions can change the overhead. MTU, fragmentation, and MSS are covered separately.

---

## Point-to-Point GRE Characteristics

In a static point-to-point GRE tunnel, each router is configured with a **tunnel source** and a **tunnel destination**.

| Component | Purpose |
| --- | --- |
| Tunnel interface | Logical Layer 3 interface used for overlay forwarding |
| Tunnel source | Local address or interface used as the outer IP source |
| Tunnel destination | Remote endpoint used as the outer IP destination |
| Tunnel mode | Defines the encapsulation and transport protocol |
| Tunnel IP address | Overlay address used for communication over the tunnel |
| Underlay route | Provides reachability to the remote tunnel destination |
| Overlay route | Directs passenger traffic into the tunnel |

A static point-to-point GRE tunnel has **one configured remote destination**. GRE does not dynamically discover tunnel peers or negotiate tunnel establishment.

For two-way communication, the remote router must also be configured to send GRE traffic back toward the local router.

> A tunnel interface being up does not necessarily prove that the remote endpoint can receive and decapsulate GRE packets. Without keepalives or another reachability mechanism, a tunnel can appear up even if the remote endpoint is not functioning.

---

## Passenger, Carrier, and Transport Protocols

GRE distinguishes between three roles:

| Role | Meaning | Example |
| --- | --- | --- |
| **Passenger protocol** | Original protocol carried through the tunnel | IPv4 or IPv6 |
| **Carrier protocol** | Protocol that encapsulates the passenger | GRE |
| **Transport protocol** | Outer protocol used to reach the remote endpoint | IPv4 or IPv6 |

GRE can carry different passenger protocols without requiring a separate tunnel for each one.

Common combinations include:

| Tunnel mode | Passenger | Transport |
| --- | --- | --- |
| `tunnel mode gre ip` | IPv4 or IPv6 | IPv4 |
| `tunnel mode gre ipv6` | IPv4 or IPv6 | IPv6 |

For example, **IPv6 over IPv4 GRE** carries an IPv6 packet inside GRE and an outer IPv4 header:

```text
+-------------------+------------+-------------------+---------+
| Outer IPv4 Header | GRE Header | Inner IPv6 Header | Payload |
+-------------------+------------+-------------------+---------+
```

This allows two IPv6 networks to communicate across an IPv4-only underlay.

`gre ip` and `gre ipv6` describe the **transport protocol**, not the passenger protocol. In particular, `tunnel mode gre ip` does not restrict the tunnel to carrying IPv4 traffic.

GRE is different from direct IPv6-in-IPv4 encapsulation using `tunnel mode ipv6ip`, which does not add a GRE header.

---

## Routing over GRE

Because the GRE tunnel appears as a logical Layer 3 interface, it can be used as an outgoing interface for routes.

For example:

```text
R1 LAN: 10.1.1.0/24
R2 LAN: 10.2.2.0/24

R1 routes 10.2.2.0/24 through Tunnel0.
R2 routes 10.1.1.0/24 through Tunnel0.
```

These routes can be configured statically or learned through a dynamic routing protocol running over the tunnel.

GRE can encapsulate multicast traffic, which makes it useful for routing-protocol hellos and other multicast control traffic even when the underlay provides only unicast connectivity.

The routers can therefore form an OSPF or EIGRP neighborship across the tunnel while the intermediate underlay routers remain unaware of the overlay routing protocol.

---

## Tunnel State and Failure Detection

GRE itself does not establish a session or require a handshake before traffic is encapsulated.

The operational state of a tunnel depends on factors such as the tunnel configuration, tunnel source interface, and route to the remote tunnel destination. However, these checks alone do not guarantee that the remote endpoint is receiving traffic.

GRE keepalives can provide additional failure detection for supported point-to-point GRE over IPv4 tunnels. Routing-protocol hellos can also detect a failed adjacency when dynamic routing is used.

A common failure is **recursive routing**, where the route to the GRE tunnel destination is learned through the GRE tunnel itself. The tunnel requires underlay reachability to its endpoint, so such a route cannot be used to sustain the tunnel.

---

## GRE and MTU

Encapsulation increases the packet size because the outer IP and GRE headers are added to the original packet.

For example, a 1500-byte original IPv4 packet becomes **1524 bytes** after basic GRE-over-IPv4 encapsulation (assuming no optional headers).

If the underlay MTU is 1500 bytes, the encapsulated packet exceeds that limit. Depending on packet size, configuration, and fragmentation restrictions, packets may need to be fragmented or may be dropped.

Tunnel MTU, Path MTU Discovery, and TCP MSS adjustment can help avoid fragmentation and connectivity problems.

---

## GRE and IPsec

GRE and IPsec have different purposes.

| GRE | IPsec |
| --- | --- |
| Encapsulates passenger traffic for transport across an IP network | Protects IP traffic using cryptographic security services |
| Supports routing-protocol multicast and other passenger protocols | Provides confidentiality, integrity, and authentication through its security mechanisms |
| Does not encrypt packets | Can encrypt packets |
| Does not authenticate tunnel peers | Can authenticate peers as part of IPsec establishment |

GRE can be combined with IPsec when both flexible encapsulation and traffic protection are required.

> A GRE tunnel by itself is **not an encrypted VPN**.

---

## Key Points

- GRE encapsulates passenger packets inside GRE and an outer IP header.
- Static point-to-point GRE tunnels have explicitly configured tunnel endpoints.
- The underlay provides IP reachability between tunnel source and destination addresses.
- The overlay provides a logical routed connection between the tunnel interfaces.
- Intermediate routers forward based on the **outer IP destination**, not the inner destination.
- GRE over IPv4 uses **IP protocol 47**, not TCP or UDP port 47.
- The basic GRE-over-IPv4 encapsulation overhead is **24 bytes**.
- `tunnel mode gre ip` can carry **IPv4 and IPv6** traffic over an IPv4 underlay.
- GRE can carry multicast routing-protocol traffic through a unicast underlay.
- GRE does not encrypt or authenticate traffic. IPsec can be used for protection.
- Tunnel interface state does not always indicate end-to-end connectivity.
- The route to the remote tunnel destination must resolve through the underlay to avoid recursive routing.
- Additional encapsulation overhead can cause MTU and fragmentation problems.

---

## References

- [Cisco IOS XE 17.x: Implementing Tunnels](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_ir-impl-tun-xe.html)
- [Cisco IOS XE 17.x: GRE IPv6 Tunnels](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_ir-gre-ipv6-tunls-xe.html)
- [RFC 2784: Generic Routing Encapsulation (GRE)](https://www.rfc-editor.org/rfc/rfc2784.html)
