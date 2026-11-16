# 1.5.c (i) Attribute Manipulation

**BGP attribute manipulation** allows administrators to modify path attributes to influence routing decisions and control how routes are advertised to other BGP speakers.

On Cisco IOS XE, BGP attributes are commonly modified using **route maps** containing `set` commands.

Attribute manipulation can be used to:

- Influence local BGP best-path selection.
- Control preferred exit points from an autonomous system.
- Influence how external autonomous systems select paths.
- Modify NEXT_HOP information.
- Tag routes with communities for policy-based processing.

Unlike commands that modify the BGP best-path algorithm, attribute manipulation changes the **values that the algorithm evaluates**.

See [BGP Best Path Selection Configuration](../path-selection/best-path/configuration.md).

---

## BGP Attribute Manipulation Overview

The following BGP attributes can be manipulated using routing policies.

| Attribute | Route-Map Command | Common Purpose |
|---|---|---|
| Weight | `set weight` | Influence local path selection |
| LOCAL_PREF | `set local-preference` | Influence outbound path selection throughout an AS |
| AS_PATH | `set as-path prepend` | Influence remote path selection |
| ORIGIN | `set origin` | Modify the route's origin type |
| MED | `set metric` | Suggest a preferred entry point into an AS |
| MED (based on IGP) | `set metric-type internal` | Derive advertised MED from IGP cost |
| NEXT_HOP | `set ip next-hop` | Modify the BGP next-hop address |
| Standard Community | `set community` | Tag routes for routing policies |
| Extended Community | `set extcommunity` | Set extended attributes such as route targets and Cost Communities |
| AIGP | `set aigp-metric` | Set the accumulated IGP metric attribute |

Not all attributes are propagated to all neighbors.

For example:

- **Weight** is local to a Cisco router and is never advertised.
- **LOCAL_PREF** is propagated through IBGP but not ordinary EBGP.
- **MED** can be advertised to EBGP neighbors, but is not normally propagated unchanged across multiple external AS boundaries.
- **Communities** can be propagated when community advertisement is enabled and the relevant policy permits them.

---

## Inbound vs. Outbound Attribute Manipulation

Attribute manipulation can be performed in either direction.

### Inbound Attribute Manipulation

Inbound route maps modify attributes of routes received from a BGP neighbor.

```text
         BGP Neighbor
              |
              | UPDATE
              v
      +----------------+
      | Inbound        |
      | Route Map      |
      +----------------+
              |
              | Modified attributes
              v
          BGP Table
              |
              v
       Best-Path Selection
```

Inbound policies are commonly used to manipulate:

- Weight
- LOCAL_PREF
- NEXT_HOP
- ORIGIN
- MED
- Communities

For example:

```text
R1(config)# route-map ISP-IN permit 10
R1(config-route-map)# set local-preference 200

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map ISP-IN in
```

All accepted routes from the neighbor receive LOCAL_PREF 200.

This affects the local BGP selection process and can influence other routers in the AS when the routes are propagated through IBGP.

### Outbound Attribute Manipulation

Outbound route maps modify attributes of routes advertised to a BGP neighbor.

```text
          BGP Table
              |
              v
       Advertisement
         Selection
              |
              v
      +----------------+
      | Outbound       |
      | Route Map      |
      +----------------+
              |
              | Modified attributes
              v
         BGP Neighbor
```

Outbound policies are commonly used to manipulate:

- AS_PATH
- MED
- ORIGIN
- NEXT_HOP
- Communities

For example:

```text
R1(config)# route-map ISP-OUT permit 10
R1(config-route-map)# set metric 100

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map ISP-OUT out
```

This sets MED to 100 for routes permitted by the outbound route map.

**Important:** Modifying an attribute in an outbound route map does not directly change the attribute stored in the local BGP table for best-path selection.

---

## Manipulating Weight

**Weight** is a Cisco-specific BGP selection parameter.

