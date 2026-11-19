# 1.5.e Convergence and Scalability

**BGP convergence and scalability** refer to how efficiently BGP responds to network changes and how well it operates as the number of routers, BGP sessions, and advertised prefixes increases.

BGP is designed to exchange routing information between autonomous systems, but it is also widely used within large enterprise and service provider networks.

As networks grow, BGP must handle:

- **Increasing numbers of BGP peers:** More sessions consume additional memory, CPU resources, and control-plane bandwidth.
- **Large routing tables:** Processing and storing more prefixes increases resource requirements.
- **Routing updates:** Network changes can trigger BGP UPDATE messages, best-path recalculations, and route withdrawals.
- **Convergence:** Routers must detect changes, select alternative paths, and update their forwarding tables.
- **Redundancy:** Multiple paths and BGP speakers must be coordinated to maintain connectivity during failures.

Two important mechanisms for improving BGP scalability are:

1. **Route reflectors:** Reduce the number of required IBGP sessions.
2. **Route aggregation:** Reduce the number of prefixes advertised to other BGP routers.

These mechanisms address different scalability challenges.

---

## BGP Convergence

**BGP convergence** is the process by which BGP routers reach a consistent routing state following a network change.

For example, when a BGP neighbor fails:

1. The failure must be detected.
2. Routes learned through the failed neighbor become unavailable.
3. BGP selects alternative paths, if available.
4. The selected routes are installed in the routing table, subject to RIB installation rules.
5. Routing changes are advertised to appropriate BGP neighbors.

The time required for these operations contributes to BGP convergence time.

### Factors Affecting Convergence

Several factors influence BGP convergence.

| Factor | Effect |
|---|---|
| Neighbor failure detection | Determines how quickly a failed neighbor or path is detected |
| BGP timers | Influence session failure detection and update processing |
| BGP table size | Affects the amount of routing information that must be processed |
| Best-path selection | Determines which alternative paths are selected |
| Routing policies | Can affect route eligibility and advertisement |
| Route reflectors | Influence the distribution of routing information within an AS |
| Route aggregation | Can reduce the number of updates propagated through the network |

**Scalability and convergence are closely related**, but they are not identical.

A design that reduces the number of BGP sessions or advertised prefixes may improve resource efficiency without necessarily providing faster convergence for every failure.

For example, route reflectors reduce the number of required IBGP sessions, but they can also affect path visibility and the availability of alternative paths.

BGP neighbor failure detection, BFD, and other fast-convergence mechanisms are covered separately in [BGP Fast Convergence and BFD](../other-features/fast-convergence.md).

---

## BGP Scalability

**BGP scalability** refers to the ability of a BGP deployment to support increasing numbers of routers, peers, and routes without excessive resource consumption.

Two major scalability concerns are:

- **IBGP session scalability:** The number of BGP sessions required within an autonomous system.
- **Routing information scalability:** The number of prefixes that must be stored, processed, and advertised.

Route reflectors primarily address the first concern.

Route aggregation primarily addresses the second.

---

## 1.5.e (i) Route Reflectors

By default, IBGP routers do not advertise routes learned from one IBGP neighbor to another IBGP neighbor.

This is known as the **IBGP split-horizon rule**.

Consequently, a traditional IBGP deployment requires a **full mesh** of IBGP sessions to distribute routing information throughout an autonomous system.

### IBGP Full Mesh

Consider four routers in AS 65000:

```text
               AS 65000

          R1-------------R2
           |\           /|
           | \         / |
           |  \       /  |
           |   \     /   |
           |    \   /    |
           |     \ /     |
           |     / \     |
           |    /   \    |
           |   /     \   |
           |  /       \  |
           | /         \ |
           |/           \|
          R3-------------R4

          All links: IBGP
```

Each router must establish an IBGP session with every other router.

The number of sessions required for a full mesh is:

**IBGP sessions = n × (n - 1) / 2**

Where `n` is the number of IBGP routers.

| IBGP Routers | Required Sessions |
|---|---|
| 3 | 3 |
| 4 | 6 |
| 5 | 10 |
| 10 | 45 |
| 50 | 1,225 |
| 100 | 4,950 |

The number of sessions increases quadratically as the number of routers grows.

This becomes increasingly difficult to configure, maintain, and scale.

