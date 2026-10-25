# Import and Export Maps

**Import maps** filter RT-eligible BGP VPN routes entering a VRF. **Export maps** apply policies to VPN routes leaving a VRF, including selectively assigning route targets (RTs).

Route targets (RTs) control which VPN routes are eligible for import. Import and export maps provide more selective control over which prefixes are shared.

## Main Idea

Consider two VRFs on R1:

```text
VRF_A                             VRF_B
3.3.3.3/32                        4.4.4.4/32
33.33.33.33/32                    44.44.44.44/32
```

Suppose the goal is to exchange only the Loopback0 prefixes:

| Source VRF | Prefix | Leak to Other VRF? |
| --- | --- | --- |
| `VRF_A` | `3.3.3.3/32` | Yes |
| `VRF_A` | `33.33.33.33/32` | No |
| `VRF_B` | `4.4.4.4/32` | Yes |
| `VRF_B` | `44.44.44.44/32` | No |

A route map can select the relevant prefixes. An import map filters RT-eligible imports, while an export map can selectively assign RTs to exported routes.

## Import Map vs. Export Map

| Command | Configured Under | Operation |
| --- | --- | --- |
| `import map <route-map>` | Destination VRF | Filters VPN routes eligible for import into that VRF |
| `export map <route-map>` | Source VRF | Sets export attributes, including RTs, on selected VPN routes |

Conceptually:

```text
Source VRF BGP routes
          |
          | export map (source VRF)
          v
      BGP VPNv4
          |
          | RT import eligibility
          | import map (destination VRF)
          v
Destination VRF BGP routes
```

An **import map does not replace `route-target import`**. A VPN route must first be eligible for import based on a matching RT, and the import map can then further restrict the routes accepted into the destination VRF.

An export map acts on the source side. In this lab, it selectively assigns RTs using `set extcommunity rt`. **Simply matching a prefix in a permit route-map entry is not an export filter:** if `route-target export` is configured, nonmatching prefixes still receive the unconditional export RT and can be imported.

## Lab Topology

R1 maintains `VRF_A` and `VRF_B`.

```text
                  R3
          Lo0: 3.3.3.3/32
        Lo1: 33.33.33.33/32
                   |
              E0/1 (VRF_A)
                   |
                  R1
                   |
              E0/2 (VRF_B)
                   |
                  R4
          Lo0: 4.4.4.4/32
        Lo1: 44.44.44.44/32
```

| Device | Interface | IPv4 Address | Routing Context on R1 |
| --- | --- | --- | --- |
| R1 | Ethernet0/1 | `10.1.3.1/24` | `VRF_A` |
| R3 | Ethernet0/0 | `10.1.3.3/24` | Connected to `VRF_A` |
| R3 | Loopback0 | `3.3.3.3/32` | Behind `VRF_A` |
| R3 | Loopback1 | `33.33.33.33/32` | Behind `VRF_A` |
| R1 | Ethernet0/2 | `10.1.4.1/24` | `VRF_B` |
| R4 | Ethernet0/0 | `10.1.4.4/24` | Connected to `VRF_B` |
| R4 | Loopback0 | `4.4.4.4/32` | Behind `VRF_B` |
| R4 | Loopback1 | `44.44.44.44/32` | Behind `VRF_B` |

**Goal:** Use import maps and export maps to leak `3.3.3.3/32` and `4.4.4.4/32` between the VRFs while excluding both Loopback1 prefixes.

## Step 1: Configure Routes to Both Loopbacks

R1 needs routes to the four loopback addresses in their original routing contexts.

```text
R1(config)# ip route vrf VRF_A 3.3.3.3 255.255.255.255 10.1.3.3
R1(config)# ip route vrf VRF_A 33.33.33.33 255.255.255.255 10.1.3.3
R1(config)# ip route vrf VRF_B 4.4.4.4 255.255.255.255 10.1.4.4
R1(config)# ip route vrf VRF_B 44.44.44.44 255.255.255.255 10.1.4.4
```

Verify:

```text
R1# show ip route vrf VRF_A static
R1# show ip route vrf VRF_B static
```

Both prefixes from R3 belong to `VRF_A`, while both prefixes from R4 belong to `VRF_B`.

## Step 2: Configure RDs and RTs

Configure the VRFs for bidirectional BGP VPN route exchange.

