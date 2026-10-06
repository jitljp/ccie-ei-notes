# PE-CE Static Routing

Static routing is the simplest PE-CE routing option in an MPLS L3VPN.

The CE router does not run a routing protocol with the PE.

Instead, both sides use static routes.

```text
CE:
Static route toward the PE.

PE:
VRF static route toward the CE.
```

This page focuses on how static routing fits into an MPLS L3VPN.

---

## Basic Topology

Use the same topology:

```text
Customer A Site 1                    Customer A Site 2

10.10.10.0/24                        11.11.11.0/24
      |                                      |
     CE1                                    CE2
      |                                      |
     PE1 -------- P1 --------- P2 --------- PE2
    1.1.1.1     2.2.2.2      3.3.3.3      4.4.4.4
```

PE-CE addressing:

```text
CE1 --- PE1: 192.168.1.0/24
CE2 --- PE2: 192.168.2.0/24
```

Example interface addresses:

```text
PE1 to CE1: 192.168.1.1/24
CE1 to PE1: 192.168.1.2/24

PE2 to CE2: 192.168.2.1/24
CE2 to PE2: 192.168.2.2/24
```

Customer LANs:

```text
CE1 LAN: 10.10.10.0/24
CE2 LAN: 11.11.11.0/24
```

---

## Where Static Routing Is Used

With static PE-CE routing, static routes are configured in two places.

### CE to PE

The CE needs a route toward the provider.

This is often a default route:

```text
CE1(config)# ip route 0.0.0.0 0.0.0.0 192.168.1.1
CE2(config)# ip route 0.0.0.0 0.0.0.0 192.168.2.1
```

Or the CE can use specific static routes to remote customer prefixes:

```text
CE1(config)# ip route 11.11.11.0 255.255.255.0 192.168.1.1
CE2(config)# ip route 10.10.10.0 255.255.255.0 192.168.2.1
```

### PE to CE

The PE needs VRF static routes for the customer prefixes behind the CE.

On PE1:

```text
PE1(config)# ip route vrf CUSTOMER_A 10.10.10.0 255.255.255.0 192.168.1.2
```

On PE2:

```text
PE2(config)# ip route vrf CUSTOMER_A 11.11.11.0 255.255.255.0 192.168.2.2
```

These routes are installed in the `CUSTOMER_A` VRF.

---

## Basic Route Flow

For CE1's route to reach CE2, the flow is:

```text
1. PE1 has a VRF static route to 10.10.10.0/24.
2. PE1 redistributes the static route into BGP under the CUSTOMER_A VRF address family.
3. PE1 exports the route into VPNv4.
4. PE1 attaches the export RT and a VPN label.
5. PE1 advertises the VPNv4 route to PE2.
6. PE2 imports the route into CUSTOMER_A if the RT matches.
7. CE2 sends traffic for 10.10.10.0/24 to PE2 using its static/default route.
8. PE2 forwards the traffic through the MPLS L3VPN.
```

The important L3VPN-specific point is this:

```text
A VRF static route is not automatically advertised to other PEs.

It must be redistributed into BGP under the VRF address family.
```

---

## PE1 Configuration

PE1 has three main jobs:

```text
1. Put the CE-facing interface in the VRF.
2. Configure a VRF static route toward CE1.
3. Redistribute static routes into BGP for the VRF.
```

Example:

```text
PE1(config)# vrf definition CUSTOMER_A
PE1(config-vrf)# rd 65000:1
PE1(config-vrf)# address-family ipv4
PE1(config-vrf-af)# route-target both 1:1

PE1(config)# interface Ethernet0/1
PE1(config-if)# description To CE1
PE1(config-if)# vrf forwarding CUSTOMER_A
PE1(config-if)# ip address 192.168.1.1 255.255.255.0
PE1(config-if)# no shutdown

PE1(config)# ip route vrf CUSTOMER_A 10.10.10.0 255.255.255.0 192.168.1.2

PE1(config)# router bgp 65000
PE1(config-router)# address-family ipv4 vrf CUSTOMER_A
PE1(config-router-af)# redistribute static
```

The static route is placed in the VRF:

```text
ip route vrf CUSTOMER_A 10.10.10.0 255.255.255.0 192.168.1.2
```

