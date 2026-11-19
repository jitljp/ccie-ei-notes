# Allowas-in

The BGP **Allowas-in** feature allows a router to accept routes containing its own Autonomous System Number (ASN) in the AS_PATH attribute.

By default, BGP rejects routes containing the local ASN because they are considered potential routing loops.

The `neighbor allowas-in` command relaxes this loop-prevention mechanism for routes received from a specific EBGP neighbor.

Allowas-in is commonly used in:

- **MPLS Layer 3 VPNs:** When multiple customer sites use the same ASN.
- **Multisite enterprise networks:** When geographically separated locations share an ASN but communicate through another autonomous system.
- **AS migration scenarios:** When an ASN may temporarily appear multiple times in a routing path.
- **Specialized EBGP designs:** When a valid route must pass through an AS that uses the receiving router's ASN elsewhere in the network.

Allowas-in does **not** modify the AS_PATH attribute. It changes whether routes containing the local ASN are accepted.

---

## AS_PATH Loop Prevention

The AS_PATH attribute records the autonomous systems a BGP route has traversed.

When a router advertises a route to an EBGP neighbor, it normally prepends its own ASN.

Consider the following topology:

```text
       AS 65001           AS 65002           AS 65003

          R1 ---------------- R2 ---------------- R3
                 EBGP                EBGP

     10.10.10.0/24
```

R1 originates `10.10.10.0/24`.

The AS_PATH changes as follows:

| Router | AS_PATH |
|---|---|
| R1 | Empty |
| R2 | `65001` |
| R3 | `65002 65001` |

Now suppose R3 advertises this route back to another router belonging to AS 65001.

The receiving router sees:

```text
AS_PATH: 65003 65002 65001
```

Because its own ASN, `65001`, appears in the AS_PATH, it rejects the route.

This prevents routes from circulating indefinitely between autonomous systems.

### Default Behavior

A router rejects a received route if its own ASN appears anywhere in the AS_PATH.

For example, R1 belongs to AS 65001.

| Received AS_PATH | Default Result |
|---|---|
| `65002` | Accepted |
| `65002 65003` | Accepted |
| `65002 65001` | Rejected |
| `65002 65001 65003` | Rejected |
| `65002 65001 65003 65001` | Rejected |

The local ASN does not need to appear at the beginning or end of the AS_PATH.

**A single occurrence of the local ASN is sufficient to trigger normal AS_PATH loop prevention.**

This check is independent of AS_PATH length comparison during BGP best-path selection.

---

## Allowas-in Configuration

Cisco IOS XE supports the following command:

```text
R1(config-router-af)# neighbor <IP> allowas-in [<number>]
```

The command can also be entered in BGP router configuration mode for supported address families and IOS XE releases.

The optional number specifies the **maximum number of occurrences of the local ASN** permitted in received AS_PATHs.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 allowas-in 2
```

This allows R1 to accept routes from neighbor `10.0.12.2` containing AS 65001 up to two times.

### Allowas-in Occurrence Limit

| Parameter | Value |
|---|---|
| Minimum configurable value | 1 |
| Maximum configurable value | 10 |
| Default when omitted | 3 |
| Direction | Inbound |
| Scope | Per neighbor |
| AS_PATH modified? | No |

For example:

```text
R1(config-router-af)# neighbor 10.0.12.2 allowas-in
```

Without an explicit number, the maximum permitted occurrence count is **3**.

This is equivalent in effect to:

```text
R1(config-router-af)# neighbor 10.0.12.2 allowas-in 3
```

The feature is disabled by default.

### Occurrence Limit Behavior

Suppose R1 belongs to AS 65001 and has configured:

```text
R1(config-router-af)# neighbor 10.0.12.2 allowas-in 2
```

The following table shows how received routes are handled.

| Received AS_PATH | Occurrences of 65001 | Result |
|---|---:|---|
| `65002 65003` | 0 | Accepted |
| `65002 65001` | 1 | Accepted |
| `65002 65001 65003 65001` | 2 | Accepted |
| `65002 65001 65001 65001` | 3 | Rejected |
| `65002 65001 65003 65001 65004 65001` | 3 | Rejected |

The configured limit is **inclusive**.

With `allowas-in 2`:

- One occurrence is allowed.
- Two occurrences are allowed.
- Three or more occurrences are rejected.

**Important:** The number specifies occurrences of the receiving router's ASN, not the total AS_PATH length.

For example:

```text
65002 65003 65004 65005 65001
```

This AS_PATH contains five ASN occurrences, but only one occurrence of the local AS 65001.

Therefore, it passes the `allowas-in 2` occurrence check.

---

## Allowas-in Example: Two Sites Using the Same ASN

Consider an enterprise with two sites connected through a service provider.

Both sites belong to AS 65001.

The service provider belongs to AS 65000.

### Topology

```text
        Site A                  ISP                  Site B
       AS 65001              AS 65000              AS 65001

          R1 ------------------ R2 ------------------ R3
            E0/0          E0/0   E0/1          E0/0
             |              |     |              |
          10.0.12.1    10.0.12.2 10.0.23.2    10.0.23.3

   Lo0: 10.10.10.1/24                   Lo0: 10.20.20.1/24
