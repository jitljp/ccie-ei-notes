# Global-to-VRF Import and VRF-to-Global Export
BGP provides two complementary ways to exchange selected IPv4 routes between the **global routing table** and a **VRF**:

- **Global-to-VRF import:** Copy selected routes from the global BGP IPv4 unicast table into a VRF.

- **VRF-to-global export:** Copy selected routes from a VRF's BGP IPv4 unicast table into the global BGP table.

Both features use route maps to select prefixes:

```text
R1(config-vrf-af)# import ipv4 unicast map <route-map>
R1(config-vrf-af)# export ipv4 unicast map <route-map>
```

The commands are configured under `vrf definition` → `address-family ipv4`, not under `router bgp`.

| Feature | Direction | Source of Eligible Routes | Destination |
| --- | --- | --- | --- |
| `import ipv4 unicast map` | Global → VRF | Global BGP IPv4 unicast table | VRF BGP and routing tables |
| `export ipv4 unicast map` | VRF → Global | VRF BGP IPv4 unicast table | Global BGP and routing tables |

**Global-to-VRF import does not require route targets (RTs).** For VRF-to-global export, configure `route-target export` under the source VRF. The export route map selects the prefixes; the export RT enables their VPN route export processing. A `route-target import` statement is not required for this operation. These features are distinct from the shorter VPNv4 `import map` and `export map` commands.

## Main Idea
The global and VRF routing tables are separate. For a host in a VRF to communicate with a host in the global routing domain, routes must exist in **both directions**. The imported and exported routes must also have resolvable next hops so packets can cross the VRF boundary.

```text
                    Global-to-VRF IMPORT
Global IPv4 BGP table                     VRF_A BGP table
    2.2.2.2/32     ---- import map ---->      2.2.2.2/32
                    VRF-to-Global EXPORT
Global IPv4 BGP table                     VRF_A BGP table
    3.3.3.3/32     <--- export map -----      3.3.3.3/32
```

For example, R3 can reach R2's `2.2.2.2/32` only if R1's `VRF_A` has a route toward R2. R2 can return traffic to R3's `3.3.3.3/32` only if the global routing domain has a route toward R3.

BGP prefix import and export provide those routes dynamically:

```text
R3 (3.3.3.3) -> R1 VRF_A -> R1 Global -> R2 (2.2.2.2)
R2 (2.2.2.2) -> R1 Global -> R1 VRF_A -> R3 (3.3.3.3)
```

The `import ipv4 unicast map` route map determines which global prefixes each VRF receives. The `export ipv4 unicast map` route map determines which VRF prefixes become available in global BGP. For export, configure `route-target export` in the source VRF as well; the RT does not replace the prefix-selection route map.

**Both features are prefix filters.** A route map that does not permit a prefix prevents that prefix from being copied in the specified direction.

## Lab Topology
R1 connects a global BGP neighbor (R2) to two VRFs containing R3 and R4.

```text
                     R2
               Lo0: 2.2.2.2/32
                      |
                E0/0 (Global)
                      |
R3 ----- E0/1 ------- R1 ------- E0/2 ----- R4
Lo0:     (VRF_A)                 (VRF_B)     Lo0:
3.3.3.3/32                               4.4.4.4/32
```

| Device | Interface | IPv4 Address | Routing Context on R1 |
| --- | --- | --- | --- |
| R1 | Ethernet0/0 | `10.1.2.1/24` | Global |
| R2 | Ethernet0/0 | `10.1.2.2/24` | Connected to global |
| R2 | Loopback0 | `2.2.2.2/32` | Behind global |
| R2 | Loopback1 | `22.22.22.22/32` | Behind global |
| R1 | Ethernet0/1 | `10.1.3.1/24` | `VRF_A` |
| R3 | Ethernet0/0 | `10.1.3.3/24` | Connected to `VRF_A` |
| R3 | Loopback0 | `3.3.3.3/32` | Behind `VRF_A` |
| R3 | Loopback1 | `33.33.33.33/32` | Behind `VRF_A` |
| R1 | Ethernet0/2 | `10.1.4.1/24` | `VRF_B` |
| R4 | Ethernet0/0 | `10.1.4.4/24` | Connected to `VRF_B` |
| R4 | Loopback0 | `4.4.4.4/32` | Behind `VRF_B` |
| R4 | Loopback1 | `44.44.44.44/32` | Behind `VRF_B` |

