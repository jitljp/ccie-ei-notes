# Route-Target Import/Export

**Route-target (RT) import/export** allows BGP routes to be exchanged between VRFs based on their route-target extended communities.

On IOS XE, this mechanism can be used to leak routes between VRFs on the **same router**, without a BGP neighbor or an MPLS core.

## Main Idea

Suppose R1 has two VRFs:

```text
VRF_A: route to 3.3.3.3/32
VRF_B: route to 4.4.4.4/32
```

By default, neither VRF has a route to the other VRF's destination.

Route-target leaking uses BGP as an intermediate control plane:

```text
VRF_A                          BGP VPNv4                         VRF_B
3.3.3.3/32  -- export RT 65000:100 --> VPNv4 -- import RT 65000:100 --> 3.3.3.3/32
4.4.4.4/32  <-- import RT 65000:200 -- VPNv4 <-- export RT 65000:200 -- 4.4.4.4/32
```

A route is eligible for import when an **import RT configured on the destination VRF matches an RT carried by the VPN route**.

- `route-target export` attaches an RT to routes exported from a VRF into BGP VPNv4.
- `route-target import` identifies VPN routes that a VRF can import.

The import and export RT values must match for the intended direction of leaking.

## Route Distinguisher vs. Route Target

Both **route distinguishers (RDs)** and **route targets (RTs)** are used in BGP VPN routing, but they have different purposes.

| Attribute | Purpose |
| --- | --- |
| RD | Makes otherwise identical IPv4 prefixes unique in VPNv4 NLRI |
| Export RT | Marks VPN routes with an extended community for import policy |
| Import RT | Determines which VPN routes are eligible for import into the VRF |

For example:

```text
VRF_A:
 RD:       65000:1
 Export RT: 65000:100

VRF_B:
 RD:       65000:2
 Import RT: 65000:100
```

If `VRF_A` exports `3.3.3.3/32` into BGP, the VPN route has:

```text
VPNv4 NLRI: 65000:1:3.3.3.3/32
Export RT:   65000:100
```

`VRF_B` can import the route because its import RT matches **65000:100**.

**The RDs do not need to match.** RDs identify VPN routes; RTs control route import and export.

## Lab Topology

R1 connects two routing contexts, `VRF_A` and `VRF_B`.

```text
              R3
        Lo0: 3.3.3.3/32
               |
          E0/1 (VRF_A)
               |
              R1
               |
          E0/2 (VRF_B)
               |
              R4
        Lo0: 4.4.4.4/32
```

| Device | Interface | IPv4 Address | Routing Context on R1 |
| --- | --- | --- | --- |
| R1 | Ethernet0/1 | `10.1.3.1/24` | `VRF_A` |
| R3 | Ethernet0/0 | `10.1.3.3/24` | Connected to `VRF_A` |
| R3 | Loopback0 | `3.3.3.3/32` | Behind `VRF_A` |
| R1 | Ethernet0/2 | `10.1.4.1/24` | `VRF_B` |
| R4 | Ethernet0/0 | `10.1.4.4/24` | Connected to `VRF_B` |
| R4 | Loopback0 | `4.4.4.4/32` | Behind `VRF_B` |

**Goal:** Establish bidirectional connectivity between `3.3.3.3/32` in `VRF_A` and `4.4.4.4/32` in `VRF_B` using BGP route-target import/export on R1.

## Step 1: Configure Routes to the Loopbacks

First, R1 needs a route to each loopback in its original VRF.

```text
R1(config)# ip route vrf VRF_A 3.3.3.3 255.255.255.255 10.1.3.3
R1(config)# ip route vrf VRF_B 4.4.4.4 255.255.255.255 10.1.4.4
```

Verify the routes:

```text
R1# show ip route vrf VRF_A 3.3.3.3
R1# show ip route vrf VRF_B 4.4.4.4
```

At this point, the prefixes are still in separate routing tables:

```text
VRF_A: S 3.3.3.3/32 via 10.1.3.3
VRF_B: S 4.4.4.4/32 via 10.1.4.4
```

An ordinary static route inside a VRF is **not automatically exported as a BGP VPN route**.

## Step 2: Configure RDs and Route Targets

Configure the RD and RTs for `VRF_A`:

```text
R1(config)# vrf definition VRF_A
R1(config-vrf)# rd 65000:1
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# route-target export 65000:100
R1(config-vrf-af)# route-target import 65000:200
```

Configure the RD and RTs for `VRF_B`:

```text
R1(config)# vrf definition VRF_B
R1(config-vrf)# rd 65000:2
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# route-target export 65000:200
R1(config-vrf-af)# route-target import 65000:100
```

The two sets of RTs provide leaking in opposite directions:

| Direction | Source VRF Export RT | Destination VRF Import RT |
| --- | --- | --- |
| `VRF_A` → `VRF_B` | `65000:100` | `65000:100` |
| `VRF_B` → `VRF_A` | `65000:200` | `65000:200` |

