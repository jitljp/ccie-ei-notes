# DMVPN Phase 2

**DMVPN Phase 2** uses **multipoint GRE (mGRE) tunnels on both the hub and spokes**. Spokes register their tunnel-to-NBMA mappings with the hub, as in Phase 1, but they can also use NHRP to discover other spokes and forward traffic **directly between spokes**.

The defining requirement is that the **overlay route to a remote spoke's network must use that spoke's tunnel address as the next hop**. The source spoke can then resolve the next hop to an NBMA address through NHRP and send GRE packets directly to the destination spoke. The routing protocol still exchanges routes through the hub; **direct data forwarding does not require spoke-to-spoke routing-protocol neighborships**.

This page builds on the Phase 1 lab with **Hub1 and three spokes**, using **EIGRP** as the primary overlay routing protocol. OSPF and BGP are presented as alternatives. IPsec/IKEv2 protection is covered separately in the Phase 3 section.

---

## Phase 2 Architecture

```text
                         DMVPN overlay
                              Hub1
                       Tunnel0: 172.16.0.1
                              (mGRE)
                            /   |   \
                           /    |    \
                          /     |     \
                         /      |      \
                      Spoke1  Spoke2  Spoke3
                       .11     .12     .13
                      (mGRE)  (mGRE)  (mGRE)
                         \______/ 
                       Direct GRE path
                       after NHRP resolution

                 Tunnel subnet: 172.16.0.0/24
```

All four tunnel interfaces are in the same overlay subnet and use mGRE:

| Router | Tunnel address | Tunnel type | NHRP role |
| --- | --- | --- | --- |
| Hub1 | `172.16.0.1/24` | mGRE | NHS |
| Spoke1 | `172.16.0.11/24` | mGRE | NHC |
| Spoke2 | `172.16.0.12/24` | mGRE | NHC |
| Spoke3 | `172.16.0.13/24` | mGRE | NHC |

Neither the hub nor the spokes configure a fixed `tunnel destination`. The **remote GRE destination is chosen using an NHRP mapping** associated with the packet's overlay next hop.

### How Phase 2 Differs from Phase 1

| Property | Phase 1 | Phase 2 |
| --- | --- | --- |
| Hub tunnel | mGRE | mGRE |
| Spoke tunnels | Point-to-point GRE | **mGRE** |
| Spoke `tunnel destination` | Required | **Not configured** |
| Spoke NHRP registration | Yes | Yes |
| Dynamic NHRP resolution for remote spokes | No | **Yes** |
| Spoke-to-spoke user traffic | Through hub | **Direct after resolution** |
| Required routing next hop for direct forwarding | Hub | **Destination spoke** |
| EIGRP split horizon on hub | Disable | Disable |
| EIGRP next-hop-self on hub | Retain | **Disable** |
| Hub route summarization toward spokes | Compatible | Generally defeats direct forwarding for summarized prefixes |
| NHRP redirect/shortcut | No | **No**; these are Phase 3 features |

The change to mGRE makes direct forwarding *possible*, but **routing next-hop selection determines whether it actually happens**. Simply changing a Phase 1 spoke from point-to-point GRE to mGRE is not sufficient.

### Underlay Versus Overlay

The physical connections are unchanged:

```text
             Hub1                       Hub2
               \                         /
                \                       /
                 +-------- ISP --------+
                          / | \
                         /  |  \
                    Spoke1 Spoke2 Spoke3
```

**Hub2 is not used in this Phase 2 configuration.** The ISP routes between the WAN addresses but does not participate in NHRP or EIGRP and has no routes to the site LAN prefixes.

The underlay must allow **spoke-to-spoke reachability between NBMA addresses**, not just connectivity from each spoke to the hub. This becomes important for GRE filtering, NAT, and firewalls.

---

## Addressing Plan

The same addressing is retained from Phase 1:

| Router | WAN address | Tunnel0 address | Loopback0 (simulated LAN) |
| --- | --- | --- | --- |
| Hub1 | `198.51.100.1/30` | `172.16.0.1/24` | `10.1.1.1/24` |
| Spoke1 | `198.51.100.9/30` | `172.16.0.11/24` | `10.11.11.11/24` |
| Spoke2 | `198.51.100.13/30` | `172.16.0.12/24` | `10.12.12.12/24` |
| Spoke3 | `198.51.100.17/30` | `172.16.0.13/24` | `10.13.13.13/24` |

Each WAN /30 connects to the ISP, and each DMVPN router uses a default route to its local ISP next hop.

The overlay parameters remain:

- **NHRP network ID:** `100`
- **GRE tunnel key:** `100`
- **NHRP authentication string:** `NHRP100`
- **EIGRP AS:** `100`

The NHRP network ID selects the local NHRP context and does not itself have to match across routers as an on-wire domain identifier. The **GRE key**, if configured, is carried in GRE packets and must be consistent across this cloud. It is not an encryption key.

---

## Complete Phase 2 Configuration with EIGRP

The following configurations create a working Phase 2 overlay with mGRE on every DMVPN router and EIGRP AS 100. They include the underlay and simulated LAN interfaces so that each router's configuration can be applied independently.

### ISP

```cisco
hostname ISP
!
interface Ethernet0/0
 ip address 198.51.100.2 255.255.255.252
 no shutdown
!
interface Ethernet0/1
 ip address 198.51.100.6 255.255.255.252
 no shutdown
!
interface Ethernet0/2
 ip address 198.51.100.10 255.255.255.252
 no shutdown
!
interface Ethernet0/3
 ip address 198.51.100.14 255.255.255.252
 no shutdown
!
interface Ethernet1/0
 ip address 198.51.100.18 255.255.255.252
 no shutdown
```

All WAN subnets are directly connected to the ISP. The link to Hub2 is present but not used by this overlay.

### Hub1

