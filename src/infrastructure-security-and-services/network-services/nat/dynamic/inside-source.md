# Dynamic Inside Source NAT and Address Pools

**Dynamic inside source NAT** allocates an **inside global** IPv4 address from a configured pool when an inside host sends traffic that matches a NAT rule.

Unlike static inside source NAT, the administrator specifies a range of available addresses rather than a fixed mapping for each host.

Without the `overload` keyword, dynamic inside source NAT creates temporary **one-to-one address bindings**. Each inside local address with an active binding occupies one inside global address. Multiple connections from that inside host can use the same binding, but another inside host cannot simultaneously use that allocated global address under ordinary non-overloaded NAT.

Dynamic inside source NAT can conserve addresses when only a subset of inside hosts needs external connectivity at any given time. It does not provide the many-to-one address sharing associated with PAT.

---

## Configuration Syntax

Three components are required in addition to NAT interface roles and routing:

1. A **NAT pool** defines the available inside global addresses.
2. An **ACL** identifies the inside addresses eligible for translation.
3. An **inside source NAT rule** associates the ACL with the pool.

```text
ip nat pool <name> <start-ip> <end-ip> {netmask <mask> | prefix-length <length>}
ip access-list standard <acl-name>
 permit <inside-network> <wildcard-mask>
ip nat inside source list <acl-name> pool <name>
```

For example:

```text
Local_Router(config)# ip nat pool PUBLIC 7.7.7.50 7.7.7.51 prefix-length 24
Local_Router(config)# ip access-list standard NAT_INSIDE
Local_Router(config-std-nacl)# permit 10.1.1.0 0.0.0.255
Local_Router(config-std-nacl)# exit
Local_Router(config)# ip nat inside source list NAT_INSIDE pool PUBLIC
```

The ACL selects inside local addresses, whereas the pool supplies inside global addresses. The NAT rule connects these two components.

**Do not configure `overload` for this exercise.** Adding it changes the translation mechanism to dynamic PAT, allowing multiple inside hosts to share a global address using protocol-specific identifiers.

---

## Lab Topology

We will continue using the topology from [Static Inside Source NAT](../static/inside-source.md).

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

  NAT pool PUBLIC: 7.7.7.50 - 7.7.7.51
```

| Device | Address | Default gateway / role |
|---|---|---|
| LH5 | `10.1.1.5/24` | `10.1.1.1` |
| LH6 | `10.1.1.6/24` | `10.1.1.1` |
| Local_Router E0/0 | `10.1.1.1/24` | NAT inside |
| Local_Router E0/1 | `7.7.7.2/24` | NAT outside |
| ISP1 E0/1 | `7.7.7.1/24` | Toward Local_Router |
| ISP1 E0/0 | `3.3.3.1/24` | Toward remote hosts |
| RH7 | `3.3.3.7/24` | `3.3.3.1` |
| RH8 | `3.3.3.8/24` | `3.3.3.1` |

The pool contains two addresses. Although the ACL permits the entire `10.1.1.0/24` subnet, only two distinct inside local addresses can simultaneously hold bindings from this pool.

Before starting, remove NAT rules and dynamic translations from the previous exercises so they do not interfere with the results. In particular, remove the previous static mapping for LH5 if it is still configured.

---

## Configure Dynamic Inside Source NAT

### Step 1: Configure Interface Roles and Routing

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
!
Local_Router(config)# ip route 0.0.0.0 0.0.0.0 7.7.7.1
```

On ISP1:

```text
ISP1(config)# interface Ethernet0/1
ISP1(config-if)# ip address 7.7.7.1 255.255.255.0
ISP1(config-if)# no shutdown
!
ISP1(config)# interface Ethernet0/0
ISP1(config-if)# ip address 3.3.3.1 255.255.255.0
ISP1(config-if)# no shutdown
```

ISP1 does not need a route to `10.1.1.0/24` for translated traffic. It does need to deliver packets addressed to the pool's inside global addresses to Local_Router.

### Step 2: Define the NAT Pool

```text
Local_Router(config)# ip nat pool PUBLIC 7.7.7.50 7.7.7.51 netmask 255.255.255.0
```

Alternatively, specify the prefix length:

```text
Local_Router(config)# ip nat pool PUBLIC 7.7.7.50 7.7.7.51 prefix-length 24
```

These are alternative forms of the same configuration, not two separate pools.

| Argument | Value | Purpose |
|---|---|---|
| Pool name | `PUBLIC` | Identifies the pool in NAT rules |
| Start IP | `7.7.7.50` | First allocatable address |
| End IP | `7.7.7.51` | Last allocatable address |
| Netmask or prefix length | `/24` | Subnet information for the pool |

