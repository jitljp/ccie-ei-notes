
# Route Reflector Operation and Configuration

A **BGP Route Reflector (RR)** allows IBGP routes to be advertised between IBGP neighbors without requiring a full mesh of IBGP sessions.

Normally, a router does not advertise an IBGP-learned route to another IBGP neighbor. A route reflector provides an exception to this rule.

The primary configuration command is:

```text
R1(config-router-af)# neighbor <IP> route-reflector-client
```

This identifies an IBGP neighbor as a **route reflector client**.

An RR and its clients form a **cluster**.

There is no separate command to enable the route reflector role. A router becomes an RR when one or more of its IBGP neighbors are configured as route reflector clients.

---

## Route Reflector Operation

An RR divides its IBGP neighbors into two groups:

- **Clients:** Neighbors configured with `neighbor route-reflector-client`.
- **Non-clients:** Other IBGP neighbors that have not been configured as clients.

A route reflector can advertise IBGP-learned routes to other IBGP neighbors according to specific reflection rules.

| Route Learned From | Advertised To |
|---|---|
| EBGP neighbor | Clients and non-clients |
| IBGP client | Other clients and non-clients |
| IBGP non-client | Clients only |

These rules assume that the route is eligible for advertisement and permitted by outbound routing policies.

By default, an RR advertises its **selected best path**, not every path it receives.

An RR still performs normal BGP best-path selection. Being configured as a route reflector does not change the fundamental BGP path selection algorithm.

See [Route Reflection Rules](reflection-rules.md) for a detailed explanation.

---

## Basic Route Reflector Configuration

Consider the following topology:

```text
                         AS 65000

                            R1
                      Route Reflector
                         1.1.1.1
                        /   |   \
                       /    |    \
                      /     |     \
                     R2     R3     R4
                  2.2.2.2 3.3.3.3 4.4.4.4
                     \      |      /
                        RR Clients
```

All routers belong to AS 65000.

R1 acts as the route reflector, while R2, R3, and R4 are its clients.

The diagram represents **logical IBGP sessions**, not physical links. The physical network can use any topology that provides the required IP reachability.

### Addressing

| Router | Loopback0 | Role |
|---|---|---|
| R1 | 1.1.1.1/32 | Route Reflector |
| R2 | 2.2.2.2/32 | RR Client |
| R3 | 3.3.3.3/32 | RR Client |
| R4 | 4.4.4.4/32 | RR Client |

The loopback addresses are used as both the BGP router IDs and the addresses for IBGP peering.

Assume all Loopback0 addresses are reachable through an IGP such as OSPF.

The routers do not need to be directly connected because IBGP supports multihop peering by default.

### R1: Route Reflector

```text
R1(config)# router bgp 65000
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# no bgp default ipv4-unicast

R1(config-router)# neighbor 2.2.2.2 remote-as 65000
R1(config-router)# neighbor 2.2.2.2 update-source Loopback0

R1(config-router)# neighbor 3.3.3.3 remote-as 65000
R1(config-router)# neighbor 3.3.3.3 update-source Loopback0

R1(config-router)# neighbor 4.4.4.4 remote-as 65000
R1(config-router)# neighbor 4.4.4.4 update-source Loopback0

R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 2.2.2.2 activate
R1(config-router-af)# neighbor 3.3.3.3 activate
R1(config-router-af)# neighbor 4.4.4.4 activate

R1(config-router-af)# neighbor 2.2.2.2 route-reflector-client
R1(config-router-af)# neighbor 3.3.3.3 route-reflector-client
R1(config-router-af)# neighbor 4.4.4.4 route-reflector-client
```

The three `route-reflector-client` commands are the only commands required to enable route reflection for these IBGP neighbors, assuming their sessions have already been configured.

The `no bgp default ipv4-unicast` command disables automatic IPv4 unicast activation for newly configured neighbors. Therefore, the example explicitly activates the neighbors under the IPv4 unicast address family.

### R2: RR Client

```text
R2(config)# router bgp 65000
R2(config-router)# bgp router-id 2.2.2.2
R2(config-router)# no bgp default ipv4-unicast
R2(config-router)# neighbor 1.1.1.1 remote-as 65000
R2(config-router)# neighbor 1.1.1.1 update-source Loopback0

R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 1.1.1.1 activate
```

### R3: RR Client

```text
R3(config)# router bgp 65000
R3(config-router)# bgp router-id 3.3.3.3
R3(config-router)# no bgp default ipv4-unicast
R3(config-router)# neighbor 1.1.1.1 remote-as 65000
R3(config-router)# neighbor 1.1.1.1 update-source Loopback0

R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# neighbor 1.1.1.1 activate
```