The `redistribute static` command makes the route eligible for VPNv4 export:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute static
```

---

## PE2 Configuration

PE2 uses the same logic for CE2's customer LAN.

```text
PE2(config)# vrf definition CUSTOMER_A
PE2(config-vrf)# rd 65000:1
PE2(config-vrf)# address-family ipv4
PE2(config-vrf-af)# route-target both 1:1

PE2(config)# interface Ethernet0/1
PE2(config-if)# description To CE2
PE2(config-if)# vrf forwarding CUSTOMER_A
PE2(config-if)# ip address 192.168.2.1 255.255.255.0
PE2(config-if)# no shutdown

PE2(config)# ip route vrf CUSTOMER_A 11.11.11.0 255.255.255.0 192.168.2.2

PE2(config)# router bgp 65000
PE2(config-router)# address-family ipv4 vrf CUSTOMER_A
PE2(config-router-af)# redistribute static
```

Now PE2 can export `11.11.11.0/24` into VPNv4.

PE1 can import it into its local `CUSTOMER_A` VRF.

---

## CE1 Configuration

CE1 does not need MPLS, MP-BGP, VRFs, RDs, RTs, or VPN labels.

It only needs normal IP routing.

Example:

```text
CE1(config)# interface Ethernet0/0
CE1(config-if)# description To PE1
CE1(config-if)# ip address 192.168.1.2 255.255.255.0
CE1(config-if)# no shutdown

CE1(config)# interface Loopback0
CE1(config-if)# ip address 10.10.10.1 255.255.255.0

CE1(config)# ip route 0.0.0.0 0.0.0.0 192.168.1.1
```

The default route sends traffic for remote VPN prefixes to PE1.

```text
CE1 route:
0.0.0.0/0 via 192.168.1.1
```

For a more controlled configuration, CE1 could use a specific route instead:

```text
CE1(config)# ip route 11.11.11.0 255.255.255.0 192.168.1.1
```

---

## CE2 Configuration

CE2 uses the same approach.

```text
CE2(config)# interface Ethernet0/0
CE2(config-if)# description To PE2
CE2(config-if)# ip address 192.168.2.2 255.255.255.0
CE2(config-if)# no shutdown

CE2(config)# interface Loopback0
CE2(config-if)# ip address 11.11.11.1 255.255.255.0

CE2(config)# ip route 0.0.0.0 0.0.0.0 192.168.2.1
```

Or use a specific route:

```text
CE2(config)# ip route 10.10.10.0 255.255.255.0 192.168.2.1
```

---

## Complete PE-CE Static Route Logic

For CE1 to reach CE2:

```text
CE1:
Default route to PE1.

PE1:
VRF route to 11.11.11.0/24 learned from VPNv4.

PE1 core forwarding:
Transport label to PE2 + VPN label advertised by PE2.

PE2:
VRF static route to 11.11.11.0/24 via CE2.

CE2:
Connected route to 11.11.11.0/24.
```

For the return traffic:

```text
CE2:
Default route to PE2.

PE2:
VRF route to 10.10.10.0/24 learned from VPNv4.

PE2 core forwarding:
Transport label to PE1 + VPN label advertised by PE1.

PE1:
VRF static route to 10.10.10.0/24 via CE1.

CE1:
Connected route to 10.10.10.0/24.
```

L3VPN traffic requires working routes in both directions.

---

## Multiple Customer Prefixes

If CE1 has multiple customer prefixes, configure multiple VRF static routes on PE1.

Example:

```text
PE1(config)# ip route vrf CUSTOMER_A 10.10.10.0 255.255.255.0 192.168.1.2
PE1(config)# ip route vrf CUSTOMER_A 10.10.20.0 255.255.255.0 192.168.1.2
PE1(config)# ip route vrf CUSTOMER_A 10.10.30.0 255.255.255.0 192.168.1.2
```

If `redistribute static` is configured under the BGP VRF address family, all three static routes can be exported into VPNv4.

Example:

```text
PE1# show bgp vpnv4 unicast vrf CUSTOMER_A

Route Distinguisher: 65000:1 (default for vrf CUSTOMER_A)
 *>   10.10.10.0/24    192.168.1.2              0         32768 ?
 *>   10.10.20.0/24    192.168.1.2              0         32768 ?
 *>   10.10.30.0/24    192.168.1.2              0         32768 ?
 *>i  11.11.11.0/24    4.4.4.4                  0    100      0 ?
