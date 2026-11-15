# BGP Path Selection and RIB Installation

**BGP best-path selection** and **Routing Information Base (RIB) installation** are separate processes.

BGP first selects its preferred path using the BGP best-path algorithm. It then attempts to install that path into the appropriate IP routing table.

However, **the BGP best path is not necessarily installed in the RIB**.

For example, another routing protocol might already provide a route to the same prefix with a lower administrative distance.

---

## BGP Best Path vs. RIB Installation

The process consists of three stages:

```text
        Multiple BGP Paths
                 |
                 v
       +--------------------+
       | BGP Best Path      |
       | Selection          |
       +--------------------+
                 |
                 v
       +--------------------+
       | Selected BGP       |
       | Best Path          |
       +--------------------+
                 |
                 v
       +--------------------+
       | RIB Installation   |
       |                    |
       | Administrative     |
       | Distance           |
       +--------------------+
             /       \
            /         \
           v           v
       Installed    RIB Failure
```

| Stage | Purpose |
|---|---|
| BGP best-path selection | Select the preferred BGP path |
| RIB installation | Select between routes offered by different routing sources |
| FIB programming | Program forwarding information based on the selected routes |

**Administrative distance is not evaluated during BGP best-path selection.**

It becomes relevant when the selected BGP path competes with routes from other sources for RIB installation.

---

## BGP Administrative Distance

Cisco IOS XE uses the following default administrative distances:

| Routing Source | Administrative Distance |
|---|---|
| Connected | 0 |
| Static | 1 |
| EBGP | 20 |
| EIGRP internal | 90 |
| OSPF | 110 |
| EIGRP external | 170 |
| IBGP | 200 |
| Locally originated BGP | 200 |

A lower administrative distance is preferred when routes to the **same prefix and prefix length** compete for installation.

### Example: IBGP vs. OSPF

R1 learns `192.0.2.0/24` through both IBGP and OSPF.

| Route | Next Hop | AD |
|---|---|---|
| IBGP | 10.0.12.2 | 200 |
| OSPF | 10.0.13.3 | 110 |

Assume the IBGP path is selected as the BGP best path.

However, OSPF has a lower administrative distance.

Therefore:

- The IBGP path remains the BGP best path.
- The OSPF route is installed in the IP routing table.
- The BGP path experiences a **RIB failure**.

### Longest Prefix Match vs. Administrative Distance

Administrative distance is not compared between routes to different prefix lengths.

For example:

```text
OSPF: 192.0.2.0/24 [110/20]
BGP:  192.0.2.0/25 [200/0]
```

Both routes can be installed.

Traffic to `192.0.2.10` matches both routes, but the `/25` is used because it is more specific.

**Longest prefix match takes precedence during forwarding.** Administrative distance determines route preference when competing routes have the same prefix and prefix length.

---

## Changing BGP Administrative Distance

Cisco IOS XE supports the following command:

```text
R1(config-router)# distance bgp <external> <internal> <local>
```

The default values are:

```text
R1(config)# router bgp 65001
R1(config-router)# distance bgp 20 200 200
```

The arguments are:

| Argument | Default | Applies To |
|---|---|---|
| External | 20 | EBGP-learned routes |
| Internal | 200 | IBGP-learned routes |
| Local | 200 | Locally originated BGP routes |

### Example: Prefer IBGP over OSPF

Configure the following:

```text
R1(config)# router bgp 65001
R1(config-router)# distance bgp 20 90 200
```

This changes the IBGP administrative distance from 200 to 90.

Now:

| Route | AD |
|---|---|
| IBGP | 90 |
| OSPF | 110 |

Assuming both routes are valid and there are no other restrictions, the IBGP route is preferred for RIB installation.

**Important:** This changes administrative distance for IBGP routes generally, not just one prefix.

Changing administrative distance can affect routing behavior throughout the network and should be done carefully.

Restore the default values:

```text
R1(config-router)# no distance bgp
```

---

## BGP RIB Failure

A **BGP RIB failure** occurs when BGP selects a best path but cannot install that path into the routing table.