```

R1 and R3 use the same ASN, but they are located at different sites.

R1 advertises `10.10.10.0/24`.

R3 advertises `10.20.20.0/24`.

### Initial R1 Configuration

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip address 10.0.12.1 255.255.255.252
R1(config-if)# no shutdown

R1(config)# interface Loopback0
R1(config-if)# ip address 10.10.10.1 255.255.255.0

R1(config)# router bgp 65001
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# neighbor 10.0.12.2 remote-as 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# network 10.10.10.0 mask 255.255.255.0
```

### R2 Configuration

```text
R2(config)# interface Ethernet0/0
R2(config-if)# ip address 10.0.12.2 255.255.255.252
R2(config-if)# no shutdown

R2(config)# interface Ethernet0/1
R2(config-if)# ip address 10.0.23.2 255.255.255.252
R2(config-if)# no shutdown

R2(config)# router bgp 65000
R2(config-router)# bgp router-id 2.2.2.2
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
R2(config-router)# neighbor 10.0.23.3 remote-as 65001
```

### Initial R3 Configuration

```text
R3(config)# interface Ethernet0/0
R3(config-if)# ip address 10.0.23.3 255.255.255.252
R3(config-if)# no shutdown

R3(config)# interface Loopback0
R3(config-if)# ip address 10.20.20.1 255.255.255.0

R3(config)# router bgp 65001
R3(config-router)# bgp router-id 3.3.3.3
R3(config-router)# neighbor 10.0.23.2 remote-as 65000
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# network 10.20.20.0 mask 255.255.255.0
```

### Initial Routing Behavior

R1 advertises `10.10.10.0/24` to R2.

R2 receives:

```text
AS_PATH: 65001
```

R2 advertises the route to R3.

R3 receives:

```text
AS_PATH: 65000 65001
```

R3 belongs to AS 65001.

Therefore, it rejects the route because its own ASN appears in the AS_PATH.

The BGP session remains established, but R3 does not accept the route.

The same problem occurs in the opposite direction.

R3 advertises `10.20.20.0/24` to R2.

R2 advertises the route to R1 with:

```text
AS_PATH: 65000 65001
```

R1 also rejects the route.

**Both EBGP sessions can be Established even though the customer sites cannot exchange routes.**

---

## Enabling Allowas-in

To resolve the problem, configure `allowas-in` on both customer routers.

