# Origin and MED

**ORIGIN** and **Multi-Exit Discriminator (MED)** are BGP path attributes used during best path selection.

Both are evaluated after Weight, LOCAL_PREF, local origination, and AS_PATH length.

However, they serve different purposes:

- **ORIGIN:** Indicates how the route was originally introduced into BGP.
- **MED:** Suggests a preferred entry point into an autonomous system.

| Characteristic | ORIGIN | MED |
|---|---|---|
| Attribute type code | 1 | 4 |
| Category | Well-known mandatory | Optional non-transitive |
| Values | IGP, EGP, Incomplete | 0-4294967295 |
| Preferred value | IGP > EGP > Incomplete | Lowest |
| Selection order | Before MED | After ORIGIN |
| Advertised through IBGP? | Yes | Yes |
| Advertised through EBGP? | Yes | Yes, subject to propagation rules |
| Typical purpose | Identifies route origination method | Influences inbound traffic paths |

---

# ORIGIN Attribute

The **ORIGIN** attribute indicates how a route was originally introduced into BGP.

It is a well-known mandatory attribute, meaning that it must be recognized by all BGP implementations and included in conventional UPDATE messages advertising reachable routes.

There are three possible ORIGIN values:

| ORIGIN | Numeric Value | BGP Table Code | Preference |
|---|---|---|---|
| IGP | 0 | `i` | Most preferred |
| EGP | 1 | `e` | Second |
| Incomplete | 2 | `?` | Least preferred |

BGP prefers the **lowest numeric ORIGIN value**.

## ORIGIN IGP

ORIGIN IGP indicates that a route was originated within the BGP AS using a mechanism such as the `network` statement.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# network 192.0.2.0 mask 255.255.255.0
```

Assuming the exact prefix exists in the local routing table, BGP originates it with:

```text
ORIGIN: IGP (i)
```

### Important Distinction

**ORIGIN IGP does not mean that the route was learned through OSPF, EIGRP, or another IGP.**

It refers to the way the route was introduced into BGP.

For example, suppose R1 learns `192.0.2.0/24` through OSPF.

If R1 advertises the route into BGP using a `network` statement, the ORIGIN is still IGP.

```text
OSPF route
    |
    v
BGP network statement
    |
    v
ORIGIN = IGP (i)
```

Likewise, a connected or static route introduced using a `network` statement normally receives ORIGIN IGP.

### Aggregated Routes

Routes generated using `aggregate-address` also normally receive ORIGIN IGP.

For example:

```text
R1(config-router-af)# aggregate-address 192.0.2.0 255.255.254.0
```

The resulting aggregate normally has ORIGIN IGP.

However, aggregation options that preserve or derive attributes from contributing routes can affect the aggregate's attributes.

See [BGP Aggregation](../../scalability/aggregation/overview.md).

---

## ORIGIN EGP

ORIGIN EGP indicates that a route was originated using the original **Exterior Gateway Protocol (EGP)**.

EGP was a predecessor to BGP and is obsolete.

Modern BGP routers rarely originate routes with ORIGIN EGP naturally.

However, the value can still appear when explicitly configured through routing policy.

Example:

```text
R1(config)# route-map SET-ORIGIN permit 10
R1(config-route-map)# set origin egp 65001
```

The resulting ORIGIN code is:

```text
e
```

**Important:** ORIGIN EGP does not mean that a route was learned through EBGP.

An ordinary EBGP-learned route can have ORIGIN IGP, EGP, or Incomplete.

---

## ORIGIN Incomplete

ORIGIN Incomplete indicates that the original source of the route is not known to BGP.

It is commonly associated with routes introduced through **redistribution**.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# redistribute ospf 1
```

Routes redistributed from OSPF into BGP normally receive:

```text
ORIGIN: Incomplete (?)
```

This also generally applies to redistribution from other routing sources, including EIGRP, RIP, connected, and static routes.

### Network vs. Redistribution

Consider the same OSPF-learned prefix, `192.0.2.0/24`.

