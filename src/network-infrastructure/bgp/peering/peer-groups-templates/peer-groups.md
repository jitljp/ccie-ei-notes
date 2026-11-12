
# BGP Peer Groups

**BGP peer groups** allow multiple BGP neighbors to share a common set of configuration parameters.

Instead of configuring identical settings individually for each neighbor, the settings are configured once on a named peer group. Neighbors assigned to that group inherit its configuration.

Peer groups are useful when multiple neighbors share:

- The same remote AS.
- BGP timers and session parameters.
- An update source.
- Inbound or outbound routing policies.
- Community advertisement settings.
- Route reflector client configuration.

Peer groups reduce configuration duplication and simplify management.

---

## Peer Group Configuration

Configuring a BGP peer group involves three steps:

1. Create the peer group.
2. Configure common settings on the peer group.
3. Assign neighbors to the peer group.

### Basic IBGP Peer Group

Consider the following topology:

```text
                      AS 65001

             R2                 R3
          Lo0: 2.2.2.2      Lo0: 3.3.3.3
               \               /
                \             /
                 \           /
                      R1
                 Lo0: 1.1.1.1

           IBGP sessions to R1
```

R1 forms IBGP sessions with R2 and R3 using their loopback addresses.

Assume an IGP provides reachability between all loopbacks.

**R1:**

```text
R1(config)# router bgp 65001
R1(config-router)# no bgp default ipv4-unicast
R1(config-router)# neighbor IBGP-PEERS peer-group
R1(config-router)# neighbor IBGP-PEERS remote-as 65001
R1(config-router)# neighbor IBGP-PEERS update-source Loopback0
R1(config-router)# neighbor IBGP-PEERS timers 10 30
R1(config-router)# neighbor 2.2.2.2 peer-group IBGP-PEERS
R1(config-router)# neighbor 3.3.3.3 peer-group IBGP-PEERS
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor IBGP-PEERS activate
```

This configuration creates a peer group named `IBGP-PEERS` and assigns two neighbors to it.

Both neighbors inherit:

| Parameter | Inherited value |
|---|---|
| Remote AS | 65001 |
| Update source | Loopback0 |
| Keepalive timer | 10 seconds |
| Hold timer | 30 seconds |
| IPv4 unicast | Activated |

The corresponding BGP configurations on R2 and R3 must reference R1's loopback address and use appropriate session settings.

### Important Commands

| Command | Purpose |
|---|---|
| `neighbor NAME peer-group` | Create a peer group |
| `neighbor NAME remote-as ASN` | Configure a common remote AS |
| `neighbor NAME update-source INTERFACE` | Configure a common update source |
| `neighbor IP peer-group NAME` | Assign a neighbor to a peer group |
| `neighbor NAME activate` | Activate peer-group members for an address family |

The peer-group name is a local configuration identifier. It is not exchanged with remote BGP neighbors.

---

## Address-Family Configuration

Peer groups support address-family-specific settings.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# no bgp default ipv4-unicast
R1(config-router)# neighbor IBGP-PEERS peer-group
R1(config-router)# neighbor IBGP-PEERS remote-as 65001
R1(config-router)# neighbor 2.2.2.2 peer-group IBGP-PEERS
R1(config-router)# neighbor 3.3.3.3 peer-group IBGP-PEERS
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor IBGP-PEERS activate
R1(config-router-af)# neighbor IBGP-PEERS next-hop-self
R1(config-router-af)# neighbor IBGP-PEERS send-community both
```

The following settings apply to both members:

- IPv4 unicast activation.
- `next-hop-self`.
- Advertisement of standard and extended BGP communities.

**Session-level settings** are configured in BGP router configuration mode, while **address-family-specific settings** are configured under the corresponding address family.

By default, IOS XE automatically activates IPv4 unicast for neighbors created using `remote-as`. The examples use `no bgp default ipv4-unicast` to require explicit activation.

---

## Peer Group Configuration Inheritance

All members of a peer group inherit the group's configuration.

If a setting is changed on the peer group, the change applies to its members.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor IBGP-PEERS timers 5 15
```

Both R2 and R3 now inherit:

- Keepalive timer: 5 seconds.
- Hold timer: 15 seconds.

There is no need to modify each neighbor separately.

### Per-Neighbor Overrides

Individual neighbors can override certain inherited settings.

However, **not all peer-group settings can be overridden**.

Traditional Cisco BGP peer groups distinguish between settings that affect outbound UPDATE generation and settings that do not.

| Setting | Individual override |
|---|---|
| Inbound route map | Yes |
| Inbound prefix list | Yes |
| Inbound filter list | Yes |
| Outbound route map | No, traditionally shared |
| Outbound prefix list | No, traditionally shared |
| `next-hop-self` | Inherited group setting |
| `update-source` | Inherited group setting |

The traditional peer-group model requires common outbound update policies. Modern IOS XE dynamic update groups provide additional flexibility in UPDATE generation, but the supported configuration overrides still depend on the platform and release.

For example, different inbound route maps can be applied to individual members:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 2.2.2.2 route-map FROM-R2 in
R1(config-router-af)# neighbor 3.3.3.3 route-map FROM-R3 in
```

Both neighbors remain members of `IBGP-PEERS`, but their received routes are processed using different inbound policies.

**Important:** If neighbors require substantially different outbound routing policies, use separate peer groups or consider [BGP Peer Templates](peer-templates.md).

---

## Peer Groups with EBGP Neighbors

A peer group does not require all its members to belong to the same remote AS.

For EBGP neighbors in different ASes, configure `remote-as` individually rather than on the peer group.

### Topology

```text
       AS 65002           AS 65001           AS 65003

          R2 --------------- R1 --------------- R3
       10.0.12.2       10.0.12.1  
                              10.0.13.1     10.0.13.3    
                          
```

R1 has two EBGP neighbors:

- R2 belongs to AS 65002.
- R3 belongs to AS 65003.

Both neighbors require the same BGP timers and community advertisement settings.

### R1 Configuration

```text
R1(config)# router bgp 65001
R1(config-router)# no bgp default ipv4-unicast
R1(config-router)# neighbor EBGP-PEERS peer-group
R1(config-router)# neighbor EBGP-PEERS timers 10 30
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# neighbor 10.0.13.3 remote-as 65003
R1(config-router)# neighbor 10.0.12.2 peer-group EBGP-PEERS
R1(config-router)# neighbor 10.0.13.3 peer-group EBGP-PEERS
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor EBGP-PEERS activate
R1(config-router-af)# neighbor EBGP-PEERS send-community both
```

Notice that the peer group itself does **not** have a `remote-as` statement.

Each neighbor specifies its own remote ASN.

If `remote-as 65002` were configured on the peer group, all its members would inherit that ASN rather than using independently configured remote ASNs.

This distinction is particularly important when configuring EBGP peer groups.

---

## Peer Groups and Routing Policies

Routing policies can be applied to a peer group instead of individual neighbors.

For example, configure an outbound prefix list that permits only `192.0.2.0/24`:

```text
R1(config)# ip prefix-list ADVERTISE permit 192.0.2.0/24
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor EBGP-PEERS prefix-list ADVERTISE out
```

Both peer-group members inherit the outbound prefix-list filter.

Only `192.0.2.0/24` is permitted by this prefix list. The route must also be eligible for BGP advertisement.

### Different Inbound Policies

Members of the same peer group can use different inbound policies.

```text
R1(config)# ip prefix-list FROM-R2 permit 10.0.0.0/8 le 24
R1(config)# ip prefix-list FROM-R3 permit 172.16.0.0/12 le 24
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 prefix-list FROM-R2 in
R1(config-router-af)# neighbor 10.0.13.3 prefix-list FROM-R3 in
```

R1 accepts:

- From R2: prefixes within `10.0.0.0/8`, with prefix lengths /8 through /24.
- From R3: prefixes within `172.16.0.0/12`, with prefix lengths /12 through /24.

Both peers retain their inherited group configuration.

This allows common outbound behavior while applying neighbor-specific inbound filtering.

---

## Peer Groups and Route Reflectors

Peer groups are commonly used with IBGP route reflectors.

Instead of configuring `route-reflector-client` separately for every client, configure it on the peer group.

```text
R1(config)# router bgp 65001
R1(config-router)# no bgp default ipv4-unicast
R1(config-router)# neighbor RR-CLIENTS peer-group
R1(config-router)# neighbor RR-CLIENTS remote-as 65001
R1(config-router)# neighbor RR-CLIENTS update-source Loopback0
R1(config-router)# neighbor 2.2.2.2 peer-group RR-CLIENTS
R1(config-router)# neighbor 3.3.3.3 peer-group RR-CLIENTS
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor RR-CLIENTS activate
R1(config-router-af)# neighbor RR-CLIENTS route-reflector-client
```

Both R2 and R3 are configured as route reflector clients of R1.

This is particularly useful when a route reflector serves many clients.

---

## Peer Groups and Dynamic Update Groups

Historically, BGP peer groups provided both:

1. Configuration simplification.
2. More efficient generation of outbound BGP UPDATE messages.

Traditional peer groups shared outbound policies so BGP could avoid independently generating equivalent UPDATE messages for every neighbor.

Modern Cisco IOS XE uses **dynamic update groups** to optimize UPDATE processing automatically.

Neighbors with compatible outbound advertisement characteristics can share UPDATE generation, regardless of whether they belong to the same manually configured peer group.

This means:

- Peer groups remain useful for configuration management.
- Dynamic update groups handle UPDATE generation optimization.
- A manual peer group and a dynamic update group are different concepts.

Dynamic update group membership can be examined using:

```text
R1# show ip bgp update-group
```

---

## Removing or Disabling Peer Groups

### Remove a Neighbor from a Peer Group

```text
R1(config)# router bgp 65001
R1(config-router)# no neighbor 2.2.2.2 peer-group IBGP-PEERS
```

This removes R2's membership in the peer group.

Verify the remaining neighbor configuration because settings inherited from the group no longer apply.

### Remove a Peer Group

```text
R1(config)# router bgp 65001
R1(config-router)# no neighbor IBGP-PEERS peer-group
```

Removing the group also removes its shared configuration from its members. Review the resulting neighbor configuration and session state.

### Administratively Disable a Peer Group

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor IBGP-PEERS shutdown
```

