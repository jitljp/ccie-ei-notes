# Outbound Traffic Engineering

**BGP outbound traffic engineering** controls which external connection an autonomous system uses when forwarding traffic toward destinations outside the AS.

In a multihomed network, multiple ISPs may advertise routes to the same destinations. BGP routing policies allow the enterprise to select preferred exit points based on business, performance, or redundancy requirements.

Common objectives include:

- Prefer one ISP for all outbound traffic.
- Use one ISP as primary and another as backup.
- Prefer different ISPs for different destination prefixes.
- Distribute outbound traffic across multiple connections.
- Avoid undesirable or expensive transit paths.
- Provide automatic failover when a preferred BGP path becomes unavailable.

**The key principle is that outbound traffic engineering primarily modifies how the local AS selects paths to external destinations.**

Unlike inbound traffic engineering, the enterprise generally has direct control over the relevant routing decisions.

---

## Outbound Traffic Engineering Overview

Consider an enterprise with two Internet connections:

```text
             ISP1                  ISP2
           AS 65100              AS 65200
               |                     |
               |                     |
               R1 ------------------ R2
                      IBGP
                    AS 65000
                       |
                       |
                Internal Network
```

Both ISPs advertise routes toward Internet destinations.

For example, R1 and R2 might both learn:

```text
203.0.113.0/24
198.51.100.0/24
0.0.0.0/0
```

The enterprise can configure BGP policies to determine which ISP is preferred for each destination.

For example:

| Destination | Preferred ISP | Backup ISP |
|---|---|---|
| `203.0.113.0/24` | ISP1 | ISP2 |
| `198.51.100.0/24` | ISP2 | ISP1 |
| Other destinations | ISP1 | ISP2 |

BGP selects the preferred path for each prefix independently.

The forwarding table then determines where packets are sent.

### Control Plane vs. Data Plane

Outbound traffic engineering involves two related processes.

```text
            BGP Routes Received
                     |
                     v
              Inbound Policies
                     |
                     v
            BGP Best-Path Selection
                     |
                     v
             Routing Table (RIB)
                     |
                     v
           Forwarding Table (FIB)
                     |
                     v
               Packet Forwarding
```

**BGP routing policies influence the control plane. The routing and forwarding tables determine how packets are actually forwarded.**

A path selected as best by BGP must still be eligible for installation in the RIB.

For example, an existing static route with a lower administrative distance may prevent the selected BGP route from being installed.

See [BGP Path Selection and RIB Installation](../../path-selection/best-path/rib-installation.md).

---

## Outbound Traffic Engineering Mechanisms

The primary BGP mechanisms are:

| Mechanism | Scope | Preferred Value | Purpose |
|---|---|---|---|
| Weight | Local router | Highest | Prefer a path on one Cisco router |
| LOCAL_PREF | Entire AS | Highest | Prefer an exit throughout the AS |
| Route filtering | Local router | N/A | Exclude unwanted routes from consideration |
| AS_PATH | BGP path | Shortest, by default | Influence selection when earlier criteria tie |
| IGP metric to NEXT_HOP | Local router | Lowest | Prefer the closest BGP exit when earlier criteria tie |
| BGP multipath | Local forwarding | N/A | Install multiple eligible BGP paths |

The most important configurable attributes for outbound traffic engineering are **Weight and LOCAL_PREF**.

### Relevant Best-Path Selection Order

The early steps in Cisco's normal BGP best-path selection process include:

```text
1. Weight (highest)
2. LOCAL_PREF (highest)
3. Locally originated routes
4. AIGP (lowest, when applicable)
5. AS_PATH length (shortest)
6. ORIGIN (lowest)
7. MED (lowest, under applicable comparison rules)
8. EBGP over IBGP
9. IGP metric to NEXT_HOP (lowest)
...
```

This is a simplified view. Cisco's Cost Community `pre-bestpath` point of insertion can be evaluated before Weight, and other configured features can change the algorithm.

For example, a higher LOCAL_PREF can cause BGP to prefer a path with a longer AS_PATH.

Likewise, a higher Weight can override LOCAL_PREF.

See [BGP Best Path Selection Steps](../../path-selection/best-path/selection-steps.md).

---

## Weight-Based Outbound Traffic Engineering

**Weight** is a Cisco-specific parameter that influences BGP path selection on a single router.

Important characteristics:

