# BGP PE-CE Multihoming

This page explains **PE-CE BGP multihoming** in an MPLS L3VPN.

A customer site is **multihomed** when it connects to the MPLS VPN through more than one PE router.

This improves redundancy, but it also introduces important BGP design considerations.

## Topology

This page uses the following topology:

```text
                        PE2 --- CE2
                        //       |
CE1 --- PE1 === MPLS VPN         | SITE B
  SITE A                \\       |
                        PE3 --- CE3
```

Site A is single-homed.

Site B is multihomed.

```text
Site A:
CE1 connects to PE1.

Site B:
CE2 connects to PE2.
CE3 connects to PE3.
CE2 and CE3 are part of the same customer site.
```

The MPLS VPN provider core connects PE1, PE2, and PE3.

```text
Provider AS: 65000

Site A AS:   65100
Site B AS:   65200
```

## Goal

The goal is to allow Site A and Site B to communicate through the MPLS VPN.

```text
CE1 LAN:      10.10.10.0/24
Site B LAN:   11.11.11.0/24
```

Site B should remain reachable if one PE-CE connection fails.

For example:

```text
Normal path:
CE1 -> PE1 -> MPLS VPN -> PE2 -> CE2 -> Site B

Backup path:
CE1 -> PE1 -> MPLS VPN -> PE3 -> CE3 -> Site B
```

Or the opposite, depending on BGP best path selection.

## Addressing

```text
PE1-CE1 link: 192.168.1.0/24
PE1:          192.168.1.1
CE1:          192.168.1.2

PE2-CE2 link: 192.168.2.0/24
PE2:          192.168.2.1
CE2:          192.168.2.2

PE3-CE3 link: 192.168.3.0/24
PE3:          192.168.3.1
CE3:          192.168.3.2

CE2-CE3 link: 192.168.23.0/24
CE2:          192.168.23.2
CE3:          192.168.23.3

Site A LAN:   10.10.10.0/24
Site B LAN:   11.11.11.0/24
```

## AS Numbers

```text
Provider AS: 65000

CE1 AS:      65100
CE2 AS:      65200
CE3 AS:      65200
```

CE2 and CE3 use the same AS because they belong to the same customer site.

```text
              Site B
        CE2 -------- CE3
        AS 65200     AS 65200
```

This is common in PE-CE multihoming.

## Important Design Point

Site B is one customer site.

CE2 and CE3 are not two separate sites.

They are two exit points for the same site.

That means both CE2 and CE3 can advertise the Site B prefix toward the MPLS VPN.

```text
CE2 advertises 11.11.11.0/24 to PE2.
CE3 advertises 11.11.11.0/24 to PE3.
```

PE1 can then learn two VPN paths to Site B.

```text
PE1 learns 11.11.11.0/24 through PE2.
PE1 learns 11.11.11.0/24 through PE3.
```

BGP chooses the best path.

If the best path fails, BGP can use the alternate path.

## Why Multihoming Matters

A single-homed site depends on one PE-CE connection.

```text
CE1 --- PE1
```

If the PE-CE link fails, the site is disconnected from the MPLS VPN.

A multihomed site has more than one path.

```text
PE2 --- CE2
        |
        | SITE B
        |
PE3 --- CE3
```

If the PE2-CE2 path fails, Site B can still use the PE3-CE3 path.

This provides:

```text
Redundancy
Maintenance flexibility
Optional load sharing
```

But it also creates new routing concerns.

```text
Duplicate routes
Best path selection
Unequal return paths
Possible routing loops
Route feedback into the same site
Need for filtering or Site of Origin
```

## RD and RT Design

In simple examples, every PE often uses the same RD for the VRF.

```text
rd 65000:1
```

For multihomed L3VPN sites, it is often better to use a **unique RD per PE per VRF**.

Example:

```text
PE1 RD: 65000:1
PE2 RD: 65000:2
PE3 RD: 65000:3
```

The route target stays the same for a simple setup:

```text
RT: 1:1
```

The RD and RT have different purposes.

| Value | Purpose                           |
| ----- | --------------------------------- |
| RD    | Makes VPNv4 routes unique         |
| RT    | Controls import and export policy |

Use different RDs to preserve path diversity in the provider BGP table.