- Higher Weight is preferred.
- Default Weight for learned routes is 0.
- Default Weight for locally originated routes is 32768.
- Weight is evaluated before LOCAL_PREF.
- Weight is never included in BGP UPDATE messages.

### Configuring Weight with a Route Map

```text
R1(config)# ip prefix-list PREFERRED permit 203.0.113.0/24

R1(config)# route-map SET-WEIGHT permit 10
R1(config-route-map)# match ip address prefix-list PREFERRED
R1(config-route-map)# set weight 500

R1(config)# route-map SET-WEIGHT permit 20

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-WEIGHT in
```

The result is:

- `203.0.113.0/24` receives Weight 500.
- Other routes retain their normal Weight values.
- All routes are permitted by the route map.

### Neighbor-Level Weight

Weight can also be configured directly on a neighbor:

```text
R1(config-router-af)# neighbor 10.0.12.2 weight 500
```

This assigns Weight 500 to routes received from the neighbor.

If both neighbor-level Weight and an inbound route map using `set weight` apply, **the route-map Weight takes precedence**.

Because Weight is locally significant, it is useful for influencing outbound traffic from one router, but not for implementing a consistent AS-wide policy.

See [Weight and Local Preference](../path-selection/attributes/weight-local-preference.md).

---

## Manipulating LOCAL_PREF

**LOCAL_PREF** is used to influence preferred exit paths throughout an autonomous system.

- Higher LOCAL_PREF is preferred.
- Default LOCAL_PREF is 100.
- LOCAL_PREF is evaluated after Weight.
- LOCAL_PREF is propagated through IBGP.
- LOCAL_PREF is not advertised to ordinary EBGP peers.

### Example: Prefer One ISP

Consider:

```text
     ISP1                       ISP2
   AS 65100                   AS 65200
       |                         |
       | EBGP                    | EBGP
       |                         |
       R1 -------- R3 --------- R2
                   IBGP
                 AS 65000
```

Both ISPs advertise `203.0.113.0/24`.

The goal is to prefer ISP1 for outbound traffic throughout AS 65000.

On R1:

```text
R1(config)# route-map ISP1-IN permit 10
R1(config-route-map)# set local-preference 200

R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map ISP1-IN in
```

Assuming routes received from ISP2 retain LOCAL_PREF 100:

| Path | LOCAL_PREF |
|---|---|
| Via ISP1 | 200 |
| Via ISP2 | 100 |

The ISP1 path is preferred when earlier selection criteria are equal.

R1 can also advertise the route through IBGP with LOCAL_PREF 200, allowing other routers in AS 65000 to make the same preference decision.

### Changing Default LOCAL_PREF

The default value can be changed for the local BGP process:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp default local-preference 150
```

This changes the default LOCAL_PREF from 100 to 150 for applicable paths that do not have an explicitly assigned value.

For selective traffic engineering, inbound route maps are generally preferred.

See [Weight and Local Preference](../path-selection/attributes/weight-local-preference.md).

---

## Manipulating AS_PATH

**AS_PATH** records the autonomous systems a route has traversed.

BGP normally prefers the shortest AS_PATH after higher-priority selection criteria have been evaluated.

AS_PATH manipulation is commonly used to influence inbound traffic from external networks.

### AS_PATH Prepending

Use `set as-path prepend` to add additional ASN occurrences to the AS_PATH.

```text
R1(config)# route-map PREPEND permit 10
R1(config-route-map)# set as-path prepend 65001 65001

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map PREPEND out
```

For a locally originated route, the AS_PATH advertised to the EBGP neighbor normally becomes:

```text
65001 65001 65001
```

Two copies are added by the route map, and one copy is added during normal EBGP advertisement.

A remote router may consider this path less attractive than an alternative with a shorter AS_PATH.

However, **AS_PATH prepending does not guarantee that remote networks will select a different path**.

Their own LOCAL_PREF and other routing policies can override AS_PATH length.

### Selective AS_PATH Prepending

To prepend only specific prefixes:

```text
R1(config)# ip prefix-list BACKUP permit 192.0.2.0/24