### R4: RR Client

```text
R4(config)# router bgp 65000
R4(config-router)# bgp router-id 4.4.4.4
R4(config-router)# no bgp default ipv4-unicast
R4(config-router)# neighbor 1.1.1.1 remote-as 65000
R4(config-router)# neighbor 1.1.1.1 update-source Loopback0

R4(config-router)# address-family ipv4 unicast
R4(config-router-af)# neighbor 1.1.1.1 activate
```

**Important:** Route reflector clients do not require any special client-side configuration.

The `route-reflector-client` command is configured only on the RR.

From the clients' perspective, the RR is an ordinary IBGP neighbor.

---

## Route Propagation Example

Suppose R2 originates the prefix `192.0.2.0/24`.

For example, configure a second loopback interface on R2:

```text
R2(config)# interface Loopback1
R2(config-if)# ip address 192.0.2.1 255.255.255.0

R2(config)# router bgp 65000
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# network 192.0.2.0 mask 255.255.255.0
```

The `network` statement requires an exact matching route in R2's IP routing table.

The connected Loopback1 route satisfies this requirement.

For this example, `192.0.2.0/24` is not advertised into the IGP.

### Step 1: R2 Advertises the Prefix to R1

R2 originates `192.0.2.0/24` into BGP and advertises it to R1 through IBGP.

The BGP NEXT_HOP is R2's loopback peering address:

```text
Prefix:     192.0.2.0/24
NEXT_HOP:   2.2.2.2
AS_PATH:    empty
ORIGIN:     IGP
```

The AS_PATH is empty because the route was originated locally and has not crossed an EBGP AS boundary.

### Step 2: R1 Selects the Best Path

R1 receives the update and processes it using the normal BGP best-path selection algorithm.

Assuming R1 selects the route as its best path, it can reflect the route to R3 and R4.

Without route reflection, R1 would not advertise this IBGP-learned route to either router.

### Step 3: R1 Reflects the Route

R1 advertises the prefix to R3 and R4.

```text
                       R1 (RR)
                           |
                Reflects 192.0.2.0/24
                       /       \
                      v         v
                     R3         R4
```

R1 adds route reflection attributes:

- `ORIGINATOR_ID`: Identifies R2 as the original BGP speaker.
- `CLUSTER_LIST`: Records R1's cluster ID.

By default, R1's cluster ID is its BGP router ID, `1.1.1.1`.

### Step 4: R3 and R4 Receive the Route

R3 and R4 receive the reflected route from R1.

The route retains its original BGP NEXT_HOP of `2.2.2.2`.

The clients can install the route in their IP routing tables if it is valid, selected as the best path, its next hop is reachable, and it wins the RIB selection process.

**The route reflector does not automatically become the next hop for reflected routes.**

---

## Route Reflectors and NEXT_HOP

By default, route reflection does not modify the BGP `NEXT_HOP` attribute.

This is an important consideration when designing a route reflector topology.

In the previous example:

```text
                    R1 (RR)
                   /       \
                  /         \
                 R2         R3

          Prefix: 192.0.2.0/24
          Originated by R2

          R3 receives:
          BGP NEXT_HOP = 2.2.2.2
```

Although R3 receives the route from R1, its BGP next hop remains R2.

R3 must have IP reachability to `2.2.2.2`.

### The `next-hop-self` Command

On Cisco IOS XE, the normal `neighbor next-hop-self` command does not rewrite the next hop of IBGP-learned routes being reflected.

For example:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 3.3.3.3 next-hop-self
```

This does not cause R1 to become the next hop for the route originated by R2 and reflected to R3.

Cisco IOS XE also supports the `all` keyword:

```text
R1(config-router-af)# neighbor 3.3.3.3 next-hop-self all
```

The `all` keyword allows the RR to rewrite the next hop of both EBGP-learned and IBGP-learned routes, including reflected routes.

An alternative is to use an outbound route map with `set ip next-hop`, where supported.

Changing the next hop makes the RR part of the forwarding path, so it should only be done when the design requires it.

**In most route reflector designs, the original NEXT_HOP is preserved, and the IGP provides reachability to it.**

This allows the route reflector to operate primarily as a control-plane device rather than carrying traffic between clients.

---

## Configuring Route Reflectors with Peer Groups

When an RR has many clients, configuring every neighbor individually becomes repetitive.

**BGP peer groups** allow the clients to share common settings.

For example, on R1:

```text
R1(config)# router bgp 65000
R1(config-router)# no bgp default ipv4-unicast

R1(config-router)# neighbor RR-CLIENTS peer-group
R1(config-router)# neighbor RR-CLIENTS remote-as 65000
R1(config-router)# neighbor RR-CLIENTS update-source Loopback0