The most common reason is that another routing source provides the same prefix with a lower administrative distance.

Other possible causes include:

- Resource or memory limitations.
- Route limits.
- Other RIB installation restrictions.

A RIB failure does not necessarily indicate that the BGP route is invalid.

### Example

R1 receives an IBGP route:

```text
Prefix:   192.0.2.0/24
NEXT_HOP: 10.0.12.2
AD:       200
```

However, R1 also learns the same prefix through OSPF:

```text
Prefix:   192.0.2.0/24
NEXT_HOP: 10.0.13.3
AD:       110
```

The OSPF route wins RIB installation.

Verify the BGP table:

```text
R1# show bgp ipv4 unicast

   Network          Next Hop            Metric LocPrf Weight Path
r>i 192.0.2.0/24    10.0.12.2                0    100      0 i
```

The relevant status codes are:

| Code | Meaning |
|---|---|
| `r` | RIB failure |
| `>` | BGP best path |
| `i` | IBGP-learned path |

**The `r` and `>` codes can appear together.**

This means BGP selected the path as best, but the path was not installed in the RIB.

Verify the installed route:

```text
R1# show ip route 192.0.2.0

Routing entry for 192.0.2.0/24
  Known via "ospf 1", distance 110, metric 20
  Routing Descriptor Blocks:
  * 10.0.13.3, via Ethernet0/1
```

Although the BGP best path uses `10.0.12.2`, traffic is forwarded using the installed OSPF route through `10.0.13.3`.

### Display RIB Failures

Use:

```text
R1# show ip bgp rib-failure
```

Example:

```text
Network         Next Hop       RIB-failure           RIB-NH Matches
192.0.2.0/24    10.0.12.2      Higher admin distance  n/a
```

The output identifies the reason BGP could not install the route.

The `RIB-NH Matches` field is discussed later.

---

## RIB Failure Does Not Cause BGP to Select Its Second-Best Path

An important distinction is that BGP does not normally reconsider its own best-path decision simply because the selected path loses RIB installation.

For example, R1 receives two BGP paths:

| Attribute | Path A | Path B |
|---|---|---|
| Peer type | IBGP | EBGP |
| LOCAL_PREF | 200 | 100 |
| BGP AD | 200 | 20 |

BGP selects **Path A** because it has higher LOCAL_PREF.

Suppose OSPF also provides the same prefix with AD 110.

The result is:

1. BGP selects Path A as its best path.
2. Path A loses RIB installation to OSPF.
3. Path A is marked with a RIB failure.
4. BGP does not automatically install Path B instead merely because its EBGP administrative distance is 20.

**BGP best-path selection occurs before administrative distance comparison.**

The lower administrative distance of Path B does not override the BGP best-path selection algorithm.

This describes normal single-best-path operation. Features such as BGP multipath can provide additional installation candidates.

---

## Advertising BGP Routes with RIB Failures

By default, Cisco IOS XE can **advertise a BGP best path even when it has a RIB failure**.

For example:

```text
R1 BGP Table:
192.0.2.0/24 via 10.0.12.2
Status: Best, RIB Failure

R1 IP Routing Table:
192.0.2.0/24 via 10.0.13.3
Source: OSPF
```

Although the BGP path is not installed in the RIB, R1 can still advertise the BGP route to eligible neighbors.

This behavior allows BGP route propagation to operate independently of local RIB installation.

However, it can also cause a router to advertise a BGP path that differs from the route it actually uses for forwarding.

### Verify Advertised Routes

```text
R1# show bgp ipv4 unicast neighbors 10.0.14.4 advertised-routes
```

If permitted by normal BGP advertisement rules and outbound policies, the RIB-failed best path can appear in the advertised-routes output.

**A RIB failure does not automatically prevent BGP advertisement.**

---

## Suppressing Inactive BGP Routes

Cisco IOS XE supports the following command:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp suppress-inactive
```

This modifies the default advertisement behavior for inactive BGP routes.

However, **the command does not suppress every route experiencing a RIB failure**.

For the common administrative-distance RIB-failure scenario, suppression depends on whether the forwarding next hop of the installed route matches the BGP path's resolved forwarding next hop.

### RIB Failure with Different Next Hops

Consider:

```text
BGP Best Path:
192.0.2.0/24 via 10.0.12.2

