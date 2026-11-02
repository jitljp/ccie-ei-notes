# 4.2.c (iv) IPv6 Device Tracking

**SISF (Switch Integrated Security Features) device tracking** discovers and monitors hosts connected to a switch. It learns their **IPv4 and IPv6 addresses**, MAC addresses, VLANs, and points of attachment by inspecting supported control-plane messages. It can also actively check whether a previously learned host remains reachable.

Although the CCIE blueprint calls this topic **IPv6 Device Tracking**, modern Cisco IOS XE uses a **single SISF framework for both IPv4 and IPv6**. This page covers both address families using the `device-tracking` command family.

Device tracking has three principal functions:

1. **Discovery:** Learn host addresses and attachment information from ARP, DHCPv4, Neighbor Discovery (ND), and DHCPv6.
2. **Validation:** Determine whether new or changed address ownership information should be accepted, according to a policy's security level.
3. **Reachability tracking:** Refresh and verify existing host information using received control messages and, if enabled, active probes.

The learned information is stored in the [IPv6 Binding Table](binding-table.md), a unified IPv4/IPv6 database. That page covers individual binding entries, source codes, states, lifetimes, static bindings, and database operations in detail.

---

## Device Tracking and the SISF Framework

Consider an access switch with two dual-stack hosts:

```text
                         R1 (Gateway)
                 192.0.2.1 / 2001:DB8:10::1
                              |
                           Gi1/0/24
                              |
                         +---------+
                         |   SW1   |
                         | VLAN 10 |
                         +---------+
                          /       \
                     Gi1/0/1   Gi1/0/2
                        |         |
                       PC1       PC2
PC1: 0011.2233.4455             PC2: 0011.2233.4466
     192.0.2.10                      192.0.2.20
     2001:DB8:10::10                 2001:DB8:10::20
```

As PC1 and PC2 communicate, SW1 can observe ARP, DHCPv4, ND, and DHCPv6 messages on their access ports. Device tracking uses those observations to associate each IP address with a MAC address, VLAN, and switch interface.

For example, one host can have several bindings:

| Address | Learning mechanism | Host MAC | Port |
| --- | --- | --- | --- |
| `192.0.2.10` | ARP or DHCPv4 | `0011.2233.4455` | `Gi1/0/1` |
| `FE80::211:22FF:FE33:4455` | ND | `0011.2233.4455` | `Gi1/0/1` |
| `2001:DB8:10::10` | ND or DHCPv6 | `0011.2233.4455` | `Gi1/0/1` |

The switch **does not need to be the hosts' default gateway** to track them. A Layer 2 access switch can inspect the relevant messages without an IPv4 or IPv6 SVI in the client VLAN.

### Device Tracking Versus Other Features

| Feature | Primary responsibility |
| --- | --- |
| **SISF device tracking** | Learn, validate, and monitor IPv4/IPv6 host addresses and locations. |
| **SISF binding table** | Store those bindings and their states. |
| **IPv4 DHCP snooping** | Control DHCPv4 trust and maintain its DHCP snooping binding database. |
| **RA Guard** | Block unauthorized IPv6 Router Advertisements. |
| **DHCPv6 Guard** | Block unauthorized DHCPv6 server/relay messages. |
| **ND inspection** | Protect the integrity of IPv6 Neighbor Discovery and address ownership. |
| **IPv6 Source Guard** | Filter IPv6 data traffic using authorized source addresses. |

**Device tracking is not itself a blanket IP data-plane filter.** Learning an address does not automatically authorize all traffic from that address, and enabling device tracking does not automatically enable IPv6 Source Guard. However, SISF with `security-level guard` can reject certain control packets, including unauthorized Router Advertisements, without a separate RA Guard policy.

---

## How Device Tracking Learns Addresses

Device tracking primarily learns from **control traffic received on a protected interface**. The supported learning protocols are configured using `protocol` under a device-tracking policy.

| Policy protocol | Address family | Packets inspected | Example of learned information |
| --- | --- | --- | --- |
| `arp` | IPv4 | ARP Requests and Replies | IPv4 sender address and MAC |
| `dhcp4` | IPv4 | DHCPv4 exchanges | DHCP-assigned IPv4 address and client identity |
| `ndp` | IPv6 | Neighbor Discovery, including NS and NA | IPv6 address and associated MAC |
| `dhcp6` | IPv6 | DHCPv6 exchanges | Assigned IPv6 addresses and delegated prefixes |

By default, SISF policies permit gleaning from these four protocols. **SISF does not require DHCP to learn an address:** ARP can identify a statically addressed IPv4 host, and ND can identify an IPv6 host using SLAAC or a manually configured address.

### IPv4: ARP and DHCPv4

A host with a static address can generate a binding simply by sending an ARP Request:

```text
PC1 sends an ARP Request on VLAN 10:
Sender IPv4: 192.0.2.10
Sender MAC:  0011.2233.4455
Ingress:     Gi1/0/1
```

SW1 associates the IPv4 address and MAC with `Gi1/0/1`. Subsequent relevant ARP messages can refresh the entry and reveal a change of attachment.

DHCPv4 exchanges provide an additional source of address assignment information. The source code shown for an entry in `show device-tracking database` indicates which mechanism supplied the recorded information; it is not a guarantee that every DHCP-configured host will always appear with a DHCP source code.

**IPv4 ARP and DHCPv4 learning are independent of whether the access switch has an SVI in that VLAN.**