| BGP Origination Method | ORIGIN |
|---|---|
| `network 192.0.2.0 mask 255.255.255.0` | IGP (`i`) |
| `redistribute ospf 1` | Incomplete (`?`) |

The underlying route can be identical, but the BGP ORIGIN attribute differs.

**The ORIGIN attribute describes how the route entered BGP, not necessarily how the originating router learned the route.**

---

## Configuring ORIGIN with a Route Map

Use `set origin` to explicitly modify the ORIGIN attribute.

Syntax:

```text
R1(config-route-map)# set origin {igp | egp <asn> | incomplete}
```

### Modifying Received Routes

For example, change the ORIGIN of routes received from an EBGP neighbor to IGP:

```text
R1(config)# route-map SET-ORIGIN permit 10
R1(config-route-map)# set origin igp

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-ORIGIN in
```

All routes accepted through this route map receive ORIGIN IGP on R1.

This can influence local BGP best path selection.

### Modifying Selected Prefixes

To modify only a specific prefix:

```text
R1(config)# ip prefix-list PFX-ORIGIN permit 192.0.2.0/24

R1(config)# route-map SET-ORIGIN permit 10
R1(config-route-map)# match ip address prefix-list PFX-ORIGIN
R1(config-route-map)# set origin igp

R1(config)# route-map SET-ORIGIN permit 20

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-ORIGIN in
```

The result is:

- `192.0.2.0/24` receives ORIGIN IGP.
- Other routes retain their existing ORIGIN.
- The second route-map entry permits all other routes.

An outbound route map can also modify ORIGIN in routes advertised to a neighbor.

---

## ORIGIN in Best Path Selection

BGP prefers:

```text
IGP (0) > EGP (1) > Incomplete (2)
```

However, ORIGIN is evaluated only after higher-priority selection criteria.

### Example

R1 receives two paths to `192.0.2.0/24`.

| Attribute | Path A | Path B |
|---|---|---|
| Weight | 0 | 0 |
| LOCAL_PREF | 100 | 100 |
| AS_PATH | `65002 65004` | `65003 65004` |
| AS_PATH length | 2 | 2 |
| ORIGIN | IGP | Incomplete |
| MED | 200 | 10 |

Assuming both paths are eligible and no earlier criterion differs, **Path A is preferred**.

Although Path B has a lower MED, Path A has the more preferred ORIGIN.

BGP does not proceed to MED comparison.

---

# Multi-Exit Discriminator (MED)

**Multi-Exit Discriminator (MED)** is an optional non-transitive BGP attribute used to suggest a preferred entry point into an autonomous system.

It is also commonly called the **BGP metric**.

Important characteristics:

- Lower MED is preferred.
- MED is a 32-bit unsigned integer.
- The valid range is 0-4294967295.
- By default, Cisco treats a missing MED as 0 for best-path comparison.
- MED is evaluated after ORIGIN.
- By default, MED is compared only between paths received from the same neighboring AS.
- MED is typically used to influence **inbound traffic** from a neighboring AS.

Unlike LOCAL_PREF, which is used internally to select an AS's preferred exit, MED allows an AS to suggest which entry point another AS should use.

---

## MED Operation

Consider two autonomous systems connected by multiple EBGP links.

```text
                  AS 65001
                 Customer AS

             R1              R2
             |               |
             | EBGP          | EBGP
             |               |
             R3--------------R4
                   IBGP
                  AS 65002
                  Provider AS

       Destination: 192.0.2.0/24
```

AS 65001 advertises `192.0.2.0/24` through both R1 and R2.

Suppose:

- R1 advertises MED 50.
- R2 advertises MED 200.

R3 and R4 learn:

| Path | MED |
|---|---|
| Via R1 | 50 |
| Via R2 | 200 |

Assuming both paths are available at the selecting router, their higher-priority attributes are equal, and no policy overrides MED:

**AS 65002 prefers the path through R1 because its MED is lower.**

This is how MED can influence traffic entering AS 65001.

### MED vs. LOCAL_PREF

| Attribute | Primary Traffic Engineering Purpose |
|---|---|
| LOCAL_PREF | Influence outbound traffic from your AS |
| MED | Suggest a preferred inbound entry point to a neighboring AS |

