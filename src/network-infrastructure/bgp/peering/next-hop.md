# BGP Next-Hop Handling

The **NEXT_HOP** path attribute identifies the IP address that a router should use to reach a BGP-advertised destination.

A BGP router must be able to **resolve the NEXT_HOP attribute** before it can use a received path. The next hop may be different from the address of the BGP neighbor that advertised the route.

The most important distinction is:

- **EBGP** normally changes the next hop to an address of the advertising router.
- **IBGP** normally preserves the next hop of a route learned through EBGP.

---

## Default Next-Hop Behavior

| Advertisement | Default NEXT_HOP behavior |
|---|---|
| EBGP-learned route advertised to an IBGP neighbor | Preserve the existing next hop |
| IBGP-learned route advertised to an EBGP neighbor | Change to the advertising router's address, subject to exceptions |
| EBGP-learned route advertised to another EBGP neighbor | Change to the advertising router's address, except for third-party next-hop behavior |
| Locally originated route advertised to an IBGP neighbor | Use an address of the advertising router |
| Locally originated route advertised to an EBGP neighbor | Use an address of the advertising router |
| IBGP-learned route advertised to another IBGP neighbor | Not advertised by default, unless route reflection or another supported mechanism permits it |

The next hop used on an advertisement is not necessarily the same as the **TCP source address** used for the BGP session. In simple, directly connected designs, they often coincide.

### Locally Originated Routes

In the *local* BGP table, a route originated using `network` commonly appears with next hop `0.0.0.0`:

```text
R1# show bgp ipv4 unicast

   Network          Next Hop            Metric LocPrf Weight Path
*> 192.0.2.0/24     0.0.0.0                  0         32768 i
```

Here, `0.0.0.0` indicates a **locally originated BGP path**. It does **not** mean that the router advertises `0.0.0.0` to its neighbors as the next hop.

---

## EBGP Next-Hop Behavior

When a router advertises a route to an EBGP neighbor, it normally sets NEXT_HOP to its own address on the path toward that neighbor.

Consider three autonomous systems:

```text
       AS 65001             AS 65002             AS 65003

         R1 ------------------ R2 ------------------ R3
     10.0.12.1        10.0.12.2
                               10.0.23.2         10.0.23.3

    192.0.2.0/24
```

R1 originates `192.0.2.0/24`.

| Router receiving the advertisement | NEXT_HOP | AS_PATH |
|---|---|---|
| R2 | `10.0.12.1` | `65001` |
| R3 | `10.0.23.2` | `65002 65001` |

R2 changes the next hop when advertising the route to R3. R3 therefore needs reachability to `10.0.23.2`, not necessarily to `10.0.12.1`.

This is the normal **next-hop-self behavior of EBGP**, even without a `neighbor next-hop-self` command.

### Third-Party Next Hop

There is an important exception when an EBGP router advertises a route whose existing next hop is on the same subnet as the receiving EBGP peer.

```text
                   Shared Ethernet: 10.0.123.0/24

          R1                    R2                    R3
       AS 65001              AS 65002              AS 65003
      10.0.123.1            10.0.123.2            10.0.123.3
          |                     |                     |
          +---------------------+---------------------+

      192.0.2.0/24
```

1. R1 advertises `192.0.2.0/24` to R2 with NEXT_HOP `10.0.123.1`.
2. R2 advertises the route to R3.
3. Because `10.0.123.1` is directly reachable by R3 on their shared subnet, R2 can **preserve `10.0.123.1`** rather than replace it with `10.0.123.2`.
4. R3 can send traffic directly to R1 instead of forwarding it through R2.

This is called a **third-party next hop**. It is an automatic EBGP behavior; it does not require `neighbor next-hop-unchanged`.

> Do not assume that an EBGP advertisement always carries the advertising router's address as NEXT_HOP. Check for the third-party exception.

---

## IBGP Next-Hop Behavior