- Weight is locally significant.
- Weight is not a standard BGP path attribute.
- Weight is never advertised in BGP UPDATE messages.
- Higher Weight is preferred.
- Default Weight for learned routes is 0.
- Default Weight for locally originated routes is 32768.
- Weight is evaluated before LOCAL_PREF in the normal best-path algorithm.

Weight is useful when multiple ISP connections terminate on the same router.

### Example Topology

```text
       ISP1                         ISP2
     AS 65100                     AS 65200
    10.0.12.2                    10.0.13.3
         |                            |
         |                            |
    E0/0 |                            | E0/1
         +------------- R1 -----------+
                    AS 65000
```

R1 receives `203.0.113.0/24` from both ISPs.

By default, the learned paths have Weight 0.

The requirement is to prefer ISP1.

### Configuring Weight for a Neighbor

```text
R1(config)# router bgp 65000
R1(config-router)# neighbor 10.0.12.2 remote-as 65100
R1(config-router)# neighbor 10.0.13.3 remote-as 65200
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 weight 500
```

This assigns Weight 500 to routes received from ISP1.

Routes received from ISP2 retain their normal Weight of 0.

| Path | Weight |
|---|---|
| Via ISP1 | 500 |
| Via ISP2 | 0 |

Assuming both paths are valid and available, R1 prefers ISP1.

**Important:** `neighbor ... weight` affects only the router on which it is configured.

It does not influence path selection on other routers within the enterprise AS.

### Configuring Weight with a Route Map

Weight can also be assigned selectively using an inbound route map.

For example, prefer ISP1 only for `203.0.113.0/24`.

```text
R1(config)# ip prefix-list PREFERRED permit 203.0.113.0/24

R1(config)# route-map ISP1-IN permit 10
R1(config-route-map)# match ip address prefix-list PREFERRED
R1(config-route-map)# set weight 500

R1(config)# route-map ISP1-IN permit 20

R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map ISP1-IN in
```

The result is:

- `203.0.113.0/24` learned from ISP1 receives Weight 500.
- Other routes learned from ISP1 retain their normal Weight.
- All routes are permitted by the route map.

**Important:** The second route-map sequence is necessary to permit routes that do not match the prefix list.

Without it, those routes would be rejected by the route map's implicit deny.

### Neighbor Weight vs. Route-Map Weight

If both commands apply to a route:

```text
neighbor 10.0.12.2 weight 500
```

and:

```text
set weight 1000
```

the **route-map Weight takes precedence** for matching routes.

This allows a general neighbor preference to be overridden for selected prefixes.

---

## LOCAL_PREF-Based Outbound Traffic Engineering

**LOCAL_PREF (Local Preference)** is a well-known discretionary BGP path attribute used to select preferred exit points throughout an AS.

Unlike Weight, LOCAL_PREF is propagated through IBGP.

Important characteristics:

- Higher LOCAL_PREF is preferred.
- Default LOCAL_PREF is 100.
- LOCAL_PREF is propagated between IBGP peers.
- LOCAL_PREF is not advertised to ordinary EBGP neighbors.
- LOCAL_PREF is evaluated after Weight.
- It provides AS-wide outbound traffic-engineering control.

**LOCAL_PREF is generally the preferred method for outbound traffic engineering in multirouter enterprise networks.**

### Example Topology

```text
       ISP1                         ISP2
     AS 65100                     AS 65200
         |                            |
         | EBGP                       | EBGP
         |                            |
         R1 ------------------------- R2
                    IBGP
         |                            |
         +------------+---------------+
                      |
                      R3
                 AS 65000
```

The enterprise wants to prefer ISP1 for outbound traffic.

R1 assigns LOCAL_PREF 200 to routes learned from ISP1.

R2 assigns LOCAL_PREF 100 to routes learned from ISP2.

| Path | LOCAL_PREF |
|---|---|
| Via ISP1 | 200 |
| Via ISP2 | 100 |

The higher LOCAL_PREF causes the ISP1 path to be preferred throughout the AS, assuming the paths are valid and no earlier selection criterion overrides the comparison.

### Configuring LOCAL_PREF on R1

```text
R1(config)# route-map ISP1-IN permit 10
R1(config-route-map)# set local-preference 200

R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map ISP1-IN in
```

This assigns LOCAL_PREF 200 to routes received from ISP1.

### Configuring LOCAL_PREF on R2

