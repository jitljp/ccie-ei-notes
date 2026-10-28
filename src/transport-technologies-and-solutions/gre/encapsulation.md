# GRE Encapsulation and Packet Forwarding

**GRE** (Generic Routing Encapsulation) carries a **passenger packet** through an IP network by adding a GRE header and a new outer IP header.

In a static point-to-point GRE tunnel, the ingress router encapsulates the original packet and sends it toward the configured tunnel destination. The remote tunnel endpoint removes the added headers and processes the original packet.

The intermediate routers only need to know how to reach the **outer IP destination**. They do not need routes to the networks carried inside the tunnel.

---

## Basic Topology

Use the following topology:

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

There are two different networks involved:

| Network | Addresses | Purpose |
| --- | --- | --- |
| **Underlay** | `192.0.2.1` and `198.51.100.2` | Carries GRE packets between the tunnel endpoints. |
| **Overlay** | `10.0.12.1/30` and `10.0.12.2/30` | Provides a logical point-to-point routed link between R1 and R2. |
| **LANs** | `10.1.1.0/24` and `10.2.2.0/24` | Networks exchanging traffic through the GRE tunnel. |

Assume a host at `10.1.1.10` sends a packet to `10.2.2.10`.

R1 has a route to `10.2.2.0/24` through `Tunnel0`. R2 has a route to `10.1.1.0/24` through its own `Tunnel0`.

---

## Passenger, Carrier, and Transport Protocols

GRE tunneling involves three protocol roles:

| Role | Description | Example |
| --- | --- | --- |
| **Passenger** | The original packet being carried. | IPv4 or IPv6 |
| **Carrier** | The encapsulation protocol identifying the passenger. | GRE |
| **Transport** | The outer protocol used to deliver the packet between tunnel endpoints. | IPv4 or IPv6 |

For an IPv4 packet carried through a GRE-over-IPv4 tunnel:

```text
Passenger: IPv4
Carrier:   GRE
Transport: IPv4
```

The passenger and transport protocols are independent. For example, an IPv6 passenger packet can be carried across an IPv4 underlay using `tunnel mode gre ip`.

---

## GRE Packet Format

The ingress router adds a GRE header and a new outer IP header in front of the original packet.

For basic **IPv4 over GRE over IPv4**, the packet has this structure:

```text
+-------------------+------------+-------------------+---------+
| Outer IPv4 Header | GRE Header | Inner IPv4 Header | Payload |
+-------------------+------------+-------------------+---------+
        20 bytes       4 bytes
```

The **inner IP header** is the header of the original packet. The **outer IP header** is created by the tunnel source router to deliver the encapsulated packet to the tunnel destination.

The original packet, including its IP header, becomes the **GRE payload**.

> GRE does not replace the inner IP header. It adds new headers around the original packet.

The Ethernet header used on each physical link is separate from the GRE encapsulation. On a normal Layer 3 GRE tunnel, the original LAN Ethernet header is not carried inside the tunnel.

---

## Outer and Inner IP Headers

Consider the packet from `10.1.1.10` to `10.2.2.10`.

Before encapsulation, the packet contains:

```text
Inner IPv4 header:
  Source:      10.1.1.10
  Destination: 10.2.2.10
  Protocol:    TCP, UDP, ICMP, etc.
```

After R1 encapsulates it, the packet contains two IP headers:

```text
Outer IPv4 header:
  Source:      192.0.2.1
  Destination: 198.51.100.2
  Protocol:    47 (GRE)

GRE header:
  Protocol Type: 0x0800 (IPv4)

Inner IPv4 header:
  Source:      10.1.1.10
  Destination: 10.2.2.10
  Protocol:    Original upper-layer protocol
```

| Field | Outer IP header | Inner IP header |
| --- | --- | --- |
| Source address | Tunnel source, `192.0.2.1` | Original source, `10.1.1.10` |
| Destination address | Tunnel destination, `198.51.100.2` | Original destination, `10.2.2.10` |
| Protocol | GRE (47) | Original protocol, such as TCP (6) or ICMP (1) |
| Forwarding purpose | Deliver the encapsulated packet to R2 | Deliver the original packet to its final destination |

