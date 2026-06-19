# BGP AS-Override and Allowas-in

This page explains **BGP AS-Override** and **Allowas-in** in an MPLS L3VPN PE-CE design.

These features are used when multiple customer sites use the **same BGP AS number**.

## The Problem

In the basic PE-CE BGP example, each site used a different customer AS.

```text
CE1 AS 65100
CE2 AS 65200
Provider AS 65000
```

That design works normally.

But some customers use the same AS at every site.

```text
CE1 AS 65100
CE2 AS 65100
Provider AS 65000
```

This creates a BGP AS_PATH loop prevention problem.

## Topology

```text
             MPLS Provider Core
        MP-BGP VPNv4 + MPLS forwarding

        PE1 ================= PE2
         |                     |
      eBGP PE-CE           eBGP PE-CE
         |                     |
        CE1                   CE2
     AS 65100              AS 65100
         |                     |
   10.10.10.0/24         11.11.11.0/24
```

## Addressing

```text
PE1-CE1 link: 192.168.1.0/24
PE1:          192.168.1.1
CE1:          192.168.1.2

PE2-CE2 link: 192.168.2.0/24
PE2:          192.168.2.1
CE2:          192.168.0.2

CE1 LAN:      10.10.10.0/24
CE2 LAN:      11.11.11.0/24
```

## AS Numbers

```text
Provider AS: 65000

CE1 AS:      65100
CE2 AS:      65100
```

Both CE routers use the same customer AS.

This is common in MPLS VPN environments. The customer may want all sites to use one private AS number.

## Baseline PE-CE BGP Configuration

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
  neighbor 192.168.2.2 remote-as 65100
  neighbor 192.168.2.2 activate
```

CE1:

```text
router bgp 65100
 neighbor 192.168.1.1 remote-as 65000
 network 10.10.10.0 mask 255.255.255.0
```

CE2:

```text
router bgp 65100
 neighbor 192.168.2.1 remote-as 65000
 network 11.11.11.0 mask 255.255.255.0
```

The BGP neighbors come up.

The local PE learns the local CE route.

The remote PE receives the VPNv4 route.

But the remote CE rejects the route.

## Why the Route Is Rejected

CE1 advertises `10.10.10.0/24` to PE1.

```text
CE1 -> PE1

Prefix:  10.10.10.0/24
AS_PATH: 65100
```

PE1 converts the route into a VPNv4 route and advertises it to PE2 with MP-BGP.

```text
PE1 -> PE2

Prefix:  10.10.10.0/24
AS_PATH: 65100
```

PE2 advertises the route to CE2 using eBGP.

Because PE2 is in AS 65000, it prepends AS 65000.

```text
PE2 -> CE2

