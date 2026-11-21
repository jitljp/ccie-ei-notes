# BGP Maximum Prefix

**BGP Maximum Prefix** allows a router to limit the number of BGP prefixes accepted from a neighbor.

The feature protects routers against unexpected increases in routing information, which can consume memory and CPU resources.

For example, an ISP might accidentally advertise its entire Internet routing table to a customer router that normally receives only a few prefixes.

Without protection, the customer router might attempt to process hundreds of thousands of unexpected routes.

Cisco IOS XE supports the `neighbor maximum-prefix` command to configure a limit for individual BGP neighbors.

When the configured limit is exceeded, BGP can:

1. **Terminate the BGP session:** The default behavior.
2. **Generate warnings only:** Allow the session to remain Established.
3. **Automatically restart the session:** Reestablish a session terminated for exceeding the limit after a configured interval.
4. **Discard excess prefixes:** Reject additional prefixes without terminating the session, on supported IOS XE releases.

The appropriate behavior depends on operational requirements and the capabilities of the IOS XE release.

---

## Maximum Prefix Operation

Consider the following topology:

```text
       AS 65001                     AS 65002

          R1 ------------------------- R2
       10.0.12.1                   10.0.12.2
                       EBGP

       Maximum: 3              Advertises:
                               192.0.2.0/24
                               198.51.100.0/24
                               203.0.113.0/24
                               10.10.10.0/24
```

R1 is configured to accept a maximum of three prefixes from R2.

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 maximum-prefix 3
```

As R2 advertises its routes, R1 tracks the number of accepted prefixes associated with the neighbor.

| Prefix Count | Result |
|---|---|
| 1 | Accepted |
| 2 | Accepted |
| 3 | Accepted, at the maximum |
| 4 | Maximum exceeded, session terminated |

By default, when the limit is exceeded:

1. R1 detects that the prefix count exceeds the configured maximum.
2. R1 sends a BGP NOTIFICATION message to R2.
3. The BGP session is terminated.
4. Routes learned through the session become unavailable for normal path selection.
5. The session remains disabled until it is manually cleared or the configuration is changed appropriately.

The automatic restart option can modify the final step.

**Important:** The default maximum-prefix action is to terminate the BGP session, not to silently discard the excess routes.

---

## Basic Configuration

Configure a maximum prefix limit under the appropriate BGP address family.

### Syntax

```text
neighbor <IP> maximum-prefix <MAXIMUM> [<THRESHOLD>] [restart <MINUTES> | warning-only | discard-extra]
```

The available options depend on the IOS XE version.

| Parameter | Purpose |
|---|---|
| `<IP>` | Identifies the BGP neighbor |
| `<MAXIMUM>` | Maximum number of accepted prefixes |
| `<THRESHOLD>` | Percentage of the maximum at which warnings begin |
| `restart <MINUTES>` | Automatically attempts to restart the session after the specified interval |
| `warning-only` | Generates warnings without terminating the session |
| `discard-extra` | Discards prefixes exceeding the limit without terminating the session |

### Example

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 maximum-prefix 1000
```

This configures a maximum of 1,000 prefixes from R2.

When the prefix count exceeds 1,000, R1 terminates the BGP session by default.

### Default Behavior

| Parameter | Default |
|---|---|
| Maximum-prefix limit | Not configured |
| Warning threshold | 75% of the configured maximum |
| Action when maximum is exceeded | Terminate BGP session |
| Automatic restart | Disabled |

There is no fixed default maximum-prefix limit on ordinary IOS XE BGP neighbors when the feature is not configured.

The default warning threshold of 75% applies when a maximum-prefix limit has been configured.

### Removing the Configuration

```text
R1(config-router-af)# no neighbor 10.0.12.2 maximum-prefix
```

This removes the configured prefix limit for the neighbor.

---

## Maximum Prefix Warning Threshold

The optional **warning threshold** determines when BGP generates a warning that the received-prefix count is approaching the configured maximum.

The threshold is expressed as a percentage of the maximum-prefix limit.

The default threshold is **75%**.

### Example

```text
R1(config-router-af)# neighbor 10.0.12.2 maximum-prefix 1000 80
```

This configures:

- Maximum prefixes: 1,000
- Warning threshold: 80%
- Default action: Terminate the session if the maximum is exceeded

The warning threshold corresponds to:

```text
1000 * 0.80 = 800 prefixes
```

The router begins generating threshold warnings when the count exceeds the configured warning percentage.

| Received Prefixes | Result |
|---|---|
| 500 | Normal |
| 800 | At warning threshold |
| 801 | Warning threshold exceeded |
| 999 | Warning condition |
| 1000 | At maximum |
| 1001 | Session terminated |

The exact log messages generated at the boundary can depend on the IOS XE implementation.

### Default Warning Threshold

If no threshold is specified:

```text
R1(config-router-af)# neighbor 10.0.12.2 maximum-prefix 1000
```

The default warning percentage is 75%.

The warning threshold corresponds to 750 prefixes.

**Important:** The warning threshold does not determine when BGP terminates the session. Session termination occurs when the configured maximum is exceeded.

---

## Default Action: Terminate the BGP Session

By default, exceeding the maximum-prefix limit causes the BGP session to be terminated.

For example:

```text
R1(config-router-af)# neighbor 10.0.12.2 maximum-prefix 1000 80
```

If R1 receives more than 1,000 counted prefixes from R2, it terminates the BGP session.

### BGP NOTIFICATION

The router sends a BGP NOTIFICATION message indicating that the maximum prefix count has been exceeded.

The notification uses:

| Field | Value |
|---|---|
| BGP Message | NOTIFICATION |
| Error Code | 6 (Cease) |
| Error Subcode | 1 (Maximum Number of Prefixes Reached) |

Cisco IOS XE can generate syslog messages similar to:

```text
%BGP-3-MAXPFXEXCEED: Number of prefixes received
from 10.0.12.2 (afi 0): 1001 exceeds limit 1000

%BGP-3-NOTIFICATION: sent to neighbor 10.0.12.2
6/1 (Maximum Number of Prefixes Reached)

%BGP-5-ADJCHANGE: neighbor 10.0.12.2
Down Peer over prefix limit
```

The exact log formatting varies by IOS XE release.

### BGP Session Recovery

Without the `restart` option, the session does not automatically reestablish after being disabled by the maximum-prefix limit.

The administrator must resolve the underlying issue and clear the session.

For example:

```text
R1# clear ip bgp 10.0.12.2
```

However, if R2 is still advertising more prefixes than permitted, the session may immediately exceed the limit again.

The administrator should first:

1. Investigate why the prefix count increased.
2. Correct the remote advertisement policy if necessary.
3. Adjust the maximum-prefix limit if the increased route count is legitimate.
4. Clear the affected BGP session.

**A maximum-prefix shutdown is different from an ordinary BGP Hold Timer expiration.** The session is intentionally disabled because a configured routing limit was exceeded.

---

## Automatic Session Restart

The **`restart`** keyword allows a BGP session terminated by the maximum-prefix feature to be reestablished automatically after a configured interval.

### Configuration

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 maximum-prefix 1000 80 restart 5
```

This configures:

- Maximum prefixes: 1,000
- Warning threshold: 80%
- Automatic restart interval: 5 minutes

### Operation

Suppose R2 advertises 1,001 prefixes.

1. R1 detects that the maximum-prefix limit has been exceeded.
2. R1 sends a BGP NOTIFICATION and terminates the session.
3. R1 waits for the configured restart interval.
4. R1 attempts to reestablish the BGP session.
5. If R2 now advertises no more than 1,000 accepted prefixes, the session can remain Established.
6. If the limit is still exceeded, R1 terminates the session again.

The restart interval is specified in **minutes**, with a documented range of 1 to 65,535.

### Important Consideration

Automatic restart does not resolve the underlying problem.

For example, if R2 continues advertising 1,500 prefixes while the maximum remains 1,000, R1 may repeatedly establish and terminate the session.

This can cause unnecessary BGP session flapping and routing instability.

**The `restart` keyword provides automatic recovery attempts, not automatic correction of excessive route advertisements.**

---

## Warning-Only Mode

The **`warning-only`** keyword prevents maximum-prefix violations from terminating the BGP session.

Instead, the router generates warning messages while continuing to accept routing information.

### Configuration

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 maximum-prefix 1000 80 warning-only
```

