# NEXT_HOP

The **NEXT_HOP** BGP path attribute identifies the IP address used to reach an advertised destination prefix.

BGP uses NEXT_HOP for two important purposes:

1. **Path eligibility:** A BGP path must have a resolvable next hop before it can be selected as best.
2. **Best path selection:** When earlier criteria are equal, BGP can prefer the path with the lowest IGP metric to its next hop.

Unlike Weight, LOCAL_PREF, AS_PATH length, ORIGIN, and MED, NEXT_HOP does not have a simple higher-or-lower preference.

Instead, BGP first verifies next-hop reachability, then considers the cost of reaching the next hop at a later stage of best-path selection.

| Property | Value |
|---|---|
| Attribute type code | 3 |
| Category | Well-known mandatory |
| Attribute flags | `0x40` (without extended length) |
| Size | 4 bytes for traditional IPv4 unicast |
| Primary purpose | Identifies the next-hop IP address |
| Path eligibility | Next hop must be resolvable |
| Best-path preference | Lowest IGP metric to next hop, when reached |
| EBGP behavior | Normally changes NEXT_HOP |
| IBGP behavior | Normally preserves NEXT_HOP for EBGP-learned routes |

**Important:** In MP-BGP, next-hop information is carried in MP_REACH_NLRI rather than necessarily using the traditional NEXT_HOP attribute.

---

## BGP NEXT_HOP vs. Forwarding Next Hop

The **BGP NEXT_HOP attribute** and the **immediate forwarding next hop** are separate concepts.

For example, a router might learn:

```text
BGP Prefix: 192.0.2.0/24
NEXT_HOP:   10.10.10.10
```

However, `10.10.10.10` might not be directly connected.

The router must perform a recursive routing-table lookup to determine how to reach it.

Suppose R1 has:

```text
R1# show ip route 10.10.10.10

O 10.10.10.10/32 [110/30] via 10.0.12.2, Ethernet0/0
```

R1 resolves the BGP next hop as follows:

```text
Destination: 192.0.2.10
            |
            v
BGP Prefix: 192.0.2.0/24
BGP NEXT_HOP: 10.10.10.10
            |
            v
Recursive RIB Lookup
10.10.10.10/32
            |
            v
OSPF Route
Metric: 30
Via 10.0.12.2
            |
            v
Immediate Forwarding Next Hop:
10.0.12.2, Ethernet0/0
```

The distinction is:

| Concept | Address |
|---|---|
| Destination prefix | `192.0.2.0/24` |
| BGP NEXT_HOP | `10.10.10.10` |
| Immediate forwarding next hop | `10.0.12.2` |
| Outgoing interface | Ethernet0/0 |

**The BGP NEXT_HOP attribute does not change simply because recursive resolution selects a different forwarding next hop.**

---

## Default NEXT_HOP Advertisement Rules

The default behavior depends on the type of BGP relationship.

| Route Advertisement | Default Behavior |
|---|---|
| EBGP to EBGP | Normally changes NEXT_HOP to the advertising router |
| EBGP to IBGP | Normally preserves NEXT_HOP |
| IBGP to IBGP, when permitted | Normally preserves NEXT_HOP |
| Locally originated route to EBGP | Uses an address of the advertising router |
| Locally originated route to IBGP | Uses an address of the advertising router |

Exceptions include:

- **Third-party next hops:** An EBGP router can preserve a next hop on a subnet shared with the receiving peer.
- **`next-hop-self`:** Sets the advertising router as the next hop for applicable routes.
- **`next-hop-unchanged`:** Preserves next hops in supported EBGP scenarios.
- **Outbound route maps:** Can explicitly modify NEXT_HOP.

These mechanisms are covered in [BGP Next-Hop Handling](../../peering/next-hop.md).

### Locally Originated Routes

A locally originated route commonly appears with next hop `0.0.0.0` in the local BGP table.

```text
R1# show bgp ipv4 unicast

   Network          Next Hop            Metric LocPrf Weight Path
*> 192.0.2.0/24     0.0.0.0                  0         32768 i
```

Here, `0.0.0.0` indicates local origination.

It does not mean R1 advertises `0.0.0.0` as the NEXT_HOP to its neighbors.

---

# NEXT_HOP and Path Eligibility

A BGP path with an **unreachable NEXT_HOP** is not eligible for normal best-path selection.

This check occurs before comparing attributes such as Weight, LOCAL_PREF, and AS_PATH length.

### Example

Consider the following topology:

```text
            AS 65001                 AS 65002

               R1 --------------------- R2 -------- R3
                       EBGP                    IBGP

          192.0.2.0/24
```

R1 advertises `192.0.2.0/24` to R2.

R2 receives:

```text
Prefix:   192.0.2.0/24
NEXT_HOP: 10.0.12.1
AS_PATH:  65001
```

R2 then advertises the route to R3 through IBGP, preserving NEXT_HOP `10.0.12.1`.

Suppose R3 has no route to `10.0.12.1`.

R3 can still have an Established IBGP session with R2 and receive the prefix.

However, the path is **not eligible for best-path selection** because its next hop cannot be resolved.

### Resolving the Problem

Two common solutions are:

1. Configure R2 with `neighbor next-hop-self` toward R3.
2. Provide R3 with an IGP or static route to `10.0.12.1`.

For example, on R2:

```text
R2(config)# router bgp 65002
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.23.3 next-hop-self
```

Alternatively, on R3:

```text
R3(config)# ip route 10.0.12.0 255.255.255.252 10.0.23.2
```

Either method can make the BGP path eligible, assuming no other restrictions.

### Unreachable NEXT_HOP vs. RIB Failure

These are different conditions.

| Condition | Meaning |
|---|---|
| Unreachable NEXT_HOP | BGP cannot resolve the path's next hop, so the path is ineligible for normal best-path selection |
| RIB failure | BGP selected a best path but could not install it into the IP routing table |

For example, an IBGP path can have a reachable next hop and be selected as the BGP best path, but lose RIB installation to an OSPF route with a lower administrative distance.

See [BGP Path Selection and RIB Installation](../best-path/rib-installation.md).

---

# IGP Metric to NEXT_HOP

When multiple BGP paths remain tied after earlier best-path criteria, Cisco BGP prefers the path with the **lowest IGP metric to its NEXT_HOP**.

This is sometimes called *hot-potato routing* or *closest-exit routing*.

The comparison occurs after the usual preference for EBGP over IBGP.

### Example Topology

```text
                      AS 65000

           R2                         R3
           |                          |
           | OSPF Cost 10             | OSPF Cost 30
           |                          |
           +------------ R1 ----------+
                        |
                        |
                Internal Network

         R2 and R3 learn 192.0.2.0/24
         from external AS 65100
```

R1 receives two IBGP paths to `192.0.2.0/24`.

Assume R2 and R3 advertise themselves as the next hops.

| Attribute | Via R2 | Via R3 |
|---|---|---|
| Weight | 0 | 0 |
| LOCAL_PREF | 100 | 100 |
| AS_PATH | `65100` | `65100` |
| ORIGIN | IGP | IGP |
| MED | 0 | 0 |
| Peer type | IBGP | IBGP |
| IGP metric to NEXT_HOP | 10 | 30 |

BGP selects **the path through R2**, because its IGP metric to the next hop is lower.

### Important Details

- The comparison uses the routing information for reaching the BGP NEXT_HOP, not the original destination prefix's metric.
- It is evaluated only after earlier best-path criteria are equal.
- It does not change the NEXT_HOP attribute.
- If both paths use the same BGP next hop, their next-hop metric is normally the same.
- Changes in the underlying IGP metric can change the BGP best path even when BGP path attributes remain unchanged.

### Example: LOCAL_PREF Overrides IGP Cost

Consider:

| Attribute | Path A | Path B |
|---|---|---|
| LOCAL_PREF | 200 | 100 |
| IGP metric to NEXT_HOP | 100 | 10 |

**Path A is preferred**, despite having a higher IGP metric to its next hop.

This is because LOCAL_PREF is compared earlier.

---

## Recursive Next-Hop Resolution

BGP performs a recursive lookup to determine how to forward traffic toward its NEXT_HOP.

### Example

Suppose R1 learns:

```text
BGP Prefix: 203.0.113.0/24
NEXT_HOP:   10.10.10.10
```

Its routing table contains:

```text
O 10.10.10.10/32 [110/30] via 10.0.12.2, Ethernet0/0
```

The recursive lookup is:

```text
203.0.113.0/24
       |
       v
BGP NEXT_HOP: 10.10.10.10
       |
       v
OSPF Route: 10.10.10.10/32
       |
       v
Forward via 10.0.12.2
       |
       v
Ethernet0/0
```

The router can now use the BGP route, assuming it is selected and installed.

### Recursive Resolution Changes

Suppose an OSPF topology change causes the route to `10.10.10.10` to change:

Before:

```text
O 10.10.10.10/32 [110/30] via 10.0.12.2, Ethernet0/0
```

After:

```text
O 10.10.10.10/32 [110/50] via 10.0.13.3, Ethernet0/1
```

The BGP NEXT_HOP remains `10.10.10.10`.

However:

- The immediate forwarding next hop changes.
- The outgoing interface changes.
- The IGP metric to the BGP next hop changes.

This can trigger BGP best-path reevaluation.

**An IGP topology change can affect BGP path selection without a new BGP UPDATE message.**

### Recursive Resolution Failures

Recursive resolution must terminate in a usable forwarding path.

Potential problems include:

- No route to the BGP next hop.
- Recursive routing loops.
- A resolving route rejected by BGP next-hop policy.
- A route that does not provide usable forwarding resolution.

A route's mere presence in the RIB does not always guarantee that BGP considers its next hop valid.

---

# BGP Next-Hop Address Tracking

**BGP Next-Hop Address Tracking (NHT)** allows BGP to respond to changes affecting next-hop reachability without waiting for the traditional periodic BGP scanner.

NHT is enabled by default on supported Cisco IOS XE platforms.

BGP registers its next-hop addresses with the routing infrastructure.

When a relevant routing-table change occurs, BGP receives a notification and can reevaluate affected paths.

### Events That Can Trigger NHT

Examples include:

- A BGP next hop becomes reachable.
- A BGP next hop becomes unreachable.
- The route used to resolve a BGP next hop changes.
- The IGP metric to a BGP next hop changes.
- The immediate forwarding next hop or outgoing interface changes.

### Example

R1 has:

```text
BGP Prefix: 203.0.113.0/24
NEXT_HOP:   10.10.10.10
```

Initially:

```text
O 10.10.10.10/32 [110/20] via 10.0.12.2
```

Then OSPF withdraws the route.

NHT notifies BGP of the next-hop change.

If R1 no longer has a usable route to `10.10.10.10`, BGP considers paths using that next hop unreachable.

If another eligible BGP path exists, BGP can select it.

### NHT Trigger Delay

The NHT trigger delay controls how long BGP waits before processing next-hop change notifications.

On the IOS XE implementations documented by Cisco, the default delay is 5 seconds.

Configure it under the address family:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp nexthop trigger delay 20
```

This configures a 20-second delay.

A shorter delay generally allows faster reaction to changes, while a longer delay can reduce repeated BGP recalculations during IGP instability.

Enable NHT explicitly:

```text
R1(config-router-af)# bgp nexthop trigger enable
```

Disable NHT:

```text
R1(config-router-af)# no bgp nexthop trigger enable
```

Disabling NHT is not normally recommended.

For full details, see [BGP Next-Hop Handling](../../peering/next-hop.md#bgp-next-hop-tracking).

---

## Selective Next-Hop Route Filtering

Cisco IOS XE supports selective filtering of the routes that BGP may use to resolve next-hop addresses.

The command is:

```text
R1(config-router-af)# bgp nexthop route-map <route-map>
```

This applies a route map to the **RIB route used for next-hop resolution**, not to the BGP destination prefix itself.

### Example: Reject Default-Route Resolution

Suppose R1 receives:

```text
BGP Prefix: 203.0.113.0/24
NEXT_HOP:   10.10.10.10
```

R1 has only a default route covering the next-hop address:

```text
S* 0.0.0.0/0 via 10.0.12.2
```

To prevent BGP from using a default route for next-hop resolution:

```text
R1(config)# ip prefix-list DEFAULT-ROUTE permit 0.0.0.0/0

