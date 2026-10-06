# VRF-Lite Overview

VRF stands for **Virtual Routing and Forwarding**.

A VRF allows one router or Layer 3 switch to maintain multiple separate routing tables.

Each VRF is like a separate virtual router.

```text
One physical router
    |
    +-- Global routing table
    +-- VRF CUSTOMER_A routing table
    +-- VRF CUSTOMER_B routing table
    +-- VRF MANAGEMENT routing table
```

Routes in one VRF are separate from routes in another VRF.

By default, a route in one VRF is not used to forward traffic in another VRF.

## Why VRFs Are Used

VRFs are used to provide Layer 3 separation.

Common uses include:

```text
Separate customers
Separate departments
Separate management traffic
Overlapping IP address space
Lab segmentation
MPLS L3VPN customer routing
```

A single router can have multiple interfaces connected to different customers.

Each customer can have its own routing table.

The same IP address space can even be reused in different VRFs.

Example:

```text
CUSTOMER_A uses 10.10.10.0/24
CUSTOMER_B uses 10.10.10.0/24
```

This works because the routes are stored in different routing tables.

```text
show ip route vrf CUSTOMER_A
    10.10.10.0/24

show ip route vrf CUSTOMER_B
    10.10.10.0/24
```

The prefix is the same, but the routing context is different.

## VRF-Lite

**VRF-Lite** means VRFs are used without MPLS in the forwarding path.

In VRF-Lite, the router separates traffic by assigning Layer 3 interfaces to VRFs.

```text
Packet enters interface Gi0/1
    Gi0/1 belongs to VRF CUSTOMER_A
    Router looks up the destination in CUSTOMER_A's routing table

Packet enters interface Gi0/2
    Gi0/2 belongs to VRF CUSTOMER_B
    Router looks up the destination in CUSTOMER_B's routing table
```

The input interface determines the VRF.

VRF-Lite does not require MPLS labels.

VRF-Lite does not require LDP.

VRF-Lite does not require MP-BGP VPNv4.

It is simply multiple routing tables on the same device.

## VRF-Lite vs MPLS L3VPN

VRF-Lite and MPLS L3VPN both use VRFs.

The difference is how routes and packets are carried between routers.

| Feature | VRF-Lite | MPLS L3VPN |
|---|---|---|
| Uses VRFs | Yes | Yes |
| Uses MPLS labels | No | Yes |
| Uses LDP | No | Usually yes |
| Uses MP-BGP VPNv4/VPNv6 | Not required | Yes |
| Main purpose | Local routing-table separation | Provider VPN service across a core |
| Core awareness | No MPLS core | P routers forward labeled traffic |
| Typical role | Enterprise router, CE, firewall edge, lab router | PE router in MPLS L3VPN |

VRF-Lite is often used at the edge of a network.

MPLS L3VPN uses VRFs on PE routers and carries VPN routes across a provider core.

## Global Routing Table

The normal routing table is often called the **global routing table**.

```text
show ip route
```

This command shows the global routing table.

VRF routes are not shown in the global routing table.

To view a VRF routing table, specify the VRF.

```text
show ip route vrf CUSTOMER_A
show ip route vrf CUSTOMER_B
```

A common mistake is checking only `show ip route` and thinking a VRF route is missing.

The route may exist, but inside a VRF.

## Interface Membership

A Layer 3 interface can belong to only one VRF at a time.

Examples of Layer 3 interfaces include:

```text
Routed physical interface
Subinterface
SVI
Loopback
Tunnel
```

When an interface is placed into a VRF, connected routes from that interface are installed into the VRF routing table.

Example:

```text
interface GigabitEthernet0/1
 vrf forwarding CUSTOMER_A
 ip address 10.10.10.1 255.255.255.0
```

The connected route is installed in `CUSTOMER_A`.

```text
show ip route vrf CUSTOMER_A connected
```

The route is not installed in the global routing table.

## Basic Topology

In this example, one router separates two customers.

Both customers use the same subnet.

```text
                         R1
                  +---------------+
                  |               |
CUSTOMER_A -------| E0/1          |
10.10.10.0/24     |               |
                  |               |
CUSTOMER_B -------| E0/2          |
10.10.10.0/24     +---------------+
```

R1 can use the same IP address on both interfaces because the interfaces are in different VRFs.

```text
Gi0/1 in VRF CUSTOMER_A: 10.10.10.1/24
Gi0/2 in VRF CUSTOMER_B: 10.10.10.1/24
```

Without VRFs, this would not be possible on the same router.

With VRFs, the addresses are in different routing contexts.

## Basic Configuration

Create the VRFs.

```text
R1(config)# vrf definition CUSTOMER_A
R1(config-vrf)# address-family ipv4

R1(config)# vrf definition CUSTOMER_B
R1(config-vrf)# address-family ipv4
```

Assign interfaces to the VRFs.

```text
R1(config)# interface Ethernet0/1
R1(config-if)# vrf forwarding CUSTOMER_A
R1(config-if)# ip address 10.10.10.1 255.255.255.0
R1(config-if)# no shutdown

R1(config)# interface Ethernet0/2
R1(config-if)# vrf forwarding CUSTOMER_B
R1(config-if)# ip address 10.10.10.1 255.255.255.0
R1(config-if)# no shutdown
```

### Configuration Order

Configure the VRF before assigning the interface IP address.

Recommended order:

```text
1. Create the VRF.
2. Enter interface configuration mode.
3. Assign the interface to the VRF.
4. Configure the IP address.
```

Example:

```text
R1(config)# vrf definition CUSTOMER_A
R1(config-vrf)# address-family ipv4

R1(config)# interface Ethernet0/1
R1(config-if)# vrf forwarding CUSTOMER_A
R1(config-if)# ip address 10.10.10.1 255.255.255.0
```

Changing VRF membership on an interface removes the existing IP address.

This is why you should configure `vrf forwarding` before `ip address`.

## Forwarding Behavior

A router chooses the routing table based on the incoming interface.

Example:

```text
Packet enters Gi0/1.
Gi0/1 belongs to CUSTOMER_A.
R1 checks the CUSTOMER_A routing table.
```

The global routing table is not checked.

The `CUSTOMER_B` routing table is not checked.

Only the `CUSTOMER_A` routing table is used.

If there is no matching route in that VRF, the packet is dropped.

A matching route in the global routing table does not help.

A matching route in another VRF does not help.

## VRFs Are Separate by Default

VRFs do not leak routes by default.

Example:

```text
CUSTOMER_A has a route to 10.10.10.0/24.
CUSTOMER_B has a route to 10.10.10.0/24.
The global routing table has a default route.
```

These routing tables are separate.

```text
CUSTOMER_A cannot use CUSTOMER_B routes by default.
CUSTOMER_B cannot use CUSTOMER_A routes by default.
VRFs cannot use global routes by default.
The global table cannot use VRF routes by default.
```

To allow communication between VRFs, you must configure route leaking or use a service path.

> A **service path** means traffic does not move directly from one VRF to another by route leaking.
>
> Instead, traffic is sent through some intermediate device that connects the VRFs.

Route leaking is covered later in this section.

## VRF-Aware Commands

Many commands have a VRF-aware version.

Examples:

```text
ping vrf CUSTOMER_A 10.10.10.2
traceroute vrf CUSTOMER_A 10.10.10.2
show ip route vrf CUSTOMER_A
show ip cef vrf CUSTOMER_A
```

Without the `vrf` keyword, the router uses the global routing table.

Example:

```text
R1# ping 10.10.10.2
```

This uses the global routing table.

```text
R1# ping vrf CUSTOMER_A 10.10.10.2
```

This uses the `CUSTOMER_A` routing table.

## Routing Protocols and VRFs

Routing protocols can run inside a VRF.

Examples:

```text
OSPF in VRF CUSTOMER_A
EIGRP in VRF CUSTOMER_A
BGP address-family ipv4 vrf CUSTOMER_A
Static routes in VRF CUSTOMER_A
```

A routing process or address family must be VRF-aware to install routes into the correct table.

Examples:

```text
ip route vrf CUSTOMER_A 0.0.0.0 0.0.0.0 10.10.10.254

router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 10.10.10.2 remote-as 65100
  neighbor 10.10.10.2 activate
```

VRF-aware routing is covered in the next page.

## RD and RT in VRF-Lite

VRFs are often configured with an RD and route targets.

```text
vrf definition CUSTOMER_A
 rd 65000:1
 address-family ipv4
  route-target export 65000:1
  route-target import 65000:1
```

These values are important in MPLS L3VPN and BGP-based route leaking.

**RD** stands for **Route Distinguisher**.

The RD makes otherwise identical prefixes unique in VPN routing.

Example:

```text
CUSTOMER_A: 10.10.10.0/24
CUSTOMER_B: 10.10.10.0/24
```

By adding RDs, these can become unique VPN routes.

```text
65000:1:10.10.10.0/24
65000:2:10.10.10.0/24
```

Although the IPv4 prefix is the same, the RDs make them unique VPNv4 routes.

**RT** stands for **Route Target**.

Route targets control which VPN routes are imported into or exported from a VRF.

RDs and RTs matter when using BGP-based route leaking or MPLS L3VPN-style control-plane behavior.

## Management VRFs

Many routers have a dedicated management VRF.

A management VRF separates management traffic from production traffic.

Examples:

```text
SSH source interface in a management VRF
NTP server reachable through a management VRF
TACACS+ server reachable through a management VRF
Syslog server reachable through a management VRF
```

When a service must use a VRF, the service configuration usually needs a VRF-aware option.

Example:

```text
ntp server vrf MGMT 192.0.2.10
ip tacacs source-interface Ethernet0/0
```

The second command doesn't specify a VRF, but assume that Ethernet0/0 is assigned to the management VRF.

## Common Mistakes

### Checking the Wrong Routing Table

```text
show ip route
```

This shows the global routing table.

For a VRF, use:

```text
show ip route vrf CUSTOMER_A
```

### Forgetting the VRF Keyword

This uses the global table:

```text
ping 10.10.10.2
```

This uses the VRF table:

```text
ping vrf CUSTOMER_A 10.10.10.2
```

### Configuring the IP Address Before VRF Membership

This order causes the IP address to be removed:

```text
interface GigabitEthernet0/1
 ip address 10.10.10.1 255.255.255.0
 vrf forwarding CUSTOMER_A
```

Use this order instead:

```text
interface GigabitEthernet0/1
 vrf forwarding CUSTOMER_A
 ip address 10.10.10.1 255.255.255.0
```

## Key Points

- VRF-Lite allows one physical router or Layer 3 switch to act like multiple virtual routers.
- Each VRF has its own routing table.
- Interfaces are assigned to VRFs.
- The incoming interface determines which VRF routing table is used.
- The same IP prefix can exist in multiple VRFs.
- The global routing table and VRF routing tables are separate.
- VRFs do not share routes by default.
- Use VRF-aware commands to test and verify VRF traffic.
- VRF-Lite does not require MPLS.