# Client-to-Client Reflection Configuration

By default, a **BGP Route Reflector (RR)** reflects routes learned from one IBGP client to other IBGP clients.

However, Cisco IOS XE allows administrators to control this behavior using the `bgp client-to-client reflection` command.

With the **BGP Multiple Cluster IDs (MCID)** feature, client-to-client reflection can also be controlled for individual clusters.

This allows an RR to:

- Enable or disable client-to-client reflection globally.
- Disable reflection between clients belonging to the same cluster.
- Continue reflecting routes between clients belonging to different clusters.
- Disable intra-cluster reflection for specific clusters while retaining it for others.

These features can reduce unnecessary BGP updates in topologies where certain clients already have direct IBGP sessions with each other.

---

## Default Client-to-Client Reflection

Client-to-client reflection is **enabled by default**.

Consider the following topology:

```text
                    AS 65000

                      R1
                 Route Reflector
                    /     \
                   /       \
                  R2       R3
                Client   Client

            All connections are IBGP
```

R1 is the RR, and R2 and R3 are its clients.

Suppose R2 originates `192.0.2.0/24` and advertises it to R1.

Assuming the route is selected as R1's best path, R1 reflects it to R3.

This is the default behavior.

### Configuration

On R1:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 2.2.2.2 route-reflector-client
R1(config-router-af)# neighbor 3.3.3.3 route-reflector-client
```

The example assumes the IBGP neighbor sessions have already been configured.

No additional command is necessary to enable client-to-client reflection.

The following command explicitly enables or restores the default behavior:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp client-to-client reflection
```

This command is configured in BGP router configuration mode.

---

## Disabling Client-to-Client Reflection Globally

Cisco IOS XE supports the following command:

```text
R1(config-router)# no bgp client-to-client reflection
```

This disables **all client-to-client reflection**, including both intra-cluster and inter-cluster reflection.

It does not disable route reflection entirely.

### Example

Consider:

```text
                      R1
                 Route Reflector
                    /     \
                   /       \
                  R2-------R3
                Client   Client

               All links are IBGP
```

R2 and R3 have a direct IBGP session.

Because they are fully meshed with each other, they can exchange their locally originated and EBGP-learned routes directly.

R1 does not need to reflect routes between them.

Configure R1:

```text
R1(config)# router bgp 65000
R1(config-router)# no bgp client-to-client reflection
```

Now suppose R2 originates `192.0.2.0/24`.

R2 advertises the route directly to R3 and also advertises it to R1.

However, R1 does not reflect the route to R3.

R3 learns the route directly from R2 instead.

### Which Advertisements Are Affected?

Disabling client-to-client reflection does not prevent other forms of route advertisement.

Consider:

```text
                  R2 (Client)
                       |
                       |
                    R1 (RR)
                    /     \
                   /       \
                  R3       R4
                Client   Non-Client
```

Suppose R2 advertises a prefix to R1.

With client-to-client reflection disabled:

- R1 does not reflect the route to R3.
- R1 can still reflect the route to R4.

Similarly, R1 can still reflect routes learned from R4 to R2 and R3.

### Reflection Rules

| Route Learned From | Advertise to Clients | Advertise to Non-Clients |
|---|---|---|
| IBGP client | No | Yes |
| IBGP non-client | Yes | No |
| EBGP neighbor | Yes | Yes |
| Locally originated | Yes | Yes |

The first row refers to other clients, not the neighbor from which the route was received.

These rules assume that the route is eligible for advertisement and is not prevented by routing policies or loop prevention.

**Important:** Disabling client-to-client reflection does not remove the RR client designation or terminate the IBGP sessions.

### The `all` Keyword

The following command is also supported:

```text
R1(config-router)# no bgp client-to-client reflection all
```

The optional `all` keyword does not change the behavior.

It emphasizes that the command disables both:

- Intra-cluster client-to-client reflection.
- Inter-cluster client-to-client reflection.

The following commands are functionally equivalent:

```text
no bgp client-to-client reflection
no bgp client-to-client reflection all
```

The `all` keyword does not appear in the running configuration.

To restore client-to-client reflection globally:

```text
R1(config-router)# bgp client-to-client reflection
```

However, any separately configured intra-cluster restrictions must also be considered.

---

## Intra-Cluster vs. Inter-Cluster Reflection

Cisco IOS XE supports more granular reflection control through the **BGP Multiple Cluster IDs (MCID)** feature.

An RR can have:

- A global cluster ID.
- Additional cluster IDs assigned to individual IBGP neighbors.

Multiple neighbors can be assigned the same cluster ID.

This allows a single RR to participate in multiple logical clusters.

### Example Topology

Consider the following topology:

```text
                       AS 65000

                      R1 (RR)
                     / \   \  \
                    /   \   \  \
                   /     \   \  \
                  /       \   \  \
                 R2-------R3 R4---R5

                 Cluster A     Cluster B
                 CID 10        CID 20
```

R1 has IBGP sessions with all four clients.

Additionally:

- R2 and R3 have a direct IBGP session.
- R4 and R5 have a direct IBGP session.
- There are no direct IBGP sessions between Cluster A and Cluster B.

R1 belongs to both logical clusters.

The clients are assigned cluster IDs as follows:

| Client | Cluster ID |
|---|---|
| R2 | 10.10.10.10 |
| R3 | 10.10.10.10 |
| R4 | 20.20.20.20 |
| R5 | 20.20.20.20 |

### Intra-Cluster Reflection

**Intra-cluster reflection** occurs when the RR reflects a route from one client to another client in the same cluster.

Examples:

```text
R2 -> R1 -> R3
R4 -> R1 -> R5
```

Both the originating client and receiving client belong to the same logical cluster.

### Inter-Cluster Reflection

**Inter-cluster reflection** occurs when the RR reflects a route from a client in one cluster to a client in another cluster.

Examples:

```text
R2 -> R1 -> R4
R2 -> R1 -> R5

R4 -> R1 -> R2
R4 -> R1 -> R3
```

The originating and receiving clients belong to different logical clusters.

**Important:** In the MCID feature, inter-cluster reflection can occur through a single RR serving multiple clusters.

It does not necessarily involve an IBGP session between two different route reflectors.

---

## Configuring Multiple Cluster IDs

To configure multiple logical clusters on one RR, use the `neighbor cluster-id` command.

### R1 Configuration

```text
R1(config)# router bgp 65000
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# bgp cluster-id 1.1.1.1

R1(config-router)# neighbor 2.2.2.2 remote-as 65000
R1(config-router)# neighbor 3.3.3.3 remote-as 65000
R1(config-router)# neighbor 4.4.4.4 remote-as 65000
R1(config-router)# neighbor 5.5.5.5 remote-as 65000

R1(config-router)# neighbor 2.2.2.2 cluster-id 10.10.10.10
R1(config-router)# neighbor 3.3.3.3 cluster-id 10.10.10.10

R1(config-router)# neighbor 4.4.4.4 cluster-id 20.20.20.20
R1(config-router)# neighbor 5.5.5.5 cluster-id 20.20.20.20

R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 2.2.2.2 route-reflector-client
R1(config-router-af)# neighbor 3.3.3.3 route-reflector-client
R1(config-router-af)# neighbor 4.4.4.4 route-reflector-client
R1(config-router-af)# neighbor 5.5.5.5 route-reflector-client
```

This example assumes the IBGP sessions use the router loopbacks and have the necessary `update-source`, activation, and underlying IP reachability configuration.

### Global vs. Per-Neighbor Cluster IDs

The global cluster ID is configured using:

```text
R1(config-router)# bgp cluster-id 1.1.1.1
```

The per-neighbor cluster IDs are configured using:

```text
R1(config-router)# neighbor 2.2.2.2 cluster-id 10.10.10.10
```

The difference is important:

| Command | Purpose |
|---|---|
| `bgp cluster-id <ID>` | Configure the global cluster ID |
| `neighbor <IP> cluster-id <ID>` | Assign a cluster ID to an individual IBGP neighbor |
| `neighbor <IP> route-reflector-client` | Configure the neighbor as an RR client |

The `neighbor cluster-id` command does not itself make a neighbor an RR client.

The RR client relationship must still be configured separately.

### Effect on CLUSTER_LIST

With the MCID feature, the RR uses the **cluster ID associated with the neighbor from which the reflected route was learned**.

For example, if R1 learns a route from R2:

```text
R2 Cluster ID: 10.10.10.10
```

R1 adds:

```text
CLUSTER_LIST: 10.10.10.10
```

If R1 learns a route from R4:

```text
R4 Cluster ID: 20.20.20.20
```

R1 adds:

```text
CLUSTER_LIST: 20.20.20.20
```

If no per-neighbor cluster ID is configured for the source neighbor, R1 uses its global cluster ID.

The MCID feature also extends CLUSTER_LIST loop prevention to account for the configured global and per-neighbor cluster IDs.

See [Cluster IDs, Originator IDs, and Loop Prevention](loop-prevention.md).

---

## Disabling Intra-Cluster Reflection for a Specific Cluster

Cisco IOS XE supports the following command:

```text
R1(config-router)# no bgp client-to-client reflection intra-cluster cluster-id <ID>
```

This disables client-to-client reflection **within the specified cluster**.

Inter-cluster reflection remains enabled unless another command disables it.

### Example: Disable Reflection Within Cluster A

Using the previous topology:

```text
                       AS 65000

                      R1 (RR)
                     / \   \  \
                    /   \   \  \
                   /     \   \  \
                  /       \   \  \
                 R2-------R3 R4---R5

                 Cluster A     Cluster B
                 CID 10        CID 20
```

Suppose:

- R2 and R3 are fully meshed.
- R4 and R5 are not fully meshed, or their direct session is not used for the required route exchange.

We want to disable intra-cluster reflection for Cluster A while keeping it enabled for Cluster B.

Configure:

```text
R1(config)# router bgp 65000
R1(config-router)# no bgp client-to-client reflection intra-cluster cluster-id 10.10.10.10
```

### Result

| Route Source | Destination | Reflected? |
|---|---|---|
| R2 | R3 | No |
| R3 | R2 | No |
| R2 | R4 | Yes |
| R2 | R5 | Yes |
| R4 | R5 | Yes |
| R5 | R4 | Yes |
| R4 | R2 | Yes |
| R4 | R3 | Yes |

These results assume the route is selected as R1's eligible best path.

The command disables only reflection **between clients belonging to Cluster A**.

It does not prevent Cluster A clients from exchanging routes with Cluster B clients through the RR.

### Why Is This Useful?

R2 and R3 already exchange routes directly through their IBGP session.

Reflecting those routes through R1 would be redundant.

However, R1 must continue reflecting routes between Cluster A and Cluster B because the clients in the two clusters do not have direct IBGP sessions with one another.

Selective intra-cluster filtering reduces redundant advertisements without preventing inter-cluster route exchange.

---

## Disabling Intra-Cluster Reflection for Multiple Clusters

Multiple cluster IDs can be specified in one command.

For example:

```text
R1(config)# router bgp 65000
R1(config-router)# no bgp client-to-client reflection intra-cluster cluster-id 10.10.10.10 20.20.20.20
```

This disables intra-cluster reflection for both clusters.

However, inter-cluster reflection remains enabled.

For example:

| Route Source | Destination | Reflected? |
|---|---|---|
| R2 | R3 | No |
| R4 | R5 | No |
| R2 | R4 | Yes |
| R3 | R5 | Yes |
| R4 | R2 | Yes |
| R5 | R3 | Yes |

This is useful when clients within each cluster are fully meshed, but clients in different clusters rely on R1 to exchange routes.

---

## Disabling Intra-Cluster Reflection for All Clusters

The following command disables intra-cluster reflection for **every cluster**:

```text
R1(config)# router bgp 65000
R1(config-router)# no bgp client-to-client reflection intra-cluster cluster-id any
```

The `any` keyword means that the restriction applies to all clusters configured on the RR.

However, inter-cluster reflection remains enabled.

### Example

Consider three logical clusters:

```text
Cluster A: R2, R3
Cluster B: R4, R5
Cluster C: R6, R7
```

With:

```text
no bgp client-to-client reflection intra-cluster cluster-id any
```

The RR does not reflect routes between clients within the same cluster.

For example:

```text
R2 -> R3: Not reflected
R4 -> R5: Not reflected
R6 -> R7: Not reflected
```

However, reflection between clusters continues:

```text
R2 -> R4: Reflected
R2 -> R6: Reflected
R4 -> R7: Reflected
```

This is different from:

```text
no bgp client-to-client reflection
```

The global command disables **both intra-cluster and inter-cluster client-to-client reflection**.

### Comparison

| Command | Intra-Cluster | Inter-Cluster |
|---|---|---|
| Default | Enabled | Enabled |
| `no bgp client-to-client reflection` | Disabled | Disabled |
| `no bgp client-to-client reflection intra-cluster cluster-id any` | Disabled | Enabled |
| `no bgp client-to-client reflection intra-cluster cluster-id <ID>` | Disabled for specified cluster | Enabled |

The distinction between the second and third rows is particularly important.

---

## Configuration Precedence

Cisco IOS XE supports three levels of client-to-client reflection configuration.

The commands are evaluated from **least specific to most specific**.

### Level 1: Global Reflection

```text
no bgp client-to-client reflection
```

This disables all client-to-client reflection.

It affects both intra-cluster and inter-cluster reflection.

### Level 2: Intra-Cluster Reflection for Any Cluster

```text
no bgp client-to-client reflection intra-cluster cluster-id any
```

This disables intra-cluster reflection for all clusters.

Inter-cluster reflection remains enabled if global reflection has not been disabled.

### Level 3: Intra-Cluster Reflection for Specific Clusters

```text
no bgp client-to-client reflection intra-cluster cluster-id 10.10.10.10
```

This disables intra-cluster reflection only for the specified cluster.

### Precedence Rules

The important rule is:

**Once a less-specific configuration disables reflection, a more-specific configuration cannot re-enable it.**

The evaluation order is:

1. Global reflection.
2. Intra-cluster reflection for any cluster.
3. Intra-cluster reflection for specific clusters.

If a command disables reflection at one level, further evaluation stops.

### Example 1: Global Disable Overrides Everything

Suppose R1 has:

```text
R1(config)# router bgp 65000
R1(config-router)# no bgp client-to-client reflection
```

You then enter:

```text
R1(config-router)# bgp client-to-client reflection intra-cluster cluster-id 10.10.10.10
```

This does **not** restore intra-cluster reflection for Cluster A.

The global command still disables all client-to-client reflection.

To restore intra-cluster reflection, first restore global reflection:

```text
R1(config-router)# bgp client-to-client reflection
```

Then check whether any remaining intra-cluster restrictions also need to be removed.

### Example 2: `any` Overrides a Specific Cluster

Suppose R1 has:

```text
R1(config)# router bgp 65000
R1(config-router)# no bgp client-to-client reflection intra-cluster cluster-id any
```

You then enter:

```text
R1(config-router)# bgp client-to-client reflection intra-cluster cluster-id 10.10.10.10
```

This does **not** restore reflection within Cluster A.

The broader `any` restriction still disables intra-cluster reflection for all clusters.

To restore intra-cluster reflection generally:

```text
R1(config-router)# bgp client-to-client reflection intra-cluster cluster-id any
```

### Example 3: Selectively Disable One Cluster

Suppose global reflection is enabled and no `any` restriction is active.

Configure:

```text
R1(config-router)# no bgp client-to-client reflection intra-cluster cluster-id 10.10.10.10
```

The results are:

| Cluster | Intra-Cluster Reflection |
|---|---|
| 10.10.10.10 | Disabled |
| 20.20.20.20 | Enabled |
| 30.30.30.30 | Enabled |

Inter-cluster reflection remains enabled.

### Configuration Precedence Summary

| Configuration | Result |
|---|---|
| No restrictions | All client-to-client reflection enabled |
| Global disable | All client-to-client reflection disabled |
| Disable intra-cluster `any` | Intra-cluster disabled for all clusters |
| Disable specific cluster | Intra-cluster disabled for that cluster |
| Global disable + specific enable | Still globally disabled |
| Intra-cluster `any` disable + specific enable | Still disabled for that cluster |

**Important:** More-specific positive commands cannot override broader negative commands.

Only the negative configurations are normally displayed in the running configuration.

---

## Verification

The most important verification command is:

```text
R1# show ip bgp cluster-ids
```

This displays:

- The global cluster ID.
- Configured per-neighbor cluster IDs.
- The number of neighbors assigned to each cluster.
- Configured client-to-client reflection settings.
- The reflection settings actually used by BGP.

### Example Output

Suppose R1 has:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp cluster-id 1.1.1.1
R1(config-router)# no bgp client-to-client reflection intra-cluster cluster-id 10.10.10.10
```

Cluster A has two clients assigned cluster ID `10.10.10.10`.

Cluster B has two clients assigned cluster ID `20.20.20.20`.

The output can resemble:

```text
R1# show ip bgp cluster-ids