This configures:

- Maximum prefix value: 1,000
- Warning threshold: 80%
- Action when maximum is exceeded: Generate warnings only

### Operation

| Received Prefixes | Result |
|---|---|
| 500 | Normal |
| 801 | Warning threshold exceeded |
| 1000 | At maximum |
| 1001 | Warning generated; session remains Established |
| 1500 | Session remains Established |

With `warning-only`, the configured maximum does not function as an enforced limit.

BGP continues processing routes that would otherwise exceed the maximum.

### Use Cases

Warning-only mode can be useful when:

- Monitoring the number of routes received from an ISP.
- Establishing baseline route counts before enforcing a limit.
- Testing an appropriate maximum-prefix threshold.
- Avoiding an unexpected session shutdown during initial deployment.

However, it provides less protection than an enforced maximum-prefix limit.

**Important:** `warning-only` does not discard excess routes or protect against memory exhaustion caused by excessive accepted routing information.

It is primarily a monitoring mechanism.

---

## Discarding Excess Prefixes

Starting with **Cisco IOS XE 17.13.1a**, Cisco introduced the **`discard-extra`** option.

This allows BGP to discard prefixes exceeding the configured maximum without terminating the BGP session.

### Configuration

On supported platforms:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 maximum-prefix 1000 discard-extra
```

This configures a maximum of 1,000 prefixes while keeping the BGP session Established if additional prefixes arrive.

### Operation

Suppose R2 advertises 1,500 prefixes.

Without `discard-extra`:

```text
Prefix count exceeds 1000
          |
          v
   BGP session reset
```

With `discard-extra`:

```text
Prefix count exceeds 1000
          |
          v
  Excess prefixes discarded
          |
          v
 BGP session remains Established
```

The feature provides an alternative to terminating the BGP session.

### Important Restrictions

Cisco documents several limitations of `discard-extra`:

- The router does not provide a mechanism to select exactly which excess prefixes are discarded.
- Different routers may retain different subsets of advertised prefixes.
- Dropping prefixes can produce inconsistent routing information across the network.
- Inconsistent routing information can contribute to routing loops.
- Discarded routes are not necessarily recovered automatically when the prefix count subsequently decreases.
- `discard-extra` cannot coexist with traditional inbound soft reconfiguration on the affected neighbor.
- Maximum-prefix action options are mutually exclusive for a given neighbor.

For example, suppose R1 has a maximum of three prefixes and R2 advertises five.

R1 might retain three routes and discard the others.

If R2 subsequently withdraws one of the retained routes, R1 might have only two retained routes even though previously discarded routes remain available on R2.

A soft reset can be used to request eligible routing information again:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

This requires an applicable route refresh or soft reconfiguration mechanism.

**Important:** Because the router cannot precisely control which excess routes are discarded, this feature can create unexpected routing behavior. Use it carefully.

### Comparing Maximum-Prefix Actions

| Configuration | Session Terminated? | Excess Prefixes Accepted? |
|---|---|---|
| Default | Yes | No, session terminated |
| `restart` | Yes, followed by restart attempts | No, while session is down |
| `warning-only` | No | Yes |
| `discard-extra` | No | No, excess prefixes discarded |

The `discard-extra` option is available only on supported IOS XE releases and platforms.

---

## Prefix Counting and Inbound Filtering

An important distinction is that **maximum-prefix does not simply count routes installed in the IP routing table**.

BGP maintains routing information independently of the global routing table.

For example, a router may receive several BGP paths but install only a subset in the IP routing table because of:

- BGP best-path selection.
- Next-hop reachability.
- Administrative distance.
- RIB installation rules.

The maximum-prefix feature operates at the BGP neighbor/address-family level, rather than using the number of routes installed in the IP routing table.

### Pre-Policy vs. Post-Policy Counting

There are two general approaches to maximum-prefix enforcement.

| Method | Prefixes Counted |
|---|---|
| Pre-policy | Received prefixes before inbound routing policy |
| Post-policy | Prefixes accepted after inbound routing policy |

Cisco IOS XE's traditional `neighbor maximum-prefix` mechanism uses **post-policy counting**.

This means that prefixes rejected by inbound filtering generally do not contribute to the accepted prefix count used by this feature.

Consider R2 advertising 1,000 prefixes to R1.

R1 has an inbound prefix list that accepts only 100 of them.

```text
R2 advertises 1000 prefixes
              |
              v
       Inbound Prefix List
              |
       +------+------+
       |             |
       v             v
  100 accepted   900 rejected
       |
       v
   BGP Prefix Count
       |
       v
  Maximum-Prefix Check
