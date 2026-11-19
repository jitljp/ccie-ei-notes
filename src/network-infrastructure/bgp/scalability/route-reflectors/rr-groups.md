# BGP Route Reflector Groups

**BGP Route Reflector Groups (RR Groups)** provide a mechanism for filtering VPNv4 routes received by a route reflector based on their **Route Target (RT) extended communities**.

In large MPLS Layer 3 VPN networks, a route reflector may receive VPNv4 routes belonging to hundreds or thousands of different customer VPNs.

However, an RR does not necessarily need to retain routes belonging to every VPN.

Cisco IOS supports the `bgp rr-group` command to selectively accept VPNv4 routes based on their route targets.

The primary command is:

```text
R1(config-router-af)# bgp rr-group <extcommunity-list-number>
```

This enables **automatic inbound route-target filtering** using an extended community list.

The feature can reduce the number of VPNv4 routes maintained by an RR, improving scalability.

**Important:** An RR group is not the same as an RR cluster, BGP peer group, or BGP update group.

---

## The Problem: VPNv4 Route Scalability

In an MPLS L3VPN network, PE routers exchange customer routes using MP-BGP.

Each VPNv4 route includes:

- A Route Distinguisher (RD), which makes otherwise identical IPv4 prefixes unique.
- A VPNv4 prefix.
- One or more Route Target (RT) extended communities.
- A VPN label.
- Other BGP path attributes.

The RD identifies a unique VPNv4 route, while RTs control which VRFs are eligible to import the route.

See [RD and RT](../../../../transport-technologies-and-solutions/mpls/l3vpn/rd-rt.md) for more information.

### Example Topology

Consider the following MPLS L3VPN network:

```text
                        AS 65000

                         R1 (RR)
                        /   |   \
                       /    |    \
                      /     |     \
                    PE1    PE2    PE3
                     |      |      |
                   VPN A  VPN A  VPN B
                   RT 100 RT 100 RT 200

               All connections are MP-IBGP
```

R1 is a VPNv4 route reflector.

PE1 and PE2 participate in VPN A:

```text
Route Target: 65000:100
```

PE3 participates in VPN B:

```text
Route Target: 65000:200
```

Suppose the PE routers advertise:

| Router | VPNv4 Prefix | Route Target |
|---|---|---|
| PE1 | 10.10.10.0/24 | 65000:100 |
| PE2 | 10.20.20.0/24 | 65000:100 |
| PE3 | 10.30.30.0/24 | 65000:200 |

Without additional filtering, the RR can learn and reflect routes belonging to both VPNs.

In a large service-provider network, this can require maintaining substantial VPNv4 routing information.

### Partitioning the VPNv4 Routing Information

Suppose R1 is intended to serve only VPN A.

R1 needs to accept routes carrying:

```text
RT 65000:100
```

It does not need to retain routes carrying only:

```text
RT 65000:200
```

The `bgp rr-group` command allows this filtering to be performed automatically.

---

## How BGP RR Groups Work

The `bgp rr-group` command references an extended community list containing permitted route targets.

For example:

```text
R1(config)# ip extcommunity-list 10 permit rt 65000:100

R1(config)# router bgp 65000
R1(config-router)# address-family vpnv4
R1(config-router-af)# bgp rr-group 10
```

This configures the RR to accept VPNv4 routes matching RT `65000:100`.

Routes that do not match the permitted route targets are filtered.

### Route Processing

When R1 receives a VPNv4 BGP UPDATE:

1. R1 examines the route's RT extended communities.
2. R1 evaluates the configured extended community list.
3. If the route matches a permitted RT, it passes the RR-group filter.
4. If no permitted RT matches, the route is filtered.
5. Accepted routes continue through normal BGP processing, including best-path selection and route reflection.

The RR group therefore controls which VPNv4 routes the RR retains and can subsequently reflect.

### Example

With the following configuration:

```text
R1(config)# ip extcommunity-list 10 permit rt 65000:100

R1(config)# router bgp 65000
R1(config-router)# address-family vpnv4
R1(config-router-af)# bgp rr-group 10
```

