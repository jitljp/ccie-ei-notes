# 4.2.c (vi) IPv6 Source Guard

**IPv6 Source Guard** prevents IPv6 source-address spoofing by filtering data packets received on a switch interface against authorized IPv6 addresses recorded in the **SISF (Switch Integrated Security Features) binding table**.

It builds on the features covered earlier in this chapter:

- [IPv6 Device Tracking](device-tracking.md) discovers hosts and maintains their IP-to-MAC-to-port information.
- [IPv6 Binding Table](binding-table.md) stores that information, including dynamically learned and static bindings.
- [ND Inspection and Snooping](nd-inspection.md) learns and validates Neighbor Discovery information and can reject conflicting or spoofed control messages.
- **IPv6 Source Guard uses the resulting bindings to filter IPv6 data traffic.** It does not itself discover hosts or inspect ND/DHCPv6 packets.

**Device tracking and IPv6 Source Guard are separate functions.** Enabling SISF tracking, even with `security-level guard`, does not automatically enable IPv6 Source Guard's data-plane filtering.

---

## How IPv6 Source Guard Works

Consider an access switch with two IPv6 hosts:

```text
                         R1 (Gateway)
                              |
                           Gi1/0/24
                              |
                         +---------+
                         |   SW1   |
                         | VLAN 10 |
                         +---------+
                          /       \\
                     Gi1/0/1   Gi1/0/2
                        |         |
                       PC1       PC2
                2001:DB8:10::10  2001:DB8:10::20
                0011.2233.4455  0011.2233.4466
```

SISF has learned these bindings:

| IPv6 address | MAC address | VLAN | Interface |
| --- | --- | --- | --- |
| `2001:DB8:10::10` | `0011.2233.4455` | 10 | `Gi1/0/1` |
| `2001:DB8:10::20` | `0011.2233.4466` | 10 | `Gi1/0/2` |

With IPv6 Source Guard attached to the host-facing interfaces:

1. SISF learns or obtains authorized IPv6 address bindings through ND, DHCPv6, or static configuration.
2. Source Guard uses the binding information to program ingress filtering entries in switch hardware (TCAM).
3. An IPv6 data packet arriving on `Gi1/0/1` with source `2001:DB8:10::10` matches the authorization for that port and is permitted.
4. A packet arriving on `Gi1/0/1` with source `2001:DB8:10::20` is rejected because that address belongs to `Gi1/0/2`, not `Gi1/0/1`.
5. A packet using an unknown IPv6 source address is dropped unless explicitly permitted by the policy.

**An address being present somewhere in the SISF database is not sufficient:** Source Guard uses the address's attachment information when determining whether traffic is authorized on the ingress interface.

### Control Traffic Versus Data Traffic

| Feature | Main traffic examined | Function |
| --- | --- | --- |
| SISF device tracking | ND, DHCPv6, ARP, DHCPv4 and configured learning sources | Learns host identity and attachment |
| SISF `security-level guard` / ND inspection | Address-ownership information in control messages | Validates learning and can reject conflicting control information |
| RA Guard | Router Advertisements | Rejects unauthorized router advertisements |
| DHCPv6 Guard | DHCPv6 server/relay messages | Rejects unauthorized DHCPv6 infrastructure |
| **IPv6 Source Guard** | **IPv6 data packets** | **Drops traffic with unauthorized source addresses** |

Source Guard **does not replace ND inspection, RA Guard, or DHCPv6 Guard**.

Source Guard relies on SISF Device Tracking and its binding table to determine which IPv6 source addresses are permitted in data packets.

Features like ND Inspection, RA Guard, and DHCPv6 guard protect control plane operations.

---

## Dependency on the SISF Binding Table

An IPv6 Source Guard policy is effective only when the switch has an appropriate source of IPv6 bindings. **Enable device tracking for the protected hosts**, whether through an interface, VLAN, or box-wide attachment, so their addresses can be learned through ND and DHCPv6.

If a host's address has not been learned and no static binding exists, Source Guard cannot authorize ordinary data packets from that address. This is why enabling Source Guard without the corresponding device-tracking/snooping setup can prevent legitimate IPv6 connectivity.

### Binding Changes and Enforcement

| Binding event | Source Guard consequence |
| --- | --- |
| A valid binding is learned | The corresponding source address can be permitted on its associated interface |
| Another IPv6 address is learned for the same host | That address can also be authorized independently |
| A binding is removed | Its corresponding permit entry is removed; subsequent ordinary data traffic from that source can be dropped |
| A host moves to another port | SISF must update the host's attachment information before the new port can authorize its traffic |
| A host has no binding | Ordinary data traffic using that source is blocked, subject to policy exceptions |