The route-sharing objectives are:

| Direction | Route | Policy |
| --- | --- | --- |
| Global → `VRF_A` | `2.2.2.2/32` | Import |
| Global → `VRF_A` | `22.22.22.22/32` | Do not import |
| Global → `VRF_B` | `22.22.22.22/32` | Import |
| Global → `VRF_B` | `2.2.2.2/32` | Do not import |
| `VRF_A` → Global | `3.3.3.3/32` | Export |
| `VRF_A` → Global | `33.33.33.33/32` | Do not export |
| `VRF_B` → Global | `4.4.4.4/32` | Export |
| `VRF_B` → Global | `44.44.44.44/32` | Do not export |

These policies establish bidirectional connectivity between **R3 Loopback0 and R2 Loopback0**, and between **R4 Loopback0 and R2 Loopback1**.

## Prepare the Lab

### Step 1: Establish Global BGP Between R1 and R2
R2 originates both loopback prefixes into BGP and advertises them to R1 over eBGP.

On R2:

```text
R2(config)# router bgp 65002
R2(config-router)# neighbor 10.1.2.1 remote-as 65000
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.1.2.1 activate
R2(config-router-af)# network 2.2.2.2 mask 255.255.255.255
R2(config-router-af)# network 22.22.22.22 mask 255.255.255.255
```

The BGP `network` statements match R2's connected loopback routes.

On R1:

```text
R1(config)# router bgp 65000
R1(config-router)# neighbor 10.1.2.2 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.1.2.2 activate
```

Verify the BGP session and the global BGP routes. The following is the original captured CLI output:

```text
R1# show bgp ipv4 unicast summary
BGP router identifier 10.1.2.1, local AS number 65000
BGP table version is 1, main routing table version 1
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.1.2.2       4       65002       2       2        1    0    0 00:00:23        2

R1# show bgp ipv4 unicast
BGP table version is 3, local router ID is 10.1.2.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found
     Network          Next Hop            Metric LocPrf Weight Path
 *>   2.2.2.2/32       10.1.2.2                 0             0 65002 i
 *>   22.22.22.22/32   10.1.2.2                 0             0 65002 i
```

### Step 2: Configure Routes Toward R3 and R4
R1 needs routes to the prefixes behind R3 and R4 in their respective VRFs. These routes will later provide the source prefixes for VRF-to-global export.

Use fully specified static routes, with both the outgoing interface and next-hop IP address:

```text
R1(config)# ip route vrf VRF_A 3.3.3.3 255.255.255.255 Ethernet0/1 10.1.3.3
R1(config)# ip route vrf VRF_A 33.33.33.33 255.255.255.255 Ethernet0/1 10.1.3.3
R1(config)# ip route vrf VRF_B 4.4.4.4 255.255.255.255 Ethernet0/2 10.1.4.4
R1(config)# ip route vrf VRF_B 44.44.44.44 255.255.255.255 Ethernet0/2 10.1.4.4
```

The outgoing interface is important for **cross-VRF next-hop resolution** when these routes are exported into global BGP.

In my lab, next-hop-only static routes were able to enter the global BGP table, and all required routes appeared in the global and VRF routing tables.

However, R1 could not resolve the next hops of the exported routes in its global CEF table, preventing successful packet forwarding.

Using fully specified static routes (outgoing interface and next-hop IP address) resolved the forwarding issue. Learning the routes through OSPF also worked.

We could also use static routing between R1 and R2 instead of eBGP.

Regardless of the routing method, the prefixes must be present in the appropriate **BGP tables on R1**:

- **Global-to-VRF import:** The prefixes must exist in R1's **global IPv4 unicast BGP table**.

- **VRF-to-global export:** The prefixes must exist in R1's **VRF-specific IPv4 unicast BGP tables**.

If static routes are used, they can be originated into BGP using `network` statements or `redistribute static`.

