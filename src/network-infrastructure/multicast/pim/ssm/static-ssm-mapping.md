# Static and DNS-Based SSM Mapping

**SSM Mapping** allows the last-hop router (LHR) to translate group-only IGMP or MLD membership reports into source-specific `(S,G)` forwarding state.

Cisco IOS XE supports two methods for determining the source addresses associated with a multicast group:

- **Static SSM Mapping:** Uses locally configured ACLs to associate multicast groups with source addresses.
- **DNS-Based SSM Mapping:** Uses DNS to obtain the source addresses associated with multicast groups.

Both methods are supported for IPv4 and IPv6.

| Feature | IPv4 | IPv6 |
|---|---|---|
| Legacy receiver protocol | IGMPv1/IGMPv2 | MLDv1 |
| Static mapping command | `ip igmp ssm-map static` | `ipv6 mld ssm-map static` |
| DNS mapping control | `ip igmp ssm-map query dns` | `ipv6 mld ssm-map query dns` |
| Mapping verification | `show ip igmp ssm-mapping` | `show ipv6 mld ssm-map` |
| VRF-aware mapping | `ip igmp vrf <vrf> ssm-map ...` | `ipv6 mld vrf <vrf> ssm-map ...` |

SSM Mapping operates on the LHR. Intermediate PIM routers do not need to configure or learn the group-to-source mappings.

## Static SSM Mapping

Static SSM Mapping associates multicast groups with source addresses using locally configured ACLs.

The ACL identifies the **multicast group or group range**, while the SSM Mapping command specifies the **source address**.

### IPv4 Static SSM Mapping

The global configuration command is:

```text
ip igmp ssm-map static <acl> <source-address>
```

Suppose we want to configure:

```text
Group:   232.1.1.1
Source:  10.1.1.10
```

First, enable IPv4 multicast routing, PIM-SSM, and SSM Mapping:

```text
R1(config)# ip multicast-routing
R1(config)# ip pim ssm default
R1(config)# ip igmp ssm-map enable
```

Disable DNS-based SSM Mapping if only static mappings will be used:

```text
R1(config)# no ip igmp ssm-map query dns
```

Create a standard ACL matching the multicast group:

```text
R1(config)# ip access-list standard SSM-GROUPS
R1(config-std-nacl)# permit host 232.1.1.1
```

Associate the permitted group with source `10.1.1.10`:

```text
R1(config)# ip igmp ssm-map static SSM-GROUPS 10.1.1.10
```

The resulting mapping is:

```text
232.1.1.1 → 10.1.1.10
```

When R1 receives an IGMPv1 or IGMPv2 Report for `232.1.1.1`, it can translate the membership into:

```text
(10.1.1.10, 232.1.1.1)
```

It then establishes source-specific PIM forwarding state toward `10.1.1.10`.

### Multiple Sources

Multiple `ip igmp ssm-map static` commands can map the same multicast group to different sources.

For example:

```text
R1(config)# ip igmp ssm-map static SSM-GROUPS 10.1.1.10
R1(config)# ip igmp ssm-map static SSM-GROUPS 10.2.2.20
```

The mapping becomes:

```text
232.1.1.1 → 10.1.1.10
            10.2.2.20
```

When a legacy receiver joins `232.1.1.1`, R1 establishes interest in both:

```text
(10.1.1.10, 232.1.1.1)
(10.2.2.20, 232.1.1.1)
```

**All matching static mappings contribute source addresses.** The router does not select only one of the matching entries.

IOS XE supports up to **20 mapped sources per IPv4 group**.

### Mapping Group Ranges

A single ACL can match multiple multicast groups.

For example:

```text
R1(config)# ip access-list standard SSM-GROUPS
R1(config-std-nacl)# permit 232.10.10.0 0.0.0.255
R1(config)# ip igmp ssm-map static SSM-GROUPS 10.1.1.10
```

Every permitted group is associated with source `10.1.1.10`.

For example:

```text
232.10.10.1 → 10.1.1.10
232.10.10.2 → 10.1.1.10
232.10.10.3 → 10.1.1.10
```

The ACL is used for **group-to-source mapping**, not to filter multicast data packets.

The mapped groups must also belong to the configured IPv4 SSM range.

## IPv6 Static SSM Mapping

