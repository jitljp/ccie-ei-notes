# BGP Diverse Path

**BGP Diverse Path** allows a route reflector (RR) to advertise an alternative BGP path instead of its normal best path to a route reflector client.

The feature improves **path diversity** in networks that use route reflectors, where clients may otherwise learn only one path to a destination.

Cisco IOS XE supports Diverse Path using commands such as:

- `bgp additional-paths select backup`
- `bgp additional-paths select best-external`
- `neighbor ... advertise diverse-path backup`
- `neighbor ... advertise diverse-path mpath`
- `bgp additional-paths install`

Despite using the `additional-paths` keyword, **BGP Diverse Path is distinct from the RFC 7911 ADD-PATH feature**.

Diverse Path can operate without ADD-PATH Send/Receive capability negotiation.

---

## The Route Reflector Path-Hiding Problem

By default, a route reflector advertises only its selected best path for a prefix.

Consider a network with two route reflectors and two exit routers:

```text
                    AS 65000

             PE1               PE2
              |                 |
              |                 |
              +-------+---------+
                      |
                 +----+----+
                 |         |
                RR1       RR2
                 |         |
                 +----+----+
                      |
                     PE3
```

Assume:

- PE1 and PE2 advertise different paths to `203.0.113.0/24`.
- RR1 and RR2 both learn these paths.
- Both route reflectors select the path through PE1 as best.
- PE3 is a client of both route reflectors.

### Without Diverse Path

Both RRs advertise the same best path to PE3:

```text
RR1 -> PE3: Path via PE1
RR2 -> PE3: Path via PE1
```

Although PE3 receives routes from two RRs, both advertisements ultimately use the same exit point.

PE3 does not know about the alternative path through PE2.

If PE1 fails, PE3 must wait for updated routing information before using PE2.

This is an example of **BGP path hiding**.

### With Diverse Path

Configure RR2 to select and advertise a backup path.

```text
RR1 -> PE3: Best path via PE1
RR2 -> PE3: Backup path via PE2
```

PE3 now learns two different paths through two separate BGP sessions.

```text
                     PE1        PE2
                      |          |
                   Best        Backup
                      |          |
                     RR1        RR2
                       \        /
                        \      /
                          PE3
```

PE3 can select its preferred path while retaining knowledge of the alternative.

If the preferred path becomes unavailable, PE3 may be able to select the other path more quickly.

**Diverse Path improves path visibility by allowing different RRs to advertise different paths to the same client.**

---

## Diverse Path vs. ADD-PATH

Diverse Path and ADD-PATH solve similar problems, but they use different mechanisms.

### BGP Diverse Path

In a traditional Diverse Path deployment, PE3 learns:

```text
RR1: 203.0.113.0/24 via PE1
RR2: 203.0.113.0/24 via PE2
```

Each route reflector advertises a different path over its own BGP session.

The receiving router can distinguish the paths because they come from different neighbors.

No RFC 7911 Path ID is required.

### BGP ADD-PATH

With RFC 7911 ADD-PATH, a single RR can advertise multiple paths over one session:

```text
RR1 -> PE3:
  203.0.113.0/24, Path ID 10, via PE1
  203.0.113.0/24, Path ID 20, via PE2
```

ADD-PATH uses a 32-bit Path Identifier to distinguish multiple paths advertised by the same neighbor.

It also requires compatible Send/Receive capability negotiation.

### Comparison

| Feature | Diverse Path | ADD-PATH |
|---|---|---|
| Purpose | Provide alternative paths | Advertise multiple paths |
| Typical deployment | Multiple RRs or shadow RR | One or more ADD-PATH-capable BGP speakers |
| Requires Path IDs | No | Yes |
| Requires ADD-PATH capability negotiation | No | Yes |
| Can advertise multiple paths over one session | Not through the traditional Diverse Path mechanism | Yes |
| Common benefit | Avoid RR path hiding | Avoid RR path hiding |
| Cisco selection commands | `select backup`, `select best-external` | `select best 2`, `select best 3`, `select group-best`, `select all` |

**Important:** Cisco IOS XE's `bgp additional-paths select backup` belongs to the Diverse Path feature, not the RFC 7911 ADD-PATH selection methods.

See [BGP Additional Paths (ADD-PATH)](add-path.md).

---

## Selecting a Backup Path

The **`backup`** keyword instructs BGP to calculate an additional path that can serve as an alternative to the normal best path.