The result is:

| VPNv4 Prefix | Route Target | Accepted by RR Group? |
|---|---|---|
| 10.10.10.0/24 | 65000:100 | Yes |
| 10.20.20.0/24 | 65000:100 | Yes |
| 10.30.30.0/24 | 65000:200 | No |
| 10.40.40.0/24 | 65000:300 | No |

This allows R1 to maintain routes for VPN A without retaining routes belonging exclusively to other VPNs.

**Important:** The filter operates on RT extended communities, not on the IPv4 prefix, Route Distinguisher, or AS_PATH.

---

## BGP RR Group Configuration

The configuration requires two primary steps:

1. Define an extended community list containing the permitted RTs.
2. Reference that list using `bgp rr-group` under the VPNv4 address family.

### Step 1: Configure an Extended Community List

Create a standard extended community list:

```text
R1(config)# ip extcommunity-list 10 permit rt 65000:100
```

This permits routes carrying RT `65000:100`.

The extended community list is configured globally.

It is not configured under BGP or an individual VRF.

### Step 2: Configure the RR Group

Under BGP:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family vpnv4
R1(config-router-af)# bgp rr-group 10
```

The argument `10` refers to extended community list 10.

**It is not a route reflector cluster ID or an independently assigned RR group identifier.**

The command causes the RR to use the referenced list for inbound VPNv4 route-target filtering.

### Command Syntax

```text
bgp rr-group <extcommunity-list-number>
```

| Parameter | Description |
|---|---|
| `bgp rr-group` | Enable route-target-based RR group filtering |
| `<extcommunity-list-number>` | Number of the referenced extended community list |

Cisco documents a supported list-number range of **1 through 500**.

Only one extended community list is referenced by an RR group.

However, that list can contain multiple entries.

### Removing an RR Group

Use:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family vpnv4
R1(config-router-af)# no bgp rr-group 10
```

This removes the RR-group filter.

It does not disable BGP or remove the route reflector client relationships.

Other BGP policies and route-target filtering mechanisms may still apply.

---

## Complete Configuration Example

Consider the following topology:

```text
                         AS 65000

                        R1 (RR)
                       /   |   \
                      /    |    \
                     /     |     \
                   PE1    PE2    PE3
                   R2     R3     R4
                    |      |      |
                  VPN A  VPN A  VPN B
                  RT 100 RT 100 RT 200
```

### Addressing

| Router | Loopback0 | Role |
|---|---|---|
| R1 | 1.1.1.1/32 | VPNv4 RR |
| R2 | 2.2.2.2/32 | PE1 |
| R3 | 3.3.3.3/32 | PE2 |
| R4 | 4.4.4.4/32 | PE3 |

All routers belong to AS 65000.

Assume:

- Loopback addresses are reachable through the provider IGP.
- The PE routers already have working MPLS L3VPN configurations.
- The PE routers advertise VPNv4 routes to R1.
- R1 is used as a control-plane route reflector and does not require local customer VRFs.

The example focuses on the RR configuration.

### R1: Configure the Extended Community List

```text
R1(config)# ip extcommunity-list 10 permit rt 65000:100
```

This permits VPNv4 routes belonging to VPN A.

### R1: Configure BGP

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
```

### R1: Configure the VPNv4 Address Family

```text
R1(config-router)# address-family vpnv4

R1(config-router-af)# bgp rr-group 10

R1(config-router-af)# neighbor 2.2.2.2 activate
R1(config-router-af)# neighbor 2.2.2.2 route-reflector-client
R1(config-router-af)# neighbor 2.2.2.2 send-community extended

R1(config-router-af)# neighbor 3.3.3.3 activate
R1(config-router-af)# neighbor 3.3.3.3 route-reflector-client
R1(config-router-af)# neighbor 3.3.3.3 send-community extended

