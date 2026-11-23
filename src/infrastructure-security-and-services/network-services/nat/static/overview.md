# 4.5.d (i) Static NAT, PAT

**Static Network Address Translation (NAT)** creates an explicitly configured mapping between IP addresses. **Static Port Address Translation (PAT)** creates a fixed mapping between an IP address/protocol/port combination and another IP address/protocol/port combination.

Unlike dynamic NAT, the mapping is defined by the administrator rather than allocated from a pool when traffic first matches a rule.

Static NAT and PAT are useful when:

- An inside server must always be reachable using the same outside address.
- An internal service must be reachable through a particular TCP or UDP port.
- An entire network must be represented by another network of the same size.
- An outside device must appear under a different address to inside devices.
- A translated address needs to remain predictable for application configuration, DNS, or routing policy.

**Static describes how the mapping is configured and maintained.**

This section focuses on *traditional inside/outside NAT* on Cisco IOS XE. Policy-based, VRF-aware, and VASI NAT are covered separately.

---

## Static NAT vs. Static PAT

The distinction is whether the mapping identifies an **entire IP address** or a **specific transport-layer endpoint**.

| Feature | Static NAT | Static PAT |
|---|---|---|
| Mapping | IP address to IP address | Protocol, IP address, and port to protocol, IP address, and port |
| Typical relationship | One-to-one address mapping | One-to-one mapping of a TCP or UDP endpoint |
| TCP/UDP ports | Normally unchanged | Can be preserved or changed |
| Common use | Give an inside host a fixed external address | Publish a specific service or perform port forwarding |
| Basic command | `ip nat inside source static` | `ip nat inside source static tcp` or `udp` |
| Requires `overload`? | No | No |

**Do not confuse static PAT with dynamic PAT (NAT overload).**

- **Static PAT** reserves a specific address/port mapping through configuration.
- **Dynamic PAT** creates per-flow translations as matching traffic arrives, allowing many sessions and hosts to share an inside global address.

For an introduction to all NAT types, see [NAT Address Terminology and Translation Types](../terminology.md).

---

## Inside and Outside Interface Roles

Traditional Cisco IOS XE NAT generally requires the relevant interfaces to be identified as **inside** and **outside**.

Consider this topology:

```text
     Inside network                 R1                       Outside
       10.1.1.0/24             NAT router                 via ISP

   PC / Server              G1      G2                      ISP
    10.1.1.10 ----------------[ R1 ]-----------------------[ R2 ]
                            |       |                        |
                         10.1.1.1  192.0.2.2              192.0.2.1
                         /24       /30                    /30
                         INSIDE    OUTSIDE
```

