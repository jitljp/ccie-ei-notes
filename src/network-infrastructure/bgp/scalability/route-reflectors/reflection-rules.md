# Route Reflection Rules

A **BGP Route Reflector (RR)** allows routes learned from IBGP neighbors to be advertised to other IBGP neighbors, providing an exception to the normal **IBGP split-horizon rule**.

However, an RR does not advertise every route to every neighbor.

The advertisement rules depend on whether the route was learned from:

1. An **IBGP client**.
2. An **IBGP non-client**.
3. An **EBGP neighbor**.

These rules are defined in [RFC 4456](https://www.rfc-editor.org/rfc/rfc4456.html).

---

## Route Reflection Rules Overview

The three fundamental rules are:

| Route Learned From | Advertise to Clients | Advertise to Non-Clients |
|---|---|---|
| IBGP Client | Yes | Yes |
| IBGP Non-Client | Yes | No |
| EBGP Neighbor | Yes | Yes |

**Important:** These rules assume that the route is selected as the best path, is otherwise eligible for advertisement, and is not denied by outbound policies or loop prevention.

An RR does not normally advertise a route back to the neighbor from which it learned the route.

### Example Topology

Consider the following IBGP topology:

```text
                       AS 65000

               R2                 R3
             Client             Client
                  \             /
                   \           /
                    \         /
                      R1 (RR)
                     /       \
                    /         \
                   /           \
                 R4-------------R5
             Non-Client     Non-Client

            All connections are IBGP
```

R1 is the route reflector.

Its IBGP neighbors are:

| Router | Relationship to R1 |
|---|---|
| R2 | Client |
| R3 | Client |
| R4 | Non-client |
| R5 | Non-client |

R2 and R3 do not require an IBGP session with each other because R1 can reflect routes between them.

R4 and R5 are fully meshed because they are non-clients.

**A route reflector does not remove the full-mesh requirement between non-clients.** In larger deployments, non-client IBGP speakers must still have appropriate full-mesh connectivity or another route distribution mechanism.

---

## Rule 1: Routes Learned from IBGP Clients

When an RR learns a route from an **IBGP client**, it can reflect the route to:

- Other IBGP clients.
- IBGP non-clients.

For example, suppose R2 advertises `192.0.2.0/24` to R1.

```text
                  R2 (Client)
                       |
                       | 192.0.2.0/24
                       v
                    R1 (RR)
                   /   |   \
                  v    v    v
                 R3    R4    R5
```

R1 can reflect the route to R3, R4, and R5.

### Client-to-Client Reflection

R1 reflects the route learned from R2 to R3.

This is called **client-to-client reflection**.

Because R1 performs this function, R2 and R3 do not need to establish an IBGP session with each other.

Client-to-client reflection is enabled by default on Cisco IOS XE.

### Client-to-Non-Client Reflection

R1 also reflects the route learned from R2 to R4 and R5.

This is called **client-to-non-client reflection**.

Non-clients can therefore learn routes originated by RR clients even though they do not have direct IBGP sessions with those clients.

### Configuration

On R1:

```text
R1(config)# router bgp 65000
R1(config-router)# neighbor 2.2.2.2 remote-as 65000
R1(config-router)# neighbor 3.3.3.3 remote-as 65000
R1(config-router)# neighbor 4.4.4.4 remote-as 65000
R1(config-router)# neighbor 5.5.5.5 remote-as 65000

R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 2.2.2.2 route-reflector-client
R1(config-router-af)# neighbor 3.3.3.3 route-reflector-client
```

R2 and R3 are clients.

R4 and R5 are non-clients because they have not been configured as route reflector clients.

The example assumes the IBGP sessions have already been activated and any necessary `update-source` commands have been configured.

**An RR does not require a separate command to enable client-to-non-client reflection.**

---

## Rule 2: Routes Learned from IBGP Non-Clients

When an RR learns a route from an **IBGP non-client**, it can reflect the route to:

- IBGP clients.

However, it **cannot reflect the route to other IBGP non-clients**.

For example, suppose R4 advertises `198.51.100.0/24` to R1.

```text
                 R4 (Non-Client)
                       |
                       | 198.51.100.0/24
                       v
                    R1 (RR)
                   /       \
                  v         v
                 R2         R3
              Client      Client

              R5: Not reflected
```

R1 can reflect the route to R2 and R3.

However, R1 cannot reflect the IBGP-learned route to R5.

Why?

Because R4 and R5 are both non-clients.

The ordinary IBGP split-horizon rule still applies to advertisements between non-clients.

### How Does R5 Learn the Route?

R5 must learn the route through another mechanism, normally its direct IBGP session with R4.

```text
             R4 ---------------- R5
           Non-Client         Non-Client
                      IBGP
```

This explains why non-clients must maintain the necessary IBGP full-mesh relationships.

### Important Distinction

The RR's ability to reflect a route is determined by the **relationship between the RR and the neighbor that advertised the selected path**.

It does not depend on whether the route was originally introduced into the AS by a client or non-client elsewhere.

For example, consider:

```text
R2 (Client) ---- R1 (RR) ---- R4 (Non-Client)
                                  |
                                  | IBGP
                                  |
                            R5 (Non-Client)
```

Suppose R5 originates a prefix and advertises it to R4.

R4 then attempts to advertise the route to R1.

However, R4 learned the route through IBGP from R5.

Because R4 is not an RR for R5, the normal IBGP split-horizon rule prevents R4 from advertising that route to R1 in the first place.

R5 must advertise the route directly to R1 or through another valid route reflection path.

---

## Rule 3: Routes Learned from EBGP Neighbors

When a router learns a route from an **EBGP neighbor**, it can advertise that route to all eligible IBGP neighbors, including both clients and non-clients.

For example:

```text
          R6
           |
           | EBGP
           |
        R1 (RR)
        /  |  \
       /   |   \
      R2   R3   R4
    Client Client Non-Client
```

Suppose R6 advertises `203.0.113.0/24` to R1.

R1 can advertise the route to R2, R3, and R4.

**This is ordinary EBGP-to-IBGP route advertisement, not route reflection in the strict sense.**

The RR configuration is not needed for R1 to advertise its EBGP-learned route to IBGP neighbors.

A normal IBGP router can already perform this function.

Route reflection specifically provides an exception that permits an **IBGP-learned route to be advertised to another IBGP neighbor**.

### NEXT_HOP Considerations

When R1 advertises an EBGP-learned route to IBGP neighbors, the original EBGP NEXT_HOP is normally preserved unless next-hop modification is configured.

For example:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 2.2.2.2 next-hop-self
```

This causes R1 to advertise itself as the BGP next hop for eligible routes advertised to R2, including routes learned from EBGP.

In contrast, the ordinary `next-hop-self` command does not rewrite the next hop of reflected IBGP routes.

On IOS XE, `next-hop-self all` can also rewrite the next hop of reflected routes.

See [Route Reflector Operation and Configuration](configuration.md) for more details.

---

## Locally Originated Routes

When an RR originates a route locally, it can advertise that route to both clients and non-clients.

For example, suppose R1 originates `10.10.10.0/24` using a BGP `network` statement.

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# network 10.10.10.0 mask 255.255.255.0
```

Assuming the exact prefix exists in R1's routing table, R1 can originate the route and advertise it to R2, R3, R4, and R5.

This is not route reflection because R1 originated the route itself.

The route reflector role does not restrict normal advertisement of locally originated routes.

---

## Disabling Client-to-Client Reflection

By default, an RR reflects routes learned from one client to other clients.

However, Cisco IOS XE allows this behavior to be disabled.

### Configuration

On R1:

```text
R1(config)# router bgp 65000
R1(config-router)# no bgp client-to-client reflection
```

This disables client-to-client route reflection.

To restore the default behavior:

```text
R1(config-router)# bgp client-to-client reflection
```

The command is configured at the **BGP router configuration level**, not under an address family.

### Effect on Route Advertisements

Consider:

```text
                     R1 (RR)
                    /       \
                   /         \
                 R2           R3
               Client       Client
```

Suppose R2 advertises `192.0.2.0/24` to R1.

By default:

- R1 receives the route from R2.
- R1 selects it as the best path.
- R1 reflects it to R3.

After configuring:

```text
R1(config-router)# no bgp client-to-client reflection
```

R1 no longer reflects the route from R2 to R3.

R3 therefore needs another mechanism to learn the route.

### Client-to-Non-Client Reflection Still Works

Disabling client-to-client reflection does **not** prevent routes learned from clients from being reflected to non-clients.

For example:

```text
                 R2 (Client)
                      |
                      v
                   R1 (RR)
                  /       \
                 X         v
                R3         R4
              Client    Non-Client
```

With client-to-client reflection disabled:

- R1 does not reflect R2's route to R3.
- R1 can still reflect R2's route to R4.

Similarly, routes learned from non-clients can still be reflected to clients.

### Updated Reflection Rules

With client-to-client reflection disabled:

| Route Learned From | Advertise to Clients | Advertise to Non-Clients |
|---|---|---|
| IBGP Client | No | Yes |
| IBGP Non-Client | Yes | No |
| EBGP Neighbor | Yes | Yes |

In the first row, "No" means other clients, not the router that advertised the route.

### When Should Client-to-Client Reflection Be Disabled?

The most common reason is that the RR clients are already fully meshed.

For example:

```text
                     R1 (RR)
                    /       \
                   /         \
                  R2---------R3
                       IBGP
```

R2 and R3 have a direct IBGP session.

They can exchange their locally originated and EBGP-learned routes directly, so reflection between them may be unnecessary.

Disabling client-to-client reflection can reduce redundant advertisements and update processing.

However, it should only be disabled when clients have the necessary alternative route distribution paths.

**Disabling client-to-client reflection does not remove the RR client designation or disable the IBGP sessions.**

---

## Route Reflectors and Best-Path Selection

An important rule is that **an RR normally reflects only its selected best path**.

It does not automatically reflect every path received from its clients and non-clients.

Consider the following scenario:

```text
           R2 (Client)        R3 (Client)
                \              /
                 \            /
                  \          /
                     R1 (RR)
                        |
                        |
                  R4 (Client)
```

R2 and R3 both advertise routes to `192.0.2.0/24`.

Suppose R1 receives:

| Source | Prefix | LOCAL_PREF | AS_PATH |
|---|---|---|---|
| R2 | 192.0.2.0/24 | 200 | 65010 |
| R3 | 192.0.2.0/24 | 100 | 65020 |

R1 selects the path received from R2 because it has the higher LOCAL_PREF.

R1 reflects the R2 path to R3 and R4.

R1 does not normally reflect the alternative R3 path to R2 or R4.

### Path Hiding

This behavior can result in **path hiding**.

R4 receives only the path selected by R1, even though R3 has an alternative path to the same destination.

R4 might have preferred R3's path if it had received both paths and performed best-path selection independently.

This is one of the fundamental tradeoffs of using route reflectors instead of a full IBGP mesh.

Features such as BGP Additional Paths can provide additional path visibility in supported deployments, but standard route reflection advertises only the selected best path.

### The Source of the Best Path Matters

Suppose R1 receives two routes to the same prefix:

- One from R2, an RR client.
- One from R4, an RR non-client.

If R1 selects the route from R2, it can reflect the route to other clients and non-clients.

If R1 selects the route from R4, it can reflect the route only to clients.

**The route reflection rules apply to the selected best path, not independently to every candidate path.**

---

## Route Reflection and BGP Attributes

When an RR reflects an IBGP route, it normally preserves the route's existing path attributes.

RFC 4456 specifically recommends that an RR not modify:

- NEXT_HOP
- AS_PATH
- LOCAL_PREF
- MED

Modifying these attributes during reflection can cause inconsistent routing decisions or routing loops.

### Route Reflection Attributes

To provide loop prevention, an RR uses two additional BGP path attributes:

| Attribute | Type Code | Purpose |
|---|---|---|
| ORIGINATOR_ID | 9 | Identifies the original BGP speaker within the AS |
| CLUSTER_LIST | 10 | Records the clusters through which the route was reflected |

Both are optional, non-transitive attributes.

For example:

```text
R2 (Client) ---- R1 (RR) ---- R3 (Client)
```

If R2's BGP router ID is `2.2.2.2` and R1's cluster ID is `1.1.1.1`, a route reflected from R2 through R1 to R3 carries:

```text
ORIGINATOR_ID: 2.2.2.2
CLUSTER_LIST:  1.1.1.1
```

These attributes help prevent reflected routes from circulating indefinitely.

They can also affect BGP best-path tie-breaking.

See [Cluster IDs, Originator IDs, and Loop Prevention](loop-prevention.md) for details.

### Outbound Route Maps

Cisco IOS XE restricts attribute modifications through outbound route maps applied to reflected routes.

Most outbound `set` clauses are ignored for reflected IBGP routes.

A notable exception is:

```text
set ip next-hop <IP>
```

This can modify the NEXT_HOP attribute of a reflected route.

Normal route filtering policies can still control whether routes are advertised to particular neighbors.

---

## Verification

The most important verification commands are:

```text
R1# show ip bgp summary
R1# show ip bgp neighbors
R1# show ip bgp 192.0.2.0/24
R1# show ip bgp neighbors 3.3.3.3 advertised-routes
R1# show ip bgp cluster-ids
R1# show running-config | section router bgp
```

### Verifying Client Relationships

Use:

```text
R1# show ip bgp neighbors 2.2.2.2
```

Check that the neighbor is identified as an RR client.

A neighbor without the RR client designation is treated as a non-client.

### Verifying Advertisements

To verify whether R1 advertises a route to R3:

```text
R1# show ip bgp neighbors 3.3.3.3 advertised-routes
```

If the route is missing, check:

1. Is the IBGP session Established?
2. Is the selected best path learned from a client or non-client?
3. Is the destination neighbor a client or non-client?
4. Is client-to-client reflection enabled?
5. Is the route selected as the best path?
6. Is an outbound policy preventing advertisement?
7. Is the route rejected because of loop prevention attributes?

### Verifying Reflected Routes

On the receiving client:

```text
R3# show ip bgp 192.0.2.0/24
```

For a reflected route, the output can display:

```text
Originator: 2.2.2.2, Cluster list: 1.1.1.1
```

This confirms that the route was reflected and identifies its originator and the cluster through which it passed.

---

## Route Reflection Scenarios

The following scenarios summarize the reflection rules.

Assume R1 is an RR, R2 and R3 are clients, and R4 and R5 are non-clients.

| Scenario | Reflected by R1? | Explanation |
|---|---|---|
| R2 to R3 | Yes | Client to client |
| R2 to R4 | Yes | Client to non-client |
| R4 to R2 | Yes | Non-client to client |
| R4 to R5 | No | Non-client to non-client |
| R2 to R3 with client-to-client reflection disabled | No | Client-to-client reflection disabled |
| R2 to R4 with client-to-client reflection disabled | Yes | Client-to-non-client reflection unaffected |
| R4 to R2 with client-to-client reflection disabled | Yes | Non-client-to-client reflection unaffected |

These examples assume that the path learned from the source neighbor is R1's eligible best path.

---

## Key Points

- An RR provides an exception to the normal IBGP split-horizon rule.
- Routes learned from **IBGP clients** can be reflected to other clients and non-clients.
- Routes learned from **IBGP non-clients** can be reflected only to clients.
- **EBGP-learned routes** can be advertised to all IBGP neighbors under normal BGP advertisement rules.
- **Locally originated routes** can be advertised to clients and non-clients.
- Client-to-client reflection is enabled by default.
- `no bgp client-to-client reflection` disables reflection between clients without affecting other reflection scenarios.
- Non-clients must maintain the necessary full-mesh IBGP relationships.
- RRs normally reflect only their selected best path.
- The relationship between the RR and the source of the selected path determines which IBGP neighbors can receive the reflected route.
- Reflected routes normally preserve NEXT_HOP, AS_PATH, LOCAL_PREF, and MED.
- ORIGINATOR_ID and CLUSTER_LIST provide additional loop prevention.

---

## References

- [RFC 4456: BGP Route Reflection](https://www.rfc-editor.org/rfc/rfc4456.html)
- [Cisco: Configuring Internal BGP Features, IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-int-features-0.html)
- [Cisco: BGP Command Reference, bgp client-to-client reflection](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-a1.html)
- [Cisco: BGP Multiple Cluster IDs](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-multicluster-id.html)