R1(config)# route-map PREPEND permit 10
R1(config-route-map)# match ip address prefix-list BACKUP
R1(config-route-map)# set as-path prepend 65001 65001 65001

R1(config)# route-map PREPEND permit 20
```

Only matching routes are prepended.

Other routes are advertised without additional route-map prepending.

See [AS Path Prepending](../as-path/prepending.md) for additional options and traffic-engineering examples.

---

## Manipulating ORIGIN

The **ORIGIN** attribute identifies how a route was introduced into BGP.

BGP supports three values:

| ORIGIN | Code | Preference |
|---|---|---|
| IGP | `i` | Most preferred |
| EGP | `e` | Second |
| Incomplete | `?` | Least preferred |

Use `set origin` to modify the ORIGIN attribute.

### Syntax

```text
R1(config-route-map)# set origin {igp | egp <asn> | incomplete}
```

### Example

```text
R1(config)# route-map SET-ORIGIN permit 10
R1(config-route-map)# set origin igp

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-ORIGIN in
```

All accepted routes from the neighbor receive ORIGIN IGP.

The original ORIGIN value is replaced.

An outbound route map can also modify ORIGIN in advertised routes.

### Important Distinction

Changing ORIGIN does not change how the router actually learned the route.

For example, a route originally introduced through redistribution may have ORIGIN Incomplete.

Applying `set origin igp` changes the attribute to IGP, but the route was still introduced through redistribution.

See [Origin and MED](../path-selection/attributes/origin-med.md).

---

## Manipulating MED

**Multi-Exit Discriminator (MED)** is used to suggest a preferred entry point into an autonomous system.

- Lower MED is preferred.
- MED is an optional non-transitive BGP attribute.
- By default, Cisco compares MED only between paths from the same neighboring AS, subject to additional configuration.
- MED is evaluated after AS_PATH length and ORIGIN in the normal best-path algorithm.

MED is commonly configured in outbound route maps applied to EBGP neighbors.

### Setting a Fixed MED

```text
R1(config)# route-map SET-MED permit 10
R1(config-route-map)# set metric 100

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-MED out
```

The advertised routes receive MED 100.

Although the command uses the keyword `metric`, it modifies the BGP MED attribute when used in this context.

### MED-Based Inbound Traffic Engineering

Consider two connections between the same autonomous systems:

```text
                 AS 65001
             R1 ---------- R2
              |            |
              | EBGP       | EBGP
              |            |
             R3 ---------- R4
                 AS 65002