Use the same RT so all PE routers import the customer routes into the same VPN.

## Why Unique RDs Matter

Assume CE2 and CE3 both advertise `11.11.11.0/24`.

PE2 creates this VPNv4 route:

```text
RD 65000:2:11.11.11.0/24
Next hop: PE2
RT: 1:1
```

PE3 creates this VPNv4 route:

```text
RD 65000:3:11.11.11.0/24
Next hop: PE3
RT: 1:1
```

These are different VPNv4 NLRIs because the RDs are different.

```text
65000:2:11.11.11.0/24
65000:3:11.11.11.0/24
```

After import into the VRF, both represent the same customer IPv4 prefix.

```text
11.11.11.0/24
```

But the provider control plane can still keep the two paths separate.

This is especially important when route reflectors are used.

A route reflector normally reflects only its best path for a given BGP NLRI.

If PE2 and PE3 use the same RD for the same prefix, the route reflector may hide one path from remote PEs.

With unique RDs, the route reflector sees two different VPNv4 routes and can reflect both.

## Route Target Design

All three PEs use the same RT.

```text
route-target export 1:1
route-target import 1:1
```

This means:

```text
Routes exported by PE1 can be imported by PE2 and PE3.
Routes exported by PE2 can be imported by PE1 and PE3.
Routes exported by PE3 can be imported by PE1 and PE2.
```

The RT controls VPN membership.

The RD does not control VPN membership.

## VRF Configuration

PE1:

```text
vrf definition CUSTOMER_A
 rd 65000:1
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
```

```text
interface Ethernet0/1
 description PE1-to-CE1
 vrf forwarding CUSTOMER_A
 ip address 192.168.1.1 255.255.255.0
 no shutdown
```

PE2:

```text
vrf definition CUSTOMER_A
 rd 65000:2
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
```

```text
interface Ethernet0/1
 description PE2-to-CE2
 vrf forwarding CUSTOMER_A
 ip address 192.168.2.1 255.255.255.0
 no shutdown
```

PE3:

```text
vrf definition CUSTOMER_A
 rd 65000:3
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
```

```text
interface Ethernet0/1
 description PE3-to-CE3
 vrf forwarding CUSTOMER_A
 ip address 192.168.3.1 255.255.255.0
 no shutdown
```

## PE-CE BGP Configuration

PE1:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.1.2 remote-as 65100
  neighbor 192.168.1.2 activate
```

PE2:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.2.2 remote-as 65200
  neighbor 192.168.2.2 activate
```

PE3:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.3.2 remote-as 65200
  neighbor 192.168.3.2 activate
```

PE2 and PE3 both peer with Site B.

```text
PE2 peers with CE2.
PE3 peers with CE3.
```

## CE Configuration

CE1:

```text
interface Ethernet0/0
 description CE1-to-PE1
 ip address 192.168.1.2 255.255.255.0
 no shutdown

interface Loopback0
 description Site-A-LAN
 ip address 10.10.10.1 255.255.255.0
 no shutdown
```

```text
router bgp 65100
 neighbor 192.168.1.1 remote-as 65000
 network 10.10.10.0 mask 255.255.255.0
```

CE2:

```text
interface Ethernet0/0
 description CE2-to-PE2
 ip address 192.168.2.2 255.255.255.0
 no shutdown

interface Ethernet0/1
 description CE2-to-CE3
 ip address 192.168.23.1 255.255.255.0
 ip ospf 1 area 0
 no shutdown

interface Loopback0
 description Site-B-LAN
 ip address 11.11.11.1 255.255.255.0
 ip ospf 1 area 0
 no shutdown
```

```text
router bgp 65200
 neighbor 192.168.2.1 remote-as 65000
 network 11.11.11.0 mask 255.255.255.0
```

CE3:

```text
interface Ethernet0/0
 description CE3-to-PE3
 ip address 192.168.3.2 255.255.255.0
 no shutdown

interface Ethernet0/1
 description CE3-to-CE2
 ip address 192.168.23.2 255.255.255.0
 ip ospf 1 area 0
 no shutdown
```

```text
router bgp 65200
 neighbor 192.168.3.1 remote-as 65000
 network 11.11.11.0 mask 255.255.255.0