Installed Static Route:
192.0.2.0/24 via 10.0.13.3
```

The BGP route loses RIB installation because the static route has a lower administrative distance.

The forwarding next hops are different:

| Route | Forwarding Next Hop |
|---|---|
| BGP | 10.0.12.2 |
| Static | 10.0.13.3 |

Configure:

```text
R1(config-router-af)# bgp suppress-inactive
```

**R1 suppresses advertisement of the RIB-failed BGP route**, assuming normal next-hop resolution produces different forwarding adjacencies.

Verify:

```text
R1# show ip bgp rib-failure
```

Example:

```text
Network         Next Hop       RIB-failure           RIB-NH Matches
192.0.2.0/24    10.0.12.2      Higher admin distance  No
```

`RIB-NH Matches: No` indicates that the installed route and BGP path resolve to different forwarding next hops.

### RIB Failure with Matching Next Hops

Now suppose the installed static route uses the same next hop as BGP:

```text
BGP Best Path:
192.0.2.0/24 via 10.0.12.2

Installed Static Route:
192.0.2.0/24 via 10.0.12.2
```

The BGP route still experiences a RIB failure because the static route has a lower AD.

However, the forwarding next hops match.

Verify:

```text
R1# show ip bgp rib-failure
```

Example:

```text
Network         Next Hop       RIB-failure           RIB-NH Matches
192.0.2.0/24    10.0.12.2      Higher admin distance  Yes
```

**Even with `bgp suppress-inactive` configured, R1 continues advertising this BGP route.**

The reason is that the installed route and the BGP path use the same forwarding next hop.

---

## Understanding `RIB-NH Matches`

The `show ip bgp rib-failure` command includes the `RIB-NH Matches` field.

| Value | Meaning | Suppressed with `bgp suppress-inactive`? |
|---|---|---|
| `Yes` | Installed route and BGP path resolve to the same forwarding next hop | No |
| `No` | Installed route and BGP path resolve to different forwarding next hops | Yes |
| `n/a` | Next-hop match comparison is not applicable, typically because `bgp suppress-inactive` is not configured | No, by default |

These results describe the usual higher-administrative-distance RIB-failure case.

**Important:** The comparison concerns the resolved forwarding next hop, not necessarily the literal BGP NEXT_HOP attribute.

For example, the BGP next hop and installed route's configured next hop could be different IP addresses but recursively resolve to the same forwarding adjacency.

In that case, `RIB-NH Matches` can still display `Yes`.

### Example: Recursive Next-Hop Resolution

```text
BGP NEXT_HOP: 10.10.10.10
    |
    v
Recursive lookup
    |
    v
Forward via 10.0.12.2

Installed route:
192.0.2.0/24 via 10.0.12.2
```

Both routes ultimately forward through `10.0.12.2`.

Therefore, the forwarding next hops can match even though the BGP NEXT_HOP attribute is `10.10.10.10`.

---

## Configuring and Removing `bgp suppress-inactive`

Enable suppression:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp suppress-inactive
```

Disable suppression:

```text
R1(config-router-af)# no bgp suppress-inactive
```

The default behavior is to allow advertisement of eligible BGP best paths that experience RIB failures.

This command affects BGP route advertisement. It does not:

- Change BGP best-path selection.
- Change administrative distance.
- Force a BGP path into the routing table.
- Change the forwarding next hop.

---

## Other Causes of RIB Installation Failure

Although a lower administrative distance is the most common cause, BGP routes can fail RIB installation for other reasons.

Examples include:

| Cause | Description |
|---|---|
| Higher administrative distance | Another route source is preferred |
| Resource limitations | Insufficient resources for installation |
| VRF route limits | The applicable route limit has been reached |
| Other RIB restrictions | The routing infrastructure rejects the route |

Use the RIB-failure verification command to determine the actual cause:

```text
R1# show ip bgp rib-failure
```

Do not assume that every RIB failure is caused by administrative distance.