Configure:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp additional-paths select backup
```

This causes BGP to calculate a backup path for eligible prefixes.

### Example

Suppose R1 learns three paths to `203.0.113.0/24`:

| Path | NEXT_HOP | BGP Preference |
|---|---|---|
| A | `2.2.2.2` | Best |
| B | `3.3.3.3` | Second-best |
| C | `4.4.4.4` | Third-best |

With:

```text
bgp additional-paths select backup
```

R1 selects:

- **Path A:** Normal best path.
- **Path B:** Backup path.

Path B is the next-best eligible alternative to Path A.

### How Is the Backup Selected?

The router first determines the normal BGP best path.

It then calculates an alternative path using the applicable BGP path-selection criteria.

The backup path does not need to have attributes identical to the best path.

For example, it may have:

- A different NEXT_HOP.
- A different AS_PATH.
- A lower LOCAL_PREF.
- A higher MED.
- A higher IGP metric to the NEXT_HOP.

This differs from BGP multipath, which requires paths to satisfy multipath eligibility rules.

### Selection Does Not Mean Installation

The command:

```text
bgp additional-paths select backup
```

**Calculates the backup path, but does not by itself install it into the routing table or advertise it to a neighbor.**

Those are separate operations.

| Command | Purpose |
|---|---|
| `bgp additional-paths select backup` | Calculate the backup path |
| `bgp additional-paths install` | Install the backup path locally |
| `neighbor ... advertise diverse-path backup` | Advertise the backup path to an RR client |

This distinction is central to the Diverse Path feature.

---

## Advertising the Backup Path

Use:

```text
R1(config-router-af)# neighbor <IP> advertise diverse-path backup
```

This configures the route reflector to advertise its selected backup path to the specified RR client.

### Example

RR2 selects a backup path:

```text
RR2(config)# router bgp 65000
RR2(config-router)# address-family ipv4 unicast
RR2(config-router-af)# bgp additional-paths select backup
```

Configure advertisement to PE3:

```text
RR2(config-router-af)# neighbor 10.1.1.1 advertise diverse-path backup
```

PE3 can now learn the alternative path from RR2.

In the two-RR design:

```text
RR1 -> PE3: Best path
RR2 -> PE3: Backup path
```

This allows PE3 to learn both paths without negotiating RFC 7911 ADD-PATH capability.

### Important Restrictions

Cisco documents the following restrictions:

- `neighbor ... advertise diverse-path` is supported for **route reflector clients**.
- Diverse Path operates within a single AS and RR cluster.
- The feature is not supported on route servers.
- The neighbor must be configured to receive the diverse path.
- If no qualifying backup exists, the best path is advertised instead.

**Important:** `neighbor ... advertise diverse-path backup` does not itself calculate a backup path.

The router must first have a mechanism for calculating the backup.

---

## Selecting the Best External Path

Diverse Path also supports selecting the **best external** path.

Configure:

```text
R1(config-router-af)# bgp additional-paths select best-external
```

This calculates an alternative path from among paths learned through EBGP.

### Backup vs. Best External

Suppose R1 learns:

| Path | Source | Preference |
|---|---|---|
| A | IBGP | Best overall |
| B | IBGP | Second-best overall |
| C | EBGP | Best external path |

The two selection methods serve different purposes.

**Backup selection:**

```text
bgp additional-paths select backup
```

Selects Path B as the alternative to Path A.

**Best External selection:**

```text
bgp additional-paths select best-external
```

Selects Path C, the best eligible externally learned path.

The best external path is not necessarily the second-best path overall.

### Configuration

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp additional-paths select best-external
```

This is useful when preserving an externally learned alternative is more important than selecting the next-best path overall.

### Selecting Both

Cisco IOS XE supports:

```text
R1(config-router-af)# bgp additional-paths select best-external backup
```

This allows both calculations to be requested.

When both selection options are configured with `bgp additional-paths install`, Cisco documents that the **backup path** is installed.

---

## Advertising Multipaths

Diverse Path can also advertise an eligible BGP multipath to a route reflector client.

Configure:

```text
R1(config-router-af)# neighbor <IP> advertise diverse-path mpath
```

The `mpath` keyword refers to a path calculated through BGP multipath processing.

This differs from ordinary backup-path selection.

### Multipath Selection

Suppose R1 has two eligible equal-cost IBGP paths:

```text
203.0.113.0/24 via 2.2.2.2
203.0.113.0/24 via 3.3.3.3
```

