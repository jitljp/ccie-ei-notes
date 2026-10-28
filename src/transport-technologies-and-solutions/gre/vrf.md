# GRE Tunnels and VRFs

A **GRE tunnel** can carry traffic belonging to one VRF across an underlay network that uses a **different VRF** or the **global routing table**.

This is called **VRF-aware GRE**. The important distinction is that GRE forwarding involves **two separate routing lookups**:

1. An **overlay lookup** for the original (inner) IP packet.
2. An **underlay lookup** for the GRE tunnel destination (outer IP packet).

On Cisco IOS XE, the tunnel interface and the tunnel transport can use different routing tables.

```text
                       R1

        VRF CUSTOMER (overlay)
                  |
       Original customer packet
                  |
               Tunnel0  <--- vrf forwarding CUSTOMER
                  |
             GRE encapsulation
                  |
        VRF TRANSPORT (underlay)
                  |
         Outer IPv4 packet     <--- tunnel vrf TRANSPORT
                  |
              Ethernet0/0
                  |
                 ISP
```

The remote router decapsulates the packet and forwards the original packet using the VRF assigned to its tunnel interface.

---

## The Two VRF Commands

These two commands have completely different purposes:

```cisco
interface Tunnel0
 vrf forwarding CUSTOMER
 tunnel vrf TRANSPORT
```

| Command | Function | Routing lookup |
| --- | --- | --- |
| `vrf forwarding CUSTOMER` | Assigns `Tunnel0` to the `CUSTOMER` VRF. | **Inner** IP packet (overlay) |
| `tunnel vrf TRANSPORT` | Specifies the VRF used to reach the GRE tunnel destination. | **Outer** IP packet (underlay) |

### `vrf forwarding`

```cisco
interface Tunnel0
 vrf forwarding CUSTOMER
 ip address 10.0.12.1 255.255.255.252
```

The connected route for `10.0.12.0/30` is installed in `CUSTOMER`, not the global routing table.

Traffic **sent into** `Tunnel0` is selected by routes in the `CUSTOMER` VRF. Traffic **received from** the tunnel is decapsulated and processed in that VRF.

As with other Layer 3 interfaces, adding or changing `vrf forwarding` normally removes existing IP addresses. Configure VRF membership **before** the interface IP address.

### `tunnel vrf`

```cisco
interface Tunnel0
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.2
 tunnel vrf TRANSPORT
```

The router resolves `198.51.100.2` using the routing table for `TRANSPORT`.

```text
R1 looks for 198.51.100.2 in:

  show ip route vrf TRANSPORT

NOT:

  show ip route vrf CUSTOMER
  show ip route
```

The GRE transport source should be reachable through the transport routing context. Typically, the source interface itself belongs to `TRANSPORT`.

**Default:** Without `tunnel vrf`, IPv4 GRE tunnel destination resolution uses the **global routing table**.

**Default:** Without `vrf forwarding` on `Tunnel0`, the tunnel interface belongs to the **global routing table**.

These defaults are separate.

---

## Common Underlay and Overlay Combinations

| Tunnel interface (overlay) | Tunnel transport (underlay) | Tunnel interface configuration |
| --- | --- | --- |
| Global | Global | Neither VRF command |
| `CUSTOMER` | Global | `vrf forwarding CUSTOMER` |
| Global | `TRANSPORT` | `tunnel vrf TRANSPORT` |
| `CUSTOMER` | `TRANSPORT` | Both commands |
| `CUSTOMER` | `CUSTOMER` | Both commands, referencing the same VRF |

The overlay and transport VRFs **may** be the same, but they do not need to be.

A VRF name is **locally significant**. The GRE header does not contain a VRF name or identifier. Each endpoint uses its own local routing tables and tunnel configuration.

---

## Lab Topology

The following examples use the same R1, ISP, and R2 underlay as the earlier GRE pages.

