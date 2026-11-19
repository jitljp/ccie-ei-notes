# 1.5.d (ii) AS Path Prepending

**BGP AS_PATH prepending** is a routing-policy mechanism that artificially increases the length of the AS_PATH attribute by adding additional occurrences of an Autonomous System Number (ASN).

Its primary purpose is **inbound traffic engineering**, influencing which paths external autonomous systems prefer when sending traffic toward the local AS.

By default, BGP normally prefers the shortest AS_PATH when higher-priority best-path selection criteria are equal.

Therefore, making one path appear longer can encourage external networks to select another path.

AS_PATH prepending is commonly used to:

- Prefer one ISP for inbound traffic.
- Make a secondary ISP less attractive while retaining backup connectivity.
- Influence inbound traffic for selected prefixes.
- Implement different inbound traffic preferences for different destinations.
- Influence route selection without changing the actual network topology.

On Cisco IOS XE, AS_PATH prepending is primarily configured using the `set as-path prepend` command in a route map.

**Important:** AS_PATH prepending influences remote BGP best-path selection, but it does not guarantee that remote networks will select the desired path.

---

## AS_PATH Fundamentals

The AS_PATH attribute records the autonomous systems that a BGP route has traversed.

It is a **well-known mandatory, transitive** BGP path attribute.

When a router advertises a route to an EBGP neighbor, it normally prepends its own ASN to the AS_PATH.

Consider:

```text
       AS 65001           AS 65002           AS 65003

          R1 ---------------- R2 ---------------- R3
                 EBGP                EBGP

     192.0.2.0/24
```

R1 originates `192.0.2.0/24`.

The AS_PATH changes as follows:

| Router | AS_PATH |
|---|---|
| R1 | Empty |
| R2 | `65001` |
| R3 | `65002 65001` |

The most recently added ASN appears on the **left**.

### AS_PATH Length

Normally, each ASN occurrence in an AS_SEQUENCE contributes 1 to the AS_PATH length.

For example:

```text
Path A: 65001 65002
Path B: 65003 65004 65005
```

| Path | AS_PATH Length |
|---|---:|
| A | 2 |
| B | 3 |

If BGP reaches the AS_PATH length comparison during best-path selection, Path A is preferred.

Importantly, repeated occurrences of the same ASN count individually.

For example:

```text
Path A: 65001 65001 65001
Path B: 65002 65003
```

| Path | AS_PATH Length |
|---|---:|
| A | 3 |
| B | 2 |

Path B is preferred if earlier criteria are equal.

This is the principle behind AS_PATH prepending.

See [AS_PATH](../path-selection/attributes/as-path.md) for information about AS_SEQUENCE, AS_SET, confederation segments, and AS_PATH length calculation.

---

## How AS_PATH Prepending Works

AS_PATH prepending intentionally adds extra occurrences of an ASN to the AS_PATH.

For example, suppose a router belongs to AS 65001.

Normally, it advertises a locally originated route to an EBGP neighbor with:

```text
AS_PATH: 65001
```

The router can prepend its ASN three additional times.

The receiving neighbor sees:

```text
AS_PATH: 65001 65001 65001 65001
```

The AS_PATH length has increased from 1 to 4.

This makes the route appear to have traversed more autonomous systems than it actually has.

### Normal EBGP Prepending vs. Additional Prepending

The normal EBGP prepend is automatic.

For example:

```text
R1(config-route-map)# set as-path prepend 65001 65001 65001
```

This explicitly adds **three occurrences** of AS 65001.

When R1 advertises a locally originated route to an EBGP neighbor, normal EBGP processing also adds its own ASN.

Therefore, the neighbor receives:

```text
65001 65001 65001 65001
```

**The number of ASN occurrences configured in `set as-path prepend` does not include the normal EBGP prepend.**

| Configured Additional Prepends | AS_PATH Received by EBGP Neighbor |
|---|---|
| 0 | `65001` |
| 1 | `65001 65001` |
| 2 | `65001 65001 65001` |
| 3 | `65001 65001 65001 65001` |

This example assumes a locally originated route with an initially empty AS_PATH and no other AS_PATH manipulation.

---

## AS_PATH Prepending Configuration

Cisco IOS XE supports the following route-map command:

```text
R1(config-route-map)# set as-path prepend <ASN> [<ASN> ...]
```