An IBGP speaker normally **does not change** the NEXT_HOP attribute of a route learned through EBGP when advertising that route to an IBGP peer.

```text
            AS 65001                    AS 65002

              R1 ----------- R2 ---------------- R3
                 EBGP                 IBGP

        E0/0: 10.0.12.1  E0/0: 10.0.12.2
                        E0/1: 10.0.23.2   E0/0: 10.0.23.3

         192.0.2.0/24
```

R1 advertises `192.0.2.0/24` to R2 through EBGP:

```text
R2 receives:
  Prefix:   192.0.2.0/24
  NEXT_HOP: 10.0.12.1
  AS_PATH:  65001
```

R2 then advertises it to R3 through IBGP:

```text
R3 receives:
  Prefix:   192.0.2.0/24
  NEXT_HOP: 10.0.12.1
  AS_PATH:  65001
```

**R3 does not automatically use R2 (`10.0.23.2`) as the next hop.** It must resolve `10.0.12.1` to use the route.

There are two common solutions:

- Give R3 an IGP or static route to the original next hop.
- Configure R2 to advertise itself as the next hop with `neighbor next-hop-self`.

These solutions have different forwarding implications. Preserving the original next hop can avoid unnecessary traversal through a BGP-speaking intermediary when the underlying network provides another path.

---

## Resolving the BGP Next Hop

BGP performs a **recursive lookup** for the next-hop address in the routing table.

Using the previous topology, suppose R3 learns:

```text
BGP prefix: 192.0.2.0/24
BGP NEXT_HOP: 10.0.12.1
```

R3 might have the following route:

```text
O 10.0.12.0/30 [110/20] via 10.0.23.2, Ethernet0/0
```

Recursive resolution works as follows:

```text
Destination: 192.0.2.10
           |
           v
BGP route: 192.0.2.0/24
BGP NEXT_HOP: 10.0.12.1
           |
           v
RIB lookup: 10.0.12.1
           |
           v
Route: 10.0.12.0/30 via 10.0.23.2
           |
           v
Forward out Ethernet0/0 toward 10.0.23.2
```

In this example, R3 can use the BGP route **without changing the BGP NEXT_HOP attribute**.

The BGP next hop (`10.0.12.1`) and the immediate forwarding next hop (`10.0.23.2`) are different.

### Unreachable Next Hop

If R3 has no usable route to `10.0.12.1`, the path is not a valid candidate for best-path selection. The BGP session to R2 can remain **Established**, and the received prefix can still appear in the BGP table, but the path cannot be installed as an active BGP route.

Check the actual route and its reachability:

```text
R3# show bgp ipv4 unicast 192.0.2.0/24
R3# show ip route 10.0.12.1
```

**An unreachable NEXT_HOP is different from a RIB failure.** A RIB failure occurs when a BGP-selected path cannot be installed in the routing table, for example because another routing source has a lower administrative distance. An unresolved next hop can prevent the BGP path from being selected as best in the first place.

---

## Configuring `next-hop-self`

Use `neighbor next-hop-self` to advertise an address of the local BGP router as the NEXT_HOP to a specified peer.

In the preceding example, configure it on **R2**, under the IPv4 unicast address family, toward **R3**:

```text
R2(config)# router bgp 65002
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.23.3 next-hop-self
```

R3 now receives:

```text
Prefix:   192.0.2.0/24
NEXT_HOP: 10.0.23.2
AS_PATH:  65001
```

R3 can resolve `10.0.23.2` using its connected route to the R2-R3 subnet.

The command changes the **outbound BGP attribute**. It does not rewrite the received route in R2's own BGP table, create a route in R3's routing table, or change the TCP source of the BGP session.

### Full Configuration Example

The following basic lab assumes the interfaces have already been assigned the IPv4 addresses shown in the topology, and R1 has a connected `192.0.2.0/24` route.

**R1 (AS 65001):**

```text
R1(config)# router bgp 65001
R1(config-router)# no bgp default ipv4-unicast
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 activate
R1(config-router-af)# network 192.0.2.0 mask 255.255.255.0
```