```text
                     IPv4 Underlay

R1 E0/0 ---------------- ISP ---------------- E0/0 R2
192.0.2.1                                   198.51.100.2
   |                                              |
   |              Point-to-point GRE              |
   +==============================================+
   |                                              |
R1 Tunnel0                                    R2 Tunnel0
10.0.12.1/30                                  10.0.12.2/30
   |                                              |
R1 Loopback1                                  R2 Loopback1
10.1.1.1/32                                   10.2.2.2/32

ISP side toward R1: 192.0.2.254/24
ISP side toward R2: 198.51.100.254/24
```

The roles of the addresses are:

| Address | Purpose | Routing table |
| --- | --- | --- |
| `192.0.2.1` | R1 tunnel source | Underlay |
| `198.51.100.2` | R2 tunnel source | Underlay |
| `10.0.12.1` | R1 tunnel interface IP | Overlay |
| `10.0.12.2` | R2 tunnel interface IP | Overlay |
| `10.1.1.1` | R1 customer network | Overlay |
| `10.2.2.2` | R2 customer network | Overlay |

The ISP uses ordinary global routing and does not need routes to the tunnel subnet or customer loopbacks.

### ISP Configuration

The ISP configuration is the same for both main lab scenarios:

```cisco
hostname ISP
!
interface Ethernet0/0
 ip address 192.0.2.254 255.255.255.0
 no shutdown
!
interface Ethernet0/1
 ip address 198.51.100.254 255.255.255.0
 no shutdown
```

The ISP has directly connected routes to both tunnel-source subnets.

---

## Scenario 1: Customer VRF over the Global Routing Table

This scenario uses:

- **Overlay:** `CUSTOMER`
- **Underlay:** Global routing table

```text
R1                                        R2

CUSTOMER VRF                              CUSTOMER VRF
  Lo1: 10.1.1.1                             Lo1: 10.2.2.2
  Tu0: 10.0.12.1  =======================  Tu0: 10.0.12.2
                     GRE tunnel

GLOBAL                                     GLOBAL
  E0/0: 192.0.2.1 ---- ISP -------------  E0/0: 198.51.100.2
```

Here, only `vrf forwarding CUSTOMER` is required on each tunnel interface. No `tunnel vrf` command is needed because the underlay is global.

### R1 Configuration

```cisco
hostname R1
!
vrf definition CUSTOMER
 address-family ipv4
 exit-address-family
!
interface Ethernet0/0
 ip address 192.0.2.1 255.255.255.0
 no shutdown
!
interface Loopback1
 vrf forwarding CUSTOMER
 ip address 10.1.1.1 255.255.255.255
!
interface Tunnel0
 vrf forwarding CUSTOMER
 ip address 10.0.12.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.2
!
ip route 198.51.100.2 255.255.255.255 192.0.2.254
ip route vrf CUSTOMER 10.2.2.2 255.255.255.255 10.0.12.2
```

The default tunnel mode is `gre ip`, so no explicit `tunnel mode gre ip` command is required.

The first static route provides **underlay** reachability through the ISP:

```cisco
ip route 198.51.100.2 255.255.255.255 192.0.2.254
```

The second static route provides **overlay** reachability through GRE:

```cisco
ip route vrf CUSTOMER 10.2.2.2 255.255.255.255 10.0.12.2
```

### R2 Configuration

```cisco
hostname R2
!
vrf definition CUSTOMER
 address-family ipv4
 exit-address-family
!
interface Ethernet0/0
 ip address 198.51.100.2 255.255.255.0
 no shutdown
!
interface Loopback1
 vrf forwarding CUSTOMER
 ip address 10.2.2.2 255.255.255.255
!
interface Tunnel0
 vrf forwarding CUSTOMER
 ip address 10.0.12.2 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 192.0.2.1
!
ip route 192.0.2.1 255.255.255.255 198.51.100.254
ip route vrf CUSTOMER 10.1.1.1 255.255.255.255 10.0.12.1
```

### Verification

Verify **outer** endpoint reachability in the global table:

```cisco
R1# show ip route 198.51.100.2
R1# ping 198.51.100.2 source 192.0.2.1
```

Verify that the tunnel subnet and customer routes belong to `CUSTOMER`:

```cisco
R1# show vrf interfaces
R1# show ip route vrf CUSTOMER
R1# show ip interface Tunnel0
R1# show interfaces Tunnel0
```

Test the tunnel IP, then the remote customer prefix:

```cisco
R1# ping vrf CUSTOMER 10.0.12.2 source 10.0.12.1
R1# ping vrf CUSTOMER 10.2.2.2 source 10.1.1.1
```

> A successful ping of the outer tunnel destination does **not** prove that the overlay is configured correctly. Always test both routing contexts.

---

## Scenario 2: Customer VRF over a Separate Transport VRF

This is the most important VRF-aware GRE scenario.

- **Overlay:** `CUSTOMER`
- **Underlay:** `TRANSPORT`

The GRE outer packet travels through `TRANSPORT`, while the original customer packet remains in `CUSTOMER`.

```text
            R1                                   R2

     +----------------+                    +----------------+
     | VRF CUSTOMER   |                    | VRF CUSTOMER   |
     | Lo1 10.1.1.1   |                    | Lo1 10.2.2.2   |
     | Tu0 10.0.12.1  |====================| Tu0 10.0.12.2  |
     +-------+--------+    GRE overlay     +--------+-------+
             |                                      |
      GRE encapsulation                      GRE decapsulation
             |                                      |
     +-------+--------+                    +--------+-------+
     | VRF TRANSPORT  |                    | VRF TRANSPORT  |
     | E0/0 192.0.2.1 |---- ISP ----------| E0/0 198.51.100.2|
     +----------------+                    +------------------+
```

The essential tunnel configuration is:

```cisco
interface Tunnel0
 vrf forwarding CUSTOMER
 ip address 10.0.12.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.2
 tunnel vrf TRANSPORT
```

The key difference from Scenario 1 is `tunnel vrf TRANSPORT`.

### R1 Configuration

This is a complete, **alternative** configuration to Scenario 1, not an addition to it.

```cisco
hostname R1
!
vrf definition TRANSPORT
 address-family ipv4
 exit-address-family
!
vrf definition CUSTOMER
 address-family ipv4
 exit-address-family
!
interface Ethernet0/0
 vrf forwarding TRANSPORT
 ip address 192.0.2.1 255.255.255.0
 no shutdown
!
interface Loopback1
 vrf forwarding CUSTOMER
 ip address 10.1.1.1 255.255.255.255
!
interface Tunnel0
 vrf forwarding CUSTOMER
 ip address 10.0.12.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.2
 tunnel vrf TRANSPORT
!
ip route vrf TRANSPORT 198.51.100.2 255.255.255.255 192.0.2.254
ip route vrf CUSTOMER 10.2.2.2 255.255.255.255 10.0.12.2
```

### R2 Configuration

```cisco
hostname R2
!
vrf definition TRANSPORT
 address-family ipv4
 exit-address-family
!
vrf definition CUSTOMER
 address-family ipv4
 exit-address-family
!
interface Ethernet0/0
 vrf forwarding TRANSPORT
 ip address 198.51.100.2 255.255.255.0
 no shutdown
!
interface Loopback1
 vrf forwarding CUSTOMER
 ip address 10.2.2.2 255.255.255.255
!
interface Tunnel0
 vrf forwarding CUSTOMER
 ip address 10.0.12.2 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 192.0.2.1
 tunnel vrf TRANSPORT
!
ip route vrf TRANSPORT 192.0.2.1 255.255.255.255 198.51.100.254
ip route vrf CUSTOMER 10.1.1.1 255.255.255.255 10.0.12.1
```

### Verify Each VRF Separately

Check that the tunnel transport endpoint is resolved inside `TRANSPORT`:

```cisco
R1# show ip route vrf TRANSPORT 198.51.100.2
R1# show ip cef vrf TRANSPORT 198.51.100.2 detail
R1# ping vrf TRANSPORT 198.51.100.2 source 192.0.2.1
```

Expected next-hop resolution:

```text
VRF TRANSPORT:
198.51.100.2/32 -> 192.0.2.254 -> Ethernet0/0
```

