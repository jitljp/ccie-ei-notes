# BGP Additional Paths (ADD-PATH)

**BGP Additional Paths (ADD-PATH)** allows a BGP router to advertise multiple paths for the same destination prefix to a neighbor over a single BGP session.

By default, BGP advertises only its selected best path for each prefix.

Although a router can learn multiple paths to a destination, its neighbors generally receive only one path from that router.

This behavior can limit path diversity, particularly in networks using BGP route reflectors.

ADD-PATH addresses this limitation by allowing multiple paths for the same prefix to be advertised without the newer advertisements replacing the earlier ones.

The feature is defined in **RFC 7911**.

ADD-PATH can improve:

- Path diversity.
- BGP convergence following failures.
- Availability of alternative paths.
- BGP multipath opportunities.
- Route reflector path visibility.
- Mitigation of certain MED oscillation scenarios.

**Important:** ADD-PATH controls the advertisement and reception of multiple BGP paths. It does not automatically cause multiple paths to be installed in the routing table or used for load balancing.

---

## Default BGP Advertisement Behavior

Consider the following topology:

```text
                 AS 65001

              R2        R3
               \        /
                \      /
                 \    /
                   R1
                    |
                    |
                   R4
```

All four routers are IBGP neighbors as follows:

- R1 is a route reflector.
- R2, R3, and R4 are route reflector clients.
- R2 and R3 learn the same destination prefix through different external paths.

Suppose R1 learns two paths to `203.0.113.0/24`:

| Path | NEXT_HOP | LOCAL_PREF | IGP Metric to NEXT_HOP |
|---|---|---|---|
| Path A, through R2 | `2.2.2.2` | 100 | 10 |
| Path B, through R3 | `3.3.3.3` | 100 | 20 |

Assume all earlier best-path criteria are equal.

R1 selects Path A as its best path because it has the lower IGP metric to the NEXT_HOP.

### Without ADD-PATH

R1 advertises only Path A to R4.

```text
                 R2        R3
                  |         |
               Path A     Path B
                  |         |
                  +----+----+
                       |
                       v
                      R1
                Best = Path A
                       |
                       | Advertise Path A
                       v
                      R4
                Learns Path A
```

R4 does not learn Path B through R1.

This is an example of **BGP path hiding**.

If Path A subsequently fails, R4 must wait for updated BGP routing information before it can use Path B.

### With ADD-PATH

R1 can advertise both paths to R4:

```text
                 R2        R3
                  |         |
               Path A     Path B
                  |         |
                  +----+----+
                       |
                       v
                      R1
                       |
                       | Path A + Path B
                       v
                      R4
                 /           \
                v             v
             Path A         Path B
              Best         Alternate
```

R4 now knows about both paths before a failure occurs.

If Path A becomes unavailable, R4 may be able to select Path B without waiting for R1 to advertise it for the first time.

**ADD-PATH reduces path hiding by allowing routers to advertise alternatives to their selected best path.**

---

## How ADD-PATH Works

Normally, BGP identifies an advertised route using its destination prefix within a particular BGP session and address family.

For example:

```text
203.0.113.0/24
```

If the same neighbor sends another advertisement for `203.0.113.0/24`, that advertisement implicitly replaces the previous advertisement for the prefix.

This is known as an **implicit withdrawal**.

### Default Behavior

```text
R1 advertises:
203.0.113.0/24 via 2.2.2.2

Later, R1 advertises:
203.0.113.0/24 via 3.3.3.3
```

The second advertisement replaces the first path.

R4 does not retain two independently advertised paths from R1.

### ADD-PATH Behavior

ADD-PATH introduces a **Path Identifier (Path ID)**.

Each advertised path is identified by the combination of:

```text
Destination Prefix + Path Identifier
```

For example:

```text
203.0.113.0/24, Path ID 10
203.0.113.0/24, Path ID 20
```

The paths can have different BGP attributes and NEXT_HOP values.

| Prefix | Path ID | NEXT_HOP |
|---|---|---|
| `203.0.113.0/24` | 10 | `2.2.2.2` |
| `203.0.113.0/24` | 20 | `3.3.3.3` |