```text
R1(config)# vrf definition VRF_A
R1(config-vrf)# rd 65000:1
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# route-target export 65000:100
R1(config-vrf-af)# route-target import 65000:200
R1(config)# vrf definition VRF_B
R1(config-vrf)# rd 65000:2
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# route-target export 65000:200
R1(config-vrf-af)# route-target import 65000:100
```

The RTs establish which VPN routes are eligible to cross from one VRF to the other.

| Direction | Export RT | Matching Import RT |
| --- | --- | --- |
| `VRF_A` → `VRF_B` | `65000:100` | `65000:100` |
| `VRF_B` → `VRF_A` | `65000:200` | `65000:200` |

The initial `route-target export` statements assign RTs to all eligible BGP routes from each VRF. The import-map exercise filters those routes on receipt. The export-map exercise will remove these unconditional export RTs and selectively assign them through route maps instead.

## Step 3: Add the Routes to BGP

Add all four loopback routes so that the filtering mechanisms can select among them.

Redistribute the static routes under both BGP VRF address families:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast vrf VRF_A
R1(config-router-af)# redistribute static
R1(config-router)# address-family ipv4 unicast vrf VRF_B
R1(config-router-af)# redistribute static
```

The four matching static routes allow BGP to originate the corresponding prefixes inside their VRF address families.

With matching RTs and no import/export maps, all four loopback prefixes are eligible to be shared:

| Receiving VRF | Routes Imported from the Other VRF |
| --- | --- |
| `VRF_A` | `4.4.4.4/32`, `44.44.44.44/32` |
| `VRF_B` | `3.3.3.3/32`, `33.33.33.33/32` |
```
R1# show bgp vpnv4 unicast all
...
     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:1 (default for vrf VRF_A)
 *>   3.3.3.3/32       10.1.3.3                 0         32768 ?
 *>   4.4.4.4/32       10.1.4.4                 0         32768 ?
 *>   33.33.33.33/32   10.1.3.3                 0         32768 ?
 *>   44.44.44.44/32   10.1.4.4                 0         32768 ?
Route Distinguisher: 65000:2 (default for vrf VRF_B)
 *>   3.3.3.3/32       10.1.3.3                 0         32768 ?
 *>   4.4.4.4/32       10.1.4.4                 0         32768 ?
 *>   33.33.33.33/32   10.1.3.3                 0         32768 ?
 *>   44.44.44.44/32   10.1.4.4                 0         32768 ?
```

## Import Map Filtering

An **import map** is configured under the destination VRF's address family.

It filters which RT-eligible VPN routes the VRF imports.

### Step 4: Create the Import Route Maps

Create prefix lists matching the routes to be imported.

```text
R1(config)# ip prefix-list IMPORT_TO_A permit 4.4.4.4/32
R1(config)# ip prefix-list IMPORT_TO_B permit 3.3.3.3/32
```

Create the route maps:

```text
R1(config)# route-map IMPORT_TO_A permit 10
R1(config-route-map)# match ip address prefix-list IMPORT_TO_A
R1(config)# route-map IMPORT_TO_B permit 10
R1(config-route-map)# match ip address prefix-list IMPORT_TO_B
```

Each route map has one permit entry and an implicit deny for nonmatching prefixes.

For `IMPORT_TO_A`:

- `4.4.4.4/32` matches and is permitted.
- `44.44.44.44/32` does not match and is denied by the implicit deny.

For `IMPORT_TO_B`:

- `3.3.3.3/32` matches and is permitted.
- `33.33.33.33/32` does not match and is denied by the implicit deny.

### Step 5: Apply the Import Maps

Apply each route map under the **receiving** VRF:

```text
R1(config)# vrf definition VRF_A
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# import map IMPORT_TO_A
R1(config)# vrf definition VRF_B
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# import map IMPORT_TO_B
```

Note the direction:

```text
VRF_A:
  Import from VRF_B
  Import map matches 4.4.4.4/32
VRF_B:
  Import from VRF_A
  Import map matches 3.3.3.3/32
```

### Step 6: Verify the Import Filters

Check the receiving VRF routing tables:

```text
R1# show ip route vrf VRF_A
...
      3.0.0.0/32 is subnetted, 1 subnets
S        3.3.3.3 [1/0] via 10.1.3.3
      4.0.0.0/32 is subnetted, 1 subnets