```
SW1# show device-tracking database
 VPC role NONE VPC state CREATING
Binding Table has 12 entries, 12 dynamic (limit 200000)
Codes: L - Local, S - Static, ND - Neighbor Discovery, ARP - Address Resolution Protocol, DH4 - IPv4 DHCP, DH6 - IPv6 DHCP, PKT - Other Packet, API - API created
Preflevel flags (prlvl):
0001:MAC and LLA match     0002:Orig trunk            0004:Orig access          &#x20;
0008:Orig trusted trunk    0010:Orig trusted access   0020:DHCP assigned        &#x20;
0040:Cga authenticated     0080:Cert authenticated    0100:Statically assigned  &#x20;
    Network Layer Address                    Link Layer Address     Interface  vlan/bd    prlvl      age        state      Time left      &#x20;
ARP 10.1.2.12                                aabb.cc00.3000         Et0/1      1          0005       69s        REACHABLE  239 s          &#x20;
ARP 10.1.2.11                                aabb.cc00.2600         Et0/2      1          0005       70s        REACHABLE  243 s          &#x20;
ARP 10.1.2.2                                 aabb.cc00.2e00         Et0/1      1          0005       44s        REACHABLE  266 s          &#x20;
ARP 10.1.2.1                                 aabb.cc00.2800         Et0/0      1          0005       57s        REACHABLE  252 s          &#x20;
ND  FE80::A8BB:CCFF:FE00:3000                aabb.cc00.3000         Et0/1      1          0005       35s        REACHABLE  276 s          &#x20;
ND  FE80::A8BB:CCFF:FE00:2E00                aabb.cc00.2e00         Et0/1      1          0005       44s        REACHABLE  262 s          &#x20;
ND  FE80::A8BB:CCFF:FE00:2600                aabb.cc00.2600         Et0/2      1          0005       31s        REACHABLE  269 s          &#x20;
ND  FE80::2                                  aabb.cc00.2e00         Et0/1      1          0005       1s         REACHABLE  307 s          &#x20;
ND  FE80::1                                  aabb.cc00.2800         Et0/0      1          0005       10s        REACHABLE  304 s          &#x20;
ND  2001:DB8::A8BB:CCFF:FE00:3000            aabb.cc00.3000         Et0/1      1          0005       34s        REACHABLE  276 s          &#x20;
ND  2001:DB8::2                              aabb.cc00.2e00         Et0/1      1          0005       1s         REACHABLE  305 s          &#x20;
ND  2001:DB8::1                              aabb.cc00.2800         Et0/0      1          0005       9s         REACHABLE  295 s      &#x20;
```

### IPv6: Neighbor Discovery and DHCPv6

For IPv6, the switch can observe ND messages as the host configures and uses its addresses:

- **Neighbor Solicitations (NS):** Used during address resolution, Neighbor Unreachability Detection, and Duplicate Address Detection (DAD).
- **Neighbor Advertisements (NA):** Provide information associating IPv6 addresses with link-layer addresses.
- **DHCPv6 exchanges:** Can communicate IA_NA addresses and IA_PD delegated prefixes.

A host using SLAAC may have a link-local address, a stable global address, and one or more temporary privacy addresses. Each address can occupy a separate binding-table entry, even though the MAC, VLAN, and physical port are the same.

A DAD Neighbor Solicitation uses the unspecified IPv6 source address (`::`). Do not treat `::` as an ordinary, completed host binding. SISF uses the appropriate ND information and validation process to learn the address being checked.

A Router Advertisement advertises network configuration and possibly prefixes; it does **not** itself prove that a particular host owns every address within an advertised prefix.

### Passive Learning Is Not Active Tracking

**Passive learning** occurs when a host generates a message SISF can inspect. **Active tracking** means the switch sends probes to check whether an existing host is still present.

These are separate functions. A policy can learn IPv4/IPv6 addresses **without active polling**. Enabling `tracking enable` is not required merely to create initial bindings.

Likewise, an ICMP Echo Request or Reply is not one of the ordinary ARP/ND/DHCP learning sources. A `ping` may **indirectly** trigger ARP or ND and cause a binding to be created, but a successful `ping` by itself does not guarantee a new SISF entry.

---

## Enabling and Configuring Device Tracking

Like [RA Guard](ra-guard.md) and [DHCPv6 Guard](dhcpv6-guard.md), **SISF device tracking uses policies that must be attached to a target before they take effect**. The workflow is:

1. Choose the built-in `default` policy or create a named policy using `device-tracking policy <name>`.
2. Attach it **globally (box-wide)**, to an **interface**, or to a **VLAN**.
3. Verify the attachments with `show device-tracking policies`, then inspect the learned addresses using `show device-tracking database`.

The same `device-tracking` command family is used at all three attachment scopes. **Where you enter the command determines what it applies to.**

### Is Device Tracking Enabled by Default?

**No.** SISF device tracking is not globally enabled by default. Creating a named policy does not activate it until the policy is attached somewhere.

There are three ways device tracking can become active:

| Method | Configuration |
| --- | --- |
| **Built-in default policy** | Enter `device-tracking` in global, interface, or VLAN configuration mode. |
| **Named policy** | Define a policy with `device-tracking policy <name>`, then attach it with `device-tracking attach-policy <name>` at the desired scope. |
| **Programmatic activation** | Another switch feature, such as DHCP snooping or an identity service, creates and attaches a device-tracking policy. |

A manually attached global policy can enable tracking across the entire switch. An interface may also be tracked because of a VLAN or programmatic policy even when there is no `device-tracking` command under the interface itself.

### The Built-In `default` Policy

IOS XE includes an unmodifiable device-tracking policy named `default`. If you enter `device-tracking` without an `attach-policy` argument, the switch attaches this policy to the current configuration scope.

#### Default Policy Settings

| Parameter | Built-in default |
| --- | --- |
| `security-level` | `guard` |
| `device-role` | `node` |
| ARP learning (IPv4) | Enabled |
| DHCPv4 learning | Enabled |
| Neighbor Discovery learning (IPv6) | Enabled |
| DHCPv6 learning | Enabled |
| Active reachability polling | Disabled unless enabled globally |
| `trusted-port` | Not configured |
| Data/destination/prefix gleaning | Not additionally enabled |

Therefore, the default policy can learn **both IPv4 and IPv6** host addresses, and uses `guard` to validate address ownership.

View the default policy:

```
SW1#show device-tracking policy default
Device-tracking policy default configuration:&#x20;
  security-level guard
  device-role node
  gleaning from Neighbor Discovery
  gleaning from DHCP6
  gleaning from ARP
  gleaning from DHCP4
  NOT gleaning from protocol unkn
Policy default is applied on the following targets:&#x20;
Target               Type  Policy               Feature        Target range
Box                  BOX   default              Device-tracking vlan all
```

### Option 1: Enable the Default Policy

#### Globally (Box-Level)

To enable device tracking across the entire switch:

```text
SW1(config)# device-tracking
```

This attaches the built-in `default` policy **to the box**, covering all VLANs.

Verify it:

```text
SW1# show device-tracking policies
Target               Type  Policy               Feature         Target range
Box                  BOX   default              Device-tracking vlan all
```

- `Box` is the entire switch.
- `BOX` is the attachment type.
- `default` is the attached policy name.
- `vlan all` indicates that the attachment covers all VLANs.

The following command removes this box-level attachment:

```text
SW1(config)# no device-tracking
```

This removes the **global** policy attachment. An independently configured interface, VLAN, or programmatic policy can still remain active.

#### On an Interface

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# device-tracking
```

The default policy applies to `Gi1/0/1`. Without a VLAN selector, it covers **all VLANs on the interface**, including VLANs carried by a trunk.

#### On a VLAN

```text
SW1(config)# vlan configuration 10
SW1(config-vlan-config)# device-tracking
```

This attaches the default policy to **VLAN 10 across the switch**, wherever that VLAN is carried.

#### Comparing the Scopes

| Where configured | Command | Attachment shown by `show device-tracking policies` |
| --- | --- | --- |
| Global configuration | `device-tracking` | `Box`, type `BOX`, `vlan all` |
| Interface configuration | `device-tracking` | Interface name, type `PORT`, `vlan all` |
| VLAN configuration | `device-tracking` | VLAN target, type `VLAN` |

These examples use exactly the same policy. Only its **attachment target** differs.

### Option 2: Create and Attach a Named Policy

Creating a policy is separate from attaching it, just as with RA Guard and DHCPv6 Guard.

**Step 1: Create the policy.**

```text
SW1(config)# device-tracking policy LAN10_HOSTS
SW1(config-device-tracking)# security-level guard
SW1(config-device-tracking)# device-role node
SW1(config-device-tracking)# tracking enable
SW1(config-device-tracking)# limit address-count 10
SW1(config-device-tracking)# exit
```

This policy validates address ownership, permits direct host learning, enables active reachability polling, and limits the number of IP bindings learned per port to ten. ARP, DHCPv4, ND, and DHCPv6 learning remain enabled by default.

**Step 2: Attach the policy at the desired scope.** You do not need to attach the policy in all three places.

#### Globally (Box-Level)

```text
SW1(config)# device-tracking attach-policy LAN10_HOSTS
```

This attaches the named policy to the **entire switch**, with a `BOX` target covering all VLANs. Unlike `device-tracking` alone, it uses your custom settings rather than the built-in defaults.

Verify with:

```text
SW1# show device-tracking policies
SW1# show device-tracking policy LAN10_HOSTS
```

#### On an Interface

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# device-tracking attach-policy LAN10_HOSTS
```

The named policy applies to all VLANs on this interface unless an interface VLAN selector is configured.

#### On a VLAN

```text
SW1(config)# vlan configuration 10
SW1(config-vlan-config)# device-tracking attach-policy LAN10_HOSTS
```

This applies the custom policy to VLAN 10 across switch interfaces.

To select more than one VLAN:

```text
SW1(config)# vlan configuration 10,20
SW1(config-vlan-config)# device-tracking attach-policy LAN10_HOSTS
```

#### On Selected VLANs of a Trunk

```text
SW1(config)# interface GigabitEthernet1/0/48
SW1(config-if)# device-tracking attach-policy SWITCH_UPLINK vlan 10,20
```

`SWITCH_UPLINK` must already be defined as a device-tracking policy.

| VLAN selector | Effect |
| --- | --- |
| Omitted or `vlan all` | All VLANs carried on the interface, including the native VLAN. |
| `vlan 10,20` | VLANs 10 and 20 only. |
| `vlan except 10,20` | All VLANs except 10 and 20. |
| `vlan add 30` | Add VLAN 30 to the current selection. |
| `vlan remove 20` | Remove VLAN 20 from the current selection. |
| `vlan none` | Select no VLANs. |

For several access ports, use `interface range`:

```text
SW1(config)# interface range GigabitEthernet1/0/1-24
SW1(config-if-range)# device-tracking attach-policy LAN10_HOSTS
```

Alternatively, reuse the configuration with an interface template:

```text
SW1(config)# template interface HOST_TRACKING
SW1(config-template)# device-tracking attach-policy LAN10_HOSTS
SW1(config-template)# exit
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# source template HOST_TRACKING
```

### Policy Precedence and Overlapping Attachments

A switch can have **box-level, VLAN-level, interface-level, and programmatic policies** at the same time. The global attachment provides a broad baseline; more specific attachments are useful for ports or VLANs that need different settings.

Important rules:

- **Interface policies take precedence over VLAN policies** for their shared interface/VLAN scope.
- **Manually attached policies take precedence over programmatically generated policies** when their settings compete.
- Certain address limits, especially IPv4-per-MAC and IPv6-per-MAC limits, are combined or handled specially rather than simply replaced.
- A `BOX` policy can coexist with more specific policies. Inspect the resulting attachments and effective policy settings when combining these scopes.

For example, configure a general discovery policy globally and a stricter policy on host-facing ports:

```text
SW1(config)# device-tracking policy DISCOVERY_ONLY
SW1(config-device-tracking)# security-level glean
SW1(config-device-tracking)# exit
SW1(config)# device-tracking attach-policy DISCOVERY_ONLY
!
SW1(config)# device-tracking policy ACCESS_GUARD
SW1(config-device-tracking)# security-level guard
SW1(config-device-tracking)# device-role node
SW1(config-device-tracking)# exit
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# device-tracking attach-policy ACCESS_GUARD
```

