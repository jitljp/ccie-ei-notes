# Extended BGP Communities

**Extended BGP communities** are 64-bit values that provide additional information about BGP routes and allow routing policies to identify, filter, and manipulate routes.

Unlike standard communities, extended communities include **type information** that defines how their values should be interpreted.

Extended communities are commonly used for:

- **Route Targets (RTs):** Control route import and export between VRFs.
- **Site-of-Origin (SoO):** Prevent routing loops in multihomed MPLS VPN customer sites.
- **Cost Communities:** Influence BGP best-path selection at specific points in the algorithm.
- **Link Bandwidth:** Communicate bandwidth information for BGP unequal-cost multipath.
- **OSPF VPN information:** Preserve OSPF routing information across MPLS L3VPN networks.

Extended communities are carried in the **EXTENDED_COMMUNITIES path attribute (Type 16)**.

The attribute is optional transitive, although individual extended-community types can be transitive or nontransitive.

---

## Extended Community Format

Each extended community occupies **8 bytes (64 bits)**.

An extended community contains:

1. A **Type field** identifying the community's format and semantics.
2. A **Value field** containing the information associated with that type.

The Type field can be one or two bytes.

For commonly used two-byte extended types, the general structure is:

```text
    1 byte      1 byte              6 bytes
+------------+------------+--------------------------+
| Type       | Sub-Type   | Value                    |
+------------+------------+--------------------------+
```

The remaining six bytes can be divided into additional subfields, depending on the type.

For example, Route Target and Site-of-Origin communities typically use:

- **Global Administrator:** Identifies the administrative authority defining the value.
- **Local Administrator:** Identifies a particular routing policy, site, or VPN.

### Extended Community Encoding Types

RFC 4360 defines several extended-community encoding formats.

| Encoding | Global Administrator | Local Administrator |
|---|---|---|
| Two-Octet AS Specific | 2-byte ASN | 4 bytes |
| IPv4 Address Specific | 4-byte IPv4 address | 2 bytes |
| Four-Octet AS Specific | 4-byte ASN | 2 bytes |

For example, the following are valid Route Target representations:

```text
65001:100
65536:100
192.0.2.1:100
```

The representation depends on the encoding used.

**Important:** Unlike standard communities, an extended community formatted as `65001:100` does not necessarily consist of two 16-bit fields.

For a Two-Octet AS Specific Route Target:

```text
    1 byte      1 byte       2 bytes       4 bytes
+------------+------------+-------------+----------------+
| Type       | Sub-Type   | ASN         | Local Value    |
+------------+------------+-------------+----------------+
```

The Local Administrator field is 32 bits.

For a Four-Octet AS Specific Route Target:

```text
    1 byte      1 byte       4 bytes       2 bytes
+------------+------------+-------------+----------------+
| Type       | Sub-Type   | ASN         | Local Value    |
+------------+------------+-------------+----------------+
```

This format supports 4-byte ASNs.

### Type and Sub-Type

The Type information identifies the structure and meaning of the extended community.

For example:

| Extended Community | Sub-Type |
|---|---|
| Route Target | `0x02` |
| Route Origin (SoO) | `0x03` |

The full extended-community type also identifies the encoding format and transitivity behavior.

**Important:** The same numerical value can represent different extended communities depending on the type.

For example:

```text
RT:65001:100
SoO:65001:100
```

These are not interchangeable.

A Route Target controls VPN route import/export, while a Site-of-Origin identifies the customer site associated with a route.

---

## Transitive and Nontransitive Extended Communities

Although the EXTENDED_COMMUNITIES path attribute is optional transitive, **individual extended communities can be transitive or nontransitive**.

The distinction is encoded in the extended community's Type field.

RFC 4360 defines a transitivity bit with the following meanings:

| Transitivity Bit | Meaning |
|---|---|
| 0 | Transitive |
| 1 | Nontransitive |

### Transitive Extended Communities

Transitive extended communities can be propagated across AS boundaries, subject to routing policies and other restrictions.

Examples include:

- Route Targets.
- Site-of-Origin communities.

For example, Route Targets can be carried between provider ASes in an inter-AS MPLS L3VPN deployment.

### Nontransitive Extended Communities