This administratively disables the peer group and its member sessions without deleting the peer-group configuration.

To reenable it:

```text
R1(config-router)# no neighbor IBGP-PEERS shutdown
```

---

## Verification

### Display Peer Groups

```text
R1# show ip bgp peer-group
```

Displays information about configured BGP peer groups.

To examine a specific peer group:

```text
R1# show ip bgp peer-group IBGP-PEERS
```

To display a summary of its members:

```text
R1# show ip bgp peer-group IBGP-PEERS summary
```

These commands can help identify peer-group members and their session status.

### Verify Individual Neighbors

```text
R1# show ip bgp summary
R1# show ip bgp neighbors 2.2.2.2
```

The detailed neighbor output can identify peer-group membership and effective BGP session parameters.

For address-family-specific information:

```text
R1# show bgp ipv4 unicast neighbors 2.2.2.2
```

### Verify Advertised and Received Routes

```text
R1# show bgp ipv4 unicast neighbors 2.2.2.2 advertised-routes
R1# show bgp ipv4 unicast neighbors 2.2.2.2 routes
```

Use these commands to verify that the expected routes are advertised to or accepted from each member.

### Verify Dynamic Update Groups

```text
R1# show ip bgp update-group
R1# show ip bgp replication
```

These commands provide information about outbound UPDATE generation and replication on supported IOS XE platforms.

### Examine the Configuration

```text
R1# show running-config | section router bgp
```

Verify that the peer group exists, neighbors belong to the expected group, and address-family policies are correctly applied.

---

## Common Issues

| Problem | Possible cause |
|---|---|
| Neighbor remains Idle | Missing or incorrect `remote-as` |
| Neighbor remains Active/Connect | Incorrect update source, routing, or TCP reachability |
| Neighbor does not exchange IPv4 routes | IPv4 unicast not activated |
| EBGP peer reports an ASN mismatch | Incorrect inherited `remote-as` |
| A neighbor receives unexpected routes | Shared outbound policy applies to the whole group |
| Different inbound filters are required | Configure neighbor-specific inbound policies |
| Neighbor loses settings after removal from group | Settings were inherited from the peer group |
| Cannot combine peer group and template | A neighbor cannot use both configuration models |

---

## Command Summary

| Command | Purpose |
|---|---|
| `neighbor NAME peer-group` | Create a peer group |
| `neighbor IP peer-group NAME` | Assign a neighbor to a peer group |
| `no neighbor IP peer-group NAME` | Remove a neighbor from the group |
| `neighbor NAME remote-as ASN` | Configure a common remote ASN |
| `neighbor IP remote-as ASN` | Configure an individual neighbor's ASN |
| `neighbor NAME update-source Loopback0` | Configure a common update source |
| `neighbor NAME timers 10 30` | Configure common keepalive and hold timers |
| `neighbor NAME activate` | Activate the group's members for an address family |
| `neighbor NAME next-hop-self` | Configure next-hop-self for group members |
| `neighbor NAME route-reflector-client` | Make group members route reflector clients |
| `neighbor NAME shutdown` | Administratively disable the peer group |
| `no neighbor NAME peer-group` | Delete a peer group |
| `show ip bgp peer-group` | Display peer-group information |
| `show ip bgp peer-group NAME summary` | Display peer-group member status |
| `show ip bgp update-group` | Display dynamic update groups |

---

## Key Points

- Peer groups allow multiple BGP neighbors to **inherit common configuration**.
- Create a peer group using `neighbor NAME peer-group`.
- Assign members using `neighbor IP peer-group NAME`.
- A peer group can contain **EBGP neighbors from different ASes** if `remote-as` is configured individually.
- Neighbor-specific inbound policies are supported.
- Traditional peer groups require common outbound policies.
- Peer groups can simplify route reflector client configuration.
- Modern IOS XE uses **dynamic update groups** for efficient UPDATE generation.
- A neighbor cannot simultaneously use a peer group and peer templates.

---

## Cisco Documentation

- [BGP 4, Cisco IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-bgp4.html)
- [Cisco IOS BGP Command Reference: Neighbor Peer Group](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [Cisco IOS BGP Command Reference: Show IP BGP Peer Group](https://www.cisco.com/c/en/us/td/docs/ios/iproute_bgp/command/reference/irg_book/irg_bgp5.html)