R1(config-router-af)# neighbor 4.4.4.4 activate
R1(config-router-af)# neighbor 4.4.4.4 route-reflector-client
R1(config-router-af)# neighbor 4.4.4.4 send-community extended
```

### Configuration Explanation

The RR configuration performs three separate functions.

**1. VPNv4 MP-BGP Peering**

```text
neighbor <IP> activate
```

Activates the neighbor for VPNv4 route exchange.

**2. VPNv4 Route Reflection**

```text
neighbor <IP> route-reflector-client
```

Identifies the neighbor as an RR client.

The RR can reflect eligible VPNv4 routes between its clients.

**3. Route-Target Filtering**

```text
bgp rr-group 10
```

Restricts the VPNv4 routes accepted by the RR according to extended community list 10.

The `send-community extended` command enables transmission of extended communities, including RTs, to the neighbor.

The PE routers must also advertise the appropriate RT extended communities to the RR.

### Expected Result

Suppose the PEs advertise:

| Source | Prefix | RD | RT |
|---|---|---|---|
| PE1 | 10.10.10.0/24 | 65000:2 | 65000:100 |
| PE2 | 10.20.20.0/24 | 65000:3 | 65000:100 |
| PE3 | 10.30.30.0/24 | 65000:4 | 65000:200 |

R1 accepts:

```text
RD 65000:2, 10.10.10.0/24
RD 65000:3, 10.20.20.0/24
```

R1 filters:

```text
RD 65000:4, 10.30.30.0/24
```

Because the PE3 route is not accepted, R1 cannot reflect that route to the other PEs.

**The filter does not prevent PE3 from establishing a VPNv4 BGP session. It controls which routes R1 accepts from that session.**

---

## Permitting Multiple Route Targets

An RR group is not limited to a single RT.

The referenced extended community list can contain multiple permit entries.

For example, suppose R1 should accept routes belonging to both VPN A and VPN B:

```text
VPN A: RT 65000:100
VPN B: RT 65000:200
```

Configure:

```text
R1(config)# ip extcommunity-list 10 permit rt 65000:100
R1(config)# ip extcommunity-list 10 permit rt 65000:200
```

Then reference the list:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family vpnv4
R1(config-router-af)# bgp rr-group 10
```

Now R1 accepts VPNv4 routes matching either permitted RT.

### Example

| Route | RT | Accepted? |
|---|---|---|
| A | 65000:100 | Yes |
| B | 65000:200 | Yes |
| C | 65000:300 | No |
| D | 65000:400 | No |

The extended community list is evaluated using its permit and deny entries.

Routes that do not match any permit entry are implicitly denied.

### Routes with Multiple RTs

A VPNv4 route can carry more than one RT extended community.

For example:

```text
Prefix: 10.10.10.0/24
RTs:
  65000:100
  65000:300
```

Suppose the RR group permits:

```text
R1(config)# ip extcommunity-list 10 permit rt 65000:100
```

The route is accepted because it contains the permitted RT `65000:100`.

It does not need to contain only permitted RTs.

The presence of other RTs does not prevent a match against the permitted RT.

This is important in L3VPN designs that use multiple route targets for shared services or route leaking.

---

## RR Groups vs. RR Clusters

Despite their similar names, RR groups and RR clusters are different mechanisms.

### RR Clusters

An RR cluster is identified by a cluster ID.

For example:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp cluster-id 10.10.10.10
```

The cluster ID is used for route-reflection loop prevention through the CLUSTER_LIST attribute.

It can also affect BGP best-path tie-breaking.

### RR Groups

An RR group uses an extended community list to control accepted VPNv4 routes.

For example:

```text
R1(config)# ip extcommunity-list 10 permit rt 65000:100

