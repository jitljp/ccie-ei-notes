# VRFs

A **VRF** is a separate routing table on a router.

In an MPLS L3VPN, VRFs are used on PE routers to keep customer routes separate.

```text
PE router
├── Global routing table
├── VRF CUSTOMER_A
└── VRF CUSTOMER_B
```

The provider core uses the global routing table.

Customer routes are placed in VRFs.

---

## Why VRFs Are Needed

Different customers can use the same IP addresses.

Example:

```text
Customer A:
10.10.10.0/24

Customer B:
10.10.10.0/24
```

If both routes were placed in the global routing table, they would overlap.

A normal IPv4 routing table cannot install both routes as separate customer routes.

VRFs solve this problem by creating separate routing tables.

```text
VRF CUSTOMER_A:
10.10.10.0/24

VRF CUSTOMER_B:
10.10.10.0/24
```

The prefixes are the same, but they are in different VRFs, so they do not conflict.

---

## VRFs Are Local to a PE

A VRF is locally configured on a PE router.

The VRF name itself is locally significant.

Example:

```text
PE1:
VRF CUSTOMER_A

PE2:
VRF CUSTOMER_A
```

The names match in this example, but they do not have to.

What matters for MPLS L3VPN route exchange is not the VRF name.

The important values are:

```text
RD = Route Distinguisher
RT = Route Target
```

The RD and RT are covered in the next section.

For now, remember:

```text
VRF name:
Local to the router.

RD and RT:
Used for VPN route exchange and VPN membership.
```

---

## VRFs on PE Routers

In an MPLS L3VPN, the PE routers have customer-facing interfaces.

Those interfaces are placed into VRFs.

Example:

```text
CE1 --- PE1
      |
    VRF CUSTOMER_A
```

The PE interface connected to CE1 belongs to `CUSTOMER_A`.

Routes learned from CE1 are placed into the `CUSTOMER_A` VRF routing table.

They are not placed into the global routing table.

```text
CE1 route learned by PE1
  ↓
Installed in VRF CUSTOMER_A
```

---

## P Routers Do Not Need VRFs

P routers are in the provider core.

They do not connect to customer sites, and they do not need VRFs or customer routes.

```text
CE1 --- PE1 --- P1 --- P2 --- PE2 --- CE2
        VRF                   VRF

P1 and P2:
No customer VRFs
No customer routes
No MP-BGP
```

The P routers only need provider core reachability and MPLS forwarding.

This is one of the main scaling benefits of MPLS L3VPNs.

---

## Global Routing Table vs VRF Routing Table

The global routing table is used for provider infrastructure reachability.

Examples:

```text
PE loopbacks
P router loopbacks
Core links
LDP transport
MP-BGP next-hop reachability
```

A VRF routing table is used for customer routes.

On a PE, this means there are multiple routing tables.

```text
Global routing table:
1.1.1.1/32
2.2.2.2/32
3.3.3.3/32
5.5.5.5/32

VRF CUSTOMER_A:
10.10.10.0/24
11.11.11.0/24
```

A route in a VRF is separate from a route in the global table.

---

## Creating a VRF

On modern IOS XE, VRFs are configured with `vrf definition`.

Example:

```text
PE1(config)# vrf definition CUSTOMER_A
PE1(config-vrf)# rd 65000:1
PE1(config-vrf)# address-family ipv4
PE1(config-vrf-af)# route-target export 65000:1
PE1(config-vrf-af)# route-target import 65000:1
```

This creates a VRF called `CUSTOMER_A`.

The `rd` and `route-target` commands are required for MPLS L3VPN route exchange.

They will be explained in more detail in another page.

For now, remember these points:

```text
rd:
Makes VPNv4 routes unique.

route-target export:
Marks routes exported from the VRF.

route-target import:
Controls which VPNv4 routes are imported into the VRF.
```

---

## Assigning an Interface to a VRF

A customer-facing PE interface is placed into the VRF with `vrf forwarding`.

Example:

```text
PE1(config)# interface Ethernet0/0
PE1(config-if)# vrf forwarding CUSTOMER_A
PE1(config-if)# ip address 192.168.1.1 255.255.255.0
```

Configure `vrf forwarding` before the IP address.

On IOS XE, applying `vrf forwarding` to an interface removes the existing IP address from the interface.

So this order is best:

```text
1. Configure the VRF.
2. Enter the interface.
3. Apply `vrf forwarding`.
4. Configure the IP address.
```

---

## PE-CE Routing Happens Inside the VRF

PE-CE routing runs inside the customer VRF.

Examples:

```text
Static routing
OSPF
EIGRP
BGP
```

For example, a static route to a customer LAN uses `ip route vrf`.

```text
PE1(config)# ip route vrf CUSTOMER_A 10.10.10.0 255.255.255.0 192.168.1.2
```

This installs the static route into the `CUSTOMER_A` VRF.

Typically, PE-CE routing uses a routing protocol like OSPF, EIGRP, or BGP.

> I will cover PE-CE routing in detail in another section.

---

## VRFs and MP-BGP

A VRF by itself is only a local routing table.

MPLS L3VPNs also need MP-BGP to exchange customer routes between PE routers.

Example:

```text
CE1 advertises 10.10.10.0/24 to PE1.
PE1 installs the route in VRF CUSTOMER_A.
PE1 converts the route into a VPNv4 route.
PE1 advertises the VPNv4 route to PE2 using MP-BGP.
PE2 imports the route into its local customer VRF.
```

The VRF is where customer routes live on the PE, and MP-BGP is how those routes are shared between PEs.

---

## VRFs and MPLS Labels

VRFs separate customer routing tables.

MPLS labels forward customer traffic across the provider core.

They are related, but they are not the same thing.

```text
VRF:
Selects the customer routing table on the PE.

Transport label:
Carries the packet across the MPLS core.

VPN label:
Tells the egress PE how to forward the packet after it reaches the PE.
```

When traffic enters PE1 from CE1:

```text
1. PE1 receives the packet on a VRF interface.
2. PE1 performs a lookup in the VRF routing table.
3. PE1 pushes a VPN label and a transport label.
4. P routers forward the packet using the transport label.
5. PE2 uses the VPN label to forward the packet in the correct VRF.
```

---

## VRF-Lite vs MPLS L3VPN VRFs

VRFs can also be used without MPLS.

This is called **VRF-Lite**.

In VRF-Lite, a router uses multiple routing tables locally, but there is no MPLS VPN.

```text
VRF-Lite:
VRFs without MPLS VPN service.

MPLS L3VPN:
VRFs + MP-BGP VPNv4 + MPLS labels.
```

VRF-Lite is what many students are familiar with from CCNA (and CCNP) studies.

---

## Key Points

* A VRF is a separate routing table on a router.
* In MPLS L3VPNs, VRFs are configured on PE routers.
* Customer-facing PE interfaces are assigned to VRFs.
* Customer routes are installed in VRF routing tables, not the global routing table.
* P routers do not need customer VRFs or customer routes.
* The VRF name is locally significant.
* RDs and RTs are used for MPLS L3VPN route exchange.
* Use VRF-aware commands such as `show ip route vrf`, `ping vrf`, and `traceroute vrf`.
* Configure `vrf forwarding` before the interface IP address.
* A VRF by itself is not a complete MPLS L3VPN.
  * It is just a separate logical routing table on a PE router.
  * MPLS L3VPNs also use MP-BGP VPNv4 routes and MPLS labels.
