# Point-to-Point GRE Configuration

A **static point-to-point GRE tunnel** is configured with a logical tunnel interface, a **tunnel source**, and a **tunnel destination** on each endpoint.

The routers must already have IP connectivity through the **underlay**. Once the tunnel is configured, the tunnel interfaces form a logical point-to-point link that can carry IPv4, IPv6, and routing-protocol traffic.

This page builds a basic **IPv4-over-IPv4 GRE tunnel** between R1 and R2. IPv6 over an IPv4 underlay, dynamic routing, GRE keepalives, MTU, and VRFs are covered in separate pages.

---

## Topology

Use the following topology:

```text
            Underlay: IPv4 network

       192.0.2.0/30            198.51.100.0/30

R1 E0/0 ---------------- ISP ---------------- E0/0 R2
192.0.2.1     192.0.2.2  198.51.100.1     198.51.100.2
    |                                               |
    |                  GRE Tunnel                   |
    +===============================================+
    |                                               |
R1 Tunnel0                                     R2 Tunnel0
10.0.12.1/30                                   10.0.12.2/30
    |                                               |
Loopback1                                       Loopback1
10.1.1.1/24                                     10.2.2.2/24
```

The ISP router represents the underlay. It forwards normal IPv4 packets between the two tunnel endpoints and does **not** need GRE configured.

| Component | R1 | R2 |
| --- | --- | --- |
| Underlay address | `192.0.2.1` | `198.51.100.2` |
| Tunnel interface | `Tunnel0` | `Tunnel0` |
| Overlay address | `10.0.12.1/30` | `10.0.12.2/30` |
| Tunnel source | `192.0.2.1` | `198.51.100.2` |
| Tunnel destination | `198.51.100.2` | `192.0.2.1` |
| Simulated LAN address | `10.1.1.1/24` | `10.2.2.2/24` |

The tunnel source on one router must correspond to the tunnel destination on the other router.

```text
R1 tunnel source      = 192.0.2.1
R2 tunnel destination = 192.0.2.1

R2 tunnel source      = 198.51.100.2
R1 tunnel destination = 198.51.100.2
```

---

## Step 1: Configure Underlay Connectivity

Configure the physical interfaces and routes needed to reach the remote tunnel endpoints.

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

The ISP has both underlay networks directly connected, so no additional static routes are needed on it.

### R2

```cisco
interface Ethernet0/0
 ip address 198.51.100.2 255.255.255.252
 no shutdown
!
ip route 192.0.2.1 255.255.255.255 198.51.100.1
```

The static **host routes** ensure that each router reaches the remote tunnel destination through the ISP, rather than through the GRE tunnel.

Before configuring GRE, verify the underlay:

```cisco
R1# ping 198.51.100.2 source 192.0.2.1
R2# ping 192.0.2.1 source 198.51.100.2
```

Both pings should succeed. If underlay connectivity fails, GRE will not fix the underlying routing problem.

---

## Step 2: Configure the GRE Tunnel

Create `Tunnel0` on both routers.

### R1

```cisco
interface Tunnel0
 ip address 10.0.12.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.2
 no shutdown
```

### R2

```cisco
interface Tunnel0
 ip address 10.0.12.2 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 192.0.2.1
 no shutdown
```

Notice that neither configuration includes `tunnel mode gre ip`. **GRE over IPv4 is already the default tunnel mode** on a Cisco IOS/IOS XE tunnel interface. The `tunnel mode` command is not required for this example.

The tunnel interfaces now provide a logical connection across the ISP network:

```text
R1 Tunnel0 ---------------- R2 Tunnel0
10.0.12.1                    10.0.12.2
```

The underlay sees GRE-encapsulated IPv4 packets traveling between `192.0.2.1` and `198.51.100.2`. The overlay sees the tunnel interfaces as directly connected.

### GRE Configuration Commands

| Command | Purpose |
| --- | --- |
| `interface Tunnel0` | Creates or enters the logical tunnel interface. |
| `ip address 10.0.12.1 255.255.255.252` | Assigns an overlay IPv4 address. |
| `tunnel source Ethernet0/0` | Uses the IPv4 address of the specified local interface as the outer source address. |
| `tunnel destination 198.51.100.2` | Sets the remote outer IP destination. |
| `tunnel mode gre ip` | Explicitly selects GRE encapsulation over IPv4 (the default). |
| `no shutdown` | Administratively enables the tunnel interface. |