Prefix:  10.10.10.0/24
AS_PATH: 65000 65100
```

CE2 is also in AS 65100.

When CE2 receives the route, it sees its own AS in the AS_PATH.

```text
CE2 local AS: 65100
Received AS_PATH: 65000 65100
```

By default, BGP rejects any route that contains the local AS in the AS_PATH.

This is normal BGP loop prevention.

CE2 assumes:

```text
My own AS is already in the path.
This route must have looped back to me.
Reject it.
```

The same problem happens in the opposite direction.

CE2 advertises `11.11.11.0/24`.

CE1 rejects it because the AS_PATH contains `65100`.

## What This Looks Like

On PE2, the route exists in the VRF BGP table.

```text
PE2# show bgp vrf CUSTOMER_A vpnv4 unicast 10.10.10.0/24
BGP routing table entry for 65000:1:10.10.10.0/24, version 6
Paths: (1 available, best #1, table CUSTOMER_A)
  Advertised to update-groups:
     3         
  Refresh Epoch 1
  65100
    1.1.1.1 (metric 31) (via default) from 1.1.1.1 (1.1.1.1)
      Origin IGP, metric 0, localpref 100, valid, internal, best
      Extended Community: RT:1:1
      mpls labels in/out nolabel/21
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 18 2026 22:12:56 UTC
```

PE2 has the route and advertises it to CE2:

```
PE2#s how bgp vpnv4 unicast all neighbors 192.168.2.2 advertised-routes 
BGP table version is 8, local router ID is 4.4.4.4
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:1 (default for vrf CUSTOMER_A)
 *>i  10.10.10.0/24    1.1.1.1                  0    100      0 65100 i

Total number of prefixes 1 
```

But CE2 does not install it.

```text
CE2# show ip route 10.10.10.0   
% Network not in table
```

With `debug ip bgp updates`, you can see why CE2 rejected the route:
```text
*Jun 18 23:00:44.962: BGP(0): 192.168.2.1 rcv UPDATE about 10.10.10.0/24 -- DENIED due to: AS-PATH contains our own AS;
```

The route is rejected because the AS_PATH contains CE2's own AS.

## Two Solutions

There are two common solutions:

```text
1. AS-Override
2. Allowas-in
```

They solve the same problem in different places.

## AS-Override

**AS-Override** is configured on the PE router.

It tells the PE:

```text
When advertising routes to this CE,
if the CE's AS appears in the AS_PATH,
replace that AS with my own AS.
```

In this example, PE2 advertises CE1's route to CE2.

Without AS-Override:

```text
PE2 -> CE2

Prefix:  10.10.10.0/24
AS_PATH: 65000 65100
```

CE2 rejects the route because `65100` is in the AS_PATH.

With AS-Override:

```text
PE2 replaces 65100 with 65000.
```

The route sent to CE2 looks like this:

```text
PE2 -> CE2

Prefix:  10.10.10.0/24
AS_PATH: 65000 65000
```

CE2 accepts the route because its own AS, `65100`, is no longer in the AS_PATH.

### AS-Override Configuration

AS-Override is configured on the PE under the VRF address family.

PE1:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.1.2 remote-as 65100
  neighbor 192.168.1.2 activate
  neighbor 192.168.1.2 as-override
```

PE2: 

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.2.2 remote-as 65100
  neighbor 192.168.2.2 activate
  neighbor 192.168.2.2 as-override
```

Configure AS-Override on both PE-CE neighbors to allow traffic to work in both directions.

```text
PE1 needs AS-Override when advertising CE2's routes to CE1.
PE2 needs AS-Override when advertising CE1's routes to CE2.
```

### AS-Override Control Plane

The route from CE1 to CE2 works like this.

```text
CE1 advertises 10.10.10.0/24 to PE1.

AS_PATH: 65100
```

PE1 advertises the VPNv4 route to PE2.

```text
AS_PATH: 65100
```

PE2 prepares to advertise the route to CE2.

CE2's AS is 65100.

PE2 sees 65100 in the AS_PATH.

AS-Override replaces 65100 with 65000.

```text
Before AS-Override:

AS_PATH: 65100

After AS-Override:

AS_PATH: 65000
```

Then PE2 sends the route to CE2 using eBGP.

Normal eBGP AS prepending adds PE2's AS, 65000.

```text
Final AS_PATH received by CE2:

65000 65000
```

CE2 accepts the route.

### Verification with AS-Override

With `debug ip bgp updates` debug activated, CE2 shows the received route:

```
*Jun 18 23:15:23.640: BGP(0): 192.168.2.1 rcvd UPDATE w/ attr: nexthop 192.168.2.1, origin i, merged path 65000 65000, AS_PATH 
```

And here it is in the BGP table:

```
CE2#show ip bgp
BGP table version is 3, local router ID is 11.11.11.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>   10.10.10.0/24    192.168.2.1                            0 65000 65000 i
 *>   11.11.11.0/24    0.0.0.0                  0         32768 i
 ```

CE2 accepts the route because its own AS is not in the AS_PATH, and installs it in the routing table:

```
CE2#show ip route | i 10.10.10.0
B        10.10.10.0 [20/0] via 192.168.2.1, 00:02:32
```

CE1 can ping CE2:

```
CE1#ping 11.11.11.1 source l0 
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 11.11.11.1, timeout is 2 seconds:
Packet sent with a source address of 10.10.10.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/5 ms
```

## Allowas-in

**Allowas-in** is configured on the receiving router.

It tells BGP:

```text
Accept routes from this neighbor even if my own AS appears in the AS_PATH.
```

In this example, PE2 sends CE1's route to CE2.

Without AS-Override, the route looks like this:

```text
PE2 -> CE2

Prefix:  10.10.10.0/24
AS_PATH: 65000 65100
```

CE2 normally rejects it because `65100` is its own AS.

With `allowas-in`, CE2 accepts it anyway.

### Allowas-in Configuration

Allowas-in is configured on the CE routers.

CE1:

```text
router bgp 65100
 neighbor 192.168.10.1 remote-as 65000
 neighbor 192.168.10.1 allowas-in
 network 10.10.10.0 mask 255.255.255.0
```

CE2:

```text
router bgp 65100
 neighbor 192.168.20.1 remote-as 65000
 neighbor 192.168.20.1 allowas-in
 network 11.11.11.0 mask 255.255.255.0
```

Here is the syntax:

```text
neighbor <neighbor-address> allowas-in [<number>]
```

The optional <number> controls how many times the local AS is allowed to appear in the received AS_PATH.

> If you don't specify a `<number>`, the default is `3`.

Example:

```text
neighbor 192.168.20.1 allowas-in 1
```

This means:

```text
Accept routes from 192.168.20.1 if my own AS appears in the AS_PATH up to 1 time.
```

Use the lowest value that solves the problem, or just use the default of `3`.

### Allowas-in Control Plane

The route from CE1 to CE2 works like this.

```text
CE1 advertises 10.10.10.0/24 to PE1.

AS_PATH: 65100
```

PE1 advertises the VPNv4 route to PE2.

```text
AS_PATH: 65100
```

PE2 advertises the route to CE2 using eBGP.

PE2 prepends provider AS 65000.

```text
AS_PATH: 65000 65100
```

CE2 receives the route.

Normally, CE2 rejects it because its own AS appears in the path.

```text
CE2 local AS: 65100
Received AS_PATH: 65000 65100
```

With Allowas-in, CE2 accepts the route.

## AS-Override vs Allowas-in

Both features solve the same basic problem.

They do it in different places.

| Feature     | Configured On | AS_PATH Modified? |
| ----------- | ------------: | ----------------: |
| `as-override` |            PE |               Yes |
| `allowas-in`  |            CE |                No |

## AS-Override Behavior

AS-Override changes the AS_PATH before sending the route to the CE.

```text
Before:

65000 65100

After:

65000 65000
```

The CE does not see its own AS.

The CE accepts the route normally.

## Allowas-in Behavior

Allowas-in does not change the AS_PATH.

```text
Received:

65000 65100
```

The CE still sees its own AS.

But the CE is configured to accept the route anyway.

## Which One Should You Use?

Both of these features solve the same problem.

In a real service provider environment, AS-Override is often cleaner for the customer.

The CE routers do not need any special configuration.

However, from a CCIE lab perspective, you should know both.

## Important Loop Prevention Warning

AS-Override and Allowas-in both weaken normal BGP AS_PATH loop prevention.

That does not mean they always create loops.

But they make loops easier to create in more complex topologies.

Be careful with:

```text
Dual-homed sites
Backdoor links
CE-to-CE links outside the MPLS VPN
Multiple PEs connected to the same customer site
Redistribution between BGP and an IGP
Mutual redistribution
```

For multihomed PE-CE designs, use additional loop-prevention tools.

Common tools include:

```text
Site of Origin
Route filtering
Prefix filtering
AS_PATH filtering
Communities
Local preference
MED
Administrative distance control
```

Site of Origin is especially important in MPLS VPN designs.

## Same AS Without a Backdoor Link

This topology is simple.

```text
CE1 -- PE1 == MPLS VPN == PE2 -- CE2
```

There is no CE-to-CE backdoor path.

AS-Override or Allowas-in can safely solve the same-AS issue.

## Same AS With a Backdoor Link

This topology is more dangerous.

```text
             MPLS VPN
        PE1 ========= PE2
         |             |
        CE1-----------CE2
          backdoor link
```

CE1 and CE2 are connected through the MPLS VPN and through a customer-controlled backdoor link.

If routes are exchanged in both directions and loop prevention is weakened, a route can come back to the site that originally advertised it.

This is where Site of Origin becomes important.

Example:

```text
CE1 advertises 10.10.10.0/24 to PE1.
PE1 advertises it through the MPLS VPN to PE2.
PE2 advertises it to CE2.
CE2 advertises it back to CE1 through the backdoor link.
```

## AS-Override with Route Filtering

You can also apply outbound route filtering toward the CE.

Example on PE2:

```text
ip prefix-list CE2-OUT seq 10 permit 10.10.10.0/24
ip prefix-list CE2-OUT seq 20 deny 0.0.0.0/0 le 32

route-map TO-CE2 permit 10
 match ip address prefix-list CE2-OUT
```

Apply it under the VRF address family.

```text
router bgp 65000
 !
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.20.2 remote-as 65100
  neighbor 192.168.20.2 activate
  neighbor 192.168.20.2 as-override
  neighbor 192.168.20.2 route-map TO-CE2 out
 exit-address-family
```

This makes the behavior more controlled.

PE2 advertises only the intended routes to CE2.

## Allowas-in with Route Filtering

Allowas-in should usually be combined with filtering.

Example on CE2:

```text
ip prefix-list FROM-MPLS seq 10 permit 10.10.10.0/24
ip prefix-list FROM-MPLS seq 20 deny 0.0.0.0/0 le 32

route-map FROM-PE2 permit 10
 match ip address prefix-list FROM-MPLS
```

Apply it inbound from PE2.

```text
router bgp 65100
 neighbor 192.168.20.1 remote-as 65000
 neighbor 192.168.20.1 allowas-in
 neighbor 192.168.20.1 route-map FROM-PE2 in
 network 11.11.11.0 mask 255.255.255.0
```

Now CE2 allows its own AS in the AS_PATH, but only for the expected prefixes.

This is safer than allowing everything from the PE.

---

## Full AS-Override Example

PE1:

```text
vrf definition CUSTOMER_A
 rd 65000:1
 route-target export 1:1
 route-target import 1"1
 !
 address-family ipv4
 exit-address-family

interface Ethernet0/1
 description PE1-to-CE1
 vrf forwarding CUSTOMER_A
 ip address 192.168.1.1 255.255.255.0
 no shutdown

router bgp 65000
 !
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.1.2 remote-as 65100
  neighbor 192.168.1.2 activate
  neighbor 192.168.1.2 as-override
 exit-address-family
```

PE2:

```text
vrf definition CUSTOMER_A
 rd 65000:1
 route-target export 65000:1
 route-target import 65000:1
 !
 address-family ipv4
 exit-address-family

interface GigabitEthernet0/1
 description PE2-to-CE2
 vrf forwarding CUSTOMER_A
 ip address 192.168.2.1 255.255.255.0
 no shutdown

router bgp 65000
 !
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.2.2 remote-as 65100
  neighbor 192.168.2.2 activate
  neighbor 192.168.2.2 as-override
 exit-address-family
```

CE1:

```text
interface Ethernet0/0
 description CE1-to-PE1
 ip address 192.168.1.2 255.255.255.0
 no shutdown

interface Loopback0
 description CE1-LAN
 ip address 10.10.10.1 255.255.255.0
 no shutdown

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

interface Loopback0
 description CE2-LAN
 ip address 11.11.11.1 255.255.255.0
 no shutdown

router bgp 65100
 neighbor 192.168.2.1 remote-as 65000
 network 11.11.11.0 mask 255.255.255.0
```

With AS-Override, the CE routers do not need Allowas-in.

## Full Allowas-in Example

PE1:

```text
vrf definition CUSTOMER_A
 rd 65000:1
 route-target export 1:1
 route-target import 1"1
 !
 address-family ipv4
 exit-address-family

interface Ethernet0/1
 description PE1-to-CE1
 vrf forwarding CUSTOMER_A
 ip address 192.168.1.1 255.255.255.0
 no shutdown

router bgp 65000
 !
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.1.2 remote-as 65100
  neighbor 192.168.1.2 activate
 exit-address-family
```

PE2:

```text
vrf definition CUSTOMER_A
 rd 65000:1
 route-target export 65000:1
 route-target import 65000:1
 !
 address-family ipv4
 exit-address-family

interface GigabitEthernet0/1
 description PE2-to-CE2
 vrf forwarding CUSTOMER_A
 ip address 192.168.2.1 255.255.255.0
 no shutdown

router bgp 65000
 !
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.2.2 remote-as 65100
  neighbor 192.168.2.2 activate
 exit-address-family
```

CE1:

```text
interface Ethernet0/0
 description CE1-to-PE1
 ip address 192.168.1.2 255.255.255.0
 no shutdown

interface Loopback0
 description CE1-LAN
 ip address 10.10.10.1 255.255.255.0
 no shutdown

router bgp 65100
 neighbor 192.168.1.1 remote-as 65000
 neighbor 192.168.1.1 allowas-in
 network 10.10.10.0 mask 255.255.255.0
```

CE2:

```text
interface Ethernet0/0
 description CE2-to-PE2
 ip address 192.168.2.2 255.255.255.0
 no shutdown

interface Loopback0
 description CE2-LAN
 ip address 11.11.11.1 255.255.255.0
 no shutdown

router bgp 65100
 neighbor 192.168.2.1 remote-as 65000
 neighbor 192.168.2.1 allowas-in
 network 11.11.11.0 mask 255.255.255.0
```

With Allowas-in, the PE routers do not need AS-Override.

## Common Mistakes

### Mistake 1: Configuring AS-Override on the CE

Incorrect:

```text
CE2(config)# router bgp 65100
CE2(config-router)# neighbor 192.168.2.1 as-override
```

AS-Override is configured on the PE toward the CE.

Correct:

```text
PE2(config)# router bgp 65000
PE2(config-router)# address-family ipv4 vrf CUSTOMER_A
PE2(config-router-af)# neighbor 192.168.2.2 as-override
```

### Mistake 2: Configuring Allowas-in on the PE

Incorrect for this problem:

```text
PE2(config)# router bgp 65000
PE2(config-router-af)# neighbor 192.168.2.2 allowas-in
```

The rejecting router is the CE.

Configure Allowas-in on the CE that receives the route containing its own AS.

Correct:

```text
CE2(config)# router bgp 65100
CE2(config-router)# neighbor 192.168.2.1 allowas-in
```

### Mistake 3: Configuring Only One Direction

If only PE2 has AS-Override, only CE2 can learn CE1's route.

```text
PE2 -> CE2 works.
PE1 -> CE1 fails.
```

For bidirectional reachability, configure both directions.

```text
PE1 neighbor CE1 as-override
PE2 neighbor CE2 as-override
```

Or:

```text
CE1 neighbor PE1 allowas-in
CE2 neighbor PE2 allowas-in
```

### Mistake 4: Ignoring Backdoor Links

AS-Override and Allowas-in solve the same-AS issue.

They do not automatically solve routing loops in complex topologies.

If the customer has backdoor links or multihomed sites, add loop-prevention policy, such as SoO.

---

## Key Points

* BGP rejects routes that contain the local AS in the AS_PATH.
* Same-AS customer sites commonly trigger this problem in MPLS L3VPN PE-CE BGP.
* AS-Override is configured on the PE.
* AS-Override rewrites the customer's AS in the AS_PATH before advertising the route to the CE.
* Allowas-in is configured on the receiving CE.
* Allowas-in allows the CE to accept routes that contain its own AS in the AS_PATH.
* AS-Override modifies the AS_PATH.
* Allowas-in does not modify the AS_PATH.
* Use route filtering and Site of Origin in more complex designs to avoid loops.
* Be especially careful when same-AS sites also have backdoor links or multihoming.