The `DISCOVERY_ONLY` policy provides box-wide coverage. `ACCESS_GUARD` is attached explicitly to `Gi1/0/1` for stricter validation. Verify the actual policy attachments and behavior rather than assuming that simply defining `ACCESS_GUARD` activates it.

### Option 3: Programmatically Enabled Device Tracking

Other switch features can activate SISF automatically, without an explicit manual policy attachment on every port. Examples include **DHCP snooping**, **802.1X/web authentication**, and other endpoint identity features.

For example, DHCP snooping can create a policy named `DT-PROGRAMMATIC`:

```text
SW1(config)# ip dhcp snooping
SW1(config)# ip dhcp snooping vlan 1
```

Inspect the result:

```text
SW1# show device-tracking policy DT-PROGRAMMATIC
Device-tracking policy DT-PROGRAMMATIC configuration:&#x20;
  security-level glean
  device-role node
  gleaning from Neighbor Discovery
  gleaning from DHCP6
  gleaning from ARP
  gleaning from DHCP4
  NOT gleaning from protocol unkn
  limit address-count for IPv4 per mac 1
  tracking (downlink only) enable
Policy DT-PROGRAMMATIC is applied on the following targets:&#x20;
Target               Type  Policy               Feature        Target range
vlan 1               VLAN  DT-PROGRAMMATIC      Device-tracking vlan all
```

A programmatic policy's settings are determined by its client feature and can differ from the built-in `default` policy. To remove a programmatically created policy, remove the configuration of the feature that installed it.

---

## Policy Parameters at a Glance

After a policy is defined and attached, its settings determine how SISF learns and validates hosts. The following sections explore these parameters in detail.

| Policy command | Function |
| --- | --- |
| `security-level glean` | Learn addresses without verifying ownership. |
| `security-level guard` | Validate address ownership and reject invalid claims. This is the default. |
| `security-level inspect` | Intermediate inspection level. |
| `device-role node` | Treat the connected device as a host; learn and validate directly connected bindings. This is the default. |
| `device-role router` | Identify a router-facing interface; it does not itself make the port trusted. |
| `device-role switch` | Identify a switch-facing interface and change how remote bindings and validation probes are handled. |
| `trusted-port` | Exempt traffic on the port from SISF guard checks and give trusted observations higher preference. |
| `protocol arp`, `protocol dhcp4` | Learn IPv4 addresses through ARP and DHCPv4. |
| `protocol ndp`, `protocol dhcp6` | Learn IPv6 addresses through ND and DHCPv6. |
| `tracking enable` / `tracking disable` | Enable or disable active reachability polling for bindings governed by the policy. |
| `limit address-count <count>` | Limit learned IP addresses on a port. |
| `data-glean`, `destination-glean`, `prefix-glean` | Optional additional learning and recovery methods. |

---

## Security Levels: Glean, Guard, and Inspect

The `security-level` parameter determines how SISF handles address information in packets entering the network.

| Security level | Binding validation | Behavior |
| --- | --- | --- |
| `glean` | None | Learns information without verifying that the sender owns the claimed IP address. |
| `inspect` | Intermediate | Available in the CLI, but not recommended by Cisco for normal designs. |
| `guard` | Yes | Checks for conflicting address ownership and can reject packets and bindings that fail validation. |

### Glean

```text
SW1(config)# device-tracking policy DISCOVERY_ONLY
SW1(config-device-tracking)# security-level glean
```

With `glean`, SISF extracts available identity and address information and attempts to add it to the binding table. **It does not validate address ownership** before learning. Applicable address-count limits still restrict the number of entries.

This is useful when the goal is host discovery for services such as identity or authentication systems, rather than SISF-based verification of address ownership.

### Guard

```text
SW1(config)# device-tracking policy ACCESS_GUARD
SW1(config-device-tracking)# security-level guard
```

`guard` is the default security level. When SISF receives a relevant packet, it checks the address information against existing bindings:

1. **No conflicting binding exists:** SISF can create a new binding, subject to the policy and resource limits.
2. **Same address, MAC, VLAN, and attachment:** SISF can refresh the existing binding.
3. **Same address, different MAC or attachment:** SISF performs additional validation. It can probe an existing host to determine whether a legitimate move occurred or another device is falsely claiming the address.
4. **Validation fails:** The conflicting binding is not accepted. The corresponding ARP or ND packet can be dropped.

Example:

```text
Existing binding:
192.0.2.10 -> 0011.2233.4455, VLAN 10, Gi1/0/1
New ARP packet received:
192.0.2.10 -> 0011.2233.9999, VLAN 10, Gi1/0/2
```

With `glean`, SISF does not verify ownership before processing the new claim. With `guard`, the new claim must pass validation before it can replace the existing information.

The same principle applies to conflicting IPv6 information carried by ND. More detailed ND behavior is covered in [ND Inspection and Snooping](nd-inspection.md).

### Inspect

```text
SW1(config-device-tracking)# security-level inspect
```

`inspect` is an intermediate mode between `glean` and `guard`. Cisco recommends using `glean` for discovery-only requirements or `guard` when binding integrity matters, rather than selecting `inspect` for a new configuration.

> Cisco doesn't give many details about how `inspect` works. Best to avoid it unless you test it thoroughly!

### Binding Action Versus Packet Action

An important distinction is that **learning and forwarding are separate decisions**.

SISF can take actions on the **binding database**, such as adding, updating, or ignoring an entry. It can also take actions on a **packet**, such as forwarding or dropping it.

For example:

- Under `glean`, reaching an address-count limit can prevent SISF from adding a binding. That does **not** automatically mean all subsequent ordinary IP data traffic is blocked.
- Under `guard`, an invalid ARP or ND ownership claim can cause both binding rejection and packet dropping.
- An entry existing in the SISF database does not, by itself, mean an ordinary IPv4 or IPv6 data packet has passed an IP Source Guard check.

