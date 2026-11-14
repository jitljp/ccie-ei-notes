# Weight and Local Preference

**Weight** and **LOCAL_PREF** are two of the most influential values in BGP best path selection.

Both prefer the **highest value**, but they differ in scope:

- **Weight:** Influences path selection on a single Cisco router.
- **LOCAL_PREF:** Influences path selection throughout an autonomous system.

BGP evaluates Weight before LOCAL_PREF during best path selection.

---

## Weight vs. LOCAL_PREF

| Characteristic | Weight | LOCAL_PREF |
|---|---|---|
| BGP path attribute? | No, Cisco-specific parameter | Yes |
| Attribute category | N/A | Well-known discretionary |
| Default value | 0 for learned routes | 100 |
| Locally originated routes | Default Weight 32768 | Default LOCAL_PREF 100 |
| Value range | 0-65535 | 0-4294967295 |
| Preferred value | Highest | Highest |
| Advertised to IBGP peers? | No | Yes |
| Advertised to EBGP peers? | No | No, except within confederations |
| Best path selection order | 1st | 2nd |
| Primary purpose | Local outbound path selection | AS-wide outbound path selection |

Neither Weight nor LOCAL_PREF directly controls which path a remote AS chooses when sending traffic toward your network.

---

## Weight

**Weight** is a Cisco-specific BGP parameter used to influence path selection on an individual router.

It is not a standard BGP path attribute and is **never included in BGP UPDATE messages**.

Important characteristics:

- Higher Weight is preferred.
- Weight is evaluated before LOCAL_PREF.
- The default Weight for paths learned from BGP neighbors is **0**.
- The default Weight for locally originated paths is **32768**.
- Weight changes affect only the local router.

### Configuring Weight for a Neighbor

Use `neighbor weight` to assign a Weight to all routes received from a particular BGP neighbor.

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 weight 500
```

All routes learned from `10.0.12.2` are assigned Weight **500** on R1.

This does not change the routes advertised to the neighbor or the Weight values used by other routers.

### Configuring Weight with a Route Map

To assign Weight only to selected routes, use an inbound route map.

For example, assign Weight 500 to `192.0.2.0/24` learned from `10.0.12.2`.

```text
R1(config)# ip prefix-list PFX-WEIGHT permit 192.0.2.0/24

R1(config)# route-map SET-WEIGHT permit 10
R1(config-route-map)# match ip address prefix-list PFX-WEIGHT
R1(config-route-map)# set weight 500

R1(config)# route-map SET-WEIGHT permit 20

R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-WEIGHT in
```

The result is:

- `192.0.2.0/24` receives Weight 500.
- Other routes from the neighbor retain their normal Weight values.
- The second route-map entry permits routes that do not match the first entry.

**Important:** Without the second permit entry, other routes would be denied by the route map's implicit deny.

If both `neighbor weight` and an inbound route map using `set weight` apply to a route, the **route-map Weight takes precedence**.

---

## LOCAL_PREF

**LOCAL_PREF (Local Preference)** is a well-known discretionary BGP path attribute used to influence path selection throughout an autonomous system.

Important characteristics:

- Higher LOCAL_PREF is preferred.
- The default LOCAL_PREF is **100**.
- LOCAL_PREF is evaluated after Weight.
- LOCAL_PREF is propagated between IBGP peers.
- LOCAL_PREF is not advertised to ordinary EBGP peers.
- It is commonly used to select the preferred exit point from an AS.

Although LOCAL_PREF is classified as well-known discretionary, it is required in ordinary IBGP UPDATE messages advertising reachable routes.

### LOCAL_PREF Operation

Consider the following network:

```text
       ISP-A                         ISP-B
      AS 65100                      AS 65200
         |                             |
         | EBGP                        | EBGP
         |                             |
         R1                            R2
         |                             |
         |         AS 65000            |
         +------------ R3 -------------+
                    IBGP
```

Both ISP-A and ISP-B advertise `192.0.2.0/24`.

Suppose R1 assigns LOCAL_PREF 200 to the path through ISP-A, while R2 assigns LOCAL_PREF 100 to the path through ISP-B.

| Path | LOCAL_PREF |
|---|---|
| Via ISP-A (R1) | 200 |
| Via ISP-B (R2) | 100 |

Assuming Weight is equal and the paths are otherwise eligible, **the path through ISP-A is preferred**.

Because LOCAL_PREF is propagated through IBGP, routers within AS 65000 can use the same policy to prefer ISP-A.

This allows an AS-wide outbound traffic policy without configuring Weight separately on every router.

**Note:** The topology assumes the necessary IBGP connectivity and next-hop reachability. Each BGP router normally advertises only its selected best path, so not every router will necessarily see both alternatives.

---

## Configuring LOCAL_PREF

### Using a Route Map

LOCAL_PREF is commonly configured on an inbound route map applied to an EBGP neighbor.

For example, on R1:

```text
R1(config)# route-map ISP-A-IN permit 10
R1(config-route-map)# set local-preference 200

