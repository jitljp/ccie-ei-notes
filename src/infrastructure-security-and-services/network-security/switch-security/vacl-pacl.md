# 4.2.a (i) VACL and PACL

A **VLAN Access Control List (VACL)** and a **Port Access Control List (PACL)** allow a switch to filter traffic without relying solely on routed-interface ACLs.

- A **VACL** filters traffic associated with specified VLANs, including traffic **bridged between hosts in the same VLAN**.

- A **PACL** filters traffic on a specified **Layer 2 switchport**. It can filter traffic before the switch forwards it to another port or routes it through an SVI.

Both are useful because an ACL applied to a switched virtual interface (SVI) does **not** normally filter traffic that is switched directly between two hosts in the same VLAN.

---

## VACL, PACL, and RACL Comparison

A **Router ACL (RACL)** is an ordinary IP ACL applied to a Layer 3 interface, including an SVI or routed physical interface.

| Characteristic | VACL | PACL | RACL |
|---|---|---|---|
| Applied to | VLANs | Layer 2 switchports or supported Layer 2 Port-Channels | Layer 3 interfaces, including SVIs |
| Main configuration | `vlan access-map` + `vlan filter` | `ip access-group` / `mac access-group` on a switchport | `ip access-group` on a routed interface or SVI |
| Intra-VLAN traffic | Yes | Yes, if it traverses the configured port | No, if traffic is only bridged |
| Inter-VLAN traffic | Can affect traffic entering or leaving a filtered VLAN | Can affect packets entering or leaving the configured port | Yes, for routed IP packets crossing the interface |
| Direction | No `in` or `out` when applied to a VLAN | `in` and, on supported platforms, `out` | `in` or `out` |
| IPv4 filtering | `match ip address` with an IPv4 ACL | IPv4 ACL | IPv4 ACL |
| IPv6 filtering | `match ipv6 address` with an IPv6 ACL (where supported) | IPv6 PACL, where supported | IPv6 ACL, where supported |
| Non-IP filtering | `match mac address` with a MAC ACL | MAC ACL | Not with an IP RACL |
| Policy decision | `action forward` / `action drop` in VLAN access-map | `permit` / `deny` ACEs | `permit` / `deny` ACEs |

For example:

```text
              VLAN 100                      VLAN 200
       H1 -------------------+         +---------------- H3
       192.0.2.11            |         |                 198.51.100.13
                            SW1 (L3 switch)
       H2 -------------------+         +---------------- H4
       192.0.2.12                     198.51.100.14
```

- **H1 to H2:** Bridged within VLAN 100. A VACL on VLAN 100 or an appropriate PACL can filter it; an SVI RACL generally cannot.

- **H1 to H3:** Routed between VLANs. A VACL affecting VLAN 100 or 200, a PACL on a traversed switchport, and an applicable RACL can affect the flow.

- **H3 to H4:** Bridged within VLAN 200. A VACL on VLAN 100 does not apply to this traffic.

---

## VLAN Access Control Lists (VACLs)

A VACL is configured as a **VLAN access-map**. The access-map contains a sequence of entries, each with an optional match condition and an action.

The main commands are:

```text
SW1(config)# vlan access-map MAP_NAME 10
SW1(config-access-map)# match ip address IP_ACL
SW1(config-access-map)# action drop
!
SW1(config)# vlan access-map MAP_NAME 20
SW1(config-access-map)# action forward
!
SW1(config)# vlan filter MAP_NAME vlan-list 100,200
```

The last command attaches the map to VLANs 100 and 200. **Defining an access-map without applying `vlan filter` does not enforce it on a VLAN.**

### VACL Processing

A VLAN access-map is evaluated in **ascending sequence-number order**. On Catalyst platforms supporting IPv6 VACLs, its protocol-specific match clauses classify three **distinct packet types**:

| VACL match command | Referenced ACL | Packets eligible to match |
|---|---|---|
| `match ip address ACL_NAME` | IPv4 ACL | IPv4 only |
| `match ipv6 address ACL_NAME` | IPv6 ACL | IPv6 only |
| `match mac address ACL_NAME` | Extended MAC ACL | Non-IP frames only, such as ARP |

An IPv4 packet is not tested against an IPv6 or MAC ACL; an IPv6 packet is not tested against an IPv4 or MAC ACL. **A MAC ACL does not match IPv4 or IPv6 packets**, even though those packets are carried in Ethernet frames.