## Global-to-VRF Import
`import ipv4 unicast map` imports selected prefixes from the **global BGP IPv4 unicast table** into the configured VRF.

```text
Global RIB -> Global BGP IPv4 unicast table
                         |
                import ipv4 unicast map
                         |
                         v
                    Destination VRF
```

The source prefixes must be in **global BGP**. A global static or connected route that has not entered BGP is not eligible for this BGP import feature.

### Step 3: Create Import Route Maps
Use a different prefix list for each destination VRF:

```text
R1(config)# ip prefix-list GLOBAL_TO_A permit 2.2.2.2/32

R1(config)# ip prefix-list GLOBAL_TO_B permit 22.22.22.22/32
```

Create the route maps:

```text
R1(config)# route-map GLOBAL_TO_A permit 10
R1(config-route-map)# match ip address prefix-list GLOBAL_TO_A

R1(config)# route-map GLOBAL_TO_B permit 10
R1(config-route-map)# match ip address prefix-list GLOBAL_TO_B
```

Each route map has an implicit deny for prefixes that do not match its permit entry:

- `GLOBAL_TO_A` permits `2.2.2.2/32` into `VRF_A` and excludes `22.22.22.22/32`.

- `GLOBAL_TO_B` permits `22.22.22.22/32` into `VRF_B` and excludes `2.2.2.2/32`.

### Step 4: Configure Global-to-VRF Import
Apply each map under the **destination VRF**:

```text
R1(config)# vrf definition VRF_A
R1(config-vrf)# rd 65000:1
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# import ipv4 unicast map GLOBAL_TO_A

R1(config)# vrf definition VRF_B
R1(config-vrf)# rd 65000:2
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# import ipv4 unicast map GLOBAL_TO_B
```

The command syntax is:

```text
import ipv4 unicast [prefix-limit] map <route-map>
```

| Element | Purpose |
| --- | --- |
| `import` | Copy selected routes from global BGP into this VRF |
| `ipv4 unicast` | Use the IPv4 unicast address family |
| `[prefix-limit]` | Optional cap on imported prefixes; default **1000** |
| `map <route-map>` | Select which global BGP prefixes are imported |

No `route-target import` or `route-target export` is needed for this global-to-VRF import step. The source prefixes are selected directly from global BGP. A `route-target export` will be added later only to support the reverse, VRF-to-global export operation.

The VRF's RD identifies its VPN routing context; it is not the prefix-selection criterion.

### Step 5: Verify Imported Routes
Check which prefixes were imported into the VPNv4 views of the VRFs:

```text
R1# show bgp vpnv4 unicast all
...
Route Distinguisher: 65000:1 (default for vrf VRF_A)
Import Map: GLOBAL_TO_A, Address-Family: IPv4 Unicast, Pfx Count/Limit: 1/1000
 *>   2.2.2.2/32       10.1.2.2                 0             0 65002 i
Route Distinguisher: 65000:2 (default for vrf VRF_B)
Import Map: GLOBAL_TO_B, Address-Family: IPv4 Unicast, Pfx Count/Limit: 1/1000
 *>   22.22.22.22/32   10.1.2.2                 0             0 65002 i
```

Notice the `Import Map` and `Pfx Count/Limit` lines. Each VRF has imported exactly one global prefix.

Check the VRF routing tables. The following original captures show the imported routes alongside local static and connected routes. The static routes were subsequently changed to the fully specified form in Step 2 to support cross-VRF forwarding:

```text
R1# show ip route vrf VRF_A
...
      2.0.0.0/32 is subnetted, 1 subnets
B        2.2.2.2 [20/0] via 10.1.2.2, 00:00:54
      3.0.0.0/32 is subnetted, 1 subnets
S        3.3.3.3 [1/0] via 10.1.3.3
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.1.3.0/24 is directly connected, Ethernet0/1
L        10.1.3.1/32 is directly connected, Ethernet0/1
      33.0.0.0/32 is subnetted, 1 subnets
S        33.33.33.33 [1/0] via 10.1.3.3

R1# show ip route vrf VRF_B
...
      4.0.0.0/32 is subnetted, 1 subnets
S        4.4.4.4 [1/0] via 10.1.4.4
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.1.4.0/24 is directly connected, Ethernet0/2
L        10.1.4.1/32 is directly connected, Ethernet0/2
      22.0.0.0/32 is subnetted, 1 subnets
B        22.22.22.22 [20/0] via 10.1.2.2, 00:00:56
      44.0.0.0/32 is subnetted, 1 subnets
S        44.44.44.44 [1/0] via 10.1.4.4
```