Check overlay forwarding separately:

```cisco
R1# show ip route vrf CUSTOMER
R1# show ip cef vrf CUSTOMER 10.2.2.2 detail
R1# ping vrf CUSTOMER 10.0.12.2 source 10.0.12.1
R1# ping vrf CUSTOMER 10.2.2.2 source 10.1.1.1
```

Expected overlay lookup:

```text
VRF CUSTOMER:
10.2.2.2/32 -> 10.0.12.2 -> Tunnel0
```

The same router uses **two VRF forwarding tables** to deliver one original customer packet.

---

## Packet Forwarding Across Both VRFs

Consider a packet sent from R1's `Loopback1` (`10.1.1.1`) to R2's `Loopback1` (`10.2.2.2`) in Scenario 2.

### Step 1: Overlay Route Lookup

R1 performs a lookup in `CUSTOMER` for the **inner** destination:

```text
Inner source:       10.1.1.1
Inner destination:  10.2.2.2
Lookup table:       VRF CUSTOMER
Result:             Tunnel0 via 10.0.12.2
```

### Step 2: GRE Encapsulation

R1 encapsulates the packet in GRE and an outer IPv4 header:

```text
+-------------------------------------------------+
| Outer IPv4 header                               |
| Source:      192.0.2.1                           |
| Destination: 198.51.100.2                        |
| Protocol:    47 (GRE)                           |
+-------------------------------------------------+
| GRE header                                      |
| Protocol Type: 0x0800 (IPv4)                     |
+-------------------------------------------------+
| Inner IPv4 header                               |
| Source:      10.1.1.1                            |
| Destination: 10.2.2.2                           |
+-------------------------------------------------+
| Original payload                                |
+-------------------------------------------------+
```

The outer IP header does **not** contain the VRF name.

### Step 3: Transport Route Lookup

R1 looks up the **outer** destination in `TRANSPORT`:

```text
Outer destination: 198.51.100.2
Lookup table:      VRF TRANSPORT
Next hop:          192.0.2.254
Exit interface:    Ethernet0/0
```

The ISP forwards the outer IPv4 packet without needing to know about `CUSTOMER`.

### Step 4: GRE Decapsulation on R2

R2 receives the outer packet addressed to its GRE endpoint, matches it to `Tunnel0`, and removes the outer IPv4 and GRE headers.

The original packet is then processed in the VRF assigned to `Tunnel0`:

```text
Decapsulated packet destination: 10.2.2.2
Tunnel0 VRF:                   CUSTOMER
R2 lookup table:              VRF CUSTOMER
Result:                       Loopback1
```

The return packet follows the reverse process using R2's `CUSTOMER` and `TRANSPORT` tables.

**The transition between routing contexts happens because of GRE encapsulation and decapsulation, not because the routes are leaked between VRFs.**

---

## Scenario 3: Both Overlay and Underlay in the Same VRF

It is also possible to place the physical transport interface and the tunnel interface in the same VRF.

For example:

```cisco
interface Ethernet0/0
 vrf forwarding CUSTOMER
 ip address 192.0.2.1 255.255.255.0
!
interface Tunnel0
 vrf forwarding CUSTOMER
 ip address 10.0.12.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.2
 tunnel vrf CUSTOMER
```

R1 must then have **both** routes in `CUSTOMER`:

```cisco
ip route vrf CUSTOMER 198.51.100.2 255.255.255.255 192.0.2.254
ip route vrf CUSTOMER 10.2.2.2 255.255.255.255 10.0.12.2
```

The same arrangement would be needed on R2, with reversed addresses.

The route to the **outer destination** must resolve through the physical network, **not** through `Tunnel0`.

> Using the same VRF for the underlay and overlay is valid, but it increases the importance of controlling tunnel-endpoint routing. A route to the tunnel destination that points into the tunnel causes recursive routing.

---

## Scenario 4: Transport VRF with a Global Overlay

The inverse arrangement is also possible:

- `Tunnel0` is in the global routing table.
- The GRE transport uses `TRANSPORT`.

