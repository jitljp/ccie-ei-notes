# L3VPN Overview

An MPLS Layer 3 VPN allows a service provider to carry customer routes across an MPLS core.

From the customer's point of view, the provider network acts like a private routed WAN.

The customer sites exchange IP routes with the provider edge routers, and the provider carries traffic between the sites.

```text
CE1 --- PE1 === MPLS Core === PE2 --- CE2
```

The customer routers do not need to run MPLS. Only the provider routers run MPLS.

---

## Basic Topology

Use this topology for the L3VPN examples:

```text
Customer A Site 1                    Customer A Site 2

10.10.10.0/24                        11.11.11.0/24
      |                                      |
     CE1                                    CE2
      |                                      |
     PE1 -------- P1 --------- P2 --------- PE2
    1.1.1.1     2.2.2.2      3.3.3.3      4.4.4.4
```

The provider core is:

```text
PE1 --- P1 --- P2 --- PE2
```

The customer edge routers are:

```text
CE1 and CE2
```

The provider edge routers are:

```text
PE1 and PE2
```

The provider core routers are:

```text
P1 and P2
```

The CE routers exchange customer routes with the PE routers.

The PE routers exchange VPN routes with each other using MP-BGP.

The P routers only switch labeled packets. They don't need to know the customer routes.

---

## Why L3VPNs Are Needed

Without L3VPNs, the provider would need to carry customer routes in the global routing table.

That does not scale well.

It also creates a problem when customers use overlapping addresses.

```text
Customer A uses 10.10.10.0/24.
Customer B also uses 10.10.10.0/24.
```

The provider cannot place both routes into the same global routing table as normal IPv4 routes.

MPLS L3VPN solves this by separating customer routing tables with VRFs.

---

## VRFs on the PE Routers

A **VRF** is a separate routing table on a PE router.

Each customer, or each customer VPN, can use its own VRF.

Example:

```text
PE1
├── Global routing table
├── VRF CUSTOMER_A
└── VRF CUSTOMER_B
```

Customer routes are placed into the customer's VRF, not into the provider's global routing table.

This allows different customers to use overlapping IP addresses.

```text
CUSTOMER_A VRF:
10.10.10.0/24

CUSTOMER_B VRF:
10.10.10.0/24
```

These routes can coexist because they are in separate VRFs.

---

## Provider Core Routing

The provider core does not need customer routes.

The P routers only need reachability to provider infrastructure addresses, especially the PE loopbacks.
These are the BGP next hops.

For example, P1 and P2 need routes to:

```text
PE1 Loopback0
PE2 Loopback0
```

They do not need routes to:

```text
10.10.10.0/24
11.11.11.0/24
```

This is one of the major benefits of MPLS L3VPNs.

The core stays simple.

```text
P router routing table:
Provider loopbacks and core links only

P router does not need:
Customer prefixes
VPN routes
VRFs
MP-BGP VPNv4 routes
```

---

## MP-BGP Between PE Routers

PE routers exchange customer VPN routes with each other using **MP-BGP** (Multiprotocol BGP).

Normal IPv4 BGP cannot carry overlapping customer routes by itself.

So L3VPNs use VPNv4 routes.

A VPNv4 route is made from:

```text
Route Distinguisher (64 bits) + IPv4 prefix (32 bits)
```

This results in a 96-bit VPNv4 route.

Example:

```text
Customer A VPNv4 route (RD 100:1):
100:1:10.10.10.0/24

Customer B VPNv4 route (RD 100:2):
100:2:10.10.10.0/24
```

The IPv4 prefix is the same, but the VPNv4 routes are different because the RDs are different.

This allows MP-BGP to carry overlapping customer routes.

---

## Route Distinguishers and Route Targets

Two important values are used in MPLS L3VPNs:

```text
RD = Route Distinguisher
RT = Route Target
```

They do different jobs.

- The **RD** makes customer routes unique in MP-BGP.
  - Overlapping IPv4 routes are made into unique VPNv4 routes by prepending a unique RD.
- The **RT** controls which VRFs import and export routes.

Simple version:

```text
RD:
Makes routes unique.

RT:
Controls VPN membership.
```

Example:

```text
VRF CUSTOMER_A
 rd 100:1
 route-target export 100:1
 route-target import 100:1
```

The RD does not decide which VRFs receive the route. That's the role of the RT.

> More info on RDs and RTs in a separate page.

---

## VPN Labels

When a PE advertises a VPNv4 route to another PE, it also advertises a VPN label.

The VPN label tells the egress PE how to handle the packet after it receives it.

For example, PE2 may advertise this VPNv4 route to PE1:

```text
VPN route: 100:1:11.11.11.0/24
Next hop:  4.4.4.4
VPN label: 200
```

PE1 now knows:

```text
To reach 11.11.11.0/24:
Send traffic toward PE2's loopback.
Use VPN label 200.
```

The VPN label is different from the transport label used by LDP.

---