```cisco
hostname Hub1
!
interface Loopback0
 ip address 10.1.1.1 255.255.255.0
!
interface Ethernet0/0
 ip address 198.51.100.1 255.255.255.252
 no shutdown
!
interface Tunnel0
 ip address 172.16.0.1 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp map multicast dynamic
 ip nhrp network-id 100
 no ip split-horizon eigrp 100
 no ip next-hop-self eigrp 100
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
ip route 0.0.0.0 0.0.0.0 198.51.100.2
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255
 network 10.1.1.1 0.0.0.0
```

The critical hub commands are:

- `ip nhrp map multicast dynamic` allows the hub's mGRE interface to replicate routing-protocol multicast packets toward dynamically registered spokes. On modern IOS XE releases this behavior is enabled by default; it is shown explicitly here.
- `no ip split-horizon eigrp 100` permits routes learned from one spoke on `Tunnel0` to be advertised to other spokes through the same interface.
- **`no ip next-hop-self eigrp 100` preserves the originating spoke's tunnel address as the next hop** when the hub advertises spoke-learned routes to other spokes. This is the important EIGRP difference from Phase 1.

Hub1 is the NHRP **Next Hop Server (NHS)**. It does not require `ip nhrp nhs` when it is not registering with another NHS.

### Spoke1

```cisco
hostname Spoke1
!
interface Loopback0
 ip address 10.11.11.11 255.255.255.0
!
interface Ethernet0/0
 ip address 198.51.100.9 255.255.255.252
 no shutdown
!
interface Tunnel0
 ip address 172.16.0.11 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp map 172.16.0.1 198.51.100.1
 ip nhrp map multicast 198.51.100.1
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
ip route 0.0.0.0 0.0.0.0 198.51.100.10
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255
 network 10.11.11.11 0.0.0.0
```

### Spoke2

```cisco
hostname Spoke2
!
interface Loopback0
 ip address 10.12.12.12 255.255.255.0
!
interface Ethernet0/0
 ip address 198.51.100.13 255.255.255.252
 no shutdown
!
interface Tunnel0
 ip address 172.16.0.12 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp map 172.16.0.1 198.51.100.1
 ip nhrp map multicast 198.51.100.1
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
ip route 0.0.0.0 0.0.0.0 198.51.100.14
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255
 network 10.12.12.12 0.0.0.0
```

### Spoke3

```cisco
hostname Spoke3
!
interface Loopback0
 ip address 10.13.13.13 255.255.255.0
!
interface Ethernet0/0
 ip address 198.51.100.17 255.255.255.252
 no shutdown
!
interface Tunnel0
 ip address 172.16.0.13 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp map 172.16.0.1 198.51.100.1
 ip nhrp map multicast 198.51.100.1
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
ip route 0.0.0.0 0.0.0.0 198.51.100.18
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255
 network 10.13.13.13 0.0.0.0
```

### Why the Spoke NHRP Configuration Changed

In Phase 1, each spoke had a fixed `tunnel destination 198.51.100.1`. In Phase 2, that command is **removed** and `tunnel mode gre multipoint` is configured instead.

Each spoke must now determine the GRE destination for packets using an NHRP mapping. The three configured NHRP commands have distinct purposes:

| Command | Purpose |
| --- | --- |
| `ip nhrp map 172.16.0.1 198.51.100.1` | Static tunnel-to-NBMA mapping for the hub, needed to reach the NHS before dynamic discovery. |
| `ip nhrp map multicast 198.51.100.1` | Sends EIGRP multicast packets toward the hub over mGRE. |
| `ip nhrp nhs 172.16.0.1` | Registers the spoke with Hub1 as its NHS and uses it for NHRP resolution. |

The three commands can alternatively be combined into a **single NHS command**:

```cisco
interface Tunnel0
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
```

The combined form identifies the hub's tunnel and NBMA addresses, configures the NHS, and enables multicast mapping toward it. Use this form **instead of**, not in addition to, the three separate mapping/NHS commands in the examples.

**Phase 2 spokes do not use `ip nhrp map multicast dynamic`** in this design, although it is enabled by default. They send routing-protocol multicast packets to a statically known hub. The hub uses the dynamic form because it must send multicast traffic to the registered spokes.

With unicast-only overlay routing such as BGP, a spoke does not need an NHRP multicast mapping merely to establish BGP or register with its NHS. The EIGRP configuration requires it because EIGRP uses multicast (`224.0.0.10`).

### Migrating from the Phase 1 Configuration

If reusing the working Phase 1 lab, keep the addressing, underlay, loopbacks, GRE key, NHRP authentication, and EIGRP processes. Apply the following tunnel changes.

**Hub1:**

```cisco
interface Tunnel0
 no ip next-hop-self eigrp 100
```

**Each spoke:**

```cisco
interface Tunnel0
 no tunnel destination 198.51.100.1
 tunnel mode gre multipoint
 ip nhrp map multicast 198.51.100.1
```

The existing spoke `ip nhrp map` and `ip nhrp nhs` commands remain configured. The complete configurations above show the resulting state.

---

## NHRP Registration and Spoke-to-Spoke Resolution

Phase 2 uses **two different NHRP processes**:

1. **Registration:** Each spoke tells the NHS how to reach its own tunnel address.
2. **Resolution:** A spoke asks for the NBMA address of another spoke when it needs to forward traffic to that spoke's tunnel next hop.

Registration alone does not create direct spoke-to-spoke mappings on the other spokes.

### 1. Spokes Register with Hub1

Each spoke sends an NHRP Registration Request to Hub1, advertising its tunnel-to-NBMA mapping. Hub1 stores registrations such as:

| Registered tunnel address | Registered NBMA address |
| --- | --- |
| `172.16.0.11` | `198.51.100.9` |
| `172.16.0.12` | `198.51.100.13` |
| `172.16.0.13` | `198.51.100.17` |

