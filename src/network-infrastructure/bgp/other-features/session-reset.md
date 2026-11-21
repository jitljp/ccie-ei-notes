# BGP Session Reset and Clearing

**BGP session resetting and clearing** allow administrators to reestablish BGP neighbor relationships, reprocess routing information, or apply updated routing policies.

Cisco IOS XE provides several methods for clearing BGP sessions and routing information.

The two primary categories are:

1. **Hard reset:** Terminates an established BGP session and causes the neighbors to establish a new session.
2. **Soft reset:** Reprocesses routing information without terminating the BGP session.

Soft resets can operate in two directions:

- **Inbound soft reset:** Reapplies policies to routes received from a neighbor.
- **Outbound soft reset:** Reapplies policies to routes advertised to a neighbor.

The appropriate operation depends on the configuration change and whether the existing BGP session must be reestablished.

**A hard reset disrupts the BGP session. A soft reset preserves the session while routing information is reevaluated.**

---

## Hard vs. Soft Reset

Consider two EBGP neighbors:

```text
       AS 65001                    AS 65002

          R1 ------------------------ R2
       10.0.12.1                  10.0.12.2

                      EBGP
```

R1 learns several prefixes from R2.

If the administrator changes an inbound prefix list on R1, the existing routes may need to be reevaluated.

This can normally be accomplished with a soft reset.

However, if the administrator needs to renegotiate BGP capabilities or reestablish the session itself, a hard reset may be necessary.

### Comparison

| Characteristic | Hard Reset | Soft Reset |
|---|---|---|
| Terminates BGP session | Yes | No |
| Terminates TCP connection | Yes | No |
| Reestablishes BGP session | Yes | No |
| Exchanges new OPEN messages | Yes | No |
| Renegotiates BGP capabilities | Yes | No |
| Reprocesses routing information | Yes | Yes |
| Can disrupt forwarding | Yes | Potentially, if policy changes alter routes |
| Typical use | Session-level changes | Routing policy changes |

Although soft resets do not terminate BGP sessions, they can still change routing and forwarding behavior.

For example, applying a new inbound filter could remove previously accepted routes from the BGP table.

**A soft reset is non-disruptive to the BGP session, but not necessarily to traffic forwarding.**

---

## BGP Hard Reset

A **BGP hard reset** terminates an existing BGP neighbor session and causes the routers to establish a new session.

On Cisco IOS XE, use:

```text
R1# clear ip bgp 10.0.12.2
```

The command initiates a hard reset of the specified neighbor.

### Hard Reset Operation

Consider the following sequence:

```text
          R1                           R2
           |                            |
           |       Established          |
           |<-------------------------->|
           |                            |
           |   clear ip bgp 10.0.12.2   |
           |                            |
           |-------- NOTIFICATION ----->|
           |                            |
           |     TCP Session Closed     |
           |                            |
           |    BGP Routes Invalidated  |
           |                            |
           |       TCP Connection       |
           |<-------------------------->|
           |                            |
           |<---------- OPEN ---------->|
           |<-------- KEEPALIVE ------->|
           |                            |
           |       Established          |
           |<-------------------------->|
           |                            |
           |<--------- UPDATEs -------->|
```

The process is:

1. R1 initiates a reset of the BGP session with R2.
2. R1 terminates the existing BGP session and TCP connection.
3. The BGP routes learned through R2 become unavailable for normal path selection, subject to features such as Graceful Restart.
4. R1 attempts to establish a new TCP connection.
5. The routers exchange BGP OPEN messages.
6. BGP capabilities and session parameters are negotiated again.
7. The routers exchange KEEPALIVE messages and reach the Established state.
8. Routing information is exchanged again.

### Syslog Messages

A hard reset can generate messages similar to:

```text
%BGP-3-NOTIFICATION_MANY: sent to 1 sessions 6/4
 (Administrative Reset) for all peers

%BGP-5-ADJCHANGE: neighbor 10.0.12.2 Down User reset

%BGP_SESSION-5-ADJCHANGE: neighbor 10.0.12.2
 IPv4 Unicast topology base removed from session User reset

%BGP-5-ADJCHANGE: neighbor 10.0.12.2 Up
```