Because the Path IDs differ, the second advertisement does not implicitly replace the first.

Both paths can coexist in the receiving router's BGP table.

### Path Identifier

The Path Identifier is a **32-bit (4-byte) value**.

It is added to the Network Layer Reachability Information (NLRI) encoding used in BGP UPDATE messages.

Conceptually:

```text
Without ADD-PATH:

+-----------------------+
| Prefix Length         |
+-----------------------+
| Prefix                |
+-----------------------+


With ADD-PATH:

+-----------------------+
| Path Identifier       |
|      (4 bytes)        |
+-----------------------+
| Prefix Length         |
+-----------------------+
| Prefix                |
+-----------------------+
```

Important characteristics:

- The Path ID uniquely identifies an advertised path for a prefix within the relevant BGP peering context.
- It is assigned by the advertising router.
- The receiving router must not interpret the numerical value as a metric or preference.
- A router readvertising a path assigns its own Path ID.
- Path IDs are not necessarily preserved across BGP session resets.

The Path ID is not a standard BGP path attribute such as LOCAL_PREF or MED.

It is part of the extended NLRI encoding.

**The purpose of the Path ID is to distinguish multiple advertisements for the same prefix from the same neighbor.**

---

## ADD-PATH Capability Negotiation

ADD-PATH requires BGP capability negotiation.

The capability is exchanged in BGP OPEN messages.

| Field | Value |
|---|---|
| Capability | ADD-PATH |
| Capability Code | 69 |
| Defined In | RFC 7911 |

ADD-PATH capability negotiation is performed separately for each AFI/SAFI.

For example:

- IPv4 unicast: AFI 1, SAFI 1.
- IPv6 unicast: AFI 2, SAFI 1.

### Send and Receive Capabilities

Each router advertises whether it supports sending additional paths, receiving additional paths, or both.

| Capability | Meaning | RFC Capability Value |
|---|---|---|
| Receive | Can receive multiple paths for one prefix | 1 |
| Send | Can send multiple paths for one prefix | 2 |
| Send and Receive | Can send and receive multiple paths | 3 |

These capabilities are directional.

### Example: R1 Sends, R2 Receives

```text
          R1                           R2
           |                            |
           | ADD-PATH Send              |
           |--------------------------->|
           |                            |
           |           ADD-PATH Receive |
           |<---------------------------|
           |                            |
           |====== Multiple Paths =====>|
```

R1 can advertise additional paths to R2 only if:

1. R1 advertises ADD-PATH Send capability.
2. R2 advertises ADD-PATH Receive capability.
3. Both capabilities apply to the same AFI/SAFI.

### Capability Combinations

| R1 Capability | R2 Capability | Multiple Paths from R1 to R2? |
|---|---|---|
| Send | Receive | Yes |
| Send and Receive | Receive | Yes |
| Send and Receive | Send and Receive | Yes |
| Send | Send | No |
| Receive | Receive | No |
| Send | No ADD-PATH support | No |

**Both routers do not need Send and Receive capability.** A sender only needs Send, while the receiver only needs Receive.

If the necessary capabilities are not negotiated, the advertising router falls back to normal BGP behavior and advertises only its best path.

### Capability Renegotiation

ADD-PATH capability is negotiated through BGP OPEN messages.

Therefore, changing the capability configuration on an established session may require the session to be reestablished.

For example:

```text
R1# clear ip bgp 4.4.4.4
```

This performs a hard reset.

A soft reset does not renegotiate ADD-PATH capabilities.

---

## ADD-PATH Configuration Overview

On Cisco IOS XE, advertising additional paths generally requires three steps:

1. **Enable ADD-PATH Send capability.**
2. **Select candidate additional paths.**
3. **Configure which selected paths are advertised to the neighbor.**

These steps are separate.

| Step | Command | Purpose |
|---|---|---|
| 1 | `bgp additional-paths send` | Advertise ADD-PATH Send capability |
| 2 | `bgp additional-paths select ...` | Select candidate additional paths |
| 3 | `neighbor ... advertise additional-paths ...` | Advertise selected additional paths to the neighbor |

