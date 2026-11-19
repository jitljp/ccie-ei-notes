# 1.5.e (i) Route Reflectors

**BGP Route Reflectors (RRs)** are a scalability mechanism that eliminates the requirement for a full mesh of IBGP sessions within an autonomous system.

Normally, an IBGP router does not advertise routes learned from one IBGP neighbor to another IBGP neighbor. This is known as the **IBGP split-horizon rule**.

Route reflectors provide an exception to this rule, allowing selected IBGP routes to be advertised between IBGP neighbors without requiring every router to establish a session with every other router.

Route reflection is defined in **RFC 4456**.

---

## The IBGP Full-Mesh Requirement

By default, IBGP requires a full mesh of peer relationships to ensure that all routers within an AS can learn externally originated BGP routes.

Consider an AS containing four BGP routers:

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

       All connections are IBGP sessions
```

Each router establishes an IBGP session with every other router.

For `n` routers, the number of required IBGP sessions is:

```text
IBGP sessions = n * (n - 1) / 2
```

For example:

| IBGP Routers | Full-Mesh Sessions |
|---|---|
| 4 | 6 |
| 10 | 45 |
| 50 | 1,225 |
| 100 | 4,950 |
| 500 | 124,750 |

As the number of BGP routers increases, the number of required sessions grows quadratically.

Each additional session consumes resources for maintaining TCP connections, exchanging BGP messages, and managing neighbor-specific routing information.

This creates significant scalability problems in large networks.

---

## How Route Reflectors Solve the Problem

A **route reflector** is an IBGP router that is permitted to advertise routes learned from certain IBGP neighbors to other IBGP neighbors.

Its neighbors are classified into two categories:

- **Route Reflector Clients:** IBGP neighbors explicitly configured as clients of the RR.
- **Non-Clients:** Regular IBGP neighbors that are not configured as RR clients.

A route reflector and its clients form a **cluster**.

### Example Topology

Consider the same four routers, but with R1 acting as the route reflector.

```text
                   AS 65000

                    R1
               Route Reflector
                 /    |    \
                /     |     \
               /      |      \
              R2      R3      R4
            Client  Client  Client

          All connections are IBGP sessions
```

R2, R3, and R4 establish IBGP sessions with R1.

They do not need IBGP sessions with each other.

The number of required sessions is reduced from six to three.

For a simple topology with one route reflector and `n - 1` clients:

```text
IBGP sessions = n - 1
```

This greatly reduces the number of BGP sessions required as the network grows.

**Important:** Route reflection changes BGP route advertisement behavior, not the physical topology. The routers must still have IP reachability to their BGP next hops and neighbors.

---

## Route Reflector Operation

Suppose R2 learns the prefix `203.0.113.0/24` from an EBGP neighbor.

Without route reflection, R2 would not be permitted to advertise an IBGP-learned route to another IBGP neighbor. However, R2 can advertise its EBGP-learned route directly to R1, and R1 can reflect that route to other clients.

The process is:

1. R2 learns `203.0.113.0/24` from an EBGP neighbor.
2. R2 advertises the route to R1 through IBGP.
3. R1 receives the route and performs BGP best-path selection.
4. If R1 selects the route as its best path, it can reflect the route to R3 and R4.
5. R3 and R4 learn the prefix without establishing an IBGP session directly with R2.

Route reflectors still perform normal BGP best-path selection.

**By default, an RR reflects only its selected best path for a prefix, not every path it learns.**

This behavior is important because it can create **path hiding**, where some potentially useful alternative paths are not advertised to clients.

---

## Route Reflection Rules

A route reflector follows specific rules when advertising routes to its IBGP neighbors.

The rules depend on where the route was learned.

| Route Learned From | Advertise To |
|---|---|
| EBGP neighbor | IBGP clients and non-clients |
| IBGP client | Other IBGP clients and non-clients |
| IBGP non-client | IBGP clients only |

These rules assume that the route is eligible for advertisement after BGP best-path selection and outbound policy processing.

### Important Considerations

- A route learned from an **IBGP client** can be reflected to both clients and non-clients.
- A route learned from an **IBGP non-client** can be reflected to clients, but not to other non-clients.
- Routes learned from EBGP neighbors follow normal BGP advertisement rules.
- A route reflector does not ordinarily advertise a route back to the neighbor from which it learned that route.
- Client-to-client reflection is enabled by default on Cisco IOS XE.

Client-to-client reflection can be disabled using:

```text
R1(config)# router bgp 65000
R1(config-router)# no bgp client-to-client reflection
```

When disabled, the RR no longer reflects routes learned from one client to other clients.

This is generally appropriate only when clients already have another mechanism for exchanging the required routing information, such as a full mesh of IBGP sessions.

See [Route Reflection Rules](reflection-rules.md) for a detailed explanation of all advertisement scenarios.

---

## Basic Route Reflector Configuration

Cisco IOS XE uses the `neighbor route-reflector-client` command to configure a route reflector relationship.

Consider the previous topology, where R1 is the RR and R2, R3, and R4 are its clients.

Assume the following loopback addresses:

| Router | Loopback0 |
|---|---|
| R1 | 1.1.1.1 |
| R2 | 2.2.2.2 |
| R3 | 3.3.3.3 |
| R4 | 4.4.4.4 |

All routers belong to AS 65000, and their loopback addresses are reachable through the IGP.

### R1: Route Reflector

```text
R1(config)# router bgp 65000
R1(config-router)# bgp router-id 1.1.1.1
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
R1(config-router-af)# neighbor 2.2.2.2 route-reflector-client
R1(config-router-af)# neighbor 3.3.3.3 route-reflector-client
R1(config-router-af)# neighbor 4.4.4.4 route-reflector-client
```

The three `route-reflector-client` commands identify R2, R3, and R4 as clients.

### R2: Route Reflector Client

```text
R2(config)# router bgp 65000
R2(config-router)# bgp router-id 2.2.2.2
R2(config-router)# neighbor 1.1.1.1 remote-as 65000
R2(config-router)# neighbor 1.1.1.1 update-source Loopback0
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 1.1.1.1 activate
```

R3 and R4 use equivalent configurations, with their own router IDs.

**Important:** The `route-reflector-client` command is configured only on the RR. Clients do not need special client-side configuration.

They treat their route reflector as an ordinary IBGP neighbor.

See [Route Reflector Operation and Configuration](configuration.md) for more detailed configuration and verification.

---

## Route Reflector Loop Prevention

The IBGP split-horizon rule ordinarily prevents routes learned through IBGP from being advertised to other IBGP neighbors.

Because route reflectors provide an exception to this rule, additional mechanisms are needed to prevent routing information loops.

BGP introduces two path attributes specifically for route reflection:

| Attribute | Type Code | Purpose |
|---|---|---|
| ORIGINATOR_ID | 9 | Identifies the original BGP speaker within the AS |
| CLUSTER_LIST | 10 | Records the clusters through which a route has been reflected |

Both attributes are **optional, non-transitive** BGP path attributes.

### ORIGINATOR_ID

When a route reflector reflects a route, it adds an `ORIGINATOR_ID` attribute if one is not already present.

This attribute contains the BGP router ID of the router that originally introduced the route into IBGP within the AS.

For example:

```text
R2 -------- R1 -------- R3
          (RR)