B        4.4.4.4 [20/0] via 10.1.4.4 (VRF_B), 00:00:45  <-- IMPORTED ROUTE
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.1.3.0/24 is directly connected, Ethernet0/1
L        10.1.3.1/32 is directly connected, Ethernet0/1
      33.0.0.0/32 is subnetted, 1 subnets
S        33.33.33.33 [1/0] via 10.1.3.3
R1# show ip route vrf VRF_B
...
      3.0.0.0/32 is subnetted, 1 subnets
B        3.3.3.3 [20/0] via 10.1.3.3 (VRF_A), 00:00:46  <-- IMPORTED ROUTE
      4.0.0.0/32 is subnetted, 1 subnets
S        4.4.4.4 [1/0] via 10.1.4.4
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.1.4.0/24 is directly connected, Ethernet0/2
L        10.1.4.1/32 is directly connected, Ethernet0/2
      44.0.0.0/32 is subnetted, 1 subnets
S        44.44.44.44 [1/0] via 10.1.4.4
R1# show bgp vpnv4 unicast all
BGP table version is 13, local router ID is 10.1.2.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found
     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:1 (default for vrf VRF_A)
 *>   3.3.3.3/32       10.1.3.3                 0         32768 ?
 *>   4.4.4.4/32       10.1.4.4                 0         32768 ?
 *>   33.33.33.33/32   10.1.3.3                 0         32768 ?  <-- ONLY IN VRF_A
Route Distinguisher: 65000:2 (default for vrf VRF_B)
 *>   3.3.3.3/32       10.1.3.3                 0         32768 ?
 *>   4.4.4.4/32       10.1.4.4                 0         32768 ?
 *>   44.44.44.44/32   10.1.4.4                 0         32768 ? <-- ONLY IN VRF_B
```

Results:

| Receiving VRF | Remote Prefix | Result |
| --- | --- | --- |
| `VRF_A` | `4.4.4.4/32` | Imported |
| `VRF_A` | `44.44.44.44/32` | Not imported |
| `VRF_B` | `3.3.3.3/32` | Imported |
| `VRF_B` | `33.33.33.33/32` | Not imported |

The source-side VPN routes can still exist in BGP VPNv4 even when an import map rejects them for a particular destination VRF.

For example, `33.33.33.33/32` can remain an exported VPNv4 route from `VRF_A` without being imported into `VRF_B`.

This illustrates how an import map can suppress a destination-VRF route without removing the source VPNv4 route.

## Export Map Filtering

An **export map** is configured under the **source** VRF's address family. Its route map can assign RT extended communities to selected routes as they are exported into BGP VPNv4.

**Important:** An export map containing only a `match` statement does not automatically prevent nonmatching routes from receiving a VRF-wide `route-target export`. An unmatched route still gets the configured export RT, so a receiving VRF with a matching import RT can import it.

For selective RT-based leaking, the preferred method here is to **remove the unconditional export RT** and have the export map assign the desired RT only to selected prefixes.

### Step 7: Remove the Import Maps

Remove both import maps so that only the export policy controls selective leaking:

```text
R1(config)# vrf definition VRF_A
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# no import map IMPORT_TO_A
R1(config)# vrf definition VRF_B
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# no import map IMPORT_TO_B
```

The `route-target import` statements stay configured so that each VRF can import routes with the appropriate RT.

### Step 8: Remove the Unconditional Export RTs

Remove the RTs that were previously attached to all exported routes:

```text
R1(config)# vrf definition VRF_A
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# no route-target export 65000:100
R1(config)# vrf definition VRF_B
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# no route-target export 65000:200
```

This leaves the following import RTs:

```text
vrf definition VRF_A
 rd 65000:1
 address-family ipv4
  route-target import 65000:200
 exit-address-family
!
vrf definition VRF_B
 rd 65000:2
 address-family ipv4
  route-target import 65000:100
 exit-address-family