```

Locally redistributed static routes have weight `32768` because they originated on this router.

Remote VPN routes are learned through MP-BGP and appear with the `i` code because they are IBGP routes.

---

## Filtering Static Route Redistribution

In labs, it is common to use this:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute static
```

This is simple, but it redistributes all static routes in that VRF.

A safer approach is to filter redistribution with a route map.

Example:

```text
PE1(config)# ip prefix-list CUSTOMER_A-STATIC-EXPORT seq 5 permit 10.10.10.0/24
PE1(config)# ip prefix-list CUSTOMER_A-STATIC-EXPORT seq 10 permit 10.10.20.0/24
PE1(config)# ip prefix-list CUSTOMER_A-STATIC-EXPORT seq 15 permit 10.10.30.0/24

PE1(config)# route-map CUSTOMER_A-STATIC-EXPORT permit 10
PE1(config-route-map)# match ip address prefix-list CUSTOMER_A-STATIC-EXPORT

PE1(config)# router bgp 65000
PE1(config-router)# address-family ipv4 vrf CUSTOMER_A
PE1(config-router-af)# redistribute static route-map CUSTOMER_A-STATIC-EXPORT
```

This prevents accidental export of unrelated static routes in the VRF.

For CCIE labs, always pay attention to whether the task expects all static routes or only selected static routes to be advertised.

---

## Connected PE-CE Subnets

The PE-CE link subnet is connected in the VRF.

Example:

```text
PE1# show ip route vrf CUSTOMER_A connected

C        192.168.1.0/24 is directly connected, Ethernet0/1
```

This connected route is not exported into VPNv4 unless you redistribute connected routes.

Example:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute connected
```

In many L3VPN designs, you do not need to advertise the PE-CE transit subnet to other sites.

Advertise it only if there is a reason.

---

## Next-Hop Resolution for VRF Static Routes

The next hop of a VRF static route must be reachable in that VRF.

Example:

```text
PE1(config)# ip route vrf CUSTOMER_A 10.10.10.0 255.255.255.0 192.168.1.2
```

PE1 resolves `192.168.1.2` inside `CUSTOMER_A`.

Check reachability:

```text
PE1# show ip route vrf CUSTOMER_A 192.168.1.2
PE1# ping vrf CUSTOMER_A 192.168.1.2
```

If the CE-facing interface is not in the VRF, the static route may not install correctly or traffic may fail.

Check the interface:

```text
PE1# show running-config interface Ethernet0/1
```

Expected:

```text
interface Ethernet0/1
 vrf forwarding CUSTOMER_A
 ip address 192.168.1.1 255.255.255.0
```

---

## Static Routes and Administrative Distance

Static routes have an administrative distance of 1 by default.

Remote VPNv4 routes imported into the VRF as IBGP routes have an administrative distance of 200.

This matters if there is overlap.

Example:

```text
PE1 has a static route:
11.11.11.0/24 via 192.168.1.2

PE1 also learns a VPNv4 route:
11.11.11.0/24 via 4.4.4.4
```

The static route wins because its administrative distance is lower.

This can create blackholes if the static route is wrong.

For troubleshooting, always check the VRF routing table, not just the VPNv4 table.

```text
show ip route vrf CUSTOMER_A 11.11.11.0
show bgp vpnv4 unicast vrf CUSTOMER_A 11.11.11.0/24
```

A route can exist in BGP but lose the RIB installation decision.

---

## Verification on the PE

Start with the VRF routing table.

On PE1:

```text
PE1#show ip route vrf CUSTOMER_A

      10.0.0.0/24 is subnetted, 1 subnets
S        10.10.10.0 [1/0] via 192.168.1.2
      11.0.0.0/24 is subnetted, 1 subnets
B        11.11.11.0 [200/0] via 4.4.4.4, 00:57:34
      192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.1.0/24 is directly connected, Ethernet0/1
L        192.168.1.1/32 is directly connected, Ethernet0/1
```

Check VPNv4 routes:

```text
PE1# show bgp vpnv4 unicast all
BGP table version is 17, local router ID is 1.1.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:1 (default for vrf CUSTOMER_A)
 *>   10.10.10.0/24    192.168.1.2              0         32768 ?
 *>i  11.11.11.0/24    4.4.4.4                  0    100      0 ?