**R2 (AS 65002):**

```text
R2(config)# router bgp 65002
R2(config-router)# no bgp default ipv4-unicast
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
R2(config-router)# neighbor 10.0.23.3 remote-as 65002
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.12.1 activate
R2(config-router-af)# neighbor 10.0.23.3 activate
R2(config-router-af)# neighbor 10.0.23.3 next-hop-self
```

**R3 (AS 65002):**

```text
R3(config)# router bgp 65002
R3(config-router)# no bgp default ipv4-unicast
R3(config-router)# neighbor 10.0.23.2 remote-as 65002
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# neighbor 10.0.23.2 activate
```

Verify the next hop on R3:

```text
R3# show bgp ipv4 unicast 192.0.2.0/24
R3# show ip route 192.0.2.0
```

If the route is eligible for installation, R3 should learn the BGP prefix with NEXT_HOP `10.0.23.2` and install it as an IBGP route (default administrative distance 200).

### Using Routing Instead of `next-hop-self`

To preserve the original next hop, remove `next-hop-self` on R2 and give R3 a route to the R1-R2 subnet:

```text
R2(config)# router bgp 65002
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# no neighbor 10.0.23.3 next-hop-self

R3(config)# ip route 10.0.12.0 255.255.255.252 10.0.23.2
```

Now the BGP NEXT_HOP on R3 can remain `10.0.12.1` while its immediate forwarding next hop is `10.0.23.2`.

In larger networks, an IGP usually supplies reachability to the BGP next-hop addresses. Avoid redistributing a large set of external BGP prefixes into the IGP merely to solve next-hop reachability.

---

## `next-hop-self` on Route Reflectors

A **route reflector (RR)** normally preserves the NEXT_HOP attribute when reflecting IBGP routes between clients.

A normal `neighbor next-hop-self` statement on an RR is **not sufficient to change the next hop of reflected IBGP-learned routes**. It primarily affects eligible non-reflected routes, including EBGP-learned routes advertised toward the IBGP neighbor.

Where supported by the platform and release, Cisco provides the `all` option to also change the next hop for reflected IBGP-learned routes:

```text
RR(config)# router bgp 65002
RR(config-router)# address-family ipv4 unicast
RR(config-router-af)# neighbor 10.0.0.3 next-hop-self all
```

| RR configuration | EBGP-learned paths advertised to client | Reflected IBGP-learned paths |
|---|---|---|
| No `next-hop-self` | Normally preserve next hop | Preserve next hop |
| `next-hop-self` | Set next hop to RR | Generally preserve next hop |
| `next-hop-self all` | Set next hop to RR | Set next hop to RR |

Cisco also documents using an **outbound route map** to explicitly set the next hop for reflected routes:

```text
RR(config)# route-map RR-NEXT-HOP permit 10
RR(config-route-map)# set ip next-hop 10.0.0.2

RR(config)# router bgp 65002
RR(config-router)# address-family ipv4 unicast
RR(config-router-af)# neighbor 10.0.0.3 route-map RR-NEXT-HOP out
```

In this example, `10.0.0.2` must be a suitable, reachable address on the RR. An outbound route map affects the routes advertised to that neighbor; take care not to unintentionally modify paths or policies that should remain unchanged.

An RR acting as the next hop becomes part of the **data forwarding path**, not just the BGP control plane. This can cause suboptimal routing or overload an RR if it was intended to function only as a control-plane device.

See [Route Reflectors](../scalability/route-reflectors/overview.md) for the route-reflection rules.

---

## Preserving the Next Hop Across EBGP with `next-hop-unchanged`

The `neighbor next-hop-unchanged` command is used in specialized **multihop EBGP** designs to preserve the existing NEXT_HOP attribute instead of applying normal EBGP next-hop rewriting.

For example:

```text
R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.0.3 remote-as 65003
R2(config-router)# neighbor 10.0.0.3 ebgp-multihop 5
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.0.3 activate
R2(config-router-af)# neighbor 10.0.0.3 next-hop-unchanged
```

For supported IOS XE address families and releases, use `allpaths` when next-hop preservation is required for **both IBGP- and EBGP-learned paths** advertised to that EBGP neighbor:

```text
R2(config-router-af)# neighbor 10.0.0.3 next-hop-unchanged allpaths
```

| Command | Purpose |
|---|---|
| `neighbor <peer> next-hop-self` | Set local router as advertised next hop for eligible paths |
| `neighbor <peer> next-hop-self all` | Also set local router as next hop for reflected IBGP paths, where supported |
| `neighbor <peer> next-hop-unchanged` | Preserve next hop to a multihop EBGP peer for applicable paths |
| `neighbor <peer> next-hop-unchanged allpaths` | Preserve next hop for both IBGP- and EBGP-learned paths to a multihop EBGP peer, where supported |

`next-hop-unchanged` is **not needed for normal IBGP next-hop preservation** and is **not the mechanism behind automatic third-party next hops**.

It is commonly associated with advanced MP-BGP/MPLS inter-AS designs, in which the remote peer already has a way to reach the preserved next-hop address.

---

## BGP Next-Hop Tracking

**BGP Next-Hop Address Tracking (NHT)** allows BGP to respond to changes in the reachability of its next-hop addresses without waiting for the traditional periodic BGP scanner.

On supported Cisco IOS XE platforms, NHT is enabled by default for IPv4 unicast.

BGP relies on the routing table (RIB) to resolve the NEXT_HOP attribute of received BGP routes. If the route used to reach a BGP next hop changes, BGP may need to reevaluate the paths that depend on it.

Without event-driven next-hop tracking, BGP would rely more heavily on periodic scanning to detect these changes.

### NHT Operation

Consider the following BGP route:

```text
BGP Prefix: 192.168.100.0/24
NEXT_HOP:   10.0.0.2
```

R1's routing table contains:

```text
O 10.0.0.2/32 [110/20] via 10.0.12.2, Ethernet0/0
```

BGP recursively resolves the next hop `10.0.0.2` through the OSPF route.

NHT registers the next-hop address for tracking.

If OSPF withdraws `10.0.0.2/32`, the routing table notifies BGP that its next-hop resolution has changed.

The process is:

1. BGP registers the next-hop address with the routing infrastructure.
2. The RIB detects a change affecting the next-hop route.
3. BGP receives a notification.
4. After the configured trigger delay, BGP reevaluates the affected paths.
5. If the next hop becomes unreachable, BGP no longer considers the affected path valid. If another valid path exists, BGP can select it.

NHT also detects changes to the route used for next-hop resolution, such as a different outgoing interface, forwarding next hop, or IGP metric.

**A change to the route used for next-hop resolution does not necessarily change the BGP NEXT_HOP attribute itself.** The BGP next-hop address can remain the same while the underlying forwarding path changes.

### NHT Trigger Delay

The default NHT trigger delay is **5 seconds** on the IOS XE implementations documented by Cisco.

The delay determines how long BGP waits after receiving a next-hop change notification before processing the corresponding tracking events.

