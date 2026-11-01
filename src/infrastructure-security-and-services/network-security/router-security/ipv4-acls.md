# 4.2.b (ii) IPv4 Access Control Lists

An **IPv4 Access Control List (ACL)** is a sequence of **permit** and **deny** statements used to filter IPv4 traffic according to source and destination addresses, protocols, TCP/UDP ports, and other packet fields.

IPv4 ACLs are commonly used to:

- Restrict access between networks and to particular servers or services.
- Block unwanted traffic entering or leaving a routed interface or switched virtual interface (SVI).
- Limit management access to the router.
- Match traffic for other features, such as route maps, NAT, QoS, or route filtering.

An ACL's effect depends on **where it is used**. This page focuses on ACLs applied as **traffic filters**. When an ACL is referenced by another feature, `permit` and `deny` may indicate whether traffic matches that feature rather than directly forwarding or dropping packets.

IPv4 and IPv6 ACLs are separate. An IPv4 ACL does not filter IPv6 packets.

---

## IPv4 ACL Operation

An ACL consists of ordered **Access Control Entries (ACEs)**. A packet is checked against them from first to last:

1. The router compares the packet with the first ACE.
2. If the packet matches, the router takes that ACE's **permit** or **deny** action and stops checking the ACL.
3. If it does not match, the next ACE is checked.
4. If none of the configured ACEs matches, the packet is denied by the **implicit `deny ip any any`**.

**The first matching ACE wins.** Later entries do not override an earlier match.

Example:

```text
R1(config)# ip access-list extended BLOCK_SSH
R1(config-ext-nacl)# deny tcp 192.0.2.0 0.0.0.255 host 198.51.100.10 eq 22
R1(config-ext-nacl)# permit ip any any
```

- TCP packets from `192.0.2.0/24` to `198.51.100.10`, destination port **22**, are denied.
- All other IPv4 packets are permitted by the second ACE.
- A packet permitted by the ACL can still be dropped due to another ACL, an absent route, or another forwarding or security condition.

### Standard Versus Extended ACLs

IPv4 supports two principal ACL types:

| Characteristic | Standard ACL | Extended ACL |
|---|---|---|
| Source IPv4 address | Yes | Yes |
| Destination IPv4 address | **No** | Yes |
| IP protocol | No | Yes |
| Source/destination TCP or UDP ports | No | Yes |
| ICMP message type | No | Yes |
| Address matching | IPv4 address and wildcard mask | IPv4 address and wildcard mask |
| Numbered range | `1–99`, `1300–1999` | `100–199`, `2000–2699` |
| Named format | `ip access-list standard NAME` | `ip access-list extended NAME` |
| Typical placement | Near the destination of traffic being filtered | Near the source of traffic being filtered |

**Standard ACLs match the source IPv4 address only.** If applied inbound on a source-facing interface, they may block a source from reaching *all* destinations, not just the intended server. This is why the usual placement guideline is to put standard ACLs near the destination and extended ACLs near the source. It is a guideline, not a technical restriction.

### Implicit Deny

Every IPv4 ACL used as a packet filter has an implicit final deny:

```text
! Implicit, not displayed in the running configuration:
deny ip any any
```

For example, this ACL permits traffic from `192.0.2.0/24` and denies every other source:

```text
ip access-list standard ALLOWED_SOURCES
 permit 192.0.2.0 0.0.0.255
```

To permit everything not explicitly denied, configure a final permit:

```text
ip access-list extended SELECTIVE_BLOCK
 deny tcp any host 198.51.100.10 eq 23
 permit ip any any
```

There are **no implicit Neighbor Discovery permits** in IPv4 ACLs. The implicit final deny is not normally shown by `show ip access-lists`.

---

## Wildcard Masks

IPv4 ACLs identify address ranges using **wildcard masks**, not subnet masks or IPv6-style prefix lengths.

Each bit of a wildcard mask has the following meaning:

- `0` bit: The corresponding address bit **must match**.
- `1` bit: The corresponding address bit is **ignored** (don't care).

For ordinary contiguous subnets, the wildcard mask is the inverse of the subnet mask.

| Prefix length | Wildcard mask | ACL address match |
|---|---|---|
| `/32` | `0.0.0.0` | One exact IPv4 address |
| `/24` | `0.0.0.255` | 256 addresses |
| `/25` | `0.0.0.127` | 128 addresses |
| `/26` | `0.0.0.63` | 64 addresses |
| `/27` | `0.0.0.31` | 32 addresses |
| `/30` | `0.0.0.3` | 4 addresses |
| `/16` | `0.0.255.255` | 65,536 addresses |

For example:

```text
192.0.2.0 0.0.0.255
```

matches every address from `192.0.2.0` through `192.0.2.255`.

### `host` and `any`

The following expressions are equivalent:

```text
host 192.0.2.10
192.0.2.10 0.0.0.0
```

Both match **exactly one IPv4 address**.

Likewise:

```text
any
0.0.0.0 255.255.255.255
```

Both match **any IPv4 address**.

### Noncontiguous Wildcard Masks

Wildcard masks are **bit masks**, so their `0` and `1` bits do not have to be contiguous.

For example:

```text
10.0.0.0 0.255.255.0
```

matches addresses whose **first octet is `10` and fourth octet is `0`**, regardless of the middle two octets:

| Source address | Matches? | Explanation |
|---|---|---|
| `10.1.2.0` | Yes | First and last octets match |
| `10.200.50.0` | Yes | First and last octets match |
| `10.1.2.1` | No | Last octet differs |
| `11.1.2.0` | No | First octet differs |

A noncontiguous wildcard mask is **not equivalent to one ordinary subnet prefix**. It can describe a set of addresses spanning multiple subnets.

---

## Standard IPv4 ACLs

A standard ACL checks **only the source IPv4 address**, regardless of destination, IP protocol, or transport ports.

### Named Standard ACL

```text
R1(config)# ip access-list standard BRANCH_SOURCES
R1(config-std-nacl)# deny 192.0.2.0 0.0.0.255
R1(config-std-nacl)# permit any
```

This denies packets from `192.0.2.0/24` and permits packets from other sources **wherever the ACL is applied as a packet filter**.

### Numbered Standard ACL

The same policy can use a numbered ACL:

```text
R1(config)# access-list 10 deny 192.0.2.0 0.0.0.255
R1(config)# access-list 10 permit any
```

The traditional `access-list` command is entered in global configuration mode. Alternatively, a numbered ACL can be managed using ACL configuration mode:

```text
R1(config)# ip access-list standard 10
R1(config-std-nacl)# deny 192.0.2.0 0.0.0.255
R1(config-std-nacl)# permit any
```

### Standard ACL Placement Example

```text
192.0.2.0/24                     198.51.100.0/24
   Clients                           Servers
      |                                  |
    Gi1                                 Gi2
      +--------------- R1 ---------------+
```

Requirement: Deny clients in `192.0.2.0/24` from reaching the server network behind `Gi2`, without unnecessarily blocking their access to other networks.

```text
R1(config)# ip access-list standard BLOCK_CLIENTS
R1(config-std-nacl)# deny 192.0.2.0 0.0.0.255
R1(config-std-nacl)# permit any
!
R1(config)# interface GigabitEthernet2
R1(config-if)# ip access-group BLOCK_CLIENTS out
```

The filter checks the source address of IPv4 packets **leaving Gi2**. Traffic from the blocked clients destined for a different interface does not traverse that outbound ACL.

This policy applies to the entire server network behind Gi2. To restrict only a particular server or service, use an **extended ACL**.

---

## Extended IPv4 ACLs

Extended ACLs can match the **source and destination addresses**, the IP protocol, and additional protocol-specific fields.

Basic syntax:

```text
{permit | deny} protocol source [source-wildcard] destination [destination-wildcard] [options]
```

Example:

```text
R1(config)# ip access-list extended SERVER_POLICY
R1(config-ext-nacl)# deny tcp 192.0.2.0 0.0.0.255 host 198.51.100.10 eq 22
R1(config-ext-nacl)# permit ip any any
```

| Element | Meaning | Example |
|---|---|---|
| `permit` / `deny` | Action when the ACE matches | `deny` |
| `protocol` | IP protocol or all IPv4 protocols | `ip`, `tcp`, `udp`, `icmp` |
| `source` | Source host or network | `192.0.2.0 0.0.0.255` |
| `destination` | Destination host or network | `host 198.51.100.10` |
| Additional options | Port, ICMP type, flags, logging, time range, etc. | `eq 22`, `log` |

### Protocol Matching

| Protocol keyword | What it matches |
|---|---|
| `ip` | **All IPv4 protocols**, including TCP, UDP, ICMP, and routing protocols |
| `tcp` | TCP (IP protocol 6) |
| `udp` | UDP (IP protocol 17) |
| `icmp` | ICMPv4 (IP protocol 1) |
| `gre` | GRE (IP protocol 47) |
| `ospf` | OSPF (IP protocol 89) |
| `eigrp` | EIGRP (IP protocol 88) |
| `esp` | IPsec ESP (IP protocol 50) |
| `ahp` | IPsec AH (IP protocol 51) |

The protocol number may be used where supported:

```text
R1(config-ext-nacl)# permit 89 any any
```

This matches IP protocol **89** (OSPF).

`permit ip any any` means **all IPv4 protocols**. It is not limited to TCP and UDP.

### TCP and UDP Port Matching

A TCP/UDP ACE can check the **source port**, **destination port**, or both.

```text
R1(config-ext-nacl)# permit tcp any host 198.51.100.10 eq 443
R1(config-ext-nacl)# permit udp 192.0.2.0 0.0.0.255 any eq 53
R1(config-ext-nacl)# permit tcp any eq 12345 host 198.51.100.10 eq 443
```

For the third ACE:

- `eq 12345` immediately after `any` matches the **source TCP port**.
- `eq 443` after `host 198.51.100.10` matches the **destination TCP port**.

| Operator | Matches | Example |
|---|---|---|
| `eq` | Equal | `eq 443` |
| `neq` | Not equal | `neq 22` |
| `lt` | Less than | `lt 1024` |
| `gt` | Greater than | `gt 1023` |
| `range` | Inclusive range | `range 10000 20000` |

Common service keywords can replace numbers, such as `eq www` (TCP/80), `eq domain` (53), or `eq ssh` (22).

**ACLs are stateless.** An outbound permit for TCP destination port 443 does not automatically permit return packets through an inbound ACL on another interface. Return packets have **source port 443** and a client-selected destination port, so their fields must be evaluated separately.

### ICMP Matching

An extended ACL can match all ICMP traffic:

```text
R1(config-ext-nacl)# permit icmp any any
```

Or match a specific ICMP type:

```text
R1(config-ext-nacl)# permit icmp any any echo
R1(config-ext-nacl)# permit icmp any any echo-reply
R1(config-ext-nacl)# permit icmp any any unreachable
R1(config-ext-nacl)# permit icmp any any time-exceeded
```

| ICMP type | Keyword | Purpose |
|---|---|---|
| 0 | `echo-reply` | Ping reply |
| 3 | `unreachable` | Destination unreachable; also used for IPv4 Path MTU Discovery (fragmentation needed, code 4) |
| 8 | `echo` | Ping request |
| 11 | `time-exceeded` | TTL expired, commonly seen in traceroute |

For example, this ACE permits ping requests **to** a server:

```text
R1(config-ext-nacl)# permit icmp any host 198.51.100.10 echo
```

Whether the ping succeeds also depends on the ACLs affecting **the echo reply**.

---

## Applying IPv4 ACLs to Interfaces

An ACL is applied to an IPv4 routed interface with `ip access-group`:

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# ip access-group SERVER_POLICY in
```

Syntax:

```text
ip access-group ACL_NAME_OR_NUMBER {in | out}
```

| Direction | Packets evaluated |
|---|---|
| `in` | IPv4 packets arriving on the interface, **before the forwarding decision** |
| `out` | IPv4 packets leaving the interface, **after the forwarding decision** |

Simplified forwarding path:

```text
Incoming interface
        |
        v
  Inbound IPv4 ACL
        |
        v
   Route lookup
        |
        v
  Outbound IPv4 ACL
        |
        v
Outgoing interface
```

An IPv4 ACL can be attached to a routed physical interface, routed subinterface, or SVI.

Defining an ACL without applying `ip access-group` does **not** activate interface filtering. The same ACL can be applied on several interfaces, and its matches are evaluated from the perspective of each interface's direction.

A routed interface can have one IPv4 ACL in each direction. IPv6 traffic filters are configured separately.

### Traffic Destined for the Router

An **inbound** ACL can filter IPv4 packets destined for the router itself, as well as transit packets. For example, an inbound ACE denying TCP/22 to the router's interface address can prevent an SSH connection from being established.

An **outbound** ACL on a routed interface normally filters **transit traffic**, not IPv4 packets generated by the router. The exception is covered below.

### Locally Generated IPv4 Traffic

By default, **IPv4 traffic generated by the router bypasses outbound interface ACLs**. This includes locally generated ICMP messages and routing or management traffic leaving the router.

To include locally generated IPv4 packets in outbound interface ACL filtering:

```text
R1(config)# ip access-list match-local-traffic
```

This is a **global** command. After it is configured, locally generated IPv4 packets leaving an interface with an outbound IPv4 ACL are subject to that ACL.

```text
R1(config)# ip access-list extended OUTBOUND_POLICY
R1(config-ext-nacl)# deny icmp any any echo
R1(config-ext-nacl)# permit ip any any
!
R1(config)# interface GigabitEthernet2
R1(config-if)# ip access-group OUTBOUND_POLICY out
```

If R1 sends a locally generated ping through Gi2:

- **By default:** The outbound ACL does not block R1's own Echo Requests.
- **With `ip access-list match-local-traffic`:** The outbound ACL can deny R1's Echo Requests.

The IPv6 equivalent is `ipv6 access-list match-local-traffic`.

---

## Configuration Examples

### Example 1: Block SSH to a Server

Topology:

```text
192.0.2.0/24                        198.51.100.0/24
    Clients                              Servers
       |                                    |
      Gi1                                  Gi2
       +--------------- R1 -----------------+
                                          198.51.100.10
```

Requirement: Deny SSH from `192.0.2.0/24` to `198.51.100.10`, but allow other IPv4 traffic.

```text
R1(config)# ip access-list extended BLOCK_SERVER_SSH
R1(config-ext-nacl)# deny tcp 192.0.2.0 0.0.0.255 host 198.51.100.10 eq 22
R1(config-ext-nacl)# permit ip any any
!
R1(config)# interface GigabitEthernet1
R1(config-if)# ip access-group BLOCK_SERVER_SSH in
```

| Packet entering Gi1 | ACL result |
|---|---|
| From `192.0.2.20` to `198.51.100.10`, TCP/22 | Deny |
| From `192.0.2.20` to `198.51.100.10`, TCP/443 | Permit |
| From `192.0.2.20` to another server, TCP/22 | Permit |
| From `192.0.2.20` to `198.51.100.10`, ICMP Echo | Permit |

This is a **selective deny policy** because the final ACE permits all other IPv4 traffic.

### Example 2: Permit Selected Services to a Server

Requirement: Clients in `192.0.2.0/24` can access the server `198.51.100.10` using **HTTPS (TCP/443)** and **DNS (UDP/TCP 53)**, but cannot send other IPv4 traffic to that server. Traffic to other destinations remains permitted.

```text
R1(config)# ip access-list extended SERVER_SERVICES
R1(config-ext-nacl)# permit tcp 192.0.2.0 0.0.0.255 host 198.51.100.10 eq 443
R1(config-ext-nacl)# permit udp 192.0.2.0 0.0.0.255 host 198.51.100.10 eq 53
R1(config-ext-nacl)# permit tcp 192.0.2.0 0.0.0.255 host 198.51.100.10 eq 53
R1(config-ext-nacl)# deny ip 192.0.2.0 0.0.0.255 host 198.51.100.10
R1(config-ext-nacl)# permit ip any any
!
R1(config)# interface GigabitEthernet1
R1(config-if)# ip access-group SERVER_SERVICES in
```

The fourth ACE catches other traffic from the client subnet to the server. Without it, the final `permit ip any any` would also permit unwanted services to that server.

This inbound client-facing ACL does not automatically filter server responses returning through Gi2. Any other ACL on the return path is evaluated separately.

### Example 3: Block a Source Subnet from Multiple Destinations

Requirement: Block TCP/23 (Telnet) from `192.0.2.0/24` to any destination; permit all other IPv4 traffic.

```text
R1(config)# ip access-list extended BLOCK_TELNET
R1(config-ext-nacl)# deny tcp 192.0.2.0 0.0.0.255 any eq 23
R1(config-ext-nacl)# permit ip any any
!
R1(config)# interface GigabitEthernet1
R1(config-if)# ip access-group BLOCK_TELNET in
```

This matches **destination TCP port 23**. It does not deny every TCP packet whose **source** port is 23.

### Removing an Interface ACL

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# no ip access-group BLOCK_TELNET in
```

Removing the interface attachment does not delete the ACL's definition. To delete the ACL itself:

```text
R1(config)# no ip access-list extended BLOCK_TELNET
```

---

## Sequence Numbers and ACL Editing

Named and numbered ACLs can be managed in **ACL configuration mode**, where each ACE has a sequence number. Sequence numbers determine evaluation order.

```text
R1(config)# ip access-list extended FILTER_IN
R1(config-ext-nacl)# 10 deny tcp any host 198.51.100.10 eq 22
R1(config-ext-nacl)# 20 permit ip any any
```

Insert a new ACE between sequences 10 and 20:

```text
R1(config-ext-nacl)# 15 deny udp any host 198.51.100.10 eq 161
```

The resulting evaluation order is **10, 15, 20**.

To delete only the second ACE:

```text
R1(config-ext-nacl)# no 15
```

### Resequencing an ACL

To renumber the ACEs so there is room for additional entries:

```text
R1(config)# ip access-list resequence FILTER_IN 10 10
```

- `FILTER_IN`: ACL to resequence.
- First `10`: Starting sequence number.
- Second `10`: Increment between entries.

For an ACL with three entries, the sequence numbers become `10`, `20`, and `30`, without changing the relative ACE order.

**An ACE entered without an explicit sequence number is normally appended to the ACL.** Reentering an ACL to add a more specific deny does not automatically place it before an existing broad permit. Specify a sequence number when inserting entries.

### Remarks

Remarks add documentation without matching packets:

```text
R1(config)# ip access-list extended FILTER_IN
R1(config-ext-nacl)# remark Deny SSH to the application server
R1(config-ext-nacl)# 10 deny tcp any host 198.51.100.10 eq 22
R1(config-ext-nacl)# 20 permit ip any any
```

---

## Additional ACE Matching Options

### TCP `established` and TCP Flags

A TCP ACL can use the `established` keyword:

```text
R1(config-ext-nacl)# permit tcp any 192.0.2.0 0.0.0.255 established
```

`established` matches TCP packets with **ACK or RST set**. It does **not** inspect a session table or verify that a legitimate TCP three-way handshake occurred.

- A normal returning TCP segment after connection setup generally has ACK set and may match.
- An initial SYN without ACK or RST does not match.
- A forged packet with ACK set can match even without a real established session.

Therefore, `established` is **not stateful inspection**. Additional TCP flag matching options may be available in extended ACL syntax.

### Time-Based ACLs

A **time range** makes an ACE active only during configured dates and/or hours.

Example: Permit HTTPS to a server only on weekdays from 09:00 to 17:00, deny HTTPS at other times, and permit other traffic.

```text
R1(config)# time-range BUSINESS_HOURS
R1(config-time-range)# periodic weekdays 09:00 to 17:00
!
R1(config)# ip access-list extended HTTPS_HOURS
R1(config-ext-nacl)# permit tcp any host 198.51.100.10 eq 443 time-range BUSINESS_HOURS
R1(config-ext-nacl)# deny tcp any host 198.51.100.10 eq 443
R1(config-ext-nacl)# permit ip any any
!
R1(config)# interface GigabitEthernet1
R1(config-if)# ip access-group HTTPS_HOURS in
```

During business hours, the first ACE matches HTTPS. Outside business hours, that ACE is inactive, so the next ACE denies HTTPS to the server.

Time ranges can also use **absolute** start and end times. Accurate device time, date, and time zone are necessary for predictable behavior.

### DSCP, Precedence, and ToS

Extended ACLs can inspect IPv4 service-marking fields.

Example: Match packets marked with **DSCP 46 (EF)**:

```text
R1(config-ext-nacl)# deny ip any any dscp 46
```

Additional options include `precedence` (the legacy three-bit IP Precedence field) and `tos` (legacy type-of-service fields). These are **matches on existing markings**; the ACL does not set or rewrite them.

### IP Fragment Filtering

IPv4 routers can encounter **fragmented IPv4 packets**. The **initial fragment** contains the TCP/UDP header, while **noninitial fragments** generally do not contain those transport-layer ports.

The `fragments` keyword matches **noninitial fragments**:

```text
R1(config-ext-nacl)# deny ip 192.0.2.0 0.0.0.255 host 198.51.100.10 fragments
```

The ACE above denies noninitial fragments from the specified source network to the specified server. It does **not** deny initial fragments or unfragmented packets.

Cisco IOS/IOS XE has specific processing rules for ACEs **without** the `fragments` keyword:

| ACE type | Noninitial fragment with matching Layer 3 fields |
|---|---|
| ACE matching only Layer 3 fields (`permit ip` or `deny ip`) | The ACE can permit or deny the fragment. |
| **Permit** ACE with Layer 4 conditions | Fragment can be **permitted based on the Layer 3 fields**, since its ports cannot be checked. |
| **Deny** ACE with Layer 4 conditions | The fragment is **not denied by that ACE**; evaluation continues with the next ACE. |
| ACE with `fragments` | Applies **only to noninitial fragments**; cannot contain Layer 4 matching conditions. |

For example:

```text
ip access-list extended BLOCK_SERVER_SSH
 deny tcp 192.0.2.0 0.0.0.255 host 198.51.100.10 eq 22
 permit ip any any
```

The TCP/22 deny filters the initial fragment of an SSH packet. A matching **noninitial fragment** cannot be checked for the TCP port, skips that port-based deny, and can match the later broad permit.

To deny **all noninitial fragments** from those clients to the server, insert a separate ACE:

```text
ip access-list extended BLOCK_SERVER_SSH
 deny tcp 192.0.2.0 0.0.0.255 host 198.51.100.10 eq 22
 deny ip 192.0.2.0 0.0.0.255 host 198.51.100.10 fragments
 permit ip any any
```

This additional ACE also blocks noninitial fragments from **other protocols and ports** between those endpoints. It cannot selectively identify TCP/22 in a noninitial fragment.

### Object Groups

**Object groups** let an ACL refer to a set of addresses or services rather than repeating nearly identical ACEs for each one.

Example using an IPv4 network object group:

```text
R1(config)# object-group network APP_SERVERS
R1(config-network-group)# host 198.51.100.10
R1(config-network-group)# host 198.51.100.11
!
R1(config)# ip access-list extended BLOCK_APP_SSH
R1(config-ext-nacl)# deny tcp 192.0.2.0 0.0.0.255 object-group APP_SERVERS eq 22
R1(config-ext-nacl)# permit ip any any
```

The first ACE matches client SSH traffic to **either** server in `APP_SERVERS`. Object groups can also contain networks and can be used for protocol/service matching with `object-group service`.

Object-group ACL support and maximum expanded ACE counts depend on the router or switch platform.

### ACL Logging

Use `log` on an ACE to generate logging information for matching traffic:

```text
R1(config-ext-nacl)# deny tcp any host 198.51.100.10 eq 22 log
```

Use `log-input` to include information about the **ingress interface** and, when available, source MAC:

```text
R1(config-ext-nacl)# deny tcp any host 198.51.100.10 eq 22 log-input
```

Logging is typically aggregated or rate-limited rather than producing a message for every packet. Logging can increase CPU load, so avoid enabling it indiscriminately on high-volume traffic.

---

## IPv4 ACLs for VTY Management Access

An IPv4 ACL can restrict which source addresses may establish remote-management sessions with a router.

To apply a VTY source restriction, use `access-class` under the VTY lines, rather than `ip access-group` under an interface.

Example: Allow SSH management connections only from `192.0.2.0/24`:

```text
R1(config)# ip access-list standard VTY_MANAGEMENT
R1(config-std-nacl)# permit 192.0.2.0 0.0.0.255
!
R1(config)# line vty 0 15
R1(config-line)# access-class VTY_MANAGEMENT in
R1(config-line)# transport input ssh
```

The implicit deny rejects other management source addresses. Authentication and SSH server configuration are separate requirements.

| Command | Location | Purpose |
|---|---|---|
| `ip access-group ACL in` | Routed interface | Filter incoming IPv4 packets |
| `ip access-group ACL out` | Routed interface | Filter outgoing IPv4 packets |
| `access-class ACL in` | VTY lines | Restrict inbound remote-management sources |

A VTY access-class applies to eligible management connections irrespective of which interface they arrive through. An interface ACL and VTY access-class can both restrict access.

---

## Verification and Troubleshooting

### Display ACLs and Match Counters

```text
R1# show ip access-lists
R1# show ip access-lists BLOCK_SERVER_SSH
R1# show access-lists
R1# show running-config | section ip access-list
```

Example:

```text
R1# show ip access-lists BLOCK_SERVER_SSH
Extended IP access list BLOCK_SERVER_SSH
    10 deny tcp 192.0.2.0 0.0.0.255 host 198.51.100.10 eq 22 (5 matches)
    20 permit ip any any (28 matches)
```

- The ACE sequence numbers show **evaluation order**.
- Match counters can help determine which ACE permits or denies traffic.
- The **implicit deny** is not displayed as a configurable ACE.
- On hardware-forwarding switches, software ACL counters may omit packets processed entirely in hardware.

### Verify Interface Attachment and Direction

```text
R1# show running-config interface GigabitEthernet1
R1# show ip interface GigabitEthernet1
```

Example:

```text
R1# show ip interface GigabitEthernet1
GigabitEthernet1 is up, line protocol is up
  Internet address is 192.0.2.1/24
  Inbound  access list is BLOCK_SERVER_SSH
  Outgoing access list is not set
```

Always verify the **actual ingress/egress interface**, the **ACL name or number**, and the **direction**.

### Clear ACL Counters

```text
R1# clear ip access-list counters BLOCK_SERVER_SSH
```

Generate test traffic, then display the ACL counters again. This helps identify the matching ACE and whether the policy operates as intended.

### Common Troubleshooting Scenarios

| Symptom | Likely explanation | What to check |
|---|---|---|
| All traffic is blocked | Missing permit before the implicit deny | ACE order and required permitted traffic |
| Traffic you intended to deny is permitted | An earlier broad permit matches | First-match evaluation and sequence numbers |
| ACL has no effect | Not attached, wrong interface, or wrong direction | `ip access-group`, routing path |
| A standard ACL blocks too much | It checks **source only** | Whether an extended ACL is needed |
| HTTPS works one way but replies fail | Stateless ACL does not automatically permit return packets | Source/destination ports and both directions |
| Ping fails but TCP works | ICMP blocked | Echo, echo-reply, or error messages |
| Large packets fail unexpectedly | Fragment handling or ICMP fragmentation-needed errors filtered | `fragments` rules and ICMP unreachable messages |
| ACL permits noninitial fragments unexpectedly | Port-based deny cannot check missing transport header | Noninitial fragment behavior and subsequent ACEs |
| A router-generated ping is not blocked by outbound ACL | Locally generated packets bypass that ACL by default | `ip access-list match-local-traffic` |
| VTY access is not restricted | Only interface ACLs configured | `access-class` on VTY lines |
| Matches not visible in software counters | Filtering occurs in hardware | Platform hardware ACL counters |
| Newly entered deny has no effect | It was appended after `permit ip any any` | Insert an earlier sequence number |

### Additional Filtering Considerations

- **An ACL is not a firewall session table.** `permit tcp ... established` matches TCP flags but does not validate the connection.
- **A route must still exist.** An ACL permit does not create reachability.
- **Inbound and outbound are relative to the router interface**, not to the user's concept of an upload or download.
- **Standard ACLs cannot select destinations or ports.** Use an extended ACL for those requirements.
- **ACLs referenced by NAT, PBR, QoS, and route maps** may use their `permit`/`deny` results as match criteria rather than as packet-filtering actions.
- An interface ACL on an SVI filters the **routed** traffic subject to that interface. It does not automatically filter all bridging between two hosts in the same VLAN. See [VACL and PACL](../switch-security/vacl-pacl.md) for Layer 2 filtering.

---

## Key Points

- IPv4 ACLs are ordered rules; **the first matching ACE determines the ACL result**.
- A **standard ACL** matches only the source IPv4 address; an **extended ACL** can match destinations, protocols, ports, and other fields.
- IPv4 ACLs use **wildcard masks**: `0` means match, `1` means ignore.
- The implicit final entry is `deny ip any any`; use an explicit final permit when desired.
- Numbered standard ACLs are `1–99` and `1300–1999`; numbered extended ACLs are `100–199` and `2000–2699`.
- `ip access-group ACL in|out` applies the packet filter to a routed interface.
- By default, locally generated IPv4 traffic **bypasses outbound interface ACLs**. `ip access-list match-local-traffic` changes that behavior globally.
- `established` matches **ACK or RST**; it is not stateful inspection.
- Noninitial IPv4 fragments normally lack Layer 4 ports, requiring special attention to **port-based ACEs** and the `fragments` keyword.
- `access-class` on VTY lines restricts inbound IPv4 remote-management source addresses.

---

## References

- [Cisco: Creating an IP Access List and Applying It to an Interface (IOS XE)](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-create-ip-apply-0.html)
- [Cisco: Catalyst 9300 IPv4 ACLs](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/26-x/configuration_guide/sec/b_26x_sec_9300_cg/configuring_ipv4_acls.html)
- [Cisco: Refining an IP Access List (Sequence Numbers, Time Ranges, Fragments)](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_data_acl/configuration/xe-16-12/sec-data-acl-xe-16-12-book/sec-refine-ip-al.html)
- [Cisco: IOS Security Command Reference (`ip access-list match-local-traffic`, `ip access-list resequence`)](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/security/d1/sec-d1-cr-book/sec-cr-i1.html)
- [Cisco: Object-group ACLs](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/security/acls/acls-configuration-guide/object-group-acl.html)
- [Cisco: Understanding Access Control List Logging](https://sec.cloudapps.cisco.com/security/center/resources/access_control_list_logging.html)
- [Cisco: Displaying and Clearing IP ACL Data](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-dply-clr-ald-acl-xe.pdf)