Nontransitive extended communities are not propagated across ordinary AS boundaries.

An important Cisco example is the **Cost Community**.

Cost Communities can propagate through:

- IBGP sessions.
- EBGP sessions within a BGP confederation.

They are not propagated to ordinary external EBGP neighbors.

**Important:** The `neighbor ... send-community extended` command does not override nontransitive behavior.

---

## Common Extended Community Types

The following extended communities are particularly relevant to Cisco enterprise networks.

| Extended Community | Primary Purpose |
|---|---|
| Route Target (RT) | VPN route import and export |
| Site-of-Origin (SoO) | Loop prevention for multihomed VPN sites |
| Cost Community | BGP best-path preference |
| Link Bandwidth | BGP unequal-cost multipath |
| OSPF Domain ID | Identify OSPF domains across MPLS VPNs |
| OSPF Route Type | Preserve OSPF route-type information across MPLS VPNs |

Not all extended communities are configured with the same commands.

For example:

- Route Targets can be configured using `route-target` or `set extcommunity rt`.
- Site-of-Origin can be configured using `neighbor ... soo` or `set extcommunity soo`.
- Cost Communities use `set extcommunity cost`.
- Link Bandwidth can be generated using `neighbor ... dmzlink-bw`.

---

## Route Target Extended Community

A **Route Target (RT)** is an extended community used to control which VPN routes are imported into particular VRFs.

Route Targets are fundamental to MPLS Layer 3 VPNs.

The main functions are:

- **Export RT:** Attached to routes exported from a VRF into BGP VPNv4/VPNv6.
- **Import RT:** Identifies which VPN routes are eligible for import into a VRF.

A destination VRF can import a VPN route when at least one of its configured import RTs matches an RT attached to that route.

### Route Target Example

Consider two PE routers connected through an MPLS provider network.

```text
                 MPLS Provider Network

           PE1 ---------------------- PE2
            |                          |
            |                          |
           CE1                        CE2
            |                          |
          VRF A                      VRF A
```

Both customer sites belong to the same VPN.

PE1 exports routes using RT `65000:100`.

PE2 imports routes carrying RT `65000:100`.

The process is:

```text
PE1                                             PE2

VRF A                                           VRF A
  |                                               ^
  | Export RT 65000:100                           |
  v                                               | Import RT 65000:100
VPNv4 Route ----------------------------------> VPNv4 Route
```

### Configuring Route Targets

On PE1:

```text
PE1(config)# vrf definition A
PE1(config-vrf)# rd 65000:1
PE1(config-vrf)# address-family ipv4
PE1(config-vrf-af)# route-target export 65000:100
PE1(config-vrf-af)# route-target import 65000:100
```

On PE2:

```text
PE2(config)# vrf definition A
PE2(config-vrf)# rd 65000:2
PE2(config-vrf)# address-family ipv4
PE2(config-vrf-af)# route-target export 65000:100
PE2(config-vrf-af)# route-target import 65000:100
```

The RDs are different, but the RTs match.

This is a common configuration because RDs and RTs serve different purposes.

### Route Distinguisher vs. Route Target

| Feature | Route Distinguisher | Route Target |
|---|---|---|
| Size | 64 bits | 64 bits |
| Extended community? | No | Yes |
| Carried in | VPN NLRI | EXTENDED_COMMUNITIES attribute |
| Main purpose | Make VPN prefixes unique | Control route import/export |
| Must match between VRFs? | No | Import/export RTs must match for the intended route exchange |

For example:

```text
VPNv4 Prefix: 65000:1:192.0.2.0/24
Route Target: RT:65000:100
```

The RD makes the VPN prefix unique.

The RT determines which VRFs can import the route.

### Multiple Route Targets

A VPN route can carry multiple RTs.

For example:

```text
RT:65000:100
RT:65000:200
RT:65000:300
```

A VRF configured to import any one of these RTs can be eligible to import the route.

This enables flexible VPN topologies, including:

- Full-mesh VPNs.
- Hub-and-spoke VPNs.
- Shared-services VPNs.
- Extranet connectivity.

**Important:** RT matching determines import eligibility. Other routing policies and route-selection requirements can still prevent import or installation.