This distinction is important when troubleshooting a host that is communicating successfully but does not appear in the binding table.

---

## Device Roles and Trusted Ports

The SISF device role describes the type of device connected to a switch port. It affects how SISF processes Neighbor Discovery messages, learns bindings, and performs validation. Device Tracking supports three roles: `node` (default), `router`, and `switch`.

Cisco documents the intended uses of these settings, but does not fully explain every packet-processing and binding-validation decision. The observations below come from **tests using IOL IOS XE** and should not be treated as an exhaustive description of the internal implementation.

### `device-role node`

```text
SW1(config-device-tracking)# device-role node
```

Use `node` on **host-facing access ports**. SISF learns host IP/MAC bindings from received ARP, DHCP, and Neighbor Discovery messages and can validate them using active probes.

In the tests, a `NODE` policy with `security-level guard` **dropped incoming Router Advertisements (RAs)** but allowed Router Solicitations (RSs) and legitimate Neighbor Advertisements (NAs). When `NODE` was applied to an inter-switch link, SISF treated a remote host as directly connected and successfully used a unicast Neighbor Solicitation (NS) to validate its binding as `REACHABLE`.

### `device-role router`

```text
SW1(config-device-tracking)# device-role router
```

Use `router` on **router-facing ports**. The role identifies the attached device type but **does not automatically trust RAs**.

In the tests, `device-role router` with `security-level guard` and **without** `trusted-port` still dropped R2's RAs. Adding `trusted-port` allowed those RAs. Thus, selecting the `router` role alone was insufficient to permit RAs in this lab.

The `device-role router` command in a Device Tracking policy is distinct from the same command in an RA Guard policy. An independently configured RA Guard policy must permit legitimate RAs as well.

### `device-role switch`

```text
SW1(config-device-tracking)# device-role switch
```

Use `switch` on **inter-switch links**. Cisco describes this role as a way to avoid unnecessary remote-host bindings and validation in multi-switch designs, particularly when paired with `trusted-port`.

The tests revealed that **`device-role switch` alone did not completely prevent remote bindings from being created**. SISF accepted remote RS and NA messages and created entries for hosts learned over the inter-switch link. It classified those entries as remote and handled their validation differently:

- With `device-role switch`, SISF logged `Refraining from sending ... DAD NS for a remote entry` for multicast and unicast probes. In one test, the remote binding remained `INCOMPLETE` and was later removed.
- With `device-role node` on the same link, SISF sent a unicast validation NS, received an NA, and moved the binding to `REACHABLE`.
- With `device-role switch` and `security-level guard` on an **untrusted** inter-switch port, an incoming RA was **dropped**.

The `switch` role tells SISF that a neighboring switch, rather than an individual host, is connected. The physical interface does not need to operate as a VLAN trunk for this classification to apply. Remote entries can still appear in the binding table; the role changes how SISF treats them.

### `trusted-port`

```text
SW1(config-device-tracking)# trusted-port
```

`trusted-port` exempts received traffic from the SISF `guard` checks that otherwise reject certain control packets. It also gives observations on trusted ports a higher binding preference. This is **independent of `device-role`**.

In the RA test, the interface still reported `Sec level is Guard`, but also reported `Trusted port`, and the RA received `Feature Device-tracking rc: OK`. The security level remained configured; the port's trust status changed the decision.

The binding table also identified R2's address as trusted with `prlvl 0011`, the combination of `0001` (MAC and link-layer address match) and `0010` (originated on a trusted access port). By comparison, a directly connected untrusted host had `0005`, and a remote address learned on the inter-switch link had `0003`.

`trusted-port` in a Device Tracking policy is separate from `ip dhcp snooping trust` and trust or role settings in RA Guard and DHCPv6 Guard. Each feature has its own policy.

### RA Handling Observed in IOL IOS XE

All of these tests used `security-level guard` in the Device Tracking policy, with no separate RA Guard policy enforcing the observed drop:

| Device Tracking policy settings | Incoming RA | Debug result |
| --- | --- | --- |
| `device-role node`, untrusted | Blocked | `DROP` |
| `device-role router`, untrusted | Blocked | `DROP` |
| `device-role switch`, untrusted | Blocked | `DROP` |
| `trusted-port` + any `device-role`| Permitted | `OK` |

For example, when SW2 received an RA from R1 over its inter-switch interface using the `SWITCH` policy, the debug showed:

```text
Et0/1 vlan 1 Parse msg ND_ROUTER_ADVERT
Found matching policy SWITCH for feature Device-tracking on Et0/1
Et0/1 vlan 1 Sec level is Guard
Et0/1 vlan 1 Unallowed RA/Redir: default action is delete entry
Et0/1 vlan 1 Unauthorized packet
Et0/1 vlan 1 Feature Device-tracking rc: DROP
```

**The important distinction:** `device-role` describes the neighboring device and affects learning and validation, while `trusted-port` determines whether a port is exempt from the observed SISF guard checks. SISF Device Tracking itself can drop RAs, even without RA Guard.

### Multi-Switch Configuration

```text
             PC1                                PC2
              |                                  |
           Gi1/0/1                            Gi1/0/1
              |                                  |
           +------+          VLAN 10          +------+
           | SW1  |===========================| SW2  |
           +------+       Inter-switch link   +------+
                         Gi1/0/48 both ends
```

Create a policy for the inter-switch ports:

```text
SW1(config)# device-tracking policy SWITCH_UPLINK
SW1(config-device-tracking)# device-role switch
SW1(config-device-tracking)# trusted-port
SW1(config-device-tracking)# exit
!
SW1(config)# interface GigabitEthernet1/0/48
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10
SW1(config-if)# device-tracking attach-policy SWITCH_UPLINK
```

Use an equivalent policy on SW2. The host-facing ports use `device-role node` without `trusted-port`.

This design is intended to let each access switch concentrate on tracking its directly connected hosts, reduce duplicated validation activity, and permit legitimate traffic from a trusted neighboring switch. The combination of `device-role switch` and `trusted-port` is important; the lab showed that `switch` alone can still create remote bindings and block incoming RAs.

Do not automatically trust an uplink toward an unprotected switch. If the downstream switch is not performing the expected validation, centralized tracking on a distribution switch may be more appropriate.

---

## Selecting the Learning Protocols

Use the `protocol` parameter to control which packet types supply information to device tracking.

The supported primary protocol keywords are:

```text
SW1(config-device-tracking)# protocol arp
SW1(config-device-tracking)# protocol dhcp4
SW1(config-device-tracking)# protocol ndp
SW1(config-device-tracking)# protocol dhcp6
```

These enable the corresponding learning sources. The four protocols are ordinarily enabled by default, so configuring all four explicitly is not required for a dual-stack access port.

### Disable IPv6 Learning

To configure a policy that learns **IPv4 only**:

```text
SW1(config)# device-tracking policy IPV4_ONLY
SW1(config-device-tracking)# no protocol ndp
SW1(config-device-tracking)# no protocol dhcp6
```

The policy can continue to learn from ARP and DHCPv4.

### Disable IPv4 Learning

To configure a policy that learns **IPv6 only**:

```text
SW1(config)# device-tracking policy IPV6_ONLY
SW1(config-device-tracking)# no protocol arp
SW1(config-device-tracking)# no protocol dhcp4
```

The policy can continue to learn from ND and DHCPv6.

**Disabling protocol gleaning affects SISF learning and related verification.** It is not a general ACL that drops all traffic of the disabled protocol.

Verify the actual enabled protocols:

```text
SW1# show device-tracking policy IPV4_ONLY
SW1# show device-tracking policy IPV6_ONLY
```

Output identifies sources as `gleaning from ...` or `NOT gleaning from ...`.

> NOTE: ADDITIONAL TESTING/VERIFICATION NEEDED FOR THE FOLLOWING SECTIONS

### Additional Learning and Recovery Options

SISF also offers optional methods for detecting missing or incomplete host information:

| Parameter | Purpose |
| --- | --- |
| `data-glean` | Detect missing bindings based on the **source IP address** of data packets and optionally initiate binding recovery. |
| `destination-glean` | Detect missing bindings based on the **destination IP address** of data packets and optionally initiate binding recovery. |
| `prefix-glean` | Learn relevant prefix information from IPv6 RA and DHCPv6 Prefix Delegation traffic. |

**Binding recovery** is the process of reconstructing a missing binding for a device that already has a valid IP address. Rather than relying solely on ARP, DHCP, or ND messages to discover addresses, SISF can use ordinary IP data traffic to trigger recovery.

For example, if a switch receives a packet from `2001:DB8::10` but has no binding for that address, `data-glean` can initiate a recovery procedure.

Both `data-glean` and `destination-glean` support two operating modes:

- **`log-only`:** Generate a syslog message when relevant data traffic is observed, without initiating binding recovery.
- **`recovery`:** Attempt to reconstruct the missing binding using a supported protocol:
  - **NDP:** Use Neighbor Discovery probes to obtain address ownership information from the host.
  - **DHCP:** Query the DHCPv6 server using DHCPv6 Leasequery to retrieve the host's address assignment information.

Configuration examples:

- `data-glean recovery ndp`
- `data-glean recovery dhcp`
- `destination-glean recovery dhcp`
- `data-glean log-only`
- `destination-glean log-only`

These mechanisms are particularly useful with **IPv6 Source Guard**, which can block traffic from legitimate hosts if their bindings are missing.

**Observing an IP address in a data packet does not automatically validate its ownership.** The configured recovery procedure must obtain the information needed to establish a valid binding.

---

## Active Reachability Tracking and Polling

By default, a device-tracking policy can learn hosts passively **without sending periodic reachability probes**. There are two scopes for active polling:

- **Global:** `SW1(config)# device-tracking tracking` enables the default polling behavior for policies that inherit the global setting.
- **Per policy:** `tracking enable` forces polling on for the attached policy, and `tracking disable` forces it off. A policy-level setting can override the global default in either direction.

The command `device-tracking tracking` in global configuration mode controls **polling behavior**. It is distinct from `device-tracking` (without `tracking`), which **does** attach the default device-tracking policy globally. Do not confuse a global polling setting with a `BOX` policy attachment.

To explicitly enable polling on a custom policy, configure:

```text
SW1(config)# device-tracking policy LAN10_HOSTS
SW1(config-device-tracking)# tracking enable
```

To disable polling while retaining passive learning:

```text
SW1(config-device-tracking)# tracking disable
```

With tracking enabled, SISF can probe a host when its reachable lifetime expires or when it needs to verify a conflicting ownership claim.

### IPv4 Versus IPv6 Probes

| Address family | Probe used by SISF | Expected response |
| --- | --- | --- |
| **IPv4** | ARP Request directed at the tracked IPv4 address | ARP Reply from the host |
| **IPv6** | Neighbor Solicitation directed at the tracked IPv6 address | Neighbor Advertisement from the host |

For a dynamically learned binding with the default reachable lifetime, a typical polling sequence is:

1. A host is learned and its binding becomes `REACHABLE`.
2. No qualifying control message refreshes it during the **300-second reachable lifetime**.
3. With active tracking enabled, the entry enters `VERIFY` and SISF sends reachability probes.
4. A valid response returns the entry to `REACHABLE`.
5. If three attempts fail, the entry becomes `STALE`. At the end of the stale lifetime, SISF can perform a final check before deleting the entry.

DHCP-originated bindings have lease-based lifetime handling, as described in [IPv6 Binding Table](binding-table.md).

### Policy-Level Reachable Lifetime

Configure a longer reachable period together with polling:

```text
SW1(config)# device-tracking policy LAN10_HOSTS
SW1(config-device-tracking)# tracking enable reachable-lifetime 600
```

This enables active tracking and configures the reachable lifetime to **600 seconds** for entries controlled by that policy.

The `reachable-lifetime` is **not** the interval at which the switch continuously sends ARP or NS messages to all hosts. It is the period for which a binding can remain reachable after its last qualifying refresh event before verification becomes relevant.

### Global Polling Retry Interval

Configure the retry interval globally:

```text
SW1(config)# device-tracking tracking retry-interval 5
```

This sets the interval between consecutive polling retries to **5 seconds**. The global retry interval applies to both IPv4 and IPv6 polling; the active tracking policy determines whether an interface's hosts are polled.

**`device-tracking tracking` controls polling; `device-tracking` controls policy attachment.** Entering `device-tracking` in global configuration mode attaches the `default` policy to the box, while entering `device-tracking tracking` alone changes only the global polling default.

---

## IPv4 Probe Source Addresses and Duplicate-Address Problems

This is an important IPv4-specific device-tracking issue.

When SISF probes an IPv4 host, it generates an **ARP Request**. If the switch has no IPv4 SVI for the host's subnet, the ARP probe can use `0.0.0.0` as its source IPv4 address.

For example:

```text
ARP Request generated by SW1 to verify PC1
Sender IP:  0.0.0.0
Target IP:  192.0.2.10
Ethernet destination MAC: 0011.2233.4455
```

Some hosts can interpret these probes as part of IPv4 Address Conflict Detection and report a duplicate-address warning. The relevant issue is the **source address used by SISF's own probe**, not an actual duplicate address assigned to the host.

### Source Address Selection

When `device-tracking tracking auto-source` is configured, SISF chooses an IPv4 probe source in this order:

1. Use the IPv4 address of the VLAN's SVI, if available.
2. Otherwise, look for a suitable IP/MAC binding in the same subnet.
3. If a fallback is configured, calculate a source address from the destination subnet and fallback host bits.
4. Without a usable source, the probe can use `0.0.0.0`.

Configure automatic source selection:

```text
SW1(config)# device-tracking tracking auto-source
```

### Fallback Source Address

On a Layer 2 switch without an SVI, configure a reserved host address as a fallback:

```text
SW1(config)# device-tracking tracking auto-source fallback 0.0.0.253 255.255.255.0
```

Suppose the destination is `192.0.2.10`. SISF calculates the fallback source by combining the destination subnet with the configured host bits:

```text
Target:    192.0.2.10
Mask:      255.255.255.0
Host bits: 0.0.0.253
Result:    192.0.2.253
```

Reserve `192.0.2.253` so no real device uses that address. The same host-bit pattern is used to derive a corresponding source address for other matching subnets.

With the optional `override` keyword:

```text
SW1(config)# device-tracking tracking auto-source fallback 0.0.0.253 255.255.255.0 override
```

SISF still prefers an existing VLAN SVI, but otherwise **skips the search for an existing host binding as a source** and uses the computed fallback. Without `override`, a suitable existing binding is preferred over the fallback.

### IPv6 Probe Source Addresses

IPv6 tracking uses **Neighbor Solicitations**, not ARP. When the switch has an SVI with an IPv6 link-local address, SISF can use that link-local address as the NS probe source. Without such an SVI address, SISF may use the unspecified IPv6 address (`::`).

For a switch already using an SVI in the client VLAN, enable IPv6 to provide a link-local source:

```text
SW1(config)# interface Vlan10
SW1(config-if)# ipv6 enable
```

The `device-tracking tracking auto-source fallback` command concerns **IPv4 ARP probes**, not IPv6 NS source selection.

---

## Address-Count Limits

Device tracking can restrict how many IP addresses it learns. This is especially relevant for IPv6 hosts with several addresses and for dual-stack ports where IPv4 and IPv6 entries share the same resource limits.

### Per-Port Policy Limit

```text
SW1(config)# device-tracking policy LAN10_HOSTS
SW1(config-device-tracking)# limit address-count 10
```

This limits learned host **IP addresses per port** under the policy. It is **not** a limit of ten MAC addresses and does not mean ten IPv4 plus ten IPv6 addresses.

For example, on a single access port:

| Address type | Bindings |
| --- | ---: |
| IPv4 addresses | 1 |
| IPv6 link-local addresses | 1 |
| IPv6 global stable addresses | 1 |
| IPv6 temporary addresses | 4 |
| **Total** | **7** |

With `limit address-count 10`, only **three additional IP bindings** remain available under the per-port limit.

The limit controls admission of entries to the binding table. Reaching it does not automatically mean that every ordinary data packet is dropped, although `guard` validation can reject relevant control packets when a binding cannot be accepted.

### Other Limits and Policy Interactions

SISF also has global binding limits configured with the `device-tracking binding max-entries` command family. Policy output may show separate **IPv4-per-MAC** and **IPv6-per-MAC** limits provided by a built-in or programmatic policy.

A new binding must satisfy **all applicable limits**. A limit reached at either the global or policy level can prevent further learning.

For global limit configuration and binding-table capacity, see [IPv6 Binding Table](binding-table.md).

---

## Dual-Stack Device Tracking Lab

Use the following topology to practice ARP and ND learning, policy attachment, active tracking, and address ownership validation. The gateway is configured separately; SW1 functions as a Layer 2 access switch.

```text
                    R1 (Dual-Stack Gateway)
                     192.0.2.1/24
                     2001:DB8:10::1/64
                             |
                           Gi1/0/24
                             |
                       +-----------+
                       |    SW1    |
                       |  VLAN 10  |
                       +-----------+
                         /       \
                    Gi1/0/1   Gi1/0/2
                        |         |
                       PC1       PC2
PC1: 192.0.2.10/24            PC2: 192.0.2.20/24
     2001:DB8:10::10/64            2001:DB8:10::20/64
```

### Step 1: Configure the Dual-Stack Gateway

```text
R1(config)# ipv6 unicast-routing
R1(config)# interface GigabitEthernet1
R1(config-if)# ip address 192.0.2.1 255.255.255.0
R1(config-if)# ipv6 address 2001:DB8:10::1/64
R1(config-if)# no shutdown
```