A pool does not need to be part of the outside interface's subnet. A separate routed prefix can also supply global addresses, as demonstrated later.

### Step 3: Select Inside Local Addresses

```text
Local_Router(config)# ip access-list standard NAT_INSIDE
Local_Router(config-std-nacl)# permit 10.1.1.0 0.0.0.255
```

The ACL matches the original source address of traffic being considered for inside source NAT. It does not match the pool addresses.

A numbered standard ACL is also valid:

```text
Local_Router(config)# access-list 10 permit 10.1.1.0 0.0.0.255
```

Use one ACL or the other in the NAT rule.

### Step 4: Associate the ACL with the Pool

```text
Local_Router(config)# ip nat inside source list NAT_INSIDE pool PUBLIC
```

Or, if using the numbered ACL:

```text
Local_Router(config)# ip nat inside source list 10 pool PUBLIC
```

No ACL needs to be applied to an interface using `ip access-group` for this NAT rule to operate. The NAT rule itself references the ACL.

### Complete NAT Router Configuration

```text
interface Ethernet0/0
 ip address 10.1.1.1 255.255.255.0
 ip nat inside
 no shutdown
!
interface Ethernet0/1
 ip address 7.7.7.2 255.255.255.0
 ip nat outside
 no shutdown
!
ip access-list standard NAT_INSIDE
 permit 10.1.1.0 0.0.0.255
!
ip nat pool PUBLIC 7.7.7.50 7.7.7.51 prefix-length 24
ip nat inside source list NAT_INSIDE pool PUBLIC
ip route 0.0.0.0 0.0.0.0 7.7.7.1
```

---

## Translation Creation and Reuse

### Before Matching Traffic

```text
Local_Router# show ip nat translations
```

With no other NAT rules or active traffic, the translation table is empty. Unlike a static mapping, the dynamic rule defines how to allocate a binding, not a preconfigured address pair.

```text
Local_Router# show ip nat statistics
```

The dynamic mapping and pool should be listed even when no addresses have been allocated. This distinction is useful when troubleshooting an empty translation table.

### LH5 Initiates Traffic

```text
LH5# ping 3.3.3.7
```

Local_Router receives the ICMP Echo Request on its inside interface and determines that the packet will exit its outside interface. The source matches `NAT_INSIDE`. If no binding exists, NAT allocates an available address from `PUBLIC`.

Assume LH5 receives `7.7.7.50`:

| Field | Before NAT | After NAT |
|---|---|---|
| Source IP | `10.1.1.5` | `7.7.7.50` |
| Destination IP | `3.3.3.7` | `3.3.3.7` |
| ICMP identifier | Unchanged | Unchanged |

The Echo Reply from RH7 is addressed to `7.7.7.50`. Local_Router translates its destination to `10.1.1.5` before routing it toward LH5.

```text
                     Local_Router
LH5                       NAT                         RH7
10.1.1.5                   |                        3.3.3.7
   |  Src 10.1.1.5         |  Src 7.7.7.50             |
   |  Dst 3.3.3.7          |  Dst 3.3.3.7              |
   +---------------------->+-------------------------->|
   |                       |                           |
   |  Src 3.3.3.7          |  Src 3.3.3.7              |
   |  Dst 10.1.1.5         |  Dst 7.7.7.50             |
   |<----------------------+<--------------------------+
```

NAT updates relevant checksums as necessary. Ordinary address-only translation does not deliberately remap TCP/UDP ports or ICMP identifiers to multiplex multiple inside hosts.

### LH5 Communicates with Another Outside Host

```text
LH5# ping 3.3.3.8
```

LH5 can use the same allocated inside global address for traffic to RH8. The router does not need to allocate a second pool address merely because the destination has changed.

### LH6 Initiates Traffic

```text
LH6# ping 3.3.3.7
```

LH6 receives the remaining address, `7.7.7.51`.

| Inside local | Inside global |
|---|---|
| `10.1.1.5` | `7.7.7.50` |
| `10.1.1.6` | `7.7.7.51` |

---

## Simple Bindings and Extended Translations

Dynamic NAT without overload can maintain an address-level binding and protocol-specific sessions associated with it.

Example translation table after the pings:

```text
Local_Router# show ip nat translations
Pro  Inside global      Inside local       Outside local      Outside global
icmp 7.7.7.50:100       10.1.1.5:100       3.3.3.7:100        3.3.3.7:100
icmp 7.7.7.50:101       10.1.1.5:101       3.3.3.8:101        3.3.3.8:101
---  7.7.7.50           10.1.1.5           ---                ---
icmp 7.7.7.51:200       10.1.1.6:200       3.3.3.7:200        3.3.3.7:200
---  7.7.7.51           10.1.1.6           ---                ---
```

The output is illustrative; identifiers, ordering, and displayed session entries vary with traffic and software.

The `---` rows represent **simple bindings**, also called **half-entries**. The ICMP rows represent **extended translations** associated with individual flows.

```text
         Simple binding (one pool address)
              10.1.1.5 <-> 7.7.7.50
                        |
              +---------+---------+
              |                   |
         Flow to RH7         Flow to RH8
```

The two LH5 flows consume one pool address, not two. An extended translation alone is not evidence that PAT is enabled. In the example the inside local and inside global identifiers are identical.

The outside local and outside global addresses are also identical because this configuration does not translate the outside hosts' addresses.

See [NAT Translation Tables and Verification](../verification.md) for further details.

---

## Pool Exhaustion

When both addresses are allocated, a third inside local address cannot establish a new ordinary one-to-one binding from this pool.

To demonstrate, add LH9 with address `10.1.1.9/24` and default gateway `10.1.1.1`. Ensure LH5 and LH6 still have bindings, then ping RH7 from LH9:

```text
LH9# ping 3.3.3.7
```

The ACL permits LH9, but NAT cannot allocate an address. With this configuration alone, the new matching traffic fails to be translated and forwarded successfully. Existing bindings remain usable.

```text
Local_Router# show ip nat statistics
```

Inspect the pool's total, allocated, and miss counters. An illustrative subset of the output is:

```text
Dynamic mappings:
-- Inside Source
[Id: 1] access-list NAT_INSIDE pool PUBLIC refcount 5
 pool PUBLIC: netmask 255.255.255.0
        start 7.7.7.50 end 7.7.7.51
        type generic, total addresses 2, allocated 2 (100%), misses 1
```

The reference count and miss counter are examples, not guaranteed values. Pool occupancy is distinct from the number of translation rows and from the overall NAT lookup miss counter.

Possible solutions include increasing the pool size, freeing unused bindings, or configuring PAT if address sharing is acceptable. Do not simply increase the range to addresses that are already in use or are not routed to the NAT device.

Detailed allocation behavior is covered in [NAT Pool Allocation and Host Matching](pool-allocation.md).

---

## Binding Lifetime and Clearing Translations

A dynamic address binding is not released immediately when an application finishes or a ping command stops. It remains until it is eligible for removal under NAT aging rules or is cleared administratively.

The general dynamic address-binding timeout defaults to **24 hours (86,400 seconds)**. Protocol-specific child translations have separate aging timers. A binding cannot normally age out while active children still depend on it.

For an isolated lab, shorten the binding and ICMP timers to observe reuse more quickly:

```text
Local_Router(config)# ip nat translation timeout 60
Local_Router(config)# ip nat translation icmp-timeout 10
```

Stop traffic, including outside-initiated test traffic, and monitor the entries:

```text
Local_Router# show ip nat translations verbose
Local_Router# show ip nat statistics
```

Create fresh translations after changing timers if necessary. Do not assume that an address is freed exactly 60 seconds after the first packet; activity, child entries, and aging processing affect when the binding is removed.

To restore the default timers:

```text
Local_Router(config)# no ip nat translation timeout
Local_Router(config)# no ip nat translation icmp-timeout
```

### Clear a Specific Binding

The EXEC command specifies the **inside global address first**, followed by the inside local address:

```text
Local_Router# clear ip nat translation inside 7.7.7.50 10.1.1.5
```

If the half-entry still has child translations, use the `forced` option to remove it together with its associated sessions:

```text
Local_Router# clear ip nat translation inside 7.7.7.50 10.1.1.5 forced
```

Substitute the actual binding from the translation table.

### Clear All Dynamic Translations

```text
Local_Router# clear ip nat translation *
```

This clears dynamic translation state, not the configured NAT rules. Active traffic can recreate translations immediately, and existing connections can be disrupted. Configured static mappings remain in place.

After freeing LH5's binding, LH9 can obtain the newly available pool address if it sends matching traffic before another inside host claims it.

---

## Outside-Initiated Traffic

Dynamic inside source NAT is not a permanent outside-facing mapping. Before LH5 creates a binding, RH7 has no configured address pair through which to reach LH5 using `7.7.7.50`.

