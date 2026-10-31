# 4.2.b Router Security Features

**Router security features** control which IP packets a router accepts and forwards and help prevent traffic with spoofed source addresses from entering the network.

This section covers the three router security features in the CCIE Enterprise Infrastructure blueprint:

1. **IPv6 Traffic Filters:** Use IPv6 access control lists (ACLs) to permit or deny IPv6 traffic.
2. **IPv4 Access Control Lists:** Use IPv4 ACLs to permit or deny traffic based on addresses, protocols, and other packet fields.
3. **Unicast Reverse Path Forwarding (uRPF):** Check whether an incoming packet's source IP address is reachable according to the routing information used for forwarding.

**ACLs define an explicit traffic policy**, whereas **uRPF checks source reachability**. They can be used together, but they answer different questions.

---

## Router Security Features at a Glance

| Feature | Main purpose | Basis for permitting or dropping traffic |
|---|---|---|
| **IPv6 Traffic Filters** | Control IPv6 traffic entering or leaving an interface. | Ordered IPv6 ACL entries matching source/destination addresses, protocols, and other fields. |
| **IPv4 ACLs** | Control IPv4 traffic entering or leaving an interface. | Ordered standard or extended ACL entries. |
| **uRPF** | Help prevent source IP address spoofing. | Reverse-path reachability lookup in the Forwarding Information Base (FIB). |

IPv4 and IPv6 interface ACLs are generally **stateless**: permitting a packet in one direction does not automatically permit its return traffic. If traffic must be allowed in both directions, the policies must account for both directions.

uRPF is different. It does not check whether a packet is part of an established connection or whether its destination and transport-layer ports are authorized. It checks the packet's **source address** against the forwarding information.

---

## IPv6 Traffic Filters

**IPv6 traffic filters** use IPv6 ACLs to control IPv6 packets received or transmitted by a routed interface.

An IPv6 ACL can filter traffic based on fields such as:

- **Source and destination IPv6 addresses** and prefixes.
- **Upper-layer protocol**, such as TCP, UDP, or ICMPv6.
- **TCP/UDP port numbers** and supported TCP flags.
- **ICMPv6 message types**.

An IPv6 ACL is created with `ipv6 access-list` and can be applied to an interface with `ipv6 traffic-filter` in the inbound or outbound direction.

```text
R1(config)# ipv6 access-list IPV6_FILTER
R1(config-ipv6-acl)# ...
!
R1(config)# interface GigabitEthernet1
R1(config-if)# ipv6 traffic-filter IPV6_FILTER in
```

An ACL entry is evaluated in sequence, and the **first matching entry determines the result**. Packets that do not match an applicable permit entry are denied by the implicit deny at the end.

IPv6 ACL design also requires attention to **ICMPv6 Neighbor Discovery**, which performs essential functions such as IPv6 neighbor address resolution. The implicit handling of Neighbor Solicitation and Neighbor Advertisement messages is an important difference from IPv4 ACLs, where ARP is not an IPv4 packet.

See [IPv6 Traffic Filters](ipv6-traffic-filters.md) for ACL syntax, matching, implicit entries, interface application, and troubleshooting.

---

## IPv4 Access Control Lists

**IPv4 ACLs** filter IPv4 packets according to a sequence of `permit` and `deny` entries.

The two primary types are:

| ACL type | What it matches | Typical use |
|---|---|---|
| **Standard ACL** | Source IPv4 address. | Permit or deny traffic from particular hosts or networks. |
| **Extended ACL** | Source/destination IPv4 addresses, protocol, and additional fields such as TCP/UDP ports. | Control specific applications or flows between hosts and networks. |

For example, an extended IPv4 ACL can permit TCP traffic to a web server on port 443 while blocking other access to that server.

IPv4 ACLs can use **numbered or named** configurations. Named ACLs support convenient editing with sequence numbers. Other useful capabilities include **time-based entries**, **logging**, and **object groups**.

An ACL must be applied where it can affect the intended traffic. A common application is an inbound or outbound ACL on a routed interface:

```text
R1(config)# ip access-list extended IPV4_FILTER
R1(config-ext-nacl)# ...
!
R1(config)# interface GigabitEthernet1
R1(config-if)# ip access-group IPV4_FILTER in
```

The main processing rules are:

1. Entries are evaluated **top to bottom**.
2. **The first match wins**; later entries are not examined.
3. If no entry matches, the packet encounters an **implicit `deny ip any any`**.
4. An ACL that is defined but **not applied** does not filter packets merely because it exists.

See [IPv4 Access Control Lists](ipv4-acls.md) for ACL types, configuration, placement, special matches, verification, and troubleshooting.

---

## Unicast Reverse Path Forwarding

**Unicast Reverse Path Forwarding (uRPF)** helps prevent **IP source address spoofing** by checking the source IP address of a packet as it enters a router interface.

The router uses the **Forwarding Information Base (FIB)** to determine whether the packet's source address is reachable. Two principal modes are available:

| Mode | Requirement | Typical application |
|---|---|---|
| **Strict (`rx`)** | A valid reverse path to the source must use the interface on which the packet arrived. | Networks with predictable, symmetric routing paths. |
| **Loose (`any`)** | A valid route to the source must exist through an acceptable interface, not necessarily the receiving interface. | Networks that may use asymmetric routing. |

For IPv4, the main interface commands are:

```text
R1(config-if)# ip verify unicast source reachable-via rx
R1(config-if)# ip verify unicast source reachable-via any
```

Consider this example:

```text
                    R1
              Gi1       Gi2
               |         |
           LAN A       ISP
       192.0.2.0/24
```

If R1 receives a packet on **Gi1** with a source address belonging to `192.0.2.0/24`, strict uRPF passes the reverse-path check when the FIB points back to that source through Gi1.

If the same packet arrives on **Gi2**, strict uRPF fails when the reverse path points to Gi1. Loose uRPF can pass the packet if the source address has a valid route through another interface.

**Strict uRPF can drop legitimate traffic on asymmetric paths.** Loose uRPF accommodates asymmetric paths but provides a weaker spoofing check. Default-route handling and ACL-based exceptions are also important uRPF configuration topics.

See [Unicast Reverse Path Forwarding](urpf.md) for strict and loose checks, FIB lookups, CEF requirements, default-route behavior, exceptions, verification, and troubleshooting.

---

## How ACLs and uRPF Work Together

Suppose a router connects an internal network to an external network.

- An **IPv4 ACL** can block traffic destined for a particular server or TCP/UDP port.
- An **IPv6 ACL** can enforce an equivalent policy for IPv6 traffic.
- **uRPF** can reject incoming packets whose source addresses fail the configured reverse-path check, even if an ACL would otherwise permit them.

For example, an ACL might permit HTTPS traffic to a server, but that permission does **not** mean every packet claiming to come from an arbitrary source address should be accepted. uRPF can provide a separate source-reachability check.

Likewise, a packet can pass uRPF while being denied by an ACL. **Passing uRPF does not mean the traffic is authorized by the ACL policy.**

### Router ACLs Versus Switch ACLs

The preceding [Switch Security Features](../switch-security/overview.md) section covers **PACLs** and **VACLs**, which can filter traffic in the Layer 2 switching path.

The IPv4 and IPv6 interface ACLs in this section primarily address **Layer 3 traffic filtering**. An ACL applied to an SVI does not ordinarily filter two hosts communicating directly within the same VLAN without routing.

---

## Key Points

- **IPv4 and IPv6 ACLs** determine whether traffic is permitted based on configured packet-matching rules.
- **ACL entry order matters:** The first matching entry determines the result.
- **IPv4 standard ACLs** primarily match source addresses; **extended ACLs** support more specific traffic matching.
- **IPv6 ACLs** require attention to ICMPv6 Neighbor Discovery and their implicit entries.
- **uRPF** validates the reachability of a packet's source address using the FIB.
- **Strict uRPF** checks whether the reverse path uses the receiving interface; **loose uRPF** checks reachability without requiring the same interface.
- **ACLs and uRPF are complementary:** A packet must pass all applicable checks to be forwarded.

---

## References

- [Cisco: CCIE Enterprise Infrastructure v1.1 Exam Topics](https://learningcontent.cisco.com/documents/marketing/exam-topics/CCIEEI-v1.1-Final_clean.pdf)
- [Cisco: IPv6 Access Control Lists](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_ip6-acls-xe.html)
- [Cisco: Access Control Lists](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/security/acls/acls-configuration-guide/access-control-lists.html)
- [Cisco: Unicast Reverse Path Forwarding Strict Mode](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_cfg-unicast-rpf-0.html)