```text
R2(config)# route-map ISP2-IN permit 10
R2(config-route-map)# set local-preference 100

R2(config)# router bgp 65000
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.23.3 route-map ISP2-IN in
```

Routes received from ISP2 receive LOCAL_PREF 100.

Explicitly setting 100 is optional when the default LOCAL_PREF is already 100.

### How LOCAL_PREF Propagates

Suppose both ISPs advertise `203.0.113.0/24`.

R1 learns:

```text
203.0.113.0/24
NEXT_HOP: 10.0.12.2
LOCAL_PREF: 200
```

R2 learns:

```text
203.0.113.0/24
NEXT_HOP: 10.0.23.3
LOCAL_PREF: 100
```

Through IBGP, the routers can learn the alternative paths and compare their LOCAL_PREF values.

Assuming normal best-path conditions, the path through ISP1 is preferred.

**Important:** LOCAL_PREF is an AS-wide routing-policy attribute, but the routers must have the relevant BGP paths available to compare them.

IBGP propagation rules, route reflection, best-path advertisement, and next-hop reachability all influence the available alternatives.

---

## Changing the Default LOCAL_PREF

Cisco IOS XE supports:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp default local-preference 150
```

This changes the default LOCAL_PREF from 100 to 150 for the local BGP process.

The command provides a default preference for routes that do not have an explicitly assigned LOCAL_PREF.

It does not override explicitly configured LOCAL_PREF values.

For example:

| Route | Explicit LOCAL_PREF | Effective LOCAL_PREF |
|---|---|---|
| Route A | None | 150 |
| Route B | 200 | 200 |
| Route C | 50 | 50 |

Restore the default:

```text
R1(config-router)# no bgp default local-preference
```

**Important:** Changing the default LOCAL_PREF is broader than applying a selective inbound route map.

For multihoming policies, route maps are usually preferable because they allow precise control over which routes receive particular values.

---

## Weight vs. LOCAL_PREF

Consider two routes to the same destination:

| Attribute | ISP1 Path | ISP2 Path |
|---|---|---|
| Weight | 500 | 0 |
| LOCAL_PREF | 100 | 300 |
| AS_PATH length | 3 | 1 |

**ISP1 is preferred** because Weight is evaluated before LOCAL_PREF.

The higher LOCAL_PREF and shorter AS_PATH of ISP2 do not affect the result.

Now consider:

| Attribute | ISP1 Path | ISP2 Path |
|---|---|---|
| Weight | 0 | 0 |
| LOCAL_PREF | 200 | 100 |
| AS_PATH length | 3 | 1 |

**ISP1 is preferred** because LOCAL_PREF is evaluated before AS_PATH length.

### When to Use Each

| Requirement | Preferred Mechanism |
|---|---|
| Prefer one ISP on a single router | Weight or LOCAL_PREF |
| Prefer one ISP throughout the AS | LOCAL_PREF |
| Apply a local exception to AS-wide preference | Weight |
| Apply destination-specific exit preferences | LOCAL_PREF with prefix lists |
| Prefer routes carrying certain communities | Community match with LOCAL_PREF |
| Control outbound routing across many routers | LOCAL_PREF |

---

## Destination-Specific Outbound Traffic Engineering

Outbound traffic can be distributed across different ISPs by assigning different LOCAL_PREF values to different destination prefixes.

Consider:

```text
       ISP1                         ISP2
     AS 65100                     AS 65200
         |                            |
         |                            |
         R1 ------------------------- R2
                    IBGP
                  AS 65000
```

Both ISPs advertise:

```text
203.0.113.0/24
198.51.100.0/24
```

The requirements are:

- Prefer ISP1 for `203.0.113.0/24`.
- Prefer ISP2 for `198.51.100.0/24`.
- Use the other ISP as backup.

### Step 1: Configure R1

Create a prefix list matching the destination that should prefer ISP1:

```text
R1(config)# ip prefix-list VIA-ISP1 permit 203.0.113.0/24
```

Configure an inbound route map:

```text
R1(config)# route-map ISP1-IN permit 10
R1(config-route-map)# match ip address prefix-list VIA-ISP1
R1(config-route-map)# set local-preference 200

R1(config)# route-map ISP1-IN permit 20
R1(config-route-map)# set local-preference 100
```

Apply the route map:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map ISP1-IN in
```

Result:

| Destination | LOCAL_PREF via ISP1 |
|---|---|
| `203.0.113.0/24` | 200 |
| `198.51.100.0/24` | 100 |

### Step 2: Configure R2

Create a prefix list:

```text
R2(config)# ip prefix-list VIA-ISP2 permit 198.51.100.0/24
```

Configure the route map:

```text
R2(config)# route-map ISP2-IN permit 10
R2(config-route-map)# match ip address prefix-list VIA-ISP2
R2(config-route-map)# set local-preference 200

R2(config)# route-map ISP2-IN permit 20
R2(config-route-map)# set local-preference 100
```

Apply the route map:

```text
R2(config)# router bgp 65000
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.23.3 route-map ISP2-IN in
```

Result:

| Destination | LOCAL_PREF via ISP2 |
|---|---|
| `203.0.113.0/24` | 100 |
| `198.51.100.0/24` | 200 |

### Step 3: Compare the Paths

| Destination | ISP1 LOCAL_PREF | ISP2 LOCAL_PREF | Preferred ISP |
|---|---|---|---|
| `203.0.113.0/24` | 200 | 100 | ISP1 |
| `198.51.100.0/24` | 100 | 200 | ISP2 |

The enterprise now uses both ISPs for different destinations.

If one path becomes unavailable, BGP can select the remaining valid path.

**Important:** This is destination-based traffic engineering, not BGP multipath.

Only one BGP path is normally selected as best for each prefix.

---

## Complete Dual-Edge-Router Example

The following example demonstrates a primary/backup outbound design using LOCAL_PREF.

### Topology

```text
           ISP1                      ISP2
         AS 65100                  AS 65200
        10.0.12.2                 10.0.23.3
             |                         |
             |                         |
       E0/0  |                         |  E0/0
             R1 ---------------------- R2
           E0/1                      E0/1
        10.0.1.1                   10.0.1.2
                    IBGP
                  AS 65000
```

The R1-R2 connection uses `10.0.1.0/30`.

Assume:

- ISP1 and ISP2 both advertise `0.0.0.0/0`.
- R1 and R2 provide forwarding connectivity to the enterprise network.
- R1 and R2 should prefer ISP1.
- ISP2 should be used when the ISP1 route is unavailable.

### R1 Configuration

Configure the interfaces:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip address 10.0.12.1 255.255.255.252
R1(config-if)# no shutdown

R1(config)# interface Ethernet0/1
R1(config-if)# ip address 10.0.1.1 255.255.255.252
R1(config-if)# no shutdown
```

Configure the inbound policy:

```text
R1(config)# route-map ISP1-IN permit 10
R1(config-route-map)# set local-preference 200
```

Configure BGP:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# neighbor 10.0.12.2 remote-as 65100
R1(config-router)# neighbor 10.0.1.2 remote-as 65000

R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 activate
R1(config-router-af)# neighbor 10.0.12.2 route-map ISP1-IN in
R1(config-router-af)# neighbor 10.0.1.2 activate
R1(config-router-af)# neighbor 10.0.1.2 next-hop-self
```

### R2 Configuration

Configure the interfaces:

```text
R2(config)# interface Ethernet0/0
R2(config-if)# ip address 10.0.23.2 255.255.255.252
R2(config-if)# no shutdown

R2(config)# interface Ethernet0/1
R2(config-if)# ip address 10.0.1.2 255.255.255.252
R2(config-if)# no shutdown
```

Configure the inbound policy:

```text
R2(config)# route-map ISP2-IN permit 10
R2(config-route-map)# set local-preference 100
```

Configure BGP:

```text
R2(config)# router bgp 65000
R2(config-router)# bgp router-id 2.2.2.2
R2(config-router)# neighbor 10.0.23.3 remote-as 65200
R2(config-router)# neighbor 10.0.1.1 remote-as 65000

R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.23.3 activate
R2(config-router-af)# neighbor 10.0.23.3 route-map ISP2-IN in
R2(config-router-af)# neighbor 10.0.1.1 activate
R2(config-router-af)# neighbor 10.0.1.1 next-hop-self
```

### Expected BGP Behavior

R1 receives the default route from ISP1:

```text
0.0.0.0/0
NEXT_HOP: 10.0.12.2
LOCAL_PREF: 200
```

R2 receives the default route from ISP2:

```text
0.0.0.0/0
NEXT_HOP: 10.0.23.3
LOCAL_PREF: 100
```