**One host can have several IPv6 addresses.** For instance, a PC can use a link-local address, a stable SLAAC address, and temporary privacy addresses. Each global address that needs Source Guard authorization must be appropriately represented by the binding information.

---

## Basic Configuration

This example applies device tracking and IPv6 Source Guard to `Gi1/0/1`, where an ordinary IPv6 host connects in VLAN 10.

### Step 1: Configure Device Tracking

```text
SW1(config)# device-tracking policy HOSTS
SW1(config-device-tracking)# security-level guard
SW1(config-device-tracking)# device-role node
SW1(config-device-tracking)# exit
```

The SISF policy learns host address information and validates address ownership. As covered previously, the default learning protocols include ND and DHCPv6.

### Step 2: Configure IPv6 Source Guard

```text
SW1(config)# ipv6 source-guard policy IPV6_SG
SW1(config-sisf-sourceguard)# validate address
SW1(config-sisf-sourceguard)# exit
```

`validate address` makes the policy validate individual IPv6 source addresses against the binding information.

### Step 3: Attach Both Policies to the Access Interface

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# device-tracking attach-policy HOSTS
SW1(config-if)# ipv6 source-guard attach-policy IPV6_SG
```

**Both attachments matter:** `device-tracking attach-policy` enables host learning/validation; `ipv6 source-guard attach-policy` enables data-plane source filtering.

An unnamed Source Guard policy can also be attached with:

```text
SW1(config-if)# ipv6 source-guard
```

This attaches the built-in default Source Guard policy. Merely defining a named policy without attaching it does not apply filtering to any port.

IPv6 Source Guard policies can also be attached at the **box-wide** or **VLAN** scope. The following section explains all three attachment scopes.

---

## Policy Application, Scope, and Defaults

Source Guard policies control **IPv6 data-plane filtering**, whereas device-tracking policies control how the switch **learns, verifies, and maintains bindings**. The two features have independent policy definitions and attachments. Both allow attachments at the **box (global), VLAN, and interface** scopes.

### Attachment Scopes

| Scope | Configuration context | Effect |
| --- | --- | --- |
| **Box-wide** | Global configuration mode | Applies a Source Guard policy across the switch, subject to the applicable ports/VLANs. |
| **VLAN** | `vlan configuration <vlan-id>` | Applies a Source Guard policy to traffic in that VLAN on applicable interfaces. |
| **Interface** | `interface <interface>` | Applies a Source Guard policy to that interface. A Layer 2 Port-channel can also be a target. |

A **policy definition** is not an attachment. Creating `ipv6 source-guard policy IPV6_SG` does not activate that policy by itself. Its effect depends on where `ipv6 source-guard` or `ipv6 source-guard attach-policy` is entered.

### Built-in `default` Policy

The built-in Source Guard policy is named `default`. Its default configuration is:

```text
SW1# show ipv6 source-guard policy default
Source guard policy default configuration:
  validate address
```

Thus, the default policy validates **individual IPv6 source addresses** using authorized binding information. It does not, by default, enable `validate prefix`, `permit link-local`, or `deny global-autoconf`.

The built-in policy exists even if it has not been attached. **Source Guard is not enabled merely because the `default` policy exists.** You must inspect the policy's targets to determine where it is applied. Entering `ipv6 source-guard` without a named policy attaches the built-in `default` policy at the current configuration scope; `ipv6 source-guard attach-policy IPV6_SG` attaches the named policy instead.

### Option 1: Apply Source Guard Box-Wide

Configure Source Guard globally to use the built-in default policy:

```text
SW1(config)# ipv6 source-guard
```

Or attach a named policy globally:

```text
SW1(config)# ipv6 source-guard attach-policy IPV6_SG
```

This is a **box-wide (BOX) attachment**, not just a global policy definition. It is useful when a common Source Guard policy is intended across the switch. Ensure SISF can learn the required host bindings on the affected ports before applying broad enforcement.

### Option 2: Apply Source Guard to a VLAN

Attach the built-in default policy to VLAN 10:

```text
SW1(config)# vlan configuration 10
SW1(config-vlan-config)# ipv6 source-guard
```

Or attach a named policy:

```text
SW1(config)# vlan configuration 10
SW1(config-vlan-config)# ipv6 source-guard attach-policy IPV6_SG
```

This is a **VLAN attachment**, applying the Source Guard policy in the selected VLAN across its applicable interfaces. `vlan configuration 10` configures a VLAN-based feature policy; it is **not** the same as configuring `interface Vlan10` (an SVI).

### Option 3: Apply Source Guard to an Interface

Attach a named policy to one host-facing interface:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# ipv6 source-guard attach-policy IPV6_SG
```