Configure:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# maximum-paths ibgp 2
```

BGP can select both paths as multipaths.

A route reflector can advertise the additional multipath to a client:

```text
R1(config-router-af)# neighbor 10.1.1.1 advertise diverse-path mpath
```

The client can learn an alternative path through this advertisement.

### `backup` vs. `mpath`

| Keyword | Advertised Path |
|---|---|
| `backup` | Calculated backup path |
| `mpath` | Additional path selected through BGP multipath |
| `backup mpath` | Multipath takes precedence when available |

For example:

```text
R1(config-router-af)# neighbor 10.1.1.1 advertise diverse-path backup mpath
```

If an eligible multipath exists, Cisco IOS XE advertises that path.

If no multipath exists, the backup path can be used.

If neither exists, the router falls back to advertising its normal best path.

**Important:** An `mpath` advertisement does not itself enable multipath installation on the receiving router.

The receiving router independently determines whether the route is eligible for ECMP.

---

## Installing the Backup Path Locally

Cisco IOS XE supports the following command:

```text
R1(config-router-af)# bgp additional-paths install
```

This enables **local backup-path installation** for BGP Prefix-Independent Convergence (PIC).

When used with:

```text
bgp additional-paths select backup
```

the router can:

1. Calculate the backup path.
2. Install it into the RIB and CEF as an alternate forwarding path.
3. Prepare forwarding to switch to the backup if the primary path fails.

### Example

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp additional-paths select backup
R1(config-router-af)# bgp additional-paths install
```

Suppose R1 learns:

```text
203.0.113.0/24 via 2.2.2.2  (Best)
203.0.113.0/24 via 3.3.3.3  (Backup)
```

The backup can be preinstalled in the forwarding plane.

If the primary path fails, CEF may be able to switch to the backup without waiting for the normal forwarding-programming process to complete.

### Is `select backup` Required with `install`?

**No.**

Cisco documents that:

```text
bgp additional-paths install
```

can independently trigger both:

- Calculation of a backup path.
- Installation of that backup path.

Therefore:

```text
R1(config-router-af)# bgp additional-paths install
```

can be sufficient to enable local backup calculation and installation.

The explicit `select backup` command is useful when the backup path is also needed for other Diverse Path operations, such as advertisement.

### Control-Plane-Only Route Reflectors

A route reflector does not necessarily forward user traffic.

For example, a dedicated RR may exist only to exchange and reflect BGP routes.

In that design, there is generally no reason to install backup forwarding paths on the RR.

Configure:

```text
R1(config-router-af)# bgp additional-paths select backup
R1(config-router-af)# neighbor 10.1.1.1 advertise diverse-path backup
```

Do not configure `bgp additional-paths install` merely to advertise a backup from a control-plane-only RR.

Cisco specifically recommends installing the backup only when the RR participates in forwarding.

**The `install` command is not required for Diverse Path advertisement when backup selection is already configured.**

---

## Shadow Route Reflectors

A **shadow route reflector** is an additional route reflector used to calculate and advertise an alternative path.

The conventional design uses:

- A primary RR advertising the normal best path.
- A shadow RR advertising the diverse path.

### Example

```text
                   AS 65000

               PE1       PE2
                 \       /
                  \     /
                 RR1   RR2
                 Best  Shadow
                   \   /
                    \ /
                    PE3
```

RR1 and RR2 receive the same candidate paths.

RR1 advertises its normal best path.

RR2 calculates and advertises a backup path.

The client learns path diversity through the two RR sessions.

Cisco documents a restriction of one shadow RR per existing RR for this feature.

### Existing RRs Can Be Used

A dedicated new RR is not always necessary.

For example, if RR1 and RR2 already provide redundant route reflection, RR2 can be configured to advertise the diverse path.

This allows an existing RR deployment to provide path diversity without adding another RR.

### IGP Metric Considerations

When primary and shadow RRs are located in different parts of the network, their IGP metrics to the BGP NEXT_HOP addresses may differ.

This can cause them to select different best paths for reasons unrelated to the intended primary/backup design.

Cisco provides:

```text
R1(config-router-af)# bgp bestpath igp-metric ignore
```

This causes BGP to ignore the IGP metric when performing the applicable best-path comparisons.

Cisco recommends using it on the primary and shadow RRs when they are not colocated.

If the RRs are colocated and have equivalent IGP metrics, it is generally unnecessary.