```cisco
interface Ethernet0/0
 vrf forwarding TRANSPORT
 ip address 192.0.2.1 255.255.255.0
!
interface Tunnel0
 ip address 10.0.12.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.2
 tunnel vrf TRANSPORT
!
ip route vrf TRANSPORT 198.51.100.2 255.255.255.255 192.0.2.254
ip route 10.2.2.2 255.255.255.255 10.0.12.2
```

In this case, the customer-facing routes and the tunnel subnet are global, but the tunnel destination is resolved through `TRANSPORT`.

The remote endpoint must have a compatible GRE configuration and return routing.

---

## VRF-Aware Routing Protocols over GRE

Static routes are not required in the overlay. A GRE tunnel in `CUSTOMER` can also carry routing-protocol traffic for that VRF, subject to platform support.

For example, OSPFv2 can form an adjacency across the tunnel and exchange customer routes.

Use the **Scenario 2** tunnel configurations, but remove the two overlay customer static routes:

```cisco
R1(config)# no ip route vrf CUSTOMER 10.2.2.2 255.255.255.255 10.0.12.2
R2(config)# no ip route vrf CUSTOMER 10.1.1.1 255.255.255.255 10.0.12.1
```

Keep the transport static routes. OSPF will advertise the customer prefixes, **not** provide underlay reachability to the GRE tunnel destination.

### R1 OSPF Configuration

```cisco
interface Tunnel0
 ip ospf network point-to-point
!
router ospf 100 vrf CUSTOMER
 router-id 10.1.1.1
 passive-interface default
 no passive-interface Tunnel0
 network 10.0.12.0 0.0.0.3 area 0
 network 10.1.1.1 0.0.0.0 area 0
```

### R2 OSPF Configuration

```cisco
interface Tunnel0
 ip ospf network point-to-point
!
router ospf 100 vrf CUSTOMER
 router-id 10.2.2.2
 passive-interface default
 no passive-interface Tunnel0
 network 10.0.12.0 0.0.0.3 area 0
 network 10.2.2.2 0.0.0.0 area 0
```

### Verification

```cisco
R1# show ip ospf neighbor
R1# show ip ospf interface Tunnel0
R1# show ip route vrf CUSTOMER ospf
R1# ping vrf CUSTOMER 10.2.2.2 source 10.1.1.1
```

OSPF Hellos are sent through the GRE **overlay**, while the encapsulated GRE packets travel through the `TRANSPORT` underlay.

Other VRF-aware routing protocols, such as EIGRP or BGP, can be configured over GRE where supported. See [Routing over GRE Tunnels](routing.md) and the [VRF-Aware Routing](../../network-infrastructure/routing-concepts/vrf-aware-routing/overview.md) chapters for their general configuration.

---

## Preventing Tunnel Recursive Routing

Recursive routing is one of the most important failures to recognize in GRE labs.

Assume R1 uses:

```cisco
interface Tunnel0
 vrf forwarding CUSTOMER
 tunnel destination 198.51.100.2
 tunnel vrf TRANSPORT
```

**Correct:** The endpoint address is routed through the physical network in `TRANSPORT`.

```text
TRANSPORT:
198.51.100.2/32 -> 192.0.2.254 -> Ethernet0/0

CUSTOMER:
10.2.2.2/32 -> 10.0.12.2 -> Tunnel0
```

**Incorrect:** The transport VRF resolves the GRE destination through the tunnel itself.

```text
TRANSPORT:
198.51.100.2/32 -> Tunnel0  [recursive tunnel resolution]
```

The tunnel cannot use itself to carry the outer packets required to bring itself up.

With separate underlay and overlay VRFs, the isolation helps prevent overlay routing updates from accidentally replacing transport routes. However, it does not eliminate the possibility of incorrect underlay routing.

For loopback-sourced tunnels, install explicit endpoint reachability in the transport table, typically using host routes or appropriate underlay IGP advertisements.

### Which VRF Should Contain Each Route?

