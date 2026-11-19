# Multiple Route Reflectors and Redundancy

**BGP Route Reflectors (RRs)** eliminate the requirement for a full mesh of IBGP sessions. However, using a single RR introduces a potential **single point of failure**.

If an RR fails, its clients lose their IBGP sessions to that RR and may lose access to routes learned through it.

To provide redundancy, multiple RRs can serve the same clients.

There are several important design considerations:

- Clients can establish IBGP sessions with multiple RRs.
- Redundant RRs can use the same or different cluster IDs.
- RRs can establish IBGP sessions with each other.
- The relationship between RRs affects route reflection behavior.
- Cluster IDs affect loop prevention and route availability during failures.
- Redundant RRs can select and advertise different best paths.

---

## Redundant Route Reflector Topology

Consider the following topology:

```text
                    AS 65000

                 R1 -------- R2
               (RR)          (RR)
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

        All connections are IBGP sessions
```

R1 and R2 are route reflectors.

R3 and R4 are clients of both RRs.

The IBGP relationships are:

| Router | IBGP Neighbors |
|---|---|
| R1 | R2, R3, R4 |
| R2 | R1, R3, R4 |
| R3 | R1, R2 |
| R4 | R1, R2 |

R3 and R4 do not need an IBGP session with each other because both have redundant route-reflector connections.

### Benefits

This topology provides several advantages.

**1. Route Reflector Redundancy**

If R1 fails, R3 and R4 can continue exchanging BGP routes through R2.

Similarly, if R2 fails, R1 can continue providing route reflection.

**2. Independent BGP Sessions**

Each client maintains separate IBGP sessions with both RRs.

A failure of one session does not necessarily affect the other.

**3. Reduced IBGP Sessions**

The clients do not require a full IBGP mesh.

With `n` clients and two fully connected RRs, the number of IBGP sessions is:

```text
IBGP sessions = 2n + 1
```

This includes two sessions per client and one RR-to-RR session.

For example, with 100 clients:

```text
2(100) + 1 = 201 sessions
```

A full mesh of 102 routers would require:

```text
102 * 101 / 2 = 5,151 sessions
```

Although redundant RRs require more sessions than a single RR, they remain significantly more scalable than a full mesh.

---

## Same vs. Different Cluster IDs

One of the most important decisions in a redundant RR design is whether the RRs should use the **same cluster ID** or **different cluster IDs**.

Both approaches are valid.

However, the choice affects how the RRs process routes received from each other.

### Option 1: Same Cluster ID

Both RRs are configured with the same cluster ID:

```text
R1:
  Router ID:  1.1.1.1
  Cluster ID: 10.10.10.10

R2:
  Router ID:  2.2.2.2
  Cluster ID: 10.10.10.10
```

Configure R1:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp cluster-id 10.10.10.10
```

Configure R2:

```text
R2(config)# router bgp 65000
R2(config-router)# bgp cluster-id 10.10.10.10
```

This is the traditional model described in RFC 4456 for multiple RRs belonging to the same cluster.

When an RR reflects a route, it prepends its cluster ID to the CLUSTER_LIST.

If another RR receives a route containing its own cluster ID, it rejects the route.

Consequently, **RRs using the same cluster ID cannot accept routes that have already been reflected by another RR in their shared cluster**.

### Example: Route Advertisement

Suppose R3 originates `192.0.2.0/24`.

R3 advertises the prefix directly to both R1 and R2.

Both RRs receive the route without ORIGINATOR_ID or CLUSTER_LIST attributes because R3 has not reflected it.

R1 can select the route as its best path and reflect it to R4.

R2 can independently select the same route and reflect it to R4.

R4 therefore has the potential to learn the route from both RRs.

Now consider what happens if R1 reflects the route to R2.

R1 adds:

```text
ORIGINATOR_ID: 3.3.3.3
CLUSTER_LIST:  10.10.10.10
```

R2 compares the received CLUSTER_LIST with its own cluster ID:

```text
R2 local cluster ID: 10.10.10.10
Received CLUSTER_LIST: 10.10.10.10
```

The values match.

R2 rejects the reflected route.

This is expected behavior.

R2 already has a direct IBGP session with R3 and can independently learn the original route.

### Advantages

Using the same cluster ID:

- Prevents reflected routes from circulating between RRs in the same cluster.
- Reduces the number of redundant reflected paths processed between RRs.
- Provides a straightforward loop-prevention model.
- Works well when RRs maintain reliable sessions with the same clients.

However, this design can introduce an important limitation during partial failures.

---

## Same Cluster IDs: Partial Failure Scenario

Consider the original topology:

```text
                 R1 -------- R2
               (RR)          (RR)
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
```

Both RRs use cluster ID `10.10.10.10`.

Suppose the following two IBGP sessions fail:

- R3 loses its session to R2.
- R4 loses its session to R1.

The remaining operational sessions are:

```text
       R3 -------- R1 -------- R2 -------- R4
                  (RR)        (RR)

                 All links are IBGP