For example:

```text
R1(config)# route-map PREPEND permit 10
R1(config-route-map)# set as-path prepend 65001 65001 65001
```

The command adds the specified ASN sequence to routes matched by the route map.

Cisco's command reference documents support for specifying up to **10 ASNs in one `set as-path prepend` command**.

Modern IOS XE also supports 4-byte ASNs.

For example:

```text
R1(config-route-map)# set as-path prepend 100000 100000
```

### Applying the Route Map

The route map is typically applied **outbound** toward an EBGP neighbor:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map PREPEND out
```

This modifies the AS_PATH of routes advertised to `10.0.12.2`.

The operation does not add the prepended ASNs to the local BGP table simply because the outbound policy is configured.

**Important:** An outbound BGP route map modifies route advertisements, not the router's own best-path selection.

---

## Inbound Traffic Engineering Example

Consider an enterprise connected to two ISPs.

The enterprise belongs to AS 65000.

ISP1 belongs to AS 65100.

ISP2 belongs to AS 65200.

A remote network belongs to AS 65300.

### Topology

```text
                      Remote AS
                       AS 65300
                       /      \
                      /        \
                   ISP1        ISP2
                 AS 65100    AS 65200
                    |           |
                    |           |
                    +-----R1----+
                        AS 65000
                           |
                    203.0.113.0/24
```

The enterprise advertises `203.0.113.0/24` through both ISPs.

The remote AS learns two paths to the prefix.

### Initial AS_PATHs

| Path | AS_PATH |
|---|---|
| Via ISP1 | `65100 65000` |
| Via ISP2 | `65200 65000` |

Both paths have an AS_PATH length of 2.

Assuming earlier BGP best-path criteria are equal, AS_PATH length does not determine the preferred path.

Other attributes must be evaluated.

### Objective

The enterprise wants to:

1. Prefer ISP1 for inbound traffic.
2. Make ISP2 less attractive.
3. Keep ISP2 available as a backup.

To accomplish this, R1 can prepend AS 65000 when advertising routes to ISP2.

### Configuration

Assume ISP2's EBGP neighbor address is `10.0.13.3`.

Configure:

```text
R1(config)# route-map PREPEND-ISP2 permit 10
R1(config-route-map)# set as-path prepend 65000 65000 65000

R1(config)# router bgp 65000
R1(config-router)# neighbor 10.0.13.3 remote-as 65200
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.13.3 route-map PREPEND-ISP2 out
```

This adds three additional occurrences of AS 65000 to all eligible routes advertised toward ISP2.

### Result

ISP1 receives the normal advertisement:

```text
AS_PATH: 65000
```

ISP2 receives:

```text
AS_PATH: 65000 65000 65000 65000
```

Each ISP prepends its own ASN when advertising the route to the remote AS.

The remote AS receives:

| Path | AS_PATH | Length |
|---|---|---:|
| Via ISP1 | `65100 65000` | 2 |
| Via ISP2 | `65200 65000 65000 65000 65000` | 5 |

If the comparison reaches AS_PATH length, the remote AS prefers ISP1.

Therefore, inbound traffic is more likely to enter the enterprise through ISP1.

### What Happens if ISP1 Fails?

If ISP1 withdraws the route, the remote AS may still have the path through ISP2.

Although ISP2's AS_PATH is longer, it remains a valid alternative.

Once the preferred path is withdrawn, the remote AS can select the remaining path through ISP2.

This allows AS_PATH prepending to support a primary/backup design without suppressing the backup route.

**Important:** AS_PATH prepending does not actually configure a backup link or detect failures. It only influences how external networks compare the routes they receive.

---

## Selective AS_PATH Prepending

A route map can prepend the AS_PATH only for selected prefixes.

This is useful when different destination networks should use different inbound paths.

### Example

The enterprise owns two prefixes:

```text
203.0.113.0/24
198.51.100.0/24
```

The objective is:

- Prefer ISP1 for traffic destined for `203.0.113.0/24`.
- Keep both ISP paths unchanged for `198.51.100.0/24`.

Configure a prefix list:

```text
R1(config)# ip prefix-list PREPEND-PFX permit 203.0.113.0/24
```

Configure the route map:

```text
R1(config)# route-map PREPEND-ISP2 permit 10
R1(config-route-map)# match ip address prefix-list PREPEND-PFX
R1(config-route-map)# set as-path prepend 65000 65000 65000