> **MAC ACLs** only match **non-IP** traffic - frames that don't contain IPv4 or IPv6 packets (e.g., ARP).

Processing then proceeds as follows:

1. The switch examines the first entry.
2. If the packet matches that entry, the configured `action` is taken and processing of the access-map stops.
3. If there is no match, the switch examines the next entry.
4. If no entry matches, the result depends on which packet types the map matches, as described below.

Sequence numbers can be specified explicitly, such as `10`, `20`, and `30`. If omitted, entries are assigned sequence numbers automatically, typically in increments of 10.

Unlike a conventional interface ACL, a VACL **does not have an `in` or `out` direction** in its VLAN attachment command. Its matching conditions can still distinguish traffic by source and destination address.

### The Meaning of `permit` and `deny` in a VACL Match ACL

**This is the most important VACL distinction.**

An IPv4, IPv6, or MAC ACL referenced by a VACL is a **classification ACL**. It determines whether the packet matches the corresponding VLAN access-map entry. It does not directly determine the VACL forwarding action.

| Result in referenced ACL | VACL entry action | Result |
|---|---|---|
| Packet matches a `permit` ACE | `action drop` | Packet is dropped |
| Packet matches a `permit` ACE | `action forward` | Packet passes this VACL entry |
| Packet matches a `deny` ACE | Either action | Entry **does not match**; examine subsequent map entries |
| Packet matches no ACE (implicit ACL deny) | Either action | Entry **does not match**; examine subsequent map entries |

For example:

```text
ip access-list extended MATCH_HTTP
 permit tcp any any eq 80
!
vlan access-map WEB_FILTER 10
 match ip address MATCH_HTTP
 action drop
```

The ACL's `permit` identifies the HTTP traffic. The map's `drop` action actually blocks it.

**Do not write `deny tcp any any eq 80` in `MATCH_HTTP` expecting that line to block HTTP through the VACL.** A denied ACE is not a positive match for the VACL entry.

Also remember that `action forward` means the packet is **allowed by that VACL**, not that it bypasses other applicable security features.

### What Happens When No VACL Entry Matches?

The implicit action depends on the packet type and on whether the VLAN access-map contains a `match` clause for that same type.

On Catalyst IOS/IOS XE platforms that support IPv6 VLAN-map matching, there are three separate categories: **IPv4**, **IPv6**, and **non-IP**.

- **If the map contains at least one match clause for a packet's category**, packets in that category that match no entry are **dropped**.
- **If the map contains no match clause for that category**, packets in that category are **forwarded** if they have not been handled by an explicit entry.
- A clause for one category does **not** change the implicit action for another. For example, `match ip address` does not create an implicit drop for unmatched IPv6 packets.

The following table assumes that **no explicit VACL entry matches the packet**:

| Match clause types present in the map | Unmatched IPv4 | Unmatched IPv6 | Unmatched non-IP |
|---|---|---|---|
| IPv4 only | **Drop** | Forward | Forward |
| IPv6 only | Forward | **Drop** | Forward |
| MAC only | Forward | Forward | **Drop** |
| IPv4 + IPv6 | **Drop** | **Drop** | Forward |
| IPv4 + MAC | **Drop** | Forward | **Drop** |
| IPv6 + MAC | Forward | **Drop** | **Drop** |
| IPv4 + IPv6 + MAC | **Drop** | **Drop** | **Drop** |
| No `match` clauses | **Catch-all entry action** | **Catch-all entry action** | **Catch-all entry action** |

**Important: An entry without a `match` command matches all packet categories**, irrespective of whether the map contains IPv4, IPv6, or MAC matches in other entries. Its `action forward` or `action drop` is taken as soon as that entry is reached. It is **not** counted as a protocol-specific `match ip`, `match ipv6`, or `match mac` clause when determining implicit defaults; in practice, a reachable catch-all entry prevents packets from reaching an implicit default at all.

For example, an explicit permit-all entry at sequence 20 is:

```cisco
SW1(config)# vlan access-map WEB_FILTER 20
SW1(config-access-map)# action forward
```

All remaining IPv4, IPv6, and non-IP packets are forwarded by this entry. If no `action` is configured in an entry, its default action is also `forward` on Catalyst IOS XE.

By contrast, **a single `match ip address` + `action drop` entry is not a selective IPv4 deny policy**: unmatched IPv4 packets are also implicitly dropped. However, unmatched IPv6 and non-IP packets are forwarded when the map has no IPv6 or MAC match clauses.

> These rules describe Catalyst-style VLAN maps with support for IPv6 matches. Cisco's older explanations sometimes use the shorthand "IP or MAC" and cover only IPv4 versus non-IP traffic. On IPv6-capable models, `match ip`, `match ipv6`, and `match mac` refer to distinct protocol categories. Feature and hardware support remain platform/release dependent.

### Example 1: Block HTTP Between Two Hosts in the Same VLAN

Requirements:

- H1: `192.0.2.11`, VLAN 100.

- H2: `192.0.2.12`, VLAN 100.

- Block **TCP/80 from H1 to H2**.

- Permit other traffic, including ARP and non-HTTP IPv4 traffic.

```text
SW1(config)# ip access-list extended H1_TO_H2_HTTP
SW1(config-ext-nacl)# permit tcp host 192.0.2.11 host 192.0.2.12 eq 80
!
SW1(config)# vlan access-map VLAN100_FILTER 10
SW1(config-access-map)# match ip address H1_TO_H2_HTTP
SW1(config-access-map)# action drop
!
SW1(config)# vlan access-map VLAN100_FILTER 20
SW1(config-access-map)# action forward
!
SW1(config)# vlan filter VLAN100_FILTER vlan-list 100
```

The processing is:

| Packet | Sequence 10 | Sequence 20 | Result |
|---|---|---|---|
| H1 to H2, TCP destination port 80 | Matches | Not reached | Drop |
| H1 to H2, TCP destination port 443 | No match | Matches catch-all | Forward |
| H2 to H1, TCP destination port 80 | No match | Matches catch-all | Forward |
| H1 to H2, ICMP | No match | Matches catch-all | Forward |
| ARP request | No IP match | Matches catch-all | Forward |

The ACL is directional in terms of **source and destination**, even though the VACL attachment is not configured with `in` or `out`.

If both directions of HTTP must be blocked, add an additional `permit` ACE to `H1_TO_H2_HTTP` with source and destination reversed. TCP client/server traffic should be analyzed by its actual source and destination port numbers, not just by a conceptual "direction".

### Example 2: Block Access to a Server from Selected Clients

Suppose VLAN 100 contains server `192.0.2.100`. Block traffic **destined for this server** from:

- `192.0.2.11` in the same VLAN.

- `198.51.100.0/24` in another VLAN.

Allow other traffic.

```text
SW1(config)# ip access-list extended BLOCK_SERVER_CLIENTS
SW1(config-ext-nacl)# permit ip host 192.0.2.11 host 192.0.2.100
SW1(config-ext-nacl)# permit ip 198.51.100.0 0.0.0.255 host 192.0.2.100
!
SW1(config)# vlan access-map SERVER_POLICY 10
SW1(config-access-map)# match ip address BLOCK_SERVER_CLIENTS
SW1(config-access-map)# action drop
!
SW1(config)# vlan access-map SERVER_POLICY 20
SW1(config-access-map)# action forward
!
SW1(config)# vlan filter SERVER_POLICY vlan-list 100
```

This demonstrates that a VACL can address **both bridged and routed traffic associated with a VLAN**. The remote subnet does not have to be in VLAN 100; the traffic destined for the server enters VLAN 100 as it is delivered to that VLAN.

Note that the policy identifies the **server as the IP destination**. It does not automatically block the reverse address pair. To block the reverse direction as well, match those addresses explicitly.

### Example 3: MAC ACL Matching and ARP

VACLs can use a named **extended MAC ACL** to match **non-IP Ethernet traffic**, including ARP.

An ARP message is **not** an IP packet, and therefore it not matched by an IP ACL.

The following example drops ARP traffic in VLAN 100, then forwards all remaining traffic:

```text
SW1(config)# mac access-list extended MATCH_ARP
SW1(config-ext-macl)# permit any any 0x806 0x0
!
SW1(config)# vlan access-map BLOCK_ARP 10
SW1(config-access-map)# match mac address MATCH_ARP
SW1(config-access-map)# action drop
!
SW1(config)# vlan access-map BLOCK_ARP 20
SW1(config-access-map)# action forward
!
SW1(config)# vlan filter BLOCK_ARP vlan-list 100
```

`0x806` is the Ethernet **ARP EtherType**. The final `0x0` is the matching mask in this MAC ACL syntax.

**This is a deliberately disruptive demonstration**, not a normal security recommendation. Blocking ARP prevents ordinary IPv4 neighbors from dynamically resolving MAC addresses, causing connectivity failures unless a suitable alternative such as static ARP entries exists.

The important distinction is:

- `match ip address ...` evaluates **IPv4** packets against an IPv4 ACL.

- `match ipv6 address ...` evaluates **IPv6** packets against an IPv6 ACL (on supported platforms).

- `match mac address ...` evaluates **non-IP** frames against a MAC ACL.

- A MAC ACL does **not** match IPv4 or IPv6 packets, even though both use Ethernet MAC addresses.

### Example 4: Explicitly Permit Only Selected IPv4 Traffic

Suppose VLAN 100 should permit only TCP destination port 443 for the IPv4 traffic governed by the VACL.

```text
SW1(config)# ip access-list extended ALLOW_HTTPS
SW1(config-ext-nacl)# permit tcp any any eq 443
!
SW1(config)# vlan access-map HTTPS_ONLY 10
SW1(config-access-map)# match ip address ALLOW_HTTPS
SW1(config-access-map)# action forward
!
SW1(config)# vlan filter HTTPS_ONLY vlan-list 100
```

- Matching IPv4 HTTPS packets are forwarded by the map.

- Other IPv4 packets are **implicitly dropped** because the map has an IP match clause.

- **IPv6 traffic is forwarded** because the map contains **no IPv6 match clause**.

- **Non-IP traffic is forwarded** because the map contains **no MAC match clause**.

This does **not** create a protocol-independent allowlist: only IPv4 has been restricted. To restrict IPv6 as well, use `match ipv6 address` with an IPv6 ACL. Add a MAC match if non-IP traffic must also be restricted.

### Example 5: IPv6 Matching and the Implicit Default

Consider a VLAN map that references **only an IPv6 ACL**:

```cisco
SW1(config)# ipv6 access-list MATCH_IPV6
SW1(config-ipv6-acl)# permit ipv6 2001:db8:100::/64 any
!
SW1(config)# vlan access-map IPV6_FILTER 10
SW1(config-access-map)# match ipv6 address MATCH_IPV6
SW1(config-access-map)# action drop
!
SW1(config)# vlan filter IPV6_FILTER vlan-list 100
```

The results are:

| Packet | Result | Reason |
|---|---|---|
| IPv6 sourced from `2001:db8:100::/64` | **Drop** | Matches sequence 10 and its `action drop` |
| Other IPv6 traffic | **Drop** | An IPv6 match clause exists; unmatched IPv6 is implicitly dropped |
| IPv4 traffic | Forward | No IPv4 match clause exists |
| ARP and other non-IP traffic | Forward | No MAC match clause exists |

If the intention is to drop **only the matched IPv6 source prefix**, add a final catch-all `action forward` entry:

```cisco
SW1(config)# vlan access-map IPV6_FILTER 20
SW1(config-access-map)# action forward
```

All IPv6 packets not matched by sequence 10, plus IPv4 and non-IP traffic, now match sequence 20 and are forwarded.

### VACL Operational Rules

- A VLAN can have **only one applied VLAN access-map** at a time. One map can contain multiple entries and can be applied to multiple VLANs.

- The order of the map's sequence entries matters. An early catch-all makes later entries unreachable.

- A VACL can filter traffic in a VLAN even when **IP routing is disabled** on the switch.

- A VACL on a VLAN applies to relevant traffic traversing that switch through its access or trunk ports. It is **not configured individually on every access port**.

- Applying an IP ACL to an SVI does not replace the intra-VLAN filtering provided by a VACL.

- A VACL cannot inspect traffic that bypasses the switch.

- Platform-specific features, including PVLANs, IPv6 policy support, and CPU/control-plane handling, can affect actual enforcement. PVLAN isolation and VACL filtering solve different problems.

---

## Port Access Control Lists (PACLs)

A PACL is an ACL applied directly to a **Layer 2 interface**.

For common Catalyst IOS XE configurations, the attachment commands are:

```text
SW1(config-if)# ip access-group ACL_NAME in
SW1(config-if)# mac access-group MAC_ACL_NAME in
```

An IP ACL attached to a Layer 2 switchport is a **PACL**, not a routed ACL. The **interface type** determines its role, even though `ip access-group` is also used on Layer 3 interfaces.

A PACL may apply to a physical switchport or a supported Layer 2 Port-Channel. **Do not attach a Port-Channel PACL to individual EtherChannel member ports** when the platform requires the policy on the logical Port-Channel.

### PACL Direction and Scope

- **Ingress PACL:** Examines packets received on the configured switchport. This is the most common deployment and the basis of the following examples.

- **Egress PACL:** Examines packets leaving a switchport, if supported by the platform and release. Modern Catalyst 9000 documentation describes inbound and outbound PACLs; many older platforms support only inbound PACLs.

- **Access port:** Applies to traffic entering or leaving the access VLAN on that port.

- **Trunk port:** Applies to traffic of **all VLANs on that trunk**, not merely one selected VLAN.

- **Voice VLAN port:** The PACL can affect both the data VLAN and voice VLAN traffic on the port.

Unlike `vlan filter`, a PACL is attached to **one interface at a time**. Applying the same ACL to several ports is possible, but each interface needs the applicable attachment command.

### Example 6: Ingress IPv4 PACL on an Access Port

Requirements:

- H1 is connected to SW1's `GigabitEthernet1/0/1` in VLAN 100.

- Block IPv4 packets received from H1 **destined for** `192.0.2.12`.

- Allow other IPv4 packets.

```text
SW1(config)# ip access-list extended H1_PACL
SW1(config-ext-nacl)# deny ip any host 192.0.2.12
SW1(config-ext-nacl)# permit ip any any
!
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 100
SW1(config-if)# ip access-group H1_PACL in
```

This is **ordinary ACL permit/deny logic**, not VLAN access-map classification logic:

- An IPv4 packet received on Gi1/0/1 and destined for `192.0.2.12` is denied.

- Other IPv4 packets received on Gi1/0/1 match `permit ip any any`.

- The ACL's implicit deny would apply to unmatched IPv4 packets if the final permit entry were omitted.

- This particular IPv4 PACL does **not** filter ARP. ARP can complete even when the corresponding IPv4 data traffic is denied.

The policy does not need `ip routing`, an SVI, or a routed interface to filter bridged IPv4 packets.

### Example 7: Apply a PACL to a Trunk

```text
SW1(config)# ip access-list extended BLOCK_TELNET
SW1(config-ext-nacl)# deny tcp any any eq 23
SW1(config-ext-nacl)# permit ip any any
!
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 100,200,300
SW1(config-if)# ip access-group BLOCK_TELNET in
```

The PACL filters **ingress IPv4 traffic received on Gi1/0/24 across VLANs 100, 200, and 300**. It is not a per-VLAN policy.

If Telnet must be blocked only in VLAN 100, a VACL on VLAN 100 may be more appropriate, or the policy needs another platform-supported means of matching VLAN identity.

### Example 8: MAC PACL for Non-IP Traffic

A MAC PACL can match **non-IP traffic**, including ARP. For example, allow non-IP frames only when their source MAC is `0001.aaaa.aaaa`:

```text
SW1(config)# mac access-list extended ALLOWED_MAC
SW1(config-ext-macl)# permit host 0001.aaaa.aaaa any
!
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# mac access-group ALLOWED_MAC in
```

On this port, other non-IP frames can be dropped by the MAC ACL's implicit deny. For instance, ARP packets sent with a different source MAC may be blocked, preventing IP reachability even if an IP PACL permits the ensuing IPv4 traffic.

A MAC ACL is **not a substitute for port security, DHCP snooping, or IP Source Guard**. Filtering a source MAC is not sufficient to prevent deliberate MAC spoofing by itself.

### Applying IP and MAC PACLs Together

A supported Layer 2 interface can use **one IPv4 ACL and one MAC ACL simultaneously**:

```text
SW1(config)# ip access-list extended IP_POLICY
SW1(config-ext-nacl)# deny tcp any any eq 23
SW1(config-ext-nacl)# permit ip any any
!
SW1(config)# mac access-list extended NON_IP_POLICY
SW1(config-ext-macl)# permit any any
!
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# ip access-group IP_POLICY in
SW1(config-if)# mac access-group NON_IP_POLICY in
```

In this example:

- The IP PACL drops matching IPv4 Telnet traffic and allows other IPv4 traffic.

- The MAC PACL permits non-IP frames, including ARP.

- The two ACLs classify **different traffic categories**, not two independent checks for every packet.

This is one reason a switchport can have both an `ip access-group` and a `mac access-group` applied without one replacing the other.

On platforms offering IPv6 PACLs, IPv6 filtering uses a separate IPv6 ACL and `ipv6 traffic-filter ACL_NAME in` (where supported). Do not assume either an IPv4 PACL or a MAC PACL provides IPv6 enforcement.

---

## PACL, VACL, and RACL Interaction

A packet may encounter multiple security features as it traverses a Layer 3 switch. However, **do not assume every configured ACL is evaluated sequentially on every packet**.

For Catalyst platforms, Cisco documents the following *general precedence*:

| Direction | Precedence, highest to lowest |
|---|---|
| Ingress | PACL → VACL → RACL |
| Egress | RACL → VACL → PACL (where egress PACL is supported) |

**Precedence is not the same as a guaranteed processing chain.** On some Catalyst forwarding paths, the presence of a PACL means it **takes precedence over**, rather than adds to, the VACL or ingress RACL. Cisco's Catalyst IOS XE ACL guide specifically describes inbound traffic on PACL-enabled ports being filtered by the PACL, while traffic on other ports is filtered by the VLAN map; its examples also describe circumstances where the PACL is the **only** applicable ingress filter.

On other hardware and software families, supported ACLs can be applied at multiple stages. Their exact interaction can depend on ingress versus egress processing, Layer 2 bridging versus Layer 3 routing, and special handling such as traffic punted to the CPU.

The CCIE-relevant conclusions are:

1. A **PACL on an ingress port can change whether the VLAN's VACL or the SVI's ingress RACL is enforced for that packet**.

2. A VACL `action forward` is **not a universal bypass** of every other applicable filter.

3. An ingress SVI RACL cannot be relied on to protect **host-to-host Layer 2 traffic** within its own VLAN.

4. An **egress RACL** can still restrict routed traffic as it leaves the appropriate Layer 3 interface, even if an ingress PACL permitted it.

5. Assess the **actual forwarding path and platform** when multiple ACL types coexist.

If the desired policy is security-critical, a direct end-to-end traffic test is more meaningful than assuming a generic, universal ACL evaluation order.

---

## Verification and Troubleshooting

### Configuration Verification

| Command | Purpose |
|---|---|
| `show vlan access-map` | Display VLAN access-map entries and their actions |
| `show vlan access-map MAP_NAME` | Display a particular VLAN access-map, where supported |
| `show vlan filter` | Show which VLANs have an applied VLAN map |
| `show vlan filter vlan 100` | Show filtering information for VLAN 100, where supported |
| `show access-lists` | Display IPv4 and MAC ACL definitions |
| `show ip access-lists ACL_NAME` | Display a particular IPv4 ACL |
| `show ipv6 access-list ACL_NAME` | Display a particular IPv6 ACL |
| `show mac access-group` | Show MAC ACL attachments, where supported |
| `show running-config interface GigabitEthernet1/0/1` | Verify the IP and MAC PACL attachments on a port |
| `show running-config | section vlan access-map` | Inspect the VACL configuration |
| `show interfaces trunk` | Verify which VLANs traverse trunk ports |

For a VACL, check **both** the ACL and map definition **and** the `vlan filter` attachment. For a PACL, check the definition and the actual Layer 2 interface attachment.

### Hardware ACL Verification on Catalyst 9000

Catalyst 9000 platforms program many ACL rules in hardware. A normal `show ip access-lists` hit count **may not reflect hardware-switched packets**.

On supported Catalyst 9000 platforms, useful commands include:

```text
SW1# show platform software fed active acl counters hardware
SW1# show platform software fed active acl counters hardware | include PACL Drop
SW1# show platform software fed active acl counters hardware | include VACL Drop
SW1# show platform software fed active acl counters hardware | include RACL Drop
SW1# show platform software fed active acl info db summary
```

The hardware counters can indicate that traffic is being dropped by a PACL, VACL, or RACL, but **aggregate counters do not necessarily identify the particular ACE responsible**.

Detailed `show platform ...` syntax and supported ASIC statistics vary between switch models and software releases.

### Common Troubleshooting Errors