The NOTIFICATION error code `6` identifies a **Cease** notification.

Subcode `4` identifies an **Administrative Reset**.

The exact messages depend on the Cisco IOS XE version and the type of reset.

### Effects on Routing

When a BGP session goes down, routes learned exclusively through that neighbor normally become unavailable.

This can cause:

- BGP best-path recalculation.
- Selection of alternative BGP paths.
- Changes to the IP routing table.
- Changes to the forwarding table.
- BGP UPDATE messages to other neighbors.
- Temporary loss of connectivity.

If another valid BGP path is available, the router may select it.

Otherwise, the destination can become unreachable until the BGP session is reestablished and its routes are relearned.

**Important:** BGP Graceful Restart can modify route retention behavior during certain session failures or restarts. Do not assume every hard reset immediately removes every route from the forwarding table.

---

## When Is a Hard Reset Required?

A hard reset may be required when changes affect the BGP session itself rather than only its routing policies.

Examples include:

- Changes requiring BGP OPEN message capability renegotiation.
- Changes to session parameters that cannot take effect on an existing connection.
- Troubleshooting a BGP session that needs to be reestablished.
- Testing BGP session establishment and failure recovery.

### Capability Negotiation

BGP capabilities are normally exchanged in OPEN messages during session establishment.

Examples include:

- Route Refresh.
- Enhanced Route Refresh.
- Multiprotocol Extensions.
- 4-byte ASN support.
- ADD-PATH.

A soft reset does not exchange new OPEN messages.

Therefore, changes to capability advertisement settings generally require session reestablishment.

For example, on a Cisco image supporting the hidden command:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 dont-capability-negotiate
```

This suppresses BGP capability negotiation with the specified neighbor.

The existing session cannot renegotiate capabilities through Route Refresh.

A hard reset can be used to establish a new session:

```text
R1# clear ip bgp 10.0.12.2
```

**Important:** Some session-level configuration changes automatically reset the affected BGP session. A manual hard reset is not always necessary.

For example, enabling inbound soft reconfiguration caused an automatic session reset on the tested Cisco IOS XE image.

---

## BGP Soft Reset

A **BGP soft reset** reprocesses routing information without terminating the established BGP session.

Soft resets are commonly used after modifying:

- Inbound or outbound prefix lists.
- BGP route maps.
- AS-path filter lists.
- Community-based filtering policies.
- BGP attribute manipulation policies.

Soft resets are classified by direction.

| Operation | Purpose |
|---|---|
| Inbound soft reset | Reapply inbound routing policies |
| Outbound soft reset | Reapply outbound routing policies |

The direction is determined from the perspective of the router executing the command.

For example, on R1:

- `soft in` affects routes **received by R1 from R2**.
- `soft out` affects routes **advertised by R1 to R2**.

---

## Inbound Soft Reset

An **inbound soft reset** causes the router to reapply inbound routing policies to routes received from a neighbor.

Configure:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

This does not terminate the BGP session.

However, the mechanism used to reprocess routes depends on whether traditional inbound soft reconfiguration or Route Refresh is available.

### Method 1: Traditional Inbound Soft Reconfiguration

If R1 stores unmodified received routes, it can reprocess them locally.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 soft-reconfiguration inbound
```

When R1 executes:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

BGP reprocesses the stored routing information using the current inbound policies.

The process is:

```text
        Stored Received Routes
                  |
                  v
         Inbound Soft Reset
                  |
                  v
        Current Inbound Policy
                  |
                  v
          Accepted Routes
                  |
                  v
           Best-Path Selection
```

No ROUTE-REFRESH request is required when the necessary received information is available locally.

The BGP session remains Established.

### Method 2: Route Refresh

If traditional inbound route storage is not configured and the neighbor supports Route Refresh, the router can request the routing information again.

```text
          R1                           R2
           |                            |
           |------- ROUTE-REFRESH ----->|
           |                            |
           |<--------- UPDATE ----------|
           |<--------- UPDATE ----------|
           |                            |
      Apply Inbound Policy
           |
      Best-Path Selection
```

The process is:

1. R1 sends a ROUTE-REFRESH message to R2.
2. R2 readvertises its eligible routing information.
3. R1 processes the received routes using the current inbound policies.
4. The BGP session remains Established.

**Route Refresh does not require `neighbor ... soft-reconfiguration inbound`.**

It is generally the preferred mechanism because it avoids storing additional copies of received routes.

### Method 3: Neither Mechanism Available

Suppose:

- Traditional inbound soft reconfiguration is not configured.
- The neighbor does not support Route Refresh.

In this situation, R1 does not have a mechanism for recovering previously rejected routes through an inbound soft reset.

Executing:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

Cannot reconstruct routing information that is neither stored locally nor available through Route Refresh.

A hard reset may be necessary to obtain the neighbor's routing information again.

### Inbound Soft Reset Behavior Summary

| Stored Inbound Routes? | Route Refresh Supported? | Normal Mechanism |
|---|---|---|
| No | Yes | Route Refresh |
| Yes | No | Traditional inbound soft reconfiguration |
| Yes | Yes | Traditional soft reconfiguration, unless backup mode selects Route Refresh |
| No | No | Cannot fully reprocess previously rejected routes using these mechanisms |

See [BGP Soft Reconfiguration](soft-reconfiguration.md) and [BGP Route Refresh](route-refresh.md).

---

## Inbound Soft Reset Configuration Example

Consider two EBGP neighbors:

```text
       AS 65001                    AS 65002

          R1 ------------------------ R2
       10.0.12.1                  10.0.12.2
                      EBGP
```

R2 advertises:

```text
2.2.2.2/32
22.22.22.22/32
```

R1 initially permits only `2.2.2.2/32`.

### Initial Configuration

```text
R1(config)# ip prefix-list FROM-R2 permit 2.2.2.2/32

R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 prefix-list FROM-R2 in
```

The accepted routes can be displayed using:

```text
R1# show ip bgp neighbors 10.0.12.2 routes
```

Initially:

```text
2.2.2.2/32
```

The second prefix is rejected.

### Modify the Policy

Permit the second prefix:

```text
R1(config)# ip prefix-list FROM-R2 permit 22.22.22.22/32
```

Now perform an inbound soft reset:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

Assume R2 supports Route Refresh.

R1 requests the routing information again, and R2 readvertises its eligible routes.

### Verification

```text
R1# show ip bgp neighbors 10.0.12.2 routes
```

Both prefixes should now be accepted:

```text
2.2.2.2/32
22.22.22.22/32
```

Check the BGP session:

```text
R1# show ip bgp summary
```

The session should remain Established throughout the operation.

**The inbound soft reset applies the updated inbound policy without reestablishing the BGP session.**

---

## Outbound Soft Reset

An **outbound soft reset** causes a router to reevaluate the routing information advertised to a BGP neighbor.

Use:

```text
R1# clear ip bgp 10.0.12.2 soft out
```

This operation is useful after changing outbound routing policies.

Unlike traditional inbound soft reconfiguration, outbound soft resets do not require additional copies of received routes.

The router already maintains the BGP routes needed to evaluate outbound advertisements.

### Outbound Soft Reset Operation

Consider:

```text
          R1                           R2
           |                            |
    BGP Table                           |
           |                            |
    Outbound Soft Reset                 |
           |                            |
    Apply Current Outbound Policy       |
           |                            |
           |----------- UPDATE -------->|
           |----------- UPDATE -------->|
           |                            |
```

When an outbound soft reset is performed:

1. R1 reevaluates the routes eligible for advertisement to R2.
2. The current outbound routing policies are applied.
3. R1 generates the necessary UPDATE or withdrawal messages.
4. R2 processes the received information.
5. The BGP session remains Established.

**Outbound soft reset does not use Route Refresh to request routes from the neighbor.**

Instead, the local router reevaluates its own outbound advertisements.

---

## Outbound Soft Reset Configuration Example

Suppose R1 advertises `192.0.2.0/24` to R2.

The administrator wants to modify the MED advertised for the prefix.

### Initial Configuration

```text
R1(config)# route-map SET-MED permit 10
R1(config-route-map)# set metric 100

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-MED out
```