**Important:** Ignoring the IGP metric changes BGP path selection and should be used only when appropriate for the design.

---

## Complete Configuration Example

Consider a network with two route reflectors:

```text
                   AS 65000

               PE1       PE2
                 \       /
                  \     /
                  RR1  RR2
                   \    /
                    \  /
                    PE3
```

Assume:

- RR1 and RR2 are existing route reflectors.
- PE3 is an RR client of both.
- Both RRs learn paths to `203.0.113.0/24` through PE1 and PE2.
- Both normally select PE1 as the best path.
- RR2 should advertise the backup path through PE2.
- IGP reachability between BGP loopbacks is already configured.
- The RR client relationships are operational.

Use the following loopback addresses:

| Router | Loopback0 |
|---|---|
| RR1 | `1.1.1.1` |
| RR2 | `2.2.2.2` |
| PE3 | `3.3.3.3` |

### RR1: Primary Route Reflector

RR1 uses normal route reflection:

```text
RR1(config)# router bgp 65000
RR1(config-router)# neighbor 3.3.3.3 remote-as 65000
RR1(config-router)# neighbor 3.3.3.3 update-source Loopback0
RR1(config-router)# address-family ipv4 unicast
RR1(config-router-af)# neighbor 3.3.3.3 activate
RR1(config-router-af)# neighbor 3.3.3.3 route-reflector-client
```

RR1 advertises its normal best path to PE3.

### RR2: Diverse-Path Route Reflector

RR2 calculates and advertises a backup:

```text
RR2(config)# router bgp 65000
RR2(config-router)# neighbor 3.3.3.3 remote-as 65000
RR2(config-router)# neighbor 3.3.3.3 update-source Loopback0
RR2(config-router)# address-family ipv4 unicast
RR2(config-router-af)# neighbor 3.3.3.3 activate
RR2(config-router-af)# neighbor 3.3.3.3 route-reflector-client
RR2(config-router-af)# bgp additional-paths select backup
RR2(config-router-af)# neighbor 3.3.3.3 advertise diverse-path backup
```

The two important commands are:

```text
bgp additional-paths select backup
neighbor 3.3.3.3 advertise diverse-path backup
```

The first calculates the backup.

The second advertises it to PE3.

There is no requirement to configure `bgp additional-paths install` on RR2 if it is a control-plane-only RR.

### PE3: Receiving Client

PE3 has ordinary IBGP sessions with both RRs:

```text
PE3(config)# router bgp 65000
PE3(config-router)# neighbor 1.1.1.1 remote-as 65000
PE3(config-router)# neighbor 1.1.1.1 update-source Loopback0
PE3(config-router)# neighbor 2.2.2.2 remote-as 65000
PE3(config-router)# neighbor 2.2.2.2 update-source Loopback0
PE3(config-router)# address-family ipv4 unicast
PE3(config-router-af)# neighbor 1.1.1.1 activate
PE3(config-router-af)# neighbor 2.2.2.2 activate
```

PE3 does not need special ADD-PATH Receive capability configuration.

### Expected Result

PE3 learns:

```text
203.0.113.0/24 via PE1, received from RR1
203.0.113.0/24 via PE2, received from RR2
```

PE3 can now perform normal BGP best-path selection using both paths.

If the preferred path becomes unavailable, the alternative may already be present in its BGP table.

---

## Verification

### Verify Available BGP Paths

On the diverse-path RR:

```text
RR2# show ip bgp 203.0.113.0
```

Verify that:

- Multiple paths exist.
- The normal best path is selected.
- An alternative path is available.

### Verify Advertised Routes

```text
RR2# show ip bgp neighbors 3.3.3.3 advertised-routes
```

Verify that the expected alternative path is advertised to PE3.

### Verify Received Paths

On PE3:

```text
PE3# show ip bgp 203.0.113.0
```

Confirm that both RR sessions provide paths to the prefix.

### Verify Local Backup Installation

If `bgp additional-paths install` is configured on a forwarding router:

```text
R1# show ip bgp 203.0.113.0
R1# show ip route 203.0.113.0
R1# show ip cef 203.0.113.0 detail
```

These commands help identify the normal best path and any installed backup path.

A backup path may be identified by status code `b` in the BGP table.

---

## Troubleshooting

