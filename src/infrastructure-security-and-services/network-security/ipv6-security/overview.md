# 4.2.c IPv6 Infrastructure Security Features

**IPv6 First Hop Security (FHS)** is a collection of features that protects hosts on a local network from attacks involving IPv6 router discovery, address assignment, Neighbor Discovery (ND), and source address spoofing.

These protections are commonly implemented on the **access switch**, where end hosts connect to the network. They prevent an unauthorized device from impersonating a router or DHCPv6 server, claiming another host's IPv6 address, or sending traffic with a forged source address.

Cisco IOS XE uses **Switch Integrated Security Features (SISF)** as the framework for device tracking, host bindings, and several IPv6 first-hop security functions. This section uses the modern SISF-based approach throughout.

---

## IPv6 First Hop Security Features at a Glance

| Feature | Primary purpose | What it examines or maintains |
| --- | --- | --- |
| **Router Advertisement (RA) Guard** | Prevent unauthorized devices from advertising themselves as IPv6 routers. | ICMPv6 Router Advertisements and configured RA policy conditions. |
| **DHCPv6 Guard** | Block DHCPv6 messages from unauthorized servers or relay agents. | DHCPv6 server/relay messages and configured DHCPv6 policy conditions. |
| **SISF Binding Table** | Record which IP addresses and MAC addresses belong to hosts on particular switch ports and VLANs. | Host address bindings, their origins, reachability states, and lifetimes. |
| **SISF Device Tracking** | Learn and maintain information about connected IPv4 and IPv6 hosts. | ND, DHCPv6, ARP, DHCPv4, and configured tracking policies. |
| **ND Inspection/Snooping** | Detect and prevent illegitimate Neighbor Discovery information, including address ownership conflicts and spoofing. | ICMPv6 Neighbor Discovery messages and SISF binding validation. |
| **IPv6 Source Guard** | Prevent hosts from sending IPv6 data packets with unauthorized source addresses. | Source IPv6 addresses and the bindings used to authorize them. |

The features address different threats. **RA Guard and DHCPv6 Guard protect control-plane information received from unauthorized devices.** Device tracking and ND inspection build and validate host identity information, while **IPv6 Source Guard enforces source-address restrictions on data traffic**.

---

## Why IPv6 Requires First Hop Security

IPv6 hosts rely heavily on **ICMPv6 Neighbor Discovery Protocol (NDP)** to discover routers, resolve neighbors' link-layer addresses, and configure addresses. IPv6 hosts can also use **DHCPv6** to obtain addresses or other network parameters.

Important Neighbor Discovery messages include:

| ICMPv6 type | Message | Purpose |
| --- | --- | --- |
| **133** | Router Solicitation (RS) | A host requests router information. |
| **134** | Router Advertisement (RA) | A router advertises information such as prefixes and default-router availability. |
| **135** | Neighbor Solicitation (NS) | Used for address resolution, neighbor reachability, and Duplicate Address Detection (DAD). |
| **136** | Neighbor Advertisement (NA) | Provides neighbor information or announces a neighbor's address. |
| **137** | Redirect | A router informs a host of a better next hop. |

An attacker connected to the same Layer 2 network could attempt to:

- Send **rogue RAs** to advertise itself as a default router or advertise incorrect prefixes.
- Operate a **rogue DHCPv6 server** and distribute unauthorized addressing information.
- Send **forged Neighbor Advertisements** to associate a victim's IPv6 address with the attacker's MAC address.
- Interfere with **Duplicate Address Detection** so that legitimate hosts cannot use their addresses.
- Send IPv6 packets with **spoofed source addresses**.

An ordinary IPv6 ACL is not a complete replacement for these protections. FHS policies understand the roles of hosts, routers, and DHCPv6 servers, and SISF can validate address ownership using host bindings.

---

## Switch Integrated Security Features (SISF)

**SISF** provides a common infrastructure for discovering hosts, maintaining their address bindings, and making that information available to security features. It supports both **IPv4 and IPv6**, which makes it useful on dual-stack access networks.

SISF consists of three closely related concepts:

1. **Device tracking:** Learns and monitors hosts using supported protocol messages and, when enabled, reachability probes.
2. **Binding table:** Stores the learned relationship between a host's IP address, MAC address, VLAN, and switch interface.
3. **Security policies:** Control what SISF learns, how it validates information, and how applicable traffic is handled.

The SISF binding database can also be used by other switch functions, including identity and access-control features.

### How the Components Work Together

Consider a host connected to a switch in VLAN 10:

```text
                    Authorized IPv6 Router
                         DHCPv6 Server
                               |
                           Gi1/0/24
                               |
                         +-----------+
                         |    SW1    |
                         |    SISF   |
                         +-----------+
                               |
                           Gi1/0/1
                               |
                              PC1
                     2001:db8:10::10
                            VLAN 10
```

When PC1 starts using IPv6:

1. PC1 exchanges IPv6 control messages, such as Router Solicitations, Neighbor Solicitations, Neighbor Advertisements, and possibly DHCPv6 messages.
2. **RA Guard** can prevent an unauthorized router advertisement from entering through a host-facing port.
3. **DHCPv6 Guard** can prevent unauthorized DHCPv6 server or relay messages from entering through a host-facing port.
4. **SISF device tracking** can learn PC1's address and attachment information from supported messages.
5. **SISF binding validation** can check for inconsistent or conflicting address ownership information in Neighbor Discovery messages.
6. **IPv6 Source Guard**, if enabled, can use authorized binding information to filter ordinary IPv6 traffic sourced from PC1.

These are **complementary protections**, not a mandatory sequence through which every packet passes. RA Guard and DHCPv6 Guard can enforce their own policies; they do not need to wait for a host binding to be created.

---

## The SISF Binding Table

The **binding table** associates a host's IP address with its point of attachment to the network.

For example:

| IPv6 address | MAC address | VLAN | Interface |
| --- | --- | --- | --- |
| `2001:db8:10::10` | `0011.2233.4455` | 10 | `Gi1/0/1` |
| `2001:db8:10::20` | `0011.2233.4466` | 10 | `Gi1/0/2` |

Entries also contain information such as their **source**, **state**, and **lifetime**. SISF can learn entries dynamically from Neighbor Discovery and DHCPv6, or maintain manually configured bindings.

Unlike an IPv4 DHCP snooping database, **SISF does not depend exclusively on DHCP**. This is especially important for IPv6 because hosts may configure addresses through **Stateless Address Autoconfiguration (SLAAC)** and may use multiple IPv6 addresses on the same interface.

For example, PC1 could have a link-local address, a stable global unicast address, and a temporary privacy address. SISF can maintain multiple address bindings for the same MAC address and port.

A binding is not simply either present or absent. SISF tracks reachability through states including **REACHABLE**, **STALE**, and **DOWN**, with other states used during learning and verification.

See [IPv6 Binding Table](binding-table.md) for binding sources, entry states, lifetimes, static entries, and verification.

---

## SISF Device-Tracking Policies

SISF uses **device-tracking policies** to define how information is learned and verified. A policy can be attached to an interface or to a VLAN.

Important policy parameters include:

| Parameter | Purpose |
| --- | --- |
| `security-level` | Determines how strictly learned host information is validated. |
| `device-role` | Identifies whether the connected device is an end node or another switch. |
| `trusted-port` | Treats information received on a port as trusted for SISF binding validation. |
| `protocol` | Selects which protocols SISF uses to learn information. |
| `limit` | Restricts the number of learned addresses or bindings. |
| `tracking` | Controls active reachability tracking of learned hosts. |

### SISF Security Levels

| Security level | Behavior |
| --- | --- |
| **Glean** | Learns address information without validating its ownership. |
| **Inspect** | An intermediate validation mode. Cisco recommends using `glean` or `guard` instead. |
| **Guard** | Validates address ownership and can reject conflicting or spoofed information. |

`guard` is the default SISF device-tracking security level. **Gleaning information is not the same as proving that it is legitimate**. A security feature that relies on bindings must account for how those bindings were learned and validated.

### Basic Device-Tracking Policy Example

Create a policy for host-facing switch ports:

```text
SW1(config)# device-tracking policy IPV6_HOSTS
SW1(config-device-tracking)# security-level guard
SW1(config-device-tracking)# device-role node
SW1(config-device-tracking)# exit
!
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# device-tracking attach-policy IPV6_HOSTS
```

The policy enables SISF-based tracking and validation on the interface. Detailed protocol selection, reachability tracking, binding limits, and trusted-port configuration are covered on the [IPv6 Device Tracking](device-tracking.md) page.

SISF uses the `device-tracking policy` and `device-tracking attach-policy` command families. This chapter does **not** use the deprecated `ipv6 snooping policy` or `ipv6 nd inspection policy` configurations.

---

## RA Guard and DHCPv6 Guard

These features prevent unauthorized devices from supplying information that hosts normally expect to receive from routers or DHCPv6 infrastructure.

### Router Advertisement Guard

**RA Guard** filters Router Advertisements received on switch ports. Host-facing interfaces should not be allowed to introduce unauthorized router information into the VLAN.

RA Guard can enforce restrictions based on policy conditions, including whether a port is expected to connect to an IPv6 router.