IPv6 uses the corresponding global command:

```text
ipv6 mld ssm-map static <acl> <source-address>
```

IPv6 uses an **IPv6 ACL**, rather than an IPv4 standard ACL.

Suppose we want:

```text
Group:   ff35::fd00:101
Source:  2001:db8:10::10
```

Enable IPv6 multicast routing and SSM Mapping:

```text
R1(config)# ipv6 multicast-routing
R1(config)# ipv6 mld ssm-map enable
```

Disable DNS lookup if only static mapping is required:

```text
R1(config)# no ipv6 mld ssm-map query dns
```

Create an IPv6 ACL matching the multicast group:

```text
R1(config)# ipv6 access-list SSM-GROUPS-V6
R1(config-ipv6-acl)# permit ipv6 any host ff35::fd00:101
```

Configure the mapping:

```text
R1(config)# ipv6 mld ssm-map static SSM-GROUPS-V6 2001:db8:10::10
```

The resulting mapping is:

```text
ff35::fd00:101 → 2001:db8:10::10
```

When R1 receives an MLDv1 Report for `ff35::fd00:101`, it can establish:

```text
(2001:db8:10::10, ff35::fd00:101)
```

### IPv6 ACL Matching

The IPv6 ACL uses the destination field to match the multicast group.

```text
permit ipv6 any host ff35::fd00:101
            |             |
            |             +-- Multicast group
            +---------------- Any IPv6 source
```

The ACL's source field is not the multicast source address that will be used for SSM forwarding.

That address is specified separately by:

```text
ipv6 mld ssm-map static <acl> <source-address>
```

### Multiple IPv6 Sources

As with IPv4, multiple static mappings can match the same group.

```text
R1(config)# ipv6 mld ssm-map static SSM-GROUPS-V6 2001:db8:10::10
R1(config)# ipv6 mld ssm-map static SSM-GROUPS-V6 2001:db8:20::20
```

The resulting source list is:

```text
ff35::fd00:101
  → 2001:db8:10::10
  → 2001:db8:20::20
```

The LHR establishes separate `(S,G)` forwarding interest toward both sources.

## DNS-Based SSM Mapping

**DNS-Based SSM Mapping** allows the LHR to obtain multicast source addresses from a DNS server rather than relying entirely on local configuration.

This provides centralized group-to-source administration for both IPv4 IGMP and IPv6 MLD receivers.

For example:

```text
       Legacy Receiver
               |
          IGMPv2/MLDv1
               |
               v
         Last-Hop Router
               |
               | DNS lookup for G
               v
           DNS Server
               |
               | Source list: S1, S2
               v
         Last-Hop Router
               |
               | PIM Join(S1,G)
               | PIM Join(S2,G)
               v
        Multicast Network
```

The router queries DNS when it needs source information for a group and uses the returned addresses to establish the corresponding `(S,G)` forwarding state.

### Enable DNS-Based SSM Mapping

DNS-based SSM Mapping is **automatically enabled** when the corresponding SSM Mapping feature is enabled.

**IPv4:**

```text
R1(config)# ip igmp ssm-map enable
```

**IPv6:**

```text
R1(config)# ipv6 mld ssm-map enable
```

The following commands can explicitly enable or reenable DNS lookup:

```text
R1(config)# ip igmp ssm-map query dns
R1(config)# ipv6 mld ssm-map query dns
```

To disable DNS lookup:

```text
R1(config)# no ip igmp ssm-map query dns
R1(config)# no ipv6 mld ssm-map query dns
```

These commands control DNS-based SSM Mapping independently for IPv4 and IPv6.

The affirmative commands normally do not appear in the running configuration because DNS lookup is enabled by default when SSM Mapping is enabled.

### Configure the Multicast DNS Domain

The global configuration command is:

```text
ip domain multicast <domain>
```

For example:

```text
R1(config)# ip domain multicast ssm.example
```

This specifies the multicast DNS domain used for SSM Mapping.

Unlike the IGMP and MLD SSM Mapping enable commands, **`ip domain multicast` is a single global command**, not separate IPv4 and IPv6 commands.

The command is optional and is unnecessary when only static SSM Mapping is used.

Cisco explicitly documents its use for IPv4 DNS-based SSM Mapping. Although IPv6 MLD SSM Mapping also supports DNS queries, Cisco's IPv6 configuration documentation does not specify the exact DNS name-construction procedure.

### Configure DNS Servers

The DNS server is configured using the global `ip name-server` command.

For example, an IPv4 DNS server:

```text
R1(config)# ip name-server 192.0.2.53
```

Or an IPv6 DNS server:

```text
R1(config)# ip name-server 2001:db8:53::53
```

The `ip name-server` command supports both IPv4 and IPv6 DNS server addresses.

> **Note:** The IP version used to reach the DNS server does not necessarily have to match the IP version of the multicast source addresses being requested. DNS transport and DNS resource record types are separate considerations.

The DNS server must contain the appropriate group-to-source records.

## IPv4 DNS Record Construction

For IPv4 DNS-based SSM Mapping, IOS XE constructs a DNS name by reversing the four octets of the multicast group address and appending the configured multicast DNS domain.

For example:

```text
Multicast Group:
232.1.1.1
Reversed Octets:
1.1.1.232
Configured Multicast Domain:
ssm.example
DNS Name:
1.1.1.232.ssm.example
```

The router looks up **DNS A records** for the constructed name.

> Although Cisco documentation describes this as a reverse DNS lookup, the returned source addresses come from A records, not PTR records.

### DNS Record Example

Suppose we want:

```text
232.1.1.1 → 10.1.1.10
            10.2.2.20
```

An example DNS zone could contain:

```dns
1.1.1.232.ssm.example.  300  IN  A  10.1.1.10
1.1.1.232.ssm.example.  300  IN  A  10.2.2.20
```

Here:

- `1.1.1.232.ssm.example.` identifies multicast group `232.1.1.1`.
- The A records specify the multicast source addresses.
- `300` specifies the DNS TTL in seconds.

When the LHR receives a legacy IGMP Report for `232.1.1.1`, it can retrieve both source addresses and establish:

```text
(10.1.1.10, 232.1.1.1)
(10.2.2.20, 232.1.1.1)
```

No new router configuration is required when another group-to-source mapping is added to the DNS database.

### DNS Caching and TTL

DNS-based mappings are cached by the router.

The DNS record's TTL controls how long the mapping can remain cached before another lookup is required.

For example:

```text
DNS TTL: 300 seconds
```

The router may continue using the cached source list until its cache entry expires.

A shorter TTL allows DNS mapping changes to propagate more quickly, but increases DNS query traffic.

A longer TTL reduces DNS traffic but can delay the use of updated source addresses.

**Changing a DNS record does not necessarily cause an immediate change in PIM forwarding state.**

After the router refreshes a mapping and detects a source-list change, the corresponding `(S,G)` forwarding state can be updated.

## IPv6 DNS-Based SSM Mapping

IPv6 MLD SSM Mapping also supports DNS-based source discovery.

After enabling IPv6 SSM Mapping and configuring a reachable DNS server, the LHR can use DNS to obtain IPv6 multicast source addresses for a requested group.

For example:

```text
MLDv1 Report:
ff35::fd00:101
DNS Mapping Result:
2001:db8:10::10
2001:db8:20::20
```

The LHR can establish:

```text
(2001:db8:10::10, ff35::fd00:101)
(2001:db8:20::20, ff35::fd00:101)
```

IPv6 SSM Mapping supports both static and DNS-based results.

Verify the returned source list using:

```text
R1# show ipv6 mld ssm-map
```

Or query a particular group:

```text
R1# show ipv6 mld ssm-map ff35::fd00:101
```

> **Note:** Cisco's IPv6 SSM Mapping documentation confirms DNS support but does not detail the IPv6 DNS naming and record-construction procedure. Do not assume IPv4's reversed-octet naming format applies to IPv6.

## Static vs. DNS Mapping Precedence

When both static and DNS-based mappings are configured, their interaction is important.

For **IPv4 IOS XE**, a matching static SSM Mapping takes precedence over a DNS-based mapping.

Suppose the DNS server contains:

```text
232.1.1.1 → 10.1.1.10
```

But R1 has a matching static mapping:

```text
R1(config)# ip igmp ssm-map static SSM-GROUPS 10.2.2.20
```

The locally configured static mapping is used:

```text
232.1.1.1 → 10.2.2.20
```

