# 1.5.c (iv) Standard and Extended Communities

**BGP communities** are attributes that allow routes to be tagged with additional information, which can then be used to implement routing policies.

Instead of configuring policies that individually match numerous prefixes, administrators can assign communities to routes and apply policies based on those communities.

Common applications include:

- **Route filtering:** Permit or deny routes based on their communities.
- **Route preference:** Identify routes that should receive a particular LOCAL_PREF, MED, or other attribute.
- **Route advertisement control:** Restrict where routes can be advertised.
- **Traffic engineering:** Request specific routing policies from upstream service providers.
- **MPLS L3VPNs:** Control route import and export between VRFs using Route Targets.
- **Loop prevention:** Identify the originating site of VPN routes using Site-of-Origin communities.

A route can carry multiple communities, allowing different policies to be applied independently.

---

## BGP Community Types

There are three major categories of BGP communities:

| Type | Size per Community | BGP Attribute Code | Common Applications |
|---|---|---|---|
| Standard | 32 bits | 8 | Route tagging, filtering, advertisement control |
| Extended | 64 bits | 16 | Route Targets, Site-of-Origin, Cost Communities |
| Large | 96 bits | 32 | Route tagging with 4-byte ASNs |

The CCIE Enterprise Infrastructure blueprint specifically identifies **Standard and Extended Communities**.

### Standard Communities

Standard communities are 32-bit values.

They are typically represented using the following format:

```text
AA:NN
```

For example:

```text
65001:100
65001:200
65002:300
```

The two fields are each 16 bits:

- `AA`: Typically identifies the autonomous system that defines the community.
- `NN`: Identifies a particular routing policy or route category.

For example, AS 65001 might define:

| Community | Meaning |
|---|---|
| `65001:100` | Customer routes |
| `65001:200` | Preferred routes |
| `65001:300` | Backup routes |
| `65001:400` | Do not advertise to selected peers |

These meanings are **administratively defined**. BGP does not inherently understand their significance.

Standard communities also include several predefined **well-known communities**.

Examples include:

- `internet`
- `no-export`
- `no-advertise`
- `local-as`

See [Standard BGP Communities](standard.md).

### Extended Communities

Extended communities are 64-bit values that provide additional structure and functionality.

Unlike standard communities, extended communities include a **type field** that identifies how their values should be interpreted.

Common extended communities include:

| Extended Community | Purpose |
|---|---|
| Route Target (RT) | Controls the import and export of VPN routes between VRFs |
| Site-of-Origin (SoO) | Identifies the originating VPN site to help prevent routing loops |
| Cost Community | Influences BGP best-path selection at specific points in the algorithm |
| Link Bandwidth | Communicates bandwidth information used by supported BGP multipath features |

For example, a Route Target might be displayed as:

```text
RT:65001:100
```

A Cost Community might be displayed as:

```text
Cost:igp:10:100
```

These communities have different meanings and are processed by different features.

See [Extended BGP Communities](extended.md).

---

## Communities as BGP Path Attributes

Communities are carried in BGP UPDATE messages as **optional transitive path attributes**.

There are separate BGP path attributes for standard and extended communities:

| Attribute | Type Code | Classification |
|---|---|---|
| COMMUNITIES | 8 | Optional transitive |
| EXTENDED_COMMUNITIES | 16 | Optional transitive |

The optional classification means that BGP speakers are not required to support these attributes.

The transitive classification allows the attributes to be propagated through BGP speakers that do not recognize them.

However, **extended communities can individually be classified as transitive or nontransitive**.

For example:

- Route Targets are transitive extended communities.
- Cost Communities are nontransitive extended communities.

A nontransitive extended community is not propagated across ordinary AS boundaries.

### Communities and Best-Path Selection

Ordinary standard communities do not directly participate in the BGP best-path selection algorithm.

For example, assigning community `65001:200` to a route does not automatically make that route more preferred.

However, a router can match that community and modify an attribute such as LOCAL_PREF.

```text
BGP UPDATE
     |
     v
Community: 65001:200
     |
     v
Inbound Route Map
     |
     v
Match Community 65001:200
     |
     v
Set LOCAL_PREF 200
     |
     v
BGP Best-Path Selection
```

In this case, the community indirectly influences best-path selection through the LOCAL_PREF modification.

**Exception:** Certain extended communities, particularly Cost Communities, can directly influence Cisco's BGP best-path selection algorithm.

See [BGP Cost Communities and Points of Insertion](../../path-selection/best-path/cost-community.md).

---

## Well-Known Standard Communities

BGP defines several standard communities with predefined meanings.

| Community | Value (Hexadecimal) | Behavior |
|---|---|---|
| `internet` | `0x00000000` | General Internet community |
| `no-export` | `0xFFFFFF01` | Do not advertise outside the AS or BGP confederation |
| `no-advertise` | `0xFFFFFF02` | Do not advertise to any BGP peer |
| `local-as` | `0xFFFFFF03` | Do not advertise outside the local AS, including across confederation sub-AS boundaries |

The distinction between `no-export`, `no-advertise`, and `local-as` is particularly important.

### NO_EXPORT

A route carrying the `no-export` community cannot be advertised beyond the local AS or BGP confederation.

It can still be advertised:

- To IBGP peers.
- To EBGP peers within a BGP confederation.

It cannot be advertised to ordinary external EBGP peers.

### NO_ADVERTISE

A route carrying `no-advertise` cannot be advertised to any BGP peer.

This includes:

- IBGP peers.
- EBGP peers.
- Confederation EBGP peers.

The route can still be used locally, assuming it is otherwise valid and selected for installation.

### LOCAL_AS

The `local-as` community, also known as `NO_EXPORT_SUBCONFED`, is particularly relevant to BGP confederations.

It prevents routes from being advertised outside the local member AS of a confederation.

Unlike `no-export`, it also prevents advertisement to other member ASes within the same confederation.

**Important:** The `local-as` community is not the same as the `neighbor ... local-as` configuration command.

See [Standard BGP Communities](standard.md) for detailed examples of these behaviors.

---

## Community-Based Routing Policies

A fundamental advantage of communities is the ability to separate **route classification** from **policy enforcement**.

For example, consider an enterprise connected to two routers within the same AS.

```text
                   AS 65001

                R1 -------- R2
                 |          |
                 |          |
                ISP1       ISP2
              AS 65002   AS 65003
```

R1 receives routes from ISP1 and tags them with community `65001:100`.

R2 receives routes from ISP2 and tags them with community `65001:200`.

The routers can then apply policies based on these tags.

For example:

- Routes tagged `65001:100` receive LOCAL_PREF 200.
- Routes tagged `65001:200` receive LOCAL_PREF 100.

This makes ISP1 the preferred outbound path when the relevant routes are available through both providers and other applicable best-path criteria permit the comparison.

### Example: Tagging Routes

On R1, configure a route map that assigns a standard community to routes received from ISP1.

```text
R1(config)# ip bgp-community new-format

R1(config)# route-map TAG-ISP1 permit 10
R1(config-route-map)# set community 65001:100 additive

R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map TAG-ISP1 in
```

The `additive` keyword preserves existing standard communities while adding the new community.

Without `additive`, the configured `set community` values replace the existing standard community set.

### Example: Matching Communities

Configure a community list matching `65001:100`:

```text
R1(config)# ip community-list standard ISP1 permit 65001:100
```

Then configure a route map:

```text
R1(config)# route-map PREFER-ISP1 permit 10
R1(config-route-map)# match community ISP1
R1(config-route-map)# set local-preference 200

R1(config)# route-map PREFER-ISP1 permit 20
```

This route map:

1. Matches routes carrying community `65001:100`.
2. Assigns LOCAL_PREF 200 to those routes.
3. Permits other routes unchanged through sequence 20.

The route map can be applied inbound to an appropriate BGP neighbor.

**Important:** The community list does not modify the routes by itself. It identifies routes that can be matched by a routing policy.

The `permit` statement in the community list means that the community matches successfully. Whether the matched route is accepted, denied, or modified depends on the route-map sequence and how the route map is applied.

See [BGP Community Configuration and Filtering](configuration.md).

---

## Advertising Communities to Neighbors

By default, Cisco IOS XE does not advertise standard and extended communities to BGP neighbors unless community advertisement is enabled.

Use the `neighbor ... send-community` command under the appropriate BGP address family.

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 send-community
```

This enables standard community advertisement.

The following options are supported:

```text
R1(config-router-af)# neighbor 10.0.12.2 send-community standard
R1(config-router-af)# neighbor 10.0.12.2 send-community extended
R1(config-router-af)# neighbor 10.0.12.2 send-community both
```

| Command | Communities Advertised |
|---|---|
| `send-community` | Standard |
| `send-community standard` | Standard |
| `send-community extended` | Extended |
| `send-community both` | Standard and extended |

On IOS XE releases supporting Large Communities, the standard and `both` options also provide Large Community advertisement.

The command controls **outbound community advertisement** to the specified neighbor.

For example:

```text
R1(config-router-af)# neighbor 10.0.12.2 send-community both
```

This allows R1 to send standard and extended communities to `10.0.12.2`, subject to the community's propagation rules and applicable routing policies.

**Important:** Configuring `send-community` does not override a community's built-in propagation restrictions. For example, `no-export` routes are still prevented from crossing the appropriate AS boundary.

---

## Extended Communities in MPLS L3VPNs

One of the most important uses of extended communities is in **MPLS Layer 3 VPNs**.

MP-BGP uses Route Target extended communities to determine which VPN routes should be imported into particular VRFs.

Consider two PE routers:

```text
         MPLS Provider Network

       PE1 ---------------- PE2
        |                    |
       CE1                  CE2

       VRF A                VRF A