Hub1 returns Registration Replies and maintains the mappings while the spokes refresh their registrations. The **hub knows how to reach all registered spokes**, but the spokes initially know only the hub's NBMA address from their static configuration.

### 2. A Route Points to the Remote Spoke

After EIGRP converges, Spoke1 learns Spoke2's simulated LAN:

```text
Destination: 10.12.12.0/24
Next hop:    172.16.0.12 (Spoke2)
Outgoing:    Tunnel0
```

This next hop is the critical difference from Phase 1. Spoke1 needs a mapping for **`172.16.0.12`**, not merely for Hub1's `172.16.0.1`.

The EIGRP route does **not** supply Spoke2's NBMA address. NHRP resolves the tunnel next hop separately.

### 3. Spoke1 Sends an NHRP Resolution Request

When Spoke1 first sends traffic to Spoke2's LAN and has no mapping for `172.16.0.12`, its unresolved mGRE adjacency triggers NHRP resolution.

```text
Spoke1                 Hub1 (NHS)                Spoke2
172.16.0.11            172.16.0.1               172.16.0.12
    |                       |                        |
    |-- Resolution Req ---->|                        |
    |   Who is              |-- Resolution Req ----->|
    |   172.16.0.12?        |                        |
    |                       |                   Records source
    |                       |                   information
    |<---------------------- Resolution Reply -------|
    |              172.16.0.12 -> 198.51.100.13      |
    |                       |                        |
    |==================== Direct GRE ===============>|
```

The steps are:

1. Spoke1 selects the EIGRP route to `10.12.12.0/24` with next hop `172.16.0.12`.
2. Its NHRP cache has no NBMA mapping for that next hop, so it sends a **Resolution Request** toward its NHS, Hub1.
3. Hub1 uses the registered destination information to **forward the request to Spoke2**.
4. Spoke2 recognizes that the request concerns its own tunnel address, and uses the source information to learn how to reach Spoke1.
5. Spoke2 sends a **Resolution Reply directly to Spoke1**, identifying `172.16.0.12` as reachable at `198.51.100.13`.
6. Spoke1 installs a dynamic NHRP entry for Spoke2. Its mGRE forwarding adjacency can now encapsulate traffic directly to `198.51.100.13`.

**The NHRP Resolution Request traverses the hub, but the successful Resolution Reply is sent directly between spokes.** The hub is involved in discovering the destination, not in normal steady-state spoke-to-spoke data forwarding.

The first packets may be delayed or dropped while the NHRP resolution completes. Avoid assuming that every initial packet must be forwarded through the hub. If the optional IPsec protection is configured, direct spoke-to-spoke IKE/IPsec negotiation can also occur during the resolution process.

### CEF Adjacency and the First Packet

When the selected EIGRP next hop is a remote spoke but its NHRP mapping is missing, the mGRE forwarding adjacency can be in a **glean/unresolved state**. The initial packet triggers NHRP control-plane resolution rather than immediately using an established remote GRE destination. Once the mapping is learned, forwarding can use the resolved mGRE adjacency. This is why a short first-packet delay or loss is possible even though the overlay route already exists.

In Phase 3, by contrast, the initial traffic can be forwarded via the hub and the NHRP redirect/shortcut mechanism subsequently optimizes the path. **Phase 2 has no such redirect-based optimization step.**

### 4. Dynamic NHRP Mappings

After Spoke1 resolves Spoke2, its NHRP cache conceptually contains:

| Tunnel address | NBMA address | Entry type |
| --- | --- | --- |
| `172.16.0.1` | `198.51.100.1` | Static hub mapping |
| `172.16.0.12` | `198.51.100.13` | Dynamic spoke mapping |

The reverse spoke may also learn Spoke1's NBMA information from the resolution exchange. Cached remote mappings age according to their holding times. When a needed mapping expires, NHRP can resolve it again.

**An NHRP mapping is not the same as a LAN route.** The EIGRP route to `10.12.12.0/24` remains in the routing table, while NHRP supplies the NBMA destination for the route's tunnel next hop.

### 5. NHRP Timers

Two useful interface commands are:

```cisco
interface Tunnel0
 ip nhrp holdtime 600
 ip nhrp registration timeout 120
```

- `ip nhrp holdtime 600` advertises a **600-second mapping holding time** to remote NHRP devices. It does not directly force every locally stored remote mapping to expire after 600 seconds.
- `ip nhrp registration timeout 120` configures the spoke's **registration refresh interval** to the NHS.

These are optional tuning examples, not part of the baseline configurations. Changes should preserve adequate registration refresh time before the NHS mapping expires.

### Phase 2 Does Not Use NHRP Redirects or Shortcuts

Do **not** add these Phase 3 commands to a Phase 2 example:

```cisco
! Phase 3 commands, not used here:
ip nhrp redirect
ip nhrp shortcut
```

In Phase 2, routing must already identify the **destination spoke as the next hop**. NHRP then resolves that next hop. Phase 3 adds a mechanism to discover direct forwarding paths even when the routing next hop is the **hub**.

---

## EIGRP Routing Behavior

The EIGRP control-plane topology is still hub-and-spoke:

```text
                EIGRP AS 100
                    Hub1
                   /  |  \
              Spoke1 Spoke2 Spoke3

          EIGRP neighbors: hub <-> each spoke
          Data forwarding: spoke <-> spoke directly
```

Hub1 forms three EIGRP neighborships. Each spoke forms one EIGRP neighborship with Hub1. Spokes do **not** need to become EIGRP neighbors with one another to forward data directly.

### Disable Split Horizon on Hub1

EIGRP split horizon normally prevents advertising a route out the same interface on which it was learned.

Hub1 learns Spoke1's `10.11.11.0/24` through `Tunnel0`. To advertise it to Spoke2 and Spoke3, disable split horizon **on Hub1's tunnel interface**:

```cisco
interface Tunnel0
 no ip split-horizon eigrp 100
```