The DNS and static source lists are not combined for that group.

If multiple static entries match, however, their source addresses are combined.

This distinction is important:

| Situation | IPv4 Mapping Result |
|---|---|
| One matching static entry | Use that static source |
| Multiple matching static entries | Use all matching static sources |
| No matching static entry, DNS enabled | Use DNS mapping |
| No matching static entry, DNS disabled | No source mapping |
| Matching static and DNS entries | Static mapping takes precedence |

Cisco documents IPv6 static and DNS mapping support, including aggregation of multiple matching static entries. Verify static-versus-DNS precedence on the target IPv6 IOS XE platform if both methods are configured simultaneously.

## Static Forwarding Using SSM Mapping

IPv4 IOS XE also supports creating static downstream multicast forwarding interest while obtaining the source addresses from SSM Mapping.

The interface command is:

```text
ip igmp static-group <group> source ssm-map
```

For example:

```text
R1(config)# interface Ethernet0/1
R1(config-if)# ip igmp static-group 232.1.1.1 source ssm-map
```

This instructs the router to determine the sources using the SSM Mapping database rather than requiring a dynamic IGMP Report.

Compare:

```text
ip igmp static-group 232.1.1.1 source 10.1.1.10
```

with:

```text
ip igmp static-group 232.1.1.1 source ssm-map
```

The first command specifies the source directly.

The second obtains the source addresses from SSM Mapping, which may use static configuration or DNS.

For IPv6, `ipv6 mld static-group` supports source-specific forwarding interest with explicitly specified IPv6 source addresses:

```text
R1(config-if)# ipv6 mld static-group ff35::fd00:101 include 2001:db8:10::10
```

Do not confuse this with an MLD SSM Mapping database entry.

## IPv4 VRF-Aware SSM Mapping

IPv4 SSM Mapping also supports **VRF-specific configuration**. Each VRF can maintain its own group-to-source mappings, independent of the global routing table and other VRFs.

The IPv4 IOS XE syntax is:

```text
ip igmp vrf <vrf-name> ssm-map enable
ip igmp vrf <vrf-name> ssm-map query dns
ip igmp vrf <vrf-name> ssm-map static <acl> <source-address>
```

### IPv4 VRF Static Mapping Example

Assume VRF `A` already exists and includes the receiver-facing interface. Enable multicast routing and configure its SSM range:

```text
R1(config)# ip multicast-routing vrf A
R1(config)# ip pim vrf A ssm default
```

Create a standard IPv4 ACL identifying the multicast groups:

```text
R1(config)# ip access-list standard SSM-GROUPS-A
R1(config-std-nacl)# permit host 232.1.1.1
```

Enable SSM Mapping in VRF `A` and define the source:

```text
R1(config)# ip igmp vrf A ssm-map enable
R1(config)# no ip igmp vrf A ssm-map query dns
R1(config)# ip igmp vrf A ssm-map static SSM-GROUPS-A 10.1.1.10
```

This creates the VRF-specific mapping:

```text
VRF A:
232.1.1.1 → 10.1.1.10
```

The mapping in VRF `A` does not automatically apply to the global routing table or another VRF. PIM must also be operating on the relevant VRF interfaces, and the source must be reachable through the appropriate multicast RPF path.

> **Syntax note:** Cisco command references also document `ip igmp ssm-map vrf <vrf-name> static <acl> <source-address>` on some releases. The `ip igmp vrf A ssm-map static ...` ordering shown above matches the VRF-aware command hierarchy confirmed in IOS XE CLI help. Use `?` to verify the exact syntax on the target platform.

### IPv4 VRF DNS Mapping

To use DNS-based SSM Mapping in VRF `A`, enable DNS queries and configure a VRF-aware DNS server:

```text
R1(config)# ip igmp vrf A ssm-map query dns
R1(config)# ip name-server vrf A 192.0.2.53
```

The DNS server must be reachable in VRF `A` and contain the appropriate group-to-source records. The multicast DNS domain is set using the global `ip domain multicast` command discussed earlier.

### IPv4 VRF Verification

```text
R1# show ip igmp vrf A ssm-mapping
R1# show ip igmp vrf A ssm-mapping 232.1.1.1
R1# show ip mroute vrf A
```

The first two commands display SSM Mapping status and the selected sources **in VRF `A`**, rather than the global table. The third checks whether the resulting `(S,G)` forwarding state is present in that VRF.

## IPv6 VRF-Aware SSM Mapping

IPv6 IOS XE supports VRF-specific SSM Mapping configuration.

The command syntax includes:

```text
ipv6 mld vrf <vrf-name> ssm-map enable
ipv6 mld vrf <vrf-name> ssm-map static <acl> <source>
ipv6 mld vrf <vrf-name> ssm-map query dns
```

For example, enable SSM Mapping within VRF `Blue`:

```text
R1(config)# ipv6 multicast-routing vrf Blue
R1(config)# ipv6 mld vrf Blue ssm-map enable
R1(config)# ipv6 mld vrf Blue ssm-map static SSM-GROUPS-V6 2001:db8:10::10
```

Disable DNS lookups for that VRF if only static mappings are required:

```text
R1(config)# no ipv6 mld vrf Blue ssm-map query dns
```

To use DNS mappings, reenable DNS lookup and configure a DNS server reachable in VRF `Blue`:

```text
R1(config)# ipv6 mld vrf Blue ssm-map query dns
R1(config)# ip name-server vrf Blue 2001:db8:53::53
```

Verify the VRF-specific mapping:

```text
R1# show ipv6 mld vrf Blue ssm-map
R1# show ipv6 mld vrf Blue ssm-map ff35::fd00:101
```

This allows multicast source mappings to be maintained independently for IPv6 multicast environments using VRFs.

## Verification

### IPv4

Useful commands:

```text
show ip igmp ssm-mapping
show ip igmp ssm-mapping <group>
show ip igmp groups detail
show ip mroute
show hosts
debug ip igmp <group>
```

#### Verify SSM Mapping Status

```text
R1# show ip igmp ssm-mapping
```

Example output:

```text
SSM Mapping  : Enabled
DNS Lookup   : Enabled
Mcast domain : ssm.example
Name servers : 192.0.2.53
```

This confirms that SSM Mapping and DNS lookup are enabled.

#### Verify a Group-to-Source Mapping

```text
R1# show ip igmp ssm-mapping 232.1.1.1
```

Example DNS-based result:

```text
Group address: 232.1.1.1
Database     : DNS
DNS name     : 1.1.1.232.ssm.example
Expire time  : 280000
Source list  : 10.1.1.10
             : 10.2.2.20
```

Important fields:

| Field | Meaning |
|---|---|
| Group address | Multicast group being mapped |
| Database | Source of the mapping, such as STATIC or DNS |
| DNS name | Constructed DNS name used for lookup |
| Expire time | Remaining cached mapping lifetime in milliseconds |
| Source list | Sources associated with the group |

`show ip igmp groups detail` can also identify mapped sources using the `M` (SSM Mapping) flag.

#### Debug SSM Mapping

Use:

```text
R1# debug ip igmp 232.1.1.1
```

Cisco documents debug messages identifying whether a legacy IGMP Report was converted using STATIC or DNS mappings.

For example:

```text
Convert IGMPv2 report (*,232.1.1.1)
to IGMPv3 with 2 source(s) using STATIC
```

This confirms that a group-only membership was translated into source-specific state.

Use debug commands cautiously on production routers.

### IPv6

Useful commands:

```text
show ipv6 mld ssm-map
show ipv6 mld ssm-map <group>
show ipv6 mld groups detail
show ipv6 pim group-map
show ipv6 mroute
debug ipv6 mld ssm-map
```

Verify the mapping for a specific IPv6 group:

```text
R1# show ipv6 mld ssm-map ff35::fd00:101
```

A successful static mapping displays information such as:

```text
Group address  : FF35::FD00:101
Group mode ssm : TRUE
Database       : STATIC
Source list    : 2001:DB8:10::10
                 2001:DB8:20::20
```

For a DNS-based mapping, the database field identifies DNS instead.

> The output above illustrates the expected fields and source list. Actual formatting may vary by IOS XE platform and release.

After verifying the mapping, use `show ipv6 mroute` to confirm the corresponding `(S,G)` forwarding state.

## Troubleshooting

| Symptom | Possible Cause |
|---|---|
| SSM Mapping disabled | `ssm-map enable` not configured |
| Group membership exists, but mapping is absent | Group not in SSM range or no matching mapping |
| Static source missing | ACL does not permit the group |
| DNS mapping fails | DNS server unreachable, record missing, or incorrect multicast domain |
| Old DNS source remains | Cached mapping has not expired |
| Wrong source selected | Matching static mapping overrides DNS (IPv4) |
| Mapping exists, but no traffic | PIM/RPF problem or source not transmitting |
| IGMPv3 group-only Report does not trigger mapping | SSM Mapping targets IGMPv1/v2 legacy Reports |

### IGMP Version Compatibility

A particularly important IOS XE limitation is that **SSM Mapping does not translate IGMPv3 group-only Reports in the same way as IGMPv1/IGMPv2 Reports**.

For example, an IGMPv3 host may request a group using:

```text
EXCLUDE {}
```

This expresses group-only interest but does not provide a source address.

If an LHR relies on SSM Mapping for legacy receivers, it should not assume that every IGMPv3 group-only Report will trigger the same mapping process.

Cisco specifically warns about enabling IGMPv3 in environments that rely exclusively on SSM Mapping for receiver compatibility.

## Command Summary

| Purpose | IPv4 | IPv6 |
|---|---|---|
| Enable mapping | `ip igmp ssm-map enable` | `ipv6 mld ssm-map enable` |
| Configure static source | `ip igmp ssm-map static <acl> <source>` | `ipv6 mld ssm-map static <acl> <source>` |
| Enable DNS lookup | `ip igmp ssm-map query dns` | `ipv6 mld ssm-map query dns` |
| Disable DNS lookup | `no ip igmp ssm-map query dns` | `no ipv6 mld ssm-map query dns` |
| Display mapping | `show ip igmp ssm-mapping` | `show ipv6 mld ssm-map` |
| Display group mapping | `show ip igmp ssm-mapping <group>` | `show ipv6 mld ssm-map <group>` |
| Enable mapping in a VRF | `ip igmp vrf <vrf> ssm-map enable` | `ipv6 mld vrf <vrf> ssm-map enable` |
| Static mapping in a VRF | `ip igmp vrf <vrf> ssm-map static <acl> <source>` | `ipv6 mld vrf <vrf> ssm-map static <acl> <source>` |
| DNS mapping in a VRF | `ip igmp vrf <vrf> ssm-map query dns` | `ipv6 mld vrf <vrf> ssm-map query dns` |
| Display VRF mapping | `show ip igmp vrf <vrf> ssm-mapping` | `show ipv6 mld vrf <vrf> ssm-map` |

Shared multicast DNS domain configuration:

```text
ip domain multicast <domain>
```

IPv4 static downstream forwarding using SSM Mapping:

```text
ip igmp static-group <group> source ssm-map
```

DNS server configuration for both address families, optionally scoped to a VRF:

```text
ip name-server <dns-server-address>
ip name-server vrf <vrf-name> <dns-server-address>
```

Both IPv4 and IPv6 SSM Mapping support VRF-specific `enable`, `static`, and `query dns` commands.

## Key Points

- **Static SSM Mapping** uses ACLs to associate multicast group addresses with source addresses.
- **DNS-Based SSM Mapping** retrieves source addresses from DNS.
- Both methods are supported for IPv4 IGMP and IPv6 MLD.
- Enabling SSM Mapping automatically enables DNS-based mapping by default.
- Multiple matching static mappings contribute multiple source addresses.
- IPv4 SSM Mapping supports up to **20 sources per group**.
- For IPv4, matching static mappings take precedence over DNS mappings.
- IPv4 DNS lookup uses a reversed group address and **DNS A records**, not PTR records.
- DNS mappings are cached according to DNS TTLs, so changes may not take effect immediately.
- IPv4 supports `ip igmp static-group <group> source ssm-map` to establish static forwarding interest using dynamically determined sources.
- **Both IPv4 and IPv6 SSM Mapping support VRF-specific configuration**, including independent enable, static mapping, and DNS lookup controls.
- VRF-aware DNS resolution can use `ip name-server vrf <vrf-name> <dns-server-address>`.
- SSM Mapping does not replace PIM forwarding. The LHR still establishes `(S,G)` state toward the mapped sources.