```

Notice that:

- R3 still has an IBGP session with R1.
- R4 still has an IBGP session with R2.
- R1 and R2 can still communicate.
- Neither RR has completely failed.

It might appear that R3 and R4 should still be able to exchange BGP routes through the RRs.

However, the shared cluster ID prevents this.

### Step 1: R3 Advertises a Route

R3 originates `192.0.2.0/24` and advertises it to R1.

R3 cannot advertise the route directly to R2 because that IBGP session is down.

### Step 2: R1 Reflects the Route to R2

R1 selects the route as its best path.

Because R2 is a non-client IBGP neighbor, R1 can reflect the client-learned route to R2.

R1 adds:

```text
ORIGINATOR_ID: 3.3.3.3
CLUSTER_LIST:  10.10.10.10
```

### Step 3: R2 Rejects the Route

R2 receives the reflected route.

However, R2 also uses cluster ID `10.10.10.10`.

Because its cluster ID is present in CLUSTER_LIST, R2 rejects the route.

### Step 4: R4 Does Not Learn the Route

R4 cannot learn the route from R1 because its IBGP session to R1 is down.

R2 cannot advertise the route to R4 because it rejected the route received from R1.

Therefore, **R4 does not learn R3's prefix through BGP**.

This demonstrates an important limitation:

**Using the same cluster ID can prevent route exchange through the surviving RR-to-RR session during certain combinations of client-to-RR failures.**

Both RRs remain operational, and both clients retain at least one RR session, but the BGP control plane is partitioned for this prefix.

This does not mean same-cluster designs are invalid. It means their redundancy depends on the underlying peering topology and failure assumptions.

---

## Option 2: Different Cluster IDs

An alternative is to configure the RRs with different cluster IDs.

For example:

```text
R1:
  Router ID:  1.1.1.1
  Cluster ID: 10.10.10.10

R2:
  Router ID:  2.2.2.2
  Cluster ID: 20.20.20.20
```

Configure R1:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp cluster-id 10.10.10.10
```

Configure R2:

```text
R2(config)# router bgp 65000
R2(config-router)# bgp cluster-id 20.20.20.20
```

These RRs belong to different logical clusters, even though they serve the same clients.

Each RR can accept routes that have been reflected by the other RR, provided the routes pass the normal BGP validation and loop-prevention checks.

### Normal Operation

Suppose R3 originates `192.0.2.0/24` and advertises it to both RRs.

R1 learns the route directly from R3.

R2 also learns the route directly from R3.

R1 can reflect its selected best path to R2:

```text
ORIGINATOR_ID: 3.3.3.3
CLUSTER_LIST:  10.10.10.10
```

R2 compares the CLUSTER_LIST with its own cluster ID:

```text
R2 local cluster ID: 20.20.20.20
Received CLUSTER_LIST: 10.10.10.10
```

The values do not match.

Therefore, R2 does not reject the route because of CLUSTER_LIST.

However, R2 already has a direct path from R3 and will perform normal BGP best-path selection.

R2 does not necessarily select the reflected path from R1.

**Different cluster IDs allow inter-RR route exchange, but they do not guarantee that an RR will select or advertise the route received from the other RR.**

---

## Different Cluster IDs: Partial Failure Scenario

Now consider the same two failures:

- R3 loses its IBGP session with R2.
- R4 loses its IBGP session with R1.

The remaining sessions are:

```text
       R3 -------- R1 -------- R2 -------- R4
                  (RR)        (RR)

             CID 10.10.10.10
                              CID 20.20.20.20
```