This requirement is shared with Phase 1 when the hub must advertise spoke-learned routes back out the tunnel.

### Disable Next-Hop-Self on Hub1

By default, EIGRP advertises itself as the next hop for routes that it forwards to another EIGRP neighbor.

If Hub1 retains this behavior, Spoke1 learns Spoke2's route with:

```text
10.12.12.0/24 via 172.16.0.1 (Hub1)
```

That does not provide the destination spoke next hop needed for Phase 2 direct resolution.

Disable next-hop-self **on Hub1's tunnel interface**:

```cisco
interface Tunnel0
 no ip next-hop-self eigrp 100
```

Hub1 can then advertise the spoke-learned route while retaining the originating spoke as the next hop:

```text
10.12.12.0/24 via 172.16.0.12 (Spoke2)
```

The spoke-to-spoke next hop is on the shared tunnel subnet, so Spoke1 can resolve it through NHRP. **Do not confuse the EIGRP neighbor that advertised the route (Hub1) with the route's forwarding next hop (Spoke2).**

### Split Horizon and Next-Hop-Self Solve Different Problems

| Hub command | Problem solved |
| --- | --- |
| `no ip split-horizon eigrp 100` | Lets the hub advertise spoke-learned routes back out the same tunnel interface. |
| `no ip next-hop-self eigrp 100` | Makes those routes use the originating spoke as the next hop. |

Disabling split horizon **alone** allows the routes to be learned but may still leave the hub as the forwarding next hop. Disabling next-hop-self **alone** does not overcome split-horizon suppression of the route advertisements.

Neither command is normally needed on the spokes in this lab, because each spoke has only one EIGRP neighbor on `Tunnel0`.

### EIGRP Summarization on the Hub

**Avoid summarizing spoke LAN routes toward other spokes when direct Phase 2 forwarding is required.**

For example, if Hub1 configures:

```cisco
interface Tunnel0
 ip summary-address eigrp 100 10.0.0.0 255.0.0.0
```

Spoke1 may receive the aggregate route `10.0.0.0/8` from Hub1 instead of the individual routes to remote spoke LANs. That aggregate uses **Hub1 (`172.16.0.1`) as the next hop**.

Without a specific route to `10.12.12.0/24` via Spoke2, Spoke1 does not perform the direct NHRP resolution for `172.16.0.12` when forwarding to that LAN. Traffic instead follows the route through Hub1. This **defeats the Phase 2 direct forwarding design for the summarized prefixes**, although the network can still route packets through the hub.

The same consideration applies when the hub advertises only a **default route** for spoke destinations. A default or summary route is not inherently forbidden by Phase 2; it is unsuitable for destinations that are expected to use direct spoke-to-spoke forwarding unless more-specific routes preserve the spoke next hops.

This contrasts with **Phase 1**, where hub forwarding is already required, and **Phase 3**, where redirects and shortcuts can support direct forwarding despite summary/default routes.

### EIGRP Stub

In larger hub-and-spoke designs, an appropriate **EIGRP stub configuration on spokes** can reduce query scope and help scale the routing control plane. It does not replace the Phase 2 next-hop requirement and should not accidentally prevent spokes from advertising prefixes needed by other sites.

---

## End-to-End Packet Forwarding

Assume Spoke1 sends ICMP traffic sourced from `10.11.11.11` to Spoke2's loopback, `10.12.12.12`.

### Before NHRP Resolution

1. Spoke1 performs a route lookup for `10.12.12.12` and selects EIGRP next hop **`172.16.0.12` (Spoke2)**.
2. The next hop is reachable through `Tunnel0`, but there is **no cached NHRP mapping** for `172.16.0.12`.
3. Spoke1 sends an NHRP Resolution Request via Hub1.
4. Hub1 forwards the request toward Spoke2; Spoke2 replies directly to Spoke1 with its tunnel-to-NBMA mapping.
5. Spoke1 installs the mapping and can forward subsequent traffic over the direct GRE path.

### After NHRP Resolution

1. Spoke1 looks up `10.12.12.12` and selects `172.16.0.12` as the overlay next hop.
2. NHRP resolves `172.16.0.12` to **`198.51.100.13`**.
3. Spoke1 encapsulates the original IP packet in GRE. The **outer source** is `198.51.100.9` and the **outer destination** is `198.51.100.13`.
4. The ISP forwards the outer packet directly to Spoke2's WAN interface using ordinary IP routing.
5. Spoke2 decapsulates the packet and delivers it to `Loopback0`.

```text
Original packet:
  Source:      10.11.11.11
  Destination: 10.12.12.12

GRE outer header after resolution:
  Source:      198.51.100.9   (Spoke1)
  Destination: 198.51.100.13  (Spoke2)

Overlay route:
  10.12.12.0/24 via 172.16.0.12, Tunnel0

Actual path:
  Spoke1 -------- GRE over ISP --------> Spoke2
              (Hub1 not in data path)
```

The **return path** must also be valid. Spoke2 learns Spoke1's `10.11.11.0/24` via `172.16.0.11` and can resolve Spoke1's NBMA address. Successful one-way NHRP resolution does not prove that the return route and return path are correct.

A routing-protocol adjacency **still exists only between each spoke and Hub1**; the direct GRE path is a forwarding relationship created through NHRP, not a newly configured point-to-point tunnel interface or an EIGRP neighbor session.

---

## Alternative Routing Protocol: OSPF

**OSPF broadcast network type** is the usual choice for Phase 2 because it models the tunnel subnet as a shared transit network. That allows OSPF to install remote spoke routes with the **destination spoke as the next hop**.

### Why Point-to-Multipoint Is Not Recommended Here

With **point-to-multipoint**, OSPF models the overlay as separate point-to-point links:

```text
Spoke1 -------- Hub1 -------- Spoke2
```

Spoke1's SPF calculation reaches Spoke2's networks through Hub1, resulting in a route such as:

```text
O 10.12.12.0/24 via 172.16.0.1 (Hub1)
```

With **broadcast**, OSPF models the tunnel addresses as one shared transit network represented by the DR's Type 2 Network LSA:

```text
             Shared OSPF transit network
                  /    |    \
              Hub1   Spoke1 Spoke2
              (DR)
```

Spoke1 can then install:

```text
O 10.12.12.0/24 via 172.16.0.12 (Spoke2)
```

It does **not** need a FULL OSPF adjacency with Spoke2 for that next hop to appear. The hub, acting as DR, provides the necessary adjacency and LSA distribution.

### DR/BDR Election Requirements

Configure `ip ospf network broadcast` on all tunnel interfaces, then ensure that **Hub1 is the DR** and the spokes cannot become the DR or BDR:

- **Hub1:** `ip ospf priority 255`
- **Each spoke:** `ip ospf priority 0`

A priority of `0` makes a spoke ineligible for DR/BDR election. The hub priority is deliberately high, but OSPF DR elections are not preemptive: if the wrong router is already DR, changing priorities alone may not immediately replace it. Reset the relevant OSPF adjacencies or restart the election when needed.

### Replacing EIGRP with OSPF

Remove EIGRP from all DMVPN routers:

```cisco
no router eigrp 100
```

**Hub1:**

```cisco
interface Loopback0
 ip ospf network point-to-point
!
interface Tunnel0
 ip ospf network broadcast
 ip ospf priority 255
!
router ospf 100
 router-id 10.1.1.1
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255 area 0
 network 10.1.1.1 0.0.0.0 area 0
```

**Spoke1:**

```cisco
interface Loopback0
 ip ospf network point-to-point
!
interface Tunnel0
 ip ospf network broadcast
 ip ospf priority 0
!
router ospf 100
 router-id 10.11.11.11
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255 area 0
 network 10.11.11.11 0.0.0.0 area 0
```

**Spoke2:**

```cisco
interface Loopback0
 ip ospf network point-to-point
!
interface Tunnel0
 ip ospf network broadcast
 ip ospf priority 0
!
router ospf 100
 router-id 10.12.12.12
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255 area 0
 network 10.12.12.12 0.0.0.0 area 0
```

**Spoke3:**

```cisco
interface Loopback0
 ip ospf network point-to-point
!
interface Tunnel0
 ip ospf network broadcast
 ip ospf priority 0
!
router ospf 100
 router-id 10.13.13.13
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255 area 0
 network 10.13.13.13 0.0.0.0 area 0
```

All tunnel interfaces belong to OSPF **area 0**. Hub1 should be DR with **three FULL spoke adjacencies**. Each spoke should have a FULL adjacency with Hub1. Other spokes are not required to become FULL neighbors with one another.

The `ip ospf network point-to-point` command on **Loopback0** serves a separate purpose: it causes OSPF to advertise the actual **/24 prefix** instead of the default loopback /32. It does not change the `Tunnel0` network type.

### OSPF Verification

```cisco
show ip ospf interface Tunnel0
show ip ospf neighbor
show ip ospf database network
show ip route ospf
show ip route 10.12.12.12
```

On Spoke1, confirm that the route to Spoke2's `10.12.12.0/24` uses `172.16.0.12`, **not Hub1**. Verify that Hub1 is DR and the spokes have priority 0.

A mismatched OSPF network type, incorrect DR election, or a tunnel multicast-mapping problem can prevent the required LSA distribution or produce unexpected next hops.

---

## Alternative Routing Protocol: BGP

BGP can also support Phase 2. The important requirement is unchanged: **the BGP NEXT_HOP installed for a remote spoke's prefix must identify the originating spoke**, allowing NHRP to resolve that spoke directly.

BGP uses **unicast TCP sessions**, so the routing protocol itself does not require mGRE multicast replication. The existing NHRP NHS configuration still provides hub reachability and spoke discovery.

### Option 1: eBGP with Third-Party Next-Hop Behavior

As in the Phase 1 example, use Hub1 AS `65000` and separate AS numbers for each spoke:

| Router | BGP AS | BGP neighbor(s) |
| --- | --- | --- |
| Hub1 | `65000` | Spoke1, Spoke2, Spoke3 |
| Spoke1 | `65101` | Hub1 |
| Spoke2 | `65102` | Hub1 |
| Spoke3 | `65103` | Hub1 |

Ordinarily, eBGP changes the NEXT_HOP to itself when advertising a route. However, when the **existing NEXT_HOP and receiving eBGP peer are on the same subnet**, Cisco BGP can preserve the original next hop. This is **third-party next-hop behavior**, and it is useful in the shared `172.16.0.0/24` DMVPN tunnel subnet.

**Hub1:**

```cisco
router bgp 65000
 bgp router-id 10.1.1.1
 no bgp default ipv4-unicast
 neighbor 172.16.0.11 remote-as 65101
 neighbor 172.16.0.12 remote-as 65102
 neighbor 172.16.0.13 remote-as 65103
 address-family ipv4 unicast
  network 10.1.1.0 mask 255.255.255.0
  neighbor 172.16.0.11 activate
  neighbor 172.16.0.12 activate
  neighbor 172.16.0.13 activate
 exit-address-family
```

**Spoke1:**

```cisco
router bgp 65101
 bgp router-id 10.11.11.11
 no bgp default ipv4-unicast
 neighbor 172.16.0.1 remote-as 65000
 address-family ipv4 unicast
  network 10.11.11.0 mask 255.255.255.0
  neighbor 172.16.0.1 activate
 exit-address-family
```

**Spoke2:**

```cisco
router bgp 65102
 bgp router-id 10.12.12.12
 no bgp default ipv4-unicast
 neighbor 172.16.0.1 remote-as 65000
 address-family ipv4 unicast
  network 10.12.12.0 mask 255.255.255.0
  neighbor 172.16.0.1 activate
 exit-address-family
```

