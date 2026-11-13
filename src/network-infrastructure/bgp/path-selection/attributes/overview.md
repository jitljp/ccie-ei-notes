# 1.5.b (i) Attributes

**BGP path attributes** provide information about routes and influence how BGP selects and advertises paths.

When a BGP router advertises a prefix, it includes path attributes in the BGP UPDATE message. These attributes describe characteristics such as the route's origin, the autonomous systems it has traversed, and its next-hop address.

BGP uses these attributes for:

- **Path selection:** Determining the preferred path to a destination.
- **Routing policy:** Influencing which paths are preferred or advertised.
- **Loop prevention:** Detecting routing loops using AS_PATH.
- **Next-hop resolution:** Identifying where traffic should be forwarded.
- **Traffic engineering:** Influencing inbound and outbound traffic paths.

Unlike OSPF and EIGRP, BGP does not calculate a single cumulative routing metric. Instead, it compares attributes in a defined order.

---

## Important BGP Path Attributes

The following table summarizes the most important attributes for CCIE Enterprise Infrastructure.

| Attribute | Purpose | Path Selection Preference |
|---|---|---|
| **Weight** | Cisco-specific value for local path selection | Highest |
| **LOCAL_PREF** | Influences path preference within an AS | Highest |
| **AS_PATH** | Records traversed autonomous systems and prevents loops | Shortest |
| **ORIGIN** | Indicates how the route was originated | IGP > EGP > Incomplete |
| **MED** | Suggests a preferred entry point into an AS | Lowest |
| **NEXT_HOP** | Identifies the next-hop address for the advertised prefix | Must be reachable |

> **Important:** Weight is a Cisco-specific local parameter, not an actual BGP path attribute carried in UPDATE messages.

Other important attributes include:

| Attribute | Purpose |
|---|---|
| **COMMUNITY** | Tags routes for routing policy |
| **EXTENDED COMMUNITY** | Carries additional policy information, including MPLS VPN route targets |
| **LARGE COMMUNITY** | Provides 12-byte community values supporting 4-byte ASNs |
| **ATOMIC_AGGREGATE** | Indicates that an aggregate may have lost some path information |
| **AGGREGATOR** | Identifies the AS and router that performed aggregation |
| **ORIGINATOR_ID** | Helps prevent routing loops with route reflectors |
| **CLUSTER_LIST** | Prevents routing loops between route reflector clusters |

Not every attribute directly participates in the default BGP best path selection algorithm. Some are primarily used for routing policies, aggregation, or loop prevention.

---

## BGP Attribute Categories

BGP path attributes are classified into four categories.

| Category | Must Be Recognized? | Required in Every Applicable UPDATE? |
|---|---|---|
| **Well-known mandatory** | Yes | Yes |
| **Well-known discretionary** | Yes | No, subject to specific protocol rules |
| **Optional transitive** | No | No |
| **Optional non-transitive** | No | No |

The distinction between **transitive** and **non-transitive** determines how an unrecognized optional attribute is handled:

- **Optional transitive:** An unrecognized attribute is preserved and propagated to other BGP peers when the route is advertised.
- **Optional non-transitive:** An unrecognized attribute is discarded rather than propagated.

Examples include:

| Attribute | Category |
|---|---|
| ORIGIN | Well-known mandatory |
| AS_PATH | Well-known mandatory |
| NEXT_HOP | Well-known mandatory (IPv4 unicast) |
| LOCAL_PREF | Well-known discretionary |
| ATOMIC_AGGREGATE | Well-known discretionary |
| MED | Optional non-transitive |
| COMMUNITY | Optional transitive |
| AGGREGATOR | Optional transitive |

> **Note:** The mandatory classification applies to traditional IPv4 unicast UPDATE messages containing reachable NLRI. Multiprotocol BGP uses attributes such as MP_REACH_NLRI to carry address-family-specific reachability and next-hop information.

See [BGP Attribute Categories](categories.md) for details.

---

## Attribute Scope and Propagation

Not all attributes have the same scope.

Some affect only the local router, while others are propagated within an AS or between autonomous systems.

| Attribute | Scope / Propagation |
|---|---|
| Weight | Local router only |
| LOCAL_PREF | Propagated within an AS through IBGP, not normally through EBGP |
| AS_PATH | Advertised through IBGP and EBGP; normally modified during EBGP advertisement |
| ORIGIN | Propagated through IBGP and EBGP |
| MED | Can be advertised through EBGP and IBGP; normally not propagated unchanged to another external AS |
| NEXT_HOP | Propagated through IBGP and EBGP; may be modified during advertisement |

This distinction is important when implementing routing policies.

For example:

- **Weight** can influence a single router's outbound traffic decisions.
- **LOCAL_PREF** can influence outbound path selection throughout an AS.
- **AS_PATH prepending** can influence how remote autonomous systems select paths toward your network.
- **MED** can influence how a neighboring AS selects among entry points into your AS.

The actual result depends on the receiving routers' routing policies and best path selection rules.

---

## BGP Attribute Comparison

BGP evaluates attributes in a specific order during best path selection.

For example, consider two paths to `192.0.2.0/24`:

| Attribute | Path A | Path B |
|---|---|---|
| Weight | 0 | 0 |
| LOCAL_PREF | 200 | 100 |
| AS_PATH | 65001 65002 65003 | 65004 |
| ORIGIN | IGP | IGP |
| MED | 100 | 0 |

Assuming both paths are valid and eligible:

1. Weight is equal, so BGP continues to LOCAL_PREF.
2. Path A has a higher LOCAL_PREF (200).
3. **Path A is selected as the best path.**

BGP does not compare AS_PATH length or MED because LOCAL_PREF already determines the preferred path.

The complete selection order is covered in [BGP Best Path Selection Algorithm](../best-path/overview.md).

---

## Verification

The following commands display BGP paths and their attributes:

```text
R1# show bgp ipv4 unicast
R1# show bgp ipv4 unicast 192.0.2.0/24
R1# show bgp ipv6 unicast
R1# show bgp ipv6 unicast 2001:db8::/32
```

The detailed prefix output is particularly useful for examining:

- Weight and LOCAL_PREF
- AS_PATH
- ORIGIN
- MED
- NEXT_HOP
- Communities
- Best path selection information

---

## Topics Covered

This section examines the following BGP attributes in detail:

1. [BGP Attribute Categories](categories.md): Well-known mandatory, well-known discretionary, optional transitive, and optional non-transitive attributes.
2. [Weight and Local Preference](weight-local-preference.md): Local and AS-wide path preference.
3. [AS_PATH](as-path.md): Path length, AS sequence, and loop prevention.
4. [Origin and MED](origin-med.md): Route origin and preferred entry points.
5. [NEXT_HOP](next-hop.md): Next-hop reachability and its impact on path selection.

---

## References

- [Cisco: Select BGP Best-path Algorithm](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html)
- [Cisco: Connecting to a Service Provider Using External BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-external-sp-0.html)
- [RFC 4271: A Border Gateway Protocol 4 (BGP-4)](https://www.rfc-editor.org/rfc/rfc4271.html)
