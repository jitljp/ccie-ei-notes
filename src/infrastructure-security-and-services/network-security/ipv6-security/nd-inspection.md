# 4.2.c (v) ND Inspection and Snooping

**IPv6 Neighbor Discovery (ND) inspection** protects hosts against forged or conflicting Neighbor Discovery information. It observes ND messages, learns IPv6-to-MAC address relationships, and checks whether a device is entitled to claim an IPv6 address on a particular switch port and VLAN.

**ND snooping** refers to observing ND traffic to learn host information. **ND inspection** adds validation and, when configured to enforce it, drops unacceptable ND messages. In modern Cisco IOS XE, these functions are implemented using **SISF (Switch Integrated Security Features) device tracking** and its shared binding table.

This page focuses on **ND packet validation, address ownership, and spoofing protection**. The mechanics of learning host addresses, tracking their reachability, and managing binding lifetimes are covered separately in [IPv6 Device Tracking](device-tracking.md) and [IPv6 Binding Table](binding-table.md).

> ND Inspection and Snooping are simply aspects of Device Tracking, the details of which we've already covered.

---

## Why ND Inspection Is Needed

IPv6 uses **Neighbor Discovery Protocol (NDP)**, rather than ARP, for several essential functions. ND messages use ICMPv6:

| ICMPv6 type | Message | Relevant function |
| --- | --- | --- |
| **133** | Router Solicitation (RS) | Ask routers for configuration information. |
| **134** | Router Advertisement (RA) | Advertise routers, prefixes, and other host configuration parameters. |
| **135** | Neighbor Solicitation (NS) | Address resolution, Neighbor Unreachability Detection (NUD), and Duplicate Address Detection (DAD). |
| **136** | Neighbor Advertisement (NA) | Report an IPv6-to-MAC association or respond to address resolution. |
| **137** | Redirect | Tell a host about a better next hop. |

ND normally assumes that devices on the local link cooperate honestly. An attacker on the VLAN could exploit this by:

- **Neighbor cache poisoning:** Sending a forged NA that associates a victim's IPv6 address with the attacker's MAC address.
- **Address theft:** Claiming an IPv6 address already used by another host.
- **DAD interference:** Pretending that an address being tested during Duplicate Address Detection is already in use.
- **Neighbor cache disruption:** Sending inconsistent ND information to interfere with normal address resolution and communication.

The switch can detect suspicious claims by comparing what it sees in ND traffic with the **SISF binding table**, which records host IP addresses, MAC addresses, VLANs, and attachment ports.

**ND inspection is not the same as RA Guard.** RA Guard targets unauthorized router advertisements. ND inspection focuses especially on the integrity of host address ownership and ND-derived bindings. Both can be part of a first-hop security deployment.

---

## ND Snooping and the Binding Table

Consider the following access network:

```text
                     R1 (IPv6 Gateway)
                            |
                         Gi1/0/24
                            |
                      +-----------+
                      |    SW1    |
                      |  VLAN 10  |
                      |   SISF    |
                      +-----------+
                       /         \
                   Gi1/0/1     Gi1/0/2
                      |           |
                     PC1         PC2

PC1 MAC: 0011.2233.4455       PC2 MAC: 0011.2233.4466
PC1 IPv6: 2001:DB8:10::10     PC2 IPv6: 2001:DB8:10::20
```

With SISF configured, the switch observes applicable ND exchanges received from PC1 and PC2. Its binding table might contain:

| IPv6 address | MAC address | VLAN | Interface |
| --- | --- | --- | --- |
| `2001:DB8:10::10` | `0011.2233.4455` | 10 | `Gi1/0/1` |
| `2001:DB8:10::20` | `0011.2233.4466` | 10 | `Gi1/0/2` |

This association is useful because it establishes **where a particular IPv6 address was learned**. It does not mean that a MAC address is limited to one IPv6 address: a single IPv6 host commonly uses a link-local address and multiple global addresses.

### Snooping Versus Inspection

| Operation | What the switch does |
| --- | --- |
| **Snooping / gleaning** | Observes ND information and populates or refreshes host bindings. |
| **Inspection / validation** | Compares incoming address claims with existing bindings and the attached policy. |
| **Enforcement** | Drops ND packets that fail validation when the applicable policy uses `security-level guard`. |

**Learning a binding is not proof that the address belongs to the sender.** Whether the switch validates ownership depends on the SISF security level.

---

## SISF Security Levels

A device-tracking policy determines how much verification is applied to learned information.

| Security level | Behavior |
| --- | --- |
| `glean` | Learns IP/MAC information without verifying address ownership. Does not drop an ND packet solely because binding verification fails. |
| `inspect` | Intermediate inspection mode. Cisco recommends using `glean` or `guard` instead. |
| `guard` | Learns and validates bindings. Rejects invalid or conflicting claims and can drop the corresponding ND packet. **Default.** |

For ND spoofing protection, **`security-level guard`** is the important setting.

### Validation Under `security-level guard`

When SISF receives relevant ND information, the general decision process is:

1. **Extract the claimed address and attachment information**, including the IP address, MAC address, VLAN, and receiving interface.
2. **Look for an existing matching binding** in the SISF database.
3. **No conflicting binding:** The switch can create a new binding if permitted by the policy and applicable limits.
4. **Same attachment:** If the existing entry identifies the same host and point of attachment, SISF can refresh it.
5. **Different attachment or ownership claim:** SISF performs further validation to decide whether this is a legitimate host move or an attempt to steal an address. Verification can include probing the previous owner.
6. **Validation fails:** The conflicting binding is not accepted, and the relevant ND packet can be **dropped**.

An IP address changing ports is **not automatically an attack**. For example, a host can legitimately reconnect to a different switch port. The purpose of validation is to distinguish legitimate changes from unauthorized claims.

### Example: Forged Neighbor Advertisement

Suppose SW1 has already learned:

```text
2001:DB8:10::10 -> 0011.2233.4455, VLAN 10, Gi1/0/1
```

PC2 then sends an NA claiming that `2001:DB8:10::10` belongs to MAC `0011.2233.4466` on `Gi1/0/2`.

With `security-level guard`, SW1 recognizes the conflicting claim. The switch checks the existing ownership information rather than blindly replacing the entry. If the new claim cannot be validated, **the forged ND packet is dropped and the legitimate binding is preserved**.

Under `security-level glean`, SISF does not provide that ownership-verification protection. Therefore, **a populated binding table alone is not sufficient to prevent ND spoofing**.

### What `guard` Does Not Do

`security-level guard` is not a general-purpose IPv6 data-plane source filter. It provides SISF validation of supported control messages, including ND, and protects binding integrity. To enforce a source IPv6 address restriction on ordinary data packets, see [IPv6 Source Guard](source-guard.md).

---

## ND Messages and Address Ownership

### Neighbor Solicitation (NS)

An NS can be used for:

- **Address resolution:** A host asks for the link-layer address corresponding to a known IPv6 address.
- **NUD:** A host checks whether a previously discovered neighbor is still reachable.
- **DAD:** A host tests whether an address it wants to use is already assigned to another device.

SISF can learn and validate address information from relevant NS messages, but the **target address of an ordinary NS is not automatically an address owned by the sender**. For example, when PC1 asks for PC2's MAC address, the target is PC2's address.

### Neighbor Advertisement (NA)

An NA can carry an IPv6 address claim and associated link-layer information. Forged NAs are particularly relevant to neighbor cache poisoning.

For example, an attacker could announce:

```text
Claimed IPv6 address: 2001:DB8:10::10
Claimed MAC address:  0011.2233.4466
Ingress interface:   Gi1/0/2
```

If the switch already has a trustworthy, conflicting binding for that address, **guard-level validation** can prevent the forged association from replacing it.

### Duplicate Address Detection (DAD)

Before using a tentative IPv6 address, a host normally sends a DAD NS:

```text
IPv6 source:       ::
IPv6 destination:  Solicited-node multicast address
NS target:         Tentative IPv6 address
```

The source is `::` because the host has not yet established that its tentative address is unique.

An attacker may try to interfere with DAD by claiming that the tentative address is already in use. SISF's ND validation and ownership checks help protect against this kind of address theft or interference.

**Do not interpret `::` as an ordinary host address binding.** DAD must be interpreted using the NS target and other relevant ND information.

### Router Advertisements and Redirects

RA and Redirect messages are also ND messages, but their primary security concerns involve **router impersonation and manipulation of forwarding decisions**. Use [RA Guard](ra-guard.md) to apply dedicated controls to router advertisements. `security-level guard` also includes protection against unauthorized router-related control messages on node-facing targets.

---

## Configuring ND Inspection with SISF

Modern IOS XE uses `device-tracking policy`, not the deprecated standalone `ipv6 nd inspection policy` or `ipv6 snooping policy` configuration.

### Step 1: Create the Policy

```text
SW1(config)# device-tracking policy ND_GUARD
SW1(config-device-tracking)# security-level guard
SW1(config-device-tracking)# device-role node
SW1(config-device-tracking)# exit
```

- `security-level guard` enables address-ownership validation and rejection of invalid control packets. This is also the default security level.
- `device-role node` identifies the connected target as an end-host attachment. This is also the default role.
- The policy permits learning from supported protocols, including **NDP**, unless protocol learning is changed explicitly.

### Step 2: Apply the Policy to Host-Facing Ports

```text
SW1(config)# interface range GigabitEthernet1/0/1 - 2
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 10
SW1(config-if-range)# device-tracking attach-policy ND_GUARD
```

**Attaching the policy activates it.** Creating a policy by itself does not enable inspection on every switch port.

