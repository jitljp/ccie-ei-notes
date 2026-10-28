# GRE Tunnels for IPv6 over IPv4

**IPv6 over IPv4 GRE** allows two IPv6 networks to communicate across an IPv4-only network. The original IPv6 packet is encapsulated in a GRE header and an **outer IPv4 header**.

The intermediate IPv4 routers only need to forward packets between the **IPv4 tunnel endpoints**. They do not need IPv6 addresses, IPv6 routes, or IPv6 routing protocols.

```text
Original packet:   IPv6 header | IPv6 payload

Across underlay:  Outer IPv4 header | GRE header | IPv6 header | IPv6 payload
```

The tunnel uses `tunnel mode gre ip`, which is the default point-to-point GRE mode on IOS/IOS XE. The `ip` keyword identifies the **IPv4 transport**, not the passenger protocol.

---

## Why Use IPv6 over IPv4 GRE?

Suppose two sites use IPv6, but the network connecting them only supports IPv4:

```text
Site 1                         IPv4-only ISP                         Site 2
IPv6 LAN                                                             IPv6 LAN
   |                                                                    |
  R1 ----------------------------- ISP -------------------------------- R2
   |                                                                    |
   +======================= IPv6 over IPv4 GRE ========================+
```

Without a tunnel, the ISP cannot route native IPv6 traffic. GRE lets the sites exchange IPv6 packets while the ISP transports ordinary IPv4 packets.

This is useful for:

- Connecting IPv6 networks through an IPv4-only WAN or service provider.
- Running IPv6 routing protocols between routers whose underlay is IPv4-only.
- Carrying both IPv4 and IPv6 across the **same** point-to-point GRE tunnel.

GRE itself does **not** encrypt or authenticate the traffic.

---

## Tunnel Modes and Address Families

The **passenger protocol** is the original packet being transported. The **transport protocol** is the outer IP protocol used to move that packet between tunnel endpoints.

| Passenger | Transport | Tunnel Mode | Encapsulation |
| --- | --- | --- | --- |
| IPv4 | IPv4 | `gre ip` | IPv4 + GRE + IPv4 |
| **IPv6** | **IPv4** | **`gre ip`** | **IPv4 + GRE + IPv6** |
| IPv4 | IPv6 | `gre ipv6` | IPv6 + GRE + IPv4 |
| IPv6 | IPv6 | `gre ipv6` | IPv6 + GRE + IPv6 |

For IPv6-over-IPv4 GRE:

```text
IPv4 addresses:  tunnel source and tunnel destination
IPv6 addresses:  tunnel interfaces and remote IPv6 networks
Tunnel mode:     gre ip
```

**Do not use `tunnel mode gre ipv6` for an IPv4 underlay.** That mode specifies IPv6 as the *outer* transport protocol.

Also, **`tunnel mode ipv6ip` is not GRE**. It is a different way to carry IPv6 directly inside IPv4, covered later on this page.

---

## Topology

Continue with the R1, ISP, and R2 topology from [Point-to-Point GRE Configuration](configuration.md). The IPv4 underlay is unchanged.

```text
                  IPv4-only underlay

       192.0.2.0/30                 198.51.100.0/30

   R1 E0/0 ----------- ISP ---------------- E0/0 R2
   192.0.2.1          /   \                198.51.100.2
                     /     \
      +=======================================+
      |                 GRE                   |
      +=======================================+
      |                                       |
  R1 Tunnel0                              R2 Tunnel0
  10.0.12.1/30                            10.0.12.2/30
  2001:db8:12::1/64                       2001:db8:12::2/64
      |                                       |
  R1 Loopback1                            R2 Loopback1
  2001:db8:1::1/64                        2001:db8:2::2/64
```

| Component | R1 | R2 |
| --- | --- | --- |
| Physical IPv4 address | `192.0.2.1/30` | `198.51.100.2/30` |
| IPv4 tunnel destination | `198.51.100.2` | `192.0.2.1` |
| Tunnel IPv4 address (existing) | `10.0.12.1/30` | `10.0.12.2/30` |
| **Tunnel IPv6 address** | **`2001:db8:12::1/64`** | **`2001:db8:12::2/64`** |
| IPv6 site prefix | `2001:db8:1::/64` | `2001:db8:2::/64` |