The receiving router must also advertise ADD-PATH Receive capability.

**Configuring Send capability alone does not cause every available path to be advertised.**

---

## Configuring ADD-PATH Capabilities

Cisco IOS XE supports ADD-PATH capability configuration at the address-family or neighbor level.

### Address-Family Configuration

Enable ADD-PATH Send for the address family:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp additional-paths send
```

Enable Receive:

```text
R1(config-router-af)# bgp additional-paths receive
```

Enable both Send and Receive:

```text
R1(config-router-af)# bgp additional-paths send receive
```

These commands control the capabilities advertised to applicable neighbors in the address family.

They do not independently select which additional paths are advertised.

### Per-Neighbor Configuration

The capability can also be configured for individual neighbors.

Enable Send:

```text
R1(config-router-af)# neighbor 4.4.4.4 additional-paths send
```

Enable Receive:

```text
R1(config-router-af)# neighbor 4.4.4.4 additional-paths receive
```

Enable both:

```text
R1(config-router-af)# neighbor 4.4.4.4 additional-paths send receive
```

The per-neighbor configuration overrides the address-family capability configuration.

For example:

```text
R1(config-router-af)# bgp additional-paths receive
R1(config-router-af)# neighbor 4.4.4.4 additional-paths send receive
```

By default, the address family supports receiving additional paths.

However, the specific neighbor `4.4.4.4` is configured to support both sending and receiving.

### Disabling ADD-PATH for a Neighbor

Use:

```text
R1(config-router-af)# neighbor 4.4.4.4 additional-paths disable
```

This disables ADD-PATH sending and receiving for that neighbor, overriding address-family or template settings.

Removing a neighbor-specific configuration using the `no` form allows the neighbor to inherit applicable settings again.

---

## Selecting Additional Paths

After enabling ADD-PATH Send capability, the router must determine which paths are candidates for advertisement.

Configure:

```text
R1(config-router-af)# bgp additional-paths select ...
```

Cisco IOS XE supports three primary selection methods:

- `best 2` or `best 3`
- `group-best`
- `all`

These selection methods can be combined.

### Best 2 and Best 3

Configure:

```text
R1(config-router-af)# bgp additional-paths select best 2
```

This selects the two highest-ranked eligible paths:

1. The normal BGP best path.
2. The second-best path.

Alternatively:

```text
R1(config-router-af)# bgp additional-paths select best 3
```

This selects up to three paths:

1. Best path.
2. Second-best path.
3. Third-best path.

The normal BGP best path is included in the number.

**`best 2` means two paths total, not the best path plus two additional paths.**

Cisco IOS XE determines the second-best path by excluding the best path and repeating the applicable best-path comparison process.

The third-best path is determined by excluding the first two paths.

For this selection mechanism, Cisco documents that paths must have unique next hops. Paths with duplicate next hops are excluded from additional-path selection.

### Example

R1 learns three paths:

| Path | NEXT_HOP | Preference |
|---|---|---|
| A | `2.2.2.2` | Best |
| B | `3.3.3.3` | Second-best |
| C | `5.5.5.5` | Third-best |

With:

```text
bgp additional-paths select best 2
```

Paths A and B are selected.

Path C is not selected.

With:

```text
bgp additional-paths select best 3
```

All three can be selected.

---

## Group-Best Selection

The **`group-best`** keyword selects the best path from each group of paths associated with the same neighboring autonomous system.

Configure:

```text
R1(config-router-af)# bgp additional-paths select group-best
```

### Example

R1 learns six paths to the same prefix:

| Path | Neighboring AS | Preference Within AS |
|---|---|---|
| A | 65010 | Best |
| B | 65010 | Second |
| C | 65020 | Best |
| D | 65020 | Second |
| E | 65030 | Best |
| F | 65030 | Second |

With:

```text
bgp additional-paths select group-best
```

R1 selects:

- Path A: Best path from AS 65010.
- Path C: Best path from AS 65020.
- Path E: Best path from AS 65030.

The router runs the applicable best-path comparisons within each AS group.

This differs from selecting the best three paths overall.

For example, the three highest-ranked paths overall might all come from AS 65010.

Using `group-best` instead preserves path diversity across the three autonomous systems.

### Important Distinction

| Selection | Meaning |
|---|---|
| `best 3` | Select the three best paths overall |
| `group-best` | Select the best path from each AS group |

Group-best selection is useful when multiple autonomous systems provide alternative paths to the same destination.

It can also help mitigate certain BGP MED oscillation scenarios.

---

## All Paths Selection

The **`all`** keyword selects all eligible paths with unique next hops.

Configure:

```text
R1(config-router-af)# bgp additional-paths select all
```

This provides the broadest additional-path selection of the three methods.

However, it does not necessarily select every path in the BGP table.

Cisco IOS XE's selection logic excludes paths with duplicate next hops.

In addition, paths must be otherwise eligible for the ADD-PATH selection and advertisement process.

### Comparison

| Selection | Meaning |
|---|---|
| `best 2` | Best two eligible paths |
| `best 3` | Best three eligible paths |
| `group-best` | Best path from each AS group |
| `all` | All eligible paths with unique next hops |

### Combining Selection Methods

Cisco IOS XE allows multiple selection methods to be configured together.

For example:

```text
R1(config-router-af)# bgp additional-paths select best 3 group-best all
```

A path can qualify under more than one selection method.

The router marks candidate paths according to the applicable selection sets.

The configured neighbor advertisement policy then determines which sets are actually advertised.

---

## Advertising Additional Paths to a Neighbor

Selecting additional paths does not automatically advertise them to every neighbor.

Use:

```text
R1(config-router-af)# neighbor <IP> advertise additional-paths ...
```

### Example: Advertise Best Two Paths

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp additional-paths send
R1(config-router-af)# bgp additional-paths select best 2
R1(config-router-af)# neighbor 4.4.4.4 advertise additional-paths best 2
```