See [RA Guard](ra-guard.md) for policy options, configuration, and verification.

### DHCPv6 Guard

**DHCPv6 Guard** blocks unauthorized DHCPv6 server and relay-agent messages received on protected ports. It prevents a client connected to an ordinary access port from acting as an unauthorized source of DHCPv6 configuration information.

See [DHCPv6 Guard](dhcpv6-guard.md) for message filtering, policy configuration, and verification.

**RA Guard and DHCPv6 Guard are separate features.** Blocking rogue RAs does not automatically block rogue DHCPv6 server messages, and vice versa.

---

## ND Inspection and IPv6 Source Guard

These two functions protect different stages of communication.

### Neighbor Discovery Inspection

**Neighbor Discovery inspection** protects the integrity of address ownership information learned through NDP. For example, it can detect an attacker attempting to claim an IPv6 address that is already associated with another device.

Modern IOS XE implements this functionality through **SISF-based device tracking and binding validation**, rather than through the deprecated standalone IPv6 ND Inspection policy.

See [ND Inspection and Snooping](nd-inspection.md) for ND learning, validation, spoofing detection, and relevant SISF policy settings.

### IPv6 Source Guard

**IPv6 Source Guard** checks the source address of IPv6 data packets entering a protected switch port. It can use the binding table to determine whether the sender is authorized to use that address.

For example, if PC1 is authorized to use `2001:db8:10::10` on `Gi1/0/1`, Source Guard can block a packet arriving on that port with an unauthorized source address such as `2001:db8:10::20`.

**ND inspection protects the validity of Neighbor Discovery information; Source Guard protects IPv6 data traffic against source address spoofing.**

See [IPv6 Source Guard](source-guard.md) for source validation, policy configuration, and verification.

---

## How the IPv6 Security Features Complement Each Other

| Attack or problem | Relevant protection |
| --- | --- |
| A host advertises itself as the default IPv6 router. | **RA Guard** |
| A rogue device sends DHCPv6 server messages. | **DHCPv6 Guard** |
| A device sends forged ND information to claim another host's address. | **SISF guard-level validation / ND inspection** |
| A host sends ordinary IPv6 packets using an unauthorized source address. | **IPv6 Source Guard** |
| A switch must learn where hosts and their IPv6 addresses are connected. | **SISF Device Tracking and Binding Table** |
| A legitimate host disconnects or changes its address. | **SISF tracking, binding state, and aging** |

A useful comparison with IPv4 security is:

- **DHCP Snooping** in IPv4 and **DHCPv6 Guard** in IPv6 both help block rogue DHCP servers, but their configurations and binding mechanisms differ.
- **Dynamic ARP Inspection** protects IPv4 ARP. IPv6 does not use ARP; **Neighbor Discovery** provides address resolution, and SISF-based validation addresses ND spoofing.
- **IPv4 IP Source Guard** and **IPv6 Source Guard** both aim to prevent source address spoofing, using the binding mechanisms appropriate to each feature.

---

## Key Points

- **IPv6 FHS** secures the access layer against rogue routers, unauthorized DHCPv6 servers, Neighbor Discovery attacks, and forged source addresses.
- **SISF** is Cisco IOS XE's unified framework for device tracking and host binding information, supporting IPv4 and IPv6.
- The **SISF binding table** records IP-to-MAC associations together with VLAN, interface, and reachability information.
- **SISF can learn IPv6 addresses from Neighbor Discovery**, not only from DHCPv6.
- **`security-level glean`** learns addresses without ownership validation; **`security-level guard`** provides validation and spoofing protection.
- **RA Guard** filters unauthorized Router Advertisements; **DHCPv6 Guard** filters unauthorized DHCPv6 server and relay messages.
- **SISF-based ND validation** replaces the deprecated standalone ND Inspection policy.
- **IPv6 Source Guard** validates source addresses in IPv6 data packets using authorized bindings.
- **Creating a binding does not automatically enable every FHS feature.** Configure the appropriate enforcement policies for the threats you want to block.

---

## References

- [Cisco: Configuring Switch Integrated Security Features (Catalyst 9300, IOS XE 17.18)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-18/configuration_guide/sec/b_1718_sec_9300_cg/configuring_sisf_based_device_tracking.html)
- [Cisco: Configuring IPv6 First Hop Security (Catalyst 9300, IOS XE 17.18)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-18/configuration_guide/sec/b_1718_sec_9300_cg/configuring_ipv6_first_hop_security.html)
- [Cisco: CCIE Enterprise Infrastructure v1.1 Exam Topics](https://learningcontent.cisco.com/documents/marketing/exam-topics/CCIEEI-v1.1-Final_clean.pdf)