Initially, R1 advertises the route with MED 100.

### Modify the Policy

Change the MED to 200:

```text
R1(config)# route-map SET-MED permit 10
R1(config-route-map)# set metric 200
```

Perform an outbound soft reset:

```text
R1# clear ip bgp 10.0.12.2 soft out
```

R1 reevaluates its outbound routing information using the modified route map.

The prefix can now be advertised with MED 200.

### Verification

On R2:

```text
R2# show ip bgp 192.0.2.0
```

Verify the updated MED in the detailed BGP path information.

The BGP session remains Established.

### Withdrawing Routes Using an Outbound Soft Reset

An outbound soft reset can also be useful after modifying outbound filters.

For example, R1 originally advertises:

```text
192.0.2.0/24
198.51.100.0/24
```

The administrator configures a prefix list that only permits `192.0.2.0/24`.

```text
R1(config)# ip prefix-list TO-R2 permit 192.0.2.0/24

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 prefix-list TO-R2 out
```

Perform an outbound soft reset:

```text
R1# clear ip bgp 10.0.12.2 soft out
```

R1 reevaluates the advertisements.

If `198.51.100.0/24` was previously advertised, R1 sends a withdrawal for that prefix.

The route can remain in R1's local BGP table.

**An outbound soft reset changes what the neighbor receives. It does not directly remove routes from the local BGP table.**

---

## Inbound vs. Outbound Soft Reset

| Characteristic | Inbound Soft Reset | Outbound Soft Reset |
|---|---|---|
| Command | `soft in` | `soft out` |
| Reapplies policies | Inbound | Outbound |
| Processes received routes | Yes | No |
| Reevaluates advertised routes | No | Yes |
| May use Route Refresh | Yes | No |
| May use stored unmodified routes | Yes | Not required |
| Can change local best-path selection | Yes | Not directly |
| Can change routes advertised to neighbor | Indirectly, if selected paths change | Yes |
| Resets BGP session | No | No |

The most important distinction is the direction of policy processing.

**Inbound soft resets affect the local router's interpretation of received routes. Outbound soft resets affect the routing information sent to the neighbor.**

---

## Clearing Specific Neighbors and Groups

The `clear ip bgp` command supports several ways to select the affected neighbors.

### Clear a Specific Neighbor

Hard reset:

```text
R1# clear ip bgp 10.0.12.2
```

Inbound soft reset:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

Outbound soft reset:

```text
R1# clear ip bgp 10.0.12.2 soft out
```

This is generally the preferred scope when troubleshooting a single BGP neighbor.

### Clear All BGP Neighbors

Hard reset all BGP sessions:

```text
R1# clear ip bgp *
```

Inbound soft reset for all neighbors:

```text
R1# clear ip bgp * soft in
```

Outbound soft reset for all neighbors:

```text
R1# clear ip bgp * soft out
```

**Important:** `clear ip bgp *` without the `soft` keyword can disrupt every BGP session on the router.

Use it cautiously, especially in production environments.

### Clear Neighbors in a Specific AS

Cisco IOS XE supports selecting neighbors based on their remote ASN.

For example:

```text
R1# clear ip bgp 65002
```

This performs a hard reset of applicable BGP neighbors belonging to AS 65002.

To perform an inbound soft reset:

```text
R1# clear ip bgp 65002 soft in
```

To perform an outbound soft reset:

```text
R1# clear ip bgp 65002 soft out
```

The ASN identifies the remote AS whose neighbors should be cleared.

### Clear All EBGP Neighbors

On supported Cisco IOS XE platforms:

```text
R1# clear ip bgp external
```

This performs a hard reset of EBGP neighbors.

Soft reset variants are also supported:

```text
R1# clear ip bgp external soft in
R1# clear ip bgp external soft out
```

This is useful when an operation must apply to external BGP peers rather than internal BGP peers.

### Clear a Peer Group

Cisco IOS XE also supports clearing BGP neighbors associated with a peer group.

For example, if a peer group named `EBGP-PEERS` exists:

```text
R1# clear ip bgp EBGP-PEERS soft out
```

This reevaluates outbound advertisements for the peer group's members on platforms supporting this syntax.

