# BGP Import Path

This page explains the BGP **import path** feature in an MPLS L3VPN.

The feature is useful when a PE receives multiple VPNv4 paths for the same VPNv4 NLRI and you want a VRF to import more than one of those paths.

This commonly comes up in PE-CE multihoming designs.

## Topology

This page uses the same topology as the PE-CE BGP multihoming page.

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

The important prefixes are:

```text
Site A LAN:      10.10.10.0/24
Site B LAN:      11.11.11.0/24
CE2-CE3 link:    192.168.23.0/24
```

The provider AS is:

```text
Provider AS: 65000
```

The customer AS numbers are:

```text
CE1 AS: 65100
CE2 AS: 65200
CE3 AS: 65200
```

## Why This Feature Exists

In MPLS L3VPN, a PE receives VPNv4 routes from other PE routers.

Those VPNv4 routes are then imported into local VRFs based on route target policy.

The normal flow is:

```text
VPNv4 table -> VRF BGP table -> VRF routing table
```

The import path feature affects this step:

```text
VPNv4 table -> VRF BGP table
```

It does not directly install routes in the routing table.

It controls which VPNv4 paths are imported into the VRF BGP table.

Then `maximum-paths` controls whether multiple eligible imported paths can be installed in the VRF routing table.

## Normal VPNv4 Import

A VPNv4 route includes:

```text
RD
IPv4 prefix
RT
BGP next hop
VPN label
BGP attributes
```

Example:

```text
RD:       65000:2
Prefix:   192.168.23.0/24
RT:       1:1
Next hop: 4.4.4.4
AS_PATH:  65200
```

A VRF imports a VPNv4 route only if the route target matches the VRF's import policy.

Example:

```text
vrf definition CUSTOMER_A
 rd 65000:1
 address-family ipv4
  route-target import 1:1
  route-target export 1:1
```

This VRF imports VPNv4 routes with RT `1:1`.

```text
Route has RT 1:1 -> eligible for import
Route has RT 2:2 -> not eligible for import
```

## Default Behavior

By default, the VRF imports only the best available path for a given prefix.

Example:

```text
VPNv4 table:
65000:2:192.168.23.0/24 via 4.4.4.4
65000:2:192.168.23.0/24 via 6.6.6.6

VRF table:
192.168.23.0/24 via 4.4.4.4 only
```

Even if PE1 has two VPNv4 paths, the VRF only imports of them.

If only one path is imported into the VRF, `maximum-paths` cannot install both next hops.

`maximum-paths` can only install paths that already exist in the VRF BGP table.

## Unique RDs vs Same RD

The preferred MPLS L3VPN multihoming design is to use unique RDs on different PEs.

```text
PE2 RD: 65000:2
PE3 RD: 65000:3
```

With unique RDs, PE2 and PE3 create different VPNv4 NLRIs.

```text
65000:2:192.168.23.0/24
65000:3:192.168.23.0/24
```

PE1 can receive both VPNv4 routes and import both into the `CUSTOMER_A` VRF.

Then BGP multipath can install both next hops in the VRF routing table.

```text
B 192.168.23.0/24 [200/0] via 4.4.4.4
                      [200/0] via 6.6.6.6
```

But in some designs, PE2 and PE3 may use the same RD.

```text
PE2 RD: 65000:2
PE3 RD: 65000:2
```

In that case, PE2 and PE3 advertise the same VPNv4 NLRI.

```text
65000:2:192.168.23.0/24
```

PE1 may still receive two BGP paths for that same VPNv4 route.

```text
65000:2:192.168.23.0/24 via 4.4.4.4
65000:2:192.168.23.0/24 via 6.6.6.6
```

But by default, the VRF imports only one of those paths.

The `import path` feature can change that behavior.

## Import Path Configuration

Configure import path under the IPv4 VRF address family.

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  import path selection all
  import path limit 2
  maximum-paths ibgp 2
```

There are three separate concepts here:

```text
import path selection
Controls which VPNv4 paths are imported into the VRF BGP table.

import path limit
Controls how many paths can be imported per prefix.

maximum-paths
Controls how many eligible BGP paths can be installed in the VRF routing table.
```

For load sharing, you normally need both of these steps:

```text
1. Import multiple paths into the VRF BGP table.
2. Install multiple paths into the VRF routing table.
```

## Command Syntax

The main commands are:

```text
import path selection {all | bestpath [strict] | multipath [strict]}
import path limit <number-of-import-paths>
```

The common configuration for same-RD load sharing is:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  import path selection all
  import path limit 2
  maximum-paths ibgp 2
```

## import path selection

This command is used to determine which paths are eligible for import from VPNv4 into a VRF.

There are three main options:
- `all`
- `bestpath`
- `multipaths`