MED is only a suggestion.

The receiving AS can override the MED preference using Weight, LOCAL_PREF, AS_PATH manipulation, or other routing policies.

---

## Configuring MED

### Using an Outbound Route Map

The most common approach is to configure MED using `set metric` in a route map.

For example, on R1:

```text
R1(config)# route-map SET-MED permit 10
R1(config-route-map)# set metric 50

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.13.3 route-map SET-MED out
```

R1 advertises matching routes with MED 50.

On R2:

```text
R2(config)# route-map SET-MED permit 10
R2(config-route-map)# set metric 200

R2(config)# router bgp 65001
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.24.4 route-map SET-MED out
```

R2 advertises matching routes with MED 200.

AS 65002 can now prefer the path through R1 when MED comparison applies.

### Applying MED to Specific Prefixes

To modify MED only for `192.0.2.0/24`:

```text
R1(config)# ip prefix-list PFX-MED permit 192.0.2.0/24

R1(config)# route-map SET-MED permit 10
R1(config-route-map)# match ip address prefix-list PFX-MED
R1(config-route-map)# set metric 50

R1(config)# route-map SET-MED permit 20

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.13.3 route-map SET-MED out
```

Only the matching prefix receives MED 50.

Other routes retain their existing MED attributes.

### Setting a Default MED

Cisco IOS XE supports `default-metric` under the BGP process.

```text
R1(config)# router bgp 65001
R1(config-router)# default-metric 100
```

This configures the default metric used for applicable BGP routes that do not already have a MED.

It does not unconditionally replace existing MED values.

For selective MED manipulation, an outbound route map is generally preferable.

---

## Setting MED Based on IGP Metric

Cisco IOS XE supports `set metric-type internal` in an outbound BGP route map.

```text
R1(config)# route-map SET-MED permit 10
R1(config-route-map)# set metric-type internal

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-MED out
```

This sets the advertised MED based on the **IGP metric associated with the route's next hop**.

For example:

| BGP Path | IGP Metric to Next Hop | Advertised MED |
|---|---|---|
| Path A | 10 | 10 |
| Path B | 50 | 50 |

This can allow the MED to reflect internal network costs instead of a manually configured value.

**Important:** `set metric-type internal` is used for outbound BGP advertisements. It is not supported as a way to set MED during route redistribution into BGP.

---

## Default MED Comparison Rules

By default, Cisco BGP compares MED only when the paths have the **same neighboring AS**.

More precisely, the comparison uses the first external ASN in the AS_SEQUENCE, ignoring preceding confederation segments.

### Example 1: Same Neighboring AS

R1 learns two paths:

| Attribute | Path A | Path B |
|---|---|---|
| AS_PATH | `65002 65010` | `65002 65020` |
| MED | 100 | 50 |

Both paths have the same neighboring AS:

```text
65002
```

Therefore, MED comparison is performed.

Assuming the preceding best-path criteria are equal:

**Path B is preferred because MED 50 is lower than MED 100.**

### Example 2: Different Neighboring AS

R1 learns:

| Attribute | Path A | Path B |
|---|---|---|
| AS_PATH | `65002 65010` | `65003 65020` |
| MED | 100 | 50 |

The neighboring ASNs differ:

```text
Path A: 65002
Path B: 65003
```

By default, **MED is not compared**.

BGP proceeds to later best-path criteria.

For example, if both paths are EBGP-learned and their earlier criteria are equal, the IGP metric to the BGP next hop may determine the preferred path.

### Important Distinction: Neighbor AS vs. BGP Peer AS

MED comparison is based on the **first external AS in AS_PATH**, not necessarily the ASN of the immediate BGP peer.

This distinction matters with IBGP.

For example, R1 receives two paths from different IBGP peers:

```text
Path A:
  Received from IBGP peer R2
  AS_PATH: 65010 65020
  MED: 100

Path B:
  Received from IBGP peer R3
  AS_PATH: 65010 65030
  MED: 50
```

Both paths have neighboring AS 65010.

Therefore, MED comparison can occur even though the routes came from different IBGP peers.

