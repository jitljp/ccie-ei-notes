# NAT Pool Allocation and Host Matching

A **NAT pool** defines a range of IPv4 addresses available to a NAT rule. The pool type determines how an address is selected from that range when a new translation is needed.

With ordinary dynamic inside source NAT (without `overload`), an inside local address is dynamically associated with an inside global address. The binding can be reused by multiple flows until it expires or is cleared.

This page focuses on **generic pools**, **host matching (`type match-host`)**, pool capacity, and pool options. For basic configuration and packet flow, see [Dynamic Inside Source NAT and Address Pools](inside-source.md).

---

## NAT Pool Types

| Pool type | Configuration | Address selection |
|---|---|---|
| **Generic** | Default; omit `type` | Allocate an available address from the pool |
| **Match-host** | `type match-host` | Preserve the host bits of the original address |
| **Rotary** | `type rotary` | Rotate through addresses for inside destination TCP load distribution |

A generic pool is the normal choice for dynamic inside source NAT. Match-host is useful when a predictable relationship between original and translated addresses is required, but the translations should still be created dynamically.

Rotary pools serve a different purpose and are covered in [Inside Destination NAT and Load Balancing](destination-nat.md). PAT is covered in [Dynamic PAT and NAT Overload](pat.md).

---

## Lab Topology

We'll continue using the NAT topology from the previous lessons.

```text
     INSIDE LAN                OUTSIDE TRANSIT                 REMOTE LAN
     10.1.1.0/24                 7.7.7.0/24                   3.3.3.0/24

 LH5 10.1.1.5 --+                                            +-- RH7 3.3.3.7
                |                                            |
                SW--[ Local_Router ]--------[ ISP1 ]--------SW
                |    E0/0       E0/1       E0/1   E0/0       |
 LH6 10.1.1.6 --+  10.1.1.1    7.7.7.2    7.7.7.1 3.3.3.1    +-- RH8 3.3.3.8
                    inside     outside
```

Local_Router's `Ethernet0/0` is configured with `ip nat inside`, and `Ethernet0/1` with `ip nat outside`. Local_Router has a default route toward `7.7.7.1`. The inside and remote hosts use their respective local routers as default gateways.

---

## Generic NAT Pools

Configure a pool of two inside global addresses:

```text
Local_Router(config)# ip nat pool PUBLIC 7.7.7.50 7.7.7.51 prefix-length 24
Local_Router(config)# access-list 10 permit 10.1.1.0 0.0.0.255
Local_Router(config)# ip nat inside source list 10 pool PUBLIC
```

The ACL identifies eligible inside source addresses. The pool provides the translated addresses. The prefix length describes the pool's network; it does **not** expand the allocation range to the entire /24.

The range is inclusive: `7.7.7.50` and `7.7.7.51` are the only addresses explicitly included.

### Address Allocation on Demand

Generate traffic from LH5, followed by LH6:

```text
LH5# ping 3.3.3.7
LH6# ping 3.3.3.7
```

An example of the resulting simple entries is:

```text
Local_Router# show ip nat translations
Pro  Inside global      Inside local       Outside local      Outside global
---  7.7.7.50           10.1.1.5           ---                ---
---  7.7.7.51           10.1.1.6           ---                ---
```

Protocol-specific child entries may also be present. These example outputs are illustrative, not live captures.

The first host to create a binding can receive the first available address. A generic pool does not require that the last octet of the inside global address match the last octet of the inside local address. Do not rely on a particular allocation order as a permanent mapping.

### Reusing a Binding

Now generate traffic from LH5 toward RH8:

```text
LH5# ping 3.3.3.8
```

LH5 continues to use its existing inside global address. It does not require another pool address merely because the outside destination changes.

The distinction is between an **address binding** and an individual **flow translation**. Multiple child entries can refer to one simple parent binding.

### Reallocation After Clearing

Stop ongoing traffic and clear dynamic translations:

```text
Local_Router# clear ip nat translation *
```

Then initiate traffic from LH6 before LH5. Examine the new bindings. A generic pool does not guarantee the original assignment after a binding is removed.