```

Now both CE2 and CE3 advertise the same customer prefix to the PEs:

```text
CE2 -> PE2: 11.11.11.0/24
CE3 -> PE3: 11.11.11.0/24
```

## Optional CE2-CE3 Internal BGP

CE2 and CE3 are in the same AS.

If they exchange BGP routes internally, the CE2-CE3 session is IBGP.

CE2:

```text
router bgp 65200
 neighbor 192.168.23.2 remote-as 65200
 neighbor 192.168.23.2 next-hop-self
```

CE3:

```text
router bgp 65200
 neighbor 192.168.23.1 remote-as 65200
 neighbor 192.168.23.1 next-hop-self
```

The `next-hop-self` command is important.

CE2 may learn `10.10.10.0/24` from PE2 with next hop `192.168.2.1`.

If CE2 advertises that route to CE3 over IBGP without `next-hop-self`, CE3 may not be able to reach the next hop.

With `next-hop-self`, CE3 sees CE2 as the next hop.

```text
CE3 learns 10.10.10.0/24 from CE2.
Next hop: 192.168.23.1
```

The same applies in the opposite direction.

## Preventing Site B from Becoming Transit

Be careful not to let Site B advertise provider-learned routes back to the provider.

For example:

```text
CE2 learns 10.10.10.0/24 from PE2.
CE2 advertises it to CE3 over IBGP.
CE3 advertises it back to PE3.
```

In most cases, BGP AS path loop prevention blocks this because the provider AS, `65000`, is already in the AS path.

But do not rely only on that.

A clean design filters what the CE advertises to the PE.

On CE2:

```text
ip prefix-list SITE-B-OUT seq 10 permit 11.11.11.0/24
ip prefix-list SITE-B-OUT seq 20 deny 0.0.0.0/0 le 32
!
route-map TO-PE2 permit 10
 match ip address prefix-list SITE-B-OUT
!
router bgp 65200
 neighbor 192.168.2.1 route-map TO-PE2 out
```

On CE3:

```text
ip prefix-list SITE-B-OUT seq 10 permit 11.11.11.0/24
ip prefix-list SITE-B-OUT seq 20 deny 0.0.0.0/0 le 32
!
route-map TO-PE3 permit 10
 match ip address prefix-list SITE-B-OUT
!
router bgp 65200
 neighbor 192.168.3.1 route-map TO-PE3 out
```

Now CE2 and CE3 advertise only the Site B prefix to the MPLS VPN.

Before:

```
CE3# sh ip bgp neighbors 192.168.3.1 adv
     Network          Next Hop            Metric LocPrf Weight Path
 r>i  10.10.10.0/24    192.168.23.1             0    100      0 65000 65100 i
 *>   11.11.11.0/24    192.168.23.1            11         32768 i
 ```

After:

```
CE3# sh ip bgp neighbors 192.168.3.1 adv
     Network          Next Hop            Metric LocPrf Weight Path
 *>   11.11.11.0/24    192.168.23.1            11         32768 i

Total number of prefixes 1
```

## Control Plane Route Flow

### Site A to Site B

CE1 advertises `10.10.10.0/24` to PE1.

```text
CE1 -> PE1

Prefix:  10.10.10.0/24
AS_PATH: 65100
```

PE1 converts the route into a VPNv4 route.

```text
RD:       65000:1
Prefix:   10.10.10.0/24
RT:       1:1
Next hop: PE1
AS_PATH:  65100
```

PE1 advertises the VPNv4 route to PE2 and PE3.

PE2 imports the route into `CUSTOMER_A`.

PE3 imports the route into `CUSTOMER_A`.

Then PE2 advertises the route to CE2.

```text
PE2 -> CE2

Prefix:  10.10.10.0/24
AS_PATH: 65000 65100
```

PE3 advertises the same route to CE3.

```text
PE3 -> CE3

Prefix:  10.10.10.0/24
AS_PATH: 65000 65100
```

Site B now has two ways to reach Site A.

```text
CE2 -> PE2 -> MPLS VPN -> PE1 -> CE1
CE3 -> PE3 -> MPLS VPN -> PE1 -> CE1
```

### Site B to Site A

CE2 advertises `11.11.11.0/24` to PE2.

```text
CE2 -> PE2