```

R1 and R2 advertise the same prefix to AS 65002.

Configure:

On R1:

```text
R1(config)# route-map TO-ISP permit 10
R1(config-route-map)# set metric 50
```

On R2:

```text
R2(config)# route-map TO-ISP permit 10
R2(config-route-map)# set metric 200
```

Apply each route map outbound toward the corresponding ISP neighbor.

Assuming the other relevant attributes are equal, routers in AS 65002 can prefer the route advertised by R1 because it has the lower MED.

MED is most useful when influencing the entry point selected by a **single neighboring AS**.

---

## Setting MED Based on IGP Metric

Cisco IOS XE supports the following route-map command:

```text
R1(config-route-map)# set metric-type internal
```

This dynamically sets the MED of routes advertised to an EBGP neighbor based on the **IGP metric to the route's BGP next hop**.

### Configuration

```text
R1(config)# route-map SET-MED permit 10
R1(config-route-map)# set metric-type internal

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-MED out
```

Suppose R1 has two BGP routes:

| Prefix | IGP Metric to BGP NEXT_HOP |
|---|---|
| `192.0.2.0/24` | 20 |
| `198.51.100.0/24` | 50 |

The advertised MED values are:

| Prefix | Advertised MED |
|---|---|
| `192.0.2.0/24` | 20 |
| `198.51.100.0/24` | 50 |

This allows MED to reflect internal routing costs rather than a fixed value.

**Important distinctions:**

- The command uses the **IGP metric associated with the BGP next hop**, not necessarily the original IGP metric of the destination prefix.
- It is intended for outbound EBGP advertisements.
- It is not supported for redistributing routes into BGP.

See [Origin and MED](../path-selection/attributes/origin-med.md) for MED comparison rules.

---

## Manipulating NEXT_HOP

The **NEXT_HOP** attribute identifies the address used to reach a BGP destination.

It can be modified with an inbound or outbound route map.

### Setting a Specific Next Hop

```text
R1(config)# route-map SET-NH permit 10
R1(config-route-map)# set ip next-hop 10.0.12.2

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-NH in
```

Accepted routes have their BGP NEXT_HOP changed to `10.0.12.2`.

The new next hop must be reachable for the routes to be usable.

### Setting NEXT_HOP to the Peer Address

Cisco also supports:

```text
R1(config-route-map)# set ip next-hop peer-address
```

Its behavior depends on the route-map direction.

| Direction | Effect |
|---|---|
| Inbound | Sets NEXT_HOP to the address of the BGP neighbor |
| Outbound | Sets NEXT_HOP to the local peering address used toward the neighbor |

This is useful when routes contain third-party next hops.

For example, an inbound route map can force matching routes to use the advertising neighbor as NEXT_HOP:

```text
R1(config)# route-map PEER-NH permit 10
R1(config-route-map)# set ip next-hop peer-address

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map PEER-NH in
```

### Route Map vs. `next-hop-self`

The command:

```text
R1(config-router-af)# neighbor 10.0.12.2 next-hop-self
```

normally changes NEXT_HOP for applicable routes advertised to the neighbor.

A route map provides more granular control by allowing NEXT_HOP to be changed only for matching routes.

See [NEXT_HOP](../path-selection/attributes/next-hop.md) and [BGP Next-Hop Handling](../peering/next-hop.md).

---

## Manipulating Standard Communities

**BGP communities** allow routes to be tagged with values that can be matched by routing policies.

Use `set community` to assign standard communities.

### Configuration

```text
R1(config)# ip bgp-community new-format

R1(config)# route-map SET-COMM permit 10
R1(config-route-map)# set community 65001:100

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-COMM out
R1(config-router-af)# neighbor 10.0.12.2 send-community
```

This assigns community `65001:100` to advertised routes.

The `ip bgp-community new-format` command enables the `ASN:VALUE` representation for standard communities.

### Replacing vs. Adding Communities

By default, `set community` replaces the existing standard community values.

For example:

```text
R1(config-route-map)# set community 65001:100
```

This replaces existing standard communities with `65001:100`.

To preserve existing communities:

```text
R1(config-route-map)# set community 65001:100 additive
```

The `additive` keyword adds the new community without removing the existing standard communities.

### Removing Communities

To remove all standard communities:

```text
R1(config-route-map)# set community none
```

To remove selected communities, use a community list with `set comm-list ... delete`.

For example:

```text
R1(config)# ip community-list standard REMOVE permit 65001:100