`all` is the simplest and most common, but I'll also cover some details about `bestpath`.

I can't get `multipaths` working in my lab, but as the name suggests it imports the best route and all paths marked as multipaths
(indicated by code `m` in the VPNv4 table).

### import path selection all

The `all` option imports all eligible paths that match the VRF's import RT, up to the configured import path limit.

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  import path selection all
  import path limit 2
```

Example:

```text
VRF CUSTOMER_A imports RT 1:1.

VPNv4 table:
65000:2:192.168.23.0/24 via 4.4.4.4, RT 1:1
65000:2:192.168.23.0/24 via 6.6.6.6, RT 1:1

Import result:
Both paths are imported into CUSTOMER_A.
```

This is the most direct option when you want the VRF to import multiple paths for the same VPNv4 NLRI.

Then `maximum-paths` can install both paths in the VRF routing table if the paths are eligible for multipath.

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  import path selection all
  import path limit 2
  maximum-paths ibgp 2
```

### import path selection bestpath

The `bestpath` option imports one path.

It imports the best available path that matches the VRF's import RT.

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  import path selection bestpath
```

This is the **default import behavior**. A router only imports the best route from VPNv4 into a VRF by default.

Always keep in mind that the imported path must match the VRF's route-target import policy.

Example:

```text
VRF import RT = 1:1

Best VPNv4 path:
via 4.4.4.4, RT 2:2

Second-best VPNv4 path:
via 6.6.6.6, RT 1:1
```

With non-strict `bestpath`, the VRF imports the best available matching path.

```text
Result:
The VRF imports the route via 6.6.6.6.
```

The overall VPNv4 best path is via `4.4.4.4`, but that path has RT `2:2`.

The VRF imports only RT `1:1`.

So the router falls back to the best path that matches RT `1:1`.

### import path selection bestpath strict

The `strict` keyword removes the fallback behavior.

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  import path selection bestpath strict
```

With `bestpath strict`, the VRF imports the route only if the actual best VPNv4 path matches the VRF's import RT.

Example with `bestpath strict`:

```text
VRF import RT = 1:1

Best VPNv4 path:
via 4.4.4.4, RT 2:2

Second-best VPNv4 path:
via 6.6.6.6, RT 1:1

Result:
The VRF does not import either route.
```

The router does not fall back to the second-best matching path.

This is the purpose of `strict`.

### Non-Strict vs Strict

The difference can be summarized like this:

| Command                                 | Behavior                                                                             |
| --------------------------------------- | ------------------------------------------------------------------------------------ |
| `import path selection bestpath`        | Import the best path that this VRF is allowed to import                              |
| `import path selection bestpath strict` | Import only the actual best VPNv4 path, and only if this VRF is allowed to import it |

Another way to say it:

```text
bestpath:
Find the best eligible path.

bestpath strict:
Use the actual best path only.
If the actual best path is not eligible, import nothing.
```

### Testing bestpath strict

I changed PE2's export RT to `2:2`.

```text
vrf definition CUSTOMER_A
 rd 65000:2
 address-family ipv4
  route-target export 2:2
  route-target import 1:1
```

PE1's `CUSTOMER_A` VRF still imports only RT `1:1`.

```text
vrf definition CUSTOMER_A
 rd 65000:1
 address-family ipv4
  route-target import 1:1
  route-target export 1:1
```

PE1 was configured with `bestpath strict`.

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  import path selection bestpath strict
```

The expected test condition is:

```text
VRF import RT = 1:1

Best VPNv4 path:
via 4.4.4.4, RT 2:2

Second-best VPNv4 path:
via 6.6.6.6, RT 1:1

With bestpath strict:
Do not fall back to the second-best matching path.

Expected result:
CUSTOMER_A imports neither path.
```

However, at first PE1 still imports the path through PE3.

```text
PE1# show ip route vrf CUSTOMER_A | include 192.168.23
B     192.168.23.0/24 [200/0] via 6.6.6.6, 00:00:10
```

The VPNv4 BGP summary shows why:

```text
PE1# show bgp vpnv4 unicast all summary 

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
4.4.4.4         4        65000      21      20       14    0    0 00:10:33        0
6.6.6.6         4        65000      24      21       14    0    0 00:10:33        2
192.168.1.2     4        65100      24      27       14    0    0 00:10:38        1
```

PE1 shows `0` prefixes from PE2.

PE2 is advertising the routes, but PE1 is not retaining them because PE2 exports RT `2:2`.

PE1 has no local VRF importing RT `2:2`.

Due to default route-target filtering, PE1 discards PE2's VPNv4 routes.

So PE1 did not actually have this condition yet:

```text
Best VPNv4 path:        PE2, RT 2:2
Second-best VPNv4 path: PE3, RT 1:1
```

Instead, PE1 retained only the PE3 path.

Therefore, the PE3 path became the best VPNv4 path and was imported into `CUSTOMER_A`.

### Default Route-Target Filtering

By default, a PE discards VPNv4 routes if no local VRF imports the route's RT.

Example:

```text
PE2 advertises:
192.168.23.0/24, RT 2:2