| Symptom | Possible Cause | Solution |
|---|---|---|
| RR advertises only its normal best path | Backup selection not configured | Configure `bgp additional-paths select backup` |
| RR advertises only its normal best path | No eligible backup exists | Verify additional BGP paths |
| RR advertises only its normal best path | Diverse-path advertisement not configured | Configure `neighbor ... advertise diverse-path backup` |
| Configuration is rejected | Neighbor is not an RR client | Verify `route-reflector-client` |
| Both RRs advertise the same path | Diverse Path is not configured correctly | Verify RR backup selection and advertisement |
| Backup path not installed locally | `bgp additional-paths install` not configured | Enable installation if the router participates in forwarding |
| Unexpected best-path differences | RRs have different IGP metrics | Evaluate `bgp bestpath igp-metric ignore` |
| Multiple paths received but only one installed | Normal BGP best-path selection | Configure multipath if active ECMP is required |

---

## Diverse Path vs. BGP PIC vs. Multipath

These features provide different forms of path diversity.

| Feature | Purpose | Typical Command |
|---|---|---|
| Diverse Path | Advertise an alternative path through a separate RR session | `neighbor ... advertise diverse-path backup` |
| BGP ADD-PATH | Advertise multiple paths over one session | `neighbor ... advertise additional-paths` |
| BGP PIC | Preinstall a backup forwarding path | `bgp additional-paths install` |
| BGP Multipath | Install multiple eligible active forwarding paths | `maximum-paths` |

**Diverse Path is a control-plane advertisement mechanism. BGP PIC is a forwarding-convergence mechanism. BGP multipath enables multiple active forwarding paths.**

These mechanisms can complement one another, but they should not be treated as interchangeable.

---

## Command Summary

| Command | Purpose |
|---|---|
| `bgp additional-paths select backup` | Calculate a backup BGP path |
| `bgp additional-paths select best-external` | Calculate the best eligible EBGP-learned alternative |
| `bgp additional-paths select best-external backup` | Request both backup and best-external calculations |
| `bgp additional-paths install` | Calculate/install a local backup path for PIC |
| `neighbor <IP> advertise diverse-path backup` | Advertise the backup path to an RR client |
| `neighbor <IP> advertise diverse-path mpath` | Advertise a multipath alternative |
| `neighbor <IP> advertise diverse-path backup mpath` | Prefer an available multipath alternative, otherwise use backup |
| `bgp bestpath igp-metric ignore` | Exclude the IGP metric from applicable BGP best-path comparisons |
| `maximum-paths ibgp <N>` | Enable eligible IBGP multipath installation |
| `show ip bgp <PREFIX>` | Verify BGP paths and backup selection |
| `show ip bgp neighbors <IP> advertised-routes` | Verify advertisements to a neighbor |
| `show ip cef <PREFIX> detail` | Verify forwarding information |

Command availability and restrictions depend on the Cisco IOS XE platform and software release.

---

## Key Points

- **BGP Diverse Path** provides alternative path visibility in route reflector networks.
- It is distinct from **RFC 7911 ADD-PATH**.
- Diverse Path does not require ADD-PATH Send/Receive capability negotiation.
- It is designed for use within a single AS and RR cluster.
- A **shadow RR** can calculate and advertise a different path from the primary RR.
- Existing redundant RRs can also be used.
- `bgp additional-paths select backup` calculates a backup path.
- `bgp additional-paths select best-external` calculates an EBGP-learned alternative.
- `neighbor ... advertise diverse-path backup` advertises the selected backup to an RR client.
- `neighbor ... advertise diverse-path mpath` advertises an eligible multipath.
- When both `backup` and `mpath` are configured for advertisement, `mpath` takes precedence when available.
- `bgp additional-paths install` installs a backup path locally for BGP PIC.
- `bgp additional-paths install` can also independently trigger backup-path calculation.
- Backup-path installation is unnecessary on a control-plane-only RR.
- `bgp bestpath igp-metric ignore` may be necessary when primary and shadow RRs are not colocated.
- Diverse Path advertisement does not automatically enable multipath forwarding on the receiving router.

---

## References

- [Cisco: BGP Diverse Path Using a Diverse-Path Route Reflector, IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg_diverse_path.html)
- [Cisco: BGP Command Reference, bgp additional-paths select](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-a1.html)
- [Cisco: BGP Command Reference, neighbor advertise diverse-path](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [Cisco: BGP Additional Paths, IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-additional-paths.html)
- [RFC 7911: Advertisement of Multiple Paths in BGP](https://www.rfc-editor.org/rfc/rfc7911)