R1(config)# route-map PREPEND-ISP2 permit 20
```

Apply it outbound:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.13.3 route-map PREPEND-ISP2 out
```

### Result

| Prefix | AS_PATH Advertised to ISP2 |
|---|---|
| `203.0.113.0/24` | `65000 65000 65000 65000` |
| `198.51.100.0/24` | `65000` |

The first prefix receives three additional ASN occurrences.

The second prefix is advertised normally.

### Why Is Sequence 20 Needed?

The route map contains:

```text
route-map PREPEND-ISP2 permit 10
 match ip address prefix-list PREPEND-PFX
 set as-path prepend 65000 65000 65000
```

Without another permitting sequence, routes that do not match sequence 10 are denied by the route map's implicit deny.

For example, `198.51.100.0/24` would not be advertised through ISP2.

Adding:

```text
R1(config)# route-map PREPEND-ISP2 permit 20
```

Permits all other eligible routes without additional AS_PATH modification.

**Important:** An outbound BGP route map can unintentionally filter routes if the implicit deny is overlooked.

---

## Different Prepending Policies for Different Prefixes

Different prefixes can receive different numbers of additional ASNs.

For example, the enterprise wants to:

- Prepend AS 65000 once for `203.0.113.0/24`.
- Prepend AS 65000 three times for `198.51.100.0/24`.
- Advertise other eligible prefixes normally.

### Configuration

```text
R1(config)# ip prefix-list PFX-A permit 203.0.113.0/24
R1(config)# ip prefix-list PFX-B permit 198.51.100.0/24

R1(config)# route-map PREPEND-ISP2 permit 10
R1(config-route-map)# match ip address prefix-list PFX-A
R1(config-route-map)# set as-path prepend 65000

R1(config)# route-map PREPEND-ISP2 permit 20
R1(config-route-map)# match ip address prefix-list PFX-B
R1(config-route-map)# set as-path prepend 65000 65000 65000

R1(config)# route-map PREPEND-ISP2 permit 30
```

