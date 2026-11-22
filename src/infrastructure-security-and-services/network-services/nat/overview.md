# 4.5.d IPv4 Network Address Translation

**Network Address Translation (NAT)** is a mechanism that changes IP addresses in packets as they pass through a router or other NAT device. Depending on the type of translation, NAT can also modify transport-layer port numbers.

NAT is commonly used to allow devices with **private IPv4 addresses** to communicate with external networks using one or more globally reachable addresses. However, NAT is not limited to Internet access or private-to-public address translation. It can also be used to publish internal services, connect networks with overlapping address spaces, or translate traffic between different routing contexts.

Cisco IOS XE supports several NAT mechanisms, ranging from basic static mappings to policy-based, VRF-aware, and VASI-based NAT.

---

## Why NAT Is Used

The main reasons to deploy NAT include:

- **IPv4 address conservation:** Multiple hosts can share a small number of externally reachable IPv4 addresses by using Port Address Translation (PAT).
- **Internal addressing flexibility:** An organization can use private IPv4 addresses internally without assigning a public address to every host.
- **Publishing internal services:** Static NAT or PAT can make a server reachable through an address or port used on the outside network.
- **Overlapping address spaces:** Two networks that use identical or overlapping IPv4 prefixes can communicate after appropriate address translation.
- **Policy-based traffic handling:** Different traffic flows can use different translations, for example when connecting to multiple ISPs.
- **VRF connectivity:** NAT can provide unique address representations for traffic associated with different VRFs.

NAT is an address-translation function, **not a routing protocol, firewall, or encryption mechanism**. Routing must still deliver packets to and from the NAT device, and access controls must be configured separately where required.

---

## Basic NAT Example

Consider a router connecting an internal network to an external network:

```text
   Inside network              NAT router                  Outside network

   PC1                            R1                              Server
   10.10.10.10              G1          G2                   198.51.100.20
       |                     |          |                           |
       +---------------------+          +---------------------------+
                        10.10.10.1    203.0.113.2
                        ip nat inside ip nat outside
```

PC1 opens an HTTPS connection to the server. Suppose PC1 uses TCP source port `51514`, and R1 translates the connection using PAT:

| Packet | Source | Destination |
|---|---|---|
| Before translation | `10.10.10.10:51514` | `198.51.100.20:443` |
| After translation | `203.0.113.2:40001` | `198.51.100.20:443` |

The server sees the connection as originating from `203.0.113.2:40001`, rather than `10.10.10.10:51514`.

R1 keeps a translation entry so that reply packets addressed to `203.0.113.2:40001` can be translated back to `10.10.10.10:51514` and forwarded to PC1.

The translated port is illustrative. PAT may preserve an original source port when possible rather than assigning a different one.

**Important:** NAT is bidirectional for an established translation. Although the example shows the source address being translated as traffic leaves the inside network, the destination address and port are translated on the return traffic.

---

## NAT Address Terminology

Cisco NAT uses four address terms. **Inside** and **outside** identify the sides of the NAT boundary, while **local** and **global** describe how an address is represented relative to that boundary.

| Term | Meaning | Example |
|---|---|---|
| **Inside local** | Address of an inside host as seen on the inside network | `10.10.10.10` |
| **Inside global** | Address used to represent that inside host to the outside network | `203.0.113.2` |
| **Outside local** | Address used to represent an outside host to the inside network | `198.51.100.20` |
| **Outside global** | Address of the outside host as it is known on the outside network | `198.51.100.20` |

In the basic PAT example, the **outside local and outside global addresses are identical** because the outside host is not being translated.

With **outside source NAT**, these addresses can differ. For example, an external server using `198.51.100.20` might be represented to inside hosts as `172.16.20.20`.

Do not confuse **inside/outside** with **private/public**. NAT can translate between private addresses, between public addresses, or between routing contexts where both networks use private addresses.

The complete address terminology, including how it applies to different NAT directions, is covered in [NAT Address Terminology and Translation Types](terminology.md).

---

## Static NAT, Dynamic NAT, and PAT

NAT configurations differ in how translations are established and whether transport-layer port numbers are changed.