```

With traditional post-policy counting, the effective count is 100, rather than 1,000.

**Important:** Post-policy counting does not necessarily prevent all resource consumption associated with processing unwanted BGP UPDATE messages.

Inbound filtering is still important, and maximum-prefix should not be treated as a complete defense against every type of BGP route leak or resource exhaustion.

### Maximum Prefix vs. Prefix List

These features serve different purposes.

| Feature | Purpose |
|---|---|
| `neighbor maximum-prefix` | Limit the number of accepted prefixes |
| `neighbor prefix-list ... in` | Control which prefixes are accepted |
| `neighbor route-map ... in` | Apply inbound route filtering and attribute manipulation |

A maximum-prefix limit does not specify which networks are permitted.

A prefix list does not independently enforce a maximum number of accepted routes.

They can be used together.

---

## Per-Neighbor and Per-Address-Family Operation

Maximum-prefix is normally configured for a specific BGP neighbor and address family.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 maximum-prefix 1000
```

This applies to IPv4 unicast routing information received from that neighbor.

A separate IPv6 unicast limit can be configured where the address family is supported:

```text
R1(config-router)# address-family ipv6 unicast
R1(config-router-af)# neighbor 10.0.12.2 maximum-prefix 500
```

This allows different limits for different address families.

### Peer Groups

Maximum-prefix can also be configured on peer groups.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor EBGP-PEERS peer-group
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor EBGP-PEERS maximum-prefix 1000 80
```

Members of the peer group can inherit the configured limit.

The limit is applied to the individual neighbors, rather than creating a single shared prefix limit for the entire peer group.

### Dynamic Neighbors

Cisco documents that maximum-prefix support for dynamic BGP neighbors requires the **Persistent Dynamic Neighbors** feature.

Support and restrictions depend on the IOS XE release.

See [BGP Dynamic Neighbors](../peering/dynamic-neighbors.md) for dynamic neighbor operation.

---

## Maximum Prefix Configuration Lab

The following lab demonstrates the default maximum-prefix behavior.

### Topology

```text
          AS 100                      AS 200

            R1 ------------------------ R2
         10.0.12.1                  10.0.12.2
                       EBGP
```

R2 originates four prefixes.

R1 initially allows only three.

### R2 Configuration

Assume the peering interfaces are already configured.

Create four static routes to Null0 so the prefixes exist in R2's routing table:

```text
R2(config)# ip route 192.0.2.0 255.255.255.0 Null0
R2(config)# ip route 198.51.100.0 255.255.255.0 Null0
R2(config)# ip route 203.0.113.0 255.255.255.0 Null0
R2(config)# ip route 10.10.10.0 255.255.255.0 Null0
```

Configure BGP:

```text
R2(config)# router bgp 200
R2(config-router)# neighbor 10.0.12.1 remote-as 100
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# network 192.0.2.0 mask 255.255.255.0
R2(config-router-af)# network 198.51.100.0 mask 255.255.255.0
R2(config-router-af)# network 203.0.113.0 mask 255.255.255.0
```

Initially, R2 advertises three prefixes.

### R1 Configuration

```text
R1(config)# router bgp 100
R1(config-router)# neighbor 10.0.12.2 remote-as 200
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 maximum-prefix 3 75
```

Verify the session:

```text
R1# show ip bgp summary
R1# show ip bgp neighbors 10.0.12.2
```

R1 should accept three prefixes.

### Exceed the Limit

On R2, originate the fourth prefix:

```text
R2(config)# router bgp 200
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# network 10.10.10.0 mask 255.255.255.0
```

R2 now advertises four prefixes.

R1's maximum is three.

When R1 receives the additional prefix, it should detect that the maximum has been exceeded and terminate the BGP session.

### Verify the Failure

```text
R1# show ip bgp summary
R1# show ip bgp neighbors 10.0.12.2
```

Check for:

- A neighbor that is no longer Established.
- A maximum-prefix violation.
- A maximum-prefix notification.
- A disabled BGP session.

### Recover the Session

Remove the fourth network statement on R2:

```text
R2(config)# router bgp 200
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# no network 10.10.10.0 mask 255.255.255.0
```

Then clear the session on R1:

```text
R1# clear ip bgp 10.0.12.2
```

Verify that the session reestablishes and R1 receives three prefixes.

### Additional Tests

Repeat the lab with:

```text
neighbor 10.0.12.2 maximum-prefix 3 75 restart 1
```

Observe automatic restart attempts.

Then test:

```text
neighbor 10.0.12.2 maximum-prefix 3 75 warning-only
```

Observe that the session remains Established even when four prefixes are received.

If your IOS XE image supports `discard-extra`, test:

```text
neighbor 10.0.12.2 maximum-prefix 3 discard-extra
```

Verify which routes are retained and whether the session remains Established.

---

## Verification

### Display Maximum Prefix Configuration

```text
R1# show running-config | section router bgp
```

Look for the configured maximum-prefix command.

### Display Neighbor Information

```text
R1# show ip bgp neighbors 10.0.12.2
```

Useful information includes:

- Current BGP session state.
- Number of received prefixes.
- Configured maximum prefix limit.
- Warning threshold.
- Maximum-prefix violation status.
- Restart configuration, if enabled.

For example, Cisco IOS XE can display:

```text
For address family: IPv4 Unicast
  Maximum prefixes allowed 1000
  Threshold for warning message 80%