Unlike the previous scenario, R1 and R2 now have different cluster IDs.

### Step 1: R3 Advertises the Route

R3 advertises `192.0.2.0/24` to R1.

R1 selects the route as its best path.

### Step 2: R1 Reflects the Route to R2

R1 reflects the route to R2, which is configured as a non-client IBGP neighbor.

The route includes:

```text
ORIGINATOR_ID: 3.3.3.3
CLUSTER_LIST:  10.10.10.10
```

### Step 3: R2 Accepts the Route

R2 compares its cluster ID with the received CLUSTER_LIST.

```text
Local Cluster ID: 20.20.20.20
Received CLUSTER_LIST: 10.10.10.10
```

There is no match.

The ORIGINATOR_ID also does not match R2's BGP router ID.

Assuming the route passes the remaining checks and becomes R2's best path, R2 can reflect it to R4.

### Step 4: R2 Reflects the Route to R4

R2 prepends its cluster ID:

```text
ORIGINATOR_ID: 3.3.3.3
CLUSTER_LIST:  20.20.20.20 10.10.10.10
```

R4 receives the reflected route.

R4 can now learn `192.0.2.0/24` even though its direct IBGP session with R1 is down.

The route follows:

```text
R3 -> R1 -> R2 -> R4
```

**Different cluster IDs allow the RR-to-RR session to provide an additional route distribution path during partial failures.**

This is a significant redundancy advantage of using different cluster IDs.

The original BGP NEXT_HOP is normally preserved during reflection, so R4 must still have IP reachability to that next hop for the route to be usable.

---

## Comparing Same and Different Cluster IDs

| Feature | Same Cluster ID | Different Cluster IDs |
|---|---|---|
| Both RRs can serve the same clients | Yes | Yes |
| Full RR failure redundancy | Yes | Yes |
| Accept routes reflected by the other RR | No, if shared cluster ID appears | Potentially |
| Inter-RR reflected path visibility | Restricted | Greater |
| Partial client-session failure resilience | More limited | Potentially better |
| Redundant reflected path processing | Lower | Potentially higher |
| CLUSTER_LIST loop prevention | Yes | Yes |
| Supported design | Yes | Yes |

### Which Should Be Used?

Both approaches are valid.

**Same cluster IDs** are appropriate when the RRs are intended to function as members of one cluster and maintain reliable connectivity to the same clients.

**Different cluster IDs** provide additional flexibility for route exchange between RRs and can improve resilience during partial failures.

Cisco's BGP command reference specifically notes that when stable sessions cannot be maintained between all peers in a cluster, overlay RR clusters with different cluster IDs can be used.

However, different cluster IDs can also increase path processing and control-plane complexity.

The correct choice depends on the topology, client connectivity, redundancy requirements, and scaling considerations.

---

## Should Route Reflectors Peer with Each Other?

In a conventional redundant RR topology, **RRs normally establish IBGP sessions with each other as non-clients**.

Consider:

```text
              R1 ---------------- R2
              RR      IBGP        RR
               \                  /
                \                /
                 R3            R4
               Client        Client
```

R1 and R2 maintain an IBGP session.

Neither RR needs to configure the other as a route reflector client.

### Configuration

On R1:

```text
R1(config)# router bgp 65000
R1(config-router)# neighbor 2.2.2.2 remote-as 65000
R1(config-router)# neighbor 2.2.2.2 update-source Loopback0
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 2.2.2.2 activate
```

On R2:

```text
R2(config)# router bgp 65000
R2(config-router)# neighbor 1.1.1.1 remote-as 65000
R2(config-router)# neighbor 1.1.1.1 update-source Loopback0
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 1.1.1.1 activate
```

Notice that neither configuration includes:

```text
neighbor <RR-IP> route-reflector-client
```

Therefore, the RRs treat each other as regular IBGP non-client neighbors.

### Why Use Non-Client Peering?

Recall the route reflection rules:

| Route Learned From | Reflected To |
|---|---|
| IBGP client | Other clients and non-clients |
| IBGP non-client | Clients only |

Suppose R1 learns a route from R3, its client.

R1 can reflect that route to R2 because R2 is a non-client.

