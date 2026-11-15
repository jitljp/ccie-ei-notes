# BGP Best Path Selection Steps

The **BGP Best Path Selection Algorithm** compares eligible BGP paths to the same destination prefix using a defined sequence of criteria.

**The first applicable criterion that distinguishes two paths determines which path is preferred.** If the paths are equal at one step, BGP proceeds to the next.

Before beginning the comparison, BGP excludes ineligible paths, such as paths with an unreachable NEXT_HOP.

The 15 selection steps can be divided into four groups:

| Group | Steps | Category |
|---|---|---|
| 1 | 1-4 | Initial Preferences |
| 2 | 5-8 | Path Attributes |
| 3 | 9-11 | Path Type and Cost |
| 4 | 12-15 | Final Tie-Breakers |

---

## Group 1: Initial Preferences (Steps 1-4)

### 1. Cost Community: `pre-bestpath`

Prefer the path with the **lowest `pre-bestpath` Cost Community**, when applicable.

- Cost Communities contain a community ID and cost value.
- Lower cost values are preferred according to the applicable comparison rules.
- Paths without a relevant Cost Community receive a default comparison cost of 2147483647.
- This comparison takes place **before Weight and LOCAL_PREF**.

Configure a `pre-bestpath` Cost Community using a route map:

```text
R1(config)# route-map SET-COST permit 10
R1(config-route-map)# set extcommunity cost pre-bestpath 10 100

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-COST in
```

Disable Cost Community comparison:

```text
R1(config-router-af)# bgp bestpath cost-community ignore
```

See [BGP Cost Communities and Points of Insertion](cost-community.md).

### 2. Weight

Prefer the path with the **highest Weight**.

- Cisco-specific parameter, not a BGP path attribute.
- Local to the router and not advertised to neighbors.
- Default Weight for received routes is 0.
- Locally originated routes typically have Weight 32768.
- Range: 0-65535.

Configure Weight for all routes received from a neighbor:

```text
R1(config-router)# neighbor 10.0.12.2 weight 200
```

Alternatively, set Weight using an inbound route map:

```text
R1(config)# route-map SET-WEIGHT permit 10
R1(config-route-map)# set weight 200

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-WEIGHT in
```

A route map allows Weight to be modified for specific prefixes using match conditions.

### 3. LOCAL_PREF

Prefer the path with the **highest LOCAL_PREF**.

- Default LOCAL_PREF is 100.
- Propagated through IBGP within an AS.
- Normally not advertised to EBGP neighbors.
- Commonly used to influence outbound traffic selection throughout an AS.

Configure the default LOCAL_PREF:

```text
R1(config-router)# bgp default local-preference 200
```

Set LOCAL_PREF using an inbound route map:

```text
R1(config)# route-map SET-LOCAL-PREF permit 10
R1(config-route-map)# set local-preference 200

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-LOCAL-PREF in
```

LOCAL_PREF is evaluated after Weight, so a path with higher Weight is preferred even if its LOCAL_PREF is lower.

### 4. Local Origination

Prefer a **locally originated path** over a path learned from a BGP neighbor.

Among locally originated paths, Cisco normally prefers:

1. Routes originated using `network` or redistribution.
2. Routes originated using `aggregate-address`.

Originate a prefix using the `network` command:

```text
R1(config-router-af)# network 192.0.2.0 mask 255.255.255.0
```

The exact prefix must exist in the appropriate routing table.

Alternatively, redistribute routes into BGP:

```text
R1(config-router-af)# redistribute ospf 1
```

Originate an aggregate:

```text
R1(config-router-af)# aggregate-address 192.0.2.0 255.255.254.0
```

An eligible contributing BGP route is required for normal aggregate origination.

**Important:** A locally originated path is not automatically preferred over every received path. Earlier criteria, including Weight and LOCAL_PREF, can still influence the decision.

---

## Group 2: Path Attributes (Steps 5-8)