See [Route-Target Import/Export](../../../routing-concepts/route-leaking/route-target-import-export.md).

---

## Site-of-Origin Extended Community

The **Site-of-Origin (SoO)** extended community identifies the customer site from which a VPN route was learned.

Its primary purpose is to prevent routing loops when a customer site has multiple connections to a provider network.

SoO is particularly useful in **multihomed MPLS L3VPN sites**.

### Multihomed Site Problem

Consider a customer site connected to two PE routers:

```text
                  MPLS Provider Network

                 PE1 ---------------- PE2
                  |                    |
                  |                    |
                  +---------+----------+
                            |
                       Customer Site
                            |
                           CE1
```

Suppose CE1 advertises `192.0.2.0/24` to PE1.

Without SoO:

1. PE1 learns the prefix from CE1.
2. PE1 advertises the VPN route to PE2 through MP-BGP.
3. PE2 imports the route into the customer VRF.
4. PE2 might advertise the route back to CE1 through its PE-CE connection.

This can lead to undesirable route feedback or routing loops, depending on the PE-CE routing protocol and topology.

### How SoO Prevents Loops

Assign the same SoO value to routes learned from the same customer site.

For example:

```text
SoO:65000:10
```

When PE1 learns a route from CE1:

1. PE1 tags the route with SoO `65000:10`.
2. PE1 advertises the VPN route to PE2.
3. PE2 recognizes that the route originated from the customer site associated with SoO `65000:10`.
4. PE2 prevents the route from being advertised back to that site.

The key requirement is that **all PE connections to the same customer site use the same SoO value**.

Different sites should normally use different SoO values.

### Configuring SoO Using a Route Map

```text
PE1(config)# route-map SET-SOO permit 10
PE1(config-route-map)# set extcommunity soo 65000:10
```

The route map can be applied to an appropriate inbound PE-CE routing policy to tag routes learned from the customer site.

### Configuring SoO on a BGP Neighbor

Cisco IOS XE also supports the `neighbor ... soo` command in the IPv4 VRF BGP address family.

For example:

```text
PE1(config)# router bgp 65000
PE1(config-router)# address-family ipv4 vrf CUSTOMER
PE1(config-router-af)# neighbor 10.0.12.2 remote-as 65100
PE1(config-router-af)# neighbor 10.0.12.2 soo 65000:10
```

On the other PE connected to the same customer site, configure the same SoO value for its corresponding CE neighbor.

This provides SoO-based route tagging and loop prevention for the PE-CE BGP relationship.

**Important:** The `neighbor ... soo` configuration is specific to supported VRF-aware BGP functionality. SoO can also be used with other PE-CE routing protocols through supported route-map or interface-based mechanisms.

### SoO vs. Route Target

| Feature | Route Target | Site-of-Origin |
|---|---|---|
| Identifies | VPN membership/import eligibility | Originating customer site |
| Primary purpose | Route import/export | Loop prevention |
| Used for multihomed sites? | Yes | Yes, specifically for loop prevention |
| Extended community? | Yes | Yes |
| Transitive? | Yes | Yes |

**Route Targets answer "Which VRFs may import this route?"**

**Site-of-Origin answers "Which site did this route come from?"**

---

## Cost Communities

The **BGP Cost Community** is a Cisco-supported extended community used to influence BGP best-path selection.

Unlike ordinary communities, Cost Communities can directly affect the best-path selection algorithm.

Cost Communities contain:

1. A Point of Insertion (POI).
2. A Community ID.
3. A Cost Value.

The general representation is:

```text
Cost:<POI>:<Community-ID>:<Cost-Value>
```

For example:

```text
Cost:igp:10:100
Cost:pre-bestpath:20:200
```

### Points of Insertion

Cisco IOS XE supports two important POIs:

| POI | Evaluated |
|---|---|
| `pre-bestpath` | Before Weight |
| `igp` | After the IGP metric to the NEXT_HOP |

The **lowest cost value is preferred** when comparing applicable Cost Communities at the same POI and Community ID.

The Community ID determines the order in which Cost Communities are compared.

### Configuring Cost Communities

Configure the `igp` POI:

```text
R1(config)# route-map SET-COST permit 10
R1(config-route-map)# set extcommunity cost igp 10 100
```