Through IBGP, the routers learn the alternative paths.

Assuming both default routes are valid, the ISP1 path is preferred because of its higher LOCAL_PREF.

Both routers can therefore select ISP1 as the preferred exit.

### Importance of `next-hop-self`

The commands:

```text
neighbor 10.0.1.2 next-hop-self
neighbor 10.0.1.1 next-hop-self
```

ensure that EBGP-learned routes advertised between R1 and R2 through IBGP use the advertising router as the BGP NEXT_HOP.

Without `next-hop-self`, an IBGP peer normally preserves the original EBGP NEXT_HOP.

For example, R2 could receive:

```text
0.0.0.0/0
NEXT_HOP: 10.0.12.2
LOCAL_PREF: 200
```

If R2 cannot resolve `10.0.12.2`, it cannot use that BGP path.

With `next-hop-self`, R2 instead receives the path with a next hop of `10.0.1.1`, which is directly reachable.

**Important:** `next-hop-self` provides next-hop reachability in this topology, but the R1-R2 connection must also carry the actual forwarded traffic.

### Failover Behavior

If ISP1 withdraws its default route:

1. R1 loses the EBGP path through ISP1.
2. R1 withdraws its corresponding advertisement from R2.
3. The remaining default route through ISP2 becomes the preferred BGP path.
4. R1 can forward outbound traffic toward R2.
5. R2 forwards the traffic through ISP2.

The reverse process occurs when ISP1 becomes available again and its higher-LOCAL_PREF path is selected.

**Important:** The example assumes R1 and R2 have the required forwarding connectivity and that the ISP1 path is actually withdrawn when unavailable.

---

## Inbound Filtering for Outbound Traffic Engineering

**Inbound BGP route filtering can influence outbound traffic** by controlling which external paths are available for selection.

This terminology is important:

- An inbound route map processes BGP advertisements received from an ISP.
- The resulting routes influence how outbound user traffic is forwarded.

Therefore, the route-map direction and traffic direction are not necessarily the same.

### Example: Prefer ISP2 for One Destination

Consider a router connected to two ISPs.

Both advertise:

```text
203.0.113.0/24
198.51.100.0/24
```

The requirement is:

- Do not use ISP1 for `203.0.113.0/24`.
- Continue accepting other routes from ISP1.
- Use ISP2 for `203.0.113.0/24`.

Configure an inbound prefix list on ISP1:

```text
R1(config)# ip prefix-list FROM-ISP1 deny 203.0.113.0/24
R1(config)# ip prefix-list FROM-ISP1 permit 0.0.0.0/0 le 32
```

Apply it:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 prefix-list FROM-ISP1 in
```

The result is:

| Prefix | ISP1 Path Accepted? |
|---|---|
| `203.0.113.0/24` | No |
| `198.51.100.0/24` | Yes |

Assuming ISP2 advertises a valid path to `203.0.113.0/24`, R1 can use that path instead.

### Route Filtering vs. Attribute Manipulation

| Approach | Effect |
|---|---|
| Inbound filtering | Removes the disallowed path from BGP consideration |
| Lower Weight | Keeps the path available but makes it less preferred |
| Lower LOCAL_PREF | Keeps the path available but makes it less preferred |

**Attribute manipulation is usually preferable when the rejected path would otherwise serve as a useful backup.**

Filtering can eliminate an alternative that might be needed during a failure.

---

## Outbound Traffic Engineering with Communities

Community-based policies allow a router to classify received routes and assign routing preferences.

For example, ISP1 might advertise selected routes with community:

```text
65100:100
```

The enterprise can match that community and assign a higher LOCAL_PREF.

### Configuration

```text
R1(config)# ip bgp-community new-format

R1(config)# ip community-list standard PREFERRED permit 65100:100

R1(config)# route-map ISP1-IN permit 10
R1(config-route-map)# match community PREFERRED
R1(config-route-map)# set local-preference 200

R1(config)# route-map ISP1-IN permit 20
```

Apply the policy:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map ISP1-IN in
```

Routes carrying the community receive LOCAL_PREF 200.

Other routes retain their default LOCAL_PREF.

**Important:** The ISP must actually advertise the communities for the policy to match them.

Alternatively, the enterprise can apply its own communities to received routes and use them for subsequent routing policies.

See [BGP Community Configuration and Filtering](../communities/configuration.md).