The **tunnel interface addresses** (`10.0.12.1` and `10.0.12.2`) do not automatically appear in either IP header for this host-to-host packet. They are overlay addresses used for routing and communication between the tunnel interfaces.

---

## IP Protocol Number 47

GRE carried over IPv4 is identified by **IP protocol number 47** in the outer IPv4 header.

```text
Outer IPv4 Protocol field = 47 (GRE)
```

This is distinct from the original packet's protocol field, which remains inside the GRE payload.

For example, if the original packet is TCP:

```text
Outer IPv4 header: Protocol 47 (GRE)
Inner IPv4 header: Protocol 6  (TCP)
```

**GRE does not use TCP or UDP ports.** An ACL or firewall that filters the underlay must permit IP protocol 47 between the tunnel endpoints, not TCP or UDP port 47.

For comparison:

| Protocol | IP protocol number |
| --- | ---: |
| ICMP | 1 |
| IPv4-in-IPv4 (without GRE) | 4 |
| TCP | 6 |
| UDP | 17 |
| GRE | **47** |
| ESP (IPsec) | 50 |

When the outer transport is IPv6, GRE is identified by **Next Header value 47** rather than an IPv4 Protocol field.

---

## GRE Header Fields

A basic GRE header is **4 bytes (32 bits)** long.

```text
   First 16 bits                 Next 16 bits
+--------------------------+----------------------+
| Flags and Version        | Protocol Type        |
+--------------------------+----------------------+
        16 bits                     16 bits
```

The first 16 bits include flags, reserved bits, and the GRE version. The second 16 bits identify the passenger protocol.

| Field | Purpose |
| --- | --- |
| **C bit** | Indicates that the optional Checksum and Reserved1 fields are present. |
| **K bit** | Indicates that an optional GRE Key field is present. |
| **S bit** | Indicates that an optional Sequence Number field is present. |
| **Reserved bits** | Reserved for defined extensions or future use. |
| **Version** | Version 0 for the GRE format described by RFC 2784 and RFC 2890. |
| **Protocol Type** | Identifies the encapsulated passenger protocol. |

The **C**, **K**, and **S** bits are not the same thing as the passenger protocol's flags. They describe optional information in the GRE header itself.

### Protocol Type Field

The GRE **Protocol Type** field uses EtherType values to identify the encapsulated protocol.

| Passenger protocol | GRE Protocol Type |
| --- | --- |
| IPv4 | `0x0800` |
| IPv6 | `0x86DD` |

Do not confuse these values with the **outer IP protocol number 47**:

```text
Outer IP Protocol = 47       Identifies GRE
GRE Protocol Type = 0x0800  Identifies the IPv4 payload
```

For IPv6 carried inside GRE over IPv4:

```text
Outer IPv4 Protocol = 47
GRE Protocol Type   = 0x86DD
Inner packet        = IPv6
```

### Optional GRE Fields

The basic 4-byte GRE header can be extended with optional fields.

```text
+----------------------------------+
| Flags/Version | Protocol Type    |  4 bytes (required)
+----------------------------------+
| Checksum      | Reserved1        |  4 bytes (optional)
+----------------------------------+
| Key                              |  4 bytes (optional)
+----------------------------------+
| Sequence Number                  |  4 bytes (optional)
+----------------------------------+
```

| Optional field | Size | Purpose |
| --- | ---: | --- |
| Checksum + Reserved1 | 4 bytes | Provides optional GRE header/payload error detection. |
| Key | 4 bytes | Identifies a logical traffic flow or tunnel context. |
| Sequence Number | 4 bytes | Provides sequence information for ordering and loss detection. |

These fields are **not automatically included** in every GRE packet. The relevant flag indicates whether each one is present.

The GRE **Key** is a numeric identifier, not a cryptographic key. It does **not** encrypt the packet or provide secure authentication. GRE sequencing is also optional; ordinary GRE does not guarantee reliable delivery or preserve packet order.

---

## GRE Encapsulation Overhead

For a GRE tunnel using IPv4 as the transport protocol, the usual overhead is:

| Header | Size |
| --- | ---: |
| Outer IPv4 header without options | 20 bytes |
| Basic GRE header | 4 bytes |
| **Total** | **24 bytes** |

Therefore:

```text
Original IPv4 packet:  1000 bytes
Outer IPv4 header:       20 bytes
GRE header:               4 bytes
---------------------------------
Encapsulated packet:   1024 bytes
```

If the GRE Key field is included, it adds another 4 bytes. For example, an IPv4-transport GRE packet with a Key field but no Checksum or Sequence Number field has **28 bytes** of encapsulation overhead.

With an IPv6 transport header (40 bytes) and the basic GRE header (4 bytes), the overhead is **44 bytes**.

The overhead is independent of whether the passenger is IPv4 or IPv6. It depends on the **outer transport header** and the GRE fields included.

### MTU Implications

A 1500-byte original packet becomes 1524 bytes when basic GRE-over-IPv4 encapsulation is added.

```text
1500-byte original IP packet
 + 20-byte outer IPv4 header
 +  4-byte GRE header
----------------------------
1524 bytes
```

If the underlay path has an IP MTU of 1500 bytes, an unfragmented GRE packet with basic IPv4 transport can carry at most **1476 bytes of passenger IP packet**:

```text
1500 - 24 = 1476 bytes
```

Larger packets may require fragmentation or may be dropped, depending on the MTU, DF bit, and tunnel configuration.

See [GRE MTU, Fragmentation, and MSS](mtu-mss.md) for the detailed packet handling.

---

## Step-by-Step GRE Packet Forwarding

Return to the topology:

```text
10.1.1.10                                                10.2.2.10
    |                                                        |
    R1 ----------- ISP1 ----------- ISP2 ------------------- R2
    |                                                        |
  Tunnel0 =============================================== Tunnel0
```

The host at `10.1.1.10` sends a packet to `10.2.2.10`.

### Step 1: R1 Selects the Tunnel

R1 receives the original IP packet and performs a routing lookup for the **inner destination** `10.2.2.10`.

Assume R1 has a route like this:

```text
10.2.2.0/24 -> Tunnel0
```

R1 therefore selects `Tunnel0` as the outgoing interface. The tunnel interface is a **logical interface**, not the physical interface used to send the outer packet.

### Step 2: R1 Encapsulates the Packet

R1 builds the GRE header and adds an outer IPv4 header using its configured tunnel source and destination.

```text
Outer source:       192.0.2.1
Outer destination:  198.51.100.2
Outer protocol:     47 (GRE)
GRE Protocol Type:  0x0800 (IPv4)
Inner source:       10.1.1.10
Inner destination:  10.2.2.10
```

The original source and destination IP addresses are retained in the inner packet. Routing-related fields, such as the inner TTL, still follow normal IP forwarding rules.

### Step 3: R1 Forwards Through the Underlay

R1 resolves the **outer destination** `198.51.100.2` through the underlay routing table and sends the encapsulated packet out the appropriate physical interface.

```text
Overlay lookup:
  10.2.2.10 -> Tunnel0

Underlay lookup:
  198.51.100.2 -> Physical next hop
```

The route to `198.51.100.2` must not resolve back through `Tunnel0`, or recursive routing results.

### Step 4: Intermediate Routers Forward the Outer Packet

ISP1 and ISP2 perform ordinary IP forwarding toward `198.51.100.2`.

```text
ISP1: Lookup 198.51.100.2 -> ISP2
ISP2: Lookup 198.51.100.2 -> R2
```

They do **not** need routes to:

```text
10.1.1.0/24
10.2.2.0/24
10.0.12.0/30
```

The underlay routers forward based on the outer header. They do not normally decapsulate or route the inner passenger packet.

### Step 5: R2 Decapsulates the Packet

R2 receives the packet addressed to its tunnel destination IP, `198.51.100.2`, and identifies it as GRE using protocol number 47.

After matching and processing the tunnel packet, R2 removes the outer IPv4 header and the GRE header.

The original packet remains:

```text
Inner source:       10.1.1.10
Inner destination:  10.2.2.10
```

### Step 6: R2 Forwards the Original Packet

R2 performs a routing lookup for `10.2.2.10` in the routing table associated with the decapsulated packet.

```text
10.2.2.0/24 -> LAN-facing interface
```

R2 forwards the original IP packet toward `10.2.2.10`.

Traffic in the opposite direction uses the reverse GRE forwarding process, with R2 encapsulating packets toward R1.

---

## GRE and TTL

The **inner** and **outer** IP headers have separate TTL fields (for IPv4) or Hop Limit fields (for IPv6).

| Field | Purpose |
| --- | --- |
| **Inner TTL** | Limits the lifetime of the original IP packet as it is routed through the overlay. |
| **Outer TTL** | Limits the lifetime of the encapsulated packet as it travels through the underlay. |

Underlay routers decrement the **outer TTL** as they forward the GRE packet. They do **not** decrement the encapsulated inner TTL.

Normal IP forwarding at the tunnel endpoints accounts for the inner packet's TTL. Therefore, the inner TTL and outer TTL are **independent**. The number of intermediate underlay hops does not directly determine how much the inner TTL is decremented.

The GRE header itself has **no TTL field**.

---

## Encapsulating Multicast and IPv6 Traffic

GRE can also carry passenger protocols and destinations that the underlay does not directly support.

For example, when R1 sends an OSPF Hello to `224.0.0.5` over `Tunnel0`, the packet can be encapsulated as GRE and delivered using the **unicast outer destination** `198.51.100.2`.

```text
Outer IPv4 destination: 198.51.100.2 (unicast)
GRE Protocol Type:      0x0800 (IPv4)
Inner IPv4 destination: 224.0.0.5 (multicast)
```

The ISP does not need to support the overlay's OSPF multicast forwarding.

Similarly, GRE over IPv4 can carry IPv6 packets:

```text
+-------------------+------------+-------------------+---------+
| Outer IPv4 Header | GRE Header | Inner IPv6 Header | Payload |
+-------------------+------------+-------------------+---------+
  Protocol = 47       Type = 0x86DD
```

The detailed IPv6 example is covered in [GRE Tunnels for IPv6 over IPv4](ipv6-over-ipv4.md).

---

## Key Points

- GRE adds a **GRE header** and a new **outer IP header** to the original passenger packet.
- The original packet, including its inner IP header, becomes the GRE payload.
- In basic GRE over IPv4, the overhead is **24 bytes**: 20 bytes for the outer IPv4 header and 4 bytes for GRE.
- The outer IPv4 **Protocol** field is **47**, identifying GRE. This is **not** a TCP or UDP port number.
- The GRE **Protocol Type** field identifies the passenger protocol, such as `0x0800` for IPv4 or `0x86DD` for IPv6.
- The basic GRE header is **4 bytes**; optional Checksum, Key, and Sequence Number fields can increase its size.
- The GRE Key is an identifier, **not** an encryption or authentication key.
- The ingress router first selects the tunnel using the **inner destination**, then forwards the encapsulated packet using the **outer destination**.
- Intermediate underlay routers only need a route to the **outer tunnel destination**.
- The egress tunnel endpoint removes the GRE and outer IP headers, then processes the original passenger packet.
- The **inner and outer TTL fields are separate**. Intermediate underlay routers decrement the outer TTL, not the inner TTL.
- GRE can carry IPv6 and multicast passenger traffic through an IPv4 unicast underlay.

---

## References

- [RFC 2784: Generic Routing Encapsulation (GRE)](https://www.rfc-editor.org/rfc/rfc2784.html)
- [RFC 2890: Key and Sequence Number Extensions to GRE](https://www.rfc-editor.org/rfc/rfc2890.html)
- [Cisco: Implementing Tunnels](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_ir-impl-tun-xe.html)
- [Cisco: Resolve IPv4 Fragmentation, MTU, MSS, and PMTUD Issues with GRE and IPsec](https://www.cisco.com/c/en/us/support/docs/ip/generic-routing-encapsulation-gre/25885-pmtud-ipfrag.html)