> The `ip address` command assigns the **inner/overlay** address. The `tunnel source` and `tunnel destination` commands determine the **outer/underlay** addresses. These are different roles.

---

## Default Tunnel Mode

The default mode for a normal IOS/IOS XE tunnel interface is **point-to-point GRE over IPv4** (`gre ip`).

These two configurations have the same tunnel mode:

```cisco
interface Tunnel0
 ip address 10.0.12.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.2
```

```cisco
interface Tunnel0
 ip address 10.0.12.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.2
 tunnel mode gre ip
```

Because `gre ip` is the default, it typically does not appear in `show running-config` when it has not been explicitly configured. However, `show interfaces Tunnel0` still identifies the protocol and transport as `GRE/IP`.

The command to explicitly select the default mode is:

```cisco
R1(config-if)# tunnel mode gre ip
```

To restore the default after changing the mode:

```cisco
R1(config-if)# no tunnel mode
```

**The `ip` keyword refers to the outer IPv4 transport, not the inner passenger protocol.** A `gre ip` tunnel can carry either IPv4 or IPv6 packets (and other supported GRE passenger protocols).

---

## Other Tunnel Modes

The `tunnel mode` command selects the encapsulation and transport used by the tunnel interface. Not every mode uses GRE.

| Tunnel Mode | Encapsulation / Outer Transport | Passenger Traffic | Notes |
| --- | --- | --- | --- |
| `gre ip` | GRE over IPv4 | IPv4, IPv6, other supported protocols | Default point-to-point GRE mode. |
| `gre ipv6` | GRE over IPv6 | IPv4, IPv6, other supported protocols | IPv6 addresses are used as the tunnel source/destination. |
| `gre multipoint` | Multipoint GRE over IPv4 | IPv4, IPv6, other supported protocols | Supports multiple remote peers; commonly used for DMVPN. |
| `ipip` | IP-in-IP over IPv4 | IPv4 | No GRE header; IP protocol 4. |
| `ipv6ip` | IPv6-in-IPv4, without GRE | IPv6 | Manual IPv6-over-IPv4 tunneling; IP protocol 41. |
| `ipsec ipv4` | IPsec VTI over IPv4 | IP routed over an IPsec tunnel | Not a GRE tunnel; uses IPsec for protection. |
| `ipsec ipv6` | IPsec VTI over IPv6 | IP routed over an IPsec tunnel | Not a GRE tunnel; uses an IPv6 transport. |

The important distinction is between the **passenger protocol** (the original packet) and the **transport protocol** (the outer IP header).

For example:

```text
IPv4 over IPv4 GRE:  tunnel mode gre ip
IPv6 over IPv4 GRE:  tunnel mode gre ip
IPv4 over IPv6 GRE:  tunnel mode gre ipv6
IPv6 over IPv6 GRE:  tunnel mode gre ipv6
```

**IPv6 over IPv4 GRE does not require `tunnel mode ipv6ip`.** That mode uses direct IPv6-in-IPv4 encapsulation without a GRE header. This difference is explained further in [GRE Tunnels for IPv6 over IPv4](ipv6-over-ipv4.md).

For static point-to-point GRE, the two ends should use compatible GRE modes and the appropriate source/destination address family. Multipoint GRE and IPsec VTI are different tunnel designs, not alternate settings needed for this basic lab.

---

## Step 3: Verify the Tunnel

### Check Tunnel Interface Status

On R1:

```cisco
R1# show interfaces Tunnel0
```

Relevant fields in the output should resemble:

```text
Tunnel0 is up, line protocol is up
  Hardware is Tunnel
  Internet address is 10.0.12.1/30
  Tunnel source 192.0.2.1 (Ethernet0/0), destination 198.51.100.2
  Tunnel protocol/transport GRE/IP
```

The tunnel mode is shown as `GRE/IP` because the GRE packets are transported over IPv4.