## Two Labels in an L3VPN Packet

Most MPLS L3VPN packets use two labels.

```text
Top label:    Transport label
Bottom label: VPN label
```

The **transport label** gets the packet across the MPLS core to the egress PE.

The **VPN label** tells the egress PE which VRF or forwarding entry to use.

Example:

```text
CE1 sends packet to 11.11.11.1
  ↓
PE1 receives the packet in VRF CUSTOMER_A
  ↓
PE1 pushes two labels
  ↓
Bottom label: VPN label for CUSTOMER_A route
Top label: transport label to PE2
  ↓
P routers switch based on the top label
  ↓
PE2 receives the packet
  ↓
PE2 uses the VPN label to forward the packet toward CE2
```

The P routers only look at the transport label.

They do not need to know about the customer route or the VPN label meaning.

VPN labels are only exchanged between PE routers using MP-BGP.

---

## Control Plane vs Data Plane

MPLS L3VPNs have several control-plane pieces.

```text
IGP:
Provides reachability to PE loopbacks.

LDP:
Provides transport labels for PE loopbacks.

MP-BGP VPNv4:
Carries customer VPN routes between PE routers.

PE-CE routing:
Exchanges customer routes between CE and PE.
```

They work together.

Example:

```text
CE2 advertises 11.11.11.0/24 to PE2.
PE2 installs the route in the customer VRF.
PE2 advertises the route to PE1 using MP-BGP VPNv4.
PE1 imports the route into the correct VRF.
CE1 can now reach 11.11.11.0/24 through the MPLS VPN.
```

The data plane then uses MPLS labels to carry the traffic across the provider core.

---

## Basic Packet Path

Assume CE1 sends traffic to CE2.

```text
Source:      10.10.10.1
Destination: 11.11.11.1
```

Packet walk:

```text
1. CE1 forwards the packet to PE1.

2. PE1 receives the packet on a VRF interface.

3. PE1 looks up 11.11.11.1 in the VRF routing table.

4. PE1 finds a VPN route learned from PE2.

5. PE1 pushes a VPN label and a transport label.

6. P routers forward the packet using the transport label.

7. The penultimate hop may pop the transport label.

8. PE2 receives the packet with the VPN label.

9. PE2 uses the VPN label to identify the correct VRF or forwarding entry.

10. PE2 forwards the packet to CE2.
```

The customer packet is carried across the core without the P routers knowing customer routes.

---

## Why the PE Loopback Is Important

In MPLS L3VPNs, VPNv4 routes usually use the remote PE loopback as the BGP next hop.

> This concept was explained in the MPLS section.

Example:

```text
PE2 advertises 11.11.11.0/24 to PE1.

BGP next hop: 4.4.4.4
```

PE1 must be able to reach `4.4.4.4`.

The provider core must also have labels for `4.4.4.4`.

That is why the general MPLS section focused on:

```text
IGP reachability to PE loopbacks
LDP label bindings for PE loopbacks
MPLS LSP ping and traceroute to PE loopbacks
```

If the transport LSP to the remote PE loopback is broken, L3VPN traffic will fail.

---

## What the P Routers Know

P routers do not run VRFs for customer VPNs.

P routers do not need MP-BGP VPNv4.

P routers do not need customer routes.

They only need provider core reachability and MPLS forwarding.

```text
P router needs:
IGP
LDP
LFIB entries for provider prefixes

P router does not need:
VRFs
RDs
RTs
VPNv4 routes
Customer prefixes
PE-CE routing
```

This is why MPLS L3VPNs scale well.

The edge routers handle the VPN complexity.

The core router configuration remains simple.

---

## Key Components

| Component | Purpose |
| :--- | :--- |
| VRF | Separate customer routing table on a PE |
| RD | Makes customer routes unique in MP-BGP |
| RT | Controls which VRFs import and export routes |
| MP-BGP VPNv4 | Carries VPN routes between PE routers |
| VPN label (bottom label) | Tells the egress PE how to forward the VPN packet |
| Transport label (top label) | Carries the packet across the MPLS core |
| PE-CE routing | Exchanges customer routes between CE and PE |
| LDP | Provides transport labels for PE loopbacks |
| IGP | Provides reachability inside the provider core |

---

## Key Points

* MPLS L3VPNs allow a provider to carry customer routes across an MPLS core.
* Customer routes are stored in VRFs on PE routers.
* P routers do not need customer routes or VRFs.
* MP-BGP VPNv4 carries customer routes between PE routers.
* The RD makes VPNv4 routes unique.
* The RT controls which VRFs import and export routes.
* The egress PE advertises a VPN label with each VPN route.
* L3VPN packets usually use two labels.
  * The transport label gets the packet to the remote PE.
  * The VPN label tells the egress PE how to forward the packet.
* The remote PE loopback is usually the BGP next hop for VPNv4 routes.
* Before troubleshooting L3VPNs, verify the provider IGP, LDP, and transport LSP.