**Spoke3:**

```cisco
router bgp 65103
 bgp router-id 10.13.13.13
 no bgp default ipv4-unicast
 neighbor 172.16.0.1 remote-as 65000
 address-family ipv4 unicast
  network 10.13.13.0 mask 255.255.255.0
  neighbor 172.16.0.1 activate
 exit-address-family
```

**Do not configure `neighbor ... next-hop-self` on Hub1 for this Phase 2 example.** Such a command would make Hub1 the NEXT_HOP for spoke routes and prevent their intended direct NHRP resolution.

The example relies on third-party next-hop behavior. **Verify the actual NEXT_HOP** in `show bgp ipv4 unicast` and the installed routing table, rather than assuming that all eBGP routes preserve it. The common tunnel subnet is essential to this behavior.

For Spoke2's `10.12.12.0/24` prefix, Spoke1 should see **`172.16.0.12`** as its next hop. BGP sessions remain hub-and-spoke, while data forwarding can be spoke-to-spoke.

### Option 2: iBGP with a Route Reflector

All routers can instead use AS `65000`, with Hub1 as a **route reflector** and the spokes as RR clients.

By default, an RR **preserves the original NEXT_HOP of reflected iBGP routes**. That is useful in Phase 2, unlike the Phase 1 iBGP design where changing the next hop to Hub1 can be desirable.

**Hub1:**

```cisco
router bgp 65000
 bgp router-id 10.1.1.1
 no bgp default ipv4-unicast
 neighbor 172.16.0.11 remote-as 65000
 neighbor 172.16.0.12 remote-as 65000
 neighbor 172.16.0.13 remote-as 65000
 address-family ipv4 unicast
  network 10.1.1.0 mask 255.255.255.0
  neighbor 172.16.0.11 activate
  neighbor 172.16.0.11 route-reflector-client
  neighbor 172.16.0.12 activate
  neighbor 172.16.0.12 route-reflector-client
  neighbor 172.16.0.13 activate
  neighbor 172.16.0.13 route-reflector-client
 exit-address-family
```

**Spoke1:**

```cisco
router bgp 65000
 bgp router-id 10.11.11.11
 no bgp default ipv4-unicast
 neighbor 172.16.0.1 remote-as 65000
 address-family ipv4 unicast
  network 10.11.11.0 mask 255.255.255.0
  neighbor 172.16.0.1 activate
 exit-address-family
```

**Spoke2:**

```cisco
router bgp 65000
 bgp router-id 10.12.12.12
 no bgp default ipv4-unicast
 neighbor 172.16.0.1 remote-as 65000
 address-family ipv4 unicast
  network 10.12.12.0 mask 255.255.255.0
  neighbor 172.16.0.1 activate
 exit-address-family
```

**Spoke3:**

```cisco
router bgp 65000
 bgp router-id 10.13.13.13
 no bgp default ipv4-unicast
 neighbor 172.16.0.1 remote-as 65000
 address-family ipv4 unicast
  network 10.13.13.0 mask 255.255.255.0
  neighbor 172.16.0.1 activate
 exit-address-family
```

**Do not configure `neighbor ... next-hop-self all` on the RR in this Phase 2 design.** That option rewrites the next hop even on reflected routes, making the hub the forwarding next hop. For the intended Phase 2 behavior, retain the originating spoke's NEXT_HOP.

The `network` statements require exact matching `/24` routes in the local RIB, which the configured loopbacks supply. No `ebgp-multihop` is needed in the eBGP example because peers use directly connected tunnel addresses; the iBGP example likewise peers over the tunnel subnet.

---

## Verification

The order of verification matters: **underlay, tunnel, NHRP registration, routing next hop, NHRP resolution, direct forwarding**. An EIGRP adjacency alone does not prove that Phase 2 spoke-to-spoke forwarding works.

### 1. Verify Underlay Reachability

From Spoke1:

```cisco
ping 198.51.100.1 source Ethernet0/0
ping 198.51.100.13 source Ethernet0/0
ping 198.51.100.17 source Ethernet0/0
show ip route 198.51.100.13
```

Confirm that the **other spokes' WAN addresses**, not only Hub1's WAN address, are reachable via the ISP. The route toward a remote NBMA address must **not resolve recursively into `Tunnel0`**.

### 2. Verify That Every DMVPN Router Uses mGRE

```cisco
show ip interface brief
show running-config interface Tunnel0
show interfaces Tunnel0
```

Every DMVPN router must have:

```cisco
tunnel mode gre multipoint
```

No Phase 2 spoke should retain the Phase 1 `tunnel destination` statement. Verify the tunnel source, key, tunnel IP address, and NHRP configuration.

### 3. Verify Spoke Registration with the Hub

On Hub1:

```cisco
show ip nhrp
show dmvpn
show ip nhrp traffic
```

Look for **three registered spokes**, with these mappings:

```text
172.16.0.11 -> 198.51.100.9
172.16.0.12 -> 198.51.100.13
172.16.0.13 -> 198.51.100.17
```

On a spoke:

```cisco
show ip nhrp
show ip nhrp nhs detail
show dmvpn
```

Confirm the statically configured hub mapping and successful NHS registration. Registration does not prove that spoke-to-spoke resolution has occurred.

### 4. Verify EIGRP Neighbors and Next Hops

```cisco
show ip eigrp neighbors
show ip route eigrp
show ip route 10.12.12.12
show ip protocols
```

Expected EIGRP neighbor counts:

| Router | EIGRP neighbors |
| --- | ---: |
| Hub1 | 3 |
| Each spoke | 1 (Hub1) |

**On Spoke1, the route to `10.12.12.0/24` must point to `172.16.0.12`**. If it points to `172.16.0.1`, investigate Hub1's `no ip next-hop-self eigrp 100` setting or summarization.