**An up/up GRE tunnel does not, by itself, prove that the remote router is correctly configured or receiving GRE packets.** Basic GRE has no session establishment or handshake, and remote failure detection requires additional mechanisms.

### Check the Route to the Tunnel Destination

On R1:

```cisco
R1# show ip route 198.51.100.2
```

The route should point toward the ISP:

```text
198.51.100.2/32 via 192.0.2.2
```

The route must **not** resolve through `Tunnel0`. If it does, the tunnel may experience a recursive routing failure.

### Ping the Remote Tunnel Address

From R1:

```cisco
R1# ping 10.0.12.2 source 10.0.12.1
```

From R2:

```cisco
R2# ping 10.0.12.1 source 10.0.12.2
```

Successful pings demonstrate that the two endpoints can encapsulate, transport, and decapsulate packets over the GRE tunnel.

The original ICMP packets have tunnel IP addresses as their source and destination. The GRE transport packets use the **underlay** addresses.

```text
Inner IPv4 packet:
  10.0.12.1 -> 10.0.12.2

Outer IPv4 packet:
  192.0.2.1 -> 198.51.100.2
  Protocol: 47 (GRE)
```

---

## Step 4: Route Traffic Across the Tunnel

The tunnel interface and its connected subnet allow R1 and R2 to reach each other's **tunnel addresses**. To carry traffic between networks behind the routers, configure overlay routes.

Use loopback interfaces to simulate the LANs shown in the topology.

### R1

```cisco
interface Loopback1
 ip address 10.1.1.1 255.255.255.0
!
ip route 10.2.2.0 255.255.255.0 10.0.12.2
```

### R2

```cisco
interface Loopback1
 ip address 10.2.2.2 255.255.255.0
!
ip route 10.1.1.0 255.255.255.0 10.0.12.1
```

The static routes use the **remote tunnel IP address** as the next hop, so traffic to the remote LAN is encapsulated in GRE.

Verify on R1:

```cisco
R1# show ip route 10.2.2.0
R1# ping 10.2.2.2 source 10.1.1.1
```

Verify on R2:

```cisco
R2# show ip route 10.1.1.0
R2# ping 10.1.1.1 source 10.2.2.2
```

The forwarding process from R1 to R2 is:

```text
Original packet: 10.1.1.1 -> 10.2.2.2
    |
    v
R1 route to 10.2.2.0/24 via 10.0.12.2 (Tunnel0)
    |
    v
R1 encapsulates the packet in GRE
Outer packet: 192.0.2.1 -> 198.51.100.2
    |
    v
ISP forwards using the outer destination IP
    |
    v
R2 decapsulates GRE and receives the original packet
    |
    v
R2 delivers the packet to 10.2.2.2
```

The ISP needs routes to the **tunnel endpoints**, not to the LANs carried inside GRE.

Dynamic routing protocols can also run across the tunnel, as covered in the *Routing over GRE Tunnels* page.

---

## Tunnel Source: Interface or IP Address

The `tunnel source` command accepts either an interface or an IP address.

### Specify an Interface

```cisco
R1(config-if)# tunnel source Ethernet0/0
```

The tunnel uses the IPv4 address of `Ethernet0/0` as the outer source address.

If that interface's IPv4 address changes, the tunnel source can follow the new interface address, provided the remaining tunnel configuration and reachability are valid.

### Specify an IP Address

```cisco
R1(config-if)# tunnel source 192.0.2.1
```

This explicitly specifies the outer source address. The address must be configured locally on the router.

Both options work in the basic topology. A **loopback interface** can also be used as the source, which is useful when the underlay provides multiple paths and the endpoint should not be tied to a single physical interface.

Example on R1:

```cisco
interface Loopback0
 ip address 203.0.113.11 255.255.255.255
!
interface Tunnel0
 tunnel source Loopback0
 tunnel destination 203.0.113.22
```

For this variation, R2 would use `203.0.113.22` as its tunnel source and `203.0.113.11` as its destination. Both routers and the intervening underlay must have routes to the remote loopback address **outside the tunnel**.

---

## Tunnel Interface Number and Addressing Options

The tunnel interface number is **locally significant**. The endpoints do **not** have to use the same number:

```text
R1: Tunnel0  <------------ GRE ------------>  R2: Tunnel100
```

The routers identify the GRE tunnel using the transport endpoints and relevant encapsulation information, not by exchanging tunnel interface numbers.

An ordinary point-to-point GRE tunnel commonly has an IPv4 address on each tunnel interface, as in the basic lab. Other interface-addressing options are available, including `ip unnumbered`, which borrows an IPv4 address from another local interface:

```cisco
interface Tunnel0
 ip unnumbered Loopback0
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.2
```

`ip unnumbered` is an alternative to assigning a separate IPv4 address to the tunnel interface. It does not eliminate the need for overlay routing or for underlay reachability to the tunnel destination.

---

## Optional GRE Tunnel Key

A point-to-point GRE tunnel can include a **key** in the GRE header.

Configure the same key on both tunnel endpoints:

### R1

```cisco
interface Tunnel0
 tunnel key 100
```

### R2

```cisco
interface Tunnel0
 tunnel key 100
```

The key is a **32-bit tunnel identifier**, not an encryption key or authentication secret. It can help identify or distinguish GRE traffic, but it does not provide cryptographic security.

If the endpoints are configured with different keys, GRE packets can be rejected instead of being delivered to the intended tunnel.

The key is **optional**. It is not needed for the basic configuration in this page.

A GRE key adds a 4-byte Key field to the GRE header, increasing encapsulation overhead. The allowed key value is a 32-bit unsigned integer (`0` to `4294967295`).

---

## Other Optional Tunnel Interface Commands

In addition to `tunnel source`, `tunnel destination`, `tunnel mode`, and `tunnel key`, IOS/IOS XE supports several other options.

### GRE Checksum

```cisco
R1(config-if)# tunnel checksum
```

Enables the optional GRE checksum for the encapsulated packet. GRE checksumming is **disabled by default**. The Checksum and Reserved1 fields add **4 bytes** to the basic GRE header.

A GRE checksum detects certain corruption of a GRE packet. It does **not** provide cryptographic integrity or authentication.

### GRE Sequence Numbers

```cisco
R1(config-if)# tunnel sequence-datagrams
```

Enables the optional GRE Sequence Number field and causes out-of-order received datagrams to be discarded. This behavior is **disabled by default** and can be useful for passenger protocols sensitive to out-of-order delivery.

The Sequence Number field adds **4 bytes** to the GRE header. It does not turn GRE into a reliable transport protocol or retransmit lost packets.

### Outer IPv4 TTL

```cisco
R1(config-if)# tunnel ttl 64
```

Sets the TTL used in the **outer IPv4 header** of encapsulated packets. The documented default tunnel TTL is **255**.

This is separate from the TTL of the original IPv4 passenger packet. Intermediate underlay routers decrement the **outer** TTL.

### Outer IPv4 ToS

```cisco
R1(config-if)# tunnel tos 184
```

Sets the outer IPv4 ToS byte to a specified value (`0` to `255`). The example value of `184` corresponds to DSCP EF (`46`) with ECN bits set to `00`.

By default, when the passenger packet is IP, the tunnel copies its ToS byte to the outer header. For non-IP passenger protocols, the default outer ToS value is `0`.

### Tunnel Bandwidth and Delay

```cisco
interface Tunnel0
 bandwidth 10000
 delay 1000
```

These commands change interface attributes used by routing protocols and other features:

- `bandwidth 10000` sets the **configured interface bandwidth** to 10,000 kbps (10 Mbps).
- `delay 1000` sets the interface delay used by routing metrics.

**Neither command increases the actual bandwidth of the tunnel or directly rate-limits its traffic.** The underlying network determines the available forwarding capacity.

### Other Useful Options

| Command | Purpose | Covered In |
| --- | --- | --- |
| `keepalive 10 3` | Sends GRE keepalives every 10 seconds and declares failure after 3 missed responses. | GRE Tunnel State and Keepalives |
| `ip mtu 1476` | Sets the tunnel IPv4 MTU to 1476 bytes. | GRE MTU, Fragmentation, and MSS |
| `ip tcp adjust-mss 1436` | Adjusts TCP MSS in packets passing through the interface. | GRE MTU, Fragmentation, and MSS |
| `tunnel path-mtu-discovery` | Enables PMTUD across the tunnel transport path. | GRE MTU, Fragmentation, and MSS |
| `tunnel vrf UNDERLAY` | Uses the named VRF to resolve the remote tunnel destination. | GRE Tunnels and VRFs |
| `tunnel protection ipsec profile PROFILE` | Applies an IPsec profile to protect tunnel traffic. | GRE with IPsec (outside this section) |

