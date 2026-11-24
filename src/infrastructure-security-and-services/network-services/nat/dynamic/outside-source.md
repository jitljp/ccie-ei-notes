# Dynamic Outside Source NAT

**Dynamic outside source NAT** creates a temporary mapping between an **outside global** address and an **outside local** address. The router selects the outside local address from a NAT pool when matching traffic arrives from the outside network.

Unlike static outside source NAT, the mapping does not exist until traffic triggers its creation.

The translation is bidirectional:

- **Outside to inside:** Translate the source address from outside global to outside local.
- **Inside to outside:** Translate the destination address from outside local to outside global, using an existing mapping.

Outside source NAT is particularly useful for representing outside hosts using a different addressing scheme, including networks with overlapping addresses. However, dynamic outside source NAT is less suitable than static NAT when inside clients must initiate connections to predictable translated addresses before any outside traffic has arrived.

See [Static Outside Source NAT and PAT](../static/outside-source.md) for the equivalent static configuration.

---

## Configuration Syntax

```text
Router(config)# ip nat outside source list <acl> pool <pool-name> [add-route]
```

| Component | Purpose |
|---|---|
| `outside source` | Translate the address of an outside host |
| `list <acl>` | Select outside global source addresses eligible for translation |
| `pool <pool-name>` | Supply outside local addresses |
| `add-route` | Automatically install routes to dynamically allocated outside local addresses |

Compare this with dynamic inside source NAT:

| Feature | Inside source NAT | Outside source NAT |
|---|---|---|
| ACL matches (standard ACL) | Inside local source | Outside global source |
| Pool contains | Inside global addresses | Outside local addresses |
| Direction that creates the binding | Inside to outside | Outside to inside |
| Reverse translation | Destination on outside-to-inside packets | Destination on inside-to-outside packets |

**Do not reverse the NAT interface roles when configuring outside source NAT.** The inside and outside interfaces remain the same.

---

## Lab Topology

We will continue using the topology from the previous NAT lessons.

```text
       INSIDE NETWORK                         OUTSIDE NETWORK

  Local_Host_5 (LH5)
     10.1.1.5/24
          |
          |                   Local_Router                    ISP1
          +---- LAN ---------- E0/0    E0/1 ---------------- E0/1
          |                10.1.1.1   7.7.7.2              7.7.7.1
          |                NAT inside NAT outside              |
  Local_Host_6 (LH6)                                           |
     10.1.1.6/24                                   E0/0: 3.3.3.1/24
                                                               |
                                                          Outside LAN
                                                               |
                                                        +------+------+
                                                        |             |
                                                Remote_Host_7  Remote_Host_8
                                                 3.3.3.7/24    3.3.3.8/24
```

| Device | Interface or role | Address |
|---|---|---|
| LH5 | Inside host | `10.1.1.5/24` |
| LH6 | Inside host | `10.1.1.6/24` |
| Local_Router | E0/0, NAT inside | `10.1.1.1/24` |
| Local_Router | E0/1, NAT outside | `7.7.7.2/24` |
| ISP1 | E0/1 | `7.7.7.1/24` |
| ISP1 | E0/0 | `3.3.3.1/24` |
| RH7 | Outside host | `3.3.3.7/24` |
| RH8 | Outside host | `3.3.3.8/24` |
| NAT pool | Outside local addresses | `172.16.1.50–172.16.1.51` |

LH5 and LH6 use `10.1.1.1` as their default gateway. RH7 and RH8 use `3.3.3.1` as their default gateway.

Remove previous NAT rules and clear dynamic translations before beginning. Substitute the actual interface names on your router if necessary. The public address ranges in this isolated lab are not documentation prefixes.

---

## Configure Dynamic Outside Source NAT

### Step 1: Configure the NAT Interfaces

```text
Local_Router(config)# interface Ethernet0/0
Local_Router(config-if)# ip address 10.1.1.1 255.255.255.0
Local_Router(config-if)# ip nat inside
Local_Router(config-if)# no shutdown
!
Local_Router(config)# interface Ethernet0/1
Local_Router(config-if)# ip address 7.7.7.2 255.255.255.0
Local_Router(config-if)# ip nat outside
Local_Router(config-if)# no shutdown
```

### Step 2: Configure Routing

On Local_Router, configure an explicit route to the actual outside network:

```text
Local_Router(config)# ip route 3.3.3.0 255.255.255.0 7.7.7.1
```

For this exercise, remove any default route on Local_Router so that the effect of `add-route` is easier to observe.

On ISP1:

```text
ISP1(config)# interface Ethernet0/1
ISP1(config-if)# ip address 7.7.7.1 255.255.255.0
ISP1(config-if)# no shutdown
!
ISP1(config)# interface Ethernet0/0
ISP1(config-if)# ip address 3.3.3.1 255.255.255.0
ISP1(config-if)# no shutdown
!
ISP1(config)# ip route 10.1.1.0 255.255.255.0 7.7.7.2
```

**The route to the inside network on ISP1 is important.** Outside source NAT does not translate the inside hosts' addresses. Without this route (or an equivalent routing entry), RH7 and RH8 cannot reach LH5 and LH6 through ISP1.

### Step 3: Select Outside Hosts

```text
Local_Router(config)# ip access-list standard OUTSIDE-NAT
Local_Router(config-std-nacl)# permit 3.3.3.0 0.0.0.255
```

The ACL selects the original source addresses of packets arriving from outside. It does not match the inside network or the outside local pool.

A deny in a NAT classification ACL excludes traffic from that NAT rule; it is not itself a packet-filtering action.

### Step 4: Configure the Outside Local Pool

```text
Local_Router(config)# ip nat pool OUTSIDE-LOCAL 172.16.1.50 172.16.1.51 prefix-length 24
```

The pool contains two addresses. The prefix length describes the pool's subnet; it does not expand the range to the entire subnet.

These are **outside local** addresses. Neither RH7 nor RH8 configures these addresses on its interfaces.

### Step 5: Configure the Dynamic Mapping

```text
Local_Router(config)# ip nat outside source list OUTSIDE-NAT pool OUTSIDE-LOCAL add-route
```

The mapping associates the ACL with the outside local pool. The `add-route` keyword will provide the route required for reverse translation.

No inside source NAT or PAT is required for this exercise.

---

## Translation Creation

Before generating any matching traffic, check:

```text
Local_Router# show ip nat translations
Local_Router# show ip nat statistics
Local_Router# show ip route 172.16.1.50
```

The dynamic NAT rule exists, but there is no allocated outside source binding yet. With no other route covering the pool, there is also no route to `172.16.1.50`.

On RH7:

```text
Remote_Host_7# ping 10.1.1.5
```

Assume the generic pool allocates `172.16.1.50` to RH7. The sample outputs in this lesson are illustrative; verify the actual allocated address and flow identifiers on your router.

```text
Local_Router# show ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
--- ---                ---                172.16.1.50        3.3.3.7
```

The simple binding associates RH7's actual address (`3.3.3.7`) with its translated address (`172.16.1.50`). Additional extended flow entries may also be present. An extended entry does not imply that PAT is configured.

### Packet Flow: Outside to Inside

| Field | Before NAT | After NAT |
|---|---|---|
| Source | `3.3.3.7` | `172.16.1.50` |
| Destination | `10.1.1.5` | `10.1.1.5` |

1. RH7 sends the Echo Request toward LH5 through ISP1.
2. Local_Router receives the packet on its NAT outside interface.
3. The source matches the outside source NAT ACL.
4. NAT allocates an outside local address and creates the binding.
5. NAT translates the source to `172.16.1.50`.
6. Local_Router routes the packet to LH5.

LH5 sees the request as originating from `172.16.1.50`.

### Packet Flow: Inside to Outside

LH5 sends the Echo Reply to the translated source address of the request.

| Field | Before NAT | After NAT |
|---|---|---|
| Source | `10.1.1.5` | `10.1.1.5` |
| Destination | `172.16.1.50` | `3.3.3.7` |

Local_Router first selects an outside forwarding path for `172.16.1.50`, then uses the existing binding in reverse to translate the destination to `3.3.3.7`.

```text
RH7                     Local_Router                    LH5
 |                            |                          |
 | 3.3.3.7 -> 10.1.1.5        |                          |
 +--------------------------->|                          |
                              | 172.16.1.50 -> 10.1.1.5  |
                              +------------------------->|
                              |                          |
                              | 10.1.1.5 -> 172.16.1.50  |
                              |<-------------------------+
 | 10.1.1.5 -> 3.3.3.7        |                          |
 |<---------------------------+                          |
```

Only the outside host's address is translated.

---

## The `add-route` Keyword

Traditional NAT performs routing before translation for inside-to-outside traffic. Therefore, Local_Router needs a route to the **outside local address**, not just the outside global address.

With `add-route` configured, a route is generated when the dynamic binding is created.

For the example binding:

```text
Outside global: 3.3.3.7
Outside local:  172.16.1.50
```

The generated route is logically equivalent to:

```text
ip route 172.16.1.50 255.255.255.255 3.3.3.7
```

Verify it:

```text
Local_Router# show ip route 172.16.1.50
Local_Router# show ip route 3.3.3.7
```

Representative routing entries:

```text
S 172.16.1.50/32 [1/0] via 3.3.3.7
S 3.3.3.0/24 [1/0] via 7.7.7.1
C 7.7.7.0/24 is directly connected, Ethernet0/1
```

The route to `172.16.1.50` recursively resolves through the outside global address to ISP1. The translation then changes the destination to the actual outside address.

### Static vs. Dynamic `add-route`

| Mapping | When the corresponding host route is generated |
|---|---|
| Static outside source NAT with `add-route` | When the static host mapping is configured |
| Dynamic outside source NAT with `add-route` | When the dynamic address binding is created |

The dynamic route is associated with the binding. Verify that it is removed when the binding expires or is cleared. Other routes covering the same address can remain.

`add-route` does not advertise routes to other routers or configure ISP1's route to the inside network.

### Lab: Omit `add-route`

Clear dynamic translations, remove the original rule, and reconfigure it without `add-route`:

```text
Local_Router# clear ip nat translation *
Local_Router# configure terminal
Local_Router(config)# no ip nat outside source list OUTSIDE-NAT pool OUTSIDE-LOCAL add-route
Local_Router(config)# ip nat outside source list OUTSIDE-NAT pool OUTSIDE-LOCAL
```

Ensure Local_Router has no route covering the outside local pool (including a default route). Ping LH5 from RH7 again.

Expected behavior:

- RH7's request can create the dynamic binding and reach LH5.
- LH5's reply is addressed to the allocated outside local address.
- Local_Router lacks an appropriate pre-translation route for that address, preventing successful return forwarding.

Verify the NAT table and routing table separately. The presence of a translation does not guarantee reachability.

### Lab: Provide a Manual Route

With the dynamic mapping still configured without `add-route`, configure a route covering the pool:

```text
Local_Router(config)# ip route 172.16.1.0 255.255.255.0 7.7.7.1
```

Repeat the ping from RH7. The route provides the outside forwarding path for reply traffic, and NAT translates the destination using the existing binding.

A default route toward ISP1 can also provide this path in a single-egress topology. A more-specific route pointing inside would take precedence over that default route.

For multi-egress environments, `add-route` can be particularly useful because each generated route resolves via the actual outside global address.

**A manual route does not create a NAT binding.** It only provides forwarding information. Conversely, a binding alone does not replace routing.

See [NAT Packet Processing and Order of Operations](../order-of-operations.md) for the complete logical packet processing sequence.

---

## Multiple Outside Hosts and Pool Exhaustion

Restore the original mapping with `add-route` if desired, removing the manual pool route to isolate the test.

Generate traffic from RH7 and then RH8:

```text
Remote_Host_7# ping 10.1.1.5
Remote_Host_8# ping 10.1.1.5
```

Example bindings:

```text
Pro Inside global      Inside local       Outside local      Outside global
--- ---                ---                172.16.1.50        3.3.3.7
--- ---                ---                172.16.1.51        3.3.3.8
```

The two outside hosts have different outside local addresses.

Each address-only binding consumes an address from the pool. A third eligible outside host cannot obtain a new binding while both addresses are allocated. Existing bindings can continue to be used.

Verify using:

```text
Local_Router# show ip nat statistics
```

Look at the outside source mapping, pool size, allocated addresses, and allocation misses.

A single outside host can use its existing binding for communication with multiple inside hosts. A new outside local address is not required for every flow.

---

## Inside-Initiated Traffic

### Before a Binding Exists

Suppose LH5 attempts to ping `172.16.1.50` before any outside traffic has created a binding.

```text
Local_Host_5# ping 172.16.1.50
```

The NAT router has no association between `172.16.1.50` and a particular outside host. It cannot infer that the destination should be RH7 rather than RH8.

Even a manually configured route to the pool does not solve this problem.

### After a Binding Exists

After RH7 generates matching traffic, determine its allocated outside local address with `show ip nat translations`.

Assuming RH7 was assigned `172.16.1.50`:

```text
Local_Host_5# ping 172.16.1.50
```

The existing simple binding can be used in reverse for this new communication. It is not restricted to the original ICMP exchange.

However, the binding is dynamic. After it expires, the address can be reassigned to another outside host. Static outside source NAT is preferable when inside hosts require stable, predetermined outside local addresses.

### Traffic Sent Directly to the Outside Global Address

If LH5 sends traffic to `3.3.3.7` instead, the destination does not match an outside local address. A reply from RH7 may nevertheless undergo outside source NAT, causing the reply source to differ from the destination LH5 originally contacted.