If R2 accepts the route and selects it as its best path, R2 can then reflect it to its clients.

This allows routing information to propagate between clusters.

### Can RRs Be Clients of Each Other?

Technically, yes.

RFC 4456 permits an RR to treat another RR as either a client or a non-client.

However, **mutually configuring redundant RRs as clients of each other is generally unnecessary and is not the conventional design**.

It changes the applicable reflection rules and can introduce additional redundant advertisements.

The standard arrangement is:

- RRs peer with each other as non-clients.
- RRs peer with their clients using `route-reflector-client`.
- Clients do not need full-mesh IBGP sessions with each other.

In hierarchical route reflector designs, an RR can legitimately be a client of an upstream RR.

This is different from simply configuring two peer-level redundant RRs as clients of each other.

### Is RR-to-RR Peering Always Required?

Not necessarily.

If every client has an operational session with both RRs, the clients can advertise their routes directly to both RRs.

In that situation, route exchange between the two RRs may not be needed for those clients to learn each other's routes.

However, establishing RR-to-RR peering is commonly used to provide connectivity between clusters and support more flexible route distribution during failures.

**An RR-to-RR session is particularly useful with different cluster IDs, because routes reflected by one RR can be accepted and reflected onward by the other.**

In larger networks containing multiple RR clusters, a conventional design uses full-mesh IBGP peering between the RRs as non-clients.

---

## Redundant Route Reflector Configuration

The following example configures two RRs and two clients.

### Topology

```text
                    AS 65000

                 R1 -------- R2
               (RR)          (RR)
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
```

### Addressing

| Router | Loopback0 | BGP Router ID | Role |
|---|---|---|---|
| R1 | 1.1.1.1/32 | 1.1.1.1 | RR |
| R2 | 2.2.2.2/32 | 2.2.2.2 | RR |
| R3 | 3.3.3.3/32 | 3.3.3.3 | Client |
| R4 | 4.4.4.4/32 | 4.4.4.4 | Client |

All routers belong to AS 65000.

Assume their loopback addresses are reachable through the IGP.

The example uses different cluster IDs:

```text
R1: 10.10.10.10
R2: 20.20.20.20
```

### R1 Configuration

```text
R1(config)# router bgp 65000
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# bgp cluster-id 10.10.10.10
R1(config-router)# no bgp default ipv4-unicast

R1(config-router)# neighbor 2.2.2.2 remote-as 65000
R1(config-router)# neighbor 2.2.2.2 update-source Loopback0

R1(config-router)# neighbor 3.3.3.3 remote-as 65000
R1(config-router)# neighbor 3.3.3.3 update-source Loopback0

R1(config-router)# neighbor 4.4.4.4 remote-as 65000
R1(config-router)# neighbor 4.4.4.4 update-source Loopback0

R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 2.2.2.2 activate
R1(config-router-af)# neighbor 3.3.3.3 activate
R1(config-router-af)# neighbor 4.4.4.4 activate

R1(config-router-af)# neighbor 3.3.3.3 route-reflector-client
R1(config-router-af)# neighbor 4.4.4.4 route-reflector-client
```

R1 treats:

- R2 as a non-client.
- R3 as a client.
- R4 as a client.

### R2 Configuration

```text
R2(config)# router bgp 65000
R2(config-router)# bgp router-id 2.2.2.2
R2(config-router)# bgp cluster-id 20.20.20.20
R2(config-router)# no bgp default ipv4-unicast

R2(config-router)# neighbor 1.1.1.1 remote-as 65000
R2(config-router)# neighbor 1.1.1.1 update-source Loopback0

R2(config-router)# neighbor 3.3.3.3 remote-as 65000
R2(config-router)# neighbor 3.3.3.3 update-source Loopback0

R2(config-router)# neighbor 4.4.4.4 remote-as 65000
R2(config-router)# neighbor 4.4.4.4 update-source Loopback0

R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 1.1.1.1 activate
R2(config-router-af)# neighbor 3.3.3.3 activate
R2(config-router-af)# neighbor 4.4.4.4 activate

R2(config-router-af)# neighbor 3.3.3.3 route-reflector-client
R2(config-router-af)# neighbor 4.4.4.4 route-reflector-client
```

R2 treats:

- R1 as a non-client.
- R3 as a client.
- R4 as a client.

### R3 Configuration

```text
R3(config)# router bgp 65000
R3(config-router)# bgp router-id 3.3.3.3
R3(config-router)# no bgp default ipv4-unicast

R3(config-router)# neighbor 1.1.1.1 remote-as 65000
R3(config-router)# neighbor 1.1.1.1 update-source Loopback0

R3(config-router)# neighbor 2.2.2.2 remote-as 65000
R3(config-router)# neighbor 2.2.2.2 update-source Loopback0

R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# neighbor 1.1.1.1 activate
R3(config-router-af)# neighbor 2.2.2.2 activate
```

R3 establishes IBGP sessions with both RRs.

It does not require any special route reflector client configuration.

### R4 Configuration

```text
R4(config)# router bgp 65000
R4(config-router)# bgp router-id 4.4.4.4
R4(config-router)# no bgp default ipv4-unicast

R4(config-router)# neighbor 1.1.1.1 remote-as 65000
R4(config-router)# neighbor 1.1.1.1 update-source Loopback0

R4(config-router)# neighbor 2.2.2.2 remote-as 65000
R4(config-router)# neighbor 2.2.2.2 update-source Loopback0

R4(config-router)# address-family ipv4 unicast
R4(config-router-af)# neighbor 1.1.1.1 activate
R4(config-router-af)# neighbor 2.2.2.2 activate
```

R4 also establishes IBGP sessions with both RRs.

### Switching to a Same-Cluster Design

To use the same cluster ID on both RRs, change R2's cluster ID:

```text
R2(config)# router bgp 65000
R2(config-router)# bgp cluster-id 10.10.10.10
```

R1 and R2 now use the same cluster ID.

The IBGP neighbor configurations do not need to change.

Be aware that changing cluster IDs in an operational network can affect route acceptance, path selection, and BGP advertisements.

---

## Failure and Convergence Considerations

Redundant RRs improve availability, but the design does not guarantee instantaneous convergence or eliminate every possible failure scenario.

### Complete RR Failure

Suppose R1 fails completely.

```text
              R1 (DOWN)       R2 (UP)
                  X             |
                  X             |
                  X             |
                  X             |
                 R3             R4
                  \_____________/
                   Via R2 only
```

Both R3 and R4 retain their IBGP sessions with R2.

They can continue learning routes through R2.

The surviving RR must have the required routes and be able to advertise them to its clients.

A complete RR failure is generally handled by either same-cluster or different-cluster designs when all clients maintain healthy sessions with the surviving RR.

### Partial Session Failure

A more difficult scenario occurs when different clients lose connectivity to different RRs.

For example:

```text
R3 -------- R1 -------- R2 -------- R4
             RR          RR
```

R3 can reach only R1.

R4 can reach only R2.

In this scenario:

- Same cluster IDs can prevent route exchange through R1 and R2.
- Different cluster IDs can allow routes to pass through both RRs.

Therefore, partial failure testing is an important part of validating a redundant RR design.

### BGP Session Detection

A client must detect the loss of its RR session before it can remove routes that were learned exclusively from that session.

By default, BGP uses keepalive and hold timers to detect certain neighbor failures.

Cisco IOS XE also supports BFD for faster BGP session failure detection.

For example:

```text
R3(config)# router bgp 65000
R3(config-router)# neighbor 1.1.1.1 fall-over bfd
```

BFD must also be appropriately configured and operational on the relevant devices and interfaces.

Using BFD can reduce failure detection time, but it does not eliminate the need for BGP route processing and convergence after the failure.

### Next-Hop Reachability

Route reflectors normally preserve the original BGP NEXT_HOP attribute.

For example:

```text
R3 -> R1 -> R2 -> R4
```

If R3 originates a route with NEXT_HOP `3.3.3.3`, R4 can still receive that route with NEXT_HOP `3.3.3.3`.

It does not automatically use R1 or R2 as its forwarding next hop.

Therefore, **the BGP control-plane path through the RRs is not necessarily the packet-forwarding path**.

The IGP or another reachability mechanism must provide a valid route to the BGP next hop.

### Control-Plane and Data-Plane Redundancy

RR redundancy protects the distribution of BGP routing information.