---

## Default Routes vs. Full Internet Routes

The effectiveness of outbound traffic engineering depends on which prefixes are available in the BGP table.

### Default Routes Only

Suppose an enterprise receives:

```text
0.0.0.0/0 via ISP1
0.0.0.0/0 via ISP2
```

BGP selects a preferred default route.

Most Internet traffic then follows that selected default route.

The enterprise cannot use BGP LOCAL_PREF to distinguish between individual Internet destinations that are represented only by the same default prefix.

### Full Internet Routes

If the enterprise receives full Internet routing tables, it can apply policies to individual destinations.

For example:

```text
203.0.113.0/24 via ISP1: LOCAL_PREF 200
203.0.113.0/24 via ISP2: LOCAL_PREF 100

198.51.100.0/24 via ISP1: LOCAL_PREF 100
198.51.100.0/24 via ISP2: LOCAL_PREF 200
```

This allows granular outbound path selection.

However, full Internet routing tables require substantially more routing and forwarding resources.

### Default Route Plus More-Specific Prefixes

A third option is to receive default routes and selected more-specific prefixes.

For example:

```text
0.0.0.0/0 via ISP1
0.0.0.0/0 via ISP2
203.0.113.0/24 via ISP2
```

The `/24` route is used for destinations inside `203.0.113.0/24`, assuming it is installed in the forwarding table.

This occurs even when the default route through ISP1 has a higher LOCAL_PREF.

**Longest-prefix matching takes precedence over BGP preference comparisons between different destination prefixes.**

BGP compares candidate paths to the same prefix. The FIB uses longest-prefix matching to select the forwarding entry for a packet.

---

## Hot-Potato Routing

**Hot-potato routing**, also called closest-exit routing, describes a policy in which an AS forwards traffic out of its network through the nearest suitable external exit.

This can reduce the distance that traffic travels through the local AS.

Consider:

```text
        ISP1                       ISP2
          |                          |
          |                          |
          R1 -------- R2 ----------- R3
                       |
                       |
                  Internal Host
```

Suppose R2 learns two equally preferred BGP paths to `203.0.113.0/24`.

The IGP costs to their BGP next hops are:

| Exit | IGP Cost to NEXT_HOP |
|---|---|
| Via R1 | 20 |
| Via R3 | 50 |

If earlier BGP best-path criteria tie, BGP prefers the path with the lower IGP metric to the NEXT_HOP.

In this case, R1 is preferred.

### Influencing Hot-Potato Routing with LOCAL_PREF

Suppose the enterprise wants to prefer R3 despite its higher IGP cost.

It can assign:

```text
Via R1: LOCAL_PREF 100
Via R3: LOCAL_PREF 200
```

The higher LOCAL_PREF causes R3 to be preferred before the IGP metric is evaluated, assuming earlier criteria are equal.

Therefore, **LOCAL_PREF can override ordinary hot-potato exit selection**.

The local AS may deliberately carry traffic farther through its network to use a preferred exit.

This is sometimes called **cold-potato routing**.

---

## Outbound Load Sharing

Multihoming can also be used to distribute traffic across multiple external connections.

There are two important approaches:

1. Destination-based traffic engineering.
2. BGP multipath.

### Destination-Based Traffic Engineering

Different destination prefixes prefer different ISPs.

For example:

| Destination | Preferred ISP |
|---|---|
| `203.0.113.0/24` | ISP1 |
| `198.51.100.0/24` | ISP2 |
| `192.0.2.0/24` | ISP1 |

Each prefix still has a single preferred BGP path.

This provides traffic distribution without requiring BGP multipath.

### BGP Multipath

BGP multipath allows multiple eligible paths to the same destination prefix to be installed in the RIB.

For example:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# maximum-paths 2
```

This allows up to two eligible EBGP paths to be installed.

However, the paths must satisfy the relevant multipath eligibility requirements.

### Different Neighboring Autonomous Systems

When paths are learned from different neighboring ASes, AS_PATH multipath relaxation may be required.

On supported IOS XE platforms:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp bestpath as-path multipath-relax
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# maximum-paths 2
```

This relaxes certain AS_PATH matching requirements.

It does not eliminate the other multipath eligibility requirements, such as compatible Weight and LOCAL_PREF.

### Unequal-Cost Load Sharing

Cisco IOS XE also supports BGP Link Bandwidth on applicable platforms.