R1(config)# route-map NH-FILTER deny 10
R1(config-route-map)# match ip address prefix-list DEFAULT-ROUTE

R1(config)# route-map NH-FILTER permit 20

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp nexthop route-map NH-FILTER
```

The route map:

1. Denies the default route as a resolving route.
2. Permits other routes used for next-hop resolution.

If the default route is the only installed route covering `10.10.10.10`, BGP treats that next hop as unreachable under this policy.

**The route map does not cause BGP to select an alternative, less-preferred RIB route that was not installed.** It filters the route identified by normal RIB resolution.

Only `match ip address` and `match source-protocol` are documented as supported match commands for this feature. `set` commands are not supported.

See [Selective Next-Hop Route Filtering](../../peering/next-hop.md#selective-next-hop-route-filtering) for detailed examples.

---

# Configuring the NEXT_HOP Attribute

NEXT_HOP can be modified using neighbor-level configuration or route maps.

## Using `next-hop-self`

To advertise the local router as the next hop:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 next-hop-self
```

This is commonly used when advertising EBGP-learned routes to IBGP peers.

### Route Reflectors

When a route reflector advertises reflected IBGP routes, ordinary `next-hop-self` does not normally rewrite their next hops.

Where supported:

```text
R1(config-router-af)# neighbor 10.0.12.2 next-hop-self all
```

The `all` keyword includes reflected routes.