R1(config)# route-map REMOVE-COMM permit 10
R1(config-route-map)# set comm-list REMOVE delete
```

This removes community `65001:100` from matching routes while preserving other communities.

### Advertising Communities

By default, standard communities are not necessarily transmitted to BGP neighbors.

Enable their transmission:

```text
R1(config-router-af)# neighbor 10.0.12.2 send-community
```

For both standard and extended communities:

```text
R1(config-router-af)# neighbor 10.0.12.2 send-community both
```

Applying `set community` does not guarantee that the neighbor receives the community. The appropriate advertisement capability must also be enabled.

See [Standard and Extended Communities](communities/overview.md).

---

## Manipulating Extended Communities

Extended communities support additional BGP applications, including MPLS VPN route targets, site-of-origin information, and Cost Communities.

For example, a route map can add a route-target extended community:

```text
R1(config)# route-map SET-RT permit 10
R1(config-route-map)# set extcommunity rt 65001:100 additive
```

The `additive` keyword preserves existing route-target values.

This type of policy is primarily relevant to MP-BGP VPN address families rather than ordinary IPv4 unicast Internet routing.

Cisco IOS XE also supports manipulating other extended-community types, including Cost Communities used at specific points in the best-path algorithm.

See [BGP Cost Communities and Points of Insertion](../path-selection/best-path/cost-community.md) and [Extended BGP Communities](communities/extended.md).

---

## Manipulating AIGP

The **Accumulated IGP (AIGP)** attribute allows an accumulated routing metric to be carried through supported BGP environments.

Cisco IOS XE supports:

```text
R1(config-route-map)# set aigp-metric 100
```

This assigns an AIGP metric value of 100.

Alternatively:

```text
R1(config-route-map)# set aigp-metric igp-metric
```

This derives the AIGP metric from the relevant routing-table metric.

For example, a route map can be used when originating a prefix:

```text
R1(config)# route-map SET-AIGP permit 10
R1(config-route-map)# set aigp-metric 100

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# network 192.0.2.0 mask 255.255.255.0 route-map SET-AIGP
```

The exact prefix must exist in the routing table for the `network` statement to originate it.

AIGP advertisement and processing also require appropriate neighbor support and configuration, such as:

```text
R1(config-router-af)# neighbor 10.0.12.2 aigp
```

AIGP is not a general replacement for LOCAL_PREF or MED. It is designed for networks where accumulated internal routing costs should influence BGP selection.

See [Accumulated IGP (AIGP)](../path-selection/attributes/aigp.md).

---

## Attribute Manipulation During Route Origination

Route maps can also modify attributes when routes are introduced into BGP.

Two common mechanisms are:

1. `network ... route-map`
2. `redistribute ... route-map`

### Network Statement with Route Map

```text
R1(config)# route-map ORIGINATE permit 10
R1(config-route-map)# set metric 50
R1(config-route-map)# set community 65001:100 additive

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# network 192.0.2.0 mask 255.255.255.0 route-map ORIGINATE
```

When the exact route exists in the local RIB, BGP can originate it with the configured attributes.

The community must still be transmitted to neighbors using the appropriate `send-community` configuration.

### Redistribution with Route Map

```text
R1(config)# ip prefix-list REDIST permit 192.0.2.0/24

R1(config)# route-map REDIST-STATIC permit 10
R1(config-route-map)# match ip address prefix-list REDIST
R1(config-route-map)# set origin igp
R1(config-route-map)# set metric 100

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# redistribute static route-map REDIST-STATIC
```

This redistributes matching static routes into BGP and assigns:

- ORIGIN IGP
- MED 100

By default, redistribution into BGP normally creates routes with ORIGIN Incomplete. The route map overrides that value.

**Important:** When route maps are used during route origination or redistribution, their permit/deny decisions can determine whether routes are introduced into BGP at all.

---

## Combining Multiple Attribute Modifications

A single route-map sequence can modify multiple BGP attributes.

For example:

```text
R1(config)# ip prefix-list PREFERRED permit 203.0.113.0/24

R1(config)# route-map ISP-IN permit 10
R1(config-route-map)# match ip address prefix-list PREFERRED
R1(config-route-map)# set weight 500
R1(config-route-map)# set local-preference 200
R1(config-route-map)# set origin igp

