# Command Summary

Configuration and verification commands for **Static RP**, **BSR**, and **Auto-RP** on Cisco IOS XE.

**Syntax:** `<argument>` represents a required value; `[option]` is optional; `{A | B}` indicates alternatives. Substitute actual values when configuring the router. Commands are shown without CLI prompts.

**Common prerequisites:** Enable multicast routing with `ip multicast-routing` (IPv4) or `ipv6 multicast-routing` (IPv6). Configure PIM on the relevant IPv4 routed interfaces with `ip pim sparse-mode`; ensure IPv6 PIM and unicast reachability are operational for IPv6. RP addresses must be reachable through unicast routing.

## Static RP

Static RP mappings are configured **locally on each router** that needs them. No RP discovery messages are exchanged.

### Configuration

| Command syntax | Purpose |
|---|---|
| `ip pim rp-address <rp-ipv4-address> [<standard-acl>] [override]` | Configures an IPv4 RP. An optional standard ACL selects multicast groups; `override` gives the mapping precedence over dynamically learned mappings. |
| `ipv6 pim rp-address <rp-ipv6-address> [<ipv6-acl>] [bidir]` | Configures an IPv6 RP. An optional IPv6 ACL selects multicast groups; `bidir` specifies Bidirectional PIM. |

**Examples:**

- `ip pim rp-address 3.3.3.3`: Map the applicable IPv4 ASM groups to RP `3.3.3.3`.
- `ip pim rp-address 3.3.3.3 RP-GROUPS`: Apply the mapping only to groups permitted by the standard ACL `RP-GROUPS`.
- `ip pim rp-address 3.3.3.3 RP-GROUPS override`: Prefer this static IPv4 mapping over dynamic RP mappings.
- `ipv6 pim rp-address 2001:DB8:3::3 RP-GROUPS-V6`: Configure a group-specific IPv6 RP mapping.

### ACLs for Group-Specific Mappings

| Command syntax | Purpose |
|---|---|
| `ip access-list standard <acl-name>` | Creates a standard IPv4 ACL used to identify multicast group addresses. |
| `permit <group-ipv4-address> <wildcard-mask>` | Permits a group or range in the standard ACL, for example `permit 239.1.0.0 0.0.255.255`. |
| `ipv6 access-list <acl-name>` | Creates an IPv6 ACL for IPv6 group matching. |
| `permit ipv6 any <group-ipv6-prefix>` | Permits an IPv6 multicast group range using the destination field, for example `permit ipv6 any FF15::/16`. |

The ACLs used with **Static RP** identify **multicast groups**, not source addresses.

### Verification

| Command syntax | Purpose |
|---|---|
| `show ip pim rp mapping` | Displays known IPv4 RP mappings and how they were learned. |
| `show ip pim rp <group-ipv4-address>` | Checks the IPv4 RP for a multicast group. |
| `show ipv6 pim interface` | Verifies IPv6 PIM interface operation. |
| `show ipv6 pim group-map` | Displays IPv6 group-to-RP mappings. |
| `show ipv6 pim group-map info-source static` | Shows IPv6 mappings learned from static configuration. |
| `show ipv6 pim group-map <group-ipv6-address>` | Checks the RP for an IPv6 multicast group. |
| `show ip rpf <rp-ipv4-address>` | Verifies IPv4 RPF reachability toward the RP. |
| `show ipv6 rpf <rp-ipv6-address>` | Verifies IPv6 RPF reachability toward the RP. |
| `show ip mroute <group-ipv4-address>` | Displays IPv4 multicast forwarding state. |
| `show ipv6 mroute <group-ipv6-address>` | Displays IPv6 multicast forwarding state. |

See [Static RP](static-rp.md).

## BSR

**Candidate BSRs (C-BSRs)** participate in BSR election; **Candidate RPs (C-RPs)** advertise to the elected BSR. The BSR distributes an RP-Set in Bootstrap Messages, and **each router selects its RP** from that information.

### Candidate BSR

| Command syntax | Purpose |
|---|---|
| `ip pim bsr-candidate <interface> [<hash-mask-length> [<priority> [accept-rp-candidate <acl>]]]` | Configures an IPv4 C-BSR using the interface address. Optional arguments specify the RP hash mask, BSR election priority, and accepted Candidate RP filter. |
| `ipv6 pim bsr candidate bsr <ipv6-address> [<hash-mask-length>] [priority <priority>] [scope] [accept-rp-candidate <ipv6-acl>]` | Configures an IPv6 C-BSR, with optional hash mask, priority, scoped BSR operation, and C-RP filtering. |

**Examples:**

- `ip pim bsr-candidate Loopback0 30 10`: IPv4 C-BSR, hash mask 30, priority 10.
- `ip pim bsr-candidate Loopback0 30 10 accept-rp-candidate ALLOWED-C-RPS`: Accept only C-RPs permitted by the specified ACL.
- `ipv6 pim bsr candidate bsr 2001:DB8:1::1 126 priority 10`: IPv6 C-BSR, hash mask 126, priority 10.
- `ipv6 pim bsr candidate bsr 2001:DB8:1::1 126 priority 10 accept-rp-candidate ALLOWED-C-RPS-V6`: Apply an IPv6 Candidate RP filter.

**BSR election:** Highest numeric priority wins; highest BSR address breaks priority ties. The hash mask affects **RP hashing**, not BSR election.

### Candidate RP

| Command syntax | Purpose |
|---|---|
| `ip pim rp-candidate <interface> [group-list <standard-acl>] [interval <seconds>] [priority <priority>]` | Advertises the interface address as an IPv4 C-RP to the elected BSR. Optionally restricts group ranges, advertisement interval, and RP priority. |
| `ipv6 pim bsr candidate rp <ipv6-address> [group-list <ipv6-acl>] [priority <priority>] [interval <seconds>] [scope <scope-value>] [bidir]` | Advertises an IPv6 C-RP; optional parameters control group ranges, priority, interval, IPv6 multicast scope, and Bidirectional PIM. |

**Examples:**

- `ip pim rp-candidate Loopback0 group-list RP-GROUPS interval 30 priority 10`.
- `ipv6 pim bsr candidate rp 2001:DB8:3::3 group-list RP-GROUPS-V6 priority 10 interval 30`.
- `ipv6 pim bsr candidate rp 2001:DB8:3::3 scope 5`: Advertise C-RP candidacy to the BSR for **site-local multicast scope (5)**.
- `ipv6 pim bsr candidate rp 2001:DB8:3::3 group-list RP-GROUPS-V6 bidir`: Advertise Bidirectional PIM RP mappings.

**RP preference:** Lowest numeric RP priority wins. Equal-priority RPs are resolved using BSR hashing, with highest RP address as the final hash tie-breaker.

### BSR-Announced RP (IPv6)

| Command syntax | Purpose |
|---|---|
| `ipv6 pim bsr announced rp <rp-ipv6-address> [group-list <ipv6-acl>] [priority <priority>] [bidir] [scope <scope-value>]` | Configures an IPv6 RP mapping **on the BSR** for distribution through BSMs, without requiring a C-RP advertisement from that RP. |

**Example:** `ipv6 pim bsr announced rp 2001:DB8:3::3 group-list RP-GROUPS-V6 priority 10`.

Unlike `ipv6 pim rp-address`, this command distributes the mapping throughout the BSR domain instead of configuring it only locally.

### BSR Filtering and Borders

| Command syntax | Purpose |
|---|---|
| `ip pim bsr-candidate <interface> <hash-mask-length> <priority> accept-rp-candidate <acl>` | Uses a standard or extended IPv4 ACL to restrict the C-RPs accepted by the elected BSR. |
| `ipv6 pim bsr candidate bsr <ipv6-address> <hash-mask-length> priority <priority> accept-rp-candidate <ipv6-acl>` | Uses an IPv6 ACL to restrict accepted C-RPs. |
| `ip pim bsr-border` | **Interface configuration:** Blocks incoming and outgoing IPv4 BSMs. |
| `ipv6 pim bsr border` | **Interface configuration:** Blocks incoming and outgoing IPv6 BSMs. |

**ACL distinction:** A C-RP `group-list` selects **groups the RP advertises**. A C-BSR `accept-rp-candidate` ACL selects **which C-RPs the BSR accepts**.

| ACL command syntax | Purpose |
|---|---|
| `ip access-list standard <acl-name>` followed by `permit host <candidate-rp-ipv4>` | Standard IPv4 C-RP address allowlist. |
| `ip access-list extended <acl-name>` followed by `permit pim host <candidate-rp-ipv4> any` | Extended IPv4 PIM C-RP filtering. |
| `ipv6 access-list <acl-name>` followed by `permit ipv6 host <candidate-rp-ipv6> any` | IPv6 C-RP filtering. |

### BSR Defaults and Timers

| Parameter | IPv4 | IPv6 |
|---|---|---|
| Default BSR priority | 0 | 0 |
| Default BSR hash mask length | 0 bits | 126 bits |
| Recommended hash mask length (RFC 5059) | 30 bits | 126 bits |
| Default C-RP priority | 0 | 192 |
| Default C-RP advertisement interval | 60 seconds | 60 seconds |
| Bootstrap Message interval | 60 seconds | 60 seconds |
| Bootstrap timeout | 130 seconds | 130 seconds |

### Verification

| Command syntax | Purpose |
|---|---|
| `show running-config \| include ip pim bsr-candidate` | Checks the configured IPv4 C-BSR command. |
| `show running-config \| include ip pim rp-candidate` | Checks the configured IPv4 C-RP command. |
| `show running-config \| include ipv6 pim bsr candidate bsr` | Checks the configured IPv6 C-BSR command. |
| `show ip pim bsr-router` | Displays the IPv4 elected BSR, priority, and hash mask. |
| `show ip pim rp mapping` | Displays IPv4 RP mappings distributed through BSR. |
| `show ip pim rp-hash <group-ipv4-address>` | Verifies the IPv4 RP selected by BSR hashing. |
| `show ipv6 pim bsr election` | Displays IPv6 BSR state, election information, and timers. |
| `show ipv6 pim bsr candidate-rp` | Displays local IPv6 C-RP advertisement state. |
| `show ipv6 pim bsr rp-cache` | Displays the IPv6 C-RP information received by the elected BSR. |
| `show ipv6 pim group-map [<group-ipv6-address>]` | Displays IPv6 group-to-RP mappings, optionally for one group. |
| `show ipv6 pim group-map info-source bsr` | Displays IPv6 mappings learned through BSR. |
| `show ip pim neighbor` | Checks IPv4 PIM neighbor relationships required for BSM forwarding. |
| `show ipv6 pim neighbor` | Checks IPv6 PIM neighbor relationships. |
| `show ip rpf <bsr-ipv4-address>` | Verifies the IPv4 RPF path toward the BSR. |
| `show ipv6 rpf <bsr-ipv6-address>` | Verifies the IPv6 RPF path toward the BSR. |
| `show access-lists <acl-name>` | Verifies an IPv4 Candidate RP or group filter ACL. |
| `show ipv6 access-list <acl-name>` | Verifies an IPv6 Candidate RP or group filter ACL. |

See [BSR](bsr/overview.md), [Candidate BSR and BSR Election](bsr/candidate-bsr.md), [Candidate RP and RP Advertisements](bsr/candidate-rp.md), [Bootstrap Messages and RP-Set](bsr/bootstrap-messages.md), and [BSR Hashing](bsr/hashing.md).

## Auto-RP

Auto-RP is **IPv4-only**. Candidate RPs multicast announcements to `224.0.1.39`; Mapping Agents distribute the **selected** RP mappings to `224.0.1.40`. Unlike BSR, the Mapping Agent performs RP selection.

### Candidate RP and Mapping Agent

| Command syntax | Purpose |
|---|---|
| `ip pim send-rp-announce <interface> scope <ttl> [group-list <standard-acl>] [interval <seconds>] [bidir]` | **Candidate RP:** Sends RP-Announce messages to `224.0.1.39`. The `scope` value is the packet TTL, not an IPv6 scope. |
| `ip pim send-rp-discovery [<interface>] scope <ttl> [interval <seconds>]` | **Mapping Agent:** Sends selected RP mappings in RP-Discovery messages to `224.0.1.40`. |

**Examples:**

- `ip pim send-rp-announce Loopback0 scope 16`: Advertise the loopback address as a Candidate RP.
- `ip pim send-rp-announce Loopback0 scope 16 group-list RP-GROUPS interval 30`: Restrict group ranges and set a 30-second RP-Announce interval.
- `ip pim send-rp-discovery Loopback0 scope 16`: Configure the router as a Mapping Agent.
- `ip pim send-rp-discovery Loopback0 scope 16 interval 30`: Send RP-Discovery messages every 30 seconds.

**Both intervals default to 60 seconds.** Each command's `scope <ttl>` controls how far its own Auto-RP messages can travel.

### Auto-RP Listener and Forwarding

| Command syntax | Purpose |
|---|---|
| `ip pim autorp listener` | **Global configuration:** Enables Dense Mode forwarding for the two Auto-RP control groups across PIM Sparse Mode interfaces. |
| `ip pim sparse-mode` | **Interface configuration:** Enables PIM Sparse Mode; use with `ip pim autorp listener` for Auto-RP bootstrap. |
| `ip pim sparse-dense-mode` | **Interface configuration:** Uses Dense Mode for groups without RP mappings; alternative to Auto-RP Listener. |

Auto-RP Listener affects `224.0.1.39` and `224.0.1.40`, not every multicast group.

### Auto-RP Filtering and Boundaries

| Command syntax | Purpose |
|---|---|
| `ip pim rp-announce-filter group-list <standard-acl>` | **Mapping Agent:** Applies the multicast group filter to **all** Candidate RPs. |
| `ip pim rp-announce-filter rp-list <standard-acl> [group-list <standard-acl>]` | **Mapping Agent:** Applies the group filter only to Candidate RPs permitted by the `rp-list`. Omitting `group-list` denies all advertised groups for those selected RPs. |
| `ip multicast boundary <standard-acl> [filter-autorp]` | **Interface configuration:** Limits multicast groups at a boundary; `filter-autorp` also filters group ranges carried in Auto-RP messages. |

**Important:** A Candidate RP denied by an Auto-RP `rp-list` is **not automatically rejected**. It is outside that filtering rule and can still be accepted. See [RP Announcement Filtering](../filtering/rp-announcement-filter.md) for multiple-rule and whitelist examples, and [Multicast Boundaries and Administrative Scope](../filtering/multicast-boundary.md) for `filter-autorp`.

### Verification

| Command syntax | Purpose |
|---|---|
| `show ip pim autorp` | Displays Auto-RP information. |
| `show ip pim rp mapping` | Shows group-to-RP mappings learned through Auto-RP. |
| `show ip pim rp <group-ipv4-address>` | Checks the RP for a group. |
| `show ip mroute 224.0.1.39` | Checks the multicast forwarding state for RP-Announce traffic. |
| `show ip mroute 224.0.1.40` | Checks the multicast forwarding state for RP-Discovery traffic. |
| `show ip pim interface` | Verifies the PIM mode on IPv4 interfaces. |
| `show ip pim neighbor` | Verifies PIM neighbor relationships. |
| `show access-lists <acl-name>` | Verifies Auto-RP group and Candidate RP filtering ACLs. |

See [Auto-RP](auto-rp/overview.md), [Candidate RP Announcements](auto-rp/rp-announcements.md), [Mapping Agent and Discovery Messages](auto-rp/mapping-agent.md), and [Auto-RP Groups and Auto-RP Listener](auto-rp/groups.md).

## References

- [Cisco IOS XE: Configuring Basic IP Multicast](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-multicast/ip-multicast/m_imc_basic_cfg-0.html)
- [Cisco IOS XE: IPv6 Multicast Bootstrap Router](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-multicast/ip-multicast/m_ip6-mcast-bsr.html)
- [Cisco IOS XE: PIM Allow RP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_imc_pim_allowrp.html)
- [Cisco: Configuring Basic IP Multicast](https://www.cisco.com/en/US/docs/ios/12_4t/ip_mcast/configuration/guide/mctbasic.html)