To use the built-in default policy:

```text
SW1(config-if)# ipv6 source-guard
```

A Layer 2 Port-channel interface can also be configured as a Source Guard attachment target:

```text
SW1(config)# interface Port-channel1
SW1(config-if)# ipv6 source-guard attach-policy IPV6_SG
```

### Combining Source Guard and Device-Tracking Scopes

There is no requirement for the **device-tracking policy attachment** and **Source Guard policy attachment** to use the same scope. For example, tracking can be enabled for VLAN 10 while Source Guard is attached to one particular interface:

```text
SW1(config)# device-tracking policy HOSTS
SW1(config-device-tracking)# security-level guard
SW1(config-device-tracking)# device-role node
SW1(config-device-tracking)# exit
!
SW1(config)# vlan configuration 10
SW1(config-vlan-config)# device-tracking attach-policy HOSTS
SW1(config-vlan-config)# exit
!
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# ipv6 source-guard attach-policy IPV6_SG
```

The VLAN-scoped device-tracking policy learns the host's bindings; interface-scoped Source Guard uses the appropriate bindings to filter traffic entering `Gi1/0/1`. Similarly, box-wide Source Guard can use bindings learned through interface-specific tracking policies.

### Verify Policy Attachments

Use the policy display to inspect both **contents** and **attachment targets**:

```text
SW1# show ipv6 source-guard policy
Source guard policy default configuration:
  validate address
Policy default is applied on the following targets:
Target               Type  Policy               Feature        Target range
vlan 1               VLAN  default              Source guard   vlan all
Box                  BOX   default              Source guard   vlan all
```

Here, `VLAN` and `BOX` in the **Type** column confirm that Source Guard is not limited to interface attachments. `vlan all` in the **Target range** column does not turn the `vlan 1` entry into a box-wide attachment; interpret it together with the target type.

To inspect a particular named policy:

```text
SW1# show ipv6 source-guard policy IPV6_SG
```

To check tracking policies and bindings as well:

```text
SW1# show device-tracking policies
SW1# show device-tracking database
```

When policies are attached at multiple scopes, inspect the resulting policy targets and effective behavior rather than assuming that multiple listed attachments mean multiple sequential checks. The operational priority of overlapping policies should be verified on the target IOS XE platform.

### Remove an Attachment

Remove the attachment in the same configuration context in which it was applied:

```text
SW1(config)# no ipv6 source-guard
SW1(config)# vlan configuration 10
SW1(config-vlan-config)# no ipv6 source-guard
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# no ipv6 source-guard
```

Removing a Source Guard attachment does **not** remove the underlying SISF device-tracking policy or delete its host bindings.

---

## IPv6 Source Guard Policy Options

| Command | Function |
| --- | --- |
| `validate address` | Validates IPv6 source addresses using individual learned or configured address bindings |
| `no validate address` | Disables individual-address validation, used when configuring prefix-based validation |
| `validate prefix` | Enables IPv6 Prefix Guard to validate traffic using authorized prefixes |
| `permit link-local` | Permits data traffic sourced from IPv6 link-local addresses without requiring an individual source binding |
| `deny global-autoconf` | Denies data traffic sourced from autoconfigured global IPv6 addresses |

The policy option `trusted` appears in some earlier Cisco documentation, but Cisco's current Catalyst SISF guidance states that the **Source Guard `trusted` option is not supported**. Do not use it as the normal way to bypass validation.

### `permit link-local`

IPv6 uses link-local addresses (`FE80::/10`) for essential local functions, including router-neighbor communication and some routing-protocol traffic.

```text
SW1(config)# ipv6 source-guard policy IPV6_SG
SW1(config-sisf-sourceguard)# permit link-local
```

This permits **all IPv6 data traffic sourced from link-local addresses** on the protected interface, rather than requiring individual bindings for those source addresses.

This is useful where link-local control traffic must continue to work, but **it is an exception to address-level filtering**. It does not mean that a specific link-local address was verified as belonging to a particular host.

### `deny global-autoconf`

```text
SW1(config)# ipv6 source-guard policy IPV6_SG
SW1(config-sisf-sourceguard)# deny global-autoconf
```

This denies traffic sourced from **autoconfigured global IPv6 addresses**, useful in a design where global IPv6 addresses must be obtained through an approved address-assignment process rather than SLAAC.