R1(config)# route-map ISP-IN permit 20

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map ISP-IN in
```

For `203.0.113.0/24`, the route map applies:

| Attribute | Result |
|---|---|
| Weight | 500 |
| LOCAL_PREF | 200 |
| ORIGIN | IGP |

All three modifications apply to the matching route.

Routes that do not match the first sequence are permitted by sequence 20 without these modifications.

### Route-Map Processing Considerations

- Multiple `set` statements can be configured in the same permit sequence.
- A matching `deny` sequence rejects the route rather than applying its `set` actions.
- Without a final permit sequence, nonmatching routes are denied by the implicit deny.
- The `continue` command can allow multiple permitted sequences to contribute attribute modifications.
- When `continue` is used, a later explicit deny can override an earlier permit.

See [BGP Prefix Lists, AS-Path ACLs, and Route Maps](filtering-tools.md).

---

## Verification

### Display BGP Attributes

```text
R1# show bgp ipv4 unicast
R1# show bgp ipv4 unicast 203.0.113.0/24
```

The detailed command is especially useful for examining:

- Weight
- LOCAL_PREF
- AS_PATH
- ORIGIN
- MED
- NEXT_HOP
- Communities
- AIGP and other supported attributes

### Verify Received and Advertised Routes

Display routes accepted from a neighbor:

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 routes
```

Display routes advertised to a neighbor:

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 advertised-routes
```

When inbound soft reconfiguration is enabled, the following command can display the retained pre-policy received routes:

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 received-routes
```

Comparing received and accepted routes can help identify attribute modifications applied by inbound policies.

### Verify Communities

```text
R1# show bgp ipv4 unicast 203.0.113.0/24
R1# show ip bgp community 65001:100
R1# show ip community-list
```

### Verify Routing Policies

```text
R1# show route-map
R1# show ip prefix-list
R1# show running-config | section router bgp
```

### Reapply Inbound Policies

After changing an inbound route map:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

This reprocesses inbound routing information using route refresh when supported.

### Reapply Outbound Policies

After changing an outbound route map:

```text
R1# clear ip bgp 10.0.12.2 soft out
```

This reapplies outbound policies and updates the advertisements sent to the neighbor without tearing down the session.

---

## Common Troubleshooting Scenarios

| Symptom | Possible Cause |
|---|---|
| Weight change does not affect other routers | Weight is locally significant |
| LOCAL_PREF is not received by EBGP neighbor | LOCAL_PREF is not advertised over ordinary EBGP |
| AS_PATH prepend does not change inbound traffic | Remote AS prefers another path based on higher-priority attributes |
| MED has no effect | MED is not compared between the relevant paths, or earlier criteria differ |
| New NEXT_HOP makes a path invalid | NEXT_HOP is unreachable |
| Community is absent on the receiving router | `send-community` is missing or policy removed the community |
| Route disappears after applying an attribute-setting route map | Route-map implicit deny rejected nonmatching routes |
| Attribute change is not reflected in existing routes | Policy has not been reapplied |

---

## Key Points

- BGP attributes are commonly manipulated with route-map `set` commands.
- **Inbound attribute manipulation** affects locally received BGP paths.
- **Outbound attribute manipulation** affects advertisements sent to neighbors.
- Weight is locally significant and is never advertised.
- LOCAL_PREF is propagated through IBGP but not ordinary EBGP.
- AS_PATH prepending is commonly used to influence inbound traffic.
- `set origin` modifies the ORIGIN attribute without changing how the route actually entered BGP.
- `set metric` manipulates MED in BGP routing policies.
- `set metric-type internal` derives outbound MED from the IGP metric to the BGP next hop.
- `set ip next-hop` modifies NEXT_HOP for matching BGP routes.
- `set community ... additive` preserves existing standard communities.
- Communities require appropriate advertisement configuration to be transmitted.
- Route maps can modify multiple attributes simultaneously.
- Route maps can be applied to BGP neighbors, network statements, and redistribution policies.

---

## References

- [Cisco: Select BGP Best-path Algorithm](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html)
- [Cisco: Connecting to a Service Provider Using External BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-external-sp-0.html)
- [Cisco: BGP Command Reference, O through show bgp](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-n1.html)
- [Cisco: BGP Configuration Guide, Understanding BGP](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr3-fwd/bgp/bgp-configuration-guide/routing-bgp.html)
- [Cisco: Configure the AIGP Metric Attribute for BGP](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/119001-configure-aigp-00.html)