On Hub1, make sure both `no ip split-horizon eigrp 100` and `no ip next-hop-self eigrp 100` are configured under `Tunnel0`.

For OSPF, use `show ip ospf interface Tunnel0`, `show ip ospf neighbor`, and `show ip route ospf`. For BGP, use `show bgp ipv4 unicast`, `show bgp ipv4 unicast summary`, and `show ip route bgp`. **Check the installed forwarding next hop**, not merely the protocol neighbor state.

### 5. Generate Spoke-to-Spoke Traffic and Inspect NHRP

On Spoke1:

```cisco
ping 10.12.12.12 source 10.11.11.11
show ip nhrp
show dmvpn
show ip nhrp traffic
show ip cef 10.12.12.12 detail
```

Look for a **dynamic** NHRP entry mapping:

```text
172.16.0.12 -> 198.51.100.13
```

On Spoke2, inspect its NHRP cache and the return route to `10.11.11.0/24`. Depending on timing and the platform, the first ping may time out while NHRP resolution is in progress. Repeat the ping after the mapping is established.

The NHRP cache shows **tunnel-to-NBMA mappings**. `show dmvpn` offers a useful view of the active peer mappings; exact peer-state and flag output varies by platform.

### 6. Confirm That the Hub Is Not in the Data Path

From Spoke1:

```cisco
traceroute 10.12.12.12 source 10.11.11.11
ping 10.12.12.12 source 10.11.11.11
```

After resolution, the user-traffic GRE packet's outer destination should be **`198.51.100.13`**, not Hub1's `198.51.100.1`.

A packet capture on Spoke1's WAN interface or the ISP links can confirm that direct encapsulation. A traceroute across the **overlay** should not show Hub1 as an intermediate routed hop for established direct forwarding, although hop display depends on ICMP/TTL behavior.

On Hub1, the presence of NHRP registration and EIGRP traffic is normal. **A hub participating in the control plane is not evidence that it forwards the spoke-to-spoke user packets.**

### 7. Examine NHRP Counters and Debugs

Useful targeted commands include:

```cisco
show ip nhrp traffic
show ip nhrp detail
debug nhrp
debug nhrp detail
```

When troubleshooting a resolution exchange, correlate:

1. The Resolution Request leaving the source spoke.
2. The request arriving at and being forwarded by Hub1.
3. The request reaching the destination spoke.
4. The Resolution Reply leaving the destination and reaching the source spoke.
5. The resulting dynamic NHRP cache entry and forwarding adjacency.

The NHRP request ID can help correlate related debug messages at different routers. If a Request reaches Spoke2 but the Reply does not return to Spoke1, investigate **direct spoke-to-spoke underlay reachability and filtering**, rather than assuming that the hub failed to forward the request.

---

## Troubleshooting Phase 2

| Symptom | Likely cause | Verification or correction |
| --- | --- | --- |
| Spoke cannot register with NHS | Incorrect static hub mapping, authentication, GRE key, tunnel source, WAN reachability | Check `show ip nhrp nhs detail`, hub registrations, tunnel config, and underlay reachability. |
| Spokes have no EIGRP neighbors with hub | Missing spoke multicast map, hub multicast replication, passive tunnel, AS mismatch | Check multicast mappings, `show ip eigrp neighbors`, and tunnel settings. |
| Hub learns spoke routes but other spokes do not | EIGRP split horizon still enabled on hub | Configure `no ip split-horizon eigrp 100` on Hub1 `Tunnel0`. |
| Spokes learn remote LAN routes via `172.16.0.1` | EIGRP next-hop-self still enabled, or hub summary/default route | Configure `no ip next-hop-self eigrp 100`; check route specificity and summarization. |
| Routes use correct remote-spoke next hops but no NHRP mapping appears | Resolution Request not sent/forwarded, failed NHRP exchange, invalid underlay path | Check NHRP counters and debugs on source, hub, and destination. |
| NHRP Request reaches destination but Reply does not return | Direct WAN reachability or GRE/IPsec filtering problem, incorrect source NBMA information | Verify spoke-to-spoke NBMA reachability and captures of the return path. |
| NHRP mapping exists but direct traffic fails | GRE blocked between spokes, asymmetric return path, incorrect key or encapsulation, forwarding adjacency issue | Check both WAN directions, reverse route, `show ip cef`, and mGRE settings. |
| First ping fails but subsequent pings work | Time needed for initial NHRP resolution or optional IPsec negotiation | Inspect NHRP entries/counters before and after traffic. |
| OSPF uses hub as next hop | Point-to-multipoint OSPF topology, incorrect broadcast/DR configuration | Set broadcast network type, hub DR priority, spoke priority 0; inspect Type 2 LSAs. |
| OSPF neighbor stuck in EXSTART/EXCHANGE or flaps | MTU, timers, authentication, multicast mapping, network-type mismatch | Check `show ip ospf neighbor detail`, interface parameters, and tunnel MTU. |
| BGP route has hub NEXT_HOP | eBGP next-hop-self configured or third-party-next-hop conditions not met; iBGP RR next-hop rewritten | Inspect BGP attributes and remove unintended next-hop rewriting. |
| Small packets work but large packets fail | GRE/IPsec overhead, fragmentation, MTU, PMTUD, TCP MSS | Check tunnel MTU and path MTU behavior; adjust as required. |
| Tunnel reports `up/up` but Phase 2 forwarding fails | GRE interface status does not prove NHRP resolution or correct routing | Validate every layer from the underlay through the route next hop and NHRP cache. |

### NHRP Resolution Failure Domains

A Phase 2 resolution problem can be isolated to three separate paths:

- **Source to hub:** The request must be deliverable to the NHS using the spoke's static hub mapping.
- **Hub to destination:** The NHS must have a valid registered mapping for the target spoke and forward the request.
- **Destination to source:** The destination must be able to send the reply directly to the requester; the same underlay must then deliver direct GRE user traffic.