Configure the `pre-bestpath` POI:

```text
R1(config)# route-map SET-COST permit 10
R1(config-route-map)# set extcommunity cost pre-bestpath 10 100
```

The default POI is `igp`, so the following is equivalent to explicitly specifying `igp`:

```text
R1(config-route-map)# set extcommunity cost 10 100
```

### Cost Community Propagation

Cost Communities are **nontransitive extended communities**.

They can be propagated through IBGP and BGP confederations but are not propagated across ordinary EBGP AS boundaries.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 send-community extended
```

This enables extended-community advertisement to the neighbor, subject to the applicable propagation rules.

Cost Communities are also used in EIGRP MPLS L3VPN environments to preserve EIGRP route preference information.

For full coverage of Cost Community values, comparison rules, default costs, Points of Insertion, and verification, see [BGP Cost Communities and Points of Insertion](../../path-selection/best-path/cost-community.md).

---

## Link Bandwidth Extended Community

The **Link Bandwidth extended community** communicates bandwidth information associated with a BGP path.

Cisco IOS XE can use this information to implement **unequal-cost multipath (UCMP)**.

Normally, BGP multipath load balancing distributes traffic across eligible paths without necessarily considering the physical bandwidth of each exit link.

Link Bandwidth allows supported platforms to distribute traffic proportionally to link bandwidth.

### Example

Consider two exit links:

```text
                       AS 65001

                 R1 -------------- R2
                  |                 |
                  | 100 Mbps        | 50 Mbps
                  |                 |
                ISP1              ISP2
```

Both routers provide paths toward the same destination.

Suppose the paths are eligible for BGP multipath and the platform supports bandwidth-aware forwarding.

The bandwidth ratio is:

```text
100 Mbps : 50 Mbps

      2 : 1
```

The intended traffic distribution is approximately:

- 66.7% through R1.
- 33.3% through R2.

Actual forwarding depends on the platform, forwarding implementation, and traffic flows.

### Configuring Link Bandwidth Advertisement

On a router with a directly connected EBGP neighbor:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 dmzlink-bw
```

The `neighbor ... dmzlink-bw` command enables BGP to associate the external link's bandwidth with routes learned through the neighbor.

To send the extended community to an IBGP peer:

```text
R1(config-router-af)# neighbor 10.0.13.3 send-community extended
```

On routers performing bandwidth-based multipath forwarding:

```text
R1(config-router-af)# bgp dmzlink-bw
```

BGP multipath must also be configured appropriately.

For example:

```text
R1(config-router-af)# maximum-paths 2
```

Or, for eligible IBGP multipath:

```text
R1(config-router-af)# maximum-paths ibgp 2
```

The exact combination of multipath and bandwidth commands depends on the topology and platform.

**Important:** The Link Bandwidth extended community does not independently make multiple BGP paths eligible for multipath. Normal BGP multipath requirements must still be satisfied.

See [BGP Load Balancing](../../path-selection/load-balancing.md).

---

## OSPF Extended Communities in MPLS L3VPNs

Extended communities are also used when OSPF routes are transported across an MPLS Layer 3 VPN.

A PE router redistributing OSPF routes into MP-BGP must preserve information needed when those routes are later advertised back into OSPF.

Relevant extended communities include:

| Extended Community | Purpose |
|---|---|
| OSPF Domain ID | Identify the originating OSPF domain |
| OSPF Route Type | Preserve information about the OSPF route type |
| OSPF Router ID | Identify the originating OSPF router |

These extended communities help remote PE routers determine how to advertise VPN routes back into OSPF.

For example, whether a route should be advertised as:

- An inter-area route using a Type 3 Summary LSA.
- An external route using a Type 5 or Type 7 LSA, as appropriate.

The PE-CE OSPF domain configuration and associated loop-prevention mechanisms also influence this process.

These communities are generally generated and processed automatically by the supported PE-CE OSPF implementation rather than manually assigned using ordinary community-tagging route maps.

---

## Configuring Extended Communities with Route Maps

Cisco IOS XE uses the `set extcommunity` command to manipulate supported extended-community types.

### Setting a Route Target

```text
R1(config)# route-map SET-RT permit 10
R1(config-route-map)# set extcommunity rt 65001:100
```