However, **ordinary dynamic one-to-one NAT is bidirectional once an address binding exists**. With a simple binding for LH5, outside traffic can use the reverse address mapping, subject to routing, NAT processing, and any separate security policies.

For example:

1. Clear dynamic translations and stop background traffic.
2. From RH7, attempt to ping `7.7.7.50`. There is no binding to LH5 yet; the packet cannot be translated to LH5 by this rule.
3. From LH5, ping `3.3.3.7` to create a binding.
4. Check the actual assigned global address on Local_Router.
5. From RH7 or RH8, initiate a new ping to that address while the binding exists.

Dynamic addresses can subsequently be freed and reassigned to different hosts. Use static NAT or static PAT for a service that needs a predictable external endpoint.

**NAT is not a firewall.** Do not rely on dynamic NAT to enforce a policy prohibiting outside-initiated connections.

---

## On-Link Pools and ARP

The `PUBLIC` pool belongs to the outside transit subnet `7.7.7.0/24`.

ISP1 considers `7.7.7.50` and `7.7.7.51` directly connected. To send traffic to one of these addresses, it resolves the destination IP through ARP rather than using `7.7.7.2` as an explicitly routed next hop.

Cisco NAT uses IP aliases for applicable translated addresses on directly connected subnets, allowing the NAT router to answer ARP for those addresses. Check which aliases exist before and after allocation rather than inferring their timing from the pool configuration alone.

```text
Local_Router# show ip aliases
Local_Router# show ip nat translations
ISP1# show ip arp 7.7.7.50
ISP1# show ip arp 7.7.7.51
```

A pool address should resolve to Local_Router's outside interface MAC when NAT is providing reachability for that address on the outside LAN.

**An IP alias, an ARP cache entry, and a NAT binding are different kinds of state.** An existing ARP entry on ISP1 can outlive a NAT binding and is not evidence that an inside host still owns that translation.

When testing ARP behavior in the lab, clear stale ARP entries on ISP1 if appropriate and repeat the test. Do not reuse addresses occupied by other devices on the outside LAN.

See [Static NAT Options and ARP Behavior](../static/options.md) for related alias behavior in static mappings.

---

## Off-Link (Routed) NAT Pools

A NAT pool does not need to be on the outside link. For example, ISP1 can route the documentation prefix `203.0.113.0/29` to Local_Router, which can allocate addresses from that prefix.

```text
ISP1(config)# ip route 203.0.113.0 255.255.255.248 7.7.7.2
```

On Local_Router, stop test traffic, clear dynamic state, and replace the existing mapping:

```text
Local_Router# clear ip nat translation *
Local_Router# configure terminal
Local_Router(config)# no ip nat inside source list NAT_INSIDE pool PUBLIC
Local_Router(config)# ip nat pool ROUTED 203.0.113.1 203.0.113.6 prefix-length 29
Local_Router(config)# ip nat inside source list NAT_INSIDE pool ROUTED
```

The old pool can be removed if no other rule references it:

```text
Local_Router(config)# no ip nat pool PUBLIC
```

The new pool provides six addresses. Local_Router's outside interface remains `7.7.7.2/24`.

```text
                         7.7.7.0/24
Local_Router -------------------------------- ISP1
  7.7.7.2                                     7.7.7.1
      ^                                           |
      +--- route to 203.0.113.0/29 via 7.7.7.2 ----+

      NAT pool: 203.0.113.1 - 203.0.113.6
```

For a reply destined to `203.0.113.1`, ISP1 performs a routing lookup and forwards the packet to next hop `7.7.7.2`. It resolves the next hop's MAC address, not the individual pool address. Local_Router then performs reverse NAT and routes the packet toward the inside host.

| Property | On-link pool | Routed pool |
|---|---|---|
| Global address range | `7.7.7.50 - 7.7.7.51` | `203.0.113.1 - 203.0.113.6` |
| ISP1 route | Connected outside subnet | Route via `7.7.7.2` |
| ISP1 ARP target | Individual translated address | Next-hop address `7.7.7.2` |
| Need for per-address ARP on outside link | Yes, under this connected-route design | No |

No interface on Local_Router needs to be assigned `203.0.113.1` merely to use it as an inside global address.

The `ip nat pool` command does not install the required route on ISP1. In a production design, the global prefix must be routed to the NAT device by the upstream network.

---

## NAT ACL Behavior and Static NAT Coexistence

A NAT ACL is a **classification mechanism**, not a packet-filtering policy by itself.

For example, modify the ACL so LH5 is not eligible for dynamic NAT:

```text
Local_Router(config)# ip access-list standard NAT_INSIDE
Local_Router(config-std-nacl)# 5 deny host 10.1.1.5
```

This assumes the existing permit statement has a higher sequence number, as it does with the default sequence numbering. Verify with `show ip access-lists NAT_INSIDE`.

Clear any preexisting dynamic binding for LH5 before testing the new classification. LH5's traffic no longer qualifies for this dynamic rule, but that does not inherently cause an ACL drop. In the lab it may fail because the outside network has no return route to `10.1.1.5`.

If the design requires LH5 to have a fixed global address and other hosts to receive dynamic addresses, configure a separate static mapping. Assuming the original `PUBLIC` pool configuration:

```text
Local_Router(config)# ip nat inside source static 10.1.1.5 7.7.7.60
```

The resulting design is:

| Inside host | Translation |
|---|---|
| LH5 | Static mapping to `7.7.7.60` |
| Other permitted inside hosts | Dynamic allocation from `7.7.7.50 - 7.7.7.51` |

Static mappings normally take precedence for matching traffic. Explicitly excluding the statically mapped host from the dynamic ACL and keeping its global address outside the dynamic pool makes the intended allocation unambiguous and avoids conflicting dynamic state.

---

## Verification and Troubleshooting

| Command | Purpose |
|---|---|
| `show ip nat translations` | Display current bindings and sessions |
| `show ip nat translations verbose` | Inspect mapping IDs, flags, aging, and other entry details |
| `show ip nat statistics` | Check interface roles, dynamic rules, pool occupancy, and counters |
| `show ip access-lists NAT_INSIDE` | Check classification rules and available hit counters |
| `show running-config \| include ip nat` | Review NAT-related configuration |
| `show ip route 3.3.3.7` | Verify forwarding toward the outside destination |
| `show ip aliases` | Inspect local address aliases |
| `show ip arp` | Inspect Layer 2 resolution |

On ISP1, check the route to the assigned global address as well as the applicable ARP entry.

In a controlled lab, debug translation processing if needed:

```text
Local_Router# debug ip nat
```

Generate test traffic from LH5, then disable debugging:

```text
Local_Router# undebug all
```

Debug visibility depends on the forwarding path and platform. Packet captures on both sides of the NAT router can independently confirm the actual source and destination addresses.

| Symptom | Possible cause |
|---|---|
| Dynamic rule exists but table is empty | No matching transit traffic yet |
| No binding after an inside host sends traffic | ACL mismatch, incorrect interface roles, routing problem, or allocation failure |
| Two hosts work but a third fails | Non-overloaded pool exhaustion |
| Binding exists but outside replies never arrive | Missing upstream route, ARP failure, or filtering |
| Reply arrives but does not reach the inside host | Missing/expired binding, inside routing, ACL, or asymmetric path |
| Pool remains full after pings stop | Simple bindings and/or child sessions have not aged out |
| Changing a NAT rule is rejected as in use | Existing translations must be removed before the change |
| Global address belongs to another device | Address collision or overlapping allocation |

Generate test traffic from an inside endpoint, not merely from Local_Router itself. Router-originated packets do not necessarily exercise the same traditional inside-to-outside transit NAT path.

---

## Key Takeaways

- Dynamic inside source NAT assigns inside global addresses from a pool on demand.
- Configure the pool with `ip nat pool` and associate it with an ACL using `ip nat inside source list ... pool ...`.
- The start and end addresses determine the pool's inclusive range; the mask does not expand that range.
- Without `overload`, an allocated global address represents one inside local address at a time.
- Multiple flows can share a single host's address binding without using PAT.
- An extended translation is not necessarily PAT.
- The NAT selection ACL identifies eligible traffic; it is not automatically an interface security ACL.
- A dynamic simple binding persists until it is cleared or can age out. The default binding timeout is 24 hours.
- Outside-initiated traffic cannot establish an ordinary inside source binding, but an existing one-to-one binding can support reverse translation.
- On-link global addresses require appropriate ARP handling; off-link pools require upstream routing.
- Verify actual bindings and pool utilization instead of assuming a particular address assignment or relying only on ping results.

---

## References

- [Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- [Cisco IOS XE: Network Address Translation Bindings](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_nat/configuration/xe-3s/nat-xe-3s-book/nat-xe-3s-book_chapter_011011.html)
- [Cisco IOS XE: Monitoring and Maintaining NAT](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_nat/configuration/xe-2/nat-xe-2-book/iadnat-monmain.html)