| Destination | Required table | Correct forwarding path |
| --- | --- | --- |
| Remote GRE endpoint `198.51.100.2` | `TRANSPORT` | Toward ISP / physical transport |
| Remote customer prefix `10.2.2.2/32` | `CUSTOMER` | Toward `Tunnel0` |
| Tunnel subnet `10.0.12.0/30` | `CUSTOMER` | Directly connected to `Tunnel0` |

---

## Does GRE Automatically Leak Routes Between VRFs?

**No.** A GRE tunnel bridges a forwarding path between its **overlay** and **transport** contexts, but it does not merge their routing tables.

For example:

```text
VRF CUSTOMER:
10.2.2.2/32 via Tunnel0

VRF TRANSPORT:
198.51.100.2/32 via Ethernet0/0
```

The customer prefix is not installed in `TRANSPORT` simply because the GRE outer packet uses that VRF.

The transport prefix is not installed in `CUSTOMER` simply because the tunnel travels over it.

This differs from **route leaking**, which imports or otherwise makes selected routes available between routing tables.

If you need routes shared between VRFs **on the same router**, use the mechanisms explained in [VRF Route Leaking](../../network-infrastructure/routing-concepts/route-leaking/overview.md), not `tunnel vrf` by itself.

---

## Multiple Customer GRE Tunnels over One Transport VRF

One transport VRF can support separate tunnels for multiple customers.

For example:

```text
                     R1

              VRF CUSTOMER_A
                 Tunnel10
                    |
                    +-----------+
                                |
                              VRF WAN ---- ISP
                                |
                    +-----------+
                    |
                 Tunnel20
              VRF CUSTOMER_B
```

The configuration pattern is:

```cisco
interface Tunnel10
 vrf forwarding CUSTOMER_A
 ip address 10.0.10.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.10
 tunnel vrf WAN
!
interface Tunnel20
 vrf forwarding CUSTOMER_B
 ip address 10.0.20.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.20
 tunnel vrf WAN
```

In this example, the tunnels use **different remote endpoint addresses** so that they can be distinguished without relying on GRE keys or platform-specific tunnel matching behavior.

The `WAN` VRF needs routes to both remote tunnel destinations. Each customer VRF contains its own overlay routes.

If both customers use overlapping IP subnets, their routes remain separated by their respective VRFs.

```text
CUSTOMER_A: 192.168.1.0/24 -> Tunnel10
CUSTOMER_B: 192.168.1.0/24 -> Tunnel20
```

GRE does not add an MPLS VPN label or a VRF identifier to the packet. The receiving router associates a packet with the appropriate tunnel according to its supported GRE tunnel matching and forwarding behavior.

---

## IPv6 with VRF-Aware GRE

The same separation applies when GRE carries **IPv6 passenger traffic over an IPv4 underlay**.

For this example, assume Scenario 2 is already configured with `CUSTOMER` and `TRANSPORT`.

Add IPv6 support to the **overlay** VRF:

```cisco
vrf definition CUSTOMER
 address-family ipv6
 exit-address-family
!
ipv6 unicast-routing
```

Configure IPv6 addresses on the existing GRE tunnel interfaces:

**R1:**

```cisco
interface Tunnel0
 ipv6 address 2001:DB8:12::1/64
!
interface Loopback1
 ipv6 address 2001:DB8:1::1/128
!
ipv6 route vrf CUSTOMER 2001:DB8:2::1/128 2001:DB8:12::2
```

**R2:**

```cisco
interface Tunnel0
 ipv6 address 2001:DB8:12::2/64
!
interface Loopback1
 ipv6 address 2001:DB8:2::1/128
!
ipv6 route vrf CUSTOMER 2001:DB8:1::1/128 2001:DB8:12::1
```

The GRE mode remains `gre ip` because the **outer transport is IPv4**. IPv6 is carried inside GRE.

Check both parts:

```cisco
R1# show ipv6 route vrf CUSTOMER
R1# ping vrf CUSTOMER ipv6 2001:DB8:2::1 source 2001:DB8:1::1
R1# show ip route vrf TRANSPORT 198.51.100.2
```