For example:

```text
R1(config-router-af)# neighbor 10.0.12.2 dmzlink-bw
R1(config-router-af)# neighbor 10.0.13.3 dmzlink-bw
R1(config-router-af)# bgp dmzlink-bw
```

Combined with appropriate multipath configuration, the feature can support bandwidth-proportional forwarding.

**Important:** BGP multipath does not guarantee exact traffic distribution. CEF forwarding and flow hashing determine which paths packets use.

See [BGP Load Balancing](../../path-selection/load-balancing.md) for detailed multipath and UCMP configuration.

---

## Primary/Backup Outbound Routing

A common multihoming requirement is to use one ISP for normal traffic and another only during failure.

LOCAL_PREF is well suited to this design.

For example:

| ISP | LOCAL_PREF | Role |
|---|---|---|
| ISP1 | 200 | Primary |
| ISP2 | 100 | Backup |

As long as ISP1 advertises a valid preferred route, its path is selected.

When that path becomes unavailable, the backup path can become best.

### Failover Requirements

Successful failover requires:

1. The backup BGP path must remain available.
2. Its NEXT_HOP must be reachable.
3. The primary path must become invalid or be withdrawn when it fails.
4. The backup route must be eligible for RIB installation.
5. The internal network must be able to reach the backup egress router.

These conditions are particularly important in multirouter designs.

### Upstream Failures

A physical interface can remain operational even when an ISP loses connectivity farther upstream.

For example:

```text
Enterprise -------- ISP Edge -------- ISP Core -------- Internet
       Link Up                     Failure
```

The EBGP session might remain established.

If the ISP continues advertising a default route, the enterprise might continue forwarding traffic through that ISP.

**The presence of a BGP default route does not guarantee end-to-end Internet reachability.**

Where appropriate, additional mechanisms can be used to detect or react to failures, including:

- BFD for supported forwarding-path failures.
- BGP Next-Hop Tracking.
- Routing-policy changes triggered by monitored conditions.
- Appropriate route withdrawal or failover policies.

See [BGP Fast Convergence and BFD](../../other-features/fast-convergence.md).

---

## Applying and Updating Outbound Traffic Policies

Although the objective is to control outbound user traffic, Weight and LOCAL_PREF are commonly configured using **inbound BGP route maps**.

For example:

```text
R1(config-router-af)# neighbor 10.0.12.2 route-map ISP1-IN in
```

This modifies attributes on routes received from ISP1.

Changing an inbound route map does not necessarily reprocess previously received routes immediately.

### Soft Inbound Reset

Use:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

When the neighbor supports Route Refresh, the router can request that routes be readvertised and reapply its inbound policies without tearing down the BGP session.

### Soft Outbound Reset

Use:

```text
R1# clear ip bgp 10.0.12.2 soft out
```

This reapplies outbound advertisement policies.

**Important:** An outbound traffic-engineering policy using inbound `set local-preference` or `set weight` generally requires inbound policy reprocessing, not an outbound soft clear.

---

## Verification

### Verify BGP Paths

```text
R1# show bgp ipv4 unicast
```

This displays BGP prefixes and their available paths.

Important fields include:

- NEXT_HOP.
- LOCAL_PREF.
- Weight.
- AS_PATH.
- Best-path status.

### Verify a Specific Prefix

```text
R1# show bgp ipv4 unicast 203.0.113.0/24
```

Example output:

```text
BGP routing table entry for 203.0.113.0/24
Paths: (2 available, best #1)

  65100
    10.0.12.2 from 10.0.12.2
      Origin IGP, localpref 200, valid, external, best

  65200
    10.0.13.3 from 10.0.13.3
      Origin IGP, localpref 100, valid, external
```

The route through ISP1 is preferred because LOCAL_PREF 200 is higher than 100.

### Verify Default-Route Selection

```text
R1# show bgp ipv4 unicast 0.0.0.0/0
R1# show ip route 0.0.0.0
```

The first command displays BGP's default-route candidates.

The second checks the installed route.

### Verify Route Maps

```text
R1# show route-map
R1# show route-map ISP1-IN
```

Verify the match conditions and configured `set` actions.

### Verify Prefix Lists

```text
R1# show ip prefix-list
R1# show ip prefix-list PREFERRED
```