The expected result is:

| VRF | Prefix | Result |
| --- | --- | --- |
| `VRF_A` | `2.2.2.2/32` | Imported as BGP |
| `VRF_A` | `22.22.22.22/32` | Not imported |
| `VRF_B` | `2.2.2.2/32` | Not imported |
| `VRF_B` | `22.22.22.22/32` | Imported as BGP |

Importing the prefixes does not remove them from the original global BGP table. The BGP next hop for both imported routes is `10.1.2.2`, which R1 reaches through the global routing context.

### Importing Global Static or Connected Routes
A route that exists **only** in the global RIB cannot be imported using `import ipv4 unicast map` until it enters global BGP.

For example, if R1 has the following global static route:

```text
R1(config)# ip route 2.2.2.2 255.255.255.255 10.1.2.2
```

R1 could originate that route into global BGP using a matching `network` statement:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# network 2.2.2.2 mask 255.255.255.255
```

The route is then eligible for global-to-VRF BGP prefix import, subject to the import route map. A filtered `redistribute static` policy is another way to originate static routes into BGP.

### Prefix Import Limit
The default import limit is **1000 prefixes per VRF**. To configure a different limit:

```text
R1(config)# vrf definition VRF_A
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# import ipv4 unicast 50 map GLOBAL_TO_A
```

The syntax is:

```text
import ipv4 unicast <prefix-limit> map <route-map>
```

The supported limit range is **1 to 2,147,483,647**. Cisco documents a maximum of **five VRFs per router** for this global-to-VRF IPv4 import feature and can report `%BGP-3-AFIMPORT_EXCEED` when the import limit is exceeded.

## VRF-to-Global Export
`export ipv4 unicast map` performs the reverse operation: it selects routes from a **VRF BGP table** and exports them into the **global BGP IPv4 unicast table**, from which they can enter the global routing table and be advertised to BGP neighbors.

In this lab, configure `route-target export` under each source VRF for VRF-to-global export to work. No `route-target import` is required.

The export RT enables the route export, while `export ipv4 unicast map` selects which VRF prefixes are copied into global BGP.

```text
VRF routing table -> VRF BGP IPv4 unicast table
                                |
                     export ipv4 unicast map
                                |
                                v
                      Global BGP and RIB
```

For the return direction, R2 must be able to reach `3.3.3.3/32` and `4.4.4.4/32` through R1's global routing domain.

### Step 6: Originate the VRF Routes into BGP
The static VRF routes installed in Step 2 must enter their respective **BGP VRF address families** before export.

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast vrf VRF_A
R1(config-router-af)# redistribute static

R1(config-router)# address-family ipv4 unicast vrf VRF_B
R1(config-router-af)# redistribute static
```

`redistribute static` originates the routes to both loopbacks in each VRF into BGP. You could also use the `network` command.

The export route maps in the following step select **only** the Loopback0 routes.

Verify the VRF BGP tables:

```text
R1# show bgp vpnv4 unicast vrf VRF_A
BGP table version is 7, local router ID is 10.1.2.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found
     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:1 (default for vrf VRF_A)
Import Map: GLOBAL_TO_A, Address-Family: IPv4 Unicast, Pfx Count/Limit: 1/1000
 *>   2.2.2.2/32       10.1.2.2                 0             0 65002 i
 *>   3.3.3.3/32       10.1.3.3                 0         32768 ?
 *>   33.33.33.33/32   10.1.3.3                 0         32768 ?

R1# show bgp vpnv4 unicast vrf VRF_B
BGP table version is 7, local router ID is 10.1.2.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found
     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:2 (default for vrf VRF_B)
Import Map: GLOBAL_TO_B, Address-Family: IPv4 Unicast, Pfx Count/Limit: 1/1000
 *>   4.4.4.4/32       10.1.4.4                 0         32768 ?
 *>   22.22.22.22/32   10.1.2.2                 0             0 65002 i
 *>   44.44.44.44/32   10.1.4.4                 0         32768 ?
```