Configure the delay under an address family:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp nexthop trigger delay 20
```

This configures a 20-second trigger delay.

| Trigger Delay | Effect |
|---|---|
| Shorter | Faster reaction to next-hop changes, potentially more frequent BGP recalculations |
| Longer | Allows more IGP changes to settle before BGP recalculates, potentially slowing convergence |

A longer delay can be useful when IGP instability causes frequent next-hop changes.

However, excessive delay can slow BGP convergence following a genuine failure.

NHT is enabled or disabled using:

```text
R1(config-router-af)# bgp nexthop trigger enable
R1(config-router-af)# no bgp nexthop trigger enable
```

Disabling NHT is not normally recommended.

---

## Selective Next-Hop Route Filtering

Cisco IOS XE supports **BGP Selective Next-Hop Route Filtering**, configured using:

```text
R1(config-router-af)# bgp nexthop route-map MAP-NAME
```

This command applies a route map to the **RIB route used to resolve a BGP next hop**.

It allows the router to control which routing-table entries are acceptable for BGP next-hop resolution.

For example, it can prevent BGP from resolving next hops through:

- Default routes
- Large summary routes
- Routes learned through particular routing protocols
- Routes that do not meet a specified prefix-length requirement

**The route map does not modify the BGP NEXT_HOP attribute.** It determines whether the routing-table route covering that next-hop address is acceptable.

### How the Route Map Works

Suppose R1 learns:

```text
BGP Prefix: 203.0.113.0/24
NEXT_HOP:   10.10.10.10
```

R1 has the following routing-table entry:

```text
O 10.10.10.0/24 [110/20] via 10.0.12.2, Ethernet0/0
```

Normally, BGP can resolve `10.10.10.10` through this OSPF route.

With `bgp nexthop route-map` configured:

1. BGP identifies the routing-table route covering `10.10.10.10`.
2. BGP evaluates that route against the configured route map.
3. If permitted, the route can be used for next-hop resolution.
4. If denied, BGP treats the next hop as unreachable for that path, even though the routing table contains a route to the address.

The route map is evaluated during BGP best-path processing.

A denied next-hop route does not get removed from the IP routing table. The restriction applies to BGP's use of that route for next-hop resolution.

### Filtering by Prefix Length

Suppose R1 should only resolve BGP next hops through routing-table prefixes with masks of `/24` or longer.

Configure:

```text
R1(config)# ip prefix-list NH-PREFIXES permit 0.0.0.0/0 ge 24

R1(config)# route-map NH-FILTER permit 10
R1(config-route-map)# match ip address prefix-list NH-PREFIXES

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp nexthop route-map NH-FILTER
```

The prefix list matches any IPv4 route with a prefix length from `/24` through `/32`.

The following routing-table routes are evaluated as follows:

| Resolving Route | Route-Map Result |
|---|---|
| `0.0.0.0/0` | Deny |
| `10.0.0.0/8` | Deny |
| `10.10.0.0/16` | Deny |
| `10.10.10.0/24` | Permit |
| `10.10.10.8/29` | Permit |
| `10.10.10.10/32` | Permit |

Because the route map has no additional permit entries, routes that do not match the prefix list are rejected by the implicit deny.

**The prefix list matches the routing-table prefix used to resolve the NEXT_HOP, not the BGP destination prefix being advertised.**

For example:

```text
BGP Prefix: 203.0.113.0/24
NEXT_HOP:   10.10.10.10

RIB contains:
O 10.10.0.0/16 via 10.0.12.2
```

Even though `10.10.0.0/16` provides IP reachability to the next hop, the route map rejects it because the prefix length is shorter than `/24`.

BGP therefore considers the NEXT_HOP unreachable under this policy.

### Denying Specific Routing Protocols

The next-hop route map can also filter routes according to their source routing protocol.

The supported `match source-protocol` command identifies the protocol that installed the resolving route in the RIB.

For example, prevent routes learned through BGP process 65001 from being used for next-hop resolution:

```text
R1(config)# route-map NH-FILTER deny 10
R1(config-route-map)# match source-protocol bgp 65001

R1(config)# route-map NH-FILTER permit 20

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp nexthop route-map NH-FILTER
```

The route map:

- Denies resolving routes sourced from BGP process 65001.
- Permits other resolving routes through sequence 20.

This helps prevent BGP next hops from being resolved through BGP-learned routing-table entries when the network design requires IGP or static reachability instead.

The ASN in `match source-protocol bgp 65001` identifies the BGP process associated with the resolving route.

### Combining Prefix-Length and Protocol Filtering

A route map can enforce multiple next-hop resolution restrictions.

For example, deny:

1. Routing-table prefixes of `/25` or shorter.
2. Routes sourced from BGP process 65001.

Permit everything else.

```text
R1(config)# ip prefix-list SHORT-PREFIXES permit 0.0.0.0/0 le 25