Apply the route map outbound toward ISP2:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.13.3 route-map PREPEND-ISP2 out
```

### Result

| Prefix | Additional Prepends | Advertised AS_PATH |
|---|---:|---|
| `203.0.113.0/24` | 1 | `65000 65000` |
| `198.51.100.0/24` | 3 | `65000 65000 65000 65000` |
| Other eligible prefixes | 0 | `65000` |

These examples assume locally originated routes with empty AS_PATHs before outbound processing.

This provides granular control over inbound traffic preferences.

---

## Outbound vs. Inbound AS_PATH Prepending

Although AS_PATH prepending is primarily used in outbound route maps, Cisco IOS XE also supports `set as-path prepend` in **inbound route maps**.

The direction determines whose path selection is influenced.

| Direction | Primary Effect |
|---|---|
| Outbound | Modify AS_PATHs advertised to neighbors |
| Inbound | Modify AS_PATHs accepted from neighbors |

### Outbound Prepending

Consider R1 in AS 65001 advertising a route to R2 in AS 65002.

```text
R1(config)# route-map PREPEND-OUT permit 10
R1(config-route-map)# set as-path prepend 65001 65001

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map PREPEND-OUT out
```

R2 receives the locally originated route with:

```text
AS_PATH: 65001 65001 65001
```

R1's local BGP table is not modified by this outbound operation.

### Inbound Prepending

Suppose R1 in AS 65001 receives a route from R2 in AS 65002.

R2 advertises:

```text
AS_PATH: 65002
```

R1 can prepend AS 65002 during inbound processing:

```text
R1(config)# route-map PREPEND-IN permit 10
R1(config-route-map)# set as-path prepend 65002 65002

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map PREPEND-IN in
```

R1 stores the modified AS_PATH:

```text
AS_PATH: 65002 65002 65002
```

The path now appears longer during R1's local best-path selection.

If R1 receives an alternative path to the same prefix, that alternative may be preferred.

Unlike Weight, an inbound AS_PATH modification may also be propagated to other BGP neighbors when the modified route is advertised.

### Why Inbound Prepending Is Less Common

Inbound AS_PATH prepending can influence the local router's best-path selection, but it is generally not the preferred way to implement outbound traffic engineering.

For outbound traffic engineering, administrators normally use:

- Weight, for preferences local to one Cisco router.
- LOCAL_PREF, for preferences throughout an autonomous system.

These attributes are evaluated before AS_PATH length and offer more direct control over local routing decisions.

**The standard use of AS_PATH prepending is outbound policy to influence inbound traffic.**

---

## Prepending the Last AS

Some Cisco platforms and software releases support the following command:

```text
R1(config-route-map)# set as-path prepend last-as <number>
```

Instead of explicitly specifying an ASN, the command automatically repeats the ASN at the beginning of the existing AS_PATH.

The `last-as` terminology can be confusing.

It refers to the last AS that advertised the route, which appears on the **left** in the displayed AS_PATH.

It does not mean the originating AS at the right end of the AS_PATH.

### Example

R1 receives a BGP route with:

```text
AS_PATH: 65002 65003 65004
```

Configure an inbound route map:

```text
R1(config)# route-map PREPEND-LAST-AS permit 10
R1(config-route-map)# set as-path prepend last-as 2
```

When applied inbound, the command repeats AS 65002 twice.

The resulting AS_PATH is:

```text
65002 65002 65002 65003 65004
```

The two additional ASN occurrences are placed at the beginning.

### last-as vs. Explicit ASN Prepending

These commands can produce the same result for a particular route:

```text
set as-path prepend 65002 65002
```

```text
set as-path prepend last-as 2
```

However, their behavior differs when routes have different AS_PATHs.

For example:

| Original AS_PATH | `prepend last-as 2` Result |
|---|---|
| `65002 65003` | `65002 65002 65002 65003` |
| `65004 65005` | `65004 65004 65004 65005` |
| `65006` | `65006 65006 65006` |

The command automatically chooses the ASN to repeat.

This can simplify inbound policies applied to routes received from different neighboring autonomous systems.

**Important:** Support for `last-as` varies among Cisco IOS XE, NX-OS, and other Cisco software implementations. Verify the syntax with `set as-path prepend ?` on the target image.

Because `last-as` depends on an existing AS_PATH, explicit local-AS prepending is more appropriate for ordinary outbound advertisements of locally originated routes.

---

## AS_PATH Prepending with Route-Map Continue

Cisco IOS XE supports the `continue` keyword in BGP route maps.

A `continue` clause allows a matching route to continue through additional route-map sequences.

The `set as-path prepend` command is an **additive operation**.

This means that ASNs prepended by multiple route-map sequences can accumulate when `continue` is used.

### Example

R1 belongs to AS 65000.

Configure:

```text
R1(config)# route-map PREPEND permit 10
R1(config-route-map)# set as-path prepend 65000
R1(config-route-map)# continue 20

R1(config)# route-map PREPEND permit 20
R1(config-route-map)# set as-path prepend 65000 65000
```

Apply the route map outbound toward an EBGP neighbor:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map PREPEND out
```

### Result

All routes matched by sequence 10 are processed as follows:

1. Sequence 10 prepends AS 65000 once.
2. `continue 20` moves processing to sequence 20.
3. Sequence 20 prepends AS 65000 twice more.
4. The route is advertised with three additional occurrences.

Including the normal EBGP prepend, a locally originated route is advertised with:

```text
65000 65000 65000 65000
```

### Additive vs. Overwriting Set Commands

Not all route-map `set` commands behave identically with `continue`.

| Set Command Type | Behavior |
|---|---|
| `set as-path prepend` | Additional values accumulate |
| `set community additive` | Additional communities accumulate |
| `set local-preference` | Later applicable value can overwrite an earlier value |
| `set metric` | Later applicable value can overwrite an earlier value |

Therefore, multiple `set as-path prepend` operations can result in a longer AS_PATH than expected.

**Important:** In a BGP route map using `continue`, prepend operations from multiple successfully processed sequences are additive.

See [BGP Prefix Lists, AS-Path ACLs, and Route Maps](../routing-policies/filtering-tools.md) for route-map operation.

---

## AS_PATH Prepending and BGP Best-Path Selection

AS_PATH length is only one criterion in the BGP best-path selection algorithm.

On Cisco IOS XE, earlier criteria may include:

1. Cost Community with a pre-bestpath point of insertion, when configured.
2. Weight.
3. LOCAL_PREF.
4. Local origination.
5. AIGP, when applicable.
6. AS_PATH length.