Prefix:  11.11.11.0/24
AS_PATH: 65200
```

CE3 also advertises `11.11.11.0/24` to PE3.

```text
CE3 -> PE3

Prefix:  11.11.11.0/24
AS_PATH: 65200
```

PE2 converts the route into a VPNv4 route.

```text
RD:       65000:2
Prefix:   11.11.11.0/24
RT:       1:1
Next hop: PE2
AS_PATH:  65200
```

PE3 converts the route into a separate VPNv4 route.

```text
RD:       65000:3
Prefix:   11.11.11.0/24
RT:       1:1
Next hop: PE3
AS_PATH:  65200
```

If PE2 and PE3 both advertise their routes, PE1 imports both routes into the `CUSTOMER_A` VRF.

From PE1's point of view, there are now two BGP paths to the same customer prefix.

```text
11.11.11.0/24 via PE2
11.11.11.0/24 via PE3
```

In this case, PE3 actually does **not** advertise its route to PE1. Let's see why:

```
PE3# show bgp vpnv4 unicast all 11.11.11.0
...       
  Refresh Epoch 2
  65200, imported path from 65000:2:11.11.11.0/24 (global)
    4.4.4.4 (metric 21) (via default) from 4.4.4.4 (4.4.4.4)
      Origin IGP, metric 0, localpref 100, valid, internal, best
      Extended Community: RT:1:1
      mpls labels in/out nolabel/23
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 19 2026 00:29:51 UTC
  Refresh Epoch 2
  65200
    192.168.3.2 (via vrf CUSTOMER_A) from 192.168.3.2 (192.168.23.3)
      Origin IGP, metric 11, localpref 100, valid, external
      Extended Community: RT:3:3
      rx pathid: 0, tx pathid: 0
      Updated on Jun 19 2026 00:44:26 UTC
```

The key is `metric 0` and `metric 11`.

11.11.11.0/24 actually resides on CE2 (Loopback0). It advertises a BGP route to PE2 with MED 0.

CE3 learns 11.11.11.0/24 from CE2 via OSPF with a metric of 11.

CE3 then advertises that metric of 11 as `MED = 11` in its advertisement to PE3.

As a result, PE3 sees PE2's route as superior to its own route, and doesn't advertise the route it learned from CE3 to PE1.

```
PE3# show bgp vpnv4 unicast all           
...
Route Distinguisher: 65000:3 (default for vrf CUSTOMER_A)
 *>i  10.10.10.0/24    1.1.1.1                  0    100      0 65100 i
 *>i  11.11.11.0/24    4.4.4.4                  0    100      0 65200 i
 *                     192.168.3.2             11             0 65200 i
 ```

 The `>` next to the route learned from 4.4.4.4 indicates that it is the best route.

 Here is the BGP best-path selection order up to MED:

```
1) Highest Weight
2) Highest LOCAL_PREF
3) Locally injected
4) Shortest AS_PATH
5) Lowest ORIGIN (i=0 > e=1 > ?=2)
6) Lowest MED
```

This serves as a good reminder of the BGP best-path selection process, but I want PE2 and PE3 to both advertise a route to PE1, 
so let's make the, advertise the CE2-CE3 link as well.

CE2 and CE3:

```
ip prefix-list SITE-B-OUT seq 15 permit 192.168.23.0/24
!
router bgp 65200
 network 192.168.23.0
```

---

## Verification on PE1

Check the VRF BGP table.

```text
      Updated on Jun 19 2026 01:08:39 UTC
PE1#show bgp vpnv4 unicast all             
BGP table version is 24, local router ID is 1.1.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:1 (default for vrf CUSTOMER_A)
 *>   10.10.10.0/24    192.168.1.2              0             0 65100 i
 *>i  11.11.11.0/24    4.4.4.4                  0    100      0 65200 i
 * i  192.168.23.0     6.6.6.6                  0    100      0 65200 i
 *>i                   4.4.4.4                  0    100      0 65200 i
Route Distinguisher: 65000:2
 *>i  11.11.11.0/24    4.4.4.4                  0    100      0 65200 i
 *>i  192.168.23.0     4.4.4.4                  0    100      0 65200 i
Route Distinguisher: 65000:3
 *>i  192.168.23.0     6.6.6.6                  0    100      0 65200 i
