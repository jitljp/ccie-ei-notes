# VRF Route Leaking

A **VRF** maintains its own routing and forwarding tables. By default, routes in one VRF are not available in another VRF or the global routing table.

**Route leaking** allows selected routes to be made available across those routing-table boundaries.

Typical uses include:

- Allowing different VRFs to reach shared services such as DNS, DHCP, or application servers.
- Allowing a management VRF to reach selected devices outside that VRF.
- Connecting two otherwise isolated networks.
- Providing selected VRFs with access to a common Internet or security-services path.

Route leaking is an exception to VRF isolation. It does not merge the routing tables or automatically share every route.

## Main Idea

Consider a router with two VRFs:

```text
                         R1
                    +---------------------+
      10.10.10.0/24 |                     |
        CUSTOMER_A  |  VRF CUSTOMER_A     |
        network ----|  E0/0               |
   192.168.20.0/24  |                     |
        CUSTOMER_B  |  VRF CUSTOMER_B     |
        network ----|  E0/1               |
                    |                     |
                    +---------------------+
```

Without route leaking:

```text
VRF CUSTOMER_A                         VRF CUSTOMER_B
10.10.10.0/24                          192.168.20.0/24
No route to 192.168.20.0/24             No route to 10.10.10.0/24
```

If hosts in `CUSTOMER_A` must communicate with hosts in `CUSTOMER_B`, the router needs a way to forward traffic between the two routing contexts.

One possible solution is **bidirectional route leaking**:

```text
VRF CUSTOMER_A                         VRF CUSTOMER_B
10.10.10.0/24                          192.168.20.0/24
192.168.20.0/24 (leaked)               10.10.10.0/24 (leaked)
```

The VRFs remain separate. Only the selected routes are shared.

> **Important:** A route appearing in a second VRF does not, by itself, guarantee end-to-end connectivity. The forwarding mechanism must support the inter-VRF path, and the destination must have a valid return path. ACLs, firewalls, and other policies can still block traffic.

## Route Leaking Directions

Route leaking can occur in several directions.

| Direction | Example |
| --- | --- |
| VRF to VRF | Import a route from `CUSTOMER_B` into `CUSTOMER_A` |
| Global to VRF | Allow `CUSTOMER_A` to reach a server through the global routing table |
| VRF to Global | Make a management subnet in `MGMT` reachable from the global routing table |
| Bidirectional | Exchange selected routes in both directions |

**Unidirectional route leaking** means only one routing table receives selected routes from another. It does not mean that application traffic can necessarily succeed in one direction without a return route.

The global routing table can be involved in route leaking, but it is not itself a configured named VRF in traditional IOS XE VRF-Lite.

## Route Leaking vs. Redistribution

These concepts are related but distinct.

**Route leaking** makes routes available across separate routing tables.

**Redistribution** introduces routes learned by one routing source into another routing protocol.

For example:

```text
VRF CUSTOMER_A
  OSPF route to 10.10.10.0/24
           |
           | Route leaking
           v
VRF CUSTOMER_B
  Route to 10.10.10.0/24
           |
           | Redistribution (optional)
           v
CUSTOMER_B routing protocol advertises the route to its neighbors
```

A leaked route can be usable locally without being redistributed to other routers. Conversely, ordinary redistribution inside `CUSTOMER_A` does not automatically copy routes into `CUSTOMER_B`.

## Route Leaking Methods

IOS XE supports multiple techniques. They differ in how routes are selected, installed, and forwarded.

| Method | Main Idea | Typical Use |
| --- | --- | --- |
| Static route leaking | Configure routes that use an appropriate inter-VRF forwarding path or next-hop resolution mechanism | A few known prefixes |
| Route-target import/export | Use BGP VPN routes and route targets to control which VRFs import routes | BGP-based sharing between VRFs |
| Import/export maps | Use route maps with supported BGP-based import/export mechanisms | Selective policy-controlled route leaking |
| Global-to-VRF BGP prefix import | Import selected global-table prefixes into a VRF through a BGP-based feature | Sharing global routes with a VRF |
| Route replication | Copy selected RIB routes from another VRF using `route-replicate` | Direct, policy-controlled RIB route sharing |
| VASI | Forward packets through a virtual interface pair connecting routing contexts | Inter-VRF forwarding with NAT, firewall, or other services |

The best method depends on the platform, software release, route sources, need for policy, and forwarding requirements.

### Static VRF Route Leaking

Static routes can provide controlled reachability across routing contexts when configured with a supported forwarding or next-hop resolution mechanism.

Two relevant cases are:

- **VRF to global:** A VRF static route can resolve its next hop through the **global routing table** using the `global` keyword.
- **Global to VRF:** A global static route can reference a VRF-facing exit interface with a fully specified next hop, subject to the platform's forwarding support.

For example, this VRF static route resolves its next hop using the global routing table:

```text
ip route vrf CUSTOMER_A <destination> <mask> <global-next-hop> global
```

The `global` keyword does **not** import every global route into `CUSTOMER_A`. The reverse direction uses different configuration, and reciprocal static routes are not a general solution for direct forwarding between two nondefault VRFs.

Unlike dynamic mechanisms, static routes must generally be updated when the destination or path changes. A plain static route in `CUSTOMER_A` does **not** automatically gain access to the routing table of `CUSTOMER_B`.

Both directions, next-hop resolution, forwarding restrictions, and return-path configuration are covered in [Static VRF Route Leaking](static-route-leaking.md).

### Route-Target Import/Export

BGP can use **route targets (RTs)** to control which VRFs import VPN routes.

Conceptually:

```text
VRF CUSTOMER_A
  Export RT 65000:100
          |
          v
  BGP VPN route carrying RT 65000:100
          |
          v
VRF SHARED_SERVICES
  Import RT 65000:100
```

An import RT controls route eligibility for import. It is **not** a packet-forwarding label and does not establish a BGP neighbor by itself.

BGP-based route exchange also requires the relevant routes to enter BGP and the necessary forwarding support. Do not confuse route-target import/export with ordinary redistribution or direct RIB route replication.

See [Route-Target Import/Export](route-target-import-export.md).

### Import and Export Maps

**Route maps** allow selective route matching and policy within supported leaking mechanisms.

For example, a prefix list might permit only:

```text
10.10.10.0/24
```

A route map can then be used to select that prefix, rather than leaking every route in the source VRF.

IOS XE supports BGP-based `import ipv4 unicast map` and `export ipv4 unicast map` configurations in relevant scenarios. Their exact behavior differs from a route map attached to `route-replicate` or to BGP VPN route-target import/export.

See [Import and Export Maps](import-export-maps.md).

### Global-to-VRF BGP Prefix Import

IOS XE also supports a BGP-based approach to importing selected **global routing table** prefixes into a VRF using a route map.

For example:

```text
Global routing table
  203.0.113.0/24
         |
         | Selected by route map
         v
VRF CUSTOMER_A
  203.0.113.0/24 (imported)
```

This is different from a VRF static route that resolves a next hop through the global table with `global`.

See [Global-to-VRF BGP Prefix Import](global-vrf-bgp-prefix-import.md).

### Route Replication

The **route replication** feature copies eligible routes between routing tables without requiring a BGP VPN route-import/export process for the leak.

Example on the receiving VRF:

```text
R1(config)# vrf definition CUSTOMER_B
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# route-replicate from vrf CUSTOMER_A unicast connected
```

This requests replication of eligible connected routes **from `CUSTOMER_A` into `CUSTOMER_B`**.

The direction is important:

```text
Configured under: CUSTOMER_B   (destination VRF)
From VRF:         CUSTOMER_A   (source VRF)
```

Depending on the platform and IOS XE release, route replication supports additional route sources, including static, OSPF, EIGRP, and BGP routes. A route map can limit which routes are replicated.

In supported IOS XE routing-table output, replicated routes are identified with a **`+`** marker. An **`&`** marker identifies a replicated local route overridden by a connected route in the receiving VRF.

See [Route Replication](route-replication.md).

### VASI

**VASI (VRF-Aware Software Infrastructure)** provides a forwarding path between routing contexts using two logically connected virtual interfaces.

The interfaces are paired by their index:

```text
VRF CUSTOMER_A                      VRF CUSTOMER_B
   VASILEFT100 <----------------> VASIRIGHT100
```

A packet forwarded out `VASILEFT100` is internally handed to `VASIRIGHT100`, where forwarding continues in the other VRF.

VASI is useful when traffic crossing VRFs must be processed by supported services, such as **NAT** or **zone-based firewall** policies.

VASI is **not itself a route-import mechanism**. Routing must direct packets toward the VASI interfaces, using static or supported dynamic routing.

See [VASI](vasi.md).

## Route Maps and Route Selection

Route maps are commonly used to control which prefixes are leaked.

For example:

```text
R1(config)# ip prefix-list SHARED_SERVICES permit 192.168.100.0/24
R1(config)# route-map LEAK_SERVICES permit 10
R1(config-route-map)# match ip address prefix-list SHARED_SERVICES
```

The route map can be referenced by a leaking feature that supports route-map filtering.