R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map ISP-A-IN in
```

All routes received from ISP-A are assigned LOCAL_PREF **200**.

R1 can then advertise these routes to its IBGP peers with LOCAL_PREF 200, subject to normal BGP advertisement rules.

An ISP-B connection using the default LOCAL_PREF of 100 will be less preferred, assuming equal Weight.

### Matching Specific Prefixes

To change LOCAL_PREF only for selected routes:

```text
R1(config)# ip prefix-list PREFERRED permit 192.0.2.0/24

R1(config)# route-map ISP-A-IN permit 10
R1(config-route-map)# match ip address prefix-list PREFERRED
R1(config-route-map)# set local-preference 200

R1(config)# route-map ISP-A-IN permit 20

R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map ISP-A-IN in
```

The result is:

- `192.0.2.0/24` receives LOCAL_PREF 200.
- Other received routes retain their normal LOCAL_PREF values.
- All routes are permitted by the route map.

This allows different prefixes to prefer different exits from the AS.

### Changing the Default LOCAL_PREF

Use `bgp default local-preference` to change the default LOCAL_PREF value for the local BGP process.

```text
R1(config)# router bgp 65000
R1(config-router)# bgp default local-preference 150
```

This changes the default from 100 to **150**.

The configured default applies to paths without an explicitly assigned LOCAL_PREF. It does not automatically override an explicit LOCAL_PREF received through IBGP.

To restore the default:

```text
R1(config-router)# no bgp default local-preference
```

**Important:** `bgp default local-preference` is a local BGP configuration command. It is not automatically configured on other routers, although LOCAL_PREF values can be propagated to them through IBGP.

For selective traffic engineering, an inbound route map is normally preferable to changing the process-wide default.

---

## Weight vs. LOCAL_PREF in Path Selection

Weight is evaluated before LOCAL_PREF.

Consider two eligible BGP paths to `192.0.2.0/24`:

| Attribute | Path A | Path B |
|---|---|---|
| Weight | 500 | 0 |
| LOCAL_PREF | 100 | 300 |
| AS_PATH length | 3 | 1 |

**Path A is preferred** because it has a higher Weight.

BGP does not compare LOCAL_PREF or AS_PATH length because Weight already determines the best path.

Now consider:

| Attribute | Path A | Path B |
|---|---|---|
| Weight | 0 | 0 |
| LOCAL_PREF | 200 | 100 |
| AS_PATH length | 3 | 1 |

**Path A is preferred** because Weight is equal and Path A has a higher LOCAL_PREF.

The shorter AS_PATH of Path B does not matter.

See [BGP Best Path Selection Steps](../best-path/selection-steps.md) for the complete algorithm.

---

## Verification

### Display Weight and LOCAL_PREF

```text
R1# show bgp ipv4 unicast
R1# show bgp ipv4 unicast 192.0.2.0/24
```

Example BGP table:

```text
   Network          Next Hop            Metric LocPrf Weight Path
*  192.0.2.0/24     10.0.12.2                0    200      0 65100 i
*>                  10.0.13.3                0    100    500 65200 i
```

The path through `10.0.13.3` is selected as best because it has Weight 500, even though the other path has a higher LOCAL_PREF.

Important fields:

| Field | Meaning |
|---|---|
| `LocPrf` | LOCAL_PREF value |
| `Weight` | Locally assigned Cisco Weight |
| `*` | Valid path |
| `>` | Best path |

**Note:** The summary table may leave `LocPrf` blank for directly EBGP-learned paths even when the effective default LOCAL_PREF is 100. Use the detailed prefix output to inspect the local preference.

### Verify Route Maps and Configuration

```text
R1# show route-map
R1# show ip prefix-list
R1# show running-config | section router bgp
R1# show bgp ipv4 unicast neighbors 10.0.12.2
```

### Reapply Inbound Policy

After modifying an inbound BGP route map, a soft inbound reset can be used to reprocess received routes:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

With route-refresh support, this allows routes to be refreshed without tearing down the BGP session.

---

## Key Takeaways

- **Weight** is Cisco-specific, locally significant, and never advertised.
- **LOCAL_PREF** is a standard BGP attribute propagated through IBGP.
- Both prefer the **highest value**.
- Weight is evaluated **before LOCAL_PREF**.
- Default Weight is **0 for learned routes** and **32768 for locally originated routes**.
- Default LOCAL_PREF is **100**.
- `neighbor weight` sets Weight for routes received from a neighbor.
- `set weight` in an inbound route map can override neighbor-level Weight for matching routes.
- `set local-preference` in a route map is commonly used for AS-wide outbound traffic engineering.
- Neither parameter directly controls inbound traffic decisions made by remote autonomous systems.

---

## References

- [Cisco: Select BGP Best-path Algorithm](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html)
- [Cisco: BGP Configuration Guide, Understanding BGP](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr3-fwd/bgp/bgp-configuration-guide/routing-bgp.html)
- [Cisco: BGP Commands A through B](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-a1.html)
- [Cisco: BGP Commands M through N](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [RFC 4271: A Border Gateway Protocol 4 (BGP-4)](https://www.rfc-editor.org/rfc/rfc4271.html)