Do not enable this option for a typical SLAAC-based host VLAN unless you intend to block those autoconfigured source addresses. **SLAAC addresses are not inherently untrusted:** ordinary address validation can authorize them when SISF learns and validates their bindings.

---

## IPv6 Prefix Guard

**IPv6 Prefix Guard** uses the same Source Guard policy framework but validates source addresses against **authorized prefixes** rather than against individual host addresses.

This is particularly useful when a downstream router receives an IPv6 prefix through **DHCPv6 Prefix Delegation (DHCPv6-PD)** and sends traffic sourced from many addresses within that delegated prefix.

For example:

```text
                         SW1
                          |
                       Gi1/0/1
                          |
                    Customer Router
                   Delegated prefix:
                     2001:DB8:100::/56
                          |
                   Downstream IPv6 LANs
```

Address-level Source Guard would need individual entries for the downstream sources. Prefix Guard can instead permit addresses within an authorized delegated prefix and deny sources outside that prefix.

### Configure Prefix Guard

```text
SW1(config)# ipv6 source-guard policy IPV6_PREFIX_GUARD
SW1(config-sisf-sourceguard)# no validate address
SW1(config-sisf-sourceguard)# validate prefix
SW1(config-sisf-sourceguard)# permit link-local
SW1(config-sisf-sourceguard)# exit
!
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# ipv6 source-guard attach-policy IPV6_PREFIX_GUARD
```

A suitable SISF learning policy is also required on the interface so the authorized prefix can be learned and maintained. **Prefix Guard does not mean that every address in any observed prefix is automatically authorized.** The prefix must be learned or supplied through an appropriate trusted source, such as DHCPv6-PD.

### Can `validate address` and `validate prefix` Be Used Together?

**No.** On Cisco Catalyst switches, IPv6 Source Guard and Prefix Guard are alternative validation modes. A policy attached to a protected target should use **either** `validate address` **or** `validate prefix`, not both. Even if the CLI accepts both commands, do not assume that the switch will perform combined address-and-prefix validation.

For prefix-only validation, use `no validate address` before `validate prefix`, as shown above. For address-only validation, configure `validate address` and `no validate prefix`. Verify the policy with `show ipv6 source-guard policy <policy-name>`.

Cisco reference: [Configuring IPv6 First Hop Security (Catalyst 9600)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9600/software/release/26-x/configuration_guide/sec/b_26x_sec_9600_cg/configuring_ipv6_first_hop_security.html).

| Mode | Validates | Typical use |
| --- | --- | --- |
| IPv6 Source Guard (`validate address`) | Individual IPv6 source addresses | End hosts using SLAAC, DHCPv6, or static IPv6 addresses |
| IPv6 Prefix Guard (`validate prefix`) | Source addresses against authorized IPv6 prefixes | Customer routers or downstream devices using delegated prefixes |

---

## Missing Bindings and Data-Glean Recovery

Consider a host that already holds a valid DHCPv6 address, but its SISF entry disappears because the switch restarted or the binding database was cleared.

1. The host still considers its assigned address valid and may not immediately restart DHCPv6.
2. Source Guard has no binding for the host's IPv6 source address.
3. The switch drops the host's ordinary data packets.
4. The host may remain disconnected until SISF relearns or recovers the binding.

SISF **data-glean recovery** provides a recovery mechanism for this situation. When an unknown source generates data traffic, SISF can attempt to recover the binding rather than requiring the host to wait for a routine DHCP renewal.

### Recover from DHCPv6

```text
SW1(config)# device-tracking policy HOSTS
SW1(config-device-tracking)# data-glean recovery dhcp
```

This configures SISF to request missing binding information through DHCP recovery. The DHCPv6 server must support and be configured for the relevant **DHCPv6 Leasequery** function.

Some software also provides an NDP-based recovery option:

```text
SW1(config-device-tracking)# data-glean recovery ndp
```

This attempts to recover missing address information using Neighbor Discovery rather than DHCPv6 Leasequery.

**Recovery is not a bypass:** ordinary traffic is not simply authorized because it triggered a lookup. The binding must be recovered and accepted before the appropriate data-plane authorization can be installed.

See [IPv6 Device Tracking](device-tracking.md) for other data-, destination-, and prefix-gleaning options. Source Guard is one of the clearest reasons why recovery is operationally important: without a binding, its default action is to deny the data traffic.

---

## Static Bindings and Hosts That Do Not Generate Learning Traffic

For a device with a known, fixed IPv6 address, an administrator can configure a binding manually:

```text
SW1(config)# device-tracking binding vlan 10 2001:DB8:10::10 interface GigabitEthernet1/0/1 0011.2233.4455
```