**The route map alone does not leak any routes.** It must be attached to an applicable route-leaking command.

Route maps can match routes and, depending on the feature, modify their attributes. Supported match/set clauses vary by route-leaking mechanism.

## Forwarding and Return Paths

Consider a host in each VRF:

```text
Host A: 10.10.10.10/24         Host B: 192.168.20.20/24
         VRF A                           VRF B
```

For Host A to communicate with Host B:

1. Host A must send its traffic toward the router that connects the routing contexts.
2. VRF A must have a usable forwarding path toward `192.168.20.20`.
3. The router must be able to deliver that packet into VRF B through the selected leaking or forwarding mechanism.
4. VRF B and Host B must have a return path toward `10.10.10.10`.
5. Any intervening ACL, NAT, firewall, or security policy must permit the required traffic.

**A route installed in the RIB is not proof that forwarding works.** Verify the forwarding table and test end-to-end connectivity.

## Overlapping IP Addresses

VRFs can use identical prefixes while they remain isolated.

For example:

```text
CUSTOMER_A: 192.168.1.0/24
CUSTOMER_B: 192.168.1.0/24
```

However, leaking identical prefixes into a common routing table creates ambiguity. That table cannot ordinarily distinguish two destinations represented by exactly the same IPv4 prefix solely through destination-based routing.

When overlapping address space must communicate, additional design choices such as NAT or separate forwarding contexts may be necessary. Simply leaking the routes is not enough.

## Verification

Useful IOS XE commands include:

```text
show vrf
show ip route vrf CUSTOMER_A
show ip route vrf CUSTOMER_B
show ip cef vrf CUSTOMER_A
show ip cef vrf CUSTOMER_B
show ip route
show ip bgp vpnv4 all
ping vrf CUSTOMER_A <destination>
traceroute vrf CUSTOMER_A <destination>
```

The appropriate BGP and VASI verification commands depend on the method being used. Not every route-leaking design requires VPNv4 BGP.

When checking a leaked route, verify:

1. The original route exists in its source routing table.
2. The route matches the leak policy.
3. The destination VRF receives and selects the expected route.
4. The selected forwarding path is valid.
5. Return routing and end-to-end connectivity work.

## Common Problems

- **Only one direction is leaked:** The remote destination cannot send replies back.
- **Route map does not match:** The expected prefix is not selected for leaking.
- **Route is present but not installed:** Another route may be preferred, or forwarding/next-hop resolution may be invalid.
- **Wrong source or destination VRF:** The leaking configuration is applied in the wrong routing context.
- **BGP route is missing:** A route may need to be originated or redistributed into the relevant BGP address family before BGP-based leaking can use it.
- **Overlapping prefixes:** The destination table may not be able to represent the intended destinations unambiguously.
- **Routing loops:** Bidirectional leaking and redistribution can reintroduce routes unless policies are carefully designed.
- **Platform limitations:** Supported commands, protocols, and IPv4/IPv6 behavior depend on IOS XE version and platform.

## Key Points

- VRFs do not exchange routes by default.
- **Route leaking** makes selected routes available across VRF or global routing-table boundaries.
- Leaking can be **unidirectional or bidirectional**.
- **Route leaking and routing-protocol redistribution are different operations.**
- Static, BGP-based, and RIB route-replication methods provide different ways to share routes.
- **Route targets** control BGP VPN route import eligibility; **route maps** can filter routes in supported leaking mechanisms.
- **`route-replicate from vrf`** is configured under the **destination VRF** address family and identifies the **source VRF**.
- **`+`** identifies a replicated route in supported IOS XE route output.
- **VASI** provides an inter-VRF forwarding path, especially useful when applying services such as NAT or firewalls.
- **Return routes, next-hop resolution, and forwarding policies** must also be considered.
- Route leaking does not automatically solve overlapping-address conflicts.

## References

- [Cisco: Configure VRF Leaks on IOS XE](https://www.cisco.com/c/en/us/support/docs/ip/ip-routing/216541-vrf-configuration-examples-on-ios-xe.html)
- [Cisco: Configure Route Leaking with Route Replication Feature](https://www.cisco.com/c/en/us/support/docs/ip/interior-gateway-protocol-igp/226098-configure-route-leaking-with-route.html)
- [Cisco: Configure Route Leak Between Global and VRF Routing Table without Next-Hop](https://www.cisco.com/c/en/us/support/docs/ip/ip-routing/200158-Configure-Route-Leaking-between-Global-a.html)
- [Cisco: Configuring the VRF-Aware Software Infrastructure](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_conf-vasi.html)