These commands establish the import/export policies. The static routes still need to enter BGP before they can be exported.

## Step 3: Originate the Loopback Routes in BGP

Configure one BGP process on R1, with an IPv4 address family for each VRF:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast vrf VRF_A
R1(config-router-af)# network 3.3.3.3 mask 255.255.255.255

R1(config-router)# address-family ipv4 unicast vrf VRF_B
R1(config-router-af)# network 4.4.4.4 mask 255.255.255.255
```

The `network` statements originate the two prefixes into their respective **BGP VRF address families**.

The static routes from Step 1 provide the exact matching entries needed for the `network` statements.

```text
VRF_A static route: 3.3.3.3/32
            |
            v
VRF_A BGP route:    3.3.3.3/32
            |
            | Export RT 65000:100
            v
VPNv4 route:        65000:1:3.3.3.3/32
            |
            | VRF_B imports RT 65000:100
            v
VRF_B BGP route:    3.3.3.3/32
```

The reverse process imports `4.4.4.4/32` into `VRF_A`.

**No BGP neighbor is required for local RT-based leaking.** The BGP process handles export and import between the local VRF address families.

### Alternative: Redistribute Static Routes

Instead of originating an exact prefix with a `network` statement, static routes can be redistributed into the respective BGP VRF address family:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast vrf VRF_A
R1(config-router-af)# redistribute static

R1(config-router)# address-family ipv4 unicast vrf VRF_B
R1(config-router-af)# redistribute static
```

This makes **eligible static routes** from each VRF available for BGP VPN export, rather than only the exact prefixes in the `network` statements.

Use the `network` method for the lab above. Route-map-based filtering of imports and exports is covered in [Import and Export Maps](import-export-maps.md).

## Step 4: Configure Return Routes on R3 and R4

R3 and R4 need routes to one another's loopbacks through R1.

On R3:

```text
R3(config)# ip route 4.4.4.4 255.255.255.255 10.1.3.1
```

On R4:

```text
R4(config)# ip route 3.3.3.3 255.255.255.255 10.1.4.1
```

R3 and R4 use ordinary static routes because they have only their local routing tables.

## Route Import and Forwarding

After both directions are configured, R1 has the following routes:

| Routing Table | Prefix | Route Source |
| --- | --- | --- |
| `VRF_A` | `3.3.3.3/32` | Original static route toward R3 |
| `VRF_A` | `4.4.4.4/32` | BGP route imported from `VRF_B` |
| `VRF_B` | `4.4.4.4/32` | Original static route toward R4 |
| `VRF_B` | `3.3.3.3/32` | BGP route imported from `VRF_A` |

When R3 sends a packet to `4.4.4.4`:

1. R3 forwards the packet to R1 at `10.1.3.1`.
2. R1 receives the packet on `Ethernet0/1` and performs a lookup in `VRF_A`.
3. `VRF_A` contains the BGP route for `4.4.4.4/32`, imported from `VRF_B`.
4. R1 forwards the packet using the resolved path toward R4 through `Ethernet0/2`.
5. R4 returns traffic through R1, which uses the corresponding imported route in `VRF_B` to reach R3.

The import process makes the remote prefix available in the receiving VRF. It does not merge the VRFs or modify their interface memberships.

## Verification

### Check the VRF Configuration

```text
R1# show vrf detail VRF_A
VRF VRF_A (VRF Id = 1); default RD 65000:1; default VPNID <not set>
  New CLI format, supports multiple address-families
  Flags: 0x180C
  Interfaces:
    Et0/1                   
Address family ipv4 unicast (Table ID = 0x1):
  Flags: 0x0
  Export VPN route-target communities
    RT:65000:100            
  Import VPN route-target communities
    RT:65000:200            
...

R1# show vrf detail VRF_B
VRF VRF_B (VRF Id = 2); default RD 65000:2; default VPNID <not set>
  New CLI format, supports multiple address-families
  Flags: 0x180C
  Interfaces:
    Et0/2                   
Address family ipv4 unicast (Table ID = 0x2):
  Flags: 0x0
  Export VPN route-target communities
    RT:65000:200            
  Import VPN route-target communities
    RT:65000:100            
...

R1# show running-config | sec vrf definition
vrf definition VRF_A
 rd 65000:1
 !
 address-family ipv4
  route-target export 65000:100
  route-target import 65000:200
 exit-address-family
vrf definition VRF_B
 rd 65000:2
 !
 address-family ipv4
  route-target export 65000:200
  route-target import 65000:100
 exit-address-family
```

Check that each VRF has its RD and the intended import/export RTs.

### Check the BGP Routes