It does not necessarily provide redundant data-plane connectivity.

For example, an RR failure might affect BGP sessions without immediately affecting packet forwarding through the network.

Conversely, the RR control plane might remain healthy while a separate forwarding failure makes a BGP next hop unreachable.

Both control-plane and data-plane redundancy must be considered.

---

## Best-Path Selection with Multiple RRs

Redundant RRs perform BGP best-path selection independently.

They do not automatically coordinate to select the same path.

Consider:

```text
                     AS 65000

               R1 (RR)       R2 (RR)
                  |             |
                  |             |
                  +------R3-----+
                         |
                         R4
```

Suppose R1 and R2 learn two routes to `192.0.2.0/24`.

Both routes have equal:

- Weight.
- LOCAL_PREF.
- Locally originated status.
- AS_PATH length.
- ORIGIN.
- MED.

However, the RRs have different IGP costs to the BGP next hops.

| Route | R1 IGP Cost | R2 IGP Cost |
|---|---|---|
| Path A | 10 | 100 |
| Path B | 100 | 10 |

R1 prefers Path A.

R2 prefers Path B.

Consequently:

- R1 advertises Path A to its clients.
- R2 advertises Path B to its clients.

The clients can receive different paths to the same destination from the two RRs.

### Path Hiding

An RR normally advertises only its selected best path.

This can hide other available paths from its clients.

For example, R1 might advertise Path A even though R3 would prefer Path B based on its own IGP metrics.

R3 cannot independently evaluate Path B if R1 does not advertise it and R3 has no other source for that path.

This is known as **BGP path hiding**.

### Does Using Multiple RRs Eliminate Path Hiding?

Not necessarily.

Multiple RRs can improve path diversity if they select different best paths.

However, if both RRs select the same best path, clients might still be unaware of other available alternatives.

Features such as **BGP Additional Paths (Add-Path)** and **Optimal Route Reflection (ORR)** can improve path visibility or client-specific path selection on supported platforms.

These are separate mechanisms from simply configuring redundant RRs.

---

## Multiple Clusters and Hierarchical Designs

Large networks can contain multiple route reflector clusters.

For example:

```text
                     AS 65000

                 R1 -------- R2
                 RR          RR
                /  \        /  \
               /    \      /    \
              R3    R4    R5    R6
            Client Client Client Client

             Cluster A     Cluster B
```

In this example:

- R1 serves R3 and R4.
- R2 serves R5 and R6.
- R1 and R2 maintain an IBGP session.
- R1 and R2 are non-clients of each other.

R1 reflects client-learned routes to R2.

R2 can reflect those routes to its own clients.

This distributes BGP routing information between the two clusters.

### Full Mesh Between Route Reflectors

In a conventional multi-cluster design, the RRs are fully meshed with each other as non-client IBGP neighbors.

For example:

```text
                    AS 65000

                 R1 -------- R2
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
                 R3 -------- R4

                  Four RRs
             Full-mesh IBGP sessions
```

Each RR establishes an IBGP session with every other RR.

Clients maintain sessions with their assigned RRs and do not need a full mesh with all other clients.

### Hierarchical Route Reflectors

In very large networks, a hierarchy of RRs can be used to further reduce IBGP peering requirements.

For example:

```text
                         R1
                     Upper-Level RR
                      /          \
                     /            \
                    R2            R3
               Lower-Level RR  Lower-Level RR
                 /     \        /     \
                R4     R5      R6     R7
              Client Client  Client Client
```

In this design:

- R1 acts as an upper-level RR.
- R2 and R3 are clients of R1.
- R2 and R3 are also RRs for their own lower-level clients.
- R4 and R5 are clients of R2.
- R6 and R7 are clients of R3.

A router can therefore act as both:

- A route reflector for downstream clients.
- A route reflector client of an upstream RR.

This design requires careful consideration of cluster IDs, ORIGINATOR_ID, CLUSTER_LIST, and best-path selection.

Redundant upper-level RRs can also be deployed to avoid a single point of failure at the top of the hierarchy.

---

## Verification

The following commands are useful when verifying redundant RR operation.

### Verify BGP Sessions

On R3:

```text
R3# show ip bgp summary
```

Verify that both RR sessions are Established.