The physical links and ISP are **IPv4-only**. IPv6 is configured on the **GRE tunnel** and on the simulated site interfaces.

We will keep the tunnel's existing IPv4 addresses to demonstrate **dual-stack GRE**. A GRE tunnel carrying IPv6 does not actually need an IPv4 address assigned to `Tunnel0`.

---

## Step 1: Verify the IPv4 Underlay

The tunnel endpoints must be able to reach each other using **IPv4**, independently of the GRE tunnel.

The relevant routes from the previous configuration page are:

### R1

```cisco
interface Ethernet0/0
 ip address 192.0.2.1 255.255.255.252
 no shutdown
!
ip route 198.51.100.2 255.255.255.255 192.0.2.2
```

### ISP

```cisco
interface Ethernet0/0
 ip address 192.0.2.2 255.255.255.252
 no shutdown
!
interface Ethernet0/1
 ip address 198.51.100.1 255.255.255.252
 no shutdown
```

### R2

```cisco
interface Ethernet0/0
 ip address 198.51.100.2 255.255.255.252
 no shutdown
!
ip route 192.0.2.1 255.255.255.255 198.51.100.1
```

Test from both endpoints:

```cisco
R1# ping 198.51.100.2 source 192.0.2.1
R2# ping 192.0.2.1 source 198.51.100.2
```

The ISP needs **no IPv6 configuration**. It forwards GRE packets according to their outer IPv4 destination addresses.

---

## Step 2: Add IPv6 to the GRE Tunnel

The existing GRE tunnel can carry IPv6 without changing its transport mode.

### R1

```cisco
ipv6 unicast-routing
!
interface Tunnel0
 ip address 10.0.12.1 255.255.255.252
 ipv6 address 2001:db8:12::1/64
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.2
 no shutdown
!
interface Loopback1
 ip address 10.1.1.1 255.255.255.0
 ipv6 address 2001:db8:1::1/64
```

### R2

```cisco
ipv6 unicast-routing
!
interface Tunnel0
 ip address 10.0.12.2 255.255.255.252
 ipv6 address 2001:db8:12::2/64
 tunnel source Ethernet0/0
 tunnel destination 192.0.2.1
 no shutdown
!
interface Loopback1
 ip address 10.2.2.2 255.255.255.0
 ipv6 address 2001:db8:2::2/64
```

**`tunnel mode gre ip` is omitted because it is the default.** You could explicitly add it on both routers:

```cisco
R1(config-if)# tunnel mode gre ip
```

Important commands:

| Command | Purpose |
| --- | --- |
| `ipv6 unicast-routing` | Enables IPv6 packet forwarding on the router. |
| `ipv6 address 2001:db8:12::1/64` | Assigns an IPv6 address to the **overlay** tunnel interface. |
| `tunnel source Ethernet0/0` | Selects the local **IPv4 underlay** source address. |
| `tunnel destination 198.51.100.2` | Identifies the remote **IPv4 underlay** endpoint. |
| `tunnel mode gre ip` | Uses GRE over IPv4, whether the passenger is IPv4 or IPv6. This is the default mode. |

The routers don't need an IPv6 address on `Ethernet0/0` in this example. They use IPv4 for transport and IPv6 for the tunneled traffic.

Adding `ipv6 address` to the tunnel interface does **not** change the outer IP header from IPv4 to IPv6.

---

## Step 3: Verify IPv6 Connectivity Across the Tunnel

First, verify the tunnel interface:

```cisco
R1# show interfaces Tunnel0
R1# show ipv6 interface Tunnel0
R1# show ipv6 interface brief
```

`show interfaces Tunnel0` should identify `GRE/IP` as the tunnel protocol/transport. `show ipv6 interface Tunnel0` should show the configured global unicast address, and normally a link-local IPv6 address as well.