This is distinct from `tunnel mode gre ipv6`, which uses an **IPv6 outer transport** and therefore requires IPv6 underlay reachability.

For additional encapsulation examples, see [GRE Tunnels for IPv6 over IPv4](ipv6-over-ipv4.md).

---

## GRE Keepalives and VRF-Aware Tunnels

Do not assume GRE keepalives work with a VRF-aware tunnel.

Cisco documents **GRE keepalives as unsupported on VRF-aware GRE tunnels** on multiple IOS and IOS XE platforms. Therefore, the configurations on this page do **not** include `keepalive`.

Instead, where supported, failure detection may be provided by routing-protocol neighbor timers, routing-protocol BFD, or another supported tracking mechanism. These are not interchangeable with GRE keepalives, and platform support must be considered.

This restriction is specific to a **VRF-aware GRE feature combination**, not a general statement that all point-to-point GRE tunnels lack keepalive support.

See [GRE Tunnel State and Keepalives](keepalives.md) for basic GRE keepalive operation.

---

## Other Considerations and Platform Restrictions

VRF-aware GRE combines interface VRF membership, GRE encapsulation, and platform forwarding support.

Additional details relevant to CCIE-level labs:

- **Tunnel mode:** `gre ip` is the default for an IPv4 GRE transport. The VRF commands do not change the tunnel encapsulation type.
- **GRE is not encrypted:** VRF separation does not provide encryption, authentication, or confidentiality for packets crossing an untrusted underlay.
- **GRE is IP protocol 47:** The network path must allow GRE packets in the transport VRF.
- **MTU:** Basic GRE over IPv4 adds 24 bytes of overhead; optional GRE fields add more. A VRF does not remove this overhead.
- **Return reachability:** Both endpoints need valid underlay routes to each other's GRE transport addresses and overlay routes for the original traffic.
- **Source/destination matching:** A GRE tunnel requires a compatible peer configuration and routable tunnel-source addresses. Platform limits on duplicate tunnel source/destination combinations may apply.
- **Platform limitations:** Cisco documents different restrictions by platform for VRF-aware GRE, including unsupported keepalives, certain routing protocols, multicast fragmentation, or hardware-scale limits.

For example, Cisco's Catalyst GRE VRF membership documentation lists restrictions on GRE keepalives and IS-IS. Some device families have additional restrictions on OSPFv3, BFD, SVI-sourced tunnels, or the number of supported tunnel sources. Avoid assuming that every routing or tunnel feature supported on a software router is available in a particular hardware forwarding implementation.

---

## Troubleshooting VRF-Aware GRE

A common error is to troubleshoot only the overlay routing table, even though the tunnel is failing because its **transport destination** is unreachable in the specified transport VRF.

Use the following sequence.

### 1. Verify VRF Membership

```cisco
show vrf
show vrf interfaces
show running-config interface Tunnel0
show running-config interface Ethernet0/0
```

Confirm:

```text
Ethernet0/0 -> TRANSPORT
Tunnel0     -> CUSTOMER
Tunnel transport -> TRANSPORT (tunnel vrf)
```

### 2. Verify Underlay Endpoint Reachability

```cisco
show ip route vrf TRANSPORT 198.51.100.2
show ip cef vrf TRANSPORT 198.51.100.2 detail
ping vrf TRANSPORT 198.51.100.2 source 192.0.2.1
```

If the destination is resolved only in the **global** table, it does not satisfy the underlay requirement when the tunnel uses `tunnel vrf TRANSPORT`.

### 3. Verify Tunnel State

```cisco
show interfaces Tunnel0
show tunnel interface
show ip interface Tunnel0
```

Check the following:

```text
Tunnel source
Tunnel destination
Tunnel protocol/transport
Tunnel state
Tunnel interface VRF
Tunnel transport VRF
```

Depending on platform, some details appear in `show interfaces Tunnel0` and others in interface configuration or VRF commands.

A tunnel showing `up/up` is not, by itself, proof that remote overlay packet forwarding succeeds.

### 4. Verify Overlay Routes