```

When the prefix limit is exceeded and the session is disabled, the output can include:

```text
Peer had exceeded the max. no. of prefixes configured.
Maximum prefixes allowed 1000
Threshold for warning message 80%
Reduce the no. of prefix and clear ip bgp 10.0.12.2
to restore peering
```

### Display BGP Summary

```text
R1# show ip bgp summary
```

When the neighbor has been disabled because of a prefix-limit violation, the output can show:

```text
Neighbor        AS      State/PfxRcd
10.0.12.2       200     Idle (PfxCt)
```

The **`PfxCt`** indication identifies a BGP neighbor disabled due to a prefix-count violation.

### Verify Accepted Routes

```text
R1# show ip bgp neighbors 10.0.12.2 routes
```

This displays the BGP routes accepted from the neighbor.

Compare these routes with the maximum-prefix information in the detailed neighbor output.

### Verify Discarded Prefixes

On supported IOS XE versions using `discard-extra`:

```text
R1# show ip bgp neighbors 10.0.12.2
```

Inspect the detailed address-family statistics for information about discarded paths.

Cisco's maximum-prefix documentation provides examples of the counters displayed when this feature is enabled.

---

## Troubleshooting

| Symptom | Possible Cause | Solution |
|---|---|---|
| BGP neighbor displays `Idle (PfxCt)` | Maximum-prefix limit exceeded | Reduce advertised prefixes or adjust the limit, then clear the session |
| Neighbor repeatedly establishes and terminates | Maximum-prefix limit is still exceeded with automatic restart enabled | Correct the route advertisement or limit |
| No session termination despite excessive prefixes | `warning-only` configured | Remove `warning-only` if enforcement is required |
| Session remains up but routes are missing | `discard-extra` enabled | Verify discarded prefixes and consider a soft reset |
| Warning generated earlier than expected | Warning percentage configured too low | Verify the warning threshold |
| Prefix count is lower than the number advertised by the neighbor | Inbound policies reject some routes | Examine inbound filtering and accepted routes |
| Expected route missing from the IP routing table | Route not installed due to BGP best-path selection, NEXT_HOP resolution, or RIB selection | Inspect the BGP table and RIB |
| Peer remains disabled after reducing route advertisements | No automatic restart configured | Manually clear the affected BGP session |
| Configuration command rejects `discard-extra` | Unsupported IOS XE version or conflicting configuration | Verify release support and feature restrictions |

### Troubleshooting Sequence

When a BGP neighbor is disabled by maximum-prefix:

1. Check the BGP neighbor's current state.
2. Examine the maximum-prefix limit and warning threshold.
3. Determine how many prefixes the neighbor is expected to advertise.
4. Check inbound routing policies.
5. Investigate unexpected increases in received prefixes.
6. Correct the cause of the excess advertisements.
7. Adjust the configured maximum only if appropriate.
8. Clear the BGP session or allow the configured restart timer to recover it.

**Do not simply increase the maximum-prefix value without determining why the route count increased.** The increased count might indicate a routing policy error or route leak.

---

## Maximum Prefix vs. Other BGP Features

Maximum-prefix should not be confused with other BGP commands containing the word `maximum`.

| Feature | Command | Purpose |
|---|---|---|
| Maximum Prefix | `neighbor ... maximum-prefix` | Limit accepted prefixes from a neighbor |
| BGP Multipath | `maximum-paths` | Permit multiple eligible BGP paths to be installed |
| BGP Multipath for IBGP | `maximum-paths ibgp` | Permit multiple eligible IBGP paths |
| Maximum Prefix with Restart | `maximum-prefix ... restart` | Automatically retry a session terminated for exceeding its prefix limit |

Maximum-prefix does not control how many equal-cost BGP paths are installed in the routing table.

That is the purpose of BGP multipath configuration.

See [BGP Load Balancing](../path-selection/load-balancing.md).

---

## Command Summary

| Command | Purpose |
|---|---|
| `neighbor <IP> maximum-prefix <N>` | Configure a maximum prefix limit |
| `neighbor <IP> maximum-prefix <N> <PERCENT>` | Configure a warning threshold |
| `neighbor <IP> maximum-prefix <N> warning-only` | Generate warnings without session termination |
| `neighbor <IP> maximum-prefix <N> restart <MINUTES>` | Automatically retry a terminated session |
| `neighbor <IP> maximum-prefix <N> discard-extra` | Discard excess prefixes without resetting the session, where supported |
| `no neighbor <IP> maximum-prefix` | Remove the maximum-prefix configuration |
| `show ip bgp neighbors <IP>` | Verify prefix limits and neighbor status |
| `show ip bgp summary` | Verify BGP session states |
| `clear ip bgp <IP>` | Manually reset and attempt to reestablish the neighbor session |

---

## Key Points

- **BGP Maximum Prefix** limits the number of prefixes accepted from a BGP neighbor.
- Configure it with `neighbor <IP> maximum-prefix <MAXIMUM>`.
- Maximum-prefix is disabled by default on ordinary IOS XE BGP neighbors.
- The default warning threshold is **75%** of the configured maximum.
- The warning threshold and maximum limit are separate values.
- By default, exceeding the maximum terminates the BGP session.
- The router sends a **Cease NOTIFICATION (6/1)** when the limit is exceeded.
- Without `restart`, a session disabled by maximum-prefix generally requires a manual clear.
- `restart <MINUTES>` automatically attempts to reestablish the BGP session.
- `warning-only` generates warnings but does not enforce the maximum.
- `discard-extra` drops excess prefixes while maintaining the session on supported IOS XE releases.
- **`discard-extra` was introduced in Cisco IOS XE 17.13.1a.**
- `discard-extra` has important restrictions, including incompatibility with traditional inbound soft reconfiguration.
- Cisco IOS XE's traditional maximum-prefix mechanism uses post-policy accepted-prefix counting.
- Maximum-prefix is configured per neighbor and can be applied to specific address families.
- A maximum-prefix limit is different from `maximum-paths`, which controls BGP multipath installation.

---

## References

- [Cisco: BGP Maximum-Prefix on IOS XE](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-max-prefix.html)
- [Cisco: Configure the BGP Maximum Prefix Feature](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/25160-bgp-maximum-prefix.html)
- [Cisco: BGP Command Reference, neighbor maximum-prefix](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [Cisco: Protecting Border Gateway Protocol for the Enterprise](https://sec.cloudapps.cisco.com/security/center/resources/protecting_border_gateway_protocol)
- [IETF: BGP Maximum Prefix Limits Inbound](https://www.ietf.org/archive/id/draft-sas-idr-maxprefix-inbound-04.html)
- [RFC 4486: Subcodes for BGP Cease Notification Message](https://www.rfc-editor.org/rfc/rfc4486)