| Mechanism | Translation behavior | Typical use |
|---|---|---|
| **Static NAT** | Manually configured, persistent one-to-one address mapping | Publish a server using a fixed translated address |
| **Dynamic NAT** | Assigns an address from a configured pool when matching traffic creates a translation | Provide temporary one-to-one mappings from a shared pool |
| **Static PAT** | Manually maps an address and protocol/port combination to another address and protocol/port combination | Forward an external TCP or UDP port to an internal service |
| **Dynamic PAT** | Dynamically translates address/port combinations, allowing multiple inside hosts to share an address | Provide Internet access for many inside clients |

### NAT vs. PAT

Traditional address-only NAT translates IP addresses without using transport-layer port numbers to distinguish hosts sharing the same translated address.

**Port Address Translation (PAT)**, also called **NAT overload**, allows multiple connections to share one translated IPv4 address by distinguishing translations with protocol and port information (or other protocol-specific identifiers where supported).

For example, two inside hosts could simultaneously connect to the same server using the same translated IPv4 address, provided their translated connection identifiers differ.

### Static vs. Dynamic

- **Static** mappings are explicitly configured and do not depend on the first matching packet to allocate an address mapping.
- **Dynamic** mappings are created as matching traffic is processed and are subject to translation state, address availability, and timeouts.

Static/dynamic and NAT/PAT describe different aspects of a translation. Therefore, both static PAT and dynamic PAT are possible.

---

## NAT Direction and Translation Types

The most common configuration is **inside source NAT**. Other NAT forms change which address is represented and which traffic initiates or matches a translation.

| Type | General purpose |
|---|---|
| **Inside source NAT** | Translate an inside host's source address for traffic toward the outside; translate the corresponding destination on return traffic |
| **Outside source NAT** | Translate an outside host's address as represented to inside hosts, often to deal with overlapping addressing |
| **Inside destination NAT** | Translate traffic directed at a virtual destination to one of several inside destinations, such as a pool of servers |

On IOS XE, these behaviors use command families such as `ip nat inside source`, `ip nat outside source`, and `ip nat inside destination`.

The **inside** and **outside** designations are normally configured on interfaces using `ip nat inside` and `ip nat outside`.

These translation types should not be confused with the direction in which a flow was initiated. For example, a static inside source mapping can also allow an outside host to initiate traffic toward the inside global address.

---

## Basic IOS XE Configuration

The following example configures **dynamic PAT using the outside interface's IPv4 address**.

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# ip address 10.10.10.1 255.255.255.0
R1(config-if)# ip nat inside

R1(config)# interface GigabitEthernet2
R1(config-if)# ip address 203.0.113.2 255.255.255.252
R1(config-if)# ip nat outside

R1(config)# access-list 10 permit 10.10.10.0 0.0.0.255
R1(config)# ip nat inside source list 10 interface GigabitEthernet2 overload
R1(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

The configuration has four main parts:

1. `ip nat inside` identifies the interface facing the inside network.
2. `ip nat outside` identifies the interface facing the outside network.
3. The ACL identifies the inside source addresses eligible for translation.
4. `ip nat inside source ... overload` enables PAT using the address of `GigabitEthernet2`.

The default route allows R1 to forward traffic toward the outside network. The return path to the translated address must also be reachable.

**Note:** The ACL used to select traffic for NAT is not, by itself, a security ACL blocking unmatched packets. A `deny` in this ACL means the traffic is not selected for that NAT rule, not necessarily that it will be dropped.

---

## NAT and Packet Processing

NAT interacts closely with **routing, Policy-Based Routing (PBR), ACLs, and forwarding decisions**.

For traditional inside/outside NAT, a useful high-level distinction is:

| Direction | Simplified processing order |
|---|---|
| **Inside to outside** | Routing decision, then inside-to-outside NAT translation |
| **Outside to inside** | Outside-to-inside NAT translation, then routing decision |

This distinction matters because the route lookup may use either the original or the translated address, depending on the direction.

For example, when a packet arrives from an external host toward a statically mapped inside server, the destination is translated before the routing lookup that forwards the packet toward the inside host.

These are simplified descriptions of the classic forwarding path. Advanced NAT features and platform-specific forwarding behavior introduce additional considerations. The detailed processing sequence is covered in [NAT Packet Processing and Order of Operations](order-of-operations.md).

---

## Advanced NAT Features

### Policy-Based NAT and PAT

**Policy-based NAT** selects translations based on configured traffic-matching conditions, commonly using route maps.

Instead of translating every matching source through the same address or pool, a router can choose different translations based on factors such as the destination network or the selected egress interface.

This is useful for **multi-ISP designs**, conditional translation, and certain traffic-engineering scenarios.

### VRF-Aware NAT and PAT

**VRF-aware NAT** associates translations with particular VRF routing contexts. This can allow traffic from separate VRFs to use NAT without treating their addresses as belonging to one common global routing table.

VRF-aware NAT is especially useful when different tenants use the same inside address ranges.

The exact behavior depends on the NAT configuration and the use of options such as `vrf` and `match-in-vrf`.

### VASI NAT

**VRF-Aware Software Infrastructure (VASI)** uses a pair of virtual interfaces to create an internal forwarding path between routing contexts on the same IOS XE router.

NAT can be applied to traffic traversing this path, enabling **inter-VRF translation** on supported IOS XE platforms.

VASI itself is **not a NAT mechanism**. It provides the forwarding path through which NAT and other services can be applied.

The underlying VASI interface behavior is explained in [VASI](../../../network-infrastructure/routing-concepts/route-leaking/vasi.md). This section concentrates on applying NAT to VASI traffic.

**Platform note:** NAT Virtual Interface (NVI), configured in classic IOS using `ip nat enable`, is **not supported on Cisco IOS XE**. For inter-VRF NAT on supported IOS XE platforms, the relevant feature is VASI NAT.

---

## NAT Verification and Troubleshooting

A NAT configuration can be syntactically correct yet fail because of routing, translation selection, or return-path problems.

Common verification commands include:

```text
R1# show ip nat translations
R1# show ip nat translations verbose
R1# show ip nat statistics
R1# show running-config | section ip nat
R1# show ip route
R1# show ip cef
```

Useful questions during troubleshooting include:

- Does the packet enter through the expected inside or outside interface?
- Does the traffic match the intended NAT rule, ACL, or route map?
- Is the correct translation entry created, and does it contain the expected address and port?
- Does the routing table provide a path toward the destination after the relevant translation step?
- Can the remote network route replies toward the translated address?
- Is the return traffic traversing the same device that maintains the translation state?
- Is another NAT rule or existing translation taking precedence?

For VRF-aware NAT, also verify the **correct VRF routing table** and the routing between the relevant interfaces or VASI pair.

Detailed translation-table interpretation, timers, clearing translations, and troubleshooting are covered in [NAT Translation Tables and Verification](verification.md).

---

## Topics Covered in This Section

The CCIE Enterprise Infrastructure blueprint divides IPv4 NAT into five areas:

| Section | Topic |
|---|---|
| **4.5.d (i)** | [Static NAT, PAT](static/overview.md) |
| **4.5.d (ii)** | [Dynamic NAT, PAT](dynamic/overview.md) |
| **4.5.d (iii)** | [Policy-Based NAT, PAT](policy-based/overview.md) |
| **4.5.d (iv)** | [VRF-Aware NAT, PAT](vrf/overview.md) |
| **4.5.d (v)** | [VRF-Aware Software Infrastructure (VASI) NAT](vasi/overview.md) |

These pages cover both the basic mechanisms and more advanced applications, including **static port forwarding, outside source NAT, overlapping networks, multi-ISP NAT, and inter-VRF NAT**.

The goal is to understand not just the configuration commands, but also **which addresses are translated, when translations occur, how packets are routed, and how to verify the resulting forwarding behavior**.

---

## References

- [Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- [Cisco: Understand the NAT Order of Operation](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/6209-5.html)
- [Cisco: Configuring the VRF-Aware Software Infrastructure](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_conf-vasi.html)
- [Jedadiah Casey: Understanding and Configuring NAT for Cisco CCNA, CCNP and CCIE](https://wax-trax.medium.com/understanding-and-configuring-network-address-translation-nat-for-cisco-ccna-ccnp-and-ccie-7fef92fa654c)