R2 BGP Router ID: 2.2.2.2
```

When R1 reflects an IBGP route originated by R2 to R3, it sets:

```text
ORIGINATOR_ID: 2.2.2.2
```

If R2 later receives a reflected route whose `ORIGINATOR_ID` equals its own router ID, it rejects the route.

### CLUSTER_LIST

Each route reflector belongs to a cluster identified by a **CLUSTER_ID**.

By default, the cluster ID is the BGP router ID of the route reflector.

When an RR reflects a route, it prepends its local cluster ID to the route's `CLUSTER_LIST`.

If an RR receives a route whose `CLUSTER_LIST` already contains its local cluster ID, the route is rejected.

This prevents routes from circulating repeatedly through the same route reflector cluster.

See [Cluster IDs, Originator IDs, and Loop Prevention](loop-prevention.md) for a detailed explanation.

---

## Multiple Route Reflectors and Redundancy

Using a single route reflector creates a potential single point of failure.

If the route reflector fails, its clients lose their IBGP sessions to it.

To provide redundancy, multiple route reflectors can serve the same clients.

For example:

```text
                    AS 65000

               R1             R2
               RR1            RR2
                |\           /|
                | \         / |
                |  \       /  |
                |   \     /   |
                |    \   /    |
                |     \ /     |
                |      X      |
                |     / \     |
                |    /   \    |
                |   /     \   |
                 R3         R4
                  RR Clients

       Each client peers with both RRs