See [BGP Next-Hop Handling](../../peering/next-hop.md#next-hop-self-on-route-reflectors).

---

## Using `set ip next-hop`

A BGP route map can explicitly modify NEXT_HOP.

For example:

```text
R1(config)# route-map SET-NEXT-HOP permit 10
R1(config-route-map)# set ip next-hop 10.10.10.10

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-NEXT-HOP out
```

This sets NEXT_HOP to `10.10.10.10` for matching outbound BGP advertisements, subject to platform and address-family support.

The receiving router must be able to resolve the configured next hop.

### Using `peer-address`

Cisco IOS XE also supports:

```text
R1(config-route-map)# set ip next-hop peer-address
```

Its behavior depends on route-map direction.

| Direction | Effect |
|---|---|
| Inbound | Sets NEXT_HOP to the address of the BGP neighbor |
| Outbound | Sets NEXT_HOP to the local router's peering address |

An inbound use can override third-party next-hop behavior.

An outbound use provides finer-grained control than neighbor-level `next-hop-self` because the route map can match specific prefixes.

**Do not confuse `set ip next-hop` with `bgp nexthop route-map`.** The former changes the BGP path attribute, while the latter controls which RIB routes are acceptable for resolving next hops.

---

# NEXT_HOP in Multiprotocol BGP

Traditional IPv4 unicast BGP uses the NEXT_HOP attribute (type code 3).

Multiprotocol BGP uses **MP_REACH_NLRI** (type code 14) to carry reachability and next-hop information for additional address families.

MP_REACH_NLRI includes:

- AFI
- SAFI
- Next-hop address length
- Next-hop address
- Reachable NLRI

### IPv6 Next Hops

For IPv6 unicast, MP_REACH_NLRI can carry:

| Next-Hop Length | Information |
|---|---|
| 16 bytes | One IPv6 next-hop address |
| 32 bytes | IPv6 global address and link-local address |

The link-local address is relevant to directly connected IPv6 next-hop resolution.

For example, a BGP IPv6 route might use:

```text
BGP Prefix: 2001:DB8:100::/48
NEXT_HOP:   2001:DB8:12::2
```

The receiving router must have usable IPv6 reachability to `2001:DB8:12::2`.

### IPv4 NLRI with an IPv6 Next Hop

RFC 8950 allows IPv4 NLRI to be advertised using an IPv6 next-hop address through MP-BGP.

For example:

```text
BGP Prefix: 192.0.2.0/24
NEXT_HOP:   2001:DB8:12::2
```

This is a supported protocol extension, subject to platform capabilities and configuration.

It demonstrates that the address family of the advertised destination and the address family used for the next hop are not necessarily identical.

### VRF Next-Hop Resolution

For routes in a VRF, BGP normally resolves the next hop in the appropriate VRF routing context.

A route existing in the global routing table does not automatically provide next-hop reachability for a BGP path in a VRF.

For example:

```text
R1# show ip route vrf CUSTOMER 10.10.10.10
```

This is particularly important when troubleshooting MP-BGP, MPLS VPN, and VRF-aware routing.

---

## Verification

### Display NEXT_HOP

```text
R1# show bgp ipv4 unicast
R1# show bgp ipv4 unicast 192.0.2.0/24
```

Example:

```text
   Network          Next Hop            Metric LocPrf Weight Path
*  192.0.2.0/24     10.0.12.2                0    100      0 65002 i
*>                  10.0.13.3                0    100      0 65003 i
```

The `Next Hop` column displays the BGP next-hop address for each path.

The `>` symbol identifies the best path.

### Verify Recursive Resolution

```text
R1# show ip route 10.0.12.2
R1# show ip cef 10.0.12.2 detail
```

Use these commands to determine how the router reaches the BGP next hop.

### Verify BGP Next-Hop Tracking

On supported IOS XE platforms:

```text
R1# show ip bgp nexthops
```

This displays information about tracked BGP next hops and their resolution.

### Verify IPv6 Next-Hop Resolution

```text
R1# show bgp ipv6 unicast 2001:DB8:100::/48
R1# show ipv6 route 2001:DB8:12::2
R1# show ipv6 cef 2001:DB8:12::2 detail
```

### Troubleshooting Sequence

When a BGP route is present but not usable:

1. Verify the BGP neighbor is Established.
2. Display the BGP route and identify its NEXT_HOP.
3. Check the routing table for reachability to NEXT_HOP.
4. Examine recursive forwarding resolution using CEF.
5. Check for `bgp nexthop route-map` filtering.
6. Verify that the BGP path is valid and eligible.
7. If BGP selected a best path, verify whether it was installed in the RIB.

---

## Key Takeaways

- NEXT_HOP is a **well-known mandatory** attribute for traditional IPv4 unicast reachability.
- It identifies the next-hop address used to reach an advertised prefix.
- **BGP NEXT_HOP and the immediate forwarding next hop are not necessarily the same.**
- BGP performs recursive routing-table lookups to resolve NEXT_HOP.
- An unreachable next hop makes the associated BGP path ineligible for normal best-path selection.
- When earlier criteria are equal, BGP prefers the path with the **lowest IGP metric to NEXT_HOP**.
- IGP changes can influence BGP best-path selection without changing the NEXT_HOP attribute or receiving new BGP UPDATEs.
- BGP Next-Hop Address Tracking (NHT) provides event-driven detection of next-hop changes.
- `bgp nexthop route-map` controls which RIB routes can be used for next-hop resolution.
- `set ip next-hop` modifies the advertised or received BGP NEXT_HOP attribute.
- `neighbor next-hop-self` is commonly used to make EBGP-learned routes usable by IBGP peers.
- MP-BGP carries next-hop information in MP_REACH_NLRI.
- NEXT_HOP reachability and RIB installation are separate stages of BGP route processing.

---

## References

- [Cisco: Select BGP Best-path Algorithm](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html)
- [Cisco: BGP Support for Next-Hop Address Tracking](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-nexthop-track.html)
- [Cisco: BGP Configuration Guide, Understanding BGP](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr3-fwd/bgp/bgp-configuration-guide/routing-bgp.html)
- [Cisco: Connecting to a Service Provider Using External BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-external-sp-0.html)
- [RFC 4271: A Border Gateway Protocol 4 (BGP-4)](https://www.rfc-editor.org/rfc/rfc4271.html)
- [RFC 4760: Multiprotocol Extensions for BGP-4](https://www.rfc-editor.org/rfc/rfc4760.html)
- [RFC 2545: Use of BGP-4 Multiprotocol Extensions for IPv6 Inter-Domain Routing](https://www.rfc-editor.org/rfc/rfc2545.html)
- [RFC 8950: Advertising IPv4 NLRI with an IPv6 Next Hop](https://www.rfc-editor.org/rfc/rfc8950.html)