### 5. AIGP

Prefer the path with the **lowest accumulated IGP cost**, when AIGP comparison applies.

AIGP considers:

```text
Total AIGP Cost = Received AIGP Metric + IGP Metric to NEXT_HOP
```

- Evaluated before AS_PATH length.
- Can favor a path with a longer AS_PATH.
- By default, a path carrying AIGP is preferred over a path without AIGP.

Enable AIGP for a BGP neighbor:

```text
R1(config-router-af)# neighbor 10.0.12.2 aigp
```

Set a fixed AIGP metric using a route map:

```text
R1(config)# route-map SET-AIGP permit 10
R1(config-route-map)# set aigp-metric 100
```

Alternatively, use the metric of the source route:

```text
R1(config-route-map)# set aigp-metric igp-metric
```

The route map must be applied to an appropriate BGP origination or routing-policy operation.

Modify the AIGP comparison when only one path carries AIGP:

```text
R1(config-router)# bgp bestpath aigp ignore
```

**This command does not disable all AIGP comparisons.** If both paths contain AIGP, their metrics are still compared.

See [Accumulated IGP (AIGP)](../attributes/aigp.md).

### 6. AS_PATH Length

Prefer the path with the **shortest AS_PATH**.

- An AS_SET counts as one AS, regardless of the number of members.
- Confederation segments are excluded from AS_PATH length.
- AS_PATH prepending can make a path less preferred.

Configure AS_PATH prepending using an outbound route map:

```text
R1(config)# route-map PREPEND permit 10
R1(config-route-map)# set as-path prepend 65001 65001 65001

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map PREPEND out
```

This prepends AS 65001 three additional times to matching routes advertised to the neighbor.

Disable AS_PATH length comparison:

```text
R1(config-router)# bgp bestpath as-path ignore
```

**Note:** `bgp bestpath as-path multipath-relax` is different. It relaxes AS_PATH requirements for multipath rather than disabling AS_PATH comparison for normal best-path selection.

### 7. ORIGIN

Prefer the path with the **lowest ORIGIN value**.

| ORIGIN | Numeric Value | Preference |
|---|---|---|
| IGP (`i`) | 0 | Highest |
| EGP (`e`) | 1 | Second |
| Incomplete (`?`) | 2 | Lowest |

ORIGIN typically depends on how the prefix was introduced into BGP:

- `network`: IGP
- `redistribute`: Incomplete
- `aggregate-address`: Normally IGP

Modify ORIGIN using a route map:

```text
R1(config)# route-map SET-ORIGIN permit 10
R1(config-route-map)# set origin igp

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-ORIGIN in
```

Other supported values include:

```text
R1(config-route-map)# set origin incomplete
R1(config-route-map)# set origin egp 65002
```

**Important:** ORIGIN IGP does not mean the route was learned through an IGP. It identifies how the route was originated into BGP.

### 8. MED

Prefer the path with the **lowest MED**, when MED comparison is applicable.

- By default, MED is compared only between paths from the same neighboring AS.
- A missing MED is treated as 0 by default.
- MED is normally used to suggest a preferred entry point into an AS.

Configure MED using an outbound route map:

```text
R1(config)# route-map SET-MED permit 10
R1(config-route-map)# set metric 50

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-MED out
```

Configure BGP to compare MED across different neighboring ASs:

```text
R1(config-router)# bgp always-compare-med
```

Treat a missing MED as the worst value:

```text
R1(config-router)# bgp bestpath med missing-as-worst
```

Enable deterministic MED comparison:

```text
R1(config-router)# bgp deterministic-med
```

Enable MED comparison for applicable paths originated within a confederation:

```text
R1(config-router)# bgp bestpath med confed
```

**Important distinctions:**

- `bgp always-compare-med` allows comparison across different neighboring ASs.
- `bgp deterministic-med` organizes comparisons by neighboring AS but does not itself enable cross-AS MED comparison.