### R1 Configuration

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 allowas-in 1
```

### R3 Configuration

```text
R3(config)# router bgp 65001
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# neighbor 10.0.23.2 allowas-in 1
```

Each router now accepts routes containing its own ASN up to one time.

### Result on R1

R1 receives `10.20.20.0/24` from R2.

```text
AS_PATH: 65000 65001
```

The AS_PATH contains one occurrence of R1's ASN.

Therefore, R1 accepts the route.

Assuming the route is valid, selected as best, and eligible for installation, R1 installs it in the routing table.

### Result on R3

R3 receives `10.10.10.0/24` from R2.

```text
AS_PATH: 65000 65001
```

R3 also accepts the route.

Both customer sites can now exchange routing information through the ISP.

### Before and After Allowas-in

| Condition | R1 | R3 |
|---|---|---|
| Before `allowas-in` | Rejects R3's prefix | Rejects R1's prefix |
| After `allowas-in 1` | Accepts R3's prefix | Accepts R1's prefix |

**Important:** If connectivity is required in both directions, the necessary route acceptance must be configured on both receiving routers.

Configuring `allowas-in` on only R1 does not automatically allow R3 to accept routes containing AS 65001.

---

## Allowas-in Does Not Modify AS_PATH

Unlike AS_PATH prepending, AS-Override, or some Local-AS operations, **Allowas-in does not modify the AS_PATH attribute**.

Consider R3 receiving:

```text
AS_PATH: 65000 65001
```

Without `allowas-in`, the route is rejected.

With:

```text
R3(config-router-af)# neighbor 10.0.23.2 allowas-in 1
```

The route is accepted with the same AS_PATH:

```text
AS_PATH: 65000 65001
```

Nothing is removed, replaced, or reordered.

This distinction matters because the AS_PATH continues to affect:

- BGP best-path selection.
- AS_PATH filtering.
- Subsequent route advertisements.
- AS_PATH loop detection on other routers.

For example, if R3 later advertises the route to another EBGP neighbor, normal outbound AS_PATH processing still applies, subject to BGP advertisement rules and configured policies.

**Allowas-in relaxes the receiving router's AS loop check. It does not globally disable AS_PATH loop prevention.**

---

## Allowas-in and BGP Best-Path Selection

Allowas-in affects **route acceptance**, not the best-path selection algorithm.

Once a route is accepted, it undergoes normal BGP best-path selection.

Consider R1 in AS 65001 receiving two paths to `10.20.20.0/24`.

| Attribute | Path A | Path B |
|---|---|---|
| Neighbor | R2 | R4 |
| AS_PATH | `65000 65001` | `65003` |
| Weight | 0 | 0 |
| LOCAL_PREF | 100 | 100 |
| ORIGIN | IGP | IGP |

Without `allowas-in`, Path A is rejected because it contains AS 65001.

With `allowas-in 1` configured toward R2, Path A becomes eligible for consideration.

However, R1 does not necessarily prefer Path A.

Assuming all earlier selection criteria are equal, R1 prefers Path B because its AS_PATH is shorter.

```text
Path A length: 2
Path B length: 1
```

**Allowas-in does not give a route higher preference.**

It only allows routes that would otherwise fail the local-AS loop check to participate in normal BGP processing.

Also, acceptance into the BGP table does not guarantee installation into the routing table.

A route may still fail to become best because of:

- Weight or LOCAL_PREF.
- AS_PATH length.
- Other BGP best-path criteria.
- Inaccessible NEXT_HOP.
- Inbound routing policies.

Even a selected best BGP path may fail RIB installation if another routing protocol provides a more preferred route.

---

## Allowas-in vs. AS-Override

**Allowas-in** and **AS-Override** are two common solutions for networks where multiple customer sites use the same ASN.

Both can solve the same AS_PATH loop-prevention problem, but they operate differently.

### Allowas-in

Allowas-in is configured on the receiving router.

For example, R3 belongs to AS 65001 and receives:

```text
AS_PATH: 65000 65001
```

Configure:

```text
R3(config)# router bgp 65001
R3(config-router)# neighbor 10.0.23.2 remote-as 65000
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# neighbor 10.0.23.2 allowas-in 1
```

R3 accepts the route without changing its AS_PATH.

```text
AS_PATH: 65000 65001
```

### AS-Override

AS-Override is configured on the advertising router.

In the same topology, R2 belongs to AS 65000 and R3 belongs to AS 65001.

R2 learns a route with:

```text
AS_PATH: 65001
```

R2 can configure:

```text
R2(config)# router bgp 65000
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.23.3 as-override
```

When advertising the route to R3, R2 replaces occurrences of R3's ASN in the AS_PATH with its own ASN.

After normal EBGP AS prepending, R3 receives:

```text
AS_PATH: 65000 65000
```

R3 can accept the route without requiring `allowas-in`, because AS 65001 no longer appears in the received AS_PATH.

### Comparison

| Characteristic | Allowas-in | AS-Override |
|---|---|---|
| Configured on | Receiving router | Advertising router |
| Direction | Inbound | Outbound |
| Modifies AS_PATH? | No | Yes |
| Allows the local ASN in received AS_PATH? | Yes | Not required |
| Controls occurrence count? | Yes | No |
| Typical MPLS L3VPN configuration | CE | PE |
| Normal loop prevention relaxed? | Yes | AS_PATH information is altered |

The primary difference is:

- **Allowas-in:** Accept the AS_PATH containing the local ASN.
- **AS-Override:** Modify the AS_PATH so the receiving router does not see its own ASN.

### MPLS Layer 3 VPN Example

In MPLS L3VPN environments, the topology typically looks like this:

```text
               MPLS Provider Core
              AS 65000 (MP-BGP)

          PE1 =================== PE2
           |                       |
          EBGP                    EBGP
           |                       |
          CE1                     CE2
       AS 65001                AS 65001
           |                       |
     10.10.10.0/24          10.20.20.0/24