A client should normally maintain an operational session with every RR required by the redundancy design.

### Verify RR Client Relationships

On R1:

```text
R1# show ip bgp neighbors 3.3.3.3
R1# show ip bgp neighbors 4.4.4.4
```

Verify that R3 and R4 are identified as RR clients.

Check R2:

```text
R1# show ip bgp neighbors 2.2.2.2
```

R2 should be a non-client in the conventional design.

### Verify Cluster IDs

On both RRs:

```text
R1# show ip bgp cluster-ids
R2# show ip bgp cluster-ids
```

Verify that the cluster IDs match the intended design.

### Verify Reflected Routes

Use:

```text
R1# show ip bgp 192.0.2.0/24
R2# show ip bgp 192.0.2.0/24
R4# show ip bgp 192.0.2.0/24
```

For reflected routes, the output can include:

```text
Originator: 3.3.3.3, Cluster list: 10.10.10.10
```

If a route is reflected through two different clusters, its CLUSTER_LIST can contain both cluster IDs:

```text
Originator: 3.3.3.3, Cluster list: 20.20.20.20 10.10.10.10
```

### Verify Advertisements

Check which prefixes an RR is advertising to its clients:

```text
R1# show ip bgp neighbors 4.4.4.4 advertised-routes
```

Similarly, check the RR-to-RR advertisements:

```text
R1# show ip bgp neighbors 2.2.2.2 advertised-routes
```

Remember that a route displayed as advertised by R1 is not necessarily accepted by R2.

R2 might reject the route because its cluster ID is already present in CLUSTER_LIST.

### Testing Redundancy

A useful lab exercise is to test several failure scenarios.

| Failure | Expected Result |
|---|---|
| R1 fails | Clients continue through R2 |
| R2 fails | Clients continue through R1 |
| R3 loses its session to R1 | R3 continues through R2 |
| R4 loses its session to R2 | R4 continues through R1 |
| R3 loses R2 and R4 loses R1 | Outcome depends on cluster IDs and inter-RR route exchange |
| RR-to-RR session fails | Clients may continue normally if both still connect to both RRs |

These expectations assume that the surviving RRs have the necessary routes and that no routing policies or next-hop reachability problems prevent route installation.

The most important test for comparing cluster ID designs is the **asymmetric client-session failure**, in which different clients retain sessions with different RRs.

---

## Key Points

- A single RR creates a potential single point of failure.
- Redundant RRs allow clients to maintain multiple IBGP sessions.
- Clients do not need to establish a full mesh with each other.
- Redundant RRs can use the same or different cluster IDs.
- **Same cluster IDs** cause an RR to reject routes already reflected by another RR in the shared cluster.
- **Different cluster IDs** can allow RRs to accept and reflect routes learned through each other.
- Different cluster IDs can improve resilience during asymmetric client-to-RR session failures.
- Using different cluster IDs does not guarantee that the inter-RR path will be selected as the best path.
- In conventional designs, RRs establish IBGP sessions with each other as non-clients.
- RRs can technically be clients of other RRs, particularly in hierarchical designs.
- Mutually configuring peer-level redundant RRs as clients of each other is generally unnecessary.
- An RR-to-RR IBGP session is not always required for basic redundancy, but can provide additional route distribution paths.
- RRs perform best-path selection independently and may advertise different paths.
- Redundant RRs do not eliminate BGP path hiding.
- Route reflection normally preserves the original BGP NEXT_HOP.
- BFD can improve BGP session failure detection.
- Control-plane redundancy does not automatically guarantee data-plane redundancy.
- Use `show ip bgp cluster-ids`, `show ip bgp summary`, and `show ip bgp <prefix>` to verify redundant RR operation.

---

## References

- [RFC 4456: BGP Route Reflection](https://www.rfc-editor.org/rfc/rfc4456.html)
- [Cisco: Configuring Internal BGP Features, IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-int-features-0.html)
- [Cisco: BGP Command Reference, bgp cluster-id](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-a1.html)
- [Cisco: Examine BGP Case Studies](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/26634-bgp-toc.html)
- [Cisco: Use BGP Route Reflection and Multiple Cluster IDs](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/200153-BGP-Route-Reflection-and-Multiple-Cluste.html)