This separation explains why registration can succeed and hub-to-spoke routing can look healthy while direct spoke-to-spoke communication fails.

### NHRP State and Forwarding State Are Separate

Confirm all of the following:

1. An **overlay route** exists for the remote LAN.
2. The route's **next hop is the destination spoke tunnel IP**, not the hub.
3. A valid **NHRP mapping** exists for that next hop.
4. The **underlay** can reach the mapped NBMA address without recursive dependence on the overlay.
5. GRE (and IPsec if enabled) can reach that remote endpoint in **both directions**.
6. The destination spoke has a valid **return route**.

A dynamic NHRP mapping by itself does not prove that the correct LAN route is installed, and a correct LAN route does not prove that the NHRP mapping has been resolved.

### Security, NAT, and MTU Considerations

GRE and NHRP do **not** encrypt payload traffic. When IPsec is enabled, direct spoke-to-spoke communication requires compatible IKE/IPsec configuration and permitted transport traffic between the spokes; details are covered in the Phase 3 IPsec/IKEv2 material.

An underlay that permits traffic **only between the hub and spokes** may support Phase 1 but prevent Phase 2 direct forwarding. With unprotected GRE, ensure IP protocol **47** is permitted between spoke NBMA addresses. With IPsec, account for the required IKE/IPsec transport (including NAT traversal where applicable).

GRE and optional IPsec encapsulation reduce the effective payload MTU. Troubleshooting should distinguish **NHRP resolution**, **routing**, and **large-packet transport** rather than treating all reachability failures as an NHRP problem.

---

## Phase 2 Limitations and Transition to Phase 3

Phase 2 removes the hub from the normal spoke-to-spoke data path, but it introduces constraints:

- **Next-hop dependency:** Direct spoke-to-spoke forwarding requires routes that identify the **destination spoke** as the next hop.
- **Limited summarization:** Hub-originated summary/default routes for remote spoke prefixes usually point at the hub and defeat direct forwarding for those prefixes.
- **Routing scale:** Spokes often need specific routes to many remote networks, increasing the routing-table and control-plane footprint.
- **Control-plane dependency:** The hub is still used for registration and initial NHRP resolution, even when data traffic bypasses it.
- **Direct transport requirement:** The underlay must support direct GRE or protected tunnel traffic between spoke NBMA addresses.

**Phase 3** adds **NHRP Redirect** on the hub and **NHRP Shortcut** on the spokes. This permits the hub to remain the routing next hop and advertise summary/default routes while NHRP discovers more direct forwarding paths for traffic that initially traverses the hub.

For the next stage, see [DMVPN Phase 3](phase3/overview.md). For the original hub-routed design, see [DMVPN Phase 1](phase1.md).

---

## Key Points

- **Phase 2 uses mGRE on both the hub and spokes.** Spokes do not configure a fixed `tunnel destination`.
- Each spoke **registers its tunnel-to-NBMA mapping** with the hub NHS.
- **Direct forwarding requires the route's next hop to be the destination spoke's tunnel IP address.**
- A source spoke sends an **NHRP Resolution Request through the hub**, and the destination spoke sends a **Resolution Reply directly to the source**.
- The resulting dynamic NHRP mapping lets the source encapsulate GRE packets directly to the destination spoke's NBMA address.
- **Spoke-to-spoke GRE forwarding does not require spoke-to-spoke EIGRP, OSPF, or BGP neighborships.**
- For EIGRP, configure **`no ip split-horizon eigrp 100`** on the hub to advertise spoke-learned routes back out its tunnel.
- For EIGRP, also configure **`no ip next-hop-self eigrp 100`** on the hub to retain the originating spoke as the next hop.
- The hub uses `ip nhrp map multicast dynamic`; spokes use **static multicast mapping to the hub** when multicast routing protocols are used.
- On Phase 2 spokes, `ip nhrp nhs <hub-tunnel> nbma <hub-nbma> multicast` can combine the static hub map, NHS, and multicast configuration.
- **Avoid hub summarization/default-only routing for remote spoke prefixes** if those prefixes must use direct Phase 2 forwarding.
- **OSPF broadcast network type**, with the **hub as DR** and **spoke priorities of 0**, supports direct spoke next-hop calculations.
- For eBGP, **third-party next-hop behavior** can preserve spoke tunnel NEXT_HOP values when peers share a subnet; do not override this unintentionally with `next-hop-self`.
- For iBGP route reflection, the RR **normally preserves reflected route NEXT_HOP**; avoid `next-hop-self all` in the Phase 2 direct-forwarding design.
- Phase 2 does **not** use `ip nhrp redirect` or `ip nhrp shortcut`; these belong to Phase 3.
- **Validate the data path**, not only neighbor adjacencies and NHRP registrations: confirm the remote-spoke route next hop, dynamic NHRP entry, and direct outer GRE destination.

---

## References

- [Cisco: Dynamic Multipoint IPsec VPNs (Using Multipoint GRE/NHRP to Scale IPsec VPNs)](https://www.cisco.com/c/en/us/support/docs/security-vpn/ipsec-negotiation-ike-protocols/41940-dmvpn.html)
- [Cisco: Troubleshoot DMVPN Issues](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/111976-dmvpn-troubleshoot-00.html)
- [Cisco: Troubleshoot DMVPN Phase 2 Spoke-to-Spoke Tunnel](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/221665-troubleshoot-dmvpn-phase-2-spoke-to-spok.html)
- [Cisco: Dynamic Multipoint VPN Configuration Guide](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-conn-dmvpn-dmvpn-0.html)
- [Cisco: DMVPN Configuration Using FQDN (NHRP NHS/NBMA Syntax)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-conn-dmvpn-conf-using-fqdn.html)
- [Cisco: BGP Next Hop Unchanged (Third-Party Next-Hop Behavior)](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-next-hop-0.html)