```

Routes will receive their selective export RTs from the export maps configured next.

### Step 9: Create the Export Route Maps

Match the Loopback0 prefixes:

```text
R1(config)# ip prefix-list EXPORT_FROM_A permit 3.3.3.3/32
R1(config)# ip prefix-list EXPORT_FROM_B permit 4.4.4.4/32
```

Create the export maps with `set extcommunity rt`:

```text
R1(config)# route-map EXPORT_FROM_A permit 10
R1(config-route-map)# match ip address prefix-list EXPORT_FROM_A
R1(config-route-map)# set extcommunity rt 65000:100
R1(config)# route-map EXPORT_FROM_B permit 10
R1(config-route-map)# match ip address prefix-list EXPORT_FROM_B
R1(config-route-map)# set extcommunity rt 65000:200
```

The RT assignments are:

| Source VRF | Prefix | RT Assigned by Export Map |
| --- | --- | --- |
| `VRF_A` | `3.3.3.3/32` | `65000:100` |
| `VRF_A` | `33.33.33.33/32` | None |
| `VRF_B` | `4.4.4.4/32` | `65000:200` |
| `VRF_B` | `44.44.44.44/32` | None |

The Loopback1 routes are still originated into BGP within their source VRFs. However, they do not receive an RT matching the other VRF's import policy.

### Step 10: Apply the Export Maps

Apply each export map under the **source** VRF:

```text
R1(config)# vrf definition VRF_A
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# export map EXPORT_FROM_A
R1(config)# vrf definition VRF_B
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# export map EXPORT_FROM_B
```

The complete VRF definitions for this exercise are:

```text
vrf definition VRF_A
 rd 65000:1
 !
 address-family ipv4
  export map EXPORT_FROM_A
  route-target import 65000:200
 exit-address-family
!
vrf definition VRF_B
 rd 65000:2
 !
 address-family ipv4
  export map EXPORT_FROM_B
  route-target import 65000:100
 exit-address-family
```

No `route-target export` is required because each export map assigns the appropriate RT directly.

### Step 11: Verify Selective RT Assignment

Check the routing and VPNv4 tables:

```text
R1# show ip route vrf VRF_A
...
      3.0.0.0/32 is subnetted, 1 subnets
S        3.3.3.3 [1/0] via 10.1.3.3
      4.0.0.0/32 is subnetted, 1 subnets
B        4.4.4.4 [20/0] via 10.1.4.4 (VRF_B), 01:16:28
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.1.3.0/24 is directly connected, Ethernet0/1
L        10.1.3.1/32 is directly connected, Ethernet0/1
      33.0.0.0/32 is subnetted, 1 subnets
S        33.33.33.33 [1/0] via 10.1.3.3

R1# show ip route vrf VRF_B
...
      3.0.0.0/32 is subnetted, 1 subnets
B        3.3.3.3 [20/0] via 10.1.3.3 (VRF_A), 00:00:16
      4.0.0.0/32 is subnetted, 1 subnets
S        4.4.4.4 [1/0] via 10.1.4.4
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.1.4.0/24 is directly connected, Ethernet0/2
L        10.1.4.1/32 is directly connected, Ethernet0/2
      44.0.0.0/32 is subnetted, 1 subnets
S        44.44.44.44 [1/0] via 10.1.4.4

R1# show bgp vpnv4 unicast all
...
     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:1 (default for vrf VRF_A)
 *>   3.3.3.3/32       10.1.3.3                 0         32768 ?
 *>   4.4.4.4/32       10.1.4.4                 0         32768 ?
 *>   33.33.33.33/32   10.1.3.3                 0         32768 ?
Route Distinguisher: 65000:2 (default for vrf VRF_B)
 *>   3.3.3.3/32       10.1.3.3                 0         32768 ?
 *>   4.4.4.4/32       10.1.4.4                 0         32768 ?
 *>   44.44.44.44/32   10.1.4.4                 0         32768 ?