```

The same-AS problem occurs because both CE routers use AS 65001.

Two common solutions are:

1. Configure `allowas-in` on the CE routers.
2. Configure `as-override` on the PE routers.

For a complete MPLS L3VPN example, see [BGP AS-Override and Allowas-in](../../../../transport-technologies-and-solutions/mpls/l3vpn/pe-ce/bgp/bgp-as-override-allowas.md).

---

## Allowas-in with Inbound Route Filtering

Allowas-in relaxes an important BGP loop-prevention mechanism.

Therefore, it is good practice to combine it with route filtering when only specific prefixes should be accepted.

Consider R3 in the earlier example.

R3 should accept only `10.10.10.0/24` from R2.

### Prefix List Configuration

```text
R3(config)# ip prefix-list FROM-ISP permit 10.10.10.0/24
```

Apply the prefix list inbound:

```text
R3(config)# router bgp 65001
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# neighbor 10.0.23.2 allowas-in 1
R3(config-router-af)# neighbor 10.0.23.2 prefix-list FROM-ISP in
```

This configuration applies two independent restrictions:

1. `allowas-in 1` permits a received AS_PATH containing AS 65001 once.
2. The prefix list permits only `10.10.10.0/24`.

A route must satisfy both conditions to be accepted.

For example:

| Received Prefix | AS_PATH | Result |
|---|---|---|
| `10.10.10.0/24` | `65000 65001` | Accepted |
| `10.10.10.0/24` | `65000 65001 65001` | Rejected by occurrence limit |
| `10.30.30.0/24` | `65000 65001` | Rejected by prefix list |
| `10.40.40.0/24` | `65000 65002` | Rejected by prefix list |

The prefix list has an implicit deny at the end.

### Using a Route Map

The same policy can be implemented using a route map.

```text
R3(config)# ip prefix-list FROM-ISP permit 10.10.10.0/24

R3(config)# route-map FILTER-ISP permit 10
R3(config-route-map)# match ip address prefix-list FROM-ISP

R3(config)# router bgp 65001
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# neighbor 10.0.23.2 allowas-in 1
R3(config-router-af)# neighbor 10.0.23.2 route-map FILTER-ISP in
```

The route map allows additional routing policies to be applied, such as modifying LOCAL_PREF or other BGP attributes.

**Important:** The `allowas-in` command itself is not a route-map `set` action. It is a neighbor-level BGP feature.

---

## Allowas-in and Routing Loop Risks

Allowas-in intentionally relaxes BGP's normal inter-AS loop prevention.

This does not mean that enabling it always causes routing loops.

However, allowing routes containing the local ASN can create problems in more complicated topologies.

Potential risks include:

- Receiving routes that originated locally and returned through another AS.
- Selecting an unintended path through a remote site.
- Re-advertising routes through alternate EBGP connections.
- Creating loops when combined with redistribution.
- Routing loops involving dual-homed sites or backdoor links.

### Example: Multiple Paths Between Sites

Consider:

```text
                  ISP AS 65000

              R2 ---------------- R4
              |                    |
              |                    |
             R1 ------------------ R3
           AS 65001            AS 65001
                    Backdoor
                      Link