R1(config)# route-map NH-FILTER deny 10
R1(config-route-map)# match ip address prefix-list SHORT-PREFIXES

R1(config)# route-map NH-FILTER deny 20
R1(config-route-map)# match source-protocol bgp 65001

R1(config)# route-map NH-FILTER permit 30

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp nexthop route-map NH-FILTER
```

The route map evaluates entries in ascending sequence order.

| Resolving Route | Source | Result |
|---|---|---|
| `0.0.0.0/0` | Static | Deny |
| `10.10.10.0/24` | OSPF | Deny |
| `10.10.10.8/29` | OSPF | Permit |
| `10.10.10.10/32` | BGP 65001 | Deny |
| `10.10.10.10/32` | Static | Permit |

Only routes with a prefix length of `/26` through `/32` that are not sourced from the specified BGP process are permitted.

This is useful for avoiding recursive resolution through broad summary routes or undesired BGP-learned routes.

### Supported Route-Map Commands

Cisco documents two supported match commands for BGP selective next-hop route filtering:

| Command | Purpose |
|---|---|
| `match ip address` | Match the resolving route using an ACL or prefix list |
| `match source-protocol` | Match the routing protocol that supplied the resolving route |

Other match commands and `set` commands are not supported for this feature.

In particular, `bgp nexthop route-map` is different from applying an outbound neighbor route map containing `set ip next-hop`.

| Feature | Purpose |
|---|---|
| `bgp nexthop route-map MAP` | Controls which RIB routes can resolve BGP next hops |
| `neighbor X route-map MAP out` with `set ip next-hop` | Changes the NEXT_HOP attribute advertised to a BGP neighbor |
| `neighbor X next-hop-self` | Advertises the local router as the BGP next hop for applicable routes |

Remove selective next-hop filtering using:

```text
R1(config-router-af)# no bgp nexthop route-map NH-FILTER
```

---

## NHT Verification and Troubleshooting

Verify BGP next-hop addresses:

```text
R1# show bgp ipv4 unicast
```

Inspect the attributes and validity of an individual BGP path:

```text
R1# show bgp ipv4 unicast 203.0.113.0/24
```

Verify the routing-table entry used for next-hop resolution:

```text
R1# show ip route 10.10.10.10
```

Inspect tracked BGP next hops on IOS XE platforms supporting this command:

```text
R1# show ip bgp nexthops
```

Verify the configured route map and its match counters:

```text
R1# show route-map NH-FILTER
R1# show ip prefix-list
```

Check the BGP configuration:

```text
R1# show running-config | section router bgp
```

When troubleshooting an unreachable next hop:

1. Identify the NEXT_HOP attribute in the BGP table.
2. Verify that the routing table contains a route covering the next-hop address.
3. Determine the prefix length and source protocol of that resolving route.
4. Check whether `bgp nexthop route-map` is configured.
5. Verify that the route is permitted by the next-hop route map.

A BGP path can fail next-hop validation even when the next-hop address appears reachable in the IP routing table, because a configured selective next-hop policy rejects the resolving route.

**This is different from a BGP RIB failure.** A route with an unreachable next hop is not eligible for normal best-path selection, whereas a RIB failure occurs when a selected BGP route cannot be installed in the IP routing table.

Also distinguish route reachability tracking from BGP neighbor tracking. The separate `neighbor ... fall-over [route-map ...]` feature monitors reachability to a BGP peer for faster session deactivation. It does not perform the same function as `bgp nexthop route-map`.

---

## Verification and Troubleshooting

### Essential Commands

| Command | Purpose |
|---|---|
| `show bgp ipv4 unicast <prefix>` | Inspect the selected path, NEXT_HOP, and path validity |
| `show bgp ipv4 unicast` | See next-hop values across the BGP table |
| `show ip bgp nexthops` | Inspect tracked BGP next hops and recursive resolution, where supported |
| `show ip route <next-hop>` | Verify the route used to resolve a BGP next hop |
| `show ip cef <prefix>` | Check the forwarding resolution for a destination |
| `show ip cef <next-hop> detail` | Inspect CEF resolution for the next-hop address |
| `show bgp ipv4 unicast neighbors <peer> advertised-routes` | Inspect prefixes being advertised to a peer |
| `show bgp ipv4 unicast neighbors <peer> routes` | Inspect accepted routes from a peer |
| `show bgp ipv4 unicast summary` | Verify that the BGP sessions are established |
| `show ip route bgp` | Check which BGP routes reached the IP routing table |

For a route missing from the routing table, a useful sequence is:

```text
R3# show bgp ipv4 unicast summary
R3# show bgp ipv4 unicast 192.0.2.0/24
R3# show ip route 10.0.12.1
R3# show ip cef 10.0.12.1 detail
R3# show ip route 192.0.2.0
```

The first checks whether the session is established. The second identifies the route's **actual BGP NEXT_HOP**. The next two verify how that address resolves. The last checks the route ultimately selected for forwarding.

### Common Problems

| Symptom | Likely cause | What to check |
|---|---|---|
| IBGP session is Established, but route has an inaccessible next hop | EBGP next hop was preserved and not reachable internally | Route to the NEXT_HOP, IGP, or `next-hop-self` |
| Route appears in BGP but is not best | NEXT_HOP unreachable or another BGP path is preferred | Per-prefix BGP details and next-hop reachability |
| Best BGP path is not installed in the IP RIB | Competing route or other installation restriction | `show ip route <prefix>` and any RIB-failure indication |
| EBGP peer receives an unexpected third-party next hop | Original next hop shares the receiving peer's subnet | Shared subnet and advertised NEXT_HOP |
| Route reflector clients retain the original next hop | Reflected routes are not rewritten by ordinary `next-hop-self` | RR behavior, `next-hop-self all`, or outbound route map |
| Multihop EBGP peer receives a next hop it cannot reach | `next-hop-unchanged` preserves an unreachable address | Routing to the preserved next hop |
| Traffic forwards through an unexpected router | BGP NEXT_HOP and recursively resolved immediate next hop differ | `show ip route <next-hop>` and CEF |

> The address configured in `neighbor <peer> remote-as <asn>` identifies the **BGP session endpoint**. The NEXT_HOP attribute identifies the address used to resolve a **BGP route**. They are related but not interchangeable.

---

## Key Points

- EBGP normally changes NEXT_HOP; IBGP normally preserves the next hop of EBGP-learned routes.
- EBGP can preserve a **third-party next hop** automatically when the receiving neighbor shares the next-hop subnet.
- `neighbor next-hop-self` fixes many IBGP next-hop reachability problems by making the advertising router the next hop.
- `next-hop-self` and `next-hop-self all` can behave differently on route reflectors.
- `next-hop-unchanged` is for specific multihop EBGP designs, not for ordinary IBGP propagation.
- A BGP path needs a reachable NEXT_HOP. The router can recursively resolve that address through an IGP or static route.
- An **unreachable BGP next hop** and a **BGP RIB failure** are different problems.
- Always verify both the advertised NEXT_HOP and its resolution in the IP routing and forwarding tables.

---

## Cisco Documentation

- [IP Routing Configuration Guide: BGP Support for Next-Hop Address Tracking](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-nexthop-track.html)
- [Cisco IOS IP Routing: BGP Command Reference, `neighbor next-hop-self` and `neighbor next-hop-unchanged`](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [IP Routing Configuration Guide, Cisco IOS XE 17.x: BGP Next Hop Unchanged](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-next-hop-0.html)
- [IP Routing Configuration Guide: Configuring Internal BGP Features](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-int-features-0.html)