Assuming `4.4.4.4` advertises ADD-PATH Receive capability, R1 can advertise its two selected paths.

### Example: Advertise Group-Best Paths

```text
R1(config-router-af)# bgp additional-paths select group-best
R1(config-router-af)# neighbor 4.4.4.4 advertise additional-paths group-best
```

This advertises paths marked as group-best.

### Example: Advertise All Eligible Paths

```text
R1(config-router-af)# bgp additional-paths select all
R1(config-router-af)# neighbor 4.4.4.4 advertise additional-paths all
```

This advertises all selected eligible paths to the neighbor.

### Selection vs. Advertisement

The two commands perform different functions:

```text
bgp additional-paths select best 3
```

Selects candidate paths at the address-family level.

```text
neighbor 4.4.4.4 advertise additional-paths best 2
```

Controls which selected paths are advertised to a particular neighbor.

**The advertised paths must be part of the selected candidate set.**

For example:

```text
R1(config-router-af)# bgp additional-paths select group-best
R1(config-router-af)# neighbor 4.4.4.4 advertise additional-paths best 3
```

This is not a valid combination for achieving best-three advertisement unless the best-three selection is also enabled.

Cisco IOS XE can warn that the address-family selection policy is more restrictive than the neighbor advertisement policy.

The result may be that only the normal best path is advertised.

---

## Complete ADD-PATH Configuration Example

Consider the following topology:

```text
                   AS 65000

               R2         R3
                \         /
                 \       /
                  \     /
                    R1
              Route Reflector
                     |
                     |
                    R4
                   Client
```

R1 is a route reflector.

R2 and R3 advertise different paths to `203.0.113.0/24`.

R4 is another route reflector client.

The objective is to make R1 advertise both paths to R4.

Assume:

- All routers belong to AS 65000.
- Their BGP sessions use Loopback0.
- The loopbacks are reachable through an IGP.
- R1 has already learned both valid paths.
- The paths have distinct NEXT_HOP values.
- R1's normal route reflector configuration is already operational.