### Verify Neighbor Routes

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 routes
```

This displays routes accepted from the neighbor, subject to platform behavior.

### Verify Forwarding

```text
R1# show ip cef 203.0.113.10 detail
```

This displays CEF forwarding information for the destination.

For a specific source and destination pair, supported platforms provide:

```text
R1# show ip cef exact-route 10.10.10.10 203.0.113.10
```

This can help determine which path CEF selects when multiple forwarding next hops are available.

### Verify with Traceroute

```text
R1# traceroute 203.0.113.10
```

Traceroute can help identify the path traffic follows.

However, intermediate routers may filter or rate-limit the packets used by traceroute.

---

## Troubleshooting

| Problem | Possible Cause |
|---|---|
| Preferred ISP not selected | Weight or LOCAL_PREF lower than expected |
| Higher LOCAL_PREF has no effect | Higher Weight on another path |
| Policy affects only one router | Weight used instead of LOCAL_PREF |
| Routes disappear after applying a route map | Missing permit sequence; implicit deny |
| IBGP route present but invalid | Unreachable BGP NEXT_HOP |
| Router selects ISP path but cannot forward | RIB installation, recursion, or internal forwarding problem |
| Traffic follows unexpected ISP | More-specific route in the forwarding table |
| Backup ISP not used after failure | Backup route missing, invalid, or filtered |
| ISP failure not detected | EBGP session/default route remains present |
| Multipath not working | Paths do not satisfy multipath eligibility requirements |
| LOCAL_PREF changes not reflected | Inbound policy has not been reprocessed |
| BGP best path differs from installed route | Another routing source has better administrative distance |

### Troubleshooting Checklist

1. Confirm that the expected BGP neighbors are established.
2. Verify that both ISPs advertise the relevant prefixes.
3. Check inbound route maps and prefix lists.
4. Verify Weight and LOCAL_PREF values.
5. Confirm that the BGP NEXT_HOP is reachable.
6. Examine the BGP best-path selection result.
7. Confirm that the selected BGP path is installed in the RIB.
8. Check for more-specific routes affecting forwarding.
9. Verify the CEF forwarding path.
10. Test failover by withdrawing the preferred route in a controlled lab.
11. Verify that backup routes remain valid after the failure.
12. Reapply inbound policies if necessary.

---

## Key Points

- **Outbound traffic engineering controls how traffic leaves the local AS.**
- It is primarily implemented by influencing BGP best-path selection.
- Weight and LOCAL_PREF are the most important configurable preference mechanisms.
- Weight is Cisco-specific and locally significant.
- LOCAL_PREF is propagated throughout the AS through IBGP.
- Higher Weight and higher LOCAL_PREF are preferred.
- Weight is evaluated before LOCAL_PREF in the normal Cisco BGP best-path algorithm.
- `neighbor ... weight` assigns Weight to all routes received from a neighbor.
- `set weight` allows selective Weight assignment through an inbound route map.
- `set local-preference` is commonly used for AS-wide outbound traffic engineering.
- `bgp default local-preference` changes the default LOCAL_PREF for the local BGP process.
- Inbound prefix filtering can influence outbound traffic by excluding unwanted paths.
- Attribute manipulation generally preserves more backup-path flexibility than filtering.
- Destination-specific LOCAL_PREF policies can distribute traffic across multiple ISPs without BGP multipath.
- BGP multipath allows multiple eligible routes to the same prefix to be installed.
- Longest-prefix matching determines the forwarding entry before comparing routing preferences associated with different prefixes.
- IBGP next-hop reachability is essential in multirouter multihoming designs.
- A preferred BGP path must also be eligible for RIB installation.
- Redundant ISP connections do not guarantee detection of every upstream failure.
- Changes to inbound Weight or LOCAL_PREF policies may require a soft inbound refresh.

---

## References

- [Cisco: Select BGP Best-path Algorithm](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html)
- [Cisco: Connecting to a Service Provider Using External BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-external-sp-0.html)
- [Cisco: Configure BGP with Two Different Service Providers](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/217964-configure-sample-for-bgp-with-two-differ.html)
- [Cisco: Understand Load Sharing with BGP in Single/Multihomed Environments](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13762-40.html)
- [Cisco: BGP Configuration Guide, Understanding BGP](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr3-fwd/bgp/bgp-configuration-guide/routing-bgp.html)
- [Cisco: iBGP Multipath Load Sharing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-multi-load-0.html)