Peer-group selection can simplify operational changes involving multiple neighbors that share a common policy.

---

## Address-Family-Specific Clearing

BGP can carry multiple address families over a BGP session.

Examples include:

- IPv4 unicast.
- IPv6 unicast.
- VPNv4 unicast.
- VPNv6 unicast.

Cisco IOS XE provides address-family-specific clearing commands.

### IPv4 Unicast

```text
R1# clear ip bgp ipv4 unicast 10.0.12.2 soft in
```

This initiates an inbound soft reset for IPv4 unicast routing information associated with the specified neighbor.

### IPv6 Unicast

```text
R1# clear bgp ipv6 unicast 2001:db8:12::2 soft in
```

This initiates an inbound soft reset for IPv6 unicast.

### VPNv4 Unicast

```text
R1# clear ip bgp vpnv4 unicast soft in
```

This initiates an inbound soft reset for the applicable VPNv4 unicast neighbors.

### Why Address-Family Scope Matters

Consider a BGP session carrying both IPv4 and IPv6 unicast routing information.

If only the IPv4 inbound policy changes, the administrator can initiate an IPv4-specific soft reset.

There is no need to request IPv6 routing information again.

**Important:** A hard reset terminates the underlying BGP TCP session. A hard reset targeting one address family can therefore affect other negotiated address families using that session.

Address-family-specific soft resets are preferable when only one address family's routing information must be reevaluated.

Exact command syntax and supported address families depend on the IOS XE platform and software release.

---

## Dynamic Inbound Soft Reset

Cisco IOS XE also supports the following form of inbound clearing:

```text
R1# clear ip bgp 10.0.12.2 in
```

When Route Refresh is supported, the `in` keyword can initiate a dynamic inbound soft reset.

The router sends a ROUTE-REFRESH request to the neighbor and processes the readvertised routes.

This is similar to:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

Both commands can reapply inbound routing policies without terminating the BGP session when the necessary mechanisms are available.

However, the exact behavior of the different forms depends on IOS XE support and whether traditional inbound route storage is configured.

For consistency, this book generally uses:

```text
clear ip bgp <NEIGHBOR> soft in
```

This explicitly identifies the operation as an inbound soft reset.

---

## BGP Soft Reconfiguration Backup