PE1 local VRFs import:
RT 1:1 only

Default behavior:
PE1 does not retain the RT 2:2 VPNv4 route.
```

To keep VPNv4 routes even when no local VRF imports their route target, disable default route-target filtering.

You can do that with this command in IOS XE:

```text
router bgp 65000
 address-family vpnv4
  no bgp default route-target filter
```

After this change, PE1 retains routes from both PE2 and PE3.

```text
PE1# show bgp vpnv4 unicast all summary         

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
4.4.4.4         4        65000      27      27       18    0    0 00:13:14        2
6.6.6.6         4        65000      31      28       18    0    0 00:13:14        2
192.168.1.2     4        65100      30      36       18    0    0 00:13:18        1
```

Now PE1 has both VPNv4 paths.

The PE2 paths are selected as best.

```text
PE1# show bgp vpnv4 unicast all

Route Distinguisher: 65000:2
 *>i  11.11.11.0/24    4.4.4.4                  0    100      0 65200 i
 * i                   6.6.6.6                 11    100      0 65200 i
 *>i  192.168.23.0     4.4.4.4                  0    100      0 65200 i
 * i                   6.6.6.6                  0    100      0 65200 i
```

But PE1 does not import either path into `CUSTOMER_A`.

```text
PE1# show ip route vrf CUSTOMER_A

      10.0.0.0/24 is subnetted, 1 subnets
B        10.10.10.0 [20/0] via 192.168.1.2, 00:13:56
      192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.1.0/24 is directly connected, Ethernet0/1
L        192.168.1.1/32 is directly connected, Ethernet0/1
```

This proves the `bestpath strict` behavior.

```text
VRF import RT = 1:1

Best VPNv4 path:
via 4.4.4.4, RT 2:2

Second-best VPNv4 path:
via 6.6.6.6, RT 1:1

Result:
The VRF does not import either route.
```

The router does not fall back to the second-best path that matches RT `1:1`.

> `no bgp default route-target filter` is usually not a useful command.
> I used it here just to test `bestpath strict` behavior.

## import path limit

The `import path limit` command limits how many paths can be imported per prefix.

Example:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  import path selection all
  import path limit 2
```

This means:

```text
Import all eligible paths, but import no more than 2 paths per prefix.
```

For a two-PE multihoming design, `2` is usually enough.

```text
PE2 path
PE3 path
```

> The default limit is `1`, so this command is necesary to import more than one path.

## maximum-paths and import path

The import path commands and `maximum-paths` solve different problems.

```text
import path selection
Controls what gets imported from the VPNv4 table into the VRF BGP table.

maximum-paths
Controls what gets installed from the VRF BGP table into the VRF routing table.
```

Example:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  import path selection all
  import path limit 2
  maximum-paths ibgp 2
```

This says:

```text
1. Import up to 2 matching VPNv4 paths into CUSTOMER_A.
2. Install up to 2 eligible iBGP paths in the CUSTOMER_A routing table.
```

If you configure only import path selection but not `maximum-paths`, the VRF may import multiple paths into BGP but still install only one path in the routing table.

If you configure only `maximum-paths` but not import path selection, the VRF may still have only one imported BGP path.

For same-RD load sharing, both are usually required.

## maximum-paths under VPNv4

Do not confuse these two locations:

```text
address-family vpnv4
 maximum-paths 2
```

and:

```text
address-family ipv4 vrf CUSTOMER_A
 maximum-paths ibgp 2
```

For MPLS L3VPN VRF load sharing, the important command is under the IPv4 VRF address family.

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  maximum-paths ibgp 2
```

The forwarding decision for customer traffic is made from the VRF routing table.

The VPNv4 address family carries VPN routes between PEs.

The VRF IPv4 address family controls the customer routes installed in the VRF.

## Old Syntax

Older IOS syntax allowed the import value to be attached to the `maximum-paths` command.

Examples:

```text
maximum-paths ibgp 2 import 2
```

or:

```text
maximum-paths ibgp import 2
```

In newer IOS releases, this older `import` option is converted to the newer syntax:

```text
PE1(config-router-af)# maximum-paths ibgp 2 import 2
%NOTE: Import option has been deprecated.
%      Converting to 'import path selection all; import path limit 2'.
```

For modern notes, use the newer syntax:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  import path selection all
  import path limit 2
  maximum-paths ibgp 2