This sets the Route Target to `65001:100`.

### Adding a Route Target

```text
R1(config)# route-map ADD-RT permit 10
R1(config-route-map)# set extcommunity rt 65001:100 additive
```

The `additive` keyword preserves existing RT values while adding the new RT.

For example, if a route initially carries:

```text
RT:65001:200
```

After the route map:

```text
RT:65001:100
RT:65001:200
```

Without `additive`, the configured RT value replaces the existing RT set.

**Important:** `set extcommunity rt` operates on Route Target extended communities. It should not be confused with `set community`, which manipulates standard BGP communities.

### Setting Multiple Route Targets

Multiple RT values can be specified:

```text
R1(config-route-map)# set extcommunity rt 65001:100 65001:200 additive
```

This adds both RTs while preserving existing RTs.

### Setting Site-of-Origin

```text
R1(config)# route-map SET-SOO permit 10
R1(config-route-map)# set extcommunity soo 65001:10
```

This assigns the specified SoO extended community.

### Removing Specific Extended Communities

Cisco IOS XE supports `set extcomm-list ... delete` for removing extended communities matching an extended community list.

For example:

```text
R1(config)# ip extcommunity-list standard REMOVE-RT permit rt 65001:100

R1(config)# route-map REMOVE-RT permit 10
R1(config-route-map)# set extcomm-list REMOVE-RT delete
```

This removes matching extended communities while leaving other extended communities intact.

This is useful when rewriting Route Targets in VPN routing policies.

---

## Extended Community Lists

Cisco IOS XE supports **extended community lists** for matching extended community attributes.

These are distinct from ordinary BGP community lists.

| Community Type | List Command | Route-Map Match Command |
|---|---|---|
| Standard BGP community | `ip community-list` | `match community` |
| Extended BGP community | `ip extcommunity-list` | `match extcommunity` |

Extended community lists support:

- Standard lists.
- Expanded lists.
- Named lists.
- Numbered lists.

### Standard Extended Community Lists

Standard extended community lists match explicitly specified extended community values.

For example:

```text
R1(config)# ip extcommunity-list standard BLUE-RT permit rt 65001:100
```

This matches routes carrying Route Target `65001:100`.

To match a Site-of-Origin:

```text
R1(config)# ip extcommunity-list standard SITE-A permit soo 65001:10
```

The keywords `rt` and `soo` identify which extended-community type should be matched.

For example, the following do not match one another:

```text
RT:65001:100
SoO:65001:100
```

Even though their numerical components are identical, the extended-community types differ.

### Matching an Extended Community in a Route Map

```text
R1(config)# ip extcommunity-list standard BLUE-RT permit rt 65001:100

R1(config)# route-map MATCH-RT permit 10
R1(config-route-map)# match extcommunity BLUE-RT
R1(config-route-map)# set local-preference 200

R1(config)# route-map MATCH-RT permit 20
```

This route map:

1. Matches routes carrying RT `65001:100`.
2. Sets LOCAL_PREF to 200.
3. Permits other routes unchanged through sequence 20.

The route map must be applied to an appropriate routing-policy operation to take effect.

### Multiple Extended Community List Entries

Extended community lists support permit and deny entries.

For example:

```text
R1(config)# ip extcommunity-list standard VPN-RT permit rt 65001:100
R1(config)# ip extcommunity-list standard VPN-RT permit rt 65001:200
```

The entries are evaluated in order, and the first matching entry determines the community-list result.

A route carrying either matching RT can satisfy this list.

As with standard community lists, an extended community-list `deny` means the list match fails. It does not independently cause the BGP route to be discarded.

### Expanded Extended Community Lists

Expanded extended community lists use regular expressions to match extended-community string representations.

For example:

```text
R1(config)# ip extcommunity-list expanded RT-RANGE permit RT:65001:1[0-9][0-9]
```

This expression matches Route Target strings containing values in the range:

```text
RT:65001:100
RT:65001:101
...
RT:65001:199
```

Regular expressions are useful for matching large groups of extended communities without individually specifying each value.

As with expanded standard community lists, regular-expression matching can be sensitive to the string representation and ordering of values.