Cisco IOS XE supports `bgp soft-reconfig-backup`, which allows traditional inbound soft reconfiguration to be used selectively when Route Refresh is unavailable.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp soft-reconfig-backup
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 soft-reconfiguration inbound
```

When both commands are configured, the router can select the inbound soft-reset mechanism based on the neighbor's Route Refresh capability.

| Neighbor Supports Route Refresh? | Mechanism Used by `soft in` |
|---|---|
| Yes | Route Refresh |
| No | Traditional inbound soft reconfiguration |

When Route Refresh is available, the router does not need to maintain additional unmodified received-route copies for the backup feature.

When Route Refresh is unavailable, it stores the received routes for local reprocessing.

### Configuration Interaction

| Configuration | Behavior |
|---|---|
| Neither command configured | Use Route Refresh if supported; no traditional inbound route storage |
| `soft-reconfiguration inbound` only | Store unmodified received routes |
| `bgp soft-reconfig-backup` only | Does not activate traditional storage for the neighbor on the tested IOS XE image |
| Both commands configured | Store unmodified received routes only when Route Refresh is unavailable |

### Verification

```text
R1# show ip bgp neighbors 10.0.12.2 received-routes
```

When Route Refresh is supported and backup mode is active, IOS XE can display:

```text
% Inbound soft reconfiguration is in backup mode on 10.0.12.2
```

When Route Refresh is unsupported and traditional storage is active, the command displays the stored received routes.

When neighbor-specific soft reconfiguration is not configured, IOS XE can display:

```text
% Inbound soft reconfiguration not enabled on 10.0.12.2
```

These behaviors were confirmed in a Cisco IOS XE lab.

See [BGP Soft Reconfiguration](soft-reconfiguration.md) for further details.

---

## When Should BGP Sessions Be Cleared?

Not every BGP configuration change requires a manual clear.

Many changes are processed dynamically by BGP, while others may require routing information to be reevaluated or the session to be reestablished.

The following table provides general guidance.

| Configuration Change | Typical Action |
|---|---|
| Inbound prefix list modified | Inbound soft reset |
| Inbound route map modified | Inbound soft reset |
| Inbound AS-path filter modified | Inbound soft reset |
| Outbound prefix list modified | Outbound soft reset |
| Outbound route map modified | Outbound soft reset |
| Locally originated route added using `network` | Normally processed automatically |
| Aggregate configuration changed | Normally processed automatically |
| Best-path selection configuration changed | Usually triggers local reevaluation |
| BGP capability negotiation settings changed | Session reestablishment |
| Session establishment parameters changed | Session reestablishment may be required |
| BGP neighbor stuck or malfunctioning | Hard reset may be appropriate |

Some policy changes trigger automatic reevaluation on modern IOS XE releases. A soft reset is useful when existing routing information must explicitly be reprocessed.

**Do not automatically perform a hard reset after every BGP configuration change.**

First determine whether the change affects routing information or session establishment.

---

## Verification

### Verify Neighbor State

```text
R1# show ip bgp summary
```

Check:

- Neighbor IP address.
- Remote ASN.
- BGP session state.
- Received prefix count.

For an Established neighbor, the `State/PfxRcd` column normally displays a prefix count rather than a BGP FSM state.

### Verify Neighbor Details

```text
R1# show ip bgp neighbors 10.0.12.2
```

Useful information includes:

- BGP session state.
- Uptime.
- Last reset information.
- Negotiated capabilities.
- Keepalive and Hold Timers.
- UPDATE message counters.
- Route Refresh capability.

After a hard reset, the session uptime normally restarts.

After a soft reset, the existing session uptime continues.

### Verify Accepted Routes

```text
R1# show ip bgp neighbors 10.0.12.2 routes
```

This displays routes accepted from the neighbor.

Use it after an inbound soft reset to confirm that updated inbound policies have been applied.

### Verify Stored Received Routes

```text
R1# show ip bgp neighbors 10.0.12.2 received-routes
```

This displays stored received routing information when traditional inbound soft reconfiguration is active.

It can include routes rejected by the inbound policy.

### Verify Advertised Routes

```text
R1# show ip bgp neighbors 10.0.12.2 advertised-routes
```

This displays routes advertised to the specified neighbor.

Use it after an outbound soft reset to verify the advertised prefix set.

For attribute manipulation, inspect the route on the receiving neighbor because local advertised-route summary output may not expose all transmitted attribute changes.

### Verify Route Refresh Support

```text
R1# show ip bgp neighbors 10.0.12.2
```

Look for:

```text
Neighbor capabilities:
  Route refresh: advertised and received(new)