The following IPv6 prefix is **directly connected** on both routers:

```text
2001:db8:12::/64
```

Test the remote tunnel interface from R1:

```cisco
R1# ping ipv6 2001:db8:12::2
```

And from R2:

```cisco
R2# ping ipv6 2001:db8:12::1
```

These tests require no static **IPv6** routes, because both tunnel IPv6 addresses belong to the same connected prefix.

However, they still require the **IPv4 underlay** routes to the tunnel destinations.

> An `up/up` tunnel interface is not, by itself, proof that IPv6 traffic can successfully pass between the endpoints. Test actual IPv6 reachability.

---

## Step 4: Route Between the Remote IPv6 Networks

At this point, R1 can reach R2's IPv6 address on `Tunnel0`, but it does not yet have a route to the **remote site's IPv6 prefix**, `2001:db8:2::/64`.

Likewise, R2 needs a route to `2001:db8:1::/64`.

Configure IPv6 static routes through the GRE tunnel.

### R1

```cisco
ipv6 route 2001:db8:2::/64 Tunnel0 2001:db8:12::2
```

### R2

```cisco
ipv6 route 2001:db8:1::/64 Tunnel0 2001:db8:12::1
```

The IPv6 routes use the **overlay next-hop addresses**, not the outer IPv4 tunnel destination addresses.

```text
R1:
  Underlay IPv4 route:
    198.51.100.2/32 -> 192.0.2.2 via Ethernet0/0

  Overlay IPv6 route:
    2001:db8:2::/64 -> 2001:db8:12::2 via Tunnel0
```

The IPv4 and IPv6 routing tables perform different jobs.

Check the IPv6 routes:

```cisco
R1# show ipv6 route 2001:db8:2::/64
R2# show ipv6 route 2001:db8:1::/64
```

Now test reachability between the simulated site addresses:

```cisco
R1# ping ipv6 2001:db8:2::2 source Loopback1
R2# ping ipv6 2001:db8:1::1 source Loopback1
```

Both tests should succeed when the underlay and tunnel are working correctly.

---

## IPv6 Packet Encapsulation

Suppose R1 sends an IPv6 packet to R2's Loopback1:

```text
Original IPv6 packet:

  Source:       2001:db8:1::1
  Destination:  2001:db8:2::2
```

R1 looks up `2001:db8:2::2` in its **IPv6 routing table**, which directs the packet out `Tunnel0`.

R1 then encapsulates it:

```text
+---------------------------------------+
| Outer IPv4 Header                     |
| Source:      192.0.2.1                |
| Destination: 198.51.100.2             |
| Protocol:    47 (GRE)                 |
+---------------------------------------+
| GRE Header                            |
| Protocol Type: 0x86DD (IPv6)          |
+---------------------------------------+
| Inner IPv6 Header                     |
| Source:      2001:db8:1::1            |
| Destination: 2001:db8:2::2            |
+---------------------------------------+
| Original IPv6 Payload                 |
+---------------------------------------+
```

Two protocol fields matter:

| Field | Value | Meaning |
| --- | --- | --- |
| Outer IPv4 **Protocol** | **47** | The outer IPv4 packet carries GRE. |
| GRE **Protocol Type** | **`0x86DD`** | The GRE payload is an IPv6 packet. |

The GRE Protocol Type for an **IPv4** passenger would instead be `0x0800`.

### Packet Forwarding Process

1. **R1 performs an IPv6 route lookup** for `2001:db8:2::2` and selects `Tunnel0`.
2. **R1 encapsulates** the IPv6 packet with a GRE header and an outer IPv4 header.
3. **R1 resolves `198.51.100.2` using IPv4 underlay routing** and sends the encapsulated packet toward the ISP.
4. **The ISP forwards only the outer IPv4 packet.** It does not need to examine the inner IPv6 destination.
5. **R2 receives and decapsulates the GRE packet**, removing the outer IPv4 and GRE headers.
6. **R2 processes the original IPv6 packet** and delivers it to `2001:db8:2::2` on Loopback1.