### Step 7: Create Export Route Maps
Define the routes R2 needs to reach through R1:

```text
R1(config)# ip prefix-list VRF_A_TO_GLOBAL permit 3.3.3.3/32

R1(config)# ip prefix-list VRF_B_TO_GLOBAL permit 4.4.4.4/32
```

Create the export route maps:

```text
R1(config)# route-map VRF_A_TO_GLOBAL permit 10
R1(config-route-map)# match ip address prefix-list VRF_A_TO_GLOBAL

R1(config)# route-map VRF_B_TO_GLOBAL permit 10
R1(config-route-map)# match ip address prefix-list VRF_B_TO_GLOBAL
```

- `VRF_A_TO_GLOBAL` permits `3.3.3.3/32` and denies `33.33.33.33/32`.

- `VRF_B_TO_GLOBAL` permits `4.4.4.4/32` and denies `44.44.44.44/32`.

### Step 8: Configure VRF-to-Global Export
Apply each export map beneath its **source VRF**, and configure an export RT:

```text
R1(config)# vrf definition VRF_A
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# export ipv4 unicast map VRF_A_TO_GLOBAL
R1(config-vrf-af)# route-target export 65000:100

R1(config)# vrf definition VRF_B
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# export ipv4 unicast map VRF_B_TO_GLOBAL
R1(config-vrf-af)# route-target export 65000:200
```

`route-target export` is necessary for the export to work. No `route-target import` is required in either VRF.

The `export ipv4 unicast map` command controls which prefixes enter global BGP: `3.3.3.3/32` and `4.4.4.4/32`, not the Loopback1 prefixes.

The syntax is:

```text
export ipv4 unicast [prefix-limit] map <route-map>
```

Only VRF BGP prefixes permitted by these maps are exported into the global table. **This is a true prefix-selection filter**, unlike the shorter `export map` used for VPNv4 route-target policy.

Verify the result on R1. The following output was captured after adding `route-target export` to both VRFs:

```text
R1# show bgp vpnv4 unicast all
...
Route Distinguisher: 65000:1 (default for vrf VRF_A)
Import Map: GLOBAL_TO_A, Address-Family: IPv4 Unicast, Pfx Count/Limit: 1/1000
Export Map: VRF_A_TO_GLOBAL, Address-Family: IPv4 Unicast, Pfx Count/Limit: 1/1000
 *>   2.2.2.2/32       10.1.2.2                 0             0 65002 i
 *>   3.3.3.3/32       10.1.3.3                 0         32768 ?
 *>   33.33.33.33/32   10.1.3.3                 0         32768 ?
Route Distinguisher: 65000:2 (default for vrf VRF_B)
Import Map: GLOBAL_TO_B, Address-Family: IPv4 Unicast, Pfx Count/Limit: 1/1000
Export Map: VRF_B_TO_GLOBAL, Address-Family: IPv4 Unicast, Pfx Count/Limit: 1/1000
 *>   4.4.4.4/32       10.1.4.4                 0         32768 ?
 *>   22.22.22.22/32   10.1.2.2                 0             0 65002 i
 *>   44.44.44.44/32   10.1.4.4                 0         32768 ?

R1# show bgp ipv4 unicast
BGP table version is 10, local router ID is 10.1.2.1
...
     Network          Next Hop            Metric LocPrf Weight Path
 *>   2.2.2.2/32       10.1.2.2                 0             0 65002 i
 *>   3.3.3.3/32       10.1.3.3                 0         32768 ?
 *>   4.4.4.4/32       10.1.4.4                 0         32768 ?
 *>   22.22.22.22/32   10.1.2.2                 0             0 65002 i
```

The export counters are `1/1000` in both VRFs, and the global BGP table now includes `3.3.3.3/32` and `4.4.4.4/32`. The two Loopback1 prefixes are not exported.

Check the global RIB entries:

```text
R1# show ip route 3.3.3.3
R1# show ip route 4.4.4.4
```

Expected routing results:

| Prefix | Global BGP | Global RIB | Source VRF |
| --- | --- | --- | --- |
| `3.3.3.3/32` | Present | Present | `VRF_A` |
| `4.4.4.4/32` | Present | Present | `VRF_B` |
| `33.33.33.33/32` | Not exported | Not installed by this feature | `VRF_A` |
| `44.44.44.44/32` | Not exported | Not installed by this feature | `VRF_B` |

The exported global routes have next hops in their source VRFs. R1 must be able to resolve those next hops to the correct outgoing interfaces. The fully specified static routes from Step 2 provide that information; the forwarding checks later in the lab confirm it.

### Export RT Requirement
In my lab, the export maps matched the VRF routes, but the global BGP table did not receive them until `route-target export` was configured.

The `show bgp vpnv4 unicast all` output reported `Export Map ... Pfx Count/Limit: 0/1000` before the export RT was added, and `1/1000` afterward.

The working source VRF configuration is:

```text
vrf definition VRF_A
 rd 65000:1
 address-family ipv4
  import ipv4 unicast map GLOBAL_TO_A
  export ipv4 unicast map VRF_A_TO_GLOBAL
  route-target export 65000:100
 exit-address-family
```

The import operation still works without an import RT, and no `route-target import` is needed for this global/VRF route exchange. The export RT is **not** matched by the global table: prefix selection is performed by `VRF_A_TO_GLOBAL`.

### Prefix Export Limit
The default prefix export limit is also **1000 per VRF**. To limit exports explicitly:

```text
R1(config)# vrf definition VRF_A
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# export ipv4 unicast 50 map VRF_A_TO_GLOBAL
```

The supported limit range is **1 to 2,147,483,647** prefixes. The route map continues to determine which prefixes are eligible, and the prefix limit caps how many can be exported.

## End-to-End Reachability

### Step 10: Configure Forward Routes on R3 and R4
R3 and R4 must send packets destined for R2's loopbacks to R1.

On R3:

```text
R3(config)# ip route 2.2.2.2 255.255.255.255 10.1.3.1
```

On R4:

```text
R4(config)# ip route 22.22.22.22 255.255.255.255 10.1.4.1
```

These are ordinary static routes. R3 and R4 have no separate VRFs in this lab; their outgoing interfaces connect to the corresponding VRFs on R1.

### Verify Cross-VRF Next-Hop Resolution

The routes in the BGP and IP routing tables do not, by themselves, show whether R1 can forward traffic across the VRF boundary. The exported routes must have resolvable next hops.

For example, the following next-hop-only static route in `VRF_A` was initially used:

```text
R1(config)# ip route vrf VRF_A 3.3.3.3 255.255.255.255 10.1.3.3
```

Although `3.3.3.3/32` was successfully exported into global BGP and installed in the global RIB, R1 could not resolve its next hop in the global forwarding table:

```text
R1# show ip cef 3.3.3.3 detail
3.3.3.3/32, epoch 0, flags [rib only nolabel, rib defined all labels]
  recursive via 10.1.3.3, unresolved
```

The next hop `10.1.3.3` is reachable through `VRF_A` on Ethernet0/1, not through an interface in R1's global routing context. Packets returning from R2 therefore could not be forwarded to R3.

Two route-source configurations resolved this problem:

- **Fully specified static route:** Identify both Ethernet0/1 and `10.1.3.3` in the `VRF_A` static route, as configured in Step 2.
- **VRF-aware OSPF:** Learn `3.3.3.3/32` dynamically from R3 through OSPF running in `VRF_A`.

The equivalent configurations work for the prefixes in `VRF_B` using Ethernet0/2. Both methods were successful in this topology without adding global static routes for `10.1.3.3` or `10.1.4.4`.

Verify both forwarding directions on R1:

```text
R1# show ip cef vrf VRF_A 2.2.2.2 detail
R1# show ip cef 3.3.3.3 detail
R1# show ip cef vrf VRF_B 22.22.22.22 detail
R1# show ip cef 4.4.4.4 detail
```

The imported global prefixes should resolve toward Ethernet0/0, while the exported VRF prefixes should resolve toward Ethernet0/1 or Ethernet0/2 as appropriate.