The exact sequence depends on the platform and configured BGP features.

### Example: LOCAL_PREF Overrides AS_PATH Length

A remote router receives two paths:

| Attribute | Path via ISP1 | Path via ISP2 |
|---|---:|---:|
| Weight | 0 | 0 |
| LOCAL_PREF | 100 | 200 |
| AS_PATH length | 2 | 5 |
| ORIGIN | IGP | IGP |

Although ISP1 has a shorter AS_PATH, the router prefers ISP2.

This is because LOCAL_PREF is evaluated before AS_PATH length.

The higher LOCAL_PREF of 200 determines the best path.

**AS_PATH prepending cannot override a higher-priority BGP best-path criterion.**

### Ignoring AS_PATH Length

Cisco IOS XE supports:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp bestpath as-path ignore
```

This instructs BGP to skip the AS_PATH length comparison during best-path selection.

For example:

```text
Path A: 65002
Path B: 65003 65003 65003
```

Normally, Path A is preferred if earlier criteria are equal.

With `bgp bestpath as-path ignore`, the router skips this comparison and continues evaluating other attributes.

This does not remove the AS_PATH or disable AS_PATH loop detection.

**If a remote AS ignores AS_PATH length, prepending cannot influence that router through the AS_PATH length criterion.**

---

## AS_PATH Prepending vs. MED

AS_PATH prepending and MED are both commonly used for inbound traffic engineering.

However, they operate differently.

| Characteristic | AS_PATH Prepending | MED |
|---|---|---|
| Attribute modified | AS_PATH | MED |
| Preferred value | Shorter AS_PATH | Lower MED |
| Typical scope | Potentially multiple external ASes | Primarily a directly neighboring AS |
| Common use case | Multiple ISPs | Multiple links to the same ISP |
| Configuration | `set as-path prepend` | `set metric` |
| Normal selection order | Before MED | After AS_PATH |
| Guaranteed result? | No | No |

### When to Prefer AS_PATH Prepending

AS_PATH prepending is useful when:

- An enterprise is connected to different ISPs.
- Routes are propagated through multiple external autonomous systems.
- The enterprise wants one ISP to appear less attractive.

### When to Prefer MED

MED is useful when:

- The enterprise has multiple connections to the same ISP.
- It wants to suggest which connection the ISP should use.
- The ISP compares the MED values of the advertised routes.

By default, BGP normally compares MED only between paths received from the same neighboring AS.

See [Origin and MED](../path-selection/attributes/origin-med.md).

---

## AS_PATH Prepending vs. BGP Communities

Some service providers support BGP communities that request specific routing policies.

For example, a provider might support communities that instruct it to:

- Lower LOCAL_PREF for a route.
- Prepend its own ASN toward selected upstream providers.
- Prepend its ASN toward specific geographic regions.
- Suppress advertisements toward selected peers.

These capabilities depend on the provider's community policy.

### Why Provider Communities Can Be More Effective

Suppose an enterprise prepends its ASN toward ISP2.

The route through ISP2 has a longer AS_PATH than the route through ISP1.

However, ISP2 may still assign a higher LOCAL_PREF to the route inside its own AS.

The higher LOCAL_PREF can override the longer AS_PATH.

If ISP2 supports a community requesting lower LOCAL_PREF, the enterprise may be able to influence the provider's path selection more directly.

Provider communities can also request **provider-side AS_PATH prepending**.

This differs from the enterprise prepending its own ASN toward the provider.

With provider-side prepending, the provider may add its own ASN when advertising routes toward selected external networks.

This can provide more granular control over which external ASes see a longer AS_PATH.

See [Inbound Traffic Engineering](../routing-policies/multihoming/inbound.md) and [BGP Communities](../routing-policies/communities/overview.md).

---

## Limitations and Considerations

AS_PATH prepending is a useful traffic engineering tool, but several limitations must be understood.

### No Guarantee of Remote Path Selection

External ASes independently select their preferred routes.

They may use:

- LOCAL_PREF.
- Weight.
- AS_PATH filtering.
- Routing policies.
- Other best-path criteria.

Therefore, a longer AS_PATH does not guarantee that a route will be less preferred.

### Does Not Change the Physical Path

AS_PATH prepending does not add physical routers or network hops.

For example:

```text
65000 65000 65000 65000
```

Does not mean that packets traverse four routers in AS 65000.

The repeated ASNs are artificial additions to a BGP path attribute.

### Does Not Control Outbound Traffic Directly

Outbound AS_PATH prepending primarily affects how other autonomous systems select routes toward the enterprise.

It does not directly control how the enterprise selects its own outbound paths.

Use Weight or LOCAL_PREF for that purpose.

### Does Not Perform Load Balancing

AS_PATH prepending does not directly distribute traffic across multiple links.

It can influence which external paths are preferred, but it cannot guarantee a specific traffic ratio.

For example, prepending once toward ISP2 and three times toward ISP3 does not imply any particular percentage of traffic will use either ISP.

### Excessive Prepending

Excessive AS_PATH prepending can cause operational problems.

Some networks enforce maximum AS_PATH lengths or apply policies that reject unusually long AS_PATHs.

For example, Cisco IOS XE supports:

```text
R1(config-router)# bgp maxas-limit 10
```

A router using this feature can reject received routes whose AS_PATH exceeds the configured limit.

Therefore, excessive prepending can cause routes to be rejected rather than merely made less attractive.

### Longest-Prefix Matching

AS_PATH prepending cannot override longest-prefix matching during IP forwarding.

For example, a router learns:

```text
203.0.113.0/24
203.0.113.0/25
```

Traffic destined for an address covered by the `/25` uses the `/25` route, assuming it is installed in the forwarding table.

This is true even if the `/25` has a longer AS_PATH than the `/24`.

BGP compares AS_PATHs when selecting between paths to the **same prefix**.

It does not compare different prefix lengths as competing BGP paths to one destination.

---

## Verification and Troubleshooting

Several commands are useful for verifying AS_PATH prepending.

### Verify the Route Map

```text
R1# show route-map PREPEND-ISP2
```

This displays the configured route-map sequences, match conditions, and set actions.

For example:

```text
route-map PREPEND-ISP2, permit, sequence 10
  Match clauses:
    ip address prefix-lists: PREPEND-PFX
  Set clauses:
    as-path prepend 65000 65000 65000