```

Both PE routers can use the following Route Target:

```text
65001:100
```

For example:

```text
PE1(config)# vrf definition A
PE1(config-vrf)# rd 65001:1
PE1(config-vrf)# route-target export 65001:100
PE1(config-vrf)# route-target import 65001:100
```

When PE1 exports VPN routes from VRF A into MP-BGP, the configured export Route Target is attached to those routes.

PE2 can then import the routes into a VRF configured with the corresponding import Route Target.

The Route Target is an **extended community**, not a standard community.

### Route Distinguisher vs. Route Target

Although Route Distinguishers (RDs) and Route Targets (RTs) often have similar-looking values, they serve different purposes.

| Feature | Route Distinguisher | Route Target |
|---|---|---|
| Size | 64 bits | 64 bits |
| BGP extended community? | No | Yes |
| Primary purpose | Make VPN prefixes unique | Control VPN route import/export |
| Carried as | Part of the VPN NLRI | Extended community attribute |
| Used for policy? | Not primarily | Yes |

A Route Distinguisher distinguishes otherwise identical customer prefixes.

A Route Target identifies which VPN routes are eligible for import into particular VRFs.

See [Extended BGP Communities](extended.md) for more details.

---

## Standard vs. Extended Communities

| Characteristic | Standard | Extended |
|---|---|---|
| Size | 32 bits | 64 bits |
| Path attribute code | 8 | 16 |
| Common representation | `65001:100` | `RT:65001:100` |
| Includes type information? | No | Yes |
| Well-known advertisement controls | Yes | Different extended-community types have their own semantics |
| MPLS VPN Route Targets | No | Yes |
| Site-of-Origin | No | Yes |
| Cost Community | No | Yes |
| Configuration command | `set community` | `set extcommunity` |
| Filtering mechanism | `ip community-list` | `ip extcommunity-list` |
| Advertisement control | `send-community standard` | `send-community extended` |

Although both types can be used for routing policies, they are **different BGP path attributes**.

For example:

```text
set community 65001:100
```

assigns a standard community.

```text
set extcommunity rt 65001:100
```

assigns a Route Target extended community.

These commands do not produce equivalent attributes, even though the numerical values look similar.

---

## Verification Commands

Cisco IOS XE provides several commands for examining BGP communities.

### Display Communities on a BGP Route

```text
R1# show bgp ipv4 unicast 192.0.2.0/24
```

The detailed route output can show standard and extended communities associated with the route.

### Display Routes Matching a Standard Community

```text
R1# show bgp ipv4 unicast community 65001:100
```

Display routes carrying the `no-export` community:

```text
R1# show bgp ipv4 unicast community no-export
```

### Display Community Lists

```text
R1# show ip community-list
R1# show ip extcommunity-list
```

### Display Routes Advertised to a Neighbor

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 advertised-routes
```

This is useful for checking whether routes are eligible for advertisement, particularly when well-known communities such as `no-export` or `no-advertise` are present.

For detailed attribute inspection, examine the route on the receiving neighbor as well.

### Verify Community Advertisement Configuration

```text
R1# show running-config | section router bgp
```

Verify that the appropriate `neighbor ... send-community` configuration is present.

### Reapply Routing Policies

After changing an inbound community-based routing policy:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

After changing an outbound policy:

```text
R1# clear ip bgp 10.0.12.2 soft out
```

These commands can reapply policies without resetting the established BGP session, assuming the necessary route-refresh or retained-route support is available.

---

## Key Points

- **Standard communities are 32-bit values**, while extended communities are 64-bit values.
- Communities allow BGP policies to classify and process groups of routes without matching every prefix individually.
- A route can carry multiple communities.
- Standard communities are optional transitive BGP path attributes.
- Extended communities use a separate optional transitive BGP path attribute, but individual extended-community types can be transitive or nontransitive.
- Ordinary standard communities do not directly influence best-path selection, but route maps can use them to modify attributes such as LOCAL_PREF.
- Cost Communities are extended communities that can directly influence BGP best-path selection.
- Well-known standard communities such as `no-export` and `no-advertise` impose specific advertisement restrictions.
- The `neighbor ... send-community` command controls whether communities are advertised to a neighbor.
- Route Targets are extended communities used for VPN route import and export.
- Route Distinguishers are not extended communities.

---

## Topics Covered

1. [Standard BGP Communities](standard.md): Community formats, well-known communities, route advertisement restrictions, and propagation.
2. [Extended BGP Communities](extended.md): Extended-community formats, Route Targets, Site-of-Origin, Cost Communities, and other applications.
3. [BGP Community Configuration and Filtering](configuration.md): Configuring, adding, replacing, deleting, matching, filtering, and verifying communities.

---

## References

- [Cisco: BGP Named Community Lists](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/configuration/xe-3e/irg-iproute-bgp-xe-3e-book/irg-named-community-list.html)
- [Cisco: Connecting to a Service Provider Using External BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-external-sp-0.html)
- [Cisco: BGP Frequently Asked Questions](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/5816-bgpfaq-5816.html)
- [RFC 1997: BGP Communities Attribute](https://datatracker.ietf.org/doc/html/rfc1997)
- [RFC 4360: BGP Extended Communities Attribute](https://datatracker.ietf.org/doc/html/rfc4360)
- [RFC 8092: BGP Large Communities Attribute](https://datatracker.ietf.org/doc/html/rfc8092)