---

## Comparing MED Across Different ASNs

By default, MED values from different neighboring autonomous systems are not compared.

To change this behavior:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp always-compare-med
```

This allows BGP to compare MED across paths from different neighboring ASNs.

### Example

R1 receives:

| Attribute | Path A | Path B |
|---|---|---|
| AS_PATH | `65002 65010` | `65003 65020` |
| AS_PATH length | 2 | 2 |
| ORIGIN | IGP | IGP |
| MED | 100 | 50 |

Without `bgp always-compare-med`:

- MED is not compared because the neighboring ASNs differ.
- BGP proceeds to later criteria.

With `bgp always-compare-med`:

- MED is compared.
- **Path B is preferred** because MED 50 is lower.

This assumes earlier criteria are equal and that the comparison reaches MED.

### Design Considerations

MED values are not necessarily comparable between unrelated autonomous systems.

For example:

- ISP-A might use MED values between 0 and 100.
- ISP-B might use MED values between 1000 and 10000.

A lower MED from one ISP does not necessarily indicate a better path.

**Enabling `bgp always-compare-med` should be coordinated throughout the AS.**

Inconsistent MED comparison policies can produce inconsistent best-path decisions and potentially routing loops.

---

## Missing MED

MED is an optional BGP attribute, so an advertised path may not contain it.

By default, Cisco BGP treats a **missing MED as 0** during best-path comparison.

Because lower MED is preferred, a path without MED can be preferred over one with an explicitly configured positive MED.

### Example

| Path | Received MED | Effective MED |
|---|---|---|
| A | 100 | 100 |
| B | Not present | 0 |

Assuming MED comparison applies, **Path B is preferred**.

### Configuring Missing MED as Worst

Use:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp bestpath med missing-as-worst
```

This causes a missing MED to be treated as the worst possible value for MED comparison.

The effective value is implementation-dependent in some IOS versions, with Cisco documentation describing the maximum 32-bit value or the value immediately below it.

The important behavior is that a missing MED is treated as extremely undesirable rather than 0.

### Example

| Path | Received MED | Default Behavior | Missing-as-Worst |
|---|---|---|---|
| A | 100 | 100 | 100 |
| B | Not present | 0 | Worst |

- **Default:** Path B is preferred.
- **With missing-as-worst:** Path A is preferred.

This assumes both paths are compared at the MED step.

Restore the default:

```text
R1(config-router)# no bgp bestpath med missing-as-worst
```

**Important:** An explicitly advertised MED of 0 is different from an absent MED. The `missing-as-worst` command applies to the latter.

---

## Deterministic MED

By default, Cisco BGP does not necessarily perform MED comparisons by first grouping all paths from the same neighboring AS.

This can make the outcome of best-path selection depend on the order in which paths are received or evaluated.

This is particularly relevant when multiple paths exist from multiple autonomous systems.

### Problem with Non-Deterministic MED

Consider three paths to the same prefix:

| Path | Neighboring AS | MED |
|---|---|---|
| A | 65001 | 100 |
| B | 65002 | 50 |
| C | 65001 | 20 |

Assume higher-priority attributes are equal.

The important observations are:

- Path A and Path C are from the same neighboring AS, so their MEDs can be compared.
- Path B is from another AS, so its MED is not normally compared against A or C.
- Path C has a lower MED than Path A.

Without deterministic MED, the order in which the paths are evaluated can influence the final result.

For example, BGP might compare paths from different ASNs before it has compared every candidate from AS 65001.

This can cause path selection to change when route arrival order changes, even if the path attributes themselves have not changed.

### Configuring Deterministic MED

