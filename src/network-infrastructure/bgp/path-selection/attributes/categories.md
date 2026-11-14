# BGP Attribute Categories

BGP path attributes are classified into four categories based on whether they must be recognized by BGP implementations and how they are handled during route advertisement.

The four categories are:

1. **Well-known mandatory**
2. **Well-known discretionary**
3. **Optional transitive**
4. **Optional non-transitive**

These categories determine whether an attribute must be supported, whether it must be included in an UPDATE message, and what happens if a router does not recognize it.

---

## Attribute Category Overview

| Category | Must Be Recognized? | Must Be Included? | Unrecognized Attribute Handling |
|---|---|---|---|
| Well-known mandatory | Yes | Yes, for conventional reachable NLRI | Protocol error |
| Well-known discretionary | Yes | Depends on attribute and context | Protocol error |
| Optional transitive | No | No, unless required by a specific feature | Preserve and propagate with Partial bit set |
| Optional non-transitive | No | No, unless required by a specific feature | Ignore and do not propagate |

**Well-known** means that all compliant BGP implementations must recognize the attribute.

**Optional** means that support for the attribute is not universally required.

**Transitive** and **non-transitive** determine how an unrecognized optional attribute is handled when a route is propagated.

---

## Well-Known Mandatory Attributes

**Well-known mandatory attributes** must be recognized by every BGP implementation and included in conventional BGP UPDATE messages that advertise reachable NLRI.

There are three well-known mandatory attributes:

| Attribute | Type Code | Purpose |
|---|---|---|
| ORIGIN | 1 | Identifies the origin of the route |
| AS_PATH | 2 | Records traversed autonomous systems |
| NEXT_HOP | 3 | Identifies the next-hop address |

### Important Characteristics

- All BGP implementations must recognize these attributes.
- They are required when advertising reachable IPv4 unicast NLRI using the traditional UPDATE format.
- Missing mandatory attributes constitute a protocol error.
- Their values may be modified according to BGP advertisement rules.

For example, when an EBGP router advertises a route to another AS, it normally prepends its AS number to the AS_PATH.

**Note:** An UPDATE message containing only withdrawn routes does not require these attributes.

Multiprotocol BGP uses different NLRI encoding rules, as explained later.

---

## Well-Known Discretionary Attributes

**Well-known discretionary attributes** must be recognized by every BGP implementation but are not required in every UPDATE message.

There are two well-known discretionary attributes:

| Attribute | Type Code | Purpose |
|---|---|---|
| LOCAL_PREF | 5 | Influences path preference within an AS |
| ATOMIC_AGGREGATE | 6 | Indicates that aggregation may have discarded path information |

### LOCAL_PREF

LOCAL_PREF influences which exit point an AS prefers when multiple paths exist.

Important characteristics:

- Advertised between IBGP peers.
- Required in IBGP UPDATE messages advertising routes.
- Not normally advertised to EBGP peers.
- Higher values are preferred.

Although LOCAL_PREF is classified as discretionary, **it is required in IBGP UPDATE messages containing reachable NLRI**.

### ATOMIC_AGGREGATE

ATOMIC_AGGREGATE indicates that a BGP aggregate may have lost some AS_PATH information during aggregation.

It is included when required by the aggregation rules, but is not required for ordinary route advertisements.

**Important:** Discretionary does not mean an attribute can always be omitted. Its presence may be mandatory in particular circumstances.

---

## Optional Transitive Attributes

**Optional transitive attributes** do not have to be recognized by every BGP implementation.

However, if a router receives an unrecognized optional transitive attribute, accepts the route, and advertises that route to another peer, it must preserve and propagate the attribute.

Common examples include:

| Attribute | Type Code | Purpose |
|---|---|---|
| AGGREGATOR | 7 | Identifies the router that performed aggregation |
| COMMUNITY | 8 | Tags routes for routing policies |
| EXTENDED_COMMUNITIES | 16 | Carries extended policy information |
| AS4_PATH | 17 | Preserves 4-byte AS_PATH information when interoperating with legacy speakers |
| LARGE_COMMUNITY | 32 | Supports 12-byte community values |

### Handling Unrecognized Attributes

Consider the following topology:

```text
R1 (AS 65001) --- R2 (AS 65002) --- R3 (AS 65003)
```

1. R1 advertises a route containing an optional transitive attribute.
2. R2 does not recognize the attribute.
3. R2 accepts the route and advertises it to R3.
4. R2 preserves the unrecognized attribute and sets its **Partial bit** to 1.
5. R3 receives the attribute even though R2 did not understand its meaning.

The Partial bit indicates that the attribute information is partial, for example because an intermediate router did not recognize it.

Once the Partial bit is set, subsequent routers must not clear it when propagating the attribute.

**Important:** Transitive does not mean that a route must always be advertised. Routing policies and normal BGP advertisement rules still apply.

---

## Optional Non-Transitive Attributes

**Optional non-transitive attributes** do not have to be recognized by every BGP implementation.

If a router receives an unrecognized optional non-transitive attribute, it ignores the attribute and does not propagate it.

Common examples include:

| Attribute | Type Code | Purpose |
|---|---|---|
| MULTI_EXIT_DISC (MED) | 4 | Suggests a preferred entry point into an AS |
| ORIGINATOR_ID | 9 | Prevents routing loops with route reflectors |
| CLUSTER_LIST | 10 | Prevents loops between route reflector clusters |
| MP_REACH_NLRI | 14 | Advertises multiprotocol reachability and next-hop information |
| MP_UNREACH_NLRI | 15 | Withdraws multiprotocol reachability |