```

Expected results:

| Destination VRF | Prefix from Other VRF | Result |
| --- | --- | --- |
| `VRF_A` | `4.4.4.4/32` | Imported |
| `VRF_A` | `44.44.44.44/32` | Not imported |
| `VRF_B` | `3.3.3.3/32` | Imported |
| `VRF_B` | `33.33.33.33/32` | Not imported |

**The key distinction:** Export-map policy selects which RTs are assigned to source VPN routes, whereas import-map policy filters RT-eligible VPN routes at the destination VRF.

## Alternative: Keep `route-target export` and Replace RTs

Instead of removing the unconditional export RTs, an export map can **replace the RT on an excluded prefix** with an RT the destination VRF does not import.

In this alternative, keep the original export RTs:

```text
R1(config)# vrf definition VRF_A
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# route-target export 65000:100
R1(config)# vrf definition VRF_B
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# route-target export 65000:200
```

Create prefix lists matching the **Loopback1** routes that should not be leaked:

```text
R1(config)# ip prefix-list EXCLUDE_FROM_A permit 33.33.33.33/32
R1(config)# ip prefix-list EXCLUDE_FROM_B permit 44.44.44.44/32
```

Assign a different, nonimported RT to those routes:

```text
R1(config)# route-map EXPORT_ALT_A permit 10
R1(config-route-map)# match ip address prefix-list EXCLUDE_FROM_A
R1(config-route-map)# set extcommunity rt 65000:999
R1(config)# route-map EXPORT_ALT_B permit 10
R1(config-route-map)# match ip address prefix-list EXCLUDE_FROM_B
R1(config-route-map)# set extcommunity rt 65000:998
```

Apply the alternative maps in place of the maps from the primary exercise:

```text
R1(config)# vrf definition VRF_A
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# export map EXPORT_ALT_A
R1(config)# vrf definition VRF_B
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# export map EXPORT_ALT_B
```

The expected RTs are:

| Source VRF | Prefix | Export RT | Imported into Other VRF? |
| --- | --- | --- | --- |
| `VRF_A` | `3.3.3.3/32` | `65000:100` | Yes |
| `VRF_A` | `33.33.33.33/32` | `65000:999` | No |
| `VRF_B` | `4.4.4.4/32` | `65000:200` | Yes |
| `VRF_B` | `44.44.44.44/32` | `65000:998` | No |

**Do not use `additive` for the excluded prefixes.** With `set extcommunity rt 65000:999 additive`, for example, `33.33.33.33/32` would retain RT `65000:100` as well as receive RT `65000:999`. Since `VRF_B` imports `65000:100`, the route would still be eligible for import.

Without `additive`, the `set extcommunity rt` action replaces the RT extended communities for the matched route.

The unmatched Loopback0 routes keep the configured VRF-wide `route-target export` RTs. Verify the actual RTs in BGP with the commands from Step 11.

## Adding an RT Instead of Replacing One

The `additive` keyword is useful when a selected route should carry **additional** RTs without losing its existing RTs.

For example:

```text
R1(config)# route-map EXPORT_FROM_A permit 10
R1(config-route-map)# match ip address prefix-list EXPORT_FROM_A
R1(config-route-map)# set extcommunity rt 65000:300 additive
```

If `route-target export 65000:100` is also configured, the matching route can carry both `65000:100` and `65000:300`. A VRF importing either RT can become eligible to import it. This is suitable for selectively adding destinations, not for excluding a route from one that already matches the original RT.

## Import Maps and Route Targets

An import map applies an additional filtering condition; it does not create RT eligibility by itself.

For example:

```text
vrf definition VRF_B
 address-family ipv4
  route-target import 65000:100
  import map IMPORT_TO_B
```

For `3.3.3.3/32` to be imported into `VRF_B`:

1. The exported VPN route must carry RT `65000:100`.
2. `IMPORT_TO_B` must permit `3.3.3.3/32`.

If either condition is not met, the route is not imported under this policy.

The import map cannot make a route with an unrelated RT eligible for import merely by permitting its prefix.

## Export Maps and Route Targets

An export map controls the **RTs assigned to exported VPN routes**. Unlike an import map, an export map does **not** require a `route-target export` statement if the route map assigns RTs using `set extcommunity rt`.

We covered two ways to use an export map for selective route leaking.

### Option 1: Assign RTs Only to Selected Prefixes

Remove `route-target export` and use the export map to assign the desired RT to permitted prefixes:

```text
ip prefix-list EXPORT_FROM_A permit 3.3.3.3/32
!
route-map EXPORT_FROM_A permit 10
 match ip address prefix-list EXPORT_FROM_A
 set extcommunity rt 65000:100
!
vrf definition VRF_A
 address-family ipv4
  export map EXPORT_FROM_A
```

- `3.3.3.3/32` receives RT `65000:100` and is eligible for import by a VRF configured with `route-target import 65000:100`.
- `33.33.33.33/32` does not match the export map and does not receive that RT, so it is not eligible for import based on RT `65000:100`.

**This is the preferred approach for the selective export example.** The export map assigns the RT only where needed.

### Option 2: Keep `route-target export` and Replace RTs

Alternatively, keep the VRF-wide export RT and use the export map to assign a **different RT to prefixes that should not be imported**:

```text
ip prefix-list EXCLUDE_FROM_A permit 33.33.33.33/32
!
route-map EXPORT_ALT_A permit 10
 match ip address prefix-list EXCLUDE_FROM_A
 set extcommunity rt 65000:999