The MTU and MSS values shown above are examples appropriate to a simple 1500-byte IPv4 underlay with 24 bytes of basic GRE overhead, not universal values for all tunnel designs.

---

## Common Configuration Problems

| Problem | What to Check |
| --- | --- |
| Tunnel interface is down | Verify the tunnel source, source interface state, tunnel destination, and route to the destination. |
| Tunnel is up/up, but traffic fails | Verify the remote configuration, GRE protocol 47 filtering, and return-path reachability. |
| Tunnel IP addresses cannot ping | Verify matching endpoint addresses, underlay connectivity, and GRE encapsulation/decapsulation. |
| Tunnel IPs are reachable, but remote LANs are not | Check overlay routes in both directions. |
| Tunnel repeatedly goes down | Check whether the tunnel destination route is being resolved through the tunnel itself. |
| GRE key has been configured | Confirm that the endpoints use matching tunnel keys. |

The detailed failure-detection behavior of GRE is covered in *GRE Tunnel State and Keepalives*. MTU, fragmentation, and MSS are covered in *GRE MTU, Fragmentation, and MSS*.

---

## Key Points

- A static point-to-point GRE tunnel requires a tunnel source and tunnel destination on each endpoint.
- The source address on one endpoint corresponds to the destination address on the other endpoint.
- `interface Tunnel0` creates a logical Layer 3 tunnel interface.
- `ip address` configures the **overlay** IP address.
- `tunnel source` and `tunnel destination` define the **underlay** endpoints used in the outer IP header.
- The tunnel source can be an interface or a locally configured IP address.
- `tunnel mode gre ip` explicitly configures GRE over IPv4, but the command is **optional** because this is the default tunnel mode on Cisco IOS/IOS XE.
- `no tunnel mode` restores the default tunnel mode.
- `gre ip` can carry either IPv4 or IPv6 passenger traffic over an IPv4 underlay.
- `gre ipv6` transports GRE over IPv6; `ipv6ip` transports IPv6 over IPv4 **without GRE**.
- `gre multipoint`, `ipip`, and `ipsec ipv4` are different tunnel modes, not requirements for static point-to-point GRE.
- Tunnel interface numbers are locally significant and do not need to match between endpoints.
- Underlay connectivity to the remote tunnel endpoint must exist independently of the GRE tunnel.
- Overlay routing is needed to carry traffic between networks behind the tunnel endpoints.
- The underlay forwards packets using the **outer** destination IP address and does not need to know the inner destination networks.
- GRE uses **IP protocol 47**, not TCP or UDP port 47.
- An up/up tunnel interface does not guarantee that GRE traffic reaches or is accepted by the remote endpoint.
- `tunnel key` is optional and does **not** provide encryption or strong authentication.
- `tunnel checksum` and `tunnel sequence-datagrams` are optional and disabled by default; each adds 4 bytes to the GRE header.
- `tunnel ttl` and `tunnel tos` control fields in the outer IPv4 header.
- `bandwidth` and `delay` affect interface attributes used by routing protocols; they do not set actual tunnel throughput.

---

## References

- [Cisco: Implementing Tunnels (IOS XE IP Routing Configuration Guide)](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_ir-impl-tun-xe.html)
- [Cisco: Tunnel Source and Destination Commands (IOS Interface Command Reference)](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/interface/command/ir-cr-book/ir-t2.html)
- [Cisco: GRE Tunnel IP Source and Destination VRF Membership (IOS XE 17)](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr3-fwd/gre/gre-configuration-guide/m-gre-tunnel-ip-source-and-destination-vrf-membership.html)
- [Cisco: IPv4 and IPv6 Tunnel Modes (Implementing Tunnels)](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_ir-impl-tun-xe.html)