```cisco
show ip route vrf CUSTOMER
show ip cef vrf CUSTOMER 10.2.2.2 detail
ping vrf CUSTOMER 10.0.12.2 source 10.0.12.1
ping vrf CUSTOMER 10.2.2.2 source 10.1.1.1
```

### 5. Verify Return Paths and Filtering

Check on R2 as well:

```cisco
R2# show ip route vrf TRANSPORT 192.0.2.1
R2# show ip route vrf CUSTOMER 10.1.1.1
```

Also verify that intermediate ACLs or firewalls allow the **outer GRE packet** and that MTU/fragmentation does not interfere with encapsulated data.

### Common Failure Matrix

| Symptom | Likely cause | Check |
| --- | --- | --- |
| Tunnel destination pings globally, but tunnel fails | `tunnel vrf` selects another table | `show ip route vrf TRANSPORT` |
| Customer route exists, but transport endpoint is unreachable | Missing underlay route | Transport VRF RIB and CEF |
| Transport endpoint reachable, but no customer connectivity | Missing overlay or return route | `show ip route vrf CUSTOMER` on both routers |
| Tunnel source is unexpectedly unreachable | Source interface in wrong VRF, or IP changed | Interface VRF and IP configuration |
| Tunnel has no expected connected route | `vrf forwarding` or tunnel address missing | Tunnel interface and VRF table |
| Tunnel repeatedly disables due to recursion | Transport endpoint resolves through tunnel | Route to tunnel destination in transport VRF |
| OSPF adjacency fails | Tunnel overlay VRF or protocol configuration incorrect | OSPF VRF neighbors, tunnel state, multicast GRE delivery |
| GRE packets fail across ISP | ACL/firewall, source or destination mismatch | Protocol 47 and outer IP addresses |

---

## Key Points

- A VRF-aware GRE tunnel involves **two logically separate lookups**: the inner destination in the **overlay VRF**, and the outer tunnel destination in the **underlay VRF**.
- `vrf forwarding CUSTOMER` on `Tunnel0` selects the VRF for **inner-packet forwarding**.
- `tunnel vrf TRANSPORT` selects the VRF used for **outer-packet destination resolution**.
- Without `vrf forwarding`, the tunnel interface is in the global routing table.
- Without `tunnel vrf`, the GRE destination is resolved using the global routing table.
- The overlay VRF and underlay VRF can be **different**, **the same**, or either can be **global**, depending on the design.
- The physical tunnel-source interface typically belongs to the **underlay VRF**, not the overlay VRF.
- The **remote GRE endpoint route** belongs in the underlay routing table; **customer routes** belong in the overlay routing table.
- A GRE tunnel does **not** automatically leak routes between VRFs.
- VRF names are locally significant and are not transmitted in GRE packets.
- GRE over IPv4 can transport IPv4 and IPv6 overlay traffic, including traffic carried in a named VRF.
- On platforms where VRF-aware GRE keepalives are unsupported, use supported alternatives for failure detection.
- Always verify **both** underlay and overlay reachability when troubleshooting VRF-aware GRE.

---

## Cisco References

- [GRE Tunnel IP Source and Destination VRF Membership (Cisco IOS XE 17)](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr3-fwd/gre/gre-configuration-guide/m-gre-tunnel-ip-source-and-destination-vrf-membership.html)
- [VRF-Aware Tunnels (Cisco IOS XE IP Routing Configuration Guide)](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_ir-vrf-tunnels.html)
- [Implementing Tunnels (Cisco IOS XE Interface and Hardware Component Configuration Guide)](https://www.cisco.com/c/en/us/td/docs/ios/ios_xe/interface/configuration/guide/2_xe/ir_xe_book.pdf)
- [Configuring GRE Tunnel IP Source and Destination VRF Membership (Catalyst 9500, Cisco IOS XE 17.14)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9500/software/release/17-14/configuration_guide/rtng/b_1714_rtng_9500_cg/configuring_generic_routing_encapsulation_gre__tunnel_ip_source_and_destination_vrf_membership.html)