```

This indicates that standard Route Refresh support has been exchanged.

If the output shows only:

```text
Route refresh: advertised
```

The remote neighbor has not advertised standard Route Refresh support.

### Verify with Syslog

A hard reset can generate messages such as:

```text
%BGP-5-ADJCHANGE: neighbor 10.0.12.2 Down User reset
%BGP-5-ADJCHANGE: neighbor 10.0.12.2 Up
```

A normal inbound or outbound soft reset should not generate corresponding BGP neighbor Down/Up events.

This provides a useful way to distinguish session resets from routing information refreshes.

---

## Troubleshooting

| Symptom | Possible Cause | Solution |
|---|---|---|
| Inbound policy changes are not reflected | Existing routes have not been reprocessed | Perform an inbound soft reset |
| Outbound policy changes are not reflected | Outbound advertisements have not been reevaluated | Perform an outbound soft reset |
| Previously rejected routes remain unavailable | No stored routes and no Route Refresh capability | Enable appropriate soft reconfiguration or reestablish the session |
| `received-routes` command fails | Traditional inbound route storage is not enabled | Check soft reconfiguration settings |
| Soft reset does not renegotiate capabilities | BGP OPEN messages are not exchanged during soft resets | Reestablish the BGP session |
| Hard reset causes route loss | Routes learned through the neighbor become unavailable | Verify alternative paths and consider convergence behavior |
| BGP session remains Established after `soft in` | Expected behavior | No action needed |
| BGP session remains Established after `soft out` | Expected behavior | No action needed |
| Routes still fail to enter the IP routing table | Best-path selection, NEXT_HOP reachability, or RIB installation issue | Inspect the BGP table and routing table |

### Troubleshooting Sequence

When a BGP policy change does not produce the expected result:

1. Verify that the BGP session is Established.
2. Confirm the inbound or outbound policy is configured in the intended direction.
3. Verify the policy matches the expected routes.
4. Determine whether the issue concerns received or advertised routes.
5. Perform the appropriate inbound or outbound soft reset if needed.
6. Verify the resulting BGP paths and attributes.
7. Check best-path selection and RIB installation if the expected route is still not being used.

A hard reset should generally be reserved for session-level problems or changes that require reestablishment.

---

## Command Summary

| Command | Purpose | Hard or Soft? |
|---|---|---|
| `clear ip bgp <IP>` | Reset a specific neighbor | Hard |
| `clear ip bgp *` | Reset all BGP neighbors | Hard |
| `clear ip bgp <ASN>` | Reset neighbors in a specific remote AS | Hard |
| `clear ip bgp external` | Reset EBGP neighbors | Hard |
| `clear ip bgp <IP> soft in` | Reapply inbound routing policies | Soft |
| `clear ip bgp <IP> soft out` | Reapply outbound routing policies | Soft |
| `clear ip bgp * soft in` | Inbound soft reset for all applicable neighbors | Soft |
| `clear ip bgp * soft out` | Outbound soft reset for all applicable neighbors | Soft |
| `clear ip bgp <ASN> soft in` | Inbound soft reset for neighbors in a remote AS | Soft |
| `clear ip bgp <ASN> soft out` | Outbound soft reset for neighbors in a remote AS | Soft |
| `clear ip bgp external soft in` | Inbound soft reset for EBGP neighbors | Soft |
| `clear ip bgp external soft out` | Outbound soft reset for EBGP neighbors | Soft |
| `clear ip bgp <IP> in` | Initiate dynamic inbound refresh when supported | Soft |
| `clear ip bgp ipv4 unicast <IP> soft in` | IPv4 unicast inbound soft reset | Soft |
| `clear bgp ipv6 unicast <IP> soft in` | IPv6 unicast inbound soft reset | Soft |

The available command forms can vary by platform, IOS XE release, and address family.

---

## Key Points

- A **hard reset** terminates the BGP session and TCP connection.
- A **soft reset** reprocesses routing information without terminating the BGP session.
- `clear ip bgp <IP>` performs a hard reset.
- `clear ip bgp <IP> soft in` performs an inbound soft reset.
- `clear ip bgp <IP> soft out` performs an outbound soft reset.
- Inbound soft resets use either locally stored received routes or Route Refresh.
- Route Refresh is generally preferred because it avoids maintaining additional unmodified copies of received routes.
- Outbound soft resets do not require traditional inbound soft reconfiguration or Route Refresh.
- `bgp soft-reconfig-backup` allows traditional inbound storage to be used selectively when Route Refresh is unavailable.
- `clear ip bgp *` affects all BGP neighbors and can be disruptive.
- BGP capabilities are negotiated through OPEN messages, so a soft reset does not renegotiate capabilities.
- Some configuration changes automatically trigger BGP reevaluation or session resets.
- **A soft reset preserves the BGP session but can still change routing and forwarding behavior.**
- Use the narrowest appropriate clearing operation to minimize unnecessary routing disruption.

---

## References

- [Cisco: BGP 4 Soft Configuration (IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-soft-config.html)
- [Cisco: Configuring a Basic BGP Network (IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-basic-net-0.html)
- [Cisco: BGP Command Reference, BGP Commands C through I](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-c1.html)
- [Cisco: BGP Command Reference, Neighbor Commands](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [RFC 2918: Route Refresh Capability for BGP-4](https://www.rfc-editor.org/rfc/rfc2918)
- [RFC 4486: Subcodes for BGP Cease Notification Message](https://www.rfc-editor.org/rfc/rfc4486)