Source Guard can use the static binding to authorize the specified IPv6 source address on the associated interface.

This can be useful for devices that rarely send ND or DHCPv6 traffic, or when an explicitly managed address-to-port assignment is required. Static bindings are not a substitute for handling hosts that use frequently changing privacy addresses.

---

## Verification

### Verify the Source Guard Policy

```text
SW1# show ipv6 source-guard policy IPV6_SG
```

Confirm the policy parameters and its box-wide, VLAN, or interface attachment targets.

### Verify SISF Attachments

```text
SW1# show device-tracking policies interface GigabitEthernet1/0/1
```

Verify the intended policy attachments. For box-wide or VLAN-scoped Source Guard, also inspect `show ipv6 source-guard policy` to see its complete attachment targets.

### Verify Host Bindings

```text
SW1# show device-tracking database
```

Look for the expected IPv6 address, MAC address, VLAN, interface, and binding state. If a legitimate host is being blocked, **first check whether its address has a valid binding on the correct port**.

### Inspect the Device-Tracking Policy

```text
SW1# show device-tracking policy HOSTS
```

Check that ND/DHCPv6 learning is enabled as intended and that the policy is attached to the correct interface.

### Debugging

```text
SW1# debug ipv6 snooping source-guard
```

Use debugging only when ordinary policy and binding verification do not explain the behavior; disable it when finished.

---

## Troubleshooting Scenarios

| Symptom | Likely cause | What to verify |
| --- | --- | --- |
| Source Guard enabled; legitimate host cannot send IPv6 data | No usable SISF binding | Device-tracking policy attachment, ND/DHCPv6 learning, database contents |
| Traffic from PC1 is blocked after it moves to another port | Binding still references old port | Entry interface and host-move learning |
| One of a host's IPv6 addresses works, another fails | Only one address is currently authorized | Separate link-local, stable, and privacy-address bindings |
| Hosts using SLAAC cannot send global IPv6 traffic | `deny global-autoconf` is configured, or their bindings are missing | Source Guard policy and SISF database |
| DHCPv6 client loses connectivity after a database reset | Missing binding despite valid lease | `data-glean recovery dhcp`, DHCPv6 Leasequery support |
| Traffic from downstream delegated prefixes fails | Individual-address mode used where prefix authorization is needed | Prefix Guard policy and delegated-prefix learning |
| Unexpected link-local drops | Link-local data traffic not authorized | Whether `permit link-local` is appropriate |

### Device-Tracking Address Limits

SISF `limit address-count` limits learned IP bindings on a protected port. If the limit is reached, additional IPv6 source addresses may not enter the binding table, which can subsequently cause IPv6 Source Guard to block their data traffic.

This is especially relevant when a host uses multiple IPv6 global and temporary addresses. The address-count limit is a **device-tracking policy setting**, not a Source Guard policy setting.

---

## Key Points

- **SISF device tracking learns and validates bindings. IPv6 Source Guard enforces those bindings against ingress IPv6 data traffic.**
- **`security-level guard` is not equivalent to enabling IPv6 Source Guard.** The former helps protect learned control-plane information; the latter filters ordinary IPv6 data packets.
- **Source Guard requires valid bindings** and normally uses hardware-programmed ingress filters to permit authorized source addresses on the correct interface.
- **Source Guard does not inspect ND or DHCPv6 messages.** It depends on the separate SISF learning and validation functions.
- **`validate address`** checks individual IPv6 sources; **`validate prefix`** provides Prefix Guard for authorized IPv6 prefixes.
- **`permit link-local`** permits data traffic sourced from link-local addresses; **`deny global-autoconf`** rejects traffic sourced from autoconfigured global addresses.
- **Data-glean recovery** helps restore missing bindings when a legitimate host is already using its address but has not generated a fresh DHCPv6 or ND learning exchange.

---

## References

- [Cisco FHS and SISF Configuration Guide: IPv6 First Hop Security](https\://www\.cisco.com/c/en/us/td/docs/switches/lan/c9000/sec-crypto/fhs-sisf/fhs-and-sisf-configuration-guide/ipv6-first-hop-security.html)
- [Cisco Security and VPN Configuration Guide: IPv6 Source Guard and Prefix Guard](https\://www\.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_ipv6-src-guard-xe.html)
- [Cisco: Troubleshoot SISF on Catalyst 9000 Series Switches](https\://www\.cisco.com/c/en/us/support/docs/switches/catalyst-9300-series-switches/221562-troubleshoot-sisf-on-catalyst-9000-serie.html)