```

R1 and R3 can exchange routes through the ISP and through a direct backdoor connection.

If `allowas-in` is configured without appropriate routing policies, a route may be learned through the ISP even though it originally came from the same customer AS.

If the route is then re-advertised or redistributed into another routing domain, traffic may follow an unintended path.

### Loop-Prevention Measures

Several mechanisms can reduce this risk:

- Restrict accepted prefixes with inbound prefix lists.
- Apply AS-path access lists when appropriate.
- Use route maps to control advertisement and preference.
- Avoid unnecessary mutual redistribution.
- Use Site of Origin (SoO) in relevant MPLS VPN designs.
- Configure the lowest necessary `allowas-in` occurrence limit.

**Important:** `allowas-in 1` is generally safer than `allowas-in 10` when only one occurrence of the local ASN is required.

A larger occurrence limit allows more repeated appearances of the local ASN and can permit paths that should otherwise be rejected.

---

## Allowas-in Restrictions

Cisco documents several restrictions and operational considerations.

### EBGP Requirement

The standard `neighbor allowas-in` feature is intended for **true EBGP peers**.

It is not supported for peering between different member ASNs of the same BGP confederation.

For example:

```text
       Confederation AS 65000

        Member AS 65001       Member AS 65002

               R1 ---------------- R2
                     Confed EBGP
```

This is not a normal true-EBGP peering relationship.

The standard `allowas-in` feature cannot be used to bypass confederation AS loop detection between these member ASes.

### Per-Neighbor Behavior

Allowas-in applies to routes received from the configured neighbor.

For example:

```text
R1(config-router-af)# neighbor 10.0.12.2 allowas-in 1
```

This does not automatically permit routes containing the local ASN when received from other neighbors.

The feature must be configured for each relevant neighbor or inherited through supported peer configuration mechanisms.

### No AS_PATH Modification

Allowas-in does not:

- Remove the local ASN from AS_PATH.
- Replace an ASN with another ASN.
- Prepend additional ASNs.
- Change the neighbor's ASN.
- Disable other BGP loop-prevention mechanisms.
- Change BGP best-path selection order.

### First-AS Enforcement

Allowas-in is separate from the `bgp enforce-first-as` command.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp enforce-first-as
```

First-AS enforcement verifies that the first ASN in a received EBGP AS_PATH matches the advertising neighbor's ASN.

Allowas-in controls whether the local ASN is permitted elsewhere in the AS_PATH.

Consider R1 in AS 65001 receiving a route from R2 in AS 65002.

```text
AS_PATH: 65002 65001
```

This route:

- Passes first-AS enforcement because the first ASN is 65002.
- Fails normal local-AS loop detection because AS 65001 appears in the path.
- Can be accepted with `allowas-in 1`.

Now consider:

```text
AS_PATH: 65003 65001
```

Even with `allowas-in 1`, the route can still fail first-AS enforcement because the leftmost ASN does not match R2's ASN of 65002.

**Allowas-in does not disable first-AS enforcement.**

---

## Verification and Troubleshooting

Several commands are useful for verifying Allowas-in behavior.

### Verify BGP Neighbor Status

```text
R3# show ip bgp summary
```

Check that the EBGP neighbor is Established.

A neighbor can be Established even if routes are being rejected because of AS_PATH loop prevention.

Therefore, an Established session does not prove that all expected routes have been accepted.

### Inspect the BGP Table

Before configuring `allowas-in`:

```text
R3# show ip bgp
```

The expected remote-site prefix may be missing because it was rejected during inbound processing.

After configuring:

```text
R3(config)# router bgp 65001
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# neighbor 10.0.23.2 allowas-in 1
```

And reprocessing the received routes, R3 can display an entry similar to:

```text
R3# show ip bgp
     Network          Next Hop        Metric LocPrf Weight Path
 *>  10.10.10.0/24    10.0.23.2                          0 65000 65001 i
 *>  10.20.20.0/24    0.0.0.0                       32768 i
```

This is illustrative output.

Notice that the accepted AS_PATH still contains R3's local ASN of 65001.

### Inspect a Specific Prefix

```text
R3# show ip bgp 10.10.10.0
```

Verify:

- The route exists in the BGP table.
- The AS_PATH contains the expected ASN occurrences.
- The NEXT_HOP is reachable.
- The route is selected as best if appropriate.

### Verify Routes Accepted from a Neighbor

```text
R3# show ip bgp neighbors 10.0.23.2 routes
```

This displays routes accepted from the specified neighbor.

The output helps distinguish routes accepted into the BGP table from prefixes that were rejected.

### Verify the Routing Table

```text
R3# show ip route 10.10.10.0
```

If the route is accepted by BGP but missing from the routing table, investigate:

- Whether it is the BGP best path.
- Whether the NEXT_HOP is reachable.
- Whether another route has a better administrative distance.
- Whether an inbound route policy rejects or modifies the route.

Allowas-in only addresses the local-AS loop check.