### Handling Unrecognized Attributes

Consider the following topology:

```text
R1 (AS 65001) --- R2 (AS 65002) --- R3 (AS 65003)
```

1. R1 advertises a route containing an optional non-transitive attribute.
2. R2 does not recognize the attribute.
3. R2 ignores the attribute.
4. If R2 advertises the route to R3, the unrecognized attribute is not included.

**Important:** Non-transitive does not mean that a recognized attribute can never be propagated.

For example:

- MED can be propagated within an AS through IBGP.
- ORIGINATOR_ID and CLUSTER_LIST are propagated during route reflection.

Their propagation is governed by their individual protocol rules.

---

## BGP Attribute Flags

Each path attribute in a BGP UPDATE message contains:

- **Attribute Flags:** Describes properties of the attribute.
- **Attribute Type Code:** Identifies the specific attribute.
- **Attribute Length:** Specifies the length of the attribute value.
- **Attribute Value:** Contains the attribute's information.

The Attribute Flags field is one octet (8 bits).

```text
Bit:       0 1 2 3 4 5 6 7
          +-+-+-+-+-+-+-+-+
          |O|T|P|E|       |
          +-+-+-+-+-+-+-+-+
```

| Flag | Bit | Meaning |
|---|---|---|
| Optional (O) | 0 | 0 = Well-known, 1 = Optional |
| Transitive (T) | 1 | 0 = Non-transitive, 1 = Transitive |
| Partial (P) | 2 | 0 = Complete, 1 = Partial |
| Extended Length (E) | 3 | 0 = 1-byte length field, 1 = 2-byte length field |
| Reserved | 4-7 | Set to 0 when transmitted |

The bit numbers above follow the RFC convention, starting with the most significant bit.

### Common Flag Values

Assuming the Extended Length bit is 0:

| Flags (Hex) | Meaning |
|---|---|
| `0x40` | Well-known attribute |
| `0x80` | Optional non-transitive |
| `0xC0` | Optional transitive, complete |
| `0xE0` | Optional transitive, partial |

For example:

```text
0xC0 = 11000000

Optional   = 1
Transitive = 1
Partial    = 0
```

This identifies an optional transitive attribute with the Partial bit cleared.

**Note:** Attribute flags do not distinguish well-known mandatory from well-known discretionary attributes. That distinction is determined by the attribute's type and protocol specification.

---

## Important Exceptions

### Cisco Weight

**Weight is not a BGP path attribute.**

Weight is a Cisco-specific, locally significant value used in BGP best path selection.

It is not carried in BGP UPDATE messages and therefore does not belong to any of the four attribute categories.

See [Weight and Local Preference](weight-local-preference.md).

### Multiprotocol BGP

The mandatory ORIGIN, AS_PATH, and NEXT_HOP classification originates from conventional BGP-4 UPDATE messages.

Multiprotocol BGP introduces:

- **MP_REACH_NLRI:** Carries reachable NLRI and next-hop information.
- **MP_UNREACH_NLRI:** Carries withdrawn NLRI.

Both are classified as optional non-transitive attributes.

Although MP_REACH_NLRI is optional according to the general BGP attribute classification, it is required to advertise reachability using the multiprotocol extension.

For example, IPv6 unicast reachability and next-hop information are carried in MP_REACH_NLRI rather than the conventional IPv4 NLRI and NEXT_HOP fields.

---

## Verification

The following commands display BGP paths and their attributes:

```text
R1# show bgp ipv4 unicast
R1# show bgp ipv4 unicast 192.0.2.0/24
R1# show bgp ipv6 unicast
```

The detailed prefix output can display attributes such as:

- AS_PATH
- ORIGIN
- NEXT_HOP
- LOCAL_PREF
- MED
- Communities
- Aggregator information
- Route reflector attributes

However, these commands do not normally display every attribute's raw flags.

To inspect the actual Attribute Flags, Type Code, Length, and Value fields, capture a BGP UPDATE message and examine it in a protocol analyzer such as Wireshark.

---

## Key Takeaways

- **Well-known mandatory:** Must be recognized and included in conventional UPDATE messages advertising reachable NLRI.
- **Well-known discretionary:** Must be recognized, but inclusion depends on the attribute and context.
- **Optional transitive:** Unrecognized attributes are preserved and propagated with the Partial bit set when the route is advertised.
- **Optional non-transitive:** Unrecognized attributes are ignored and not propagated.
- **Weight:** Cisco-specific local parameter, not a BGP path attribute.
- **MP_REACH_NLRI:** Optional non-transitive by classification, but required when advertising multiprotocol reachability.

---

## References

- [RFC 4271: A Border Gateway Protocol 4 (BGP-4)](https://www.rfc-editor.org/rfc/rfc4271.html)
- [RFC 4760: Multiprotocol Extensions for BGP-4](https://www.rfc-editor.org/rfc/rfc4760.html)
- [RFC 4456: BGP Route Reflection](https://www.rfc-editor.org/rfc/rfc4456.html)
- [Cisco Press: Routing TCP/IP, Volume II](https://www.ciscopress.com/content/images/9781587054709/samplepages/9781587054709.pdf)