The underlay IPv4 routers decrement the **outer IPv4 TTL**. They do **not** decrement the inner IPv6 Hop Limit at each transit hop. IPv6 Hop Limit processing occurs when the IPv6 packet is forwarded by IPv6 routers.

---

## GRE IPv6-Only Overlay

The GRE tunnel does **not** need IPv4 addresses on `Tunnel0` merely because the transport is IPv4.

If the tunnel will carry only IPv6, you can omit the tunnel's IPv4 address:

### R1

```cisco
interface Tunnel0
 no ip address
 ipv6 address 2001:db8:12::1/64
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.2
```

### R2

```cisco
interface Tunnel0
 no ip address
 ipv6 address 2001:db8:12::2/64
 tunnel source Ethernet0/0
 tunnel destination 192.0.2.1
```

This still uses **GRE over IPv4** (`gre ip`).

The tunnel's **address family** and the **outer transport address family** are independent choices. If IPv6 is enabled without a global IPv6 address (for example, with `ipv6 enable`), the tunnel can also have a link-local address, which can be used by supported IPv6 routing and next-hop mechanisms.

---

## IPv6 over IPv4 GRE vs. Manual IPv6-in-IPv4

There are **two different types of static point-to-point IPv6-over-IPv4 tunnels** that are easy to confuse.

| Property | IPv6 over IPv4 GRE | Manual IPv6-in-IPv4 |
| --- | --- | --- |
| IOS/IOS XE tunnel mode | **`gre ip`** | **`ipv6ip`** |
| Outer IP header | IPv4 | IPv4 |
| Outer IPv4 Protocol field | **47 (GRE)** | **41 (IPv6 encapsulation)** |
| GRE header | Yes | No |
| Passenger protocols | IPv4, IPv6, other supported GRE protocols | IPv6 |
| Basic overhead with a 20-byte outer IPv4 header | **24 bytes** | **20 bytes** |
| Can use the same tunnel for IPv4 and IPv6 passengers | Yes | No |

For manual IPv6-in-IPv4, the key difference is the tunnel mode:

```cisco
interface Tunnel0
 no ip address
 ipv6 address 2001:db8:12::1/64
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.2
 tunnel mode ipv6ip
```

R2 would need a corresponding manual `ipv6ip` tunnel.

The packet format for this mode is:

```text
Outer IPv4 header | Inner IPv6 header | IPv6 payload
      Protocol = 41
```

The packet format for GRE is:

```text
Outer IPv4 header | GRE header | Inner IPv6 header | IPv6 payload
      Protocol = 47       Protocol Type = 0x86DD
```

**Do not configure `gre ip` on one endpoint and `ipv6ip` on the other.** The encapsulation protocols differ, so the endpoints cannot correctly exchange tunneled packets.

---

## IPv6 Routing Protocols over GRE

IPv6 routing protocols can run across the GRE tunnel because, from the overlay's perspective, `Tunnel0` is a point-to-point IPv6-capable interface.

For example, you can use **OSPFv3** to advertise IPv6 site prefixes between R1 and R2 instead of configuring static IPv6 routes.

```text
R1 ----------- GRE over IPv4 ----------- R2
    OSPFv3 for IPv6 routes over Tunnel0
```

GRE can carry IPv6 multicast traffic, including OSPFv3 control packets. The ISP does not have to participate in OSPFv3 or support native IPv6 multicast.

Detailed static and dynamic routing configurations are covered in [Routing over GRE Tunnels](routing.md).

---

## MTU and Fragmentation

Standard GRE over IPv4 adds at least **24 bytes** of encapsulation overhead:

```text
Outer IPv4 header: 20 bytes (without IP options)
GRE header:        4 bytes (without optional fields)
-----------------------------------------------
Total:            24 bytes
```

For a path that supports a 1500-byte outer IPv4 packet, the available tunnel payload MTU is **1476 bytes**, before accounting for any additional GRE or outer IP options.