PC1 and PC2 can use the stated static IPv4/IPv6 addresses. They do **not** need a DHCP server for the ARP/ND learning portions of this lab.

### Step 2: Create a Host Policy

```text
SW1(config)# vlan 10
!
SW1(config)# device-tracking policy LAN10_HOSTS
SW1(config-device-tracking)# security-level guard
SW1(config-device-tracking)# device-role node
SW1(config-device-tracking)# tracking enable reachable-lifetime 300
SW1(config-device-tracking)# limit address-count 10
SW1(config-device-tracking)# exit
```

The standard ARP, DHCPv4, ND, and DHCPv6 learning sources remain enabled by default.

### Step 3: Attach the Policy to Access Ports

```text
SW1(config)# interface range GigabitEthernet1/0/1-2
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 10
SW1(config-if-range)# device-tracking attach-policy LAN10_HOSTS
!
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
```

In this lab, the policy is attached only to the two host-facing ports. The R1-facing port does not need the same host-learning policy.

### Step 4: Generate Learning Events

From PC1, generate IPv4 and IPv6 neighbor resolution:

```text
PC1> ping 192.0.2.1
PC1> ping 2001:DB8:10::1
```

The exact host CLI depends on the endpoint. If entries do not appear, generate fresh ARP or ND activity rather than repeating a `ping` that uses an already populated neighbor cache.

### Step 5: Verify Policies and Bindings

```text
SW1# show device-tracking policy LAN10_HOSTS
SW1# show device-tracking policies
SW1# show device-tracking database
SW1# show device-tracking database interface GigabitEthernet1/0/1
SW1# show device-tracking database vlanid 10
```

Expected observations:

- PC1's IPv4 address can be learned from ARP, and its IPv6 addresses from ND.
- PC2 can have its own entries on `Gi1/0/2`.
- One MAC can appear in multiple entries because the endpoint uses both address families.
- `show device-tracking policy LAN10_HOSTS` confirms that **tracking is enabled** and identifies the applied targets.
- With no recent control packets, an entry can transition from `REACHABLE` to `VERIFY` and be actively probed.

### Step 6: Compare Passive and Active Tracking

Disable active probing without disabling learning:

```text
SW1(config)# device-tracking policy LAN10_HOSTS
SW1(config-device-tracking)# tracking disable
```

Generate new ARP and ND traffic and confirm that bindings can still be learned or refreshed. Then re-enable active tracking:

```text
SW1(config-device-tracking)# tracking enable reachable-lifetime 300
```

Observe the difference when a host becomes silent after learning.

### Step 7: Compare Glean and Guard

Create a discovery-only policy:

```text
SW1(config)# device-tracking policy DISCOVERY_ONLY
SW1(config-device-tracking)# security-level glean
SW1(config-device-tracking)# device-role node
SW1(config-device-tracking)# exit
```

Apply the policy to an appropriate isolated test port, then compare how SISF handles a conflicting IP/MAC association under `glean` and `guard`. **Use a lab-only address conflict** rather than duplicating an address on a production VLAN.

The important observation is whether SISF accepts the new binding or performs validation and rejects the claim. The details of individual binding transitions are covered in [IPv6 Binding Table](binding-table.md).

---

## Key Points

- **SISF device tracking is dual-stack**, even though the CCIE topic is named IPv6 Device Tracking.
- The major learning sources are **ARP and DHCPv4 for IPv4**, and **ND and DHCPv6 for IPv6**.
- **Passive learning and active polling are separate.** The binding table can be populated even when `tracking enable` is not configured.
- `security-level glean` learns without validating ownership. `security-level guard` validates address ownership and can reject invalid control packets and bindings, including the RAs observed in the lab.
- `device-role node` is for host-facing ports; `device-role router` identifies router-facing ports; `device-role switch` changes remote-host binding and validation behavior on inter-switch links.
- In the IOL IOS XE tests, `guard` dropped RAs on untrusted `node`, `router`, and `switch` ports without RA Guard; `trusted-port` permitted the tested RA. On trusted inter-switch links, `trusted-port` is commonly paired with `device-role switch`.
- `no protocol ndp` and `no protocol dhcp6` disable IPv6 learning; `no protocol arp` and `no protocol dhcp4` disable IPv4 learning within a policy.
- IPv4 reachability probes use **ARP**; IPv6 probes use **Neighbor Solicitations**. IPv4 `auto-source` affects the ARP probe source address.
- `limit address-count` restricts **IP bindings per port**, counting IPv4 and IPv6 entries together.
- A policy must be attached **globally (`BOX`), to an interface, or to a VLAN** to take effect. Other switch features can also activate tracking through programmatic policies.
- `show device-tracking policies` identifies `BOX`, `PORT`, and `VLAN` attachments; `show device-tracking policy` and `show device-tracking database` display policy details and learned bindings.

---

## References

- [Cisco: Catalyst 9300 Security Command Reference, IOS XE 17.11 (device-tracking commands)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-11/command_reference/b_1711_9300_cr/security_commands.html)
- [Cisco: FHS and SISF Configuration Guide, SISF (IOS XE 17)](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/sec-crypto/fhs-sisf/fhs-and-sisf-configuration-guide/switch-integrated-security-features.html)
- [Cisco: Troubleshoot SISF on Catalyst 9000 Series Switches](https://www.cisco.com/c/en/us/support/docs/switches/catalyst-9300-series-switches/221562-troubleshoot-sisf-on-catalyst-9000-serie.html)
- [Cisco: Catalyst 9000 Family SISF White Paper](https://www.cisco.com/c/en/us/products/collateral/switches/catalyst-9000/white-paper-c11-743346.html)
- [Cisco: Catalyst 9200 Security Command Reference, IOS XE 17.15](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9200/software/release/17-15/command_reference/b_1715_9200_cr/security_commands.html)