Use:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp deterministic-med
```

This ensures that BGP groups paths from the same neighboring AS and compares their MEDs before performing comparisons across different AS groups.

Using the previous example:

1. Group paths A and C because both originate from neighboring AS 65001.
2. Compare their MED values.
3. Prefer Path C over Path A because 20 is lower than 100.
4. Continue best-path selection using the preferred candidate from AS 65001 and the candidate from AS 65002.

The final best path depends on the remaining applicable BGP selection criteria.

### Deterministic MED vs. Always-Compare-MED

These commands perform different functions.

| Command | Behavior |
|---|---|
| `bgp deterministic-med` | Ensures deterministic MED comparison among paths from the same neighboring AS |
| `bgp always-compare-med` | Allows MED comparison between paths from different neighboring ASNs |

**`bgp deterministic-med` does not enable MED comparison between different neighboring ASNs.**

Likewise, `bgp always-compare-med` does not automatically enable `bgp deterministic-med`.

The commands can be configured independently or together.

---

## MED in BGP Confederations

BGP confederations divide an AS into multiple member ASNs while presenting a single external ASN.

Confederation paths can contain AS_CONFED_SEQUENCE and AS_CONFED_SET segments.

By default, Cisco does not compare MED between paths that contain only confederation AS_PATH segments according to the special confederation comparison rules.

To enable MED comparison for these paths:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp bestpath med confed
```

This allows MED comparison between paths originated within the local confederation.

### Example

Consider these paths:

```text
Path A:
  AS_PATH: (65002)
  MED: 100

Path B:
  AS_PATH: (65003)
  MED: 50
```

Both paths contain only AS_CONFED_SEQUENCE information.

With `bgp bestpath med confed`, BGP can compare their MED values.

Assuming earlier criteria are equal, Path B is preferred.

### Optional Missing-MED Handling

Cisco also supports:

```text
R1(config-router)# bgp bestpath med confed missing-as-worst
```

This applies worst-value handling to missing MED values in the relevant confederation comparison.

**Important:** The confederation MED comparison is distinct from ordinary MED comparison for external AS_PATH segments.

---

## MED Propagation Rules

Although MED is classified as an **optional non-transitive** attribute, that does not mean it is never propagated.

### EBGP to IBGP

When a router learns a route through EBGP and advertises that route through IBGP, the MED is normally preserved.

For example:

```text
        AS 65001                AS 65002

           R1 --------------------- R2 -------- R3
                    EBGP                    IBGP
```

R1 advertises a route with MED 50.

R2 receives:

```text
AS_PATH: 65001
MED: 50
```

When R2 advertises the route to R3 through IBGP, it normally preserves MED 50.

### EBGP to EBGP

When a router advertises an EBGP-learned route to a different external AS, the received MED is normally not propagated.

For example:

```text
      AS 65001              AS 65002              AS 65003

         R1 ------------------- R2 ------------------- R3
                  EBGP                  EBGP
```

R1 advertises MED 50 to R2.

R2 normally does not pass that MED unchanged to R3.

However, R2 can explicitly configure a new MED using an outbound routing policy.

### Propagation Summary

| Advertisement | MED Behavior |
|---|---|
| EBGP to IBGP | Normally preserved |
| IBGP to IBGP, when permitted | Normally preserved |
| EBGP to another external AS | Normally not propagated |
| Locally originated route to EBGP | Can advertise a configured MED |
| EBGP with outbound MED policy | Can set or replace the advertised MED |

---

## ORIGIN and MED Comparison

ORIGIN is evaluated before MED.

Consider three paths to `192.0.2.0/24`:

| Attribute | Path A | Path B | Path C |
|---|---|---|---|
| Weight | 0 | 0 | 0 |
| LOCAL_PREF | 100 | 100 | 100 |
| AS_PATH length | 2 | 2 | 2 |
| ORIGIN | IGP | IGP | Incomplete |
| MED | 100 | 50 | 0 |
| Neighboring AS | 65002 | 65002 | 65002 |

BGP first compares ORIGIN:

- Path A: IGP
- Path B: IGP
- Path C: Incomplete

Path C is less preferred, despite having the lowest MED.

BGP then compares Path A and Path B using MED.

**Path B is preferred because its MED is lower.**

This demonstrates why MED cannot override an earlier best-path criterion.

---

## Verification

### Display ORIGIN and MED

```text
R1# show bgp ipv4 unicast
R1# show bgp ipv4 unicast 192.0.2.0/24
```

Example:

```text
R1# show bgp ipv4 unicast

   Network          Next Hop            Metric LocPrf Weight Path
*  192.0.2.0/24     10.0.12.2              100             0 65002 i
*>                  10.0.13.3               50             0 65002 i
*                   10.0.14.4                0             0 65002 ?
```

The important fields are:

| Field | Meaning |
|---|---|
| Metric | MED |
| Path | AS_PATH and trailing ORIGIN code |
| `i` (at end of Path) | ORIGIN IGP |
| `e` (at end of Path) | ORIGIN EGP |
| `?` (at end of Path) | ORIGIN Incomplete |
| `>` | Best path |

In this example, the path with MED 50 is selected over the path with MED 100.

The third path has MED 0 but ORIGIN Incomplete, so it is less preferred than the paths with ORIGIN IGP.

**Do not confuse the `i` ORIGIN code at the end of AS_PATH with the `i` status code for an IBGP-learned route.**

They appear in different columns and have different meanings.

### Detailed Path Information

```text
R1# show bgp ipv4 unicast 192.0.2.0/24
```

The detailed output can display information such as:

```text
Origin IGP, metric 50, localpref 100, valid, external, best
```

Here:

- `Origin IGP` is the ORIGIN attribute.
- `metric 50` is the MED value.
- `localpref 100` is LOCAL_PREF.
- `external` indicates an EBGP-learned path.

### Display Advertised Routes

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 advertised-routes
```

Use this to examine the routes selected for advertisement to a neighbor.

For precise verification of the attributes actually transmitted, particularly after outbound routing-policy processing, inspect the neighbor's received route attributes or capture the BGP UPDATE messages.

### Verify Configuration

```text
R1# show running-config | section router bgp
R1# show route-map
R1# show bgp ipv4 unicast neighbors 10.0.12.2
```

### Reapply Routing Policy

After modifying an inbound BGP route map, use a soft inbound reset to reprocess routes:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

For an outbound policy change:

```text
R1# clear ip bgp 10.0.12.2 soft out
```

Soft resets avoid tearing down the BGP session. Inbound route refresh depends on peer capabilities or locally retained received routes.

---

## Key Takeaways

### ORIGIN

- ORIGIN is a **well-known mandatory** attribute.
- There are three ORIGIN values: IGP (0), EGP (1), and Incomplete (2).
- **IGP is preferred over EGP, which is preferred over Incomplete.**
- `network` statements normally generate ORIGIN IGP.
- Redistribution normally generates ORIGIN Incomplete.
- `set origin` can modify ORIGIN using a route map.
- ORIGIN IGP does not mean the route was learned through OSPF or another IGP.

### MED

- MED is an **optional non-transitive** attribute.
- **Lower MED is preferred.**
- MED is commonly used to influence inbound traffic from a neighboring AS.
- By default, MED is compared only between paths from the same neighboring AS.
- By default, missing MED is treated as 0.
- `set metric` configures MED in a route map.
- `default-metric` sets a default MED for applicable routes.
- `set metric-type internal` can set MED based on the IGP metric to the next hop.
- `bgp always-compare-med` enables comparison across different neighboring ASNs.
- `bgp deterministic-med` groups paths by neighboring AS to ensure deterministic MED comparison.
- `bgp bestpath med missing-as-worst` treats missing MED as the least preferred value.
- `bgp bestpath med confed` enables MED comparison for relevant confederation paths.
- MED does not override Weight, LOCAL_PREF, local origination, AS_PATH length, or ORIGIN.

---

## References

- [Cisco: Select BGP Best-path Algorithm](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html)
- [Cisco: Use MultiExit Discriminator to Influence BGP Best Path Selection](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13759-37.html)
- [Cisco: How the bgp deterministic-med Command Differs from the bgp always-compare-med Command](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/16046-bgp-med.html)
- [Cisco: BGP Configuration Guide, Understanding BGP](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr3-fwd/bgp/bgp-configuration-guide/routing-bgp.html)
- [Cisco: BGP Command Reference, Commands A through B](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-a1.html)
- [RFC 4271: A Border Gateway Protocol 4 (BGP-4)](https://www.rfc-editor.org/rfc/rfc4271.html)