```text
1500 - 20 - 4 = 1476 bytes
```

This calculation is the same whether the GRE passenger is IPv4 or IPv6. In this example, the **IPv6 packet, including its 40-byte IPv6 header**, must fit inside the GRE payload MTU to avoid additional fragmentation handling.

An important IPv6 requirement is that the tunnel must be able to carry an **IPv6 packet of at least 1280 bytes** without fragmenting the *inner IPv6 packet*. IPv6 routers do not fragment transit IPv6 packets; oversized IPv6 packets require appropriate MTU signaling or handling by the tunnel ingress.

Useful verification:

```cisco
R1# show interfaces Tunnel0
R1# show ipv6 interface Tunnel0
```

If the IPv4 underlay cannot accommodate the encapsulated packet, the tunnel may experience fragmentation or packet loss. MTU, outer IPv4 fragmentation, PMTUD, and MSS adjustment are covered in [GRE MTU, Fragmentation, and MSS](mtu-mss.md).

---

## Common Problems

| Symptom | Likely Cause | Check |
| --- | --- | --- |
| Tunnel interface down | IPv4 source/destination or underlay problem | `show interfaces Tunnel0`, `show ip route` |
| IPv4 underlay works, IPv6 tunnel ping fails | IPv6 not enabled/addressed on the tunnel, or GRE blocked | `show ipv6 interface Tunnel0`, ACLs permitting IP protocol 47 |
| Tunnel IPv6 ping succeeds, remote site IPv6 ping fails | Missing IPv6 route or return route | `show ipv6 route` on **both** endpoints |
| IPv6 packet is sent but not delivered | GRE encapsulation/mode mismatch, ACL, or MTU problem | Tunnel mode, protocol 47, tunnel MTU |
| Route to remote tunnel endpoint points into Tunnel0 | Recursive tunnel routing | IPv4 route to the **outer** tunnel destination |

Remember that **IPv4 underlay reachability and IPv6 overlay reachability are separate**. Successful IPv4 pings between the physical interfaces do not prove that IPv6 routing across GRE is working.

---

## Key Points

- IPv6 over IPv4 GRE carries **IPv6 passenger packets** through an **IPv4-only underlay**.
- The tunnel uses **`tunnel mode gre ip`**, the default point-to-point GRE mode.
- `gre ip` uses **IPv4 transport**. `gre ipv6` uses **IPv6 transport**.
- IPv4 addresses are used for `tunnel source` and `tunnel destination` in IPv6-over-IPv4 GRE.
- IPv6 addresses can be configured on `Tunnel0` without configuring IPv6 on the physical underlay interfaces.
- A GRE tunnel can carry **IPv4 and IPv6 simultaneously**.
- `ipv6 unicast-routing` enables IPv6 forwarding on the endpoint routers.
- Static IPv6 routes or IPv6 routing protocols provide reachability to **remote IPv6 prefixes** across the tunnel.
- Outer IPv4 **Protocol = 47** identifies GRE. GRE **Protocol Type = `0x86DD`** identifies an IPv6 passenger.
- Intermediate IPv4 routers forward based on the **outer IPv4 destination**, not the inner IPv6 destination.
- Standard IPv6-over-IPv4 GRE adds **24 bytes** of headers (20-byte outer IPv4 header + 4-byte GRE header).
- **`tunnel mode ipv6ip` is not GRE**. It carries IPv6 directly over IPv4 using **IP protocol 41**, with no GRE header.
- GRE provides **no encryption or authentication** by itself.

---

## References

- [Cisco: Implementing Tunnels (IOS XE)](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_ir-impl-tun-xe.html)
- [Cisco: IPv6 over IPv4 GRE Tunnels](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/interface/configuration/15-sy/ir-15-sy-book/ip6-ip4-gre-tunls.html)
- [RFC 2784: Generic Routing Encapsulation (GRE)](https://www.rfc-editor.org/rfc/rfc2784.html)
- [RFC 7676: IPv6 Support for GRE](https://www.rfc-editor.org/rfc/rfc7676.html)