```

R3 and R4 each establish IBGP sessions with both R1 and R2.

Either RR can continue advertising routes to the clients if the other fails, provided the surviving RR has the necessary routes.

### Cluster ID Configuration

When multiple route reflectors serve the same clients, they can be configured with either **the same cluster ID** or **different cluster IDs**.

Both designs are valid, but they have different implications for route reflection and loop prevention.

For example, to configure both RRs with the same cluster ID:

**R1:**
```text
R1(config)# router bgp 65000
R1(config-router)# bgp cluster-id 10.10.10.10
```

**R2:**
```text
R2(config)# router bgp 65000
R2(config-router)# bgp cluster-id 10.10.10.10
```

Alternatively, each RR can use a different cluster ID. By default, an RR uses its BGP router ID as its cluster ID.

The choice between **same-cluster** and **different-cluster** designs affects route advertisement behavior, loop prevention, path diversity, and redundancy.

See [Multiple Route Reflectors and Redundancy](redundancy.md) for a detailed comparison of both approaches.

---

## Route Reflector Scalability and Design Considerations

Route reflectors greatly improve IBGP scalability, but introduce certain tradeoffs.

### Advantages

- **Fewer IBGP sessions:** Clients do not need a full mesh.
- **Simpler configuration:** Each client can maintain sessions with a small number of RRs.
- **Lower session maintenance overhead:** Fewer TCP connections and BGP neighbor relationships are required.
- **Hierarchical designs:** Multiple route reflector clusters can be used to scale large networks.
- **Redundancy:** Multiple RRs can serve the same clients.

### Limitations

**Path Hiding**

An RR generally advertises only its selected best path.

If multiple alternative paths exist, clients may not learn all available paths. This can affect traffic engineering, path diversity, and convergence.

**Suboptimal Routing**

The RR chooses the best path from its own perspective.

For example, it may prefer one BGP next hop based on its IGP metric, even though a different next hop would be preferable for one of its clients.

**Additional Failure Dependencies**

If a client relies on a single RR and that RR fails, the client can lose BGP reachability information.

Redundant RRs mitigate this problem.

**Next-Hop Reachability**

Route reflection does not automatically make the RR the forwarding next hop.

By default, reflected routes retain their original `NEXT_HOP` attribute.

Therefore, clients must have IP reachability to the advertised BGP next hops.

On Cisco IOS XE, the normal `neighbor next-hop-self` command does not rewrite the next hop of routes reflected from IBGP clients. An outbound route map can be used when deliberate next-hop modification is required.

---

## Verification Commands

The following commands are useful when verifying route reflector operation.

| Command | Purpose |
|---|---|
| `show ip bgp summary` | Verify BGP neighbor sessions |
| `show ip bgp neighbors` | Examine neighbor details, including RR client relationships |
| `show ip bgp` | Examine learned routes and best paths |
| `show ip bgp <prefix>` | Examine individual paths and route reflection attributes |
| `show ip bgp cluster-ids` | Display route reflector cluster information |
| `show ip bgp neighbors <IP> advertised-routes` | Examine routes advertised to a neighbor |
| `show running-config \| section router bgp` | Verify BGP and route reflector configuration |

When troubleshooting route reflection, verify:

1. The IBGP sessions are established.
2. The correct neighbors are configured as RR clients.
3. The expected prefixes exist in the RR's BGP table.
4. The RR has selected an eligible best path.
5. The reflection rules permit advertisement to the destination neighbor.
6. The route is not rejected due to `ORIGINATOR_ID` or `CLUSTER_LIST` loop prevention.
7. Outbound routing policies are not filtering the route.
8. The receiving router can reach the BGP next hop.

---

## Key Points

- Route reflectors eliminate the requirement for a full mesh of IBGP sessions.
- An RR can advertise selected IBGP-learned routes to other IBGP neighbors.
- RR neighbors are classified as **clients** or **non-clients**.
- A route learned from a client can be reflected to other clients and non-clients.
- A route learned from a non-client can be reflected only to clients.
- Client-to-client reflection is enabled by default.
- RRs normally reflect only their selected best path for each prefix.
- `ORIGINATOR_ID` and `CLUSTER_LIST` prevent routing information loops.
- Clients do not require special route reflector configuration.
- Redundant RRs can serve the same clients using a common cluster ID.
- Route reflection improves scalability but can introduce path hiding and suboptimal routing.

---

## Topics Covered

The following pages explore route reflectors in greater detail:

1. [Route Reflector Operation and Configuration](configuration.md)
   - Configuring RRs and clients.
   - Route reflector behavior.
   - Verification and troubleshooting.

2. [Route Reflection Rules](reflection-rules.md)
   - Client-to-client reflection.
   - Client-to-non-client reflection.
   - Non-client-to-client reflection.
   - Disabling client-to-client reflection.

3. [Cluster IDs, Originator IDs, and Loop Prevention](loop-prevention.md)
   - Cluster IDs.
   - ORIGINATOR_ID.
   - CLUSTER_LIST.
   - Loop prevention and best-path tie-breaking.

4. [Multiple Route Reflectors and Redundancy](redundancy.md)
   - Redundant RRs.
   - Same-cluster and different-cluster designs.
   - RR-to-RR peering.
   - Redundancy and convergence considerations.

---

## References

- [RFC 4456: BGP Route Reflection](https://www.rfc-editor.org/rfc/rfc4456.html)
- [Cisco: Configuring Internal BGP Features (IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-int-features-0.html)
- [Cisco: Use BGP Route Reflection and Multiple Cluster IDs](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/200153-BGP-Route-Reflection-and-Multiple-Cluste.html)