For exact RT or SoO values, standard extended community lists are generally simpler.

### Numbered Extended Community Lists

Cisco IOS XE supports the following numbered ranges:

| List Type | Range |
|---|---|
| Standard extended community list | 1-99 |
| Expanded extended community list | 100-500 |

For example:

```text
R1(config)# ip extcommunity-list 10 permit rt 65001:100
```

Or:

```text
R1(config)# ip extcommunity-list 100 permit RT:65001:.*
```

These are numbered equivalents of standard and expanded extended community lists.

---

## Route Target Filtering and Rewrite

An important application of extended community lists is **Route Target filtering and replacement**.

For example, a PE router might need to replace:

```text
RT:65001:100
```

with:

```text
RT:65001:200
```

This can be accomplished using a route map.

### Example Configuration

Create an extended community list:

```text
R1(config)# ip extcommunity-list standard OLD-RT permit rt 65001:100
```

Configure the route map:

```text
R1(config)# route-map REWRITE-RT permit 10
R1(config-route-map)# match extcommunity OLD-RT
R1(config-route-map)# set extcomm-list OLD-RT delete
R1(config-route-map)# set extcommunity rt 65001:200 additive

R1(config)# route-map REWRITE-RT permit 20
```

The route map:

1. Matches routes carrying RT `65001:100`.
2. Removes RT `65001:100`.
3. Adds RT `65001:200`.
4. Preserves other existing RTs through `additive`.
5. Permits other routes unchanged through sequence 20.

This type of policy can be applied to supported BGP VPNv4 import, export, or neighbor policy operations.

**Important:** Modifying a Route Target can change which VRFs import a route. Incorrect RT manipulation can accidentally leak routes between VPNs or prevent intended VPN connectivity.

---

## Advertising Extended Communities