**Warning:** Clearing all dynamic NAT translations can disrupt active connections. Use this command in an isolated lab or during an appropriate maintenance window.

---

## Pool Capacity and Exhaustion

Without `overload`, each simultaneously allocated ordinary one-to-one binding requires a distinct global address.

For the two-address pool above, add a third inside test host, LH9, with address `10.1.1.9/24` and gateway `10.1.1.1`.

After LH5 and LH6 have obtained bindings, initiate traffic from LH9:

```text
LH9# ping 3.3.3.7
```

The ACL permits LH9, but the generic pool has no available address. The router cannot create the required translation. Existing bindings can continue to be used.

An ACL match does not guarantee successful allocation. Nor does a short-lived application session guarantee that its parent NAT binding has been released.

Use:

```text
Local_Router# show ip nat statistics
Local_Router# show ip nat translations verbose
```

Inspect the pool's total and allocated address counts and its allocation misses. A pool miss is not the same as an ordinary translation-table lookup miss: a new flow can miss the translation table and subsequently be translated successfully.

Once an existing binding and its dependent entries are removed, the freed address can be assigned to another eligible host. See [NAT Translation Tables and Verification](../verification.md) for aging timers and selective clearing.

---

## Host Matching: `type match-host`

A **match-host pool** preserves the host portion of the original IPv4 address when selecting the translated address. The pool's configured mask determines which bits are host bits.

The conceptual operation is:

```text
Translated address = pool network bits + original address host bits
```

Equivalently, using bitwise operations:

```text
Host mask          = inverse of pool netmask
Host bits          = original address AND host mask
Pool network bits  = pool address AND pool netmask
Candidate address  = pool network bits OR host bits
```

This calculation identifies a candidate, not an unconditional allocation. The candidate must be within the configured range and available for the required binding.

### Configuration Example: /24 Pool

Replace the previous rule and pool after stopping traffic and clearing dynamic state:

```text
Local_Router# clear ip nat translation *
Local_Router(config)# no ip nat inside source list 10 pool PUBLIC
Local_Router(config)# no ip nat pool PUBLIC
Local_Router(config)# ip nat pool MATCH 7.7.7.5 7.7.7.20 prefix-length 24 type match-host
Local_Router(config)# ip nat inside source list 10 pool MATCH
```

ACL 10 remains unchanged. Generate traffic from LH6 first, then LH5.

Expected address bindings:

```text
Pro  Inside global      Inside local       Outside local      Outside global
---  7.7.7.5            10.1.1.5           ---                ---
---  7.7.7.6            10.1.1.6           ---                ---
```

The host bits are the last eight bits because the pool has a /24 mask. LH5 maps to `7.7.7.5`, and LH6 maps to `7.7.7.6`, independently of which host initiates traffic first, assuming both addresses are available.

### Match-Host Does Not Mean Match the Last Octet

The host field is defined by the pool mask, not by octet boundaries or the source host's configured subnet mask.

Consider this alternative pool:

```text
Local_Router(config)# ip nat pool MATCH26 7.7.7.65 7.7.7.126 prefix-length 26 type match-host
```

The pool belongs to `7.7.7.64/26`. Six bits constitute the host field.

| Original address | Low six bits (decimal) | Candidate inside global |
|---|---:|---|
| `10.1.1.5` | 5 | `7.7.7.69` |
| `10.1.1.6` | 6 | `7.7.7.70` |
| `10.1.1.70` | 6 | `7.7.7.70` |
| `10.1.1.130` | 2 | `7.7.7.66` |

For example, the final octet of `10.1.1.70` is `01000110` in binary. Its low six bits are `000110` (6). Combining these with the pool network bits gives `01000110` (70).

Notice that two different inside local addresses can yield the same candidate. Match-host does not guarantee that every eligible inside local address has a unique translated address.

### Candidate Outside the Pool Range

Consider a /24 match-host pool restricted to `7.7.7.50` through `7.7.7.60`:

```text
ip nat pool LIMITED 7.7.7.50 7.7.7.60 prefix-length 24 type match-host
```