Use the outside local address to test reverse translation.

---

## Predictable Address Allocation with `match-host`

A generic pool dynamically selects available addresses. It does not guarantee that RH7 will always be mapped to a particular outside local address.

If host-bit preservation is required, a `match-host` pool is an alternative:

```text
Local_Router(config)# ip nat pool MATCH-HOST 172.16.1.1 172.16.1.254 prefix-length 24 type match-host
Local_Router(config)# ip nat outside source list OUTSIDE-NAT pool MATCH-HOST add-route
```

Remove the preceding dynamic rule and clear its translations before applying this alternative.

The pool's `/24` prefix determines which bits are preserved:

| Outside global | Candidate outside local |
|---|---|
| `3.3.3.7` | `172.16.1.7` |
| `3.3.3.8` | `172.16.1.8` |

The candidate must be within the configured range and available. Unlike static network NAT, this does not preconfigure bindings for all outside hosts.

See [NAT Pool Allocation and Host Matching](pool-allocation.md) for details on pool types and host-bit calculations.

---

## Aging and Clearing Dynamic Bindings

Dynamic outside source bindings are subject to NAT aging. A simple binding can remain after individual extended flow entries expire.

```text
Local_Router# show ip nat translations verbose
Local_Router# show ip nat statistics
```

The general dynamic binding timeout can be adjusted with:

```text
Local_Router(config)# ip nat translation timeout 60
```

This is a global NAT setting, not a timer specific to the outside source rule. Active child translations can affect when the parent binding can be removed.

In an isolated lab, clear dynamic translations:

```text
Local_Router# clear ip nat translation *
```

Check the NAT table, pool allocation, and any routes created by `add-route` again. Clearing translations can interrupt active traffic. Stop matching traffic before testing, or entries may be recreated immediately.

For more information, see [NAT Translation Tables and Verification](../verification.md).

---

## Verification and Troubleshooting

```text
Local_Router# show running-config | include ip nat
Local_Router# show access-lists OUTSIDE-NAT
Local_Router# show ip nat translations
Local_Router# show ip nat translations verbose
Local_Router# show ip nat statistics
Local_Router# show ip route 172.16.1.50
Local_Router# show ip route 3.3.3.7
ISP1# show ip route 10.1.1.5
```

When necessary in a controlled lab, use:

```text
Local_Router# debug ip nat
Local_Router# undebug all
```

Generate traffic between the hosts, rather than originating the test ping on the NAT router itself, so the packet follows the intended inside/outside forwarding path.

| Symptom | Possible cause |
|---|---|
| No translation before traffic | Expected for dynamic NAT |
| RH7 sends traffic but no binding appears | ACL does not match outside global source, wrong interface role, or traffic never reaches NAT router |
| Binding exists but outside-initiated ping fails | Missing outside-local route for the reply, filtering, or host routing problem |
| LH5 cannot initiate traffic to a pool address | No existing binding, incorrect translated address, or missing route |
| Generated route is absent | No binding yet, `add-route` omitted, or unresolved outside global next hop |
| New outside host fails while existing hosts work | Pool exhausted |
| ISP1 cannot deliver traffic to LH5 | Missing route to inside network |
| Inside host ARPs for the outside local address | Pool overlaps its directly connected subnet; verify delivery to NAT router and outside forwarding route separately |

For the basic lab, the outside local pool is deliberately separate from the inside LAN, so LH5 and LH6 send packets for translated addresses to their default gateway rather than resolving them directly using ARP.

---

## Key Takeaways

- Dynamic outside source NAT maps **outside global** addresses to **outside local** addresses selected from a pool.
- The NAT ACL matches outside global source addresses, not inside addresses.
- Matching outside-to-inside traffic creates the dynamic binding.
- Outside-to-inside traffic undergoes source translation; inside-to-outside traffic undergoes reverse destination translation.
- Inside-initiated traffic can use an existing binding but cannot select an arbitrary outside host from an unallocated generic pool address.
- The outside network needs reachability to the inside hosts because outside source NAT does not translate their addresses.
- Inside-to-outside routing uses the outside local destination before NAT.
- `add-route` generates a route to the allocated outside local address via the outside global address when a binding is created.
- An appropriate existing route can be used instead of `add-route`.
- Dynamic bindings age out, and the addresses can be reused.

---

## References

- [Cisco: Configure IP NAT Outside Source List Command](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/13770-1.html)
- [Cisco IOS IP Addressing Services Command Reference: ip nat outside source](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr/command/ipaddr-cr-book/ipaddr-i3.html)
- [Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