```

Check VPN labels:

```text
PE1# show bgp vpnv4 unicast all
BGP table version is 17, local router ID is 1.1.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:1 (default for vrf CUSTOMER_A)
 *>   10.10.10.0/24    192.168.1.2              0         32768 ?
 *>i  11.11.11.0/24    4.4.4.4                  0    100      0 ?
PE1#show bgp vpnv4 unicast all labels
   Network          Next Hop      In label/Out label
Route Distinguisher: 65000:1 (CUSTOMER_A)
   10.10.10.0/24    192.168.1.2     21/nolabel
   11.11.11.0/24    4.4.4.4         nolabel/21
```

Check the forwarding entry for a remote customer prefix:

```text
PE1# show ip cef vrf CUSTOMER_A 11.11.11.0/24 detail
11.11.11.0/24, epoch 0, flags [rib defined all labels]
  recursive via 4.4.4.4 label 21
    nexthop 10.1.1.2 Ethernet0/0 label 17-(local:16)
```

This confirms that PE1 has both labels:

```text
Transport label:
Gets traffic to PE2.

VPN label:
Tells PE2 how to forward the packet in the VPN.
```

---

## Common Problems

### Missing `ip route vrf`

A normal static route on the PE goes into the global routing table.

Wrong:

```text
PE1(config)# ip route 10.10.10.0 255.255.255.0 192.168.1.2
```

Correct:

```text
PE1(config)# ip route vrf CUSTOMER_A 10.10.10.0 255.255.255.0 192.168.1.2
```

Customer routes must be in the customer VRF.

---

### Missing `redistribute static`

The static route exists in the VRF but is not advertised to the remote PE.

Symptom on PE1:

```text
show ip route vrf CUSTOMER_A
```

The local static route exists.

But on PE2:

```text
show ip route vrf CUSTOMER_A
```

The remote route is missing.

Fix:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute static
```

---

### Static Route Next Hop Is Not Reachable in the VRF

If the PE cannot resolve the static route next hop inside the VRF, the route may not work.

Check:

```text
show ip route vrf CUSTOMER_A 192.168.1.2
ping vrf CUSTOMER_A 192.168.1.2
show running-config interface Ethernet0/1
```

Make sure the PE-CE interface is assigned to the correct VRF.

---

### Accidentally Redistributing Too Much

This command redistributes all static routes in the VRF:

```text
redistribute static
```

If the VRF contains static defaults, backup routes, or temporary test routes, they may be exported into VPNv4.

Use a route map when needed:

```text
redistribute static route-map CUSTOMER_A-STATIC-EXPORT
```

---

## Static PE-CE Routing vs Dynamic PE-CE Routing

Static PE-CE routing is simple.

```text
Advantages:
Simple configuration
No PE-CE routing protocol adjacency
Easy to understand
Useful for small sites or labs
```

But it does not scale well.

```text
Limitations:
Manual route configuration
No automatic discovery of new customer prefixes
No dynamic failure detection beyond interface/next-hop behavior
No metric exchange
More operational overhead as the number of sites grows
```

Dynamic PE-CE routing protocols solve these problems, but they introduce their own L3VPN-specific behavior.

Examples:

```text
OSPF:
Superbackbone, domain ID, down bit, domain tag, sham links.

EIGRP:
Route type preservation, metric preservation, Site of Origin.

BGP:
AS override, allowas-in, Site of Origin, multihoming behavior.
```

Static routing is a good starting point because of the different considerations needed for each dynamic routing protocol listed above.

---

## Key Points

* Static PE-CE routing uses normal static routes between the CE and PE.
* CE routers do not need MPLS, MP-BGP, VRFs, RDs, RTs, or VPN labels.
* Customer-facing PE interfaces must be assigned to the correct VRF.
* PE static routes to customer prefixes must use `ip route vrf`.
* A VRF static route is not automatically advertised to other PEs.
* Redistribute static routes under `address-family ipv4 vrf <vrf-name>` to export them into VPNv4.
* Use route maps if only selected static routes should be exported.
* CE routers need a default route or specific routes pointing to the PE.
* If traffic fails, check both directions: CE route, PE VRF route, VPNv4 route, label stack, and return route.
