# Dynamic PAT and NAT Overload

**Dynamic Port Address Translation (PAT)**, also called **NAT overload**, allows multiple inside hosts to share an inside global IPv4 address. Instead of allocating an entire global address to each inside local address, PAT distinguishes traffic using protocol-specific identifiers, such as TCP/UDP port numbers and ICMP Echo identifiers.

Dynamic PAT is the most common form of IPv4 NAT for Internet access. On Cisco IOS XE, the `overload` keyword enables PAT in a dynamic inside source NAT rule.

---

## Dynamic NAT vs. Dynamic PAT

| Feature | Dynamic NAT | Dynamic PAT |
|---|---|---|
| Address relationship | One-to-one while a binding exists | Many-to-one |
| Global address selection | Address pool | Address pool or interface address |
| TCP/UDP source ports | Unchanged by ordinary address-only translation | Preserved when possible or translated |
| ICMP Echo identifiers | Normally unchanged | Can be translated |
| Main resource | Available global addresses | Available global addresses, protocol identifiers, and translation capacity |
| `overload` keyword | No | Yes |

For example, with two inside hosts and only one global address, ordinary dynamic NAT can allocate that address to only one host at a time. PAT allows both hosts to use it simultaneously.

**PAT does not necessarily change the port number.** It can preserve the original source port if the resulting translation does not conflict with existing allocations.

See [Dynamic Inside Source NAT and Address Pools](inside-source.md) for address-only dynamic NAT.

---

## Lab Topology

We will continue using the topology from the previous NAT lessons.

```text
       INSIDE NETWORK                         OUTSIDE NETWORK

  LH5: 10.1.1.5/24
          |
          |                   Local_Router                    ISP1
          +---- LAN ---------- E0/0    E0/1 ---------------- E0/1
          |                10.1.1.1   7.7.7.2              7.7.7.1
          |                NAT inside NAT outside              |
  LH6: 10.1.1.6/24                                  E0/0: 3.3.3.1/24
                                                               |
                                                          Outside LAN
                                                               |
                                                        +------+------+
                                                        |             |
                                                  RH7 3.3.3.7   RH8 3.3.3.8
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

LH5 and LH6 use `10.1.1.1` as their default gateway. RH7 and RH8 use `3.3.3.1`. ISP1 does not need a route to `10.1.1.0/24` for successfully translated traffic.

Substitute the actual interface names on your IOS XE devices. The public addresses used here are for an isolated lab only.

Remove previous NAT rules and clear dynamic translations before starting each independent exercise. The sample translation tables below illustrate expected behavior; exact allocated ports and output formatting can differ.

---

## Interface-Based PAT

Interface-based PAT uses the IP address of a specified router interface as the inside global address. A NAT pool is not required.

### Configuration

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
Local_Router(config)# ip access-list standard NAT-INSIDE
Local_Router(config-std-nacl)# permit 10.1.1.0 0.0.0.255
!
Local_Router(config)# ip nat inside source list NAT-INSIDE interface Ethernet0/1 overload
Local_Router(config)# ip route 0.0.0.0 0.0.0.0 7.7.7.1
```

The NAT rule has three important parts:

| Argument | Purpose |
|---|---|
| `list NAT-INSIDE` | Selects inside local source addresses eligible for translation |
| `interface Ethernet0/1` | Supplies the inside global address (`7.7.7.2`) |
| `overload` | Allows multiple translations to share that address |

The interface argument specifies the address to use for translation; it is not an egress-interface match condition. Routing still determines the forwarding path. Route-map-based NAT can select translations based on additional conditions, as explained in [Route-Map-Based NAT and Conditional Translation](../policy-based/route-maps.md).

An ACL referenced by a NAT rule classifies traffic. A deny does not itself discard the packet; the packet might match another NAT rule or be forwarded untranslated.

### When to Use Interface-Based PAT

- A single outside address is available.
- The outside interface obtains an address dynamically, such as through DHCP.
- A dedicated public address pool is unnecessary.

If the interface address changes, existing connections using the old translated address cannot simply continue using it. New translations can use the new interface address.

---

## TCP Translation Example