| Symptom | Likely issue |
|---|---|
| Applying a VACL appears to do nothing | `vlan filter` was not configured, the wrong VLAN was selected, or the traffic does not traverse that switch |
| All IPv4 traffic in the VLAN stops | A map contains an IPv4 match with no appropriate allow sequence; unmatched IPv4 traffic is implicitly dropped |
| All IPv6 traffic in the VLAN stops | A map contains an IPv6 match but no appropriate forwarding entry; unmatched IPv6 traffic is implicitly dropped |
| IPv4 filtering works but IPv6 bypasses the VACL policy | Only `match ip address` exists; add `match ipv6 address` where supported |
| `deny` ACE in a VACL's referenced ACL does not block traffic | `deny` means the VACL entry **does not match**; use `permit` to match followed by `action drop` |
| IPv4 works but ARP does not | MAC PACL or VACL is blocking ARP, or another Layer 2 security feature is involved |
| ARP works but IPv4 connections fail | An IP PACL or VACL may be dropping the IPv4 packets |
| VACL blocks too much traffic | Early catch-all entry, incorrect source/destination match, or missing final forwarding sequence |
| PACL unexpectedly filters multiple VLANs | It is applied to a trunk and affects traffic from all VLANs using that port |
| SVI ACL does not block two same-VLAN hosts | Traffic is bridged and does not traverse the SVI's routed path |
| The intended VACL or RACL seems bypassed | An ingress PACL is taking precedence under the platform's ACL interaction rules |
| ACL counters remain at zero | Forwarding may be hardware switched; inspect platform hardware statistics |
| ACL works inconsistently after an edit | Hardware programming or resource limitations, such as TCAM exhaustion |

### TCAM and Hardware Resource Limitations

Switches commonly implement PACLs, VACLs, and RACLs using **ternary content-addressable memory (TCAM)** or related hardware lookup resources.

- Different ACL types and directions may consume distinct hardware regions or policy labels.

- Wide IPv6 matches, complex Layer 4 conditions, or ACL expansion can consume additional entries.

- An ACL accepted in the running configuration is not necessarily proof that every entry was successfully programmed into hardware.

- Cisco documents **out-of-resource** conditions where ACL programming fails and the resulting enforcement behavior can become highly disruptive.

For hardware-related failures, check platform ACL programming status, resource utilization, and system logs in addition to the ordinary ACL configuration.

---

## Key Points

1. **VACL = VLAN-wide policy.** It is especially useful for filtering **bridged traffic within a VLAN**.
2. **PACL = Layer 2 port policy.** An ingress PACL filters packets received on the specified port, including a trunk carrying multiple VLANs.
3. **RACL = Layer 3 interface policy.** An SVI RACL does not normally inspect traffic bridged entirely within the same VLAN.
4. **A VACL's ACL `permit` means match; its `action forward/drop` determines the result.** ACL `deny` means the entry does not match.
5. **VACL implicit behavior is protocol-category dependent.** IPv4, IPv6, and non-IP are separate categories on platforms supporting IPv6 VACLs. Unmatched packets are implicitly dropped if the map contains a match clause for their category; otherwise, they are forwarded.
6. **An entry without a `match` statement matches all packet types.** An explicit final `action forward` is useful for selective blocking; an omitted `action` also defaults to `forward`.
7. **IPv4, IPv6, and MAC ACLs handle distinct packet categories.** ARP is a particularly important non-IP example.
8. **PACL precedence can affect VACL and RACL enforcement.** Do not assume all ACLs always run cumulatively.
9. **Check hardware counters and forwarding behavior**, not only the running configuration, when troubleshooting switch ACLs.

---

## Cisco References

- [ACLs Configuration Guide: Access Control Lists (Catalyst 9000, IOS XE 17)](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/security/acls/acls-configuration-guide/access-control-lists.html)

- [Catalyst 9300 IOS XE 17.17 Command Reference: `match` (VLAN access-map configuration)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-17/command_reference/b_1717_9300_cr/security_commands.html)

- [Catalyst 9300 IOS XE 17.18 Security Configuration Guide: IPv6 ACLs](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-18/configuration_guide/sec/b_1718_sec_9300_cg/configuring_ipv6_acls.html)

- [Validate Security ACLs on Catalyst 9000 Switches](https://www.cisco.com/c/en/us/support/docs/switches/catalyst-9500-series-switches/217266-validate-security-acls-on-catalyst-9000.html)

- [VLAN Access Control Lists (Catalyst 2960-XR)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst2960xr/software/15-2_5_e/configuration_guide/b_1525e_consolidated_2960xr_cg/b_1525e_consolidated_2960xr_cg_chapter_01000011.pdf)