On R1:

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# ip address 10.1.1.1 255.255.255.0
R1(config-if)# ip nat inside
R1(config-if)# no shutdown
!
R1(config)# interface GigabitEthernet2
R1(config-if)# ip address 192.0.2.2 255.255.255.252
R1(config-if)# ip nat outside
R1(config-if)# no shutdown
!
R1(config)# ip route 0.0.0.0 0.0.0.0 192.0.2.1
```

This configures the **NAT interface roles and basic routing**, but it does **not** create any NAT translations by itself. A NAT rule must still be configured.

The interface roles do not indicate whether a connection was initiated from inside or outside. They identify the respective NAT domains.

The basic operation of address translation and the different processing orders are covered in [NAT Packet Processing and Order of Operations](../order-of-operations.md).

---

## Static Inside Source NAT

The most common type of static NAT maps an **inside local** address to an **inside global** address.

Its basic syntax is:

```text
ip nat inside source static <inside-local> <inside-global>
```

Suppose the inside server at `10.1.1.10` must be represented to outside devices as `203.0.113.10`.

```text
R1(config)# ip nat inside source static 10.1.1.10 203.0.113.10
```

| Address type | Value |
|---|---|
| Inside local | `10.1.1.10` |
| Inside global | `203.0.113.10` |

### Inside-to-Outside Packet

When the inside server initiates a connection to `198.51.100.20`, R1 changes the **source** address:

| Point in path | Source | Destination |
|---|---|---|
| Before NAT | `10.1.1.10:51000` | `198.51.100.20:443` |
| After NAT | `203.0.113.10:51000` | `198.51.100.20:443` |

Only the source **IP address** is translated. The TCP ports remain unchanged.

### Outside-to-Inside Packet

If a packet arrives from the outside addressed to the inside global address, R1 uses the same static mapping in reverse and changes the **destination** address:

| Point in path | Source | Destination |
|---|---|---|
| Before NAT | `198.51.100.20:443` | `203.0.113.10:51000` |
| After NAT | `198.51.100.20:443` | `10.1.1.10:51000` |

A static address mapping is not restricted to one application port. Subject to the routing path, other NAT rules, and security policies, packets for different protocols can use the same address mapping.

### Reachability of the Inside Global Address

**A static NAT statement does not automatically make the inside global address reachable throughout the outside network.**

In this example, `203.0.113.10` is not part of R1's outside transit subnet (`192.0.2.0/30`). The upstream network must have a route toward R1 for the translated address.

For example, on the adjacent ISP router:

```text
R2(config)# ip route 203.0.113.10 255.255.255.255 192.0.2.2
```

Alternatively, if the inside global address belongs to the directly connected outside subnet, ARP handling can be relevant. The routing and ARP requirements depend on how the translated address is allocated and connected.

A static NAT mapping **does not replace ordinary routing or firewall policy**.

See [Static Inside Source NAT](inside-source.md) for a complete configuration and routing examples.

---

## Static PAT and Port Forwarding

**Static PAT** maps a particular TCP or UDP port on an inside local address to a port on an inside global address.

The basic inside source syntax is:

```text
ip nat inside source static {tcp | udp} <inside-local> <local-port> <inside-global> <global-port>
```

Suppose the inside server listens on TCP port `443`, but outside users connect to `203.0.113.10` on TCP port `8443`.

```text
R1(config)# ip nat inside source static tcp 10.1.1.10 443 203.0.113.10 8443
```

Here, the outside-facing endpoint `203.0.113.10:8443` maps to `10.1.1.10:443`.

### Packet Arriving from Outside

An outside client sends a TCP connection request to `203.0.113.10:8443`:

| Point in path | Source | Destination |
|---|---|---|
| Before NAT | `198.51.100.20:52000` | `203.0.113.10:8443` |
| After NAT | `198.51.100.20:52000` | `10.1.1.10:443` |

R1 translates the destination IP address **and** TCP destination port.

The reverse translation is applied to replies from the server, so the outside client communicates with the expected outside endpoint.

```text
Outside client                      R1                         Server
198.51.100.20:52000            Static PAT                   10.1.1.10:443
        |                           |                              |
        +-- TCP to 203.0.113.10:8443 -->                           |
                                    +---- TCP to 10.1.1.10:443 ---->+
