# 4.2.b (i) IPv6 Traffic Filters

An **IPv6 traffic filter** is an IPv6 Access Control List (ACL) applied to an interface to **permit or deny IPv6 packets** based on fields such as source and destination addresses, upper-layer protocol, TCP/UDP ports, and ICMPv6 message types.

IPv6 ACLs are commonly used to:

- Restrict which IPv6 subnets can reach particular destinations or services.
- Block unwanted traffic entering or leaving a routed interface or switched virtual interface (SVI).
- Control access to services such as SSH, HTTPS, and DNS.
- Filter selected ICMPv6 messages while preserving the control traffic required for IPv6 operation.

IPv6 traffic filtering is separate from **IPv4 ACLs**. An IPv4 ACL does not filter IPv6 traffic, and an IPv6 ACL does not filter IPv4 traffic.

---

## IPv6 ACL Operation

Like an IPv4 extended ACL, an IPv6 ACL consists of ordered **Access Control Entries (ACEs)**. Each ACE specifies a permit or deny action and matching conditions.

Packets are processed as follows:

1. The router checks the first ACE against the packet.
2. If the ACE matches, its **permit** or **deny** action is taken. No subsequent entries are evaluated.
3. If it does not match, the router checks the next ACE.
4. If no configured ACE matches, the ACL's **implicit entries** determine the result.

The **first matching ACE wins**. An ACL is not a list of rules that all independently apply to the same packet.

For example:

```text
ipv6 access-list BLOCK_SSH
 deny tcp 2001:DB8:10::/64 host 2001:DB8:20::10 eq 22
 permit ipv6 any any
```

- TCP packets from `2001:DB8:10::/64` to `2001:DB8:20::10`, destination port **22**, are dropped.
- Other IPv6 traffic is permitted by the second ACE.
- A packet permitted by an ACL can still be dropped for another reason, such as a missing route or another security policy.

### IPv6 ACLs Versus IPv4 ACLs

| Characteristic | IPv4 ACL | IPv6 ACL |
|---|---|---|
| ACL types | Standard and extended | Extended-style filtering |
| ACL identification | Named or numbered | **Named only** |
| Address matching | IPv4 addresses and wildcard masks | IPv6 addresses and prefix lengths |
| Source and destination matching | Extended ACLs | Supported in IPv6 ACLs |
| TCP/UDP ports | Extended ACLs | Supported |
| ICMP filtering | ICMPv4 | ICMPv6 message types |
| Interface attachment | `ip access-group` | `ipv6 traffic-filter` |
| Implicit end behavior | Deny remaining IPv4 traffic | Neighbor Discovery permits followed by deny on many IOS XE platforms; see below |

IPv6 does **not** use the IPv4 ACL syntax of a network address followed by a wildcard mask. For example, use `2001:DB8:10::/64`, not an IPv4-style wildcard expression.

---

## Creating an IPv6 ACL

Create an IPv6 ACL with `ipv6 access-list` in global configuration mode:

```text
R1(config)# ipv6 access-list FILTER_IN
R1(config-ipv6-acl)# deny tcp 2001:DB8:10::/64 host 2001:DB8:20::10 eq 22
R1(config-ipv6-acl)# permit ipv6 any any
```

The basic ACE syntax is:

```text
{permit | deny} protocol source destination [options]
```

| Element | Meaning | Example |
|---|---|---|
| `permit` / `deny` | Action when the packet matches | `deny` |
| `protocol` | Upper-layer protocol or any IPv6 protocol | `ipv6`, `tcp`, `udp`, `icmp` |
| `source` | Source IPv6 host, prefix, or any | `2001:DB8:10::/64` |
| `destination` | Destination IPv6 host, prefix, or any | `host 2001:DB8:20::10` |
| `options` | Ports, ICMPv6 message type, logging, DSCP, etc. | `eq 443`, `log` |

### Address Matching

There are three common ways to specify a source or destination:

```text
any
host 2001:DB8:10::10
2001:DB8:10::/64
```

- `any`: Any IPv6 address.
- `host`: One exact IPv6 address (equivalent to a `/128` match).
- `2001:DB8:10::/64`: All addresses in that IPv6 prefix.

Examples:

```text
R1(config-ipv6-acl)# permit ipv6 2001:DB8:10::/64 any
R1(config-ipv6-acl)# deny ipv6 host 2001:DB8:10::99 host 2001:DB8:20::10
R1(config-ipv6-acl)# permit ipv6 any 2001:DB8:30::/64
```

These respectively permit traffic **from** a subnet, deny traffic between two particular hosts, and permit traffic **to** a subnet.

### Protocol Matching

Common protocol keywords include:

| Keyword | Matches |
|---|---|
| `ipv6` | All IPv6 packets, regardless of upper-layer protocol |
| `tcp` | TCP, protocol number 6 |
| `udp` | UDP, protocol number 17 |
| `icmp` | ICMPv6, protocol number 58 |
| Protocol number `89` | OSPFv3 packets |
| `esp` | IPsec ESP, protocol number 50 |
| `ahp` | IPsec AH, protocol number 51 |

Protocol numbers can be used where the corresponding named keyword is not provided.

**`permit ipv6 any any` permits all IPv6 protocols**, not just TCP, UDP, and ICMPv6.

### TCP and UDP Port Matching

Port conditions can be specified after the **source** and/or **destination** address.

```text
R1(config-ipv6-acl)# permit tcp any host 2001:DB8:20::10 eq 443
R1(config-ipv6-acl)# permit udp 2001:DB8:10::/64 any eq 53
R1(config-ipv6-acl)# permit tcp any eq 12345 host 2001:DB8:20::10 eq 443
```

In the third ACE:

- `eq 12345` after `any` matches the **source port**.
- `eq 443` after the destination host matches the **destination port**.

Common port operators include:

| Operator | Meaning | Example |
|---|---|---|
| `eq` | Equal | `eq 443` |
| `neq` | Not equal | `neq 22` |
| `lt` | Less than | `lt 1024` |
| `gt` | Greater than | `gt 1023` |
| `range` | Inclusive range | `range 10000 20000` |

An ACL matching TCP destination port 443 does **not** automatically permit the return traffic from source port 443. An IPv6 ACL is a **stateless packet filter**, so return traffic must satisfy the applicable ACL in its own direction.

---

## Implicit Entries and Neighbor Discovery

This is an important difference between IPv6 and IPv4 ACLs.

For many Cisco IOS/IOS XE implementations, an IPv6 ACL with at least one configured ACE ends with these **implicit entries**, in order:

```text
permit icmp any any nd-na
permit icmp any any nd-ns
deny ipv6 any any
```

The first two entries allow **ICMPv6 Neighbor Advertisement (NA)** and **Neighbor Solicitation (NS)** messages. These are part of IPv6 Neighbor Discovery (ND), which performs functions such as address resolution and Duplicate Address Detection (DAD).

The final entry drops unmatched IPv6 traffic.

> **Platform distinction:** Cisco's IOS XE ASR 1000 IPv6 ACL guide specifies that ASR 1000 ACLs **do not include the implicit ND permits**. For portable configurations that must allow NS/NA, permit those messages explicitly.

### Example: Implicit ND Permits

Consider this ACL:

```text
ipv6 access-list WEB_ONLY
 permit tcp any host 2001:DB8:20::10 eq 443
```

On an implementation with the implicit ND permits, the effective order is:

```text
permit tcp any host 2001:DB8:20::10 eq 443
! Implicit:
permit icmp any any nd-na
permit icmp any any nd-ns
deny ipv6 any any
```

Thus, TCP traffic to the server's HTTPS port and NS/NA messages are permitted, while unmatched IPv6 traffic is dropped.

### Explicit Entries Override Implicit Entries

The implicit permits occur **after all configured ACEs**. An explicit deny that matches an ND packet takes precedence:

```text
ipv6 access-list BLOCK_ND
 deny icmp any any nd-ns
 permit ipv6 any any
```

The `deny` blocks Neighbor Solicitations because the packet matches it before reaching any later entry.

Likewise:

```text
ipv6 access-list BLOCK_ALL
 deny ipv6 any any
```

An explicit `deny ipv6 any any` catches **all IPv6 packets**, including Neighbor Discovery messages, before any implicit permits can be reached.

To permit ND while explicitly denying everything else:

```text
R1(config)# ipv6 access-list ALLOW_ND_ONLY
R1(config-ipv6-acl)# permit icmp any any nd-ns
R1(config-ipv6-acl)# permit icmp any any nd-na
R1(config-ipv6-acl)# deny ipv6 any any
```

**Do not confuse these implicit ND permits with Router Solicitation (RS) and Router Advertisement (RA) permits.** Ordinary IOS XE IPv6 ACLs do not implicitly permit RS and RA in the same way. If an ACL must allow them, add explicit ACEs.

Also, if an ACL ends with an explicit `permit ipv6 any any`, then no unmatched traffic reaches the implicit deny.

---

## ICMPv6 Traffic Filtering

**ICMPv6 is essential to IPv6 operation.** It is used for Neighbor Discovery, Router Advertisements, error reporting, Path MTU Discovery, and diagnostics.

An IPv6 ACL can match the entire ICMPv6 protocol:

```text
R1(config-ipv6-acl)# permit icmp any any
```

Or a specific ICMPv6 type:

```text
R1(config-ipv6-acl)# permit icmp any any echo-request
R1(config-ipv6-acl)# permit icmp any any packet-too-big
R1(config-ipv6-acl)# deny icmp any any router-advertisement
```

### Important ICMPv6 Types

| Type | ICMPv6 message | IOS XE keyword | Purpose |
|---|---|---|---|
| 1 | Destination Unreachable | `destination-unreachable` | Destination or service cannot be reached |
| 2 | Packet Too Big | `packet-too-big` | Path MTU Discovery |
| 3 | Time Exceeded | `time-exceeded` | Hop-limit expiration and diagnostics |
| 4 | Parameter Problem | `parameter-problem` | Invalid IPv6 header or parameter |
| 128 | Echo Request | `echo-request` | Ping request |
| 129 | Echo Reply | `echo-reply` | Ping reply |
| 133 | Router Solicitation | `router-solicitation` | Host requests router information |
| 134 | Router Advertisement | `router-advertisement` | Router advertises prefixes and other settings |
| 135 | Neighbor Solicitation | `nd-ns` | Address resolution and DAD |
| 136 | Neighbor Advertisement | `nd-na` | Address resolution and neighbor reachability |
| 137 | Redirect | `redirect` | Router indicates a better first hop |

### Neighbor Discovery Versus Router Discovery

The four messages below serve different functions:

- **NS (135):** Used to discover an IPv6 neighbor's link-layer address and to detect duplicate addresses.
- **NA (136):** Used to supply neighbor information and signal reachability changes.
- **RS (133):** A host requests Router Advertisements.
- **RA (134):** A router advertises its availability, prefixes, and other IPv6 configuration information.

For a restrictive inbound ACL on a host-facing IPv6 interface, examples of explicit permits are:

```text
R1(config-ipv6-acl)# permit icmp any any nd-ns
R1(config-ipv6-acl)# permit icmp any any nd-na
R1(config-ipv6-acl)# permit icmp any any router-solicitation
```

For an outbound filter on an interface where hosts depend on router advertisements, an appropriate RA permit might be:

```text
R1(config-ipv6-acl)# permit icmp any any router-advertisement
```

The direction matters: hosts normally send RS messages toward routers, and routers send RA messages toward hosts.

**RA Guard** is a separate Layer 2 security feature designed to protect against unauthorized Router Advertisements. It is covered under IPv6 infrastructure security rather than ordinary router ACL filtering.

### Path MTU Discovery

IPv6 routers **do not fragment packets** in transit. When a packet is too large for the next link, a router can discard it and return an ICMPv6 **Packet Too Big** message (Type 2) to the source.

A filter that blocks these messages can cause **Path MTU Discovery failures**: small packets succeed, but larger packets or applications may stall.

For example:

```text
R1(config-ipv6-acl)# permit icmp any any packet-too-big
R1(config-ipv6-acl)# permit icmp any any destination-unreachable
R1(config-ipv6-acl)# permit icmp any any time-exceeded
R1(config-ipv6-acl)# permit icmp any any parameter-problem
```

These ACEs permit important IPv6 error messages. In a restrictive policy, place them **before** a broad deny and apply the policy in the direction where the error messages are expected.

---

## Applying an IPv6 ACL to an Interface

An IPv6 ACL is applied with the **`ipv6 traffic-filter`** interface command:

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# ipv6 traffic-filter FILTER_IN in
```

The syntax is:

```text
ipv6 traffic-filter ACL_NAME {in | out}
```

| Direction | Packets evaluated |
|---|---|
| `in` | IPv6 packets **received** on the interface |
| `out` | IPv6 packets **leaving** the interface |

For a packet traversing a router, a typical processing path is:

```text
Incoming interface
        |
        v
  Inbound IPv6 ACL
        |
        v
  Routing/forwarding
        |
        v
  Outbound IPv6 ACL
        |
        v
Outgoing interface
```

An IPv6 ACL can be applied to a **routed physical interface**, **subinterface**, or **SVI**, as supported by the device.

Creating an ACL alone does **not** activate interface filtering. It must be attached to an interface and direction.

### Example 1: Block SSH to a Specific IPv6 Server

Topology:

```text
Client network                               Server network
2001:DB8:10::/64                             2001:DB8:20::/64
      |                                             |
      | Gi1                                    Gi2  |
      +---------------- R1 -------------------------+
                                                    |
                                             2001:DB8:20::10
```

Requirement: Block clients in `2001:DB8:10::/64` from accessing SSH (TCP/22) on `2001:DB8:20::10`, but permit other IPv6 traffic.

```text
R1(config)# ipv6 access-list BLOCK_SERVER_SSH
R1(config-ipv6-acl)# deny tcp 2001:DB8:10::/64 host 2001:DB8:20::10 eq 22
R1(config-ipv6-acl)# permit ipv6 any any
!
R1(config)# interface GigabitEthernet1
R1(config-if)# ipv6 traffic-filter BLOCK_SERVER_SSH in
```

Expected results:

| Packet entering Gi1 | ACL result |
|---|---|
| Client to `2001:DB8:20::10`, TCP/22 | Denied |
| Client to `2001:DB8:20::10`, TCP/443 | Permitted |
| Client to another destination, TCP/22 | Permitted |
| ICMPv6 Echo Request | Permitted |

Because the second ACE permits all remaining IPv6 packets, this is a **selective deny policy**, not an allowlist.

### Example 2: Permit Only Selected Services

Requirement: From `2001:DB8:10::/64`, allow DNS queries and HTTPS traffic to `2001:DB8:20::10`. Permit essential ICMPv6 messages, but deny other traffic.

```text
R1(config)# ipv6 access-list CLIENT_POLICY
R1(config-ipv6-acl)# permit tcp 2001:DB8:10::/64 host 2001:DB8:20::10 eq 443
R1(config-ipv6-acl)# permit udp 2001:DB8:10::/64 host 2001:DB8:20::10 eq 53
R1(config-ipv6-acl)# permit tcp 2001:DB8:10::/64 host 2001:DB8:20::10 eq 53
R1(config-ipv6-acl)# permit icmp any any nd-ns
R1(config-ipv6-acl)# permit icmp any any nd-na
R1(config-ipv6-acl)# permit icmp any any router-solicitation
R1(config-ipv6-acl)# permit icmp any any packet-too-big
R1(config-ipv6-acl)# permit icmp any any destination-unreachable
R1(config-ipv6-acl)# permit icmp any any time-exceeded
R1(config-ipv6-acl)# permit icmp any any parameter-problem
R1(config-ipv6-acl)# deny ipv6 any any log
!
R1(config)# interface GigabitEthernet1
R1(config-if)# ipv6 traffic-filter CLIENT_POLICY in
```

The explicit ICMPv6 permits are placed **before** the explicit deny. DNS uses both TCP and UDP port 53, so both are included.

The policy is applied **inbound** on the client-facing interface. It does not automatically govern return packets entering from the server-facing interface. Also, DHCPv6 or routing protocols may require additional permits if they must traverse that same filter.

### Removing or Replacing an Applied Filter

To remove the inbound IPv6 ACL from an interface:

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# no ipv6 traffic-filter BLOCK_SERVER_SSH in
```

To replace it, configure the new ACL and apply its name in the intended direction.

---

## Additional ACE Matching Options

### Sequence Numbers

IPv6 ACL entries are evaluated in **sequence-number order**. IOS XE normally assigns sequence numbers automatically when new entries are added.

To specify an ACE's sequence number explicitly, use the `sequence` option:

```text
R1(config)# ipv6 access-list TEST
R1(config-ipv6-acl)# deny tcp any any eq 22 sequence 10
R1(config-ipv6-acl)# permit tcp any any eq 443 sequence 20
R1(config-ipv6-acl)# permit ipv6 any any sequence 30
```

A later ACE can be inserted between existing sequence numbers:

```text
R1(config-ipv6-acl)# deny udp any any eq 161 sequence 15
```

The evaluation order becomes **10, 15, 20, 30**, regardless of the order in which the commands were entered.

Check the actual numbers with `show ipv6 access-list TEST`.

### TCP Flags and `established`

IPv6 ACLs can match TCP flags and support the `established` keyword:

```text
R1(config-ipv6-acl)# permit tcp any 2001:DB8:10::/64 established
```

`established` matches TCP packets with the **ACK or RST flag set**. It does **not** maintain a connection table or verify that a connection was genuinely established.

This means a packet carrying an ACK or RST flag may match even without a legitimate existing session. Use a stateful firewall when actual session tracking is required.

### Time-Based ACEs

A **time range** lets an ACE operate only during a defined period.

Example: Permit HTTPS traffic during weekday business hours, deny HTTPS outside those hours, and permit other traffic.

```text
R1(config)# time-range BUSINESS_HOURS
R1(config-time-range)# periodic weekdays 09:00 to 17:00
!
R1(config)# ipv6 access-list TIME_POLICY
R1(config-ipv6-acl)# permit tcp any host 2001:DB8:20::10 eq 443 time-range BUSINESS_HOURS
R1(config-ipv6-acl)# deny tcp any host 2001:DB8:20::10 eq 443
R1(config-ipv6-acl)# permit ipv6 any any
!
R1(config)# interface GigabitEthernet1
R1(config-if)# ipv6 traffic-filter TIME_POLICY in
```

When the time range is inactive, the first ACE does not permit matching traffic, so the second ACE denies HTTPS to the server. The router's clock and time zone determine when the time range is active.

### DSCP Matching

IPv6 ACLs can filter by **Differentiated Services Code Point (DSCP)**, which is carried within the IPv6 Traffic Class field.

For example:

```text
R1(config-ipv6-acl)# deny ipv6 any any dscp 46
```

This denies packets marked with DSCP 46 (EF). The router is matching the **existing marking**, not changing it.

### IPv6 Fragment Matching

An IPv6 packet can carry a **Fragment extension header**. The initial fragment contains the upper-layer header; subsequent fragments normally do not contain the TCP or UDP port fields.

The `fragments` keyword can match **noninitial IPv6 fragments**:

```text
R1(config-ipv6-acl)# deny ipv6 any any fragments
```

The `fragments` keyword is used with the `ipv6` protocol match, not a TCP/UDP port-specific match.

**Why this matters:** A port-specific ACE cannot assume every fragment contains the TCP/UDP ports. When filtering fragmented IPv6 traffic, consider separate address/protocol-wide rules and fragment handling rather than relying only on port numbers.

### IPv6 Extension Headers

IPv6 can include a chain of **extension headers** between the base IPv6 header and the upper-layer header. ACL filtering can depend on locating the actual upper-layer protocol and its ports.

IOS XE supports additional match options on supported platforms, including fields such as a **routing extension header**, **destination options**, and **flow label**. Syntax and hardware support vary, so use the relevant CLI help and platform command reference before relying on these matches.

### Logging

Add `log` to an ACE to request logging of matching traffic:

```text
R1(config-ipv6-acl)# deny tcp any any eq 22 log
```

Or `log-input` to include incoming interface information where supported:

```text
R1(config-ipv6-acl)# deny tcp any any eq 22 log-input
```

Logging is useful for troubleshooting but may increase processing overhead. ACL logging is not necessarily generated for every individual packet, and logging support depends on the forwarding platform.

---

## IPv6 ACLs for Router Management (VTY)

An IPv6 ACL can also limit which clients may establish remote management sessions with a router.

For IPv6 VTY access control, use **`ipv6 access-class`** under the line, rather than `ipv6 traffic-filter` under an interface.

Example: Permit IPv6 management access only from `2001:DB8:100::/64`.

```text
R1(config)# ipv6 access-list VTY_IPV6
R1(config-ipv6-acl)# permit ipv6 2001:DB8:100::/64 any
R1(config-ipv6-acl)# deny ipv6 any any
!
R1(config)# line vty 0 15
R1(config-line)# ipv6 access-class VTY_IPV6 in
```

The ACL controls which **source addresses** may access the VTY lines over IPv6. Authentication and SSH configuration are separate requirements.

| Command | Applied under | Purpose |
|---|---|---|
| `ipv6 traffic-filter ACL in` | Interface | Filter IPv6 traffic arriving on an interface |
| `ipv6 traffic-filter ACL out` | Interface | Filter IPv6 traffic leaving an interface |
| `ipv6 access-class ACL in` | VTY line | Control incoming IPv6 remote-management connections |

A router interface ACL is not a replacement for a VTY access policy. The VTY access-class applies to management sessions independent of which eligible interface they arrive through.

---

## Verification and Troubleshooting

### Display IPv6 ACLs and Counters

```text
R1# show ipv6 access-list
R1# show ipv6 access-list BLOCK_SERVER_SSH
R1# show access-lists
```

Example output:

```text
R1# show ipv6 access-list BLOCK_SERVER_SSH
IPv6 access list BLOCK_SERVER_SSH
    deny tcp 2001:DB8:10::/64 host 2001:DB8:20::10 eq 22 (5 matches) sequence 10
    permit ipv6 any any (28 matches) sequence 20
```

The match counters help identify which ACEs are handling traffic. Hardware-forwarded traffic and implicit entries may not appear in software counters in the same way on all devices.

### Verify Interface Application

```text
R1# show running-config interface GigabitEthernet1
R1# show ipv6 interface GigabitEthernet1
R1# show running-config | section ipv6 access-list
```

Confirm **both the ACL name and direction**:

```text
interface GigabitEthernet1
 ipv6 address 2001:DB8:10::1/64
 ipv6 traffic-filter BLOCK_SERVER_SSH in
```

### Clear Match Counters

```text
R1# clear ipv6 access-list BLOCK_SERVER_SSH
```

Then generate test traffic and inspect the ACL again.

### Common Troubleshooting Scenarios

