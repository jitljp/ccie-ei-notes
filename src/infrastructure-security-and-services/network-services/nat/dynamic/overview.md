# 4.5.d (ii) Dynamic NAT, PAT

**Dynamic Network Address Translation (NAT)** creates address mappings as matching traffic is processed, rather than requiring an administrator to configure a separate mapping for every host. **Dynamic Port Address Translation (PAT)**, also called **NAT overload**, additionally uses protocol and port information to allow multiple inside hosts to share an inside global address.

These mechanisms are primarily used to provide connectivity from an inside network to an outside network while conserving IPv4 addresses.

The main options are:

- **Dynamic inside source NAT:** Assign an inside global address from a pool to an inside host.
- **Dynamic PAT using a pool:** Allow multiple inside hosts to share addresses from a configured pool.
- **Dynamic PAT using an interface:** Allow multiple inside hosts to share an interface's IP address.
- **Dynamic outside source NAT:** Assign outside local addresses to outside hosts as traffic matches a rule.
- **Dynamic inside destination NAT:** Translate a virtual destination address to one of several inside servers, using a rotary pool.

This section covers traditional Cisco IOS XE NAT using `ip nat inside` and `ip nat outside`. Policy-based NAT, VRF-aware NAT, and VASI NAT are covered in later sections.

---

## Dynamic NAT vs. Dynamic PAT

Both mechanisms create translation state dynamically. The main difference is whether multiple hosts can share the same translated IP address.

| Feature | Dynamic NAT (without overload) | Dynamic PAT (overload) |
|---|---|---|
| Translation | Inside local IP to inside global IP | Inside local IP/port to inside global IP/port |
| Address relationship | One-to-one while the mapping is active | Many-to-one, or many-to-few |
| TCP/UDP ports | Normally unchanged | May be preserved or translated |
| Translated address source | NAT pool | NAT pool or interface address |
| Multiple inside hosts sharing one global address | No | Yes |
| `overload` keyword | No | Yes |
| Address conservation | Limited by the number of pool addresses | Many sessions can use each global address |
| Outside-initiated connections | Require an existing mapping | Normally require matching translation state or a separate static mapping |

**Dynamic does not mean that an address changes for every packet or connection.** Without overload, an allocated address mapping can be reused by multiple connections from the same inside host until that mapping expires or is cleared.

Unlike [static NAT and PAT](../static/overview.md), dynamic mappings generally do not exist before matching traffic causes them to be created.

---

## Lab Topology

The examples use the same devices and addresses as the static NAT section.

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

Basic configuration on Local_Router:

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

The host default gateways are `10.1.1.1` for the inside hosts and `3.3.3.1` for the remote hosts.

**The NAT rules in the following examples are alternatives.** Test them individually rather than applying every rule to the same traffic at once.

---

## Dynamic Inside Source NAT

Dynamic inside source NAT allocates an available **inside global** address to an inside host when its traffic matches the NAT rule.

For example, suppose the router has the following pool:

```text
7.7.7.50 - 7.7.7.51
```

When LH5 and LH6 initiate traffic to RH7, Local_Router may create these mappings:

| Inside local | Inside global |
|---|---|
| `10.1.1.5` | `7.7.7.50` |
| `10.1.1.6` | `7.7.7.51` |

The specific allocation depends on the available pool addresses and existing translation state.

### Configuration

```text
Local_Router(config)# ip nat pool GLOBAL-POOL 7.7.7.50 7.7.7.51 netmask 255.255.255.0
Local_Router(config)# access-list 10 permit 10.1.1.0 0.0.0.255
Local_Router(config)# ip nat inside source list 10 pool GLOBAL-POOL
```

The configuration has three components:

1. `ip nat pool` defines the available inside global addresses.
2. The ACL selects the inside source addresses eligible for translation.
3. `ip nat inside source list ... pool ...` associates the ACL with the address pool.

The pool's `netmask` describes the subnet containing the pool addresses. It does **not** specify the number of addresses allocated. That is determined by the start and end addresses.

The same pool could be defined using `prefix-length 24` instead of `netmask 255.255.255.0`.

### Packet Translation

For a packet sent from LH5 to RH7:

| Point in path | Source IP | Destination IP |
|---|---|---|
| Before NAT | `10.1.1.5` | `3.3.3.7` |
| After NAT | `7.7.7.50` | `3.3.3.7` |
| Return packet before NAT | `3.3.3.7` | `7.7.7.50` |
| Return packet after NAT | `3.3.3.7` | `10.1.1.5` |

The mapping is associated with the inside host, not with a particular destination server or TCP/UDP port.

### Pool Exhaustion

The example pool contains **two addresses**, so dynamic NAT without overload can provide simultaneous mappings for **two inside hosts**.

If a third host needs a new mapping while both addresses are in use, the router cannot allocate another address from this pool. That host will not obtain a translation from this rule until an address becomes available. Another applicable NAT rule, if configured, may produce a different result.

An unused dynamic mapping can eventually expire, freeing its address for reuse.

For further details, see [Dynamic Inside Source NAT and Address Pools](inside-source.md).

---

## NAT Pool Allocation and Host Matching

NAT pools specify the addresses available for dynamic allocation:

```text
ip nat pool <name> <start-ip> <end-ip> {netmask <mask> | prefix-length <length>}
```

A pool can be used by different NAT mechanisms, but **the meaning of its addresses depends on the NAT rule**. An inside source pool contains inside global addresses, whereas a dynamic outside source pool contains outside local addresses.

Two pool types are particularly important:

| Pool type | Purpose |
|---|---|
| Default address pool | Allocate available translated addresses dynamically |
| `type match-host` | Preserve the host portion of the original address in the translated address |
| `type rotary` | Rotate among real inside servers for destination NAT TCP load distribution |

The `match-host` option is about **address correspondence**, not PAT or port allocation. The `rotary` option is used for a different purpose from ordinary source NAT address allocation.

Other relevant considerations include overlapping static and dynamic mappings, address availability, pool statistics, and whether the translated addresses are reachable from the outside network.

See [NAT Pool Allocation and Host Matching](pool-allocation.md).

---

## Dynamic PAT (NAT Overload)

Dynamic PAT allows multiple inside hosts to share one or more inside global addresses by maintaining separate translations for their traffic.

For TCP and UDP, PAT uses port numbers to distinguish translations. For ICMP queries, it can use ICMP identifiers. Protocols that cannot be overloaded may require a whole address allocation instead.

### PAT Using the Outside Interface Address

The most common configuration uses the IP address of the outside interface:

```text
Local_Router(config)# access-list 10 permit 10.1.1.0 0.0.0.255
Local_Router(config)# ip nat inside source list 10 interface Ethernet0/1 overload
```

All matching inside hosts can use `7.7.7.2`, the IP address of Ethernet0/1.

For example, LH5 and LH6 each open a TCP connection to RH7 on port `443`, both using source port `50000`:

| Inside local endpoint | Inside global endpoint | Destination |
|---|---|---|
| `10.1.1.5:50000` | `7.7.7.2:40001` | `3.3.3.7:443` |
| `10.1.1.6:50000` | `7.7.7.2:40002` | `3.3.3.7:443` |

These are **illustrative translated ports**. IOS XE may preserve an original source port when it is available, rather than choosing a different one.

The return traffic is translated according to the corresponding PAT entry, allowing the router to deliver each reply to the correct inside host.

### PAT Using an Address Pool

PAT can also use multiple global addresses:

```text
Local_Router(config)# ip nat pool PAT-POOL 7.7.7.50 7.7.7.51 netmask 255.255.255.0
Local_Router(config)# access-list 10 permit 10.1.1.0 0.0.0.255
Local_Router(config)# ip nat inside source list 10 pool PAT-POOL overload
```

**The `overload` keyword is the critical difference** between this configuration and ordinary dynamic inside source NAT.

Without `overload`, a global address is assigned to only one inside host while the mapping exists. With `overload`, many connections from different inside hosts can use the same global address, provided their translated connection identifiers can be distinguished.

PAT does not offer unlimited capacity. Available ports, translation limits, protocol handling, and platform resources can restrict the number of simultaneous translations.

See [Dynamic PAT and NAT Overload](pat.md) for address/port allocation, port preservation, multiple inside hosts, and exhaustion scenarios.

---

## Dynamic Outside Source NAT

Dynamic outside source NAT changes how an **outside host** is represented to the inside network.

For example, RH7 uses the outside global address `3.3.3.7`. Local_Router can assign it an outside local address from a pool, such as `172.16.100.50`.

| Address type | Example |
|---|---|
| Outside global | `3.3.3.7` |
| Outside local | `172.16.100.50` |

A basic configuration is:

```text
Local_Router(config)# ip nat pool OUTSIDE-LOCAL 172.16.100.50 172.16.100.51 netmask 255.255.255.0
Local_Router(config)# access-list 20 permit host 3.3.3.7
Local_Router(config)# ip nat outside source list 20 pool OUTSIDE-LOCAL add-route
```

The ACL selects **outside source addresses**, and the pool contains the **outside local addresses** assigned to matching hosts.

When traffic from RH7 enters the outside interface and triggers a translation, the router changes the source address from `3.3.3.7` to the allocated outside local address. Reverse traffic can then have its destination translated back to `3.3.3.7`.

Unlike static outside source NAT, a dynamic mapping is not created in advance. Routing toward the outside local addresses is especially important; `add-route` can provide routes associated with dynamically created translations.

See [Dynamic Outside Source NAT](outside-source.md).

---

## Dynamic Inside Destination NAT and Load Balancing

Dynamic NAT can also translate **destination addresses**. One use is to distribute new TCP connections for a virtual service address among several inside servers.

Suppose outside clients connect to `7.7.7.100`, but the actual servers are LH5 (`10.1.1.5`) and LH6 (`10.1.1.6`).

```text
Local_Router(config)# ip nat pool SERVERS 10.1.1.5 10.1.1.6 netmask 255.255.255.0 type rotary
Local_Router(config)# access-list 30 permit host 7.7.7.100
Local_Router(config)# ip nat inside destination list 30 pool SERVERS
```

In this configuration:

- The ACL identifies the **virtual destination address**.
- The rotary pool contains the **real inside server addresses**.
- New matching TCP connections can be assigned to different servers.
- Subsequent packets belonging to the same translation continue to reach the selected server.

This is different from ordinary dynamic source NAT and PAT. The router is choosing a **destination**, not assigning a source address for an outbound client.

Basic NAT rotary load distribution should not be confused with a full-featured load balancer with sophisticated health checks and application-aware decisions.

See [Inside Destination NAT and Load Balancing](destination-nat.md).

---

## Translation Creation, Reuse, and Expiration

Dynamic NAT is **stateful**: the router records the translations it allocates so that subsequent packets can be handled consistently.

The general sequence is:

1. A packet reaches an interface configured for NAT.
2. The router determines whether the packet matches a NAT rule and whether a usable translation already exists.
3. If needed, the router allocates an address or an address/port mapping.
4. The packet is translated and forwarded, provided ordinary routing and filtering requirements are satisfied.
5. Subsequent matching packets reuse the applicable translation state.
6. Idle dynamic translations eventually expire, releasing their resources.

Without overload, the principal resource is an **address mapping**. With overload, the router also tracks **per-flow translations** and the identifiers required to distinguish simultaneous traffic.

NAT timeouts vary by entry type and protocol. Useful commands include:

```text
Local_Router(config)# ip nat translation timeout <seconds>
Local_Router(config)# ip nat translation tcp-timeout <seconds>
Local_Router(config)# ip nat translation udp-timeout <seconds>
Local_Router(config)# ip nat translation icmp-timeout <seconds>
```

Do not confuse the lifetime of a dynamic address binding with the lifetime of every TCP, UDP, or ICMP translation associated with it.

For more about translation tables, aging, and timers, see [NAT Translation Tables and Verification](../verification.md).

---

## NAT ACLs and Traffic Selection

In a standard inside source NAT rule such as:

```text
Local_Router(config)# access-list 10 permit 10.1.1.0 0.0.0.255
Local_Router(config)# ip nat inside source list 10 interface Ethernet0/1 overload
```

The ACL determines **which source addresses are eligible for translation**.

- `permit` means the traffic matches this NAT rule.
- `deny` means the traffic does not match this NAT rule.
- A NAT ACL `deny` is **not a packet-filtering deny**. Routing and separate security policies determine whether unmatched traffic is forwarded or dropped.

For example, the ACL could exclude a particular inside host:

```text
Local_Router(config)# ip access-list standard NAT-SOURCES
Local_Router(config-std-nacl)# deny host 10.1.1.6
Local_Router(config-std-nacl)# permit 10.1.1.0 0.0.0.255
!
Local_Router(config)# ip nat inside source list NAT-SOURCES interface Ethernet0/1 overload
```

Now LH5 matches the rule, but LH6 does not.

The meaning of the ACL depends on the NAT command. In `ip nat inside destination list`, for example, the ACL identifies the **destination** being translated.

More selective translation based on destination, route map, or egress path belongs to [Policy-Based NAT and PAT](../policy-based/overview.md).

---

## Routing, ARP, and Return Traffic

**NAT does not replace routing.** A correct translation is only one requirement for successful communication.

For dynamic inside source NAT:

- Local_Router must know how to reach the inside hosts and outside destinations.
- The outside network must be able to return traffic to the allocated inside global addresses.
- If a global address is on a directly connected outside subnet, the upstream router may use ARP to resolve it. NAT alias and ARP behavior therefore matter.
- If the global address comes from a separately routed block, the upstream network needs an appropriate route toward the NAT router.

For interface-based PAT, return traffic addressed to the outside interface's own address already has a directly connected destination on that link.

For inside destination NAT, the **virtual address** must likewise be reachable from the outside, and the selected inside server needs a return path through the NAT device.

Also consider [NAT Packet Processing and Order of Operations](../order-of-operations.md) when ACLs, PBR, and routing interact with translations.

---

## Verification and Troubleshooting

The most useful operational commands are:

```text
Local_Router# show ip nat translations
Local_Router# show ip nat translations verbose
Local_Router# show ip nat statistics
Local_Router# show running-config | include ip nat
Local_Router# show access-lists
Local_Router# show ip route
```

### Dynamic NAT Example

With LH5 and LH6 actively using separate global addresses, the translation table may contain entries similar to:

```text
Pro  Inside global     Inside local      Outside local     Outside global
---  7.7.7.50         10.1.1.5          ---               ---
---  7.7.7.51         10.1.1.6          ---               ---
```

### Dynamic PAT Example

When LH5 and LH6 use interface PAT to reach RH7, entries may resemble:

```text
Pro  Inside global     Inside local      Outside local     Outside global
tcp  7.7.7.2:40001    10.1.1.5:50000   3.3.3.7:443       3.3.3.7:443
tcp  7.7.7.2:40002    10.1.1.6:50000   3.3.3.7:443       3.3.3.7:443
```

These outputs are illustrative. Addresses, ports, and translation entries depend on the traffic generated and the router's existing state.

`show ip nat statistics` is particularly useful for identifying active mappings, pool usage, allocation failures, and translation counters.

To remove dynamic translation state during a controlled lab test:

```text
Local_Router# clear ip nat translation *
```

**Caution:** Clearing translations interrupts traffic using those entries. Use it carefully on production routers.

### Common Problems

| Symptom | Possible cause |
|---|---|
| No translation appears | Incorrect NAT interface roles, ACL mismatch, routing, or traffic not crossing the NAT boundary |
| Only some hosts can use dynamic NAT | Address pool exhausted, existing bindings, or restrictive ACL |
| Dynamic PAT sessions fail as load increases | Port/resource exhaustion, translation limits, or platform restrictions |
| Return packets do not reach the NAT router | Missing upstream route or ARP resolution for the inside global address |
| Traffic reaches the NAT router but fails | NAT/ACL mismatch, asymmetric return path, or application issue |
| An outside host cannot initiate a new session | No existing suitable dynamic translation or static mapping |
| A new NAT rule behaves unexpectedly | Overlapping rules, static mappings, or existing dynamic translation entries |
| Outside source NAT fails in one direction | Outside local routing or missing translation state |

For detailed verification, debugging, and timeout behavior, see [NAT Translation Tables and Verification](../verification.md).

---

## Key Takeaways

- **Dynamic NAT** allocates translated addresses when matching traffic creates a mapping.
- **Dynamic NAT without overload** provides one-to-one address mappings while they remain active.
- **Dynamic PAT (overload)** allows multiple inside hosts to share a translated IPv4 address by distinguishing their traffic with protocol-specific identifiers.
- Dynamic PAT can use either a **NAT pool** or an **interface address**.
- The `overload` keyword distinguishes ordinary dynamic inside source NAT from dynamic PAT.
- NAT ACLs identify traffic eligible for translation; they do not themselves filter unmatched packets.
- An exhausted NAT pool prevents new allocations from that rule until resources become available.
- Dynamic outside source NAT allocates **outside local** addresses to outside hosts.
- Dynamic inside destination NAT can distribute TCP connections across real servers using a **rotary pool**.
- Dynamic translations require appropriate routing, return reachability, and translation state.
- `show ip nat translations` and `show ip nat statistics` are fundamental verification commands.

---

## References

- [Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- [Cisco IOS IP Addressing Services Command Reference: NAT Commands](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr/command/ipaddr-cr-book/ipaddr-i3.html)
- [Cisco: Configure IP NAT Outside Source List Command](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/13770-1.html)
- [Cisco: Network Address Translation (NAT) Frequently Asked Questions](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/26704-nat-faq-00.html)