---

## Group 3: Path Type and Cost (Steps 9-11)

### 9. EBGP over IBGP

Prefer an **EBGP-learned path** over an **IBGP-learned path**.

This comparison takes place after MED and before the IGP metric to NEXT_HOP.

The peer type is determined by the BGP relationship.

For example, on R1 in AS 65001:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# neighbor 10.0.13.3 remote-as 65001
```

- `10.0.12.2` is an EBGP neighbor.
- `10.0.13.3` is an IBGP neighbor.

If both advertise equivalent paths and all earlier criteria are equal, R1 prefers the EBGP path.

**There is no ordinary BGP command that directly reverses this comparison.** To prefer an IBGP path, modify an earlier criterion such as Weight or LOCAL_PREF.

Paths learned through confederation EBGP sessions are treated as internal paths for this comparison.

### 10. IGP Metric to NEXT_HOP

Prefer the path with the **lowest IGP metric to its BGP NEXT_HOP**.

This is commonly called **hot-potato routing** or **closest-exit routing**.

For example:

| Path | IGP Metric to NEXT_HOP |
|---|---|
| Via R2 | 10 |
| Via R3 | 30 |

The path through R2 is preferred, assuming all earlier criteria are equal.

The metric depends on the routing information used to resolve the BGP NEXT_HOP.

For example, in OSPF, interface costs can be modified:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip ospf cost 10
```

This can change the IGP metric to a BGP NEXT_HOP reachable through that interface.

Alternatively, changing the BGP NEXT_HOP can influence the comparison:

```text
R2(config-router-af)# neighbor 10.0.12.1 next-hop-self
```

This command is configured on the advertising router, R2.

Verify the route used for next-hop resolution:

```text
R1# show ip route 10.10.10.10
```

**Important:** The IGP metric comparison uses the cost to the BGP NEXT_HOP, not the metric associated with the destination prefix.

### 11. Cost Community: `igp`

Prefer the path with the **lowest IGP Cost Community**, when applicable.

- Evaluated after the IGP metric to NEXT_HOP.
- The `igp` POI is the default for manually configured Cost Communities.
- Cost Communities are evaluated using their community IDs and cost values.

Configure an IGP Cost Community:

```text
R1(config)# route-map SET-COST permit 10
R1(config-route-map)# set extcommunity cost igp 10 100

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-COST in
```

The `igp` keyword can be omitted:

```text
R1(config-route-map)# set extcommunity cost 10 100
```

Disable Cost Community comparison:

```text
R1(config-router-af)# bgp bestpath cost-community ignore
```

See [BGP Cost Communities and Points of Insertion](cost-community.md).

---

## Group 4: Final Tie-Breakers (Steps 12-15)

These steps are evaluated only when the earlier criteria have not determined a preferred path.

Unlike Weight, LOCAL_PREF, and MED, most of these are not normally manipulated directly for traffic engineering.

### 12. Oldest EBGP Path

When both remaining paths are EBGP paths, prefer the **oldest path**.

This reduces unnecessary best-path changes when a newer path becomes available.

The comparison is skipped when:

- `bgp bestpath compare-routerid` is configured.
- Both paths have the same router ID.
- There is no existing best path.