| Symptom | Likely cause | What to check |
|---|---|---|
| All traffic is blocked | Missing permit before implicit deny | ACE order and intended allow entries |
| SSH still succeeds | ACL attached in wrong direction or to wrong interface | Actual packet ingress/egress path |
| Unwanted traffic passes | An earlier broad `permit ipv6 any any` matches it | First-match evaluation |
| Neighbor resolution fails | Explicit broad deny blocks NS/NA, or platform lacks implicit ND permits | Explicit ND permits and ACL order |
| SLAAC fails | Required RS/RA traffic is filtered | ICMPv6 type 133/134 and interface direction |
| Small packets work; large transfers fail | ICMPv6 Packet Too Big is blocked | Type 2 messages in return path |
| An ACL entry never matches | Wrong prefix, transport protocol, source/destination port, or direction | Packet fields and ACE conditions |
| IPv4 traffic is unaffected | Only an IPv6 ACL is applied | IPv4 ACL is configured separately |
| IPv6 SSH management still works | Only an interface ACL was adjusted, not the relevant VTY policy | `ipv6 access-class` and management ingress path |

### Important Filtering Considerations

- **Link-local addresses:** IPv6 control messages often use `FE80::/10` source addresses. A policy written only for global unicast prefixes may fail to match the expected control traffic.
- **Multicast destinations:** Neighbor Discovery and routing protocols frequently use `FF02::/16` link-local multicast addresses. Do not assume they are ordinary unicast flows.
- **Routing protocol traffic:** OSPFv3 and other IPv6 control protocols may need explicit permits when applying restrictive filters to routing links.
- **ICMPv6:** Do not treat all ICMPv6 as optional diagnostic traffic. ND and error messages have operational functions.
- **Stateless operation:** An ACL permit for a client request does not create an automatic reverse-direction permit.
- **IPv4 separation:** `ipv6 traffic-filter` controls IPv6; IPv4 filtering requires its own configuration.

---

## Key Points

- An **IPv6 traffic filter** is a named IPv6 ACL applied with `ipv6 traffic-filter ACL_NAME in|out`.
- IPv6 ACLs use **IPv6 prefixes and prefix lengths**, not IPv4 wildcard masks.
- Entries are evaluated in **sequence order**, and the **first match wins**.
- IPv6 ACLs can filter by protocol, source/destination addresses, TCP/UDP ports, ICMPv6 type, DSCP, time range, and other supported fields.
- Most IOS XE implementations include implicit **ND-NA permit**, **ND-NS permit**, and **deny IPv6 any any** entries, but the ASR 1000 is a documented exception.
- An **explicit deny** matching ND traffic takes precedence over implicit ND permits.
- **RS and RA** are not covered by the ordinary implicit NS/NA permits.
- **Packet Too Big** must be considered when designing restrictive policies because it supports IPv6 Path MTU Discovery.
- IPv6 ACLs are **stateless** and do not automatically permit return traffic.
- For IPv6 VTY access, use `ipv6 access-class`, not `ipv6 traffic-filter`.

---

## References

- [Cisco: IPv6 Access Control Lists, IOS XE Security and VPN Configuration Guide](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_ip6-acls-xe.html)
- [Cisco: Cisco IOS IPv6 Command Reference, IPv6 ACLs and Traffic Filters](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipv6/command/ipv6-cr-book/ipv6-i1.html)
- [Cisco: Catalyst 9300 IPv6 ACL Command Reference](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/26-x/command_reference/b_26x_9300_cr/security_commands.html)
- [Cisco: Catalyst 9400 IPv6 ACL Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9600/software/release/26-x/configuration_guide/sec/b_26x_sec_9400_cg/configuring_ipv6_acls.html)
- [Cisco: Clear IPv6 ACL Counters Command Reference](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipv6/command/ipv6-cr-book/ipv6-a1.html)
- [Cisco: IPv6 ACL Undetermined-Transport Support](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_ipv6-undeter-trans-xe.html)