### Route Reflector Solution

A **BGP route reflector (RR)** allows routes learned through IBGP to be advertised to other IBGP neighbors under specific reflection rules.

This eliminates the requirement for every IBGP router to establish a session with every other router.

Consider the following topology:

```text
                   AS 65000

                      R1
                 Route Reflector
                   /    |    \
                  /     |     \
                 /      |      \
                R2      R3      R4
              Client  Client  Client

               All links: IBGP
```

R1 is configured as a route reflector.

R2, R3, and R4 are configured as its **route reflector clients**.

Instead of six IBGP sessions, only three are required.

When R2 advertises a route to R1:

1. R1 receives the route from R2.
2. R1 evaluates it using the BGP best-path selection process.
3. If the route is selected as the best path, R1 can reflect it to R3 and R4.
4. R3 and R4 can learn the route without establishing an IBGP session directly with R2.

By default, route reflectors advertise their selected best path rather than every available path.

### Basic Configuration

On R1:

```text
R1(config)# router bgp 65000
R1(config-router)# neighbor 10.0.12.2 remote-as 65000
R1(config-router)# neighbor 10.0.13.3 remote-as 65000
R1(config-router)# neighbor 10.0.14.4 remote-as 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-reflector-client
R1(config-router-af)# neighbor 10.0.13.3 route-reflector-client
R1(config-router-af)# neighbor 10.0.14.4 route-reflector-client
```

The `neighbor route-reflector-client` command identifies an IBGP neighbor as a route reflector client.

**Important:** Route reflector clients do not require special client-side configuration. The route reflector relationship is configured on the RR.

Route reflection introduces additional loop-prevention mechanisms:

- **ORIGINATOR_ID:** Identifies the router that originally introduced a route into the local AS.
- **CLUSTER_LIST:** Records the route reflector clusters through which a reflected route has passed.

These attributes help prevent routing loops when routes are reflected between IBGP routers.

For redundancy, multiple route reflectors can serve the same clients.

However, route reflector designs must account for potential path hiding, suboptimal routing, and the impact of route reflector failures.

See [Route Reflectors](route-reflectors/overview.md) for details on configuration, reflection rules, loop prevention, and redundancy.

---

## 1.5.e (ii) Aggregation and AS-SET

**BGP route aggregation** combines multiple more-specific prefixes into a single, less-specific prefix.

This reduces the amount of routing information that needs to be advertised and potentially stored by other BGP routers.

### Example

Suppose R1 advertises the following prefixes to R2:

```text
192.0.2.0/24
192.0.3.0/24
```

These prefixes can be summarized as:

```text
192.0.2.0/23
```

Instead of advertising both /24 prefixes, R2 can advertise a single /23 aggregate.

```
       AS 65001           AS 65002           AS 65003

          R1 --------------- R2 --------------- R3
                    EBGP              EBGP

    192.0.2.0/24       Aggregation
    192.0.3.0/24            |
                           v
                    192.0.2.0/23  ------------>
```

By advertising one aggregate instead of multiple individual routes, BGP can reduce:

- The number of prefixes advertised to neighbors.
- Routing table entries on routers receiving only the aggregate.
- BGP UPDATE processing associated with changes to individual prefixes.
- Memory and CPU requirements associated with maintaining large routing tables.

### Basic Configuration

Cisco IOS XE supports route aggregation using the `aggregate-address` command.

For example, on R2:

```text
R2(config)# router bgp 65000
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# aggregate-address 192.0.2.0 255.255.254.0 summary-only
```

This creates the aggregate `192.0.2.0/23`.

The `summary-only` keyword suppresses advertisement of the contributing more-specific routes.

**Important:** For the aggregate to be generated, at least one matching more-specific route must exist in the local BGP table.

The `aggregate-address` command does not require all of the more-specific prefixes covered by the aggregate to exist.

Without `summary-only`, both the aggregate and the contributing more-specific routes are advertised.

### AS-SET

A potential problem with aggregation is that the aggregate does not necessarily preserve the AS_PATH information of the individual routes.

For example, suppose R2 learns:

```text
192.0.2.0/24    AS_PATH: 65001
192.0.3.0/24    AS_PATH: 65002
```

Without `as-set`, R2 creates an aggregate without preserving these individual AS_PATHs.