```

The exact output varies by IOS XE release.

### Verify the BGP Configuration

```text
R1# show running-config | section router bgp
```

Confirm that the correct route map is applied to the intended neighbor.

For example:

```text
neighbor 10.0.13.3 route-map PREPEND-ISP2 out
```

### Inspect the Local BGP Table

```text
R1# show ip bgp
R1# show ip bgp 203.0.113.0
```

An outbound prepend normally does not modify the original path stored in the local BGP table.

For a locally originated prefix, R1 may still show an empty AS_PATH.

This is expected.

### Inspect Advertised Routes

```text
R1# show ip bgp neighbors 10.0.13.3 advertised-routes
```

This command identifies routes advertised toward ISP2.

However, on some IOS XE implementations, the displayed AS_PATH may not reflect every final outbound transformation.

**The most reliable way to verify the actual AS_PATH is to inspect the received route on the neighboring BGP router.**

### Verify the Received AS_PATH

On ISP2:

```text
ISP2# show ip bgp 203.0.113.0
```

If the route was prepended three additional times by AS 65000, ISP2 should receive:

```text
65000 65000 65000 65000
```

### Reapply Outbound Policies

After changing the outbound route map, use:

```text
R1# clear ip bgp 10.0.13.3 soft out
```

This reprocesses outbound BGP advertisements without normally resetting the established BGP session.

For an inbound policy change:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

This can trigger Route Refresh when supported by the neighbor.

---

## Common Troubleshooting Scenarios

### More ASNs Than Expected

**Problem:** The route map contains:

```text
set as-path prepend 65000 65000 65000
```

But the EBGP neighbor receives:

```text
65000 65000 65000 65000
```

**Explanation:** The configured command adds three occurrences, and normal EBGP processing adds one more.

This is expected behavior for a locally originated route.

### Other Prefixes Disappear

**Problem:** After applying a selective prepend route map, some prefixes are no longer advertised.

**Cause:** The route map contains only a permitting sequence for the selected prefix.

For example:

```text
route-map PREPEND permit 10
 match ip address prefix-list PFX-A
 set as-path prepend 65000 65000