Assume RH7 is running an HTTPS server and both LH5 and LH6 initiate connections using TCP source port `50000`.

| Host | Original source | Destination | Translated source |
|---|---|---|---|
| LH5 | `10.1.1.5:50000` | `3.3.3.7:443` | `7.7.7.2:50000` |
| LH6 | `10.1.1.6:50000` | `3.3.3.7:443` | `7.7.7.2:1024` |

The second translated port is an example, not a guaranteed allocation.

```text
LH5 10.1.1.5:50000 ---+                           +--> RH7 3.3.3.7:443
                      |                           |
                      +--> Local_Router ----------+
                      |    7.7.7.2:50000 (LH5)    |
LH6 10.1.1.6:50000 ---+    7.7.7.2:1024  (LH6)    +--> RH7 3.3.3.7:443
```

Without translating at least one source port, both connections would have identical TCP endpoints as seen by RH7. PAT prevents that ambiguity.

### Inside-to-Outside Processing

For LH6's connection:

1. The packet enters E0/0 with source `10.1.1.6:50000` and destination `3.3.3.7:443`.
2. Local_Router performs the routing lookup and selects E0/1.
3. NAT matches the inside source address against the ACL and creates a translation if needed.
4. NAT changes the source address and, in this example, the source port.
5. The packet is forwarded toward ISP1.

### Outside-to-Inside Processing

RH7 sends its reply to the translated endpoint.

| Field | Before reverse translation | After reverse translation |
|---|---|---|
| Source | `3.3.3.7:443` | `3.3.3.7:443` |
| Destination | `7.7.7.2:1024` | `10.1.1.6:50000` |

Local_Router looks up the NAT state, restores LH6's original destination address and port, and routes the packet toward LH6.

**Inside source PAT translates the source on outbound packets and the destination on return packets.** See [NAT Packet Processing and Order of Operations](../order-of-operations.md).

### Translation Table

```text
Local_Router# show ip nat translations
Pro  Inside global      Inside local       Outside local      Outside global
 tcp 7.7.7.2:50000      10.1.1.5:50000    3.3.3.7:443        3.3.3.7:443
 tcp 7.7.7.2:1024       10.1.1.6:50000    3.3.3.7:443        3.3.3.7:443
```

Both are **extended translations**. The outside local and outside global columns are identical because no outside address translation is configured.

The first entry is PAT even though the source port is preserved. The presence or absence of a port change alone is not sufficient to identify the configured NAT mechanism.

---

## Port Preservation and Allocation

The classic Cisco IOS PAT allocation algorithm described in the [Cisco NAT FAQ](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/26704-nat-faq-00.html) divides the port space into three groups:

| Port group | Range |
|---|---|
| Low | `0-511` |
| Middle | `512-1023` |
| High | `1024-65535` |

TCP/UDP port zero is not normally allocated for connections. ICMP identifiers have different semantics.

For ordinary overload, the documented allocation sequence is:

1. Attempt to preserve the original source port on an eligible global address.
2. If unavailable, search for a free port in the corresponding range.
3. If that range is exhausted, try another global address if the pool has one.
4. If no eligible allocation can be made, the new translation fails.

For instance, if TCP port `50000` is unavailable, a replacement is selected from the high port range under this algorithm.

Actual IOS XE port managers and specialized NAT modes can use different allocation policies. Do not assume that the next port must always be `1024`, that allocation is round-robin, or that every port is usable. Use the translation table to verify the result.

### TCP and UDP Port Spaces

TCP and UDP have separate protocol numbers and separate port spaces. A TCP translation and a UDP translation can use the same numeric global port without conflicting solely because the numbers are equal.

For ordinary PAT, do not assume a global TCP port can be allocated to multiple independent mappings just because their destinations differ. Endpoint-dependent PAT is a separate feature discussed below.

---

## PAT for ICMP

ICMP does not use TCP or UDP ports. ICMP Echo Request and Echo Reply messages have a 16-bit **Identifier** field, which PAT can use to distinguish simultaneous echo exchanges.

Suppose LH5 and LH6 both ping RH7 using ICMP identifier `7`.

| Host | Inside local | Inside global |
|---|---|---|
| LH5 | `10.1.1.5:7` | `7.7.7.2:7` |
| LH6 | `10.1.1.6:7` | `7.7.7.2:8` |

An illustrative translation table is:

```text
Local_Router# show ip nat translations
Pro  Inside global      Inside local       Outside local      Outside global
icmp 7.7.7.2:7          10.1.1.5:7        3.3.3.7:7          3.3.3.7:7
icmp 7.7.7.2:8          10.1.1.6:7        3.3.3.7:7          3.3.3.7:7
```

The number after the colon is an ICMP identifier, **not an ICMP port**. It is also not the ICMP sequence number. A packet capture on each side of Local_Router is useful to confirm the identifier actually carried in the translated packet.

ICMP error messages, such as Destination Unreachable and Time Exceeded, are different from Echo messages. NAT can use information from the original packet quoted within an ICMP error to associate it with a translation and rewrite relevant fields. This is important for traceroute and Path MTU Discovery.

---

## Pool-Based PAT

Pool-based PAT allows the router to use one or more addresses from an explicitly configured NAT pool.

### Configuration

First clear the previous dynamic translations and remove the interface-based rule:

```text
Local_Router# clear ip nat translation *
Local_Router# configure terminal
Local_Router(config)# no ip nat inside source list NAT-INSIDE interface Ethernet0/1 overload
```

Stop traffic during the change so that new translations do not immediately reappear.

Configure a pool of three on-link addresses:

```text
Local_Router(config)# ip nat pool PUBLIC 7.7.7.50 7.7.7.52 prefix-length 24
Local_Router(config)# ip nat inside source list NAT-INSIDE pool PUBLIC overload
```

The interface roles, ACL, and default route remain unchanged.

Compare this rule with ordinary dynamic NAT:

```text
! Dynamic address-only NAT:
ip nat inside source list NAT-INSIDE pool PUBLIC
!
! Dynamic PAT:
ip nat inside source list NAT-INSIDE pool PUBLIC overload
```

**The `overload` keyword belongs to the NAT rule, not the `ip nat pool` command.** A generic pool can be used for ordinary dynamic NAT or PAT, depending on the referencing rule.

### Multiple Global Addresses

Pool-based PAT can share each global address across many connections. It does not require a separate global address for every inside host.

For example:

```text
Pro  Inside global      Inside local       Outside local      Outside global
 tcp 7.7.7.50:50000     10.1.1.5:50000    3.3.3.7:443        3.3.3.7:443
 tcp 7.7.7.50:1024      10.1.1.6:50000    3.3.3.7:443        3.3.3.7:443
```

Both hosts can use the first global address while the remaining pool addresses are unused. Additional addresses provide more allocation capacity. The exact selection policy depends on the port allocator and NAT mode.

### Single-Address Pool

A pool can contain a single address:

```text
Local_Router(config)# ip nat pool ONE 7.7.7.60 7.7.7.60 prefix-length 24
Local_Router(config)# ip nat inside source list NAT-INSIDE pool ONE overload
```

Use this as an alternative to the PUBLIC rule, not an additional overlapping rule.

A single-address pool is useful when many hosts must share a specific address that is different from the outside interface address.

Cisco advises against placing the router's own interface address in a NAT pool. Use interface overload when that address is the desired global address.

### Return Reachability and ARP

For the on-link PUBLIC pool, ISP1 considers `7.7.7.50-7.7.7.52` directly connected. It must resolve the translated destination to a MAC address. Verify NAT alias and ARP behavior using:

```text
Local_Router# show ip aliases
ISP1# show ip arp
```

For an off-link pool, upstream routing must direct the translated addresses toward Local_Router. For example, as an alternative pool:

```text
Local_Router(config)# ip nat pool ROUTED 198.51.100.10 198.51.100.20 prefix-length 24
Local_Router(config)# ip nat inside source list NAT-INSIDE pool ROUTED overload
!
ISP1(config)# ip route 198.51.100.0 255.255.255.0 7.7.7.2
```

The ISP routes packets to the NAT router's next-hop address rather than ARPing for each translated address. Only advertise or route prefixes actually allocated to the NAT device in production.

---

## PAT Capacity and Resource Exhaustion

TCP and UDP port numbers are 16 bits, yielding 65,536 possible numeric values per protocol. This does **not** mean that every global address guarantees 65,536 concurrent usable TCP or UDP translations.

Practical capacity depends on:

- Available ports in the relevant ranges.
- Ports reserved by local services or static mappings.
- Port allocator implementation and NAT mode.
- Translation table limits and available platform resources.
- Aging timers and the rate of new connection creation.

The IOS XE 17.x NAT configuration guide notes that interface overload can have a substantially smaller effective port range than the theoretical total.

A full pool of allocated global addresses is not necessarily exhausted for PAT. Existing addresses can still have available ports. Conversely, a new connection can fail even though the translation table has not reached its configured maximum, because the required port allocation cannot be made.

### Translation Limits

To limit the number of NAT translations globally:

```text
Local_Router(config)# ip nat translation max-entries 10000
```

Cisco also documents more granular limits, including per-host limits, on supported IOS XE implementations. For example:

```text
Local_Router(config)# ip nat translation max-entries all-host 500
```

Verify available subcommands using CLI help. These limits control translation entries, not the number of unique users or available pool addresses.

### Non-PATtable Traffic

TCP, UDP, and ICMP Echo traffic have identifiers suitable for ordinary PAT. Other IP protocols do not necessarily have usable identifiers, although protocol-specific NAT handling may exist.

Non-PATtable traffic, such as IP-in-IP (IP protocol 4), can create a **simple dynamic NAT binding** even when `overload` is configured.

When using an overloaded NAT pool:

- The first address in the pool is skipped when allocating simple bindings for non-PATtable traffic.
- A single-address overloaded pool cannot accommodate non-PATtable traffic.
- Subsequent PATtable traffic from the same inside host uses its existing simple binding.
- Other inside hosts can share the same inside global address through PAT despite the existence of a simple binding.

A NAT classification ACL can be used to exclude certain protocols from translation. However, denying traffic in a NAT ACL does not itself cause the traffic to be dropped.

---

## Static PAT and Dynamic PAT Together

Dynamic PAT is primarily used for inside-initiated traffic. Static PAT can publish an inside service on a predetermined endpoint.

For example, keep the interface overload rule and add an HTTPS port-forwarding rule for LH5:

```text
Local_Router(config)# ip nat inside source list NAT-INSIDE interface Ethernet0/1 overload
Local_Router(config)# ip nat inside source static tcp 10.1.1.5 443 interface Ethernet0/1 8443
```

The intended result is:

- LH5 and LH6 can initiate traffic using dynamic PAT.
- Outside clients can reach LH5's HTTPS service at `7.7.7.2:8443`.
- The static TCP endpoint must not conflict with a dynamic allocation or a router-local service.

Remove conflicting dynamic translations before adding a static port mapping if needed. Sharing the interface address between these NAT features is distinct from configuring overlapping address-only static and dynamic pool mappings.

An unsolicited inbound packet does not create a general-purpose dynamic inside source PAT mapping. Without a suitable existing translation or a static mapping, it cannot be translated to an arbitrary inside host. It might instead be processed locally or dropped depending on the destination and forwarding context.

**NAT is not a firewall.** Use ACLs or a firewall to enforce security policy.

See [Static PAT and Port Forwarding](../static/pat.md) for details.

---

## Translation Aging and Clearing

PAT translations are dynamic state. TCP, UDP, and ICMP entries have different inactivity timers; TCP state and application-specific handling may affect expiration.

Example timer configuration (values chosen for a lab, not defaults):

```text
Local_Router(config)# ip nat translation tcp-timeout 1800
Local_Router(config)# ip nat translation udp-timeout 120
Local_Router(config)# ip nat translation icmp-timeout 30
```

The generic `ip nat translation timeout` is associated with address bindings and should not be confused with the protocol-specific PAT timers.

To clear dynamic translations:

```text
Local_Router# clear ip nat translation *
```

This disrupts active translated connections. Static mappings remain configured. An active traffic source may recreate dynamic state immediately.

See [NAT Translation Tables and Verification](../verification.md) for the documented defaults, detailed aging behavior, and selective clearing.

---

## Lab Verification

### Exercise 1: Interface Overload

Configure interface-based PAT and generate traffic from LH5 and LH6:

```text
LH5# ping 3.3.3.7
LH6# ping 3.3.3.7
```

Immediately inspect the table:

```text
Local_Router# show ip nat translations
Local_Router# show ip nat translations verbose
Local_Router# show ip nat statistics
```

Confirm that both inside addresses are represented by `7.7.7.2`. ICMP entries can expire quickly.

### Exercise 2: Force a TCP Port Collision

If LH5 and LH6 are Linux hosts, start a TCP listener on RH7 (for example, using a compatible netcat implementation):

```bash
nc -lk 8080
```

On both LH5 and LH6, open a connection with the same source port:

```bash
nc -p 50000 3.3.3.7 8080
```

Keep the connections open and inspect the translation table. Netcat options and concurrent connection handling vary; a TCP server that accepts simultaneous connections is required. Alternatively, generate the two TCP flows using any suitable traffic tool.

Verify that both translated TCP endpoints are distinguishable. Capture packets on the inside and outside links to observe the original and translated source ports.

### Exercise 3: Pool Overload

Replace the interface rule with the PUBLIC pool rule and repeat the tests. Confirm that the inside global address is from the pool rather than `7.7.7.2`.

Inspect pool usage in `show ip nat statistics`. Distinguish the number of allocated global addresses from the number of translation entries.

### Exercise 4: Compare With Non-Overloaded NAT

Use a one-address pool with a rule **without** `overload`. After clearing old state, generate traffic from LH5 first, then LH6. The first host acquires the available address; the second cannot obtain a separate binding while it is occupied.

Clear the translations, replace the rule with its overloaded equivalent, and repeat. Both hosts can now share the address.

---

## Troubleshooting

| Symptom | What to check |
|---|---|
| No dynamic translations | NAT interface roles, route lookup, matching ACL, and transit traffic generation |
| Only one inside host works with a one-address pool | Missing `overload` or a non-PATtable binding |
| Incorrect global address | Interface versus pool selection, existing translations, competing rules |
| Translations exist but no replies | Upstream routing, ARP, ACLs, outside host, and return path symmetry |
| Existing flows work but new flows fail | Port exhaustion, translation limits, resource drops |
| Ping works but TCP fails | TCP-specific filtering, remote listener, port allocation |
| Outside-initiated connection fails | Missing static PAT or applicable existing translation |
| Old rule cannot be removed | Active translations; stop traffic and clear dynamic state |

Useful commands:

```text
Local_Router# show running-config | include ip nat
Local_Router# show access-lists NAT-INSIDE
Local_Router# show ip nat translations
Local_Router# show ip nat translations verbose
Local_Router# show ip nat statistics
Local_Router# show ip route 3.3.3.7
Local_Router# show ip aliases
ISP1# show ip arp
```

In a controlled lab, `debug ip nat` can help identify translation activity. Hardware or accelerated forwarding may not produce a debug message for every packet. Disable debugging with `undebug all` when finished.

Generate transit traffic from LH5 and LH6 rather than assuming a ping sourced by Local_Router exercises the same inside-to-outside NAT path.

---

## Key Takeaways

- `overload` enables dynamic PAT on an inside source NAT rule.
- Interface-based PAT uses the specified interface address; pool-based PAT uses addresses from a pool.
- PAT can preserve or change TCP/UDP ports and ICMP Echo identifiers.
- Extended entries alone do not prove that overload is configured.
- Multiple inside hosts can share one global address, but port and translation resources are finite.
- Non-PATtable traffic can consume an entire pool address on IOS XE.
- `overload extended` is an optional endpoint-dependent allocation feature, not ordinary PAT syntax required for extended entries.
- Dynamic PAT and static PAT can provide complementary outbound access and inbound service publishing.
- Routing, ARP, translation state, and security policy must all be considered during troubleshooting.

---

## References

- [Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- [Cisco: Network Address Translation (NAT) Frequently Asked Questions](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/26704-nat-faq-00.html)
- [Cisco IOS IP Addressing Services Command Reference: NAT Translation Commands](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr/command/ipaddr-cr-book/ipaddr-i4.html)