Global cluster-id: 1.1.1.1 (configured: 1.1.1.1)
BGP client-to-client reflection:         Configured    Used
  all (inter-cluster and intra-cluster): ENABLED
  intra-cluster:                         ENABLED       ENABLED

List of cluster-ids:
Cluster-id     #-neighbors C2C-rfl-CFG C2C-rfl-USE
10.10.10.10              2 DISABLED    DISABLED
20.20.20.20              2 ENABLED     ENABLED
```

### Understanding the Output

**Global cluster-id**

Displays the global cluster ID.

If no explicit global cluster ID is configured, the RR uses its BGP router ID.

**all (inter-cluster and intra-cluster)**

Displays whether global client-to-client reflection is enabled.

**intra-cluster**

Displays the configured and effective default intra-cluster reflection settings.

**Cluster-id**

Identifies each configured per-neighbor cluster ID.

**#-neighbors**

Displays the number of neighbors associated with the cluster ID.

**C2C-rfl-CFG**

Displays the configured client-to-client reflection state for the cluster.

**C2C-rfl-USE**

Displays the effective client-to-client reflection state used by BGP.

### Configured vs. Used

The distinction between `Configured` and `Used` is important.

For example, suppose global reflection is disabled:

```text
R1(config-router)# no bgp client-to-client reflection
```

The output can include:

```text
BGP client-to-client reflection:         Configured    Used
  all (inter-cluster and intra-cluster): DISABLED
  intra-cluster:                         ENABLED       DISABLED