```

Other prefixes are rejected by the implicit deny.

**Solution:** Add a final permit sequence if those other routes should remain eligible for advertisement.

```text
R1(config)# route-map PREPEND permit 20
```

### Prepending Does Not Change Traffic Flow

**Problem:** ISP2 remains the preferred inbound path despite additional AS_PATH prepending.

**Possible causes:**

- External networks assign a higher LOCAL_PREF to ISP2.
- The remote AS does not receive the alternative path.
- The remote AS ignores AS_PATH length during best-path selection.
- Another attribute determines the best path before AS_PATH length.
- More-specific advertisements attract traffic through ISP2.
- The updated BGP advertisements have not propagated.

**Troubleshooting:** Inspect the actual paths and best-path reasons on a router in the external AS, when possible.

### AS_PATH Is Longer Than Expected with continue

**Problem:** The route map contains multiple `set as-path prepend` commands in different sequences.

**Cause:** The route map uses `continue`, and prepend operations accumulate.

**Solution:** Inspect every sequence that can be processed by the route map.

---

## Command Summary

| Command | Purpose |
|---|---|
| `set as-path prepend <ASN> [<ASN> ...]` | Prepend specified ASN occurrences |
| `set as-path prepend last-as <N>` | Repeat the leftmost ASN N times, where supported |
| `match ip address prefix-list <NAME>` | Match selected prefixes |
| `neighbor <IP> route-map <NAME> out` | Apply an outbound routing policy |
| `neighbor <IP> route-map <NAME> in` | Apply an inbound routing policy |
| `continue <SEQ>` | Continue route-map processing at another sequence |
| `bgp bestpath as-path ignore` | Skip AS_PATH length during best-path selection |
| `bgp maxas-limit <N>` | Limit the accepted AS_PATH length |
| `show route-map <NAME>` | Verify route-map configuration |
| `show ip bgp <PREFIX>` | Inspect a BGP route and its AS_PATH |
| `show ip bgp neighbors <IP> advertised-routes` | Inspect advertised routes |
| `clear ip bgp <IP> soft out` | Reapply outbound routing policy |

---

## Key Points

- AS_PATH prepending artificially increases AS_PATH length.
- Repeated occurrences of the same ASN count individually toward AS_PATH length.
- Its primary purpose is **inbound traffic engineering**.
- It is usually configured using an **outbound BGP route map**.
- The command is `set as-path prepend <ASN> [<ASN> ...]`.
- The configured ASN occurrences are **additional** to normal EBGP prepending.
- Cisco documents up to 10 ASNs in a single `set as-path prepend` command.
- AS_PATH prepending can be applied selectively using prefix lists and route maps.
- An outbound route map without a final permit sequence may unintentionally filter other routes.
- Cisco IOS XE also supports AS_PATH prepending in inbound route maps.
- Inbound prepending modifies the received AS_PATH and can influence local best-path selection.
- The `last-as` option, where supported, repeats the leftmost ASN in the existing AS_PATH.
- `set as-path prepend` is an additive operation when combined with `route-map continue`.
- Outbound AS_PATH prepending does not normally modify the original AS_PATH stored in the local BGP table.
- AS_PATH length is evaluated after higher-priority criteria such as Weight and LOCAL_PREF.
- AS_PATH prepending does not guarantee remote best-path selection.
- AS_PATH prepending does not directly control outbound traffic or provide load balancing.
- Excessively long AS_PATHs may be rejected by remote routing policies.
- Longest-prefix matching is independent of AS_PATH length.
- **Verify the final AS_PATH on the receiving BGP router**, not only on the advertising router.

---

## References

- [Cisco IOS XE 17.x: Connecting to a Service Provider Using External BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-external-sp-0.html)
- [Cisco: BGP Command Reference, set as-path](https://www.cisco.com/c/en/us/td/docs/ios/iproute_bgp/command/reference/irg_book/irg_bgp4.html)
- [Cisco IOS XE 17.x: BGP Route-Map Continue Support for Outbound Policy](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-route-map-continue-outbound.html)
- [Cisco IOS XE: BGP 4 Prefix Filter and Inbound Route Maps](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/configuration/xe-3e/irg-iproute-bgp-xe-3e-book/irg-prefix-filter.html)
- [RFC 4271: A Border Gateway Protocol 4 (BGP-4)](https://www.rfc-editor.org/rfc/rfc4271.html)