!
vrf definition VRF_A
 address-family ipv4
  route-target export 65000:100
  export map EXPORT_ALT_A
```

- `3.3.3.3/32` does not match the map and retains the VRF-wide RT `65000:100`.
- `33.33.33.33/32` matches the map, so its RT is replaced with `65000:999` and it is no longer eligible for import by a VRF importing only `65000:100`.

**Do not use `additive` when replacing the RT to exclude a prefix.** The command `set extcommunity rt 65000:999 additive` would preserve RT `65000:100` and add `65000:999`, leaving the route eligible for import.

The distinction is:

- `route-target export` assigns an RT to all exported routes from the VRF.
- `export map` with `set extcommunity rt` can assign or replace RTs for selected routes, even without `route-target export`.
- `set extcommunity rt ... additive` adds an RT without removing existing RTs.
- `import map`, by contrast, filters routes *after* they satisfy the destination VRF's `route-target import` policy.

## Import and Export Map Comparison

| Characteristic | Import Map | Export Map |
| --- | --- | --- |
| Applied to | Destination VRF | Source VRF |
| Stage | VPN route import | VPN route export |
| Main purpose | Filter VPN routes eligible for import | Apply export policy, especially selective RT assignment |
| Requires matching `route-target import` to receive a route | Yes | The destination VRF requires a matching import RT |
| Requires `route-target export` | Not applicable to destination policy | No, if the export map sets the needed RT |
| Typical route-map action | `match ip address prefix-list` | `match ip address prefix-list` and `set extcommunity rt` |
| Changes original source static route | No | No |

The two maps can be used together. For example, export maps can tag selected prefixes, and import maps can apply additional filtering at the receiving VRF.

## Common Problems

- **Wrong map direction:** An import map belongs to the receiving VRF; an export map belongs to the originating VRF.
- **Unconditional RT defeats selective export:** A `route-target export` statement still assigns its RT to routes that do not match a permit clause in the export map.
- **Export map only matches a prefix:** Without a suitable `set extcommunity rt` action, matching a prefix alone does not selectively assign the RT needed for this exercise.
- **Missing import RT:** An `import map` cannot import a route unless the route also carries an RT configured by `route-target import`.
- **Incorrect `additive` use:** Adding an alternative RT without removing the original importable RT does not exclude a route.
- **No BGP origination:** The source prefix must be present in its BGP VRF address family before it can be exported as a VPN route.
- **Prefix-list mismatch:** The prefix or mask is incorrect, causing the route map not to match.
- **Missing return route:** R3 or R4 does not have a route back to the remote Loopback0 address.

## Key Points

- **Import maps** are applied under the receiving VRF and filter VPN routes after RT eligibility.
- **A matching `route-target import` is still required** for traditional VPNv4 route import. The import map does not replace RT matching.
- **Export maps** are applied under the source VRF and can selectively set route-target extended communities.
- An export map with only a matching permit entry **does not exclude unmatched routes** if an unconditional `route-target export` already tags them with a matching RT.
- **Preferred selective export method:** Remove the unconditional `route-target export` and use `set extcommunity rt` in export-map route-map entries that match the desired prefixes.
- **Alternative method:** Keep `route-target export` and use the export map to replace the original RT on prefixes that should not be imported elsewhere.
- **`set extcommunity rt` without `additive` replaces existing RTs** on matched routes; `additive` retains the original RTs and adds new ones.
- A source prefix can remain present in its original VRF even when no other VRF imports it.
- Both import and export maps can be used together.
- BGP route origination is configured under `address-family ipv4 unicast vrf <vrf-name>`; VRF RT policy is configured under `address-family ipv4` in the VRF definition.
- Bidirectional communication requires both forward and return routing.

## References

- [Cisco: Configure VRF Leaks on IOS XE](https://www.cisco.com/c/en/us/support/docs/ip/ip-routing/216541-vrf-configuration-examples-on-ios-xe.html)
- [Cisco: IOS MPLS Command Reference](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/mpls/command/mp-cr-book/mp-e1.html)
- [Cisco: Selective Export/Import Between VRFs Using Route Maps](https://community.cisco.com/t5/mpls/selective-export-import-between-vrfs-using-route-maps/td-p/314230)