R1(config-router)# neighbor 2.2.2.2 peer-group RR-CLIENTS
R1(config-router)# neighbor 3.3.3.3 peer-group RR-CLIENTS
R1(config-router)# neighbor 4.4.4.4 peer-group RR-CLIENTS

R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor RR-CLIENTS activate
R1(config-router-af)# neighbor RR-CLIENTS route-reflector-client
```

The `route-reflector-client` command is applied to the peer group rather than to every individual neighbor.

All three members inherit the RR client configuration.

Peer groups are particularly useful when large numbers of clients share identical session settings and routing policies.

Route reflector client configuration can also be inherited through BGP peer policy templates.

See [BGP Peer Groups and Templates](../../peering/peer-groups-templates/overview.md) for details.

---

## Configuring Client-to-Client Reflection

By default, an RR is permitted to reflect routes learned from one client to other clients.

For example:

```text
             R1 (RR)
              /    \
             /      \
            R2      R3
          Client   Client
```

When R2 advertises a route to R1, R1 can reflect that route to R3.

This behavior is enabled by default.

To disable it:

```text
R1(config)# router bgp 65000
R1(config-router)# no bgp client-to-client reflection
```

To reenable it:

```text
R1(config-router)# bgp client-to-client reflection
```

Disabling client-to-client reflection can be appropriate when clients are already fully meshed and can exchange routes directly.

However, disabling it without another mechanism for exchanging client routes can cause reachability problems.

See [Route Reflection Rules](reflection-rules.md) for more details.

---

## Configuring the Cluster ID

Each route reflector has a **cluster ID**, which identifies its route reflector cluster.

By default, the cluster ID is the RR's BGP router ID.

To configure it manually:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp cluster-id 10.10.10.10
```

The cluster ID can be configured in dotted-decimal format or as a 32-bit decimal value.

When multiple RRs serve the same clients, both of the following designs are valid:

- **Same cluster ID:** Both RRs identify themselves as members of the same cluster.
- **Different cluster IDs:** Each RR uses its own cluster ID, creating separate logical clusters that may serve the same clients.

These designs have different implications for loop prevention, path visibility, and redundancy.

See [Multiple Route Reflectors and Redundancy](redundancy.md) for a detailed comparison.


---

## Verifying Route Reflector Operation

Several commands can be used to verify the route reflector configuration and confirm that routes are being reflected correctly.

### Verify BGP Neighbor Sessions

On R1:

```text
R1# show ip bgp summary
```

All three neighbors should be in the Established state.

The `State/PfxRcd` column normally displays a numerical prefix count when the session is Established, rather than the word `Established`.

### Verify Route Reflector Client Relationships

On R1:

```text
R1# show ip bgp neighbors 2.2.2.2
```

The neighbor details should identify R2 as a route reflector client.

Repeat the command for R3 and R4:

```text
R1# show ip bgp neighbors 3.3.3.3
R1# show ip bgp neighbors 4.4.4.4
```

Alternatively, examine the running configuration:

```text
R1# show running-config | section router bgp
```

Verify that the expected neighbors have the `route-reflector-client` command configured.

### Verify Reflected Routes

On R3:

```text
R3# show ip bgp 192.0.2.0/24
```

The route should have the following properties:

| Attribute | Expected Value |
|---|---|
| Prefix | `192.0.2.0/24` |
| NEXT_HOP | `2.2.2.2` |
| AS_PATH | Empty |
| ORIGIN | IGP |
| ORIGINATOR_ID | `2.2.2.2` |
| CLUSTER_LIST | `1.1.1.1` |
| Learned from | `1.1.1.1` (R1) |

The output should include information similar to:

```text
Originator: 2.2.2.2, Cluster list: 1.1.1.1
```

These attributes confirm that the route was reflected through R1.

The `ORIGINATOR_ID` identifies R2, while the `CLUSTER_LIST` identifies the cluster through which the route was reflected.

### Verify Advertised Routes

On R1:

```text
R1# show ip bgp neighbors 3.3.3.3 advertised-routes
```

This displays routes R1 advertises to R3.

The prefix `192.0.2.0/24` should appear in the output.

You can also check R4:

```text
R1# show ip bgp neighbors 4.4.4.4 advertised-routes
```

### Verify Cluster Information

On R1:

```text
R1# show ip bgp cluster-ids
```

This command displays information about the route reflector's cluster IDs, associated neighbors, and client-to-client reflection settings.

### Verify Next-Hop Reachability

On R3:

```text
R3# show ip route 2.2.2.2
R3# ping 2.2.2.2 source Loopback0
```