Alternatively, attach the built-in `default` device-tracking policy to an interface:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# device-tracking
```

The built-in default policy uses `security-level guard`, `device-role node`, and learning from ND and the other default supported protocols.

### Step 3: Handle Infrastructure-Facing Ports

A port connected to an authorized router, another switch, or a DHCPv6 server must be treated according to its actual role. **Do not blindly apply a host-facing `device-role node` guard policy to an infrastructure uplink.** Otherwise, legitimate router or server control messages may be rejected.

SISF provides the following policy controls:

| Command | Purpose |
| --- | --- |
| `device-role node` | Identifies the attachment as an end host. |
| `device-role switch` | Identifies an attachment to another switch. |
| `trusted-port` | Treats information learned on the target as trusted for SISF binding validation. |

**A trusted port is not merely a port on which all security checks continue unchanged.** Trust alters binding validation, and bindings learned on trusted ports receive preference in a conflict. Trust should be reserved for genuinely trusted infrastructure links.

For router-facing interfaces, also configure [RA Guard](ra-guard.md) appropriately. For DHCPv6 server or relay-facing interfaces, see [DHCPv6 Guard](dhcpv6-guard.md).

### Optional: Limit Learned Addresses

An attacker might attempt to fill the binding database by claiming many addresses. You can restrict the number of learned addresses using a device-tracking policy:

```text
SW1(config)# device-tracking policy ND_GUARD
SW1(config-device-tracking)# limit address-count 10
```

This limits the number of host addresses tracked on the applicable target. Remember that one legitimate IPv6 host can have **multiple IPv6 addresses**, so a limit of one address per host-facing port would generally be too restrictive.

See [IPv6 Device Tracking](device-tracking.md) for additional address-count limits and policy configuration.

---

## Verification and Troubleshooting

### Inspect the Active Policy

```text
SW1# show device-tracking policy ND_GUARD
SW1# show device-tracking policies
```

Check that the policy uses **`security-level guard`**, permits ND learning, and is attached to the intended interfaces or VLANs.

### Inspect the Binding Table

```text
SW1# show device-tracking database
SW1# show device-tracking database interface GigabitEthernet1/0/1
```

The database displays fields such as the IP address, MAC address, interface, VLAN, source, age, state, and remaining lifetime. `ND` indicates Neighbor Discovery as the learning source.

A normal entry should associate the host's IPv6 address with its expected MAC address and attachment port. If an address unexpectedly appears on a different port, investigate whether the host legitimately moved or whether another device is making a conflicting claim.

### Inspect Drops and Counters

```text
SW1# show device-tracking counters
```

SISF counters can help identify dropped packets and validation activity. **A drop counter alone does not prove an ND spoofing attack:** limits, invalid messages, and other SISF checks can also cause drops.

### Troubleshooting Checklist

1. **No binding appears:** Confirm that the policy is attached and that the switch has seen relevant ND traffic. A silent host may not yet be learned.
2. **A legitimate host's ND traffic is dropped:** Check for a conflicting binding, incorrect MAC/port information, or a restrictive address-count limit.
3. **A host moves to another port:** Check whether an old binding is still present and whether SISF can validate the new attachment.
4. **Router or DHCPv6 infrastructure stops working:** Check whether a node-facing guard policy was accidentally applied to an infrastructure port, and verify the separate RA Guard/DHCPv6 Guard policies.
5. **A spoofed ordinary IPv6 data packet is forwarded:** ND inspection alone is not IPv6 Source Guard. Verify whether source-address enforcement is separately configured.

---

## ND Inspection Compared with Related Features

| Feature | Primary protection | Traffic or information involved |
| --- | --- | --- |
| **SISF ND snooping** | Learns IPv6 hosts and their attachment information. | ND control messages and binding updates. |
| **SISF ND inspection (`guard`)** | Detects conflicting address ownership claims and drops invalid ND messages. | ND control messages and binding validation. |
| **RA Guard** | Prevents unauthorized router advertisements. | RAs and policy-controlled router information. |
| **DHCPv6 Guard** | Blocks unauthorized DHCPv6 servers or relays. | DHCPv6 server/relay messages. |
| **IPv6 Source Guard** | Blocks ordinary IPv6 packets with unauthorized source addresses. | IPv6 data packets and authorized bindings. |

**Key distinction:** ND inspection protects the information used to identify IPv6 neighbors; IPv6 Source Guard uses authorized binding information to restrict the source addresses permitted in IPv6 data traffic.

---

## Legacy IPv6 ND Inspection Commands

Older Cisco documentation describes the following standalone features:

```text
ipv6 nd inspection policy <name>
ipv6 snooping policy <name>
ipv6 snooping attach-policy <name>
```

These belong to the earlier IPv6 First Hop Security configuration model. Cisco states that **standalone IPv6 ND Inspection was deprecated beginning with IOS XE 17.1.1**, with SISF replacing its functionality. For current SISF configurations, use **`device-tracking policy`** and **`device-tracking attach-policy`** instead.

### References

- [Cisco: FHS and SISF Configuration Guide, SISF](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/sec-crypto/fhs-sisf/fhs-and-sisf-configuration-guide/switch-integrated-security-features.html)
- [Cisco: Configuring IPv6 First Hop Security, Catalyst 9600 IOS XE 26.x](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9600/software/release/26-x/configuration_guide/sec/b_26x_sec_9600_cg/configuring_ipv6_first_hop_security.html)
- [Cisco: IPv6 Snooping, IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_ip6_snooping-xe.html)
