# Cluster IDs, Originator IDs, and Loop Prevention

**BGP Route Reflectors (RRs)** allow IBGP-learned routes to be advertised between IBGP neighbors, providing an exception to the normal IBGP split-horizon rule.

However, allowing IBGP routes to be reflected introduces the possibility of **routing information loops**, where the same route is repeatedly advertised between route reflectors.

To prevent these loops, BGP route reflection uses two path attributes defined in [RFC 4456](https://www.rfc-editor.org/rfc/rfc4456.html):

- **ORIGINATOR_ID:** Identifies the BGP router that originally advertised the route within the AS.
- **CLUSTER_LIST:** Records the route reflector clusters through which the route has passed.

These attributes allow BGP routers to recognize and reject routes that have returned to their originator or to a cluster they have already traversed.

---

## Route Reflection Attributes

The two loop-prevention attributes have the following characteristics:

| Attribute | Type Code | Category | Length |
|---|---|---|---|
| ORIGINATOR_ID | 9 | Optional, non-transitive | 4 bytes |
| CLUSTER_LIST | 10 | Optional, non-transitive | Variable |

Both attributes are introduced by route reflection.

They are not normally present when a route is first advertised from its originating router to an RR.

### Loop Prevention Rules

| Attribute | Rejection Condition |
|---|---|
| ORIGINATOR_ID | The received ORIGINATOR_ID matches the local BGP router ID |
| CLUSTER_LIST | The received CLUSTER_LIST contains the local cluster ID of the RR |

These checks prevent routing information from circulating indefinitely within an AS.

---

## Cluster IDs

A **cluster** consists of a route reflector and its clients.

Each route reflector has a **cluster ID**, which identifies the cluster to which it belongs.

The cluster ID is a 4-byte value, normally represented in dotted-decimal notation.

### Default Cluster ID

By default, an RR uses its **BGP router ID** as its cluster ID.

For example:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp router-id 1.1.1.1
```

If R1 is configured as an RR and no explicit cluster ID is configured:

```text
BGP Router ID: 1.1.1.1
Cluster ID:    1.1.1.1
```

The cluster ID does not need to be configured manually when there is only one RR in the cluster.

### Manually Configuring a Cluster ID

The `bgp cluster-id` command explicitly sets the cluster ID.

For example:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# bgp cluster-id 10.10.10.10
```

R1 now has:

```text
BGP Router ID: 1.1.1.1
Cluster ID:    10.10.10.10
```

The router ID and cluster ID are independent values.

The cluster ID does not need to correspond to an interface address.

An explicitly configured cluster ID is particularly useful when multiple route reflectors provide redundant route reflection services.

### Multiple Route Reflectors

When multiple RRs serve the same clients, two designs are possible:

1. **Same cluster ID:** Both RRs use a common cluster ID.
2. **Different cluster IDs:** Each RR uses a different cluster ID.

Both designs are valid, but they affect route reflection behavior.

With the same cluster ID, an RR rejects reflected routes that have already passed through the shared cluster.

With different cluster IDs, an RR can accept reflected routes from the other RR, provided they pass the normal loop-prevention checks.

For example, two RRs can be configured with a common cluster ID:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# bgp cluster-id 10.10.10.10
```

```text
R2(config)# router bgp 65000
R2(config-router)# bgp router-id 2.2.2.2
R2(config-router)# bgp cluster-id 10.10.10.10
```

Alternatively, each RR can retain its default cluster ID or use a different manually configured cluster ID.

The detailed design considerations are covered in [Multiple Route Reflectors and Redundancy](redundancy.md).

---

## ORIGINATOR_ID

The **ORIGINATOR_ID** attribute identifies the BGP router that originally advertised a route within the local AS.

It is an optional, non-transitive BGP path attribute with type code **9**.

When an RR reflects an IBGP route, it creates the ORIGINATOR_ID attribute if the route does not already have one.

The value is the **BGP router ID of the original advertising router**.

### Example

Consider the following topology:

```text
                     AS 65000

                       R1
                 Route Reflector
                  RID: 1.1.1.1
                       / \
                      /   \
                     /     \
                    R2     R3
                 Client   Client
               2.2.2.2   3.3.3.3
```

R1 is the route reflector.

R2 and R3 are RR clients.

Suppose R2 originates the prefix `192.0.2.0/24`.

### Step 1: R2 Advertises the Route

R2 advertises the prefix to R1 using IBGP.

The route does not initially have an ORIGINATOR_ID attribute because it has not been reflected.

### Step 2: R1 Reflects the Route

R1 selects the route as its best path and reflects it to R3.

When reflecting the route, R1 adds:

```text
ORIGINATOR_ID: 2.2.2.2
```

This identifies R2 as the originator of the route.

**The ORIGINATOR_ID is R2's BGP router ID, not R1's router ID.**

### Step 3: R3 Receives the Route

R3 receives the reflected route.

It compares the ORIGINATOR_ID with its own BGP router ID:

```text
Received ORIGINATOR_ID: 2.2.2.2
Local BGP Router ID:   3.3.3.3
```

The values are different, so the ORIGINATOR_ID loop-prevention check passes.

R3 can then process the route normally.

### ORIGINATOR_ID Is Preserved

If the route is reflected again by another RR, the existing ORIGINATOR_ID is preserved.

For example:

```text
R2 -------- R1 -------- R4 -------- R5
          RR 1        RR 2
```

Assume R2 originally advertised the route, and R1 and R4 subsequently reflected it.

After reflection by R1:

```text
ORIGINATOR_ID: 2.2.2.2
```

After reflection by R4:

```text
ORIGINATOR_ID: 2.2.2.2
```

The second RR does not replace the ORIGINATOR_ID with its own router ID.

This preserves the identity of the original advertising router regardless of how many RRs reflect the route.

### ORIGINATOR_ID Loop Prevention

A BGP router rejects a received route if its ORIGINATOR_ID matches the router's own BGP router ID.

For example, suppose a route originally advertised by R2 is reflected through multiple RRs and eventually returns to R2.

R2 receives:

```text
Prefix:         192.0.2.0/24
ORIGINATOR_ID:  2.2.2.2
```

R2 compares this value to its own router ID:

```text
ORIGINATOR_ID:       2.2.2.2
Local BGP Router ID: 2.2.2.2
```

Because the values match, R2 rejects the route.

This prevents R2 from accepting its own routing information after it has circulated through the route reflection topology.

**Important:** The ORIGINATOR_ID loop-prevention check applies to BGP speakers generally, not only route reflectors.

### ORIGINATOR_ID and EBGP-Learned Routes

Suppose R2 learns a route from an EBGP neighbor and advertises it to R1 through IBGP.

When R1 reflects that route, the ORIGINATOR_ID identifies **R2**, not the external EBGP neighbor.

The attribute identifies the original advertising BGP speaker within the local AS.

It does not identify the AS_PATH origin AS or necessarily the router that originally introduced the prefix into BGP.

---

## CLUSTER_LIST

The **CLUSTER_LIST** attribute records the sequence of route reflector clusters through which a route has passed.

It is an optional, non-transitive BGP path attribute with type code **10**.

Unlike ORIGINATOR_ID, which contains a single 4-byte identifier, CLUSTER_LIST can contain multiple 4-byte cluster IDs.

Each time an RR reflects a route, it **prepends its local cluster ID** to the CLUSTER_LIST.

If the route does not already have a CLUSTER_LIST, the RR creates one.

### Single Route Reflector

Consider:

```text
           R2 -------- R1 -------- R3
          Client       RR        Client

                  Cluster ID:
                  10.10.10.10
```

R2 advertises `192.0.2.0/24` to R1.

When R1 reflects the route to R3, it adds:

```text
ORIGINATOR_ID: 2.2.2.2
CLUSTER_LIST:  10.10.10.10
```

The CLUSTER_LIST contains R1's cluster ID.

### Multiple Route Reflectors

Now consider:

```text
R2 ---------- R1 ---------- R4 ---------- R5
              RR 1          RR 2
          CID: 10.10.10.10  CID: 20.20.20.20
```

Assume the IBGP relationships permit the route to be reflected along this path.

R2 originates `192.0.2.0/24`.

**Step 1: R1 Reflects the Route**

R1 adds its cluster ID:

```text
ORIGINATOR_ID: 2.2.2.2
CLUSTER_LIST:  10.10.10.10
```

**Step 2: R4 Reflects the Route**

R4 prepends its cluster ID:

```text
ORIGINATOR_ID: 2.2.2.2
CLUSTER_LIST:  20.20.20.20 10.10.10.10
```

**Step 3: R5 Receives the Route**

R5 receives the reflected route with both cluster IDs.

The CLUSTER_LIST indicates that the route has passed through two reflector clusters.

Notice that the newest cluster ID is placed at the beginning of the list.

### CLUSTER_LIST Loop Prevention

An RR rejects a received route if its own cluster ID is already present anywhere in the CLUSTER_LIST.

For example, suppose R1 has:

```text
Local Cluster ID: 10.10.10.10
```

R1 receives a route containing:

```text
CLUSTER_LIST: 20.20.20.20 10.10.10.10
```

R1 finds its own cluster ID in the list.

Therefore, it rejects the route.

This prevents the route from circulating through the same RR cluster more than once.

**The local cluster ID does not need to be the first entry in the CLUSTER_LIST. A match anywhere in the list causes the route to be rejected.**

---

## ORIGINATOR_ID vs. CLUSTER_LIST Loop Prevention

These two attributes prevent different types of routing information loops.

| Feature | ORIGINATOR_ID | CLUSTER_LIST |
|---|---|---|
| Identifies | Original advertising BGP router | Clusters traversed |
| Value | Single BGP router ID | Sequence of cluster IDs |
| Created by | First RR reflecting the route | First RR reflecting the route |
| Modified by subsequent RRs | No | Yes, prepend local cluster ID |
| Rejection condition | Matches local BGP router ID | Contains local cluster ID |
| Primary purpose | Prevent a route returning to its originator | Prevent a route returning to an RR cluster |

A route must pass both loop-prevention checks to remain eligible for normal BGP processing.

These checks are separate from the normal BGP AS_PATH loop-prevention mechanism.

Because IBGP advertisements do not normally prepend the local AS to AS_PATH, **AS_PATH loop prevention alone cannot protect against route reflection loops within an AS**.

---

## Same vs. Different Cluster IDs

The choice of cluster IDs becomes particularly important when multiple route reflectors are deployed.

### Same Cluster ID

Consider two RRs serving the same clients:

```text
                    AS 65000

                 R1 -------- R2
               (RR, CID 10) (RR, CID 10)
                  |\        /|
                  | \      / |
                  |  \    /  |
                  |   \  /   |
                  |    \/    |
                  |    /\    |
                  |   /  \   |
                  |  /    \  |
                  | /      \ |
                  |/        \|
                 R3          R4
               Client      Client

        R3 and R4 are clients of both RRs
        All connections are IBGP sessions
```

Both R1 and R2 use cluster ID `10.10.10.10`.

Suppose R3 advertises `192.0.2.0/24` to both RRs.

R1 can reflect the route to R4 and eligible non-client neighbors.

If R1 also reflects the route to R2, R1 includes:

```text
CLUSTER_LIST: 10.10.10.10
```

R2 receives the update and compares the CLUSTER_LIST against its own cluster ID.

Because R2 also uses `10.10.10.10`, it rejects the route.

However, R2 can still independently learn the prefix directly from R3.

**Using the same cluster ID prevents an RR from accepting a route that has already been reflected by another RR in the same cluster.**

### Different Cluster IDs

Now suppose R1 and R2 use different cluster IDs:

```text
R1 Cluster ID: 10.10.10.10
R2 Cluster ID: 20.20.20.20
```

R1 reflects R3's route to R2:

```text
ORIGINATOR_ID: 3.3.3.3
CLUSTER_LIST:  10.10.10.10
```

R2 compares the received CLUSTER_LIST to its own cluster ID:

```text
Received CLUSTER_LIST: 10.10.10.10
Local Cluster ID:     20.20.20.20
```

Because the values do not match, R2 does not reject the route based on CLUSTER_LIST.

The route can be considered by the normal BGP best-path selection process.

If R2 selects the route as its best path and reflects it, it adds its own cluster ID:

```text
CLUSTER_LIST: 20.20.20.20 10.10.10.10
```

If this route later returns to R1, R1 rejects it because its own cluster ID is present.

### Comparison

| Behavior | Same Cluster ID | Different Cluster IDs |
|---|---|---|
| RRs share a cluster ID | Yes | No |
| RR accepts a route already reflected by the other RR | No | Potentially |
| Additional reflected path visibility between RRs | Restricted | Possible |
| ORIGINATOR_ID loop prevention | Yes | Yes |
| CLUSTER_LIST loop prevention | Yes | Yes |

Both designs are supported.

Using different cluster IDs allows more flexibility in inter-RR route exchange, but neither design guarantees that an RR will advertise every available path.

The RR still normally advertises only its selected best path.

See [Multiple Route Reflectors and Redundancy](redundancy.md) for detailed design considerations.

---

## ORIGINATOR_ID and CLUSTER_LIST in Best-Path Selection

In addition to preventing loops, ORIGINATOR_ID and CLUSTER_LIST influence the BGP best-path selection algorithm.

They are used in the later tie-breaking steps, after more influential attributes such as LOCAL_PREF, AS_PATH length, MED, and the IGP metric to the next hop.

### ORIGINATOR_ID and Router ID Comparison

During the BGP router ID comparison, if a path contains ORIGINATOR_ID, BGP uses the **ORIGINATOR_ID instead of the advertising RR's router ID**.

For example, R5 receives two paths to the same prefix:

| Attribute | Path 1 | Path 2 |
|---|---|---|
| Advertising RR | R1 | R2 |
| Advertising RR ID | 1.1.1.1 | 2.2.2.2 |
| ORIGINATOR_ID | 4.4.4.4 | 3.3.3.3 |

Assume all earlier best-path criteria are equal.

BGP compares:

```text
Path 1: ORIGINATOR_ID 4.4.4.4
Path 2: ORIGINATOR_ID 3.3.3.3
```

Path 2 is preferred because `3.3.3.3` is lower than `4.4.4.4`.

The advertising RR's router ID does not replace ORIGINATOR_ID for this comparison.

### CLUSTER_LIST Length Comparison

If the router ID comparison does not distinguish the paths, BGP prefers the path with the **shortest CLUSTER_LIST**.

For example:

| Attribute | Path 1 | Path 2 |
|---|---|---|
| ORIGINATOR_ID | 3.3.3.3 | 3.3.3.3 |
| CLUSTER_LIST | 10.10.10.10 | 20.20.20.20 10.10.10.10 |
| CLUSTER_LIST Length | 1 | 2 |

Assume all earlier best-path criteria are equal.

BGP prefers Path 1 because its CLUSTER_LIST is shorter.

This favors routes that have passed through fewer route reflector clusters.

**Important:** CLUSTER_LIST length is a late best-path tie-breaker. It does not override higher-priority selection criteria.

### No CLUSTER_LIST Attribute

If a route does not contain a CLUSTER_LIST attribute, its length is considered **zero** during this comparison.

Therefore, assuming all earlier criteria are equal:

```text
Path 1: No CLUSTER_LIST       Length 0
Path 2: 10.10.10.10          Length 1
```

Path 1 is preferred.

### Best-Path Selection Order

The relevant late tie-breakers in the Cisco BGP best-path algorithm are:

1. Prefer the lowest BGP router ID, using ORIGINATOR_ID when present.
2. Prefer the shortest CLUSTER_LIST.
3. Prefer the lowest neighbor IP address.

These checks are performed only if earlier criteria have not determined the best path.

See [BGP Best Path Selection Steps](../../path-selection/best-path/selection-steps.md).

---

## Cisco IOS XE Configuration

### Configuring a Cluster ID

Configure the cluster ID under the BGP routing process:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp cluster-id 10.10.10.10
```

The command is configured in BGP router configuration mode, not under an address family.

To remove the manually configured cluster ID:

```text
R1(config-router)# no bgp cluster-id
```

The RR then uses its default cluster ID, derived from its BGP router ID.

### Multiple Cluster IDs

Cisco IOS XE also supports a **Multiple Cluster IDs (MCID)** feature on supported platforms and software releases.

This allows an RR to assign different cluster IDs to different neighbor groups.

For example:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp cluster-id 10.10.10.10
R1(config-router)# neighbor 2.2.2.2 cluster-id 20.20.20.20
R1(config-router)# neighbor 3.3.3.3 cluster-id 30.30.30.30
```

The per-neighbor cluster ID is associated with the specified neighbor relationship.

When reflecting a route received from a neighbor with a configured cluster ID, the RR uses that cluster ID in the CLUSTER_LIST.

Otherwise, it uses the global cluster ID.

For loop prevention, the RR checks received CLUSTER_LIST values against its configured cluster IDs.

This feature supports more complex route reflection topologies, including multiple logical clusters served by a single physical RR.

It is distinct from deploying separate RRs with different global cluster IDs.

---

## Verification

The primary verification commands are:

```text
R1# show ip bgp cluster-ids
R1# show ip bgp 192.0.2.0/24
R1# show ip bgp neighbors
R1# show running-config | section router bgp
```

### Verify the Local Cluster ID

Use:

```text
R1# show ip bgp cluster-ids
```

Example output:

```text
Global cluster-id: 10.10.10.10 (configured: 10.10.10.10)
BGP client-to-client reflection:         Configured    Used
  all (inter-cluster and intra-cluster): ENABLED
  intra-cluster:                         ENABLED       ENABLED

List of cluster-ids:
Cluster-id     #-neighbors C2C-rfl-CFG C2C-rfl-USE
```

The `Global cluster-id` field identifies the cluster ID used by the RR.

If the cluster ID has not been explicitly configured, the router uses its BGP router ID.

The `List of cluster-ids` section is particularly useful when the Multiple Cluster IDs feature is configured.

### Verify ORIGINATOR_ID and CLUSTER_LIST

Use:

```text
R3# show ip bgp 192.0.2.0/24
```

A reflected route can contain output similar to:

```text
BGP routing table entry for 192.0.2.0/24, version 5
Paths: (1 available, best #1, table default)
  Local
    2.2.2.2 (metric 10) from 1.1.1.1 (1.1.1.1)
      Origin IGP, localpref 100, valid, internal, best
      Originator: 2.2.2.2, Cluster list: 10.10.10.10
```

This indicates:

- **Originator:** R2, with BGP router ID `2.2.2.2`.
- **Cluster list:** The route passed through cluster `10.10.10.10`.
- **Advertising RR:** R1, with BGP router ID `1.1.1.1`.

The ORIGINATOR_ID is not necessarily the router ID of the neighbor from which the route was received.

### Troubleshooting Missing Routes

When an expected reflected route is missing, check the following:

1. **BGP Neighbor Relationships:** Verify that the IBGP sessions are Established.
2. **RR Client Configuration:** Verify that the neighbors have the correct client or non-client relationships.
3. **Best-Path Selection:** Verify that the RR selected an eligible best path.
4. **Reflection Rules:** Verify that the source and destination neighbor relationships permit reflection.
5. **ORIGINATOR_ID:** Check whether the received route identifies the local router as its originator.
6. **CLUSTER_LIST:** Check whether the route has already passed through the local RR cluster.
7. **Routing Policies:** Verify that route maps or other policies are not filtering the route.
8. **NEXT_HOP Reachability:** Verify that the next hop of the reflected route is reachable.

Routes rejected by loop prevention might not appear in the receiving router's normal BGP table, so the absence of a prefix does not necessarily indicate that the RR never attempted to advertise it.

Examine the route on the upstream RR and verify the attributes it would advertise.

---

## Key Points

- Route reflection uses ORIGINATOR_ID and CLUSTER_LIST to prevent routing information loops.
- Both attributes are optional, non-transitive BGP path attributes.
- **ORIGINATOR_ID (Type 9)** identifies the original advertising BGP speaker within the AS.
- ORIGINATOR_ID is created during the first reflection and preserved by subsequent RRs.
- A router rejects a route if ORIGINATOR_ID matches its own BGP router ID.
- **CLUSTER_LIST (Type 10)** records the clusters through which a route has been reflected.
- Each RR prepends its local cluster ID when reflecting a route.
- An RR rejects a route if its local cluster ID appears anywhere in CLUSTER_LIST.
- By default, the cluster ID equals the RR's BGP router ID.
- `bgp cluster-id` manually configures the cluster ID.
- Redundant RRs can use the same or different cluster IDs. Both designs are valid.
- Same-cluster RRs reject reflected routes containing their shared cluster ID.
- Different-cluster RRs can accept each other's reflected routes if the normal loop-prevention checks pass.
- During best-path selection, ORIGINATOR_ID replaces the advertising router's ID for the router ID comparison.
- BGP prefers the shortest CLUSTER_LIST when earlier best-path criteria are equal.
- A missing CLUSTER_LIST is treated as having length zero.
- `show ip bgp <prefix>` displays the Originator and Cluster list attributes.

---

## References

- [RFC 4456: BGP Route Reflection](https://www.rfc-editor.org/rfc/rfc4456.html)
- [Cisco: Configuring Internal BGP Features (IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-int-features-0.html)
- [Cisco: Use BGP Route Reflection and Multiple Cluster IDs](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/200153-BGP-Route-Reflection-and-Multiple-Cluste.html)