### R1: Route Reflector

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp additional-paths select best 2
R1(config-router-af)# neighbor 4.4.4.4 additional-paths send
R1(config-router-af)# neighbor 4.4.4.4 advertise additional-paths best 2
```

The configuration:

1. Selects the best two eligible paths for each prefix.
2. Enables ADD-PATH Send capability toward R4.
3. Advertises the selected best-two set to R4.

### R4: Receiving Router

```text
R4(config)# router bgp 65000
R4(config-router)# address-family ipv4 unicast
R4(config-router-af)# neighbor 1.1.1.1 additional-paths receive
```

R4 advertises ADD-PATH Receive capability to R1.

It does not need to enable ADD-PATH Send unless it also needs to advertise multiple paths to another neighbor.

### Result

Before ADD-PATH:

```text
R4 BGP Table:

203.0.113.0/24
  Path via 2.2.2.2
```

After ADD-PATH:

```text
R4 BGP Table:

203.0.113.0/24
  Path via 2.2.2.2
  Path via 3.3.3.3
```

Both paths are received through the same BGP session with R1.

They are distinguished using Path IDs.

R4 still performs its own BGP best-path selection to determine which path it prefers.

**Receiving multiple paths through ADD-PATH does not mean R4 installs multiple paths in the routing table.**

---

## ADD-PATH and Route Reflectors

ADD-PATH is particularly useful with route reflectors.

Without ADD-PATH, an RR generally advertises only its selected best path for each prefix.

This can hide alternative paths that its clients might otherwise prefer.

### Example: Different IGP Metrics

Suppose a route reflector and its client have different IGP costs to two exit points.

```text
               Exit A       Exit B
                  \         /
                   \       /
                    \     /
                      RR
                       |
                       |
                     Client
```

From the RR's perspective:

| Exit | IGP Metric |
|---|---|
| Exit A | 10 |
| Exit B | 30 |

The RR prefers Exit A.

However, from the client's perspective:

| Exit | IGP Metric |
|---|---|
| Exit A | 30 |
| Exit B | 10 |

The client would prefer Exit B if it could see both paths.

### Without ADD-PATH

The RR advertises only Exit A.

The client cannot select Exit B because the RR hides that path.

### With ADD-PATH

The RR can advertise both paths.

The client performs its own best-path selection and can prefer Exit B if the applicable BGP attributes and comparison rules allow it.

This can improve **hot-potato routing**, where routers prefer the nearest exit from their autonomous system.

### Additional Benefits

ADD-PATH can also help with:

- Faster recovery after BGP NEXT_HOP failures.
- Reducing certain MED-induced route oscillations.
- Providing alternative paths to route reflector clients.
- Improving path diversity when multiple RRs or exit points exist.

However, ADD-PATH does not eliminate the need for correct IGP design, NEXT_HOP reachability, or route reflector loop prevention.

See [Route Reflectors](../scalability/route-reflectors/overview.md).

---


## Installing Additional Paths (BGP PIC)

Cisco IOS XE supports the `bgp additional-paths install` command, which enables **BGP Prefix-Independent Convergence (PIC)**.

Unlike the ADD-PATH advertisement commands, this command controls **local backup-path installation**.

Configure:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp additional-paths install
```

The command causes BGP to:

1. Calculate a backup path in addition to the normal best path.
2. Install the backup path into the RIB and CEF as an alternate forwarding path.
3. Prepare the forwarding plane to switch to the backup if the primary path fails.

This can significantly reduce forwarding convergence time because the backup path is already programmed.

### ADD-PATH Advertisement vs. Installation

| Feature | Purpose |
|---|---|
| `bgp additional-paths send` | Negotiate the capability to advertise multiple paths |
| `bgp additional-paths select` | Select paths for advertisement |
| `neighbor ... advertise additional-paths` | Advertise selected paths to a neighbor |
| `bgp additional-paths install` | Install a local backup path for fast forwarding convergence |

**Important:** `bgp additional-paths install` does not advertise additional paths to BGP neighbors and does not require ADD-PATH capability negotiation.