Configure BGP to skip the oldest-path preference and compare router IDs:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp bestpath compare-routerid
```

With this command, BGP proceeds to the router ID comparison rather than retaining the older EBGP path solely because of its age.

**There is no command to manually set the age of a BGP path.**

### 13. Lowest BGP Router ID

Prefer the path associated with the **lowest BGP router ID**.

- Normally compares the router IDs of the routers that advertised the paths.
- If a path contains ORIGINATOR_ID, that value is used instead.
- This is particularly important in route reflector environments.

Configure the BGP router ID on the advertising router:

```text
R2(config)# router bgp 65002
R2(config-router)# bgp router-id 2.2.2.2
```

The router ID is normally selected automatically if it is not configured.

**Important:** This comparison uses the BGP router ID, not the BGP NEXT_HOP IP address or necessarily the BGP neighbor IP address.

Changing the router ID is not a recommended traffic-engineering technique because it can affect BGP sessions and other protocol behavior.

### 14. Shortest CLUSTER_LIST

Prefer the path with the **shortest CLUSTER_LIST**.

- CLUSTER_LIST is a BGP route reflector attribute.
- Route reflectors add their cluster IDs when reflecting routes.
- The attribute also helps prevent routing loops between route reflector clusters.

Configure a route reflector:

```text
R2(config)# router bgp 65001
R2(config-router)# neighbor 10.0.12.1 route-reflector-client
```

Configure its cluster ID:

```text
R2(config-router)# bgp cluster-id 2.2.2.2
```

The cluster ID is normally the route reflector's BGP router ID unless explicitly configured.

**There is no ordinary route-map command to set CLUSTER_LIST length.** Its length reflects the route's path through route reflector clusters.

The comparison is relevant only when earlier tie-breakers, including the router ID or ORIGINATOR_ID comparison, have not distinguished the paths.

### 15. Lowest Neighbor IP Address

Finally, prefer the path received from the **lowest BGP neighbor IP address**.

This is the neighbor address configured for the BGP session, not the NEXT_HOP path attribute.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# neighbor 10.0.13.3 remote-as 65003
```

If all previous selection criteria are equal:

| Path | Neighbor IP |
|---|---|
| Via R2 | 10.0.12.2 |
| Via R3 | 10.0.13.3 |

R1 prefers the path received from `10.0.12.2`.

**There is no dedicated BGP command to change this tie-breaker.** The neighbor address depends on the BGP session configuration.

To influence path selection predictably, use earlier criteria such as Weight or LOCAL_PREF instead.

---

## BGP Multipath Consideration

BGP multipath eligibility is evaluated after the IGP metric to NEXT_HOP comparison and before the final tie-breakers.

However, **multipath is not an additional best-path preference criterion**.

BGP still designates one best path, but additional eligible paths may be installed in the routing table.

Configure EBGP multipath:

```text
R1(config-router-af)# maximum-paths 4
```

Configure IBGP multipath:

```text
R1(config-router-af)# maximum-paths ibgp 4
```

Relax AS_PATH equality requirements for multipath:

```text
R1(config-router)# bgp bestpath as-path multipath-relax
```

See [BGP Load Balancing](../load-balancing.md).

---

## Verification

Display the BGP table:

```text
R1# show bgp ipv4 unicast
```

Display detailed best-path information for a prefix:

```text
R1# show bgp ipv4 unicast 192.0.2.0/24
```

This command is particularly useful because it displays the paths and their attributes, including:

- Weight
- LOCAL_PREF
- AS_PATH
- ORIGIN
- MED
- AIGP, when present
- Cost Communities, when present
- NEXT_HOP information
- Router ID, ORIGINATOR_ID, and CLUSTER_LIST where applicable

Verify the IGP metric to a BGP NEXT_HOP:

```text
R1# show ip route 10.10.10.10
```

Verify the installed route and forwarding information:

```text
R1# show ip route 192.0.2.0
R1# show ip cef 192.0.2.1
```

Verify routing policies and BGP configuration:

```text
R1# show route-map
R1# show running-config | section router bgp
```

**Remember:** A BGP best path is not necessarily installed in the IP routing table. RIB installation is a separate process.

See [BGP Path Selection and RIB Installation](rib-installation.md).

---

## References

- [Cisco: Select BGP Best-path Algorithm](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html)
- [Cisco: BGP Cost Community](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-n1.html)
- [Cisco: Configure the AIGP Metric Attribute for BGP](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/119001-configure-aigp-00.html)
- [Cisco: Configuring Internal BGP Features](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-int-features-0.html)