R1(config)# router bgp 65000
R1(config-router)# address-family vpnv4
R1(config-router-af)# bgp rr-group 10
```

The RR group has no direct relationship to the RR's cluster ID.

### Comparison

| Feature | RR Cluster | RR Group |
|---|---|---|
| Primary purpose | Route reflection organization and loop prevention | VPNv4 route-target filtering |
| Identifier | Cluster ID | Extended community list number |
| Main attribute | CLUSTER_LIST | RT extended communities |
| Configuration | `bgp cluster-id` | `bgp rr-group` |
| Scope | Route reflection | VPNv4 route acceptance |
| Filters by RT | No | Yes |
| Prevents reflection loops | Yes | No |

A router can use both features simultaneously.

For example:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp cluster-id 10.10.10.10

R1(config-router)# address-family vpnv4
R1(config-router-af)# bgp rr-group 10
```

The cluster ID provides loop prevention.

The RR group provides route-target filtering.

These mechanisms operate independently.

---

## RR Groups vs. Default Route-Target Filtering

Cisco IOS also supports the following command:

```text
R1(config-router)# no bgp default route-target filter
```

This command is related to VPNv4 route retention, but it is not the same as `bgp rr-group`.

### Default Route-Target Filtering

Ordinary PE routers generally need to retain VPNv4 routes that are relevant to their locally configured VRFs.

Default route-target filtering can discard VPNv4 routes whose RTs do not match the import RTs of local VRFs.

For example, suppose a PE imports:

```text
RT 65000:100
```

It generally does not need unrelated routes carrying only:

```text
RT 65000:200
```

The command:

```text
no bgp default route-target filter
```

disables that default RT filtering behavior.

This is useful for devices such as route reflectors and ASBRs that must retain VPNv4 routes even when they have no local VRF importing those routes.

On Cisco IOS, configuring VPNv4 route reflector clients can also affect the normal default route-target filtering behavior.

### RR Group Filtering

The `bgp rr-group` command provides a separate, explicitly configured RT-based filter.

For example:

```text
R1(config)# ip extcommunity-list 10 permit rt 65000:100

R1(config)# router bgp 65000
R1(config-router)# address-family vpnv4
R1(config-router-af)# bgp rr-group 10
```

This specifies the RTs that the RR should accept regardless of whether it has a locally configured customer VRF.

### Comparison

| Feature | Default RT Filtering | RR Group |
|---|---|---|
| Based on RTs | Yes | Yes |
| Uses local VRF import RTs | Yes | No |
| Uses an extended community list | No | Yes |
| Main use | Retaining locally relevant VPN routes | Partitioning VPNv4 routes on an RR |
| Relevant command | `bgp default route-target filter` | `bgp rr-group` |

**Important:** `no bgp default route-target filter` is not a substitute for `bgp rr-group`.

The first disables a default filtering mechanism.

The second explicitly identifies permitted VPNv4 RTs.

---

## RR Groups vs. Route Target Constraint

Another important feature is **BGP Route Target Constraint (RTC)**, also called **RT-Constrain**.