```text
R1# show bgp vpnv4 unicast all           
BGP table version is 5, local router ID is 10.1.2.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:1 (default for vrf VRF_A)
 *>   3.3.3.3/32       10.1.3.3                 0         32768 i
 *>   4.4.4.4/32       10.1.4.4                 0         32768 i
Route Distinguisher: 65000:2 (default for vrf VRF_B)
 *>   3.3.3.3/32       10.1.3.3                 0         32768 i
 *>   4.4.4.4/32       10.1.4.4                 0         32768 i
```

Notice that both routes appear for both VRFs.

### Check the Imported Routes

```text
R1# show ip route vrf VRF_A
...
      3.0.0.0/32 is subnetted, 1 subnets
S        3.3.3.3 [1/0] via 10.1.3.3
      4.0.0.0/32 is subnetted, 1 subnets
B        4.4.4.4 [20/0] via 10.1.4.4 (VRF_B), 00:06:04    <-- BGP ROUTE IMPORTED FROM VRF_B
C        10.1.3.0/24 is directly connected, Ethernet0/1
L        10.1.3.1/32 is directly connected, Ethernet0/1

R1# show ip route vrf VRF_B
...
      3.0.0.0/32 is subnetted, 1 subnets
B        3.3.3.3 [20/0] via 10.1.3.3 (VRF_A), 00:06:20    <-- BGP ROUTE IMPORTED FROM VRF_A
      4.0.0.0/32 is subnetted, 1 subnets
S        4.4.4.4 [1/0] via 10.1.4.4
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.1.4.0/24 is directly connected, Ethernet0/2
L        10.1.4.1/32 is directly connected, Ethernet0/2
```

### Check CEF Forwarding

```text
R1# show ip cef vrf VRF_A 4.4.4.4 detail 
4.4.4.4/32, epoch 0, flags [rib only nolabel, rib defined all labels]
  recursive via 10.1.4.4
    recursive via 10.1.4.0/24
      attached to Ethernet0/2

R1# show ip cef vrf VRF_B 3.3.3.3 detail 
3.3.3.3/32, epoch 0, flags [rib only nolabel, rib defined all labels]
  recursive via 10.1.3.3
    recursive via 10.1.3.0/24
      attached to Ethernet0/1
```

### Test Connectivity

From R3:

```text
R3# ping 4.4.4.4 source l0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 4.4.4.4, timeout is 2 seconds:
Packet sent with a source address of 3.3.3.3 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/6 ms
```

From R4:

```text
R4# ping 3.3.3.3 source l0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 3.3.3.3, timeout is 2 seconds:
Packet sent with a source address of 4.4.4.4 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```

Both directions should work after R1 imports the BGP routes and the return routes are configured.

## Route-Target Matching

An export RT is compared with the **import RTs of the destination VRF**. It does not have to equal the source VRF's RD.

For example, if `VRF_B` instead had:

```text
vrf definition VRF_B
 address-family ipv4 unicast
  no route-target import 65000:100
```

Then the `3.3.3.3/32` VPN route exported by `VRF_A` would no longer qualify for import into `VRF_B` based on that RT.

A VRF can have multiple import and export RTs. This supports more complex route-sharing relationships without changing the RD.

A single RT can also be shared by multiple VRFs. Each VRF that imports that RT is eligible to receive the VPN routes carrying it.

## Common Problems

- **No BGP route originated:** The static route exists in the VRF routing table, but the route was not originated with `network` or redistributed into BGP.
- **RT mismatch:** The source VRF exports an RT that the destination VRF does not import.
- **Missing RD:** The VRF does not have the RD required for the BGP VPN route.
- **Wrong address family:** BGP origination or RT configuration is applied to a different VRF or address family.
- **Missing return route:** R3 or R4 does not have a route toward the remote loopback through R1.
- **Conflicting route:** The receiving VRF has a more preferred route to the same prefix.

## Key Points

- RT-based route leaking **uses BGP VPN route import/export** to exchange routes between VRFs.
- `route-target export` adds RT extended communities to exported VPN routes.
- `route-target import` selects VPN routes with matching RT extended communities for import into a VRF.
- **The source export RT must match a destination import RT** for the intended leak.
- An **RD uniquely identifies a VPN route**; it does not determine which VRFs import it.
- Routes must enter the appropriate **BGP VRF address family** before they can be exported as VPN routes.
- **A BGP neighbor is not required** when leaking routes between local VRFs on the same router.
- Imported routes appear as **BGP routes** in the destination VRF routing table.
- Bidirectional connectivity requires usable routes in both directions.

## References

- [Cisco: Configure VRF Leaks on IOS XE](https://www.cisco.com/c/en/us/support/docs/ip/ip-routing/216541-vrf-configuration-examples-on-ios-xe.html)
- [Cisco: MPLS L3VPN Route Import and Export](../../../transport-technologies-and-solutions/mpls/l3vpn/route-import-export.md)