| Inside local | Required candidate | Included in pool? |
|---|---|---|
| `10.1.1.5` | `7.7.7.5` | No |
| `10.1.1.50` | `7.7.7.50` | Yes |
| `10.1.1.60` | `7.7.7.60` | Yes |
| `10.1.1.61` | `7.7.7.61` | No |

An address elsewhere in the /24 is not automatically added to the pool. A host whose required candidate is outside the range cannot obtain a host-preserving binding from this pool, even if other addresses remain unused.

### Host-Bit Collisions

Using the /26 example, `10.1.1.6` and `10.1.1.70` require the same inside global address (`7.7.7.70`). With ordinary non-overloaded NAT, both cannot hold independent one-to-one bindings to that same address simultaneously.

This issue can also arise with a /24 pool when an ACL permits multiple inside /24 networks, such as `10.1.1.0/24` and `10.1.2.0/24`. Both `.5` hosts would require the same global address.

Design the ACL and pool mask so that eligible hosts have unique host fields and their candidates fall within the configured allocation range. Do not assume a generic-pool fallback for a failed host-preserving allocation.

---

## Match-Host vs. Static Network NAT

Both mechanisms can preserve host bits, but they differ in how bindings are established.

| Feature | Match-host pool | Static network NAT |
|---|---|---|
| Configuration | Dynamic NAT rule referencing a match-host pool | Explicit network-to-network static mapping |
| Host bits preserved | Yes | Yes |
| ACL selects eligible sources | Yes, for ACL-based inside source NAT | Not required for basic mapping |
| Address binding | Allocated when qualifying traffic needs a translation | Defined by the static network rule |
| Dynamic binding aging | Yes | Configured network mapping does not age out |
| Outside-initiated traffic before inside activity | No dynamically allocated binding yet | Static mapping exists; routing and ARP must also work |

Static network NAT can also create individual translation state and aliases on demand. Do not confuse the existence of its configured mapping with the existence of every possible per-host entry or ARP alias.

See [Static Network-to-Network NAT](../static/network-mapping.md).

---

## Verification and Troubleshooting

```text
Local_Router# show running-config | include ip nat
Local_Router# show access-lists 10
Local_Router# show ip nat statistics
Local_Router# show ip nat translations
Local_Router# show ip nat translations verbose
Local_Router# show ip aliases
```

In `show ip nat statistics`, examine the dynamic mapping, referenced pool, configured range, pool type, total addresses, allocated addresses, and misses. Exact output formatting varies.

| Symptom | What to check |
|---|---|
| No binding for an inside host | NAT interface roles, routing, ACL, and matching traffic |
| Pool fully allocated | Active parent bindings and aging timers |
| Match-host fails despite unused addresses | Required host bits, configured range, and collisions |
| Unexpected translated address | Generic vs. match-host pool; existing bindings; competing NAT rules |
| Binding exists but replies fail | Upstream route, ARP resolution, aliases, and return path |
| Pool cannot be modified or removed | Active translations and references to the pool |

When changing the allocation method, stop test traffic, clear relevant dynamic state, remove the referencing rule if necessary, modify or recreate the pool, and reapply the rule. Generate fresh traffic to verify the new behavior.

---

## Key Takeaways

- Generic pools dynamically select available addresses; they do not provide a permanent host-to-address correspondence.
- A pool's start and end addresses define its allocation range; its mask defines the network and host fields.
- `type match-host` preserves host bits using the pool mask, not necessarily the last octet.
- The calculated candidate must be in the pool and available. Host-bit collisions can prevent allocation even when other addresses are free.
- Match-host NAT is dynamic, not static network NAT.
- An ordinary non-overloaded binding consumes an address, not one address per flow.
- `type rotary` is used for inside destination TCP load distribution.
- Inspect both pool statistics and translation state when troubleshooting allocation problems.

---

## References

- [Cisco IOS IP Addressing Services Command Reference: ip nat pool](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr/command/ipaddr-cr-book/ipaddr-i3.html)
- [Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- [Cisco: Network Address Translation (NAT) Frequently Asked Questions](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/26704-nat-faq-00.html)