RTC is defined in [RFC 4684](https://www.rfc-editor.org/rfc/rfc4684.html).

It allows BGP routers to advertise their interest in particular route targets.

This information can be used to limit the distribution of VPNv4 routes.

### RR Group Filtering

With `bgp rr-group`, the RR filters received VPNv4 routes.

For example:

```text
PE1 --------> RR
       VPNv4 UPDATE
       RT 65000:200

                 X
             RR-group filter
             permits only
             RT 65000:100
```

The RR rejects the route.

However, the PE may still send the VPNv4 UPDATE.

**RR-group filtering alone does not instruct the PE to stop advertising routes with unwanted RTs.**

### Route Target Constraint

RTC allows peers to exchange route-target membership information.

For example, the RR can indicate that it is interested in:

```text
RT 65000:100
```

A cooperating PE can then limit its VPNv4 advertisements according to that interest.

This can reduce unnecessary BGP UPDATE traffic as well as routing information maintained by the RR.

RTC uses a separate BGP address family:

```text
address-family rtfilter unicast
```

Support and configuration requirements depend on the platform and software.

### Comparison

| Feature | RR Group | RTC |
|---|---|---|
| Uses RT information | Yes | Yes |
| Requires extended community list | Yes | Not inherently |
| Filters received VPNv4 routes | Yes | Can prevent unwanted routes from being sent |
| Exchanges RT membership information | No | Yes |
| Uses RT-Constrain NLRI | No | Yes |
| Separate RT filtering address family | No | Yes |
| Requires cooperating peers for advertisement reduction | No | Yes |

Both mechanisms aim to improve VPN routing scalability, but they work differently.

For CCIE EI, the key distinction is:

- `bgp rr-group` applies **inbound RT-based filtering on the RR**.
- RTC can limit **which VPN routes are advertised between cooperating BGP speakers**.

---

## Verification

The most useful verification commands include:

```text
R1# show running-config | section router bgp
R1# show ip extcommunity-list
R1# show ip bgp vpnv4 all
R1# show ip bgp vpnv4 all summary
R1# show ip bgp vpnv4 all neighbors
```

Command availability and output formats can vary between IOS and IOS XE software versions.

### Verify the Extended Community List

Use:

```text
R1# show ip extcommunity-list 10
```

For example:

```text
R1# show ip extcommunity-list 10
Standard extended community-list 10
    permit RT:65000:100
    permit RT:65000:200
```

This verifies the RTs permitted by the extended community list.

### Verify RR Group Configuration

Use:

```text
R1# show running-config | section router bgp
```

Verify that the VPNv4 address family contains:

```text
address-family vpnv4
 bgp rr-group 10
```

Also verify that the expected neighbors are activated and configured as route reflector clients.

### Verify the VPNv4 BGP Table

Use:

```text
R1# show ip bgp vpnv4 all
```

This displays VPNv4 routing information retained by R1.

After applying an RR group permitting only `65000:100`, routes carrying only unrelated RTs should not be retained because of this filter.

For example:

| Prefix | RT | Expected in RR's VPNv4 Table |
|---|---|---|
| 10.10.10.0/24 | 65000:100 | Yes |
| 10.20.20.0/24 | 65000:100 | Yes |
| 10.30.30.0/24 | 65000:200 | No |

Other BGP policies and route-selection conditions can also affect the observed table.

### Verify Route Targets

To inspect a specific VPNv4 route:

```text
R1# show ip bgp vpnv4 all 10.10.10.0/24
```

Depending on platform and software, it may be preferable to specify the RD or use the corresponding `show bgp vpnv4 unicast` command.

Look for the route's extended community attributes, including its RTs.

For example:

```text
Extended Community: RT:65000:100
```

This confirms that the route carries the RT expected to match the RR-group filter.

### Verify Advertisements

Use:

```text
R1# show ip bgp vpnv4 all neighbors 3.3.3.3 advertised-routes
```

This helps determine whether an eligible VPNv4 route is being reflected to the specified neighbor.

A route filtered from the RR's VPNv4 table cannot be reflected by that RR.

### Troubleshooting Missing Routes

If an expected VPNv4 route is missing, verify:

1. The VPNv4 BGP session is Established.
2. The neighbor is activated under the VPNv4 address family.
3. The PE is advertising the expected VPNv4 route.
4. The route carries the correct RT extended community.
5. The referenced extended community list permits that RT.
6. The `bgp rr-group` command references the correct list.
7. Other inbound BGP policies are not rejecting the route.
8. The route is not rejected by ORIGINATOR_ID or CLUSTER_LIST loop prevention.
9. BGP next-hop reachability and best-path selection are functioning correctly.

Also verify that the route's RT is being propagated correctly by the advertising PE.

The RD and RT serve different purposes. An RD match does not satisfy an RT-based RR-group filter.

---

## Common Misconceptions

### 1. RR Groups Create BGP Neighbor Groups

False.

The `bgp rr-group` command does not assign neighbors to BGP peer groups or route reflector clusters.

It references an extended community list used to filter VPNv4 routes.

### 2. RR Groups Are Used for IPv4 Unicast Route Reflection

False.

The command is documented for the **VPNv4 address family**.

It is not the normal mechanism for filtering IPv4 unicast routes reflected between IBGP neighbors.

For ordinary IPv4 unicast filtering, use features such as route maps, prefix lists, and other BGP routing policies.

### 3. RR Groups Filter Routes Based on RD

False.

RR groups use **Route Targets**, not Route Distinguishers.

An RD makes a VPNv4 prefix unique.

An RT identifies the VPN membership information used for import/export and RT-based filtering.

### 4. Only One RT Can Be Permitted

False.

An RR group references one extended community list, but the list can contain multiple permit entries.

For example:

```text
ip extcommunity-list 10 permit rt 65000:100
ip extcommunity-list 10 permit rt 65000:200
```

Both RTs are permitted.

### 5. RR Groups Prevent PEs from Advertising Unwanted Routes

Not by themselves.

The PE can still send routes that the RR subsequently filters.

RTC or other supported advertisement-filtering mechanisms can reduce unnecessary VPNv4 UPDATE transmission.

### 6. An RR Must Have Local VRFs to Use RR-Group Filtering

False.

A dedicated RR can operate as a VPNv4 control-plane device without hosting the customer VRFs.

The RTs used by the RR-group filter are defined in the extended community list.

They do not need to be the import RTs of locally configured VRFs.

---

## Command Summary

| Command | Purpose |
|---|---|
| `ip extcommunity-list <N> permit rt <RT>` | Permit an RT in an extended community list |
| `bgp rr-group <N>` | Enable inbound VPNv4 RR-group filtering |
| `no bgp rr-group <N>` | Remove the RR-group filter |
| `address-family vpnv4` | Enter the VPNv4 BGP address family |
| `neighbor <IP> activate` | Activate a VPNv4 BGP neighbor |
| `neighbor <IP> route-reflector-client` | Configure a VPNv4 RR client |
| `neighbor <IP> send-community extended` | Send extended communities to a neighbor |
| `no bgp default route-target filter` | Disable default route-target filtering |
| `address-family rtfilter unicast` | Enter the RT-Constrain address family |
| `show ip extcommunity-list <N>` | Verify extended community list entries |
| `show ip bgp vpnv4 all` | Display retained VPNv4 routes |

---

## Key Points

- BGP RR groups provide RT-based inbound filtering for VPNv4 route reflectors.
- The primary command is `bgp rr-group <extcommunity-list-number>`.
- The command is configured under the VPNv4 address family.
- The referenced extended community list determines which RTs are permitted.
- The supported extended community list number range is 1 through 500.
- An RR group references one extended community list, which can contain multiple entries.
- A VPNv4 route matching a permitted RT can pass the RR-group filter.
- A route containing multiple RTs can be accepted if at least one RT matches a permit entry.
- Routes not permitted by the referenced extended community list are filtered.
- RR-group filtering can reduce the number of VPNv4 routes retained by an RR.
- RR groups are different from RR clusters, peer groups, and update groups.
- RR-group filtering does not replace ORIGINATOR_ID or CLUSTER_LIST loop prevention.
- The filtering is based on RTs, not RDs or IPv4 prefixes.
- RR-group filtering is different from default route-target filtering based on local VRF imports.
- `no bgp default route-target filter` disables default RT filtering; it does not configure an RR group.
- RR groups perform local inbound filtering and do not inherently prevent PEs from sending unwanted VPNv4 routes.
- RT-Constrain can reduce VPNv4 advertisements using exchanged RT membership information.
- Use `show ip extcommunity-list` and `show ip bgp vpnv4 all` to verify the filter and retained routes.

---

## References

- [Cisco: BGP Command Reference, bgp rr-group](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-a1.html)
- [Cisco: BGP Command Reference, ip extcommunity-list](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-c1.html)
- [Cisco: Connecting to a Service Provider Using External BGP, IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-external-sp-0.html)
- [RFC 4364: BGP/MPLS IP Virtual Private Networks](https://www.rfc-editor.org/rfc/rfc4364.html)
- [RFC 4684: Constrained Route Distribution for BGP/MPLS IP VPNs](https://www.rfc-editor.org/rfc/rfc4684.html)