### BGP PIC vs. Multipath

BGP PIC and BGP multipath both allow more than one path to be installed, but they serve different purposes.

- **BGP PIC:** Installs a backup path for fast failover. Normally, traffic uses only the primary path.
- **BGP Multipath:** Installs multiple eligible active paths so CEF can load balance traffic across them.

BGP PIC also requires an eligible alternative path with a distinct next hop.

### Verification

Useful commands include:

```text
show ip bgp <PREFIX>
show ip route <PREFIX>
show ip cef <PREFIX> detail
```

On supported platforms, the RIB and CEF outputs can identify the installed backup or alternate path.

### Key Points

- `bgp additional-paths install` enables local backup-path installation for BGP PIC.
- The backup path is installed in the RIB and CEF.
- The backup is prepared for fast failover rather than normal ECMP load balancing.
- ADD-PATH Send/Receive capability negotiation is not required.
- BFD can be used to provide rapid failure detection.
- Actual forwarding convergence depends on platform support and failure detection.

See [Cisco: BGP PIC Edge for IP and MPLS-VPN](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-bgp-mp-pic-0.html).

---

## ADD-PATH vs. BGP Multipath

ADD-PATH and BGP multipath are different features.

### ADD-PATH

Controls how many BGP paths can be advertised to a neighbor.

Example:

```text
R1(config-router-af)# bgp additional-paths select best 2
R1(config-router-af)# neighbor 4.4.4.4 advertise additional-paths best 2
```

R1 can advertise two paths for the same prefix to R4.

### BGP Multipath

Controls how many eligible BGP paths can be installed for forwarding.

Example:

```text
R1(config-router-af)# maximum-paths ibgp 2
```

R1 can install up to two eligible IBGP paths in the routing table.

### Comparison

| Feature | ADD-PATH | BGP Multipath |
|---|---|---|
| Primary purpose | Advertise multiple paths | Install multiple eligible paths |
| Controls BGP UPDATE advertisements | Yes | Not directly |
| Controls RIB installation | No | Yes |
| Requires capability negotiation | Yes | No |
| Requires paths to have equal attributes | Not necessarily | Yes, under applicable multipath rules |
| Useful with route reflectors | Yes | Yes, if multiple eligible paths are available |
| Automatically enables load balancing | No | Can enable CEF load sharing |

### Example

R1 advertises two paths to R4 using ADD-PATH.

R4 receives:

```text
203.0.113.0/24 via 2.2.2.2
203.0.113.0/24 via 3.3.3.3
```

If the paths do not meet BGP multipath eligibility requirements, R4 selects only one best path.

Even if ADD-PATH is enabled, CEF will not automatically load balance across both paths.

**ADD-PATH increases path visibility. BGP multipath controls multiple-path installation.**

See [BGP Load Balancing](../path-selection/load-balancing.md).

---

## Filtering and Modifying Additional Paths

Cisco IOS XE supports applying route maps to paths selected for ADD-PATH advertisement.

The route map can match ADD-PATH selection sets using:

```text
match additional-paths advertise-set ...
```

Supported options include:

- `best 2`
- `best 3`
- `best-range`
- `group-best`
- `all`

This allows different policies to be applied to different groups of additional paths.

### Example: Modify the MED of Selected Paths

Suppose R1 selects the best three paths:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp additional-paths select best 3
```

Create a route map:

```text
R1(config)# route-map ADDPATH-OUT permit 10
R1(config-route-map)# match additional-paths advertise-set best 3
R1(config-route-map)# set metric 200
```

Apply it:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 4.4.4.4 route-map ADDPATH-OUT out
R1(config-router-af)# neighbor 4.4.4.4 advertise additional-paths best 3
```

The route map matches paths tagged with the best-three advertise set.

The `set metric 200` command modifies the MED of matched paths.

### Applying Different Policies

A route map can contain multiple sequences matching different ADD-PATH selection sets.

For example:

```text
R1(config)# route-map ADDPATH-OUT permit 10
R1(config-route-map)# match additional-paths advertise-set best 2
R1(config-route-map)# set metric 100

R1(config)# route-map ADDPATH-OUT permit 20
R1(config-route-map)# match additional-paths advertise-set group-best
R1(config-route-map)# set metric 200
```

The first sequence applies to paths matching the best-two set.

The second applies to paths matching the group-best set that were not matched by an earlier sequence.

A path can belong to multiple selection sets. As with ordinary route maps, sequence order matters.

**Important:** A route map can filter or modify eligible ADD-PATH advertisements, but it does not replace capability negotiation or candidate-path selection.

---

## ADD-PATH and Route Withdrawal

The Path ID is also important when withdrawing routes.

Suppose R1 has advertised:

```text
203.0.113.0/24, Path ID 10
203.0.113.0/24, Path ID 20
```

If Path ID 10 becomes unavailable, R1 can withdraw that specific path.

R2 can retain Path ID 20.

```text
Before Withdrawal:

203.0.113.0/24
  Path ID 10
  Path ID 20


After Withdrawing Path ID 10:

203.0.113.0/24
  Path ID 20
```

Without ADD-PATH, BGP does not use this Path ID mechanism to distinguish multiple paths advertised by the same neighbor for the same prefix.

An ADD-PATH UPDATE or withdrawal identifies the affected route using the combination of the destination prefix and Path ID.

**An ADD-PATH withdrawal can remove one advertised path without withdrawing the other paths for the same prefix.**

---

## Verification

### Verify Negotiated Capabilities

```text
R1# show ip bgp neighbors 4.4.4.4
```

Look for ADD-PATH capability information.

Depending on the IOS XE version, the output can include entries similar to:

```text
Additional paths Send: advertised and received
Additional paths Receive: advertised and received
```

The exact output indicates which capabilities have been advertised and received.

Verify that the advertising router has successfully negotiated Send capability with a neighbor supporting Receive.

### Verify Path Selection and Path IDs

```text
R1# show ip bgp 203.0.113.0
```

This displays detailed path information for the prefix.

On ADD-PATH-capable IOS XE implementations, the output can include:

- Local Path ID.
- Received Path ID.
- ADD-PATH selection information.
- The selected BGP best path.
- Additional paths marked for advertisement.

The Local Path ID and Received Path ID may differ because Path IDs are assigned locally by BGP speakers.

### Verify Advertised Routes

```text
R1# show ip bgp neighbors 4.4.4.4 advertised-routes
```

Use this command to inspect the advertisements to R4.

For detailed Path ID information, use the prefix-specific BGP output and verify the paths on the receiving router.

### Verify Received Paths

On R4:

```text
R4# show ip bgp 203.0.113.0
```

Confirm that multiple paths are present.

Pay attention to:

- NEXT_HOP.
- AS_PATH.
- LOCAL_PREF.
- MED.
- Path IDs.
- Best-path status.

Even when several paths are present, only one might be selected as best.

### Verify RIB Installation

```text
R4# show ip route 203.0.113.0
```

Compare the BGP table with the IP routing table.

Multiple BGP paths can be present while only one is installed.

---

## Troubleshooting

| Symptom | Possible Cause | Solution |
|---|---|---|
| Only best path advertised | ADD-PATH capability not negotiated | Verify Send/Receive capabilities |
| Only best path advertised | No additional-path selection configured | Configure `bgp additional-paths select` |
| Only best path advertised | Neighbor advertisement configuration missing | Configure `neighbor ... advertise additional-paths` |
| Selected paths not advertised | Neighbor advertisement set does not match selected set | Verify selection and advertisement keywords |
| Fewer paths than expected | Paths share the same NEXT_HOP | Check unique NEXT_HOP requirement |
| ADD-PATH works toward one neighbor but not another | Per-neighbor capability override | Verify neighbor-specific configuration |
| Multiple paths received but only one installed | Normal BGP best-path selection | Check multipath eligibility if multiple-path installation is required |
| Received path becomes invalid | BGP NEXT_HOP is unreachable | Check NEXT_HOP resolution and IGP reachability |
| Route reflector hides alternative paths | ADD-PATH not enabled toward the client | Verify RR selection, advertisement, and client Receive capability |