```

## Same-RD Load Sharing Example

Assume PE2 and PE3 both use the same RD.

PE2:

```text
vrf definition CUSTOMER_A
 rd 65000:2
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
```

PE3:

```text
vrf definition CUSTOMER_A
 rd 65000:2
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
```

Both PEs advertise the same VPNv4 NLRI.

```text
65000:2:192.168.23.0/24
```

PE1 receives two paths for that NLRI.

```text
Route Distinguisher: 65000:2
 * i 192.168.23.0/24 6.6.6.6 0 100 0 65200 i
 *>i                   4.4.4.4 0 100 0 65200 i
```

Without import path selection, PE1 imports only the best path into `CUSTOMER_A`.

```text
PE1# show ip route vrf CUSTOMER_A bgp | include 192.168.23
B 192.168.23.0/24 [200/0] via 4.4.4.4
```

Now configure import path selection on PE1.

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  import path selection all
  import path limit 2
  maximum-paths ibgp 2
```

After the BGP table is refreshed, PE1 can import both paths into the VRF and install both next hops.

```text
PE1# show ip route vrf CUSTOMER_A bgp
B     192.168.23.0/24 [200/0] via 6.6.6.6
                      [200/0] via 4.4.4.4
```

---

## Common Mistakes

### Mistake 1: Configuring only maximum-paths

This is not enough if the VRF imports only one path.

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  maximum-paths ibgp 2
```

If the VRF BGP table has only one imported path, there is only one path to install.

Fix:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  import path selection all
  import path limit 2
  maximum-paths ibgp 2
```

### Mistake 2: Configuring maximum-paths under VPNv4 only

This does not fix VRF forwarding load sharing.

Incorrect for VRF load sharing:

```text
router bgp 65000
 address-family vpnv4
  maximum-paths 2
```

Correct:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  maximum-paths ibgp 2
```

The VRF routing table is where customer forwarding decisions are made.

### Mistake 3: Forgetting import path limit

This is incomplete:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  import path selection all
```

Add a limit:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  import path selection all
  import path limit 2
```

The default limit is `1`, so you must configure a higher limit if you want to import multiple paths.

### Mistake 4: Using bestpath when you want load sharing

This imports only one path.

```text
import path selection bestpath
```

For load sharing in the same-RD case, use:

```text
import path selection all
import path limit 2
maximum-paths ibgp 2
```

### Mistake 5: Forgetting RT matching

Every imported path must still match the VRF's import RT.

```text
VRF import RT: 1:1
Route RT:      2:2

Result:
Not eligible for import.
```

`import path selection all` does not mean import routes with the wrong RT.

It means import all eligible matching paths, up to the limit.

## Unique RDs vs Import Path

Use unique RDs when possible.

```text
PE2 RD: 65000:2
PE3 RD: 65000:3
```

Unique RDs make the two advertisements different VPNv4 NLRIs.

```text
65000:2:192.168.23.0/24
65000:3:192.168.23.0/24
```

This preserves path diversity naturally.

Use import path selection when PE2 and PE3 use the same RD and PE1 receives multiple VPNv4 paths for the same VPNv4 NLRI.

```text
65000:2:192.168.23.0/24 via 4.4.4.4
65000:2:192.168.23.0/24 via 6.6.6.6
```

Then configure:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  import path selection all
  import path limit 2
  maximum-paths ibgp 2
```

## Design Recommendation

For classic PE-CE BGP multihoming, use this design when possible:

```text
Different RDs per PE per VRF
Same RT for VPN membership
maximum-paths under the VRF address family if load sharing is required
```

Example:

```text
PE2:
 rd 65000:2
 route-target export 1:1
 route-target import 1:1

PE3:
 rd 65000:3
 route-target export 1:1
 route-target import 1:1

PE1:
 address-family ipv4 vrf CUSTOMER_A
  maximum-paths ibgp 2
```

Use import path selection as an additional tool when the RDs are the same or must remain the same.

## Key Points

* `import path` controls which VPNv4 paths are imported into a VRF.
* It is configured under the IPv4 VRF address family.
* It does not directly install routes in the routing table.
* `import path selection all` imports all eligible matching paths up to the path limit.
* `import path selection bestpath` imports the best available path that matches the VRF import RT.
* `import path selection bestpath strict` imports only the actual best VPNv4 path, and only if it matches the VRF import RT.
* `import path selection multipaths` imports the best path and paths marked as multipath, if they match the VRF import RT.
* `import path limit` limits the number of imported paths per prefix.
* `maximum-paths` controls how many eligible paths can be installed in the VRF routing table.
* For same-RD load sharing, you usually need `import path selection all`, `import path limit 2`, and `maximum-paths ibgp 2`.
* Unique RDs are still the preferred multihoming design when possible.