This means that information about the autonomous systems traversed by the contributing routes can be lost.

The `as-set` keyword allows the aggregate to preserve ASN information from the contributing routes.

```text
R2(config-router-af)# aggregate-address 192.0.2.0 255.255.254.0 as-set summary-only
```

The aggregate can include an AS_SET containing the contributing ASNs:

```text
AS_PATH: {65001,65002}
```

An AS_SET is an **unordered collection of ASNs**.

It does not preserve the original ordering of the contributing AS_PATHs.

The AS_SET provides additional information for **AS_PATH loop prevention**.

If a router receives an aggregate containing its own ASN in the AS_SET, normal AS_PATH loop detection can reject the route.

### Aggregation and Convergence

Aggregation can reduce BGP update processing by hiding changes to individual prefixes from downstream routers.

However, aggregation introduces tradeoffs:

- The aggregate may remain advertised when some contributing routes disappear.
- More-specific routing information is lost when individual routes are suppressed.
- Traffic matching an aggregate but not a reachable contributing prefix may be discarded.
- Using `as-set` can cause the aggregate's attributes to change when contributing routes change, potentially generating additional updates.

Therefore, aggregation improves scalability but does not necessarily improve convergence in every situation.

See [Aggregation and AS-SET](aggregation/overview.md) for detailed configuration and attribute handling.

---

## Route Reflectors vs. Route Aggregation

Although both features improve BGP scalability, they address different problems.

| Feature | Route Reflectors | Route Aggregation |
|---|---|---|
| Primary purpose | Reduce IBGP session requirements | Reduce advertised prefixes |
| Main mechanism | Reflect IBGP routes between neighbors | Combine more-specific routes |
| Key configuration | `neighbor route-reflector-client` | `aggregate-address` |
| BGP sessions | Reduces required sessions | Does not reduce sessions |
| Advertised prefixes | Does not inherently reduce the number of prefixes | Can reduce advertised prefixes |
| Loop prevention | ORIGINATOR_ID and CLUSTER_LIST | AS_PATH and optionally AS_SET |
| Main tradeoff | Reduced path visibility and potential path hiding | Loss of more-specific routing information |

These mechanisms can be used together.

For example, a large enterprise might use route reflectors to simplify its internal BGP topology while advertising aggregated routes to external networks.

---

## Verification

The following commands are useful when verifying BGP convergence and scalability mechanisms.

### Route Reflectors

```text
R1# show ip bgp summary
R1# show ip bgp neighbors
R1# show ip bgp
R1# show ip bgp cluster-ids
```

These commands help verify:

- IBGP neighbor relationships.
- Route reflector client configuration.
- BGP route propagation.
- ORIGINATOR_ID and CLUSTER_LIST attributes.
- Route reflector cluster information.

### Route Aggregation

```text
R1# show ip bgp
R1# show ip bgp 192.0.2.0/23
R1# show ip bgp neighbors 10.0.12.2 advertised-routes
R1# show ip route 192.0.2.0 255.255.254.0
```

These commands help verify:

- Whether the aggregate exists in the BGP table.
- Whether contributing routes have been suppressed.
- Whether the aggregate is advertised to neighbors.
- Whether AS_SET information is present.
- How the aggregate is represented in the IP routing table.

---

## Topics Covered

This section covers two major BGP convergence and scalability mechanisms:

1. [Route Reflectors](route-reflectors/overview.md)
   - Route reflector operation and configuration.
   - Route reflection rules for clients and non-clients.
   - Cluster IDs, ORIGINATOR_ID, and CLUSTER_LIST.
   - Multiple route reflectors and redundancy.

2. [Aggregation and AS-SET](aggregation/overview.md)
   - BGP aggregate route configuration.
   - AS_SET and path attribute preservation.
   - `summary-only`, `suppress-map`, and `unsuppress-map`.
   - `advertise-map` and `attribute-map`.

---

## References

- [Cisco: Configuring Internal BGP Features (IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-int-features-0.html)
- [Cisco: BGP Route Reflection and Multiple Cluster IDs](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/200153-BGP-Route-Reflection-and-Multiple-Cluste.html)
- [Cisco: Understand Route Aggregation in BGP](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/5441-aggregation.html)
- [Cisco: Configuring a Basic BGP Network (IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-basic-net-0.html)