### Step 11: Verify Bidirectional Connectivity

Test connectivity:

```text
R2# ping 3.3.3.3 source l0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 3.3.3.3, timeout is 2 seconds:
Packet sent with a source address of 2.2.2.2 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms

R2# ping 4.4.4.4 source l1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 4.4.4.4, timeout is 2 seconds:
Packet sent with a source address of 22.22.22.22 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```

The route requirements are:

| Router / Table | Needed Prefix | How It Is Learned |
| --- | --- | --- |
| R1 `VRF_A` | `2.2.2.2/32` | Global-to-VRF BGP import |
| R1 `VRF_B` | `22.22.22.22/32` | Global-to-VRF BGP import |
| R1 global | `3.3.3.3/32` | VRF-to-global BGP export |
| R1 global | `4.4.4.4/32` | VRF-to-global BGP export |
| R2 global | `3.3.3.3/32`, `4.4.4.4/32` | R1 eBGP advertisements |
| R3 | `2.2.2.2/32` | Static route via `10.1.3.1` |
| R4 | `22.22.22.22/32` | Static route via `10.1.4.1` |

Import and export on R1 are independent. **Full reachability depends on both policies, the routes on R2/R3/R4, and correct cross-VRF next-hop resolution on R1.**

## Comparison with VPNv4 Import and Export Maps
Do not confuse these four commands:

| Command | Route Source → Destination | Effect | RT Requirement |
| --- | --- | --- | --- |
| `import ipv4 unicast map <map>` | Global BGP → VRF | Selects prefixes copied into a VRF | **No** |
| `export ipv4 unicast map <map>` | VRF BGP → Global BGP | Selects prefixes copied into global BGP | **Export RT needed; matching import RT not needed** |
| `import map <map>` | VPNv4 → VRF | Filters VPN routes after RT eligibility | **Yes** |
| `export map <map>` | VRF BGP → VPNv4 | Can selectively set or modify RTs; not a simple export-block filter | Not necessarily; can assign RTs itself |

## Imported and Exported Routes: Restrictions
- Global IPv4 prefixes imported with `import ipv4 unicast map` cannot then be re-imported into another VPNv4 VRF through ordinary RT leaking. Configure an import policy separately on each destination VRF.

- IPv4 prefixes exported into the global table with `export ipv4 unicast map` cannot then be exported onward into a VPNv4 VRF through ordinary VPNv4 import/export.

## Key Points
- `import ipv4 unicast map <route-map>` selects prefixes from **global BGP** for import into a **VRF**.

- `export ipv4 unicast map <route-map>` selects prefixes from **VRF BGP** for export into **global BGP**, from where they can enter the global RIB and be advertised to neighbors.

- Both commands are configured under `vrf definition` → `address-family ipv4`.

- **Both directions require the source routes to exist in the appropriate BGP address family.** Merely existing in the source routing table is not enough.

- **Global-to-VRF import does not require RTs.** It selects global BGP prefixes directly using the route map.

- VRF-to-global export requires `route-target export`, but not `route-target import`. The route map determines which VRF BGP prefixes enter the global BGP table.

- Both import and export support an optional prefix limit; each defaults to **1000** per VRF.

- `import map` and `export map` without `ipv4 unicast` are different features associated with VPNv4 route import/export.

- Global-to-VRF and VRF-to-global route sharing are **independent operations**. Bidirectional connectivity requires the necessary forward and return paths.

- **Exporting a prefix into global BGP does not guarantee forwarding.** The exported next hop must also resolve across the VRF boundary.

- The selected prefixes remain in their originating routing contexts; the route maps determine where they are additionally available.

## References
- [Cisco: BGP Support for IP Prefix Import from Global Table into a VRF Table (IOS XE 17.x)](https\://www\.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-sup-ip-pref-0.html)

- [Cisco: BGP Support for IP Prefix Export from a VRF Table into the Global Table](https\://www\.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-prefix-export.html)

- [Cisco: Configure VRF Leaks on IOS XE](https\://www\.cisco.com/c/en/us/support/docs/ip/ip-routing/216541-vrf-configuration-examples-on-ios-xe.html)
