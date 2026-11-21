# BGP Soft Reconfiguration

**BGP soft reconfiguration** allows a router to reapply BGP routing policies without terminating an established BGP session.

Normally, when a BGP router receives routes from a neighbor, it processes them using its configured inbound routing policies.

These policies can:

- Accept or reject received routes.
- Modify path attributes such as Weight, LOCAL_PREF, and MED.
- Assign or remove BGP communities.
- Influence which routes are eligible for best-path selection.

When an inbound policy changes, the router may need to reevaluate routes that were previously received.

Cisco IOS XE supports two primary mechanisms for performing this operation without resetting the BGP session:

1. **Traditional inbound soft reconfiguration:** Stores unmodified copies of received routes locally so they can be reprocessed.
2. **Dynamic inbound soft reset (Route Refresh):** Requests that the neighbor readvertise its routes so they can be processed using the updated policy.

Although both mechanisms avoid a hard BGP session reset, they work differently.

This page focuses on traditional inbound soft reconfiguration. Route Refresh is covered separately in [BGP Route Refresh](route-refresh.md).

---

## Why Soft Reconfiguration Is Needed

Consider the following topology:

```text
       AS 65001                      AS 65002

          R1 -------------------------- R2
       10.0.12.1                    10.0.12.2
                      EBGP
```

R2 advertises three prefixes to R1:

```text
192.0.2.0/24
198.51.100.0/24
203.0.113.0/24
```

R1 has an inbound prefix list that only permits `192.0.2.0/24`.

```text
R1(config)# ip prefix-list FROM-R2 permit 192.0.2.0/24

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 prefix-list FROM-R2 in
```

The result is:

| Prefix | Inbound Policy | Result |
|---|---|---|
| 192.0.2.0/24 | Permit | Accepted |
| 198.51.100.0/24 | Implicit deny | Rejected |
| 203.0.113.0/24 | Implicit deny | Rejected |

Without an inbound soft reconfiguration mechanism, R1 does not retain the rejected routes in its normal BGP table.

### Changing the Inbound Policy

Suppose the administrator modifies the prefix list to permit `198.51.100.0/24` as well.

```text
R1(config)# ip prefix-list FROM-R2 permit 198.51.100.0/24
```

The updated prefix list permits two prefixes.

However, **changing the prefix list does not by itself guarantee that previously rejected routes will immediately become available**.

R1 must obtain or reprocess the original routing information.

There are three possibilities:

| Method | Operation |
|---|---|
| Hard reset | Terminate and reestablish the BGP session |
| Traditional inbound soft reconfiguration | Reprocess locally stored received routes |
| Route Refresh | Request the neighbor to readvertise its routes |

A hard reset is generally undesirable because it disrupts the BGP session and can cause temporary routing instability.

Soft reconfiguration and Route Refresh avoid this disruption.

---

## Traditional Inbound Soft Reconfiguration

**Traditional inbound soft reconfiguration** allows a BGP router to store an unmodified copy of routing information received from a neighbor.

The stored information is retained separately from the routes processed by inbound policies.

Consider the following process:

```text
                  BGP Neighbor
                       |
                       | UPDATE
                       v
              +------------------+
              | Received Routes  |
              +------------------+
                       |
              +--------+--------+
              |                 |
              v                 v
      +---------------+  +----------------+
      | Stored Copy   |  | Inbound Policy |
      | (Unmodified)  |  +----------------+
      +---------------+          |
                                 v
                        +----------------+
                        | Accepted BGP   |
                        | Routes         |
                        +----------------+
                                 |
                                 v
                         Best-Path
                         Selection
```

When traditional inbound soft reconfiguration is enabled:

1. The router receives BGP routing information from its neighbor.
2. An unmodified copy is retained in memory.
3. The inbound routing policy is applied.
4. Permitted routes are processed by BGP.
5. Rejected routes are excluded from normal best-path selection, but their original information remains available in the stored copy.

If the inbound policy changes, the router can reapply it to the stored information.

**The neighbor does not need to send the routes again.**

### Important Characteristics

- The BGP session remains Established.
- The original received routing information is retained locally.
- Routes rejected by inbound policies can still be stored.
- Modified attributes can be reevaluated using the original received attributes.
- The stored information requires additional memory.
- Route Refresh support is not required.

The memory requirement is the primary disadvantage of traditional inbound soft reconfiguration.

---

## Configuring Inbound Soft Reconfiguration

Enable traditional inbound soft reconfiguration for a neighbor:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 soft-reconfiguration inbound
```

This instructs BGP to retain unmodified inbound routing information from the specified neighbor.

The configuration is local to R1.

**R2 does not need any special configuration to support traditional inbound soft reconfiguration.**

### Configuration Syntax

```text
neighbor <IP> soft-reconfiguration inbound
```

The command can also be applied to a peer group:

```text
R1(config-router-af)# neighbor EBGP-PEERS soft-reconfiguration inbound
```

The members of the peer group inherit the configuration.

### Disabling Inbound Soft Reconfiguration

Remove the configuration:

```text
R1(config-router-af)# no neighbor 10.0.12.2 soft-reconfiguration inbound
```

This disables the additional storage associated with traditional inbound soft reconfiguration for that neighbor.

### When Does Route Storage Begin?

An important consideration is that enabling inbound soft reconfiguration causes the router to begin storing received updates.

**It does not necessarily reconstruct routes that were rejected before the feature was enabled.**

For example:

1. R1 receives `192.0.2.0/24` and `198.51.100.0/24`.
2. An inbound policy rejects `198.51.100.0/24`.
3. The administrator enables inbound soft reconfiguration.
4. The previously rejected route may not be present in the newly maintained stored information.

To ensure that the complete current set of received routes is available, the router must receive the routing information again.

This can be accomplished through Route Refresh, if supported, or by reestablishing the BGP session when necessary.

For predictable operation, configure traditional inbound soft reconfiguration before the routes need to be retained.

---

## Applying Updated Inbound Policies

After changing an inbound policy, perform an inbound soft reset.

```text
R1# clear ip bgp 10.0.12.2 soft in
```

When traditional inbound soft reconfiguration is in use, BGP can reprocess the stored received routes using the current inbound policy.

The BGP TCP session is not terminated.

### Example

Assume R2 advertises:

```text
192.0.2.0/24
198.51.100.0/24
203.0.113.0/24
```

R1 initially permits only `192.0.2.0/24`.

Configure inbound filtering and soft reconfiguration:

```text
R1(config)# ip prefix-list FROM-R2 permit 192.0.2.0/24

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 prefix-list FROM-R2 in
R1(config-router-af)# neighbor 10.0.12.2 soft-reconfiguration inbound
```

Assume the stored inbound routing information has been populated.

Initially, only one prefix is accepted:

```text
R1# show ip bgp neighbors 10.0.12.2 routes
```

The accepted routes include:

```text
192.0.2.0/24
```

However, the stored unmodified routing information contains all three prefixes.

Now modify the inbound prefix list:

```text
R1(config)# ip prefix-list FROM-R2 permit 198.51.100.0/24
```

Apply the updated policy:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

R1 reprocesses the stored information.

The results are now:

| Prefix | Original Policy | Updated Policy |
|---|---|---|
| 192.0.2.0/24 | Accepted | Accepted |
| 198.51.100.0/24 | Rejected | Accepted |
| 203.0.113.0/24 | Rejected | Rejected |

The BGP session remains Established throughout the process.

**No additional UPDATE messages from R2 are necessary when the required routes are available in R1's stored inbound information.**

---

## Soft Reconfiguration and Attribute Manipulation

Soft reconfiguration is not limited to accepting or rejecting routes.

It can also be used to reapply policies that modify BGP path attributes.

For example, suppose R1 applies an inbound route map that sets LOCAL_PREF to 200.

```text
R1(config)# route-map SET-LOCAL-PREF permit 10
R1(config-route-map)# set local-preference 200

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-LOCAL-PREF in
R1(config-router-af)# neighbor 10.0.12.2 soft-reconfiguration inbound
```

Assume a route is received with LOCAL_PREF 100.

The inbound route map changes the LOCAL_PREF to 200 for local BGP processing.

The stored unmodified copy retains the original received attribute information.

Now modify the route map:

```text
R1(config)# route-map SET-LOCAL-PREF permit 10
R1(config-route-map)# set local-preference 300
```

Perform an inbound soft reset:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

BGP reprocesses the original received routing information using the updated policy.

The resulting route has LOCAL_PREF 300.

### Why Store Unmodified Information?

Consider what would happen if BGP only retained the attributes after applying the original inbound policy.

It could be difficult to reconstruct the route's original attributes when the policy changes.

Storing the original information allows the router to evaluate the route as if it had just been received from its neighbor.

This is especially important when policies modify multiple attributes or when a previously rejected route becomes permitted.

**The purpose of inbound soft reconfiguration is to reapply policies to the original received routing information, not merely to the already modified routes.**

---

## Soft Reconfiguration vs. Route Refresh

Traditional inbound soft reconfiguration and Route Refresh both allow inbound policies to be reapplied without terminating the BGP session.

However, they obtain the routing information differently.

### Traditional Inbound Soft Reconfiguration

The router retains copies of the original received routes.

When the policy changes, it reprocesses the locally stored information.

```text
          R1                          R2
           |                           |
           |      BGP Established      |
           |<--------------------------|
           |        UPDATEs            |
           |                           |
    Store Received Routes
           |
    Inbound Policy Changes
           |
    clear ip bgp ... soft in
           |
    Reprocess Stored Routes
           |
    BGP Session Remains Up
```

No new route advertisement is required from R2 for this local reprocessing.

### Route Refresh

With Route Refresh, the router does not need to maintain a separate unmodified copy of every received route.

Instead, it asks its neighbor to readvertise the routing information.

```text
          R1                          R2
           |                           |
           |      BGP Established      |
           |                           |
    Inbound Policy Changes             |
           |                           |
    clear ip bgp ... soft in           |
           |                           |
           |------ ROUTE-REFRESH ----->|
           |                           |
           |<--------- UPDATEs --------|
           |                           |
    Apply Updated Policy               |
           |                           |
    BGP Session Remains Up             |
```

### Comparison

| Characteristic | Traditional Inbound Soft Reconfiguration | Route Refresh |
|---|---|---|
| Stores original received routes locally | Yes | Not required |
| Requires additional memory for stored routes | Yes | No separate stored copy required |
| Requires neighbor support for a special capability | No | Yes |
| Requests route readvertisement | No, when using stored routes | Yes |
| Resets the BGP session | No | No |
| Can reapply inbound policies | Yes | Yes |
| Can reconsider previously rejected routes | Yes, if stored | Yes |
| Preferred in modern networks | Usually not necessary | Generally preferred |

Route Refresh is supported by modern BGP implementations and is normally the preferred mechanism.

Traditional inbound soft reconfiguration remains useful when:

- The neighbor does not support Route Refresh.
- The administrator needs locally stored pre-policy received routes for troubleshooting.
- A specific operational requirement calls for retaining unmodified received routing information.

See [BGP Route Refresh](route-refresh.md) for Route Refresh capability negotiation and operation.

---


## BGP Soft Reconfiguration Backup

Cisco IOS XE supports **BGP Soft Reconfiguration Backup**, which allows traditional inbound soft reconfiguration to be used selectively when a BGP neighbor does not support Route Refresh.

Normally, configuring `neighbor ... soft-reconfiguration inbound` causes the router to store unmodified copies of received routes, even if the neighbor supports Route Refresh.

This can consume significant memory, especially when receiving large numbers of prefixes.

The **`bgp soft-reconfig-backup`** command changes this behavior so that unmodified received routes are stored only when Route Refresh is unavailable.

### Configuration

Two commands are required for the behavior demonstrated on Cisco IOS XE:

1. **`bgp soft-reconfig-backup`:** Enables the backup behavior globally for the BGP process.
2. **`neighbor ... soft-reconfiguration inbound`:** Identifies the neighbors for which inbound soft reconfiguration should be available.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp soft-reconfig-backup
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 soft-reconfiguration inbound
```

**Important:** Configuring `bgp soft-reconfig-backup` alone does not not enable storage of received routes. The neighbor-specific `soft-reconfiguration inbound` command is also required.

### Behavior Based on Route Refresh Capability

When both commands are configured, the router determines whether to store unmodified received routes based on the neighbor's negotiated Route Refresh capability.

| Neighbor Supports Route Refresh? | Store Unmodified Received Routes? | Inbound Soft Reset Mechanism |
|---|---|---|
| Yes | No | Route Refresh |
| No | Yes | Traditional inbound soft reconfiguration |

**If the neighbor supports Route Refresh:**

- The router does not maintain additional unmodified copies of received routes.
- When `clear ip bgp ... soft in` is executed, the router sends a ROUTE-REFRESH message to the neighbor.
- The neighbor readvertises its routes, which are processed using the current inbound policies.
- The BGP session remains Established.

**If the neighbor does not support Route Refresh:**

- The router stores unmodified copies of received routes, including routes rejected by inbound policies.
- When `clear ip bgp ... soft in` is executed, the router reprocesses the locally stored routes.
- No ROUTE-REFRESH message or new UPDATE messages are required from the neighbor.
- The BGP session remains Established.

This allows IOS XE to avoid unnecessary memory consumption while maintaining inbound soft reconfiguration support for legacy BGP neighbors.

### Behavior Without `bgp soft-reconfig-backup`

The following table summarizes the interaction between the two configuration commands.

| `bgp soft-reconfig-backup` | `neighbor ... soft-reconfiguration inbound` | Behavior |
|---|---|---|
| Not configured | Not configured | Use Route Refresh if supported; no traditional inbound route storage |
| Not configured | Configured | Store unmodified received routes, regardless of Route Refresh support |
| Configured | Not configured | Use Route Refresh if supported; traditional inbound route storage is not enabled for the neighbor |
| Configured | Configured | Store unmodified received routes **only if Route Refresh is not supported** |

**The global backup command does not independently enable traditional inbound soft reconfiguration for every neighbor.** It changes when the neighbor-specific feature stores received routes.

### Verification

Check whether the neighbor supports Route Refresh:

```text
R1# show ip bgp neighbors 10.0.12.2
```

When Route Refresh is negotiated, the output includes:

```text
Route refresh: advertised and received(new)
```

When the local router advertises Route Refresh but the neighbor does not, the output may show:

```text
Route refresh: advertised
```

To verify whether unmodified received routes are being stored:

```text
R1# show ip bgp neighbors 10.0.12.2 received-routes
```

**If Route Refresh is supported and backup mode is active:**

```text
% Inbound soft reconfiguration is in backup mode on 10.0.12.2
```

This indicates that the neighbor-specific soft reconfiguration feature is configured, but additional inbound route storage is not active because Route Refresh is available.

**If Route Refresh is not supported:**

The command displays the locally stored received routes, including routes rejected by inbound policies.

**If neighbor-specific soft reconfiguration is not enabled:**

```text
% Inbound soft reconfiguration not enabled on 10.0.12.2
```

### Key Points

- `neighbor ... soft-reconfiguration inbound` normally enables storage of unmodified received routes.
- `bgp soft-reconfig-backup` makes that storage conditional on the neighbor's Route Refresh capability.
- Both commands were required to activate backup behavior on the tested IOS XE image.
- When Route Refresh is supported, the router uses Route Refresh instead of storing additional inbound copies.
- When Route Refresh is unsupported, traditional inbound soft reconfiguration stores the received routes.
- In both cases, `clear ip bgp ... soft in` can reapply inbound policies without resetting the BGP session, provided the necessary routing information is available.

---

## Outbound Soft Reconfiguration

BGP also supports **outbound soft reconfiguration**, which allows updated outbound policies to be applied without resetting the BGP session.

Unlike traditional inbound soft reconfiguration, outbound soft reconfiguration does not require storing a separate unmodified copy of received routes.

The router already maintains the BGP routing information needed to evaluate routes for advertisement.

For example, suppose R1 modifies an outbound route map applied to R2.

```text
R1(config)# route-map SET-MED permit 10
R1(config-route-map)# set metric 100

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-MED out
```

After modifying the route map, perform an outbound soft reset:

```text
R1# clear ip bgp 10.0.12.2 soft out
```

BGP reevaluates advertisements toward R2 using the current outbound policy.

The BGP session remains Established.

### Inbound vs. Outbound Soft Reconfiguration

| Feature | Inbound | Outbound |
|---|---|---|
| Reapplies inbound policies | Yes | No |
| Reapplies outbound policies | No | Yes |
| Traditional operation may require stored received routes | Yes | No |
| Can use Route Refresh | Yes | Not required |
| Terminates the session | No | No |

**The `neighbor soft-reconfiguration inbound` command is not required for outbound soft resets.**

Outbound soft resets are covered in more detail in [BGP Session Reset and Clearing](session-reset.md).

---

## Verification

### Display Stored Received Routes

Use:

```text
R1# show ip bgp neighbors 10.0.12.2 received-routes
```

This displays the routes received from the neighbor before inbound filtering, when traditional inbound soft reconfiguration has retained the required information.

It can include:

- Routes permitted by inbound policies.
- Routes rejected by inbound policies.
- Routes whose attributes were modified by inbound policies, shown using their original received attributes.

**Important:** On conventional Cisco IOS/IOS XE BGP implementations, this command requires inbound soft reconfiguration to be enabled and the received information to be stored.

If inbound soft reconfiguration is not enabled, the command may return:

```text
% Inbound soft reconfiguration not enabled on 10.0.12.2
```

Supporting Route Refresh does not automatically mean that an unmodified copy of the received routes is stored locally.

### Display Accepted Routes

Use:

```text
R1# show ip bgp neighbors 10.0.12.2 routes
```

This displays routes learned from the neighbor that have passed the applicable inbound policies and are retained in the local BGP table.

### Comparing Received and Accepted Routes

Suppose R2 advertises:

```text
192.0.2.0/24
198.51.100.0/24
203.0.113.0/24
```

R1 permits only `192.0.2.0/24`.

Assuming traditional inbound soft reconfiguration is active:

| Command | Prefixes Displayed |
|---|---|
| `show ip bgp neighbors 10.0.12.2 received-routes` | All three received prefixes |
| `show ip bgp neighbors 10.0.12.2 routes` | `192.0.2.0/24` |
| `show ip bgp` | Accepted BGP paths, including `192.0.2.0/24` |

The last command displays the local BGP table as a whole, including routes from other neighbors and locally originated routes.

**`received-routes` and `routes` are not interchangeable.**

The distinction is particularly useful when troubleshooting inbound BGP filtering.

### Verify the Neighbor's Route Refresh Capability

Use:

```text
R1# show ip bgp neighbors 10.0.12.2
```

Look for output similar to:

```text
Neighbor capabilities:
  Route refresh: advertised and received(new)
```

This indicates that Route Refresh was successfully negotiated.

If the neighbor supports Route Refresh, traditional inbound soft reconfiguration is generally unnecessary unless locally stored received routes are specifically required.

### Verify the Configuration

```text
R1# show running-config | section router bgp
```

Check for:

```text
neighbor 10.0.12.2 soft-reconfiguration inbound
```

If applicable, also check for:

```text
bgp soft-reconfig-backup
```

### Verification Command Summary

| Command | Purpose |
|---|---|
| `show ip bgp neighbors <IP>` | Verify neighbor capabilities and session information |
| `show ip bgp neighbors <IP> received-routes` | Display stored received routes before inbound filtering |
| `show ip bgp neighbors <IP> routes` | Display accepted routes from the neighbor |
| `show ip bgp` | Display the local BGP table |
| `clear ip bgp <IP> soft in` | Reapply inbound policies without resetting the session |
| `clear ip bgp <IP> soft out` | Reapply outbound policies without resetting the session |

---

## Troubleshooting

| Symptom | Possible Cause | Solution |
|---|---|---|
| `received-routes` reports that inbound soft reconfiguration is not enabled | Unmodified inbound routes are not being stored | Configure inbound soft reconfiguration if stored routes are required |
| A previously rejected route does not appear after an inbound policy change | Existing received information has not been reprocessed | Perform an inbound soft reset |
| A previously rejected route is missing from stored information | Soft reconfiguration was enabled after the route was rejected | Obtain the routes again through Route Refresh or session reestablishment |
| Memory consumption increases significantly | Unmodified received routes are being stored | Use Route Refresh where possible or consider soft reconfiguration backup |
| Inbound soft reset cannot reprocess a rejected route | Neither stored information nor Route Refresh is available | Reestablish the BGP session if necessary |
| Outbound policy changes are not reflected in advertised routes | Advertisements have not been reevaluated | Perform an outbound soft reset |
| Neighbor supports Route Refresh but `received-routes` is unavailable | Route Refresh does not require local storage of unmodified routes | Use `routes` for accepted paths or explicitly enable inbound storage if required |

### Important Distinction: Filtering vs. RIB Installation

A route missing from the IP routing table is not necessarily being rejected by an inbound BGP policy.

For example:

1. R1 receives a route from R2.
2. The inbound policy permits the route.
3. The route enters the BGP table.
4. BGP selects the route as its best path.
5. An OSPF route to the same prefix wins RIB installation because of its lower administrative distance.

In this case, inbound soft reconfiguration is not the solution.

The problem is related to RIB installation, not inbound policy processing.

See [BGP Path Selection and RIB Installation](../path-selection/best-path/rib-installation.md).

---

## Key Points

- **Traditional inbound soft reconfiguration** stores unmodified received BGP routing information locally.
- The stored information can include routes rejected by inbound policies.
- Configure it using `neighbor <IP> soft-reconfiguration inbound`.
- Apply updated inbound policies using `clear ip bgp <IP> soft in`.
- A soft reset does not terminate the BGP session.
- Traditional inbound soft reconfiguration requires additional memory.
- **Route Refresh** allows inbound policy reprocessing without storing unmodified copies of received routes.
- Route Refresh is generally preferred when supported by both BGP neighbors.
- `bgp soft-reconfig-backup` provides selective backup behavior for peers that lack Route Refresh support.
- `show ip bgp neighbors <IP> received-routes` displays stored received information before inbound filtering.
- `show ip bgp neighbors <IP> routes` displays accepted routes from the neighbor.
- **Outbound soft reconfiguration does not require `neighbor soft-reconfiguration inbound`.**
- Soft reconfiguration changes how existing routing information is processed. It does not bypass BGP best-path selection or RIB installation rules.

---

## References

- [Cisco: BGP 4 Soft Configuration (IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-soft-config.html)
- [Cisco: Configuring a Basic BGP Network (IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-basic-net-0.html)
- [Cisco: BGP Command Reference, neighbor soft-reconfiguration](https://www.cisco.com/c/en/us/td/docs/ios/iproute_bgp/command/reference/irg_book/irg_bgp3.html)
- [Cisco: BGP Command Reference, clear ip bgp](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-c1.html)
- [RFC 2918: Route Refresh Capability for BGP-4](https://www.rfc-editor.org/rfc/rfc2918)