```

The diagram shows the logical translation; the upstream network must still deliver packets for `203.0.113.10` to R1.

A static PAT mapping does **not** automatically translate every connection from `10.1.1.10`, and it does not provide general Internet access for all inside hosts. Additional NAT rules may be needed for other traffic.

See [Static PAT and Port Forwarding](pat.md) for port preservation, port changes, TCP versus UDP mappings, and interactions with other NAT entries.

---

## Static Network-to-Network NAT

Individual static mappings can be configured for multiple hosts, but IOS XE also supports **static network mappings** on applicable platforms and releases.

For example, an organization may want each host in `10.1.1.0/24` represented by the corresponding address in `203.0.113.0/24`:

| Inside local | Inside global |
|---|---|
| `10.1.1.10` | `203.0.113.10` |
| `10.1.1.20` | `203.0.113.20` |
| `10.1.1.30` | `203.0.113.30` |

The mapping preserves the host portion of each address when translating between equal-length prefixes.

A supported IOS XE syntax is:

```text
ip nat inside source static network <local-network> <global-network> <mask>
```

For example:

```text
R1(config)# ip nat inside source static network 10.1.1.0 203.0.113.0 255.255.255.0
```

Some Cisco releases/documentation display prefix-length forms instead of the dotted-decimal mask form. Confirm the exact syntax using `?` on the target platform.

**Static network mapping is not PAT**, and it does not combine all hosts into a single translated IPv4 address. It creates a predictable address-to-address relationship between the networks.

See [Static Network-to-Network NAT](network-mapping.md) for configuration details, supported options, and troubleshooting.

---

## Static Outside Source NAT and PAT

Static NAT is not limited to translating inside hosts. **Outside source NAT** represents an outside host under a different address to the inside network.

For example, suppose an outside server really uses `198.51.100.20`, but inside hosts should use `172.16.20.20` to reach it.

The syntax is:

```text
ip nat outside source static <outside-global> <outside-local>
```

Configure:

```text
R1(config)# ip nat outside source static 198.51.100.20 172.16.20.20
```

| Address type | Value |
|---|---|
| Outside global | `198.51.100.20` |
| Outside local | `172.16.20.20` |

**Outside global comes first. Outside local comes second.** This is different from the argument names in the inside source command.

### How the Mapping Is Used

For traffic originating on the outside and traveling inside, R1 changes the **source** address from `198.51.100.20` to `172.16.20.20`.

In the reverse direction, when an inside host sends traffic to `172.16.20.20`, R1 translates the **destination** address to `198.51.100.20`.

Traditional outside source NAT is particularly sensitive to routing toward the **outside local** address before translation. Depending on the platform and design, a suitable route may need to be configured manually or installed using a supported `add-route` option.

IOS XE also supports static outside source TCP/UDP port mappings on applicable platforms, with outside-global address/port arguments preceding the outside-local address/port arguments.

See [Static Outside Source NAT and PAT](outside-source.md) for the full routing and translation sequence.

---

## Interface-Based Static Mappings and Default Inside Server

Static NAT and PAT can also use the IP address of a configured interface as the **inside global** address, rather than specifying a literal address.

For example, to publish an SSH service running on an inside host using TCP port `2222` on R1's outside interface:

```text
R1(config)# ip nat inside source static tcp 10.1.1.10 22 interface GigabitEthernet2 2222
```

Outside clients connecting to `192.0.2.2:2222` are translated to `10.1.1.10:22`, provided they can reach that interface address and security controls allow the connection.

This is **static PAT**, not interface-overload PAT. It does not require the `overload` keyword.

IOS XE also documents a **NAT Default Inside Server** feature for redirecting certain inbound packets that do not match more-specific existing translations. The feature is useful in specialized NAT designs, but should not be mistaken for a firewall DMZ or assumed to be supported identically on every router and release.

See [Interface NAT and Default/DMZ Host](interface-nat.md) for address-only interface mappings, precedence considerations, and the security implications of a default inside server.

---

## Static NAT Options

Static NAT supports optional keywords that change how a translation behaves. Availability and syntax vary by platform, release, and translation type.

| Option | Purpose |
|---|---|
| `extendable` | Allows supported overlapping or additional static mappings that otherwise may conflict; particularly useful when distinct mappings need extra identifying information |
| `no-alias` | Suppresses creation of a NAT alias for the global address; can affect how the router responds to ARP for that address |
| `no-payload` | Prevents NAT from modifying supported application payload information for the static mapping |
| `add-route` | On supported outside source NAT forms, automatically installs a route associated with the outside-local translation |
| `vrf` | Associates a supported static mapping with a particular VRF |
| `match-in-vrf` | Supports specific NAT designs where inside and outside traffic are in the same VRF |
| `stateless` | Uses a stateless static translation implementation, where supported, instead of creating per-flow state in the usual way |

These options are **not interchangeable** and are not available on every static NAT command. For instance, `add-route` is especially relevant to outside source NAT, not a generic requirement for inside source NAT.

Also distinguish **NAT alias/ARP behavior** from **routing**:

- ARP makes an address reachable on a directly connected Layer 2 segment.
- Routing determines the next hop toward an address or translated destination.
- A valid NAT mapping alone does not guarantee either one.

See [Static NAT Options and ARP Behavior](options.md) for detailed option behavior and configuration examples.

---

## Static NAT Translation Table Entries

Static address-only mappings normally appear in the NAT translation table even if no traffic has used them.

For example, after configuring:

```text
R1(config)# ip nat inside source static 10.1.1.10 203.0.113.10
```

A simple static entry can appear as:

```text
R1# show ip nat translations
Pro  Inside global      Inside local       Outside local     Outside global
---  203.0.113.10       10.1.1.10         ---               ---
```

The `---` value in the protocol column indicates a **simple address mapping**, not an error.

A static PAT entry specifies a transport protocol and port pair. For example, the earlier HTTPS mapping can appear as:

```text
R1# show ip nat translations
Pro  Inside global         Inside local         Outside local    Outside global
tcp  203.0.113.10:8443    10.1.1.10:443       ---              ---
```

Additional flow-specific entries may appear when traffic uses the mapping, depending on the platform and NAT implementation.

The static mapping itself normally remains present until its configuration is removed. **Associated flow entries can still have timers.**

Do not confuse the persistent static mapping with a dynamic translation or a flow entry that ages out due to inactivity.

For more about simple versus extended entries, parent/child entries, and aging, see [NAT Translation Tables and Verification](../verification.md).

---

## Verification and Troubleshooting

Useful commands include:

```text
R1# show running-config | include ip nat
R1# show ip nat translations
R1# show ip nat translations verbose
R1# show ip nat statistics
R1# show ip route
R1# show ip interface GigabitEthernet1
R1# show ip interface GigabitEthernet2
```

For a static NAT or PAT problem, verify the following in order:

1. **Correct mapping:** Confirm inside-local/inside-global or outside-global/outside-local argument order.
2. **Inside/outside roles:** Confirm the traffic traverses interfaces configured for the intended NAT direction.
3. **Outside reachability:** Confirm the upstream network can deliver packets for the inside-global address to R1.
4. **Inside reachability:** Confirm R1 can route translated traffic to the actual inside server.
5. **Traffic match:** For static PAT, check TCP versus UDP and both configured port numbers.
6. **Security policy:** Check interface ACLs, firewall policies, and whether a service is listening on the inside host.
7. **Translation state:** Check the translation table for static bindings and any flow-specific entries.
8. **Conflicts:** Check overlapping static mappings and interactions with dynamic NAT/PAT rules.

**A static translation appearing in `show ip nat translations` proves the mapping is configured. It does not prove that end-to-end forwarding or the application works.**

NAT does not inherently enforce an access policy. A published static mapping should be accompanied by appropriate ACLs or firewall controls where required.

---

## Static NAT/PAT Compared with Dynamic NAT/PAT

| Characteristic | Static NAT | Static PAT | Dynamic NAT | Dynamic PAT |
|---|---|---|---|---|
| Mapping configured in advance | Yes | Yes | Pool/policy configured, individual mappings allocated dynamically | Pool/interface/policy configured, individual flows allocated dynamically |
| Predictable translated endpoint | Yes | Yes | Not necessarily | Not necessarily |
| Can publish a fixed inbound service | Yes, subject to policy | Yes, for selected ports | Not normally the primary use | Requires additional inbound handling |
| Uses translation pool by default | No | No | Yes | Pool or interface |
| Shares a global address among many unrelated hosts | No, not with address-only one-to-one mapping | Can share an IP across different statically mapped endpoints | No, not without overloading | Yes |
| Common command keyword | `static` | `static tcp` / `static udp` | `list ... pool` | `overload` |

Dynamic NAT and overload are explained in [4.5.d (ii) Dynamic NAT, PAT](../dynamic/overview.md).

---

## Topics Covered in This Section

| Page | Focus |
|---|---|
| [Static Inside Source NAT](inside-source.md) | One-to-one host mappings, bidirectional flows, and routing requirements |
| [Static Network-to-Network NAT](network-mapping.md) | Subnet mappings and host-bit preservation |
| [Static PAT and Port Forwarding](pat.md) | TCP/UDP port translation and service publishing |
| [Interface NAT and Default/DMZ Host](interface-nat.md) | Interface-based mappings and fallback handling for inbound traffic |
| [Static Outside Source NAT and PAT](outside-source.md) | Outside global/local addresses, routing, and port mappings |
| [Static NAT Options and ARP Behavior](options.md) | `extendable`, `no-alias`, other keywords, and translated-address reachability |

The goal is to **configure a predictable mapping, trace both packet directions, and verify the routing and NAT state independently**.

---

## References

- [Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- [Cisco IOS XE 17.x: Information About Stateless Static NAT](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_stateless_static_nat.html)
- [Cisco IOS IP Addressing Services Command Reference: `ip nat inside source` and `ip nat outside source`](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr/command/ipaddr-cr-book/ipaddr-i3.html)