```

The intra-cluster configuration is enabled by default, but its effective state is disabled because the broader global restriction takes precedence.

Similarly, individual clusters can show:

```text
Cluster-id     #-neighbors C2C-rfl-CFG C2C-rfl-USE
10.10.10.10              2 ENABLED     DISABLED
20.20.20.20              2 ENABLED     DISABLED
```

This demonstrates that the effective behavior can differ from the directly configured setting.

---

## Verifying Route Advertisements

Use the following commands to verify actual route-reflection behavior:

```text
R1# show ip bgp 192.0.2.0/24
R1# show ip bgp neighbors 3.3.3.3 advertised-routes
R1# show ip bgp update-group
R1# show running-config | section router bgp
```

### Example: Disabled Intra-Cluster Reflection

Suppose R2 originates `192.0.2.0/24`.

R2 and R3 belong to Cluster A.

R4 belongs to Cluster B.

R1 has:

```text
no bgp client-to-client reflection intra-cluster cluster-id 10.10.10.10
```

Check whether R1 advertises the route to R3:

```text
R1# show ip bgp neighbors 3.3.3.3 advertised-routes
```

The route should not be reflected to R3 through normal client-to-client reflection.

Next, check R4:

```text
R1# show ip bgp neighbors 4.4.4.4 advertised-routes
```

R1 can reflect the same route to R4 because R4 belongs to a different cluster.

These checks assume the route is selected as R1's best path and is otherwise eligible for advertisement.

### Update Groups

Cisco IOS XE can place neighbors with compatible outbound advertisement policies into **BGP update groups**.

Use:

```text
R1# show ip bgp update-group
```

Or examine a particular group:

```text
R1# show ip bgp update-group 1
```

The output can identify:

- The address family.
- Whether the group contains RR clients.
- The cluster ID associated with the group.
- The neighbors belonging to the group.

For example, clients assigned the same cluster ID and using compatible outbound policies may be placed in the same update group.

This helps verify how BGP organizes outbound advertisements.

Update groups are an internal BGP optimization and are not the same as route reflector clusters.

---

## Configuration Changes and Soft Refresh

Changing client-to-client reflection settings affects which routes the RR is permitted to advertise.

Cisco IOS XE automatically initiates an **outbound soft refresh** to clients when the reflection state for a cluster changes.

Similarly, changing a neighbor's configured cluster ID triggers inbound soft refresh processing and outbound soft refreshes for the IBGP peers.

Therefore, these configuration changes do not normally require manually resetting all IBGP sessions.

However, they can still cause:

- BGP UPDATE messages.
- Route withdrawals.
- Best-path recalculation.
- Changes in route availability.

Care should be taken when changing reflection policies in a production network.

---

## Common Misconceptions

### 1. Disabling Client-to-Client Reflection Disables All Route Reflection

False.

The global command:

```text
no bgp client-to-client reflection
```

disables reflection between clients.

It does not disable client-to-non-client or non-client-to-client reflection.

### 2. `intra-cluster cluster-id any` Disables All Client-to-Client Reflection

False.

It disables intra-cluster reflection for every cluster.

Inter-cluster reflection remains enabled unless the global command disables it.

### 3. A Specific Enable Overrides a Global Disable

False.

A more-specific positive configuration cannot override a broader negative configuration.

For example:

```text
no bgp client-to-client reflection
bgp client-to-client reflection intra-cluster cluster-id 10.10.10.10
```

Client-to-client reflection remains disabled.

### 4. The Global Cluster ID and Per-Neighbor Cluster ID Are the Same Feature

Not exactly.

The global cluster ID identifies the RR's default cluster ID.

Per-neighbor cluster IDs allow the RR to participate in multiple logical clusters.

The MCID feature also changes CLUSTER_LIST processing and enables per-cluster reflection control.

### 5. Intra-Cluster Reflection Should Always Be Disabled When Clients Have Direct IBGP Sessions

Not necessarily.

Disabling intra-cluster reflection can reduce redundant advertisements, but it also removes an alternative BGP route distribution path.

If the direct sessions between clients fail, the RR will not reflect those routes between them while intra-cluster reflection is disabled.

The decision should account for redundancy and failure scenarios, not only update reduction.

---

## Command Summary

| Command | Purpose |
|---|---|
| `bgp client-to-client reflection` | Enable global client-to-client reflection |
| `no bgp client-to-client reflection` | Disable all client-to-client reflection |
| `no bgp client-to-client reflection all` | Same as the preceding command |
| `bgp client-to-client reflection intra-cluster cluster-id any` | Enable intra-cluster reflection for all clusters, subject to broader restrictions |
| `no bgp client-to-client reflection intra-cluster cluster-id any` | Disable intra-cluster reflection for all clusters |
| `no bgp client-to-client reflection intra-cluster cluster-id <ID>` | Disable intra-cluster reflection for a specified cluster |
| `bgp client-to-client reflection intra-cluster cluster-id <ID>` | Enable intra-cluster reflection for a specified cluster, subject to broader restrictions |
| `bgp cluster-id <ID>` | Configure the global cluster ID |
| `neighbor <IP> cluster-id <ID>` | Configure a per-neighbor cluster ID |
| `neighbor <IP> route-reflector-client` | Configure an RR client |
| `show ip bgp cluster-ids` | Verify cluster IDs and reflection settings |
| `show ip bgp update-group` | Examine BGP update groups |

---

## Key Points

- Client-to-client reflection is enabled by default.
- `no bgp client-to-client reflection` disables both intra-cluster and inter-cluster client-to-client reflection.
- Disabling client-to-client reflection does not affect client-to-non-client or non-client-to-client reflection.
- The optional `all` keyword has no additional effect.
- The MCID feature allows an RR to participate in multiple logical clusters.
- `neighbor cluster-id` assigns a cluster ID to an individual IBGP neighbor.
- **Intra-cluster reflection** occurs between clients belonging to the same cluster.
- **Inter-cluster reflection** occurs between clients belonging to different clusters.
- `no bgp client-to-client reflection intra-cluster cluster-id <ID>` disables intra-cluster reflection for a specified cluster.
- `no bgp client-to-client reflection intra-cluster cluster-id any` disables intra-cluster reflection for all clusters while preserving inter-cluster reflection.
- The global client-to-client reflection setting takes precedence over intra-cluster settings.
- More-specific positive commands cannot override broader negative commands.
- Configuration evaluation stops as soon as a level disables reflection.
- `show ip bgp cluster-ids` displays both configured and effective reflection settings.
- Changing reflection settings triggers outbound soft refresh processing.
- Disabling redundant reflection can improve scalability but can also reduce route availability during failures.

---

## References

- [Cisco: BGP-Multiple Cluster IDs, IOS XE](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-multicluster-id.html)
- [Cisco: Use BGP Route Reflection and Multiple Cluster IDs](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/200153-BGP-Route-Reflection-and-Multiple-Cluste.html)
- [Cisco: BGP Command Reference, bgp client-to-client reflection](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-a1.html)
- [Cisco: BGP Command Reference, show ip bgp cluster-ids](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-s1.html)
- [RFC 4456: BGP Route Reflection](https://www.rfc-editor.org/rfc/rfc4456.html)