### Troubleshooting Sequence

When ADD-PATH does not advertise multiple routes:

1. Verify that the advertising router has multiple eligible BGP paths.
2. Verify that the peers have negotiated compatible ADD-PATH capabilities.
3. Confirm that the address family selects additional paths.
4. Verify that the neighbor advertisement policy includes the desired selection set.
5. Confirm that the candidate paths have distinct NEXT_HOP values where required.
6. Check outbound route maps and filters.
7. Verify the actual BGP paths received by the neighbor.
8. Check best-path selection and RIB installation separately.

---

## Command Summary

| Command | Purpose |
|---|---|
| `bgp additional-paths send` | Enable ADD-PATH Send capability for the address family |
| `bgp additional-paths receive` | Enable ADD-PATH Receive capability |
| `bgp additional-paths send receive` | Enable both capabilities |
| `neighbor <IP> additional-paths send` | Enable Send capability for a specific neighbor |
| `neighbor <IP> additional-paths receive` | Enable Receive capability for a specific neighbor |
| `neighbor <IP> additional-paths send receive` | Enable both capabilities for a neighbor |
| `neighbor <IP> additional-paths disable` | Disable ADD-PATH for a neighbor |
| `bgp additional-paths select best 2` | Select the best two eligible paths |
| `bgp additional-paths select best 3` | Select the best three eligible paths |
| `bgp additional-paths select group-best` | Select the best path from each AS group |
| `bgp additional-paths select all` | Select all eligible paths with unique next hops |
| `neighbor <IP> advertise additional-paths best 2` | Advertise paths in the best-two selection set |
| `neighbor <IP> advertise additional-paths group-best` | Advertise group-best paths |
| `neighbor <IP> advertise additional-paths all` | Advertise all selected eligible paths |
| `match additional-paths advertise-set ...` | Match ADD-PATH selection sets in a route map |
| `show ip bgp neighbors <IP>` | Verify ADD-PATH capability negotiation |
| `show ip bgp <PREFIX>` | Verify path selection and Path IDs |

ADD-PATH command support and available selection methods depend on the Cisco IOS XE platform and software release.

---

## Key Points

- **BGP ADD-PATH** allows multiple paths for the same prefix to be advertised over a single BGP session.
- By default, BGP advertises only its best path.
- ADD-PATH helps reduce **route reflector path hiding**.
- The feature is defined in **RFC 7911**.
- ADD-PATH capability uses **BGP Capability Code 69**.
- ADD-PATH introduces a **32-bit Path Identifier**.
- A path is identified by its destination prefix and Path ID.
- Path IDs are locally assigned and do not indicate route preference.
- ADD-PATH supports **Send**, **Receive**, and **Send/Receive** capabilities.
- The sending router must negotiate Send capability with a receiving router that supports Receive.
- ADD-PATH configuration involves **capability negotiation, path selection, and neighbor advertisement**.
- `best 2` selects two paths total, including the normal best path.
- `best 3` selects three paths total, including the normal best path.
- `group-best` selects the best path from each AS group.
- `all` selects eligible paths with unique next hops.
- Neighbor-specific ADD-PATH capability settings override address-family settings.
- **ADD-PATH does not automatically enable BGP multipath or CEF load balancing.**
- BGP can withdraw one ADD-PATH path without withdrawing other paths for the same prefix.
- ADD-PATH can improve convergence by making alternative routes available before a failure occurs.

---

## References

- [Cisco: BGP Additional Paths, IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-additional-paths.html)
- [Cisco: BGP Command Reference, bgp additional-paths](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-a1.html)
- [Cisco: BGP Command Reference, neighbor advertise additional-paths](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [RFC 7911: Advertisement of Multiple Paths in BGP](https://www.rfc-editor.org/rfc/rfc7911)
- [RFC 4456: BGP Route Reflection](https://www.rfc-editor.org/rfc/rfc4456)