### Reprocess Routes After Configuration

After enabling `allowas-in`, previously rejected routes may need to be reprocessed.

Use an inbound soft clear:

```text
R3# clear ip bgp 10.0.23.2 soft in
```

When the neighbor supports BGP Route Refresh, this can request the routes again without resetting the established session.

If Route Refresh is unavailable, reprocessing may require previously retained inbound updates or a session reset.

### Debugging AS_PATH Loop Rejection

Cisco IOS XE can display messages indicating that received updates were rejected because the local ASN appeared in AS_PATH.

For example:

```text
BGP(0): 10.0.23.2 rcv UPDATE about 10.10.10.0/24
-- DENIED due to: AS-PATH contains our own AS;
```

This indicates that BGP rejected the route because of its local-AS loop-prevention check.

A related log message is:

```text
%BGP% Neighbor A.B.C.D recv bogus route : AS loop
```

In a controlled lab environment, BGP debugging can help identify the cause:

```text
R3# debug ip bgp updates
```

Be careful with BGP debugging on production routers because it can generate substantial output.

---

## Removing Allowas-in

To restore normal AS_PATH loop prevention for a neighbor:

```text
R3(config)# router bgp 65001
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# no neighbor 10.0.23.2 allowas-in
```

After the configuration change is applied and routes are reprocessed, routes containing the local ASN are rejected again.

For example:

```text
AS_PATH: 65000 65001
```

R3 belongs to AS 65001.

Without Allowas-in, this route fails the normal AS_PATH loop check.

---

## Command Summary

| Command | Purpose |
|---|---|
| `neighbor <IP> allowas-in` | Allow up to 3 occurrences of the local ASN |
| `neighbor <IP> allowas-in <N>` | Allow up to N occurrences of the local ASN |
| `no neighbor <IP> allowas-in` | Disable Allowas-in |
| `neighbor <IP> prefix-list <NAME> in` | Filter routes received from a neighbor |
| `neighbor <IP> route-map <NAME> in` | Apply an inbound routing policy |
| `show ip bgp` | Display BGP routes and AS_PATHs |
| `show ip bgp <PREFIX>` | Inspect a specific BGP route |
| `show ip bgp neighbors <IP> routes` | Display routes accepted from a neighbor |
| `show ip route <PREFIX>` | Verify RIB installation |
| `clear ip bgp <IP> soft in` | Reprocess inbound BGP routes |

---

## Key Points

- BGP normally rejects routes containing the local ASN in AS_PATH.
- This is an important inter-AS loop-prevention mechanism.
- `neighbor allowas-in` permits a router to accept routes containing its own ASN.
- Allowas-in is configured on the **receiving** router.
- It affects **inbound** BGP route processing.
- It is configured per neighbor.
- The optional number specifies the maximum occurrences of the local ASN permitted.
- The configurable occurrence range is **1 to 10**.
- The default occurrence limit when omitted is **3**.
- The occurrence limit is inclusive.
- The occurrence limit counts appearances of the local ASN, not the total AS_PATH length.
- Allowas-in **does not modify AS_PATH**.
- Allowas-in does not change BGP best-path selection order.
- Allowas-in does not guarantee that an accepted path will be selected as best or installed in the RIB.
- Allowas-in is commonly used when multiple customer sites share an ASN.
- In MPLS L3VPN environments, Allowas-in is commonly configured on CE routers.
- AS-Override is an alternative solution that modifies the AS_PATH on the advertising PE.
- Allowas-in and AS-Override solve similar problems but operate differently.
- Allowas-in does not disable first-AS enforcement.
- Standard Allowas-in is supported for true EBGP neighbors, not between confederation member ASes.
- Allowas-in weakens normal AS_PATH loop prevention and should be combined with appropriate routing policies.
- Use the lowest occurrence limit required by the network design.

---

## References

- [Cisco: Configure Allowas-in Feature in BGP](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/112236-allowas-in-bgp-config-example.html)
- [Cisco: AS-Override in MPLS L3VPN](https://community.cisco.com/t5/service-providers-knowledge-base/as-override-in-mpls-l3vpn/ta-p/3152004)
- [Cisco: BGP Command Reference](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [RFC 4271: A Border Gateway Protocol 4 (BGP-4)](https://www.rfc-editor.org/rfc/rfc4271.html)