R3 must be able to resolve the BGP next hop of `2.2.2.2`.

If the route is valid and selected as the best path, but the next hop is unreachable, the route cannot be installed in the IP routing table.

Check the destination route:

```text
R3# show ip route 192.0.2.0
```

With no more-preferred route from another source, the prefix should be installed as an IBGP route with the default administrative distance of 200.

---

## Troubleshooting Route Reflection

An Established IBGP session does not guarantee that routes will be reflected or installed.

When a route reflector fails to advertise a prefix, check the following conditions.

### 1. Missing Route Reflector Client Configuration

If an RR learns a route from an IBGP non-client, it cannot reflect that route to another non-client.

For example, consider the following configuration:

```text
R1(config-router-af)# neighbor 2.2.2.2 route-reflector-client
```

Only R2 is configured as a client.

R3 and R4 are non-clients.

A route learned from R3 cannot be reflected to R4 because both routers are non-clients.

However, a route learned from R2 can still be reflected to R3 and R4.

**A route reflector does not require both the source and destination neighbors to be clients for reflection to occur.**

### 2. The Route Is Not the Best Path

By default, a route reflector advertises only its selected best path.

If R1 learns multiple paths to the same destination, it normally reflects only the path selected by its BGP best-path algorithm.

Check:

```text
R1# show ip bgp 192.0.2.0/24
```

Verify which path has been selected as best.

A different best path can prevent the expected path from being reflected.

### 3. Unreachable BGP NEXT_HOP

By default, reflected routes retain their original NEXT_HOP attribute.

A client may receive a route but be unable to use it if its next hop cannot be resolved.

Verify reachability using the IP routing table.

### 4. Route Reflection Loop Prevention

An RR rejects routes that contain its own cluster ID in their `CLUSTER_LIST`.

A router also rejects routes whose `ORIGINATOR_ID` matches its own BGP router ID.

These mechanisms prevent routing information loops.

See [Cluster IDs, Originator IDs, and Loop Prevention](loop-prevention.md) for details.

### 5. Routing Policy Filtering

Inbound or outbound route maps, prefix lists, and other BGP routing policies can prevent routes from being accepted or advertised.

Check the policies configured on the relevant neighbors:

```text
R1# show ip bgp neighbors 3.3.3.3
R1# show running-config | section router bgp
```

Remember that Cisco IOS XE restricts how outbound route-map `set` clauses affect reflected routes. In particular, most attribute modifications are not applied to reflected IBGP routes, although supported next-hop modifications are an exception.

---

## Configuration Command Summary

| Command | Purpose |
|---|---|
| `neighbor <IP> remote-as <ASN>` | Configure an IBGP neighbor |
| `neighbor <IP> update-source Loopback0` | Source the BGP session from Loopback0 |
| `neighbor <IP> activate` | Activate the neighbor for an address family |
| `neighbor <IP> route-reflector-client` | Configure an IBGP neighbor as an RR client |
| `no neighbor <IP> route-reflector-client` | Remove the RR client designation |
| `bgp cluster-id <ID>` | Manually configure the cluster ID |
| `no bgp client-to-client reflection` | Disable client-to-client route reflection |
| `bgp client-to-client reflection` | Enable client-to-client reflection |
| `neighbor <IP> next-hop-self all` | Rewrite NEXT_HOP for both EBGP- and IBGP-learned routes, including reflected paths, where supported |

---

## Key Points

- An RR is enabled by configuring one or more IBGP neighbors as route reflector clients.
- The `neighbor route-reflector-client` command can be configured under an address family.
- RR clients do not require any special client-side configuration.
- Clients do not need to establish IBGP sessions with each other.
- An RR still performs normal BGP best-path selection.
- By default, only the selected best path is advertised.
- The original BGP NEXT_HOP is normally preserved when a route is reflected.
- The `next-hop-self all` option can modify the next hop of reflected routes on Cisco IOS XE.
- Peer groups and peer policy templates can simplify RR client configuration.
- Client-to-client reflection is enabled by default.
- Both same-cluster-ID and different-cluster-ID designs are valid for redundant RRs.
- Use `show ip bgp neighbors`, `show ip bgp <prefix>`, and `show ip bgp cluster-ids` to verify route reflector operation.

---

## References

- [RFC 4456: BGP Route Reflection](https://www.rfc-editor.org/rfc/rfc4456.html)
- [Cisco: Configuring Internal BGP Features (IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-int-features-0.html)
- [Cisco: Use BGP Route Reflection and Multiple Cluster IDs](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/200153-BGP-Route-Reflection-and-Multiple-Cluste.html)
- [Cisco: BGP Command Reference, Neighbor Commands](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