---

## RIB Failure vs. Unreachable NEXT_HOP

These are different conditions.

| Characteristic | RIB Failure | Unreachable NEXT_HOP |
|---|---|---|
| NEXT_HOP reachable? | Normally yes | No |
| Eligible for best-path selection? | Yes | No, under normal operation |
| Can be BGP best path? | Yes | No |
| Installed in RIB? | No | No |
| Can be advertised normally? | Yes, subject to policy and suppression | Not as an eligible best path |

### Example: Unreachable NEXT_HOP

```text
Prefix:   192.0.2.0/24
NEXT_HOP: 10.10.10.10
```

If R1 has no usable route to `10.10.10.10`, the BGP path is not eligible for normal best-path selection.

This is **not a RIB failure**.

By contrast, a RIB failure occurs after BGP has already selected a path and attempted to install it.

See [NEXT_HOP](../attributes/next-hop.md) for next-hop reachability and recursive resolution.

---

## Verification and Troubleshooting

### 1. Display BGP Best Paths

```text
R1# show bgp ipv4 unicast
R1# show bgp ipv4 unicast 192.0.2.0/24
```

Identify:

- Best-path status (`>`).
- RIB-failure status (`r`).
- BGP NEXT_HOP.
- Path attributes.

### 2. Display RIB Failures

```text
R1# show ip bgp rib-failure
```

Check:

- Prefix.
- BGP next hop.
- RIB-failure reason.
- `RIB-NH Matches`.

### 3. Verify the Installed Route

```text
R1# show ip route 192.0.2.0
```

Determine which routing source actually installed the prefix.

Check its:

- Administrative distance.
- Metric.
- Next hop.
- Outgoing interface.

### 4. Verify Forwarding

```text
R1# show ip cef 192.0.2.1 detail
```

CEF shows the actual forwarding information derived from the installed route.

Use this command when troubleshooting differences between the BGP NEXT_HOP and the forwarding next hop.

### 5. Verify Advertised Routes

```text
R1# show bgp ipv4 unicast neighbors 10.0.14.4 advertised-routes
```

Determine whether the RIB-failed route is still advertised.

### 6. Verify BGP Configuration

```text
R1# show running-config | section router bgp
```

Check for:

```text
distance bgp
bgp suppress-inactive
```

Also inspect any policies that might affect route eligibility or advertisement.

---

## Summary

| Scenario | BGP Best Path? | Installed in RIB? | Advertised by Default? |
|---|---|---|---|
| BGP best path wins RIB installation | Yes | Yes | Yes |
| BGP best path loses to lower-AD route | Yes | No | Yes |
| BGP path has unreachable NEXT_HOP | No | No | No |
| RIB failure, `suppress-inactive`, different forwarding next hop | Yes | No | No |
| RIB failure, `suppress-inactive`, same forwarding next hop | Yes | No | Yes |

Advertisement results assume the path otherwise satisfies normal BGP advertisement rules and outbound policies.

**Key points:**

- BGP selects its best path before administrative distance is evaluated.
- Administrative distance determines which routing source wins installation for the same prefix.
- A BGP best path can experience a RIB failure.
- BGP does not normally fall back to its second-best path merely because its best path loses RIB installation.
- By default, RIB-failed BGP best paths can still be advertised.
- `bgp suppress-inactive` suppresses higher-AD RIB-failed paths when their resolved forwarding next hops differ from the installed route.
- A matching forwarding next hop allows advertisement even when `bgp suppress-inactive` is configured.

---

## References

- [Cisco: Understand BGP RIB-Failure and BGP Suppress-Inactive Command](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/213286-understand-bgp-rib-failure-and-the-bgp-s.html)
- [Cisco: Next-Hop Mismatch and BGP Inactive Routes Tech Note](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/116146-config-bgp-next-hop-00.html)
- [Cisco: Configuring a Basic BGP Network](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-basic-net-0.html)
- [Cisco: BGP Command Reference - distance bgp](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-c1.html)
- [Cisco: BGP Command Reference - show ip bgp rib-failure](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-s1.html)