Cisco IOS XE uses `neighbor ... send-community` to control community advertisement.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 send-community extended
```

This enables extended-community advertisement to the neighbor.

Alternatively:

```text
R1(config-router-af)# neighbor 10.0.12.2 send-community both
```

This enables both standard and extended community advertisement.

### Community Advertisement Options

| Command | Advertisement |
|---|---|
| `send-community` | Standard communities |
| `send-community standard` | Standard communities |
| `send-community extended` | Extended communities |
| `send-community both` | Standard and extended communities |

The command controls which community attributes are sent to the neighbor.

It does not override:

- BGP route advertisement eligibility.
- Outbound routing policies.
- Nontransitive extended-community restrictions.
- VPN-specific import and export policies.

In MPLS VPN deployments, verify the applicable VPN address-family neighbor configuration and platform defaults rather than assuming ordinary IPv4-unicast behavior.

---

## Extended Communities vs. Standard Communities

| Feature | Standard Communities | Extended Communities |
|---|---|---|
| Size | 32 bits | 64 bits |
| Path attribute code | 8 | 16 |
| Type information included? | No | Yes |
| Common representation | `65001:100` | `RT:65001:100` |
| Route tagging | Yes | Yes |
| Well-known advertisement restrictions | Yes | Not the same mechanism |
| VPN Route Targets | No | Yes |
| Site-of-Origin | No | Yes |
| Cost Community | No | Yes |
| Link Bandwidth | No | Yes |
| Set command | `set community` | `set extcommunity` |
| Match command | `match community` | `match extcommunity` |
| List command | `ip community-list` | `ip extcommunity-list` |

A route can simultaneously carry standard and extended communities.

For example:

```text
Community: 65001:100 no-export
Extended Community: RT:65001:200 SoO:65001:10
```

These are separate BGP attributes.

The standard communities and extended communities are processed according to their respective meanings and policies.

---

## Verification

### Display Extended Communities on a BGP Route

For an IPv4-unicast route:

```text
R1# show bgp ipv4 unicast 192.0.2.0/24
```

For a VPNv4 route:

```text
PE1# show bgp vpnv4 unicast all
```

To inspect a particular VPN route:

```text
PE1# show bgp vpnv4 unicast rd 65000:1 192.0.2.0/24
```

Depending on platform and route attributes, output may resemble:

```text
BGP routing table entry for 65000:1:192.0.2.0/24
Paths: (1 available, best #1)
  ...
  Extended Community: RT:65000:100 SoO:65000:10
```

The extended-community field identifies the attributes carried by the route.

### Display Extended Community Lists

```text
R1# show ip extcommunity-list
```

To display a particular list:

```text
R1# show ip extcommunity-list BLUE-RT
```

### Verify Route-Map Configuration

```text
R1# show route-map
R1# show route-map REWRITE-RT
```

### Verify VRF Route Targets

```text
PE1# show vrf detail A
```

This displays information about the VRF, including configured import and export RTs.

### Verify VPN Route Advertisements

```text
PE1# show bgp vpnv4 unicast all neighbors 10.0.12.2 advertised-routes
```

This helps determine which VPN routes are advertised toward the specified neighbor.

Inspect the detailed route on the receiving PE to verify the extended communities actually received.

---

## Troubleshooting

Common extended-community problems include:

| Problem | Possible Cause |
|---|---|
| Route Target missing | Export RT not configured or route policy removed it |
| VPN route not imported | Import RT does not match any RT carried by the route |
| Unexpected VPN route leaking | Incorrect or overly permissive import RT configuration |
| SoO not preventing route feedback | Inconsistent SoO values across PE connections to the same site |
| Cost Community not influencing best path | Incorrect POI, earlier best-path criterion, or feature disabled |
| Link Bandwidth not working | Missing `dmzlink-bw`, multipath configuration, or platform support |
| Extended communities missing on a neighbor | Extended-community advertisement not enabled or policy removed them |
| Extended community list not matching | Incorrect type (`rt` vs. `soo`), value, or regular expression |

### Troubleshooting Checklist

1. Verify that the route actually carries the expected extended community.
2. Confirm that the extended community is of the correct type.
3. Verify that it is propagated across the relevant BGP sessions.
4. Check whether the community is transitive or nontransitive.
5. Verify the route-map and extended community-list configuration.
6. For VPN routes, verify import and export RTs.
7. For SoO, confirm consistent site identification across multihomed connections.
8. For Cost Communities, verify the applicable POI and best-path criteria.
9. For Link Bandwidth, confirm BGP multipath eligibility and forwarding support.

---

## Key Points

- Extended BGP communities are **64-bit values** carried in the optional transitive EXTENDED_COMMUNITIES attribute (Type 16).
- Each extended community includes type information and an associated value.
- The Type field can be one or two bytes.
- Extended communities support Two-Octet AS Specific, IPv4 Address Specific, and Four-Octet AS Specific encodings.
- Individual extended-community types can be transitive or nontransitive.
- Route Targets control VPN route import and export.
- Route Distinguishers are not extended communities.
- Site-of-Origin identifies the customer site associated with a route and helps prevent routing loops.
- All PE connections to the same multihomed customer site should use the same SoO value.
- Cost Communities can directly influence BGP best-path selection.
- Cost Communities are nontransitive.
- Link Bandwidth extended communities support bandwidth-aware BGP multipath on compatible platforms.
- `set extcommunity rt` configures Route Targets.
- `set extcommunity soo` configures Site-of-Origin.
- `set extcommunity cost` configures Cost Communities.
- `ip extcommunity-list` defines extended community lists.
- `match extcommunity` matches routes based on an extended community list.
- `set extcomm-list ... delete` removes matching extended communities.
- `neighbor ... send-community extended` enables extended-community advertisement.
- Standard and extended communities are separate BGP attributes and can coexist on the same route.

---

## References

- [Cisco: Connecting to a Service Provider Using External BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-external-sp-0.html)
- [Cisco: BGP Command Reference, match extcommunity](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [Cisco: BGP Named Community Lists](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/configuration/xe-3e/irg-iproute-bgp-xe-3e-book/irg-named-community-list.html)
- [Cisco: MPLS VPN Route Target Rewrite](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/mp_l3_vpns/configuration/xe-3s/mp-l3-vpns-xe-3s-book/mp-vpn-rte-target-rw.html)
- [RFC 4360: BGP Extended Communities Attribute](https://datatracker.ietf.org/doc/html/rfc4360)
- [RFC 4364: BGP/MPLS IP Virtual Private Networks](https://datatracker.ietf.org/doc/html/rfc4364)