```

PE1 receives routes to 192.168.23.0/24 from both PE2 and PE3.

Under RD 65000:1 (VRF CUSTOMER_A), the `>` next to PE2's route mean that it was selected as the best route.

We can confirm in the CUSTOMER_A routing table:

```
PE1#show ip route vrf CUSTOMER_A | i 192.168.23.0
B     192.168.23.0/24 [200/0] via 4.4.4.4, 00:05:35
```

In this example, PE1 selected PE2 as the best path due to its **lower BGP RID**:

```
BGP best-path selection:
1) Highest Weight
2) Highest LOCAL_PREF
3) Locally injected
4) Shortest AS_PATH
5) Lowest ORIGIN (i=0 > e=1 > ?=2)
6) Lowest MED
7) EBGP over IBGP
8) Lowest IGP metric to NEXT_HOP
9) Oldest EBGP path
10) Lowest neighbor RID
11) Shortest CLUSTER_LIST
12) Lowest neighbor IP
```

## What CE1 Sees

CE1 does not see the provider core.

CE1 sees a normal EBGP route from PE1.

```text
CE1# show ip bgp 192.168.23.0
BGP routing table entry for 192.168.23.0/24, version 16
Paths: (1 available, best #1, table default)
  Not advertised to any peer
  Refresh Epoch 3
  65000 65200
    192.168.1.1 from 192.168.1.1 (1.1.1.1)
      Origin IGP, localpref 100, valid, external, best
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 19 2026 01:05:04 UTC
```

CE1 does not know whether PE1 is using PE2 or PE3 as the egress PE.

From CE1's point of view, the next hop is simply PE1.

```text
Next hop: 192.168.1.1
```

## Manipulating Local Preference

If you want to control which egress PE is selected as the next hop by PE1, manipulate BGP attributes.

A common method is **local preference**.

Local preference is carried inside the provider AS.

Higher local preference is better.

Example design:

```text
PE2 path to Site B: local preference 200
PE3 path to Site B: local preference 100 (default)
```

PE1 prefers the path through PE2.

If the PE2 path fails, PE1 uses the PE3 path.

> In this case PE1 already prefers PE2 due to its lower RID, but in most networks it is best to control the route selection
> in a more deterministic way, earlier in the BGP best-path selection process.

### Local Preference Configuration

PE2:

Set higher local preference on routes learned from CE2.

```text
route-map CE2-IN permit 10
 set local-preference 200
!
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.2.2 route-map CE2-IN in
```

Local preference provides more control compared to the RID
(which is simply an identifier and usually not configured with route preference in mind).

## Customer-Controlled Path Preference with MED

The customer can send MED from CE2 and CE3.

I've removed the local preference configuration. Now let's configure the customer side to cause
the service provider to prefer CE3 over CE2.

Example:

```text
CE2 advertises 192.168.23.0/24 with MED 100.
CE3 advertises 192.168.23.0/24 with MED 50.
```

PE1 will prefer the path via PE3, unless the provider has other policy configured (e.g., local preference).

CE2:

```text
route-map TO-PE2 permit 10
 match ip address prefix-list SITE-B-OUT
 set metric 100
!
router bgp 65200
 neighbor 192.168.2.1 route-map TO-PE2 out
```

CE3:

```text
route-map TO-PE3 permit 10
 match ip address prefix-list SITE-B-OUT
 set metric 50
!
router bgp 65200
 neighbor 192.168.3.1 route-map TO-PE3 out
```

MED is lower in the BGP best-path algorithm than local preference.

If the provider sets local preference, local preference wins.

## Load Sharing

By default, BGP installs only one best path.

If you want PE1 to install both paths to Site B, configure BGP multipath.

In this topology, PE1 receives two IBGP VPN paths to `192.168.23.0/24`.

```text
PE1 -> PE2
PE1 -> PE3
```

> I removed the MED configurations to make the routes equal again.

Configure multipath under the VRF address family on PE1.

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  maximum-paths ibgp 2
```

> `maximum-paths 2` would not work, as it applies only to EBGP paths.

After this, PE1 can install two equal-cost BGP paths into the VRF routing table, if the paths are eligible for multipath.

```text
PE1# show ip route vrf CUSTOMER_A
B     192.168.23.0/24 [200/0] via 6.6.6.6, 00:00:05
                      [200/0] via 4.4.4.4, 00:00:05
```

Now PE1 can load-share traffic across both egress PEs.

### Multipath Requirements

BGP multipath does not mean "install any two paths."

The paths must be indentical up to **step 8** of the best-path algorithm:

```
BGP best-path selection:
1) Highest Weight
2) Highest LOCAL_PREF
3) Locally injected
4) Shortest AS_PATH
5) Lowest ORIGIN (i=0 > e=1 > ?=2)
6) Lowest MED
7) EBGP over IBGP
8) Lowest IGP metric to NEXT_HOP ← MUST MATCH UP TO HERE
9) Oldest EBGP path
10) Lowest neighbor RID
11) Shortest CLUSTER_LIST
12) Lowest neighbor IP
```

For example, I modified P2's E0/2 interface (leading to PE3) to have a higher OSPF cost:

```
                        PE2 E0/2--- CE2
                        //           |
CE1 --- PE1 == P1 === P2             |
                     E0/2            | SITE B
  SITE A                \\           |
                        PE3 E0/2--- CE3

P2(config)#int e0/2
P2(config-if)#ip ospf cost 100
```

PE1 no longer considers the PE3 route eligible for multipath because it now has a higher OSPF metric
to reach the next hop (PE3's loopback):

```
PE1(config-router-af)#do sh ip route vrf CUSTOMER_A
B     192.168.23.0/24 [200/0] via 4.4.4.4, 00:02:42
```

### Multipath and Unique RDs

Unique RDs are important for VPN path diversity.

The RD is part of the VPNv4 prefix. For example, these are two different VPNv4 NLRIs:

```text
65000:2:192.168.23.0/24
65000:3:192.168.23.0/24
```

But these are the same VPNv4 NLRI with two possible paths:

```text
65000:2:192.168.23.0/24 via PE2
65000:2:192.168.23.0/24 via PE3
```

This distinction matters because BGP selects one best path per NLRI by default.

If PE2 and PE3 use different RDs, PE1 receives two different VPNv4 routes. Both routes can be imported into the VRF, and the VRF can then install both next hops if BGP multipath is configured and the paths are otherwise equal.

If PE2 and PE3 use the same RD, they advertise the same VPNv4 route. BGP chooses one best path for that VPNv4 NLRI before the route is imported into the VRF. As a result, PE1 may have only one usable imported path for the customer IPv4 prefix, so `maximum-paths` cannot install both next hops in the VRF routing table.

I removed the previous OSPF cost configuration from P2 E0/2, so PE1 once again installs both routes:

```text
PE1(config-router-af)#do show ip route vrf CUSTOMER_A bgp
B     192.168.23.0/24 [200/0] via 6.6.6.6, 00:00:01
                      [200/0] via 4.4.4.4, 00:00:01
```

But I then changed PE3's RD to match PE2's:

```text
PE3(config)# vrf definition CUSTOMER_A
PE3(config-vrf)# no rd 65000:3
PE3(config-vrf)# rd 65000:2
PE3(config-vrf)# router bgp 65000
PE3(config-router)# address-family ipv4 vrf CUSTOMER_A
PE3(config-router-af)# neighbor 192.168.3.2 remote-as 65200
PE3(config-router-af)# neighbor 192.168.3.2 activate
```

> You have to re-configure BGP neighbors in a VRF after deleting the VRF's RD.

PE1 can no longer install both routes:

```text
PE1# show bgp vpnv4 unicast all            
...
Route Distinguisher: 65000:2
 *>i  11.11.11.0/24    4.4.4.4                  0    100      0 65200 i
 * i  192.168.23.0     6.6.6.6                  0    100      0 65200 i
 *>i                   4.4.4.4                  0    100      0 65200 i

PE1# show ip route vrf CUSTOMER_A bgp | include 192.168.23
B     192.168.23.0/24 [200/0] via 4.4.4.4, 00:01:07
```

For load sharing to work in a multihomed MPLS L3VPN, PE2 and PE3 should use unique RDs.

```text
PE2:
 rd 65000:2

PE3:
 rd 65000:3
```

In short:

```text
Different RDs preserve multiple VPNv4 routes.
The same RT imports those routes into the same VRF.
BGP multipath can then install multiple equal paths in the VRF routing table.
```

In another page we will look at how to enable multipath if RDs match using `import path` commands.

---

## Return Path Considerations

Multihoming can create asymmetric routing.

Example:

```text
Forward path:
CE1 -> PE1 -> PE2 -> CE2

Return path:
CE2 -> PE2 -> PE1 -> CE1
```

This is symmetric.

But this can also happen:

```text
Forward path:
CE1 -> PE1 -> PE2 -> CE2

Return path:
CE3 -> PE3 -> PE1 -> CE1
```

This is asymmetric.

Asymmetric routing is not automatically wrong.

But it can matter if the customer site has:

```text
Stateful firewalls
NAT
Traffic inspection
Policy-based routing
Strict security policy
Application path sensitivity
```

If symmetry is required, coordinate the provider and customer routing policies.

## Route Feedback Problem

Multihoming can cause a route to be advertised back into the site that originated it.

Example:

```text
1. CE2 advertises 11.11.11.0/24 to PE2.

2. PE2 advertises the route through the MPLS VPN.

3. PE3 imports the route into the CUSTOMER_A VRF.

4. PE3 advertises the route to CE3.

5. CE3 is in the same site as CE2.
```

This means Site B may receive its own route back through the MPLS VPN.

In this specific design, CE3 is in AS 65200.

The route originated from CE2 also has AS 65200 in the AS_PATH.

When PE3 advertises the route to CE3, CE3 may reject it because its own AS appears in the AS_PATH.

That is normal BGP AS_PATH loop prevention.

However, do not rely only on AS_PATH loop prevention in all designs.

It may not protect you if:

```text
CE2 and CE3 use different AS numbers
AS-Override is configured
Allowas-in is configured
The customer redistributes between protocols
The site has backdoor links
The provider changes AS_PATH policy
```

This is why Site of Origin is important.

## Site of Origin Preview

Site of Origin, or SoO, is a BGP extended community.

It identifies the site where a route came from.

In this topology, Site B should use the same SoO value on both PE-CE links.

```text
PE2-to-CE2 SoO: 65000:200
PE3-to-CE3 SoO: 65000:200
```

Site A would use a different SoO value.

```text
PE1-to-CE1 SoO: 65000:100
```

The idea is:

```text
If a route came from Site B,
do not advertise it back to Site B.
```

SoO is covered in detail in another page.

For this page, remember:

```text
All PE-CE links connected to the same customer site should use the same SoO value.
Different customer sites should use different SoO values.
```

## Why SoO Is Important with AS-Override and Allowas-in

The previous page explained AS-Override and Allowas-in.

Those features work around the normal AS_PATH loop prevention.

That can be necessary when multiple sites use the same customer AS.

But in a multihomed design, it can be dangerous.

Example:

```text
CE2 advertises 11.11.11.0/24 to PE2.
PE2 sends the route through the MPLS VPN.
PE3 imports the route.
PE3 advertises it to CE3.
```

Without AS-Override, CE3 may reject the route because AS 65200 appears in the AS_PATH.

With AS-Override, PE3 can rewrite AS 65200 to AS 65000.

CE3 may then accept the route.

That means Site B can relearn its own route through the MPLS VPN.

This is the kind of problem SoO is designed to prevent.

## Key Points

* PE-CE multihoming connects one customer site to multiple PE routers.
* Site B uses CE2 and CE3 as two exits into the MPLS VPN.
* CE2 and CE3 can both advertise `192.168.23.0/24`.
* PE1 can learn two VPN paths to Site B.
* BGP chooses one best path by default.
* Use local preference for primary and backup path control inside the provider AS.
* Use MED if the customer needs to influence inbound path selection.
* Use `maximum-paths ibgp` if you want PE1 to install multiple VPN paths.
* Use unique RDs per PE per VRF to preserve VPN path diversity.
* Use the same RT so all PEs participate in the same VPN.
* Filter CE-to-PE advertisements to prevent the customer site from becoming a transit path.
* Multihoming can cause route feedback into the same site.
* AS_PATH loop prevention may help, but it is not enough in all designs.
* Site of Origin is the main MPLS VPN tool for preventing a multihomed site from relearning its own routes.
