# 4.2.c (iii) IPv6 Binding Table

The **SISF (Switch Integrated Security Features) binding table** records the **IPv4 and IPv6 addresses** of hosts, their MAC addresses, and where they are connected to a switch. It is maintained by **SISF device tracking**, which learns host information from ARP, DHCPv4, IPv6 Neighbor Discovery (ND), DHCPv6, and other supported sources.

Although the CCIE blueprint topic is called **IPv6 Binding Table**, the modern SISF binding table is a **unified IPv4/IPv6 database**. The same architecture tracks dual-stack endpoints, including hosts that use DHCPv4 for IPv4 and SLAAC or DHCPv6 for IPv6.

SISF combines host identity, point of attachment, learning source, and reachability information. **A binding is information about a host, not by itself a rule that filters all traffic from that host.** Enforcement depends on the security features and policies using the table.

This page covers the **binding table itself**: its contents, IPv4/IPv6 learning sources, states, lifetimes, static bindings, limits, and verification. Policy configuration and enforcement are covered in [IPv6 Device Tracking](device-tracking.md), [ND Inspection and Snooping](nd-inspection.md), and [IPv6 Source Guard](source-guard.md).

---

## Binding Table Fundamentals

Each binding associates a network-layer address with a MAC address, VLAN, and switch interface.

Consider a dual-stack network:

```text
                       R1 (Gateway)
                            |
                        Gi1/0/24
                            |
                      +-----------+
                      |    SW1    |
                      |  VLAN 10  |
                      +-----------+
                       /         \
                   Gi1/0/1     Gi1/0/2
                      |           |
                     PC1         PC2

PC1: MAC 0011.2233.4455       PC2: MAC 0011.2233.4466
     IPv4 192.0.2.10              IPv4 192.0.2.20
     IPv6 2001:DB8:10::10         IPv6 2001:DB8:10::20
```

If SW1 learns all four addresses, its binding information is conceptually:

| IP address | MAC address | Interface | VLAN | State |
| --- | --- | --- | --- | --- |
| `192.0.2.10` | `0011.2233.4455` | `Gi1/0/1` | 10 | `REACHABLE` |
| `2001:DB8:10::10` | `0011.2233.4455` | `Gi1/0/1` | 10 | `REACHABLE` |
| `192.0.2.20` | `0011.2233.4466` | `Gi1/0/2` | 10 | `REACHABLE` |
| `2001:DB8:10::20` | `0011.2233.4466` | `Gi1/0/2` | 10 | `REACHABLE` |

A binding can also include:

- **Source:** Whether the entry was learned through ARP, DHCPv4, ND, DHCPv6, static configuration, or another mechanism.
- **Age:** Time since the relevant learning or refresh event.
- **Time left:** Remaining lifetime in the current state, or DHCP lease-related time.
- **Preference level (`prlvl`):** Flags describing characteristics such as where the binding originated, whether it was DHCP-assigned, and whether the origin is trusted.
- **Policy information:** Tracking policy and related attributes, visible in detailed output.

**One MAC can have multiple bindings.** A dual-stack PC might have an IPv4 address, an IPv6 link-local address, one stable IPv6 global address, and several temporary IPv6 addresses. These are separate IP bindings associated with the same MAC and port.

### Binding Table Versus Other Tables

| Table | Primary purpose |
| --- | --- |
| **SISF binding table** | Records IPv4/IPv6 host IP, MAC, VLAN, port, source, and reachability state. |
| **MAC address table** | Maps Ethernet MAC addresses to switch ports for Layer 2 forwarding. |
| **ARP table** | Maps IPv4 next-hop addresses to MAC addresses for Layer 3 forwarding. |
| **IPv6 neighbor table** | Maps IPv6 next-hop addresses to MAC addresses for IPv6 forwarding. |
| **IPv4 DHCP snooping binding database** | Records DHCPv4 client assignments learned through DHCP snooping. |

The SISF table is **not the same as the switch's ARP or IPv6 neighbor table**. A Layer 2 access switch can learn bindings for hosts in a VLAN even when it is not their default gateway and has no SVI for the subnet.

SISF is also not a replacement for the DHCP snooping database. DHCP snooping focuses on DHCP assignments; SISF can also learn **statically addressed IPv4 hosts via ARP** and **SLAAC IPv6 hosts via ND**. An IPv4 DHCP snooping feature can programmatically activate SISF, but the databases still serve different functions.

---

## How SISF Learns Bindings

**SISF device tracking must be enabled on the relevant interface or VLAN**, either by an attached device-tracking policy or through another feature that enables it programmatically. Defining a policy without attaching it does not start learning on every switch port.

The principal learning protocols are:

| Protocol | Address family | Information learned |
| --- | --- | --- |
| **ARP** | IPv4 | IPv4 address and MAC address observed in ARP messages. |
| **DHCPv4** | IPv4 | Client IPv4 address, MAC address, and DHCP assignment information. |
| **Neighbor Discovery (ND)** | IPv6 | IPv6 address and MAC association observed in ND messages. |
| **DHCPv6** | IPv6 | IPv6 assignments and delegated prefixes in DHCPv6 exchanges. |

### ARP (IPv4)

SISF can learn from **ARP Requests and ARP Replies**. ARP carries the sender's IPv4 address and sender MAC address, giving the switch information it can associate with the ingress port and VLAN.

For example, PC1 sends an ARP Request:

```text
ARP Request received on SW1 Gi1/0/1, VLAN 10
Sender IP:  192.0.2.10
Sender MAC: 0011.2233.4455
Target IP:  192.0.2.1
```

SISF can create or refresh the binding:

```text
192.0.2.10 -> 0011.2233.4455, VLAN 10, Gi1/0/1
```

The entry's source code is `ARP`.

Important behavior:

- **Static IPv4 addressing does not prevent learning.** A host with a manually configured IPv4 address can still be learned through ARP.
- A new ARP message can refresh a binding and help SISF detect that a host has changed ports.
- Learning from an ARP message does not, by itself, prove the sender owns the address. The device-tracking security level determines whether ownership is validated.
- In `security-level guard`, SISF can reject invalid or conflicting ARP information rather than simply overwriting an existing binding.

SISF also throttles ARP processing: Cisco documents a maximum of **50 ARP packets from the same source IP within five seconds**. Under `security-level guard`, ARP packets exceeding this threshold are dropped by SISF.

### DHCPv4

SISF can learn IPv4 bindings from **DHCPv4 negotiation**, including client Request and server ACK-related exchanges. This provides the MAC/IP association for clients that obtain addresses from DHCP.

Example:

```text
PC2 MAC:       0011.2233.4466
DHCPv4 lease:  192.0.2.20
SW1 port:      Gi1/0/2
VLAN:          10
```

SISF can record a DHCP-originated binding with source code `DH4` and associate it with the DHCP lease lifetime.

DHCP learning and ARP learning can describe the same IPv4 host. **Do not assume that a DHCP-configured host must always display `DH4`.** The source shown depends on the events processed and the SISF feature providing the information; an IPv4 host can appear as an `ARP` entry.

The SISF binding table is not required just to perform ordinary IPv4 DHCP snooping. See [DHCP Snooping and DHCP Option 82](../switch-security/dhcp-snooping/overview.md) for that feature's separate trust model and binding database.

### Neighbor Discovery (IPv6)

SISF can learn from **ICMPv6 Neighbor Discovery**, including information associated with Neighbor Solicitations (NS) and Neighbor Advertisements (NA).

For a SLAAC host, the general sequence is:

1. The host configures an IPv6 link-local address and may form global addresses from Router Advertisement prefixes.
2. It exchanges ND messages for Duplicate Address Detection (DAD), address resolution, and neighbor reachability.
3. SISF examines applicable ND packets and learns address/MAC/port information according to the tracking policy.
4. After successful validation, an entry can enter `REACHABLE`.

**A Router Advertisement containing a prefix does not create a host binding for every address in that prefix.** SISF must learn the identity and attachment of an actual endpoint.

A single host could have these bindings:

| IPv6 address | MAC address | Interface |
| --- | --- | --- |
| `FE80::211:22FF:FE33:4455` | `0011.2233.4455` | `Gi1/0/1` |
| `2001:DB8:10::10` | `0011.2233.4455` | `Gi1/0/1` |
| `2001:DB8:10::ABCD` | `0011.2233.4455` | `Gi1/0/1` |

This is normal for a host using a link-local address, a stable global address, and a temporary privacy address.

**DAD distinction:** A DAD NS has the unspecified IPv6 source address (`::`). The source address is not evidence that the host owns `::`. The tentative address and subsequent ND events must be considered when establishing a binding.

### DHCPv6

SISF can learn IPv6 assignments from **DHCPv6** messages, including information carried in:

- **IA_NA (Identity Association for Non-temporary Addresses):** IPv6 addresses assigned to the client.
- **IA_PD (Identity Association for Prefix Delegation):** IPv6 prefixes delegated to a requesting device, such as a downstream router.

A device may obtain multiple IPv6 addresses or a delegated prefix, so SISF must not assume that there is only one address per MAC. The source code for a DHCPv6-learned binding is `DH6`.

**DHCPv6-originated bindings use the DHCP lease lifetime**, rather than simply the ordinary SISF stale/down timers. A DHCP-originated entry can disappear when its lease expires even without additional reachability probes.

DHCPv6 Guard and SISF learning are separate functions. DHCPv6 Guard can block an unauthorized DHCPv6 server without first consulting a binding. See [DHCPv6 Guard](dhcpv6-guard.md).

### Learning an Address Does Not Require IP Connectivity

The switch can learn from an **ARP or ND control packet** even if the host has not yet successfully exchanged application traffic. Conversely, an IPv4 or IPv6 `ping` does **not guarantee** a new SISF entry: if the communicating devices already have the relevant ARP or neighbor-cache information, no new ARP/ND packet may be generated for SISF to observe.

---

## Binding Entry Types and Source Codes

`show device-tracking database` indicates how bindings were created. Common source codes are:

| Code | Meaning | Address family |
| --- | --- | --- |
| `ARP` | Learned from Address Resolution Protocol. | IPv4 |
| `DH4` | Learned from DHCPv4. | IPv4 |
| `ND` | Learned from Neighbor Discovery. | IPv6 |
| `DH6` | Learned from DHCPv6. | IPv6 |
| `S` | Statically configured binding. | IPv4 or IPv6 |
| `L` | Local entry, such as an address configured on a switch SVI. | IPv4 or IPv6 |
| `PKT` | Learned from another packet source. | Depends on source |
| `API` | Created through an API/client integration. | Depends on source |

The source code tells you **where the information came from**, not that the address was necessarily authenticated. An entry learned with `security-level glean`, for example, has not gone through the same ownership validation as one accepted under `security-level guard`.

### Preference-Level Flags (`prlvl`)

The `prlvl` column in `show device-tracking database` contains hexadecimal flags used to describe the binding's origin or preference. Examples include:

| Flag | Meaning |
| --- | --- |
| `0002` | Originated on a trunk. |
| `0004` | Originated on an access port. |
| `0008` | Originated on a trusted trunk. |
| `0010` | Originated on a trusted access port. |
| `0020` | DHCP-assigned. |
| `0100` | Statically assigned. |

Flags can be combined. For example, `0024` represents a DHCP-assigned address (`0020`) learned on an access port (`0004`). The source code (`DH4`, `ND`, etc.) and `prlvl` convey **different** information.

### Local and Static Entries

**Local entries** represent addresses configured on the switch itself, such as an IPv4 or IPv6 SVI address. The switch creates them automatically.

**Static entries** are manually configured for known hosts, such as devices that use fixed addresses or remain silent. Local entries have a higher priority than conflicting static entries for the same address.

---

## Binding Entry States

SISF tracks both the host's **identity** and its **reachability state**. These states apply to IPv4 and IPv6 bindings.

| State | Meaning |
| --- | --- |
| `REACHABLE` | The binding has been verified, and the host is considered reachable. |
| `STALE` | The reachable lifetime expired without a qualifying refresh; the entry is retained temporarily. |
| `DOWN` | The interface associated with the binding is down. |
| `TENTATIVE` | A transitional state while a new address or binding is evaluated. |
| `INCOMPLETE` | Required binding information or validation is not yet complete. |
| `VERIFY` | SISF is actively checking reachability or validating a change. |

`STALE` does not necessarily mean the endpoint has stopped functioning. It means SISF has not recently confirmed its reachability. A new qualifying control packet or successful reachability probe can restore the `REACHABLE` state.

### Reachable, Stale, and Down Lifetimes

Cisco documents these default lifetimes for ordinary dynamic bindings:

| State | Default lifetime | At expiration |
| --- | --- | --- |
| `REACHABLE` | **300 seconds (5 minutes)** | The entry becomes stale or enters verification if polling is enabled. |
| `STALE` | **86,400 seconds (24 hours)** | The entry is removed if it is not refreshed or successfully verified. |
| `DOWN` | **86,400 seconds (24 hours)** | The entry is removed if the associated interface remains down. |

A qualifying ARP, ND, or other learning event received while the entry is reachable can reset its reachable timer.

Without active polling, a typical lifecycle is:

```text
    ARP / ND learning event
             |
             v
         REACHABLE
          (5 min)
             |
      No valid refresh
             v
           STALE
         (24 hours)
             |
      No valid refresh
             v
       Entry removed
```

An interface going down can instead put associated entries into `DOWN`.

### Reachability Polling

With active tracking enabled, SISF can probe a host rather than relying only on passive observations.

| Address family | Probe | Typical response |
| --- | --- | --- |
| **IPv4** | Unicast **ARP probe**. | ARP Reply. |
| **IPv6** | **Neighbor Solicitation**. | Neighbor Advertisement. |

The process is the same for both:

1. The `REACHABLE` lifetime expires.
2. The entry enters `VERIFY`; SISF sends a reachability probe.
3. SISF can attempt polling **up to three times**.
4. A valid response returns the entry to `REACHABLE`.
5. If probing fails, the entry can become `STALE`.
6. When the stale lifetime expires, SISF can perform a final check before deleting the entry.

**Polling is separate from binding-table creation.** It is controlled by the effective tracking configuration; it is not automatically enabled merely because SISF has created a database. Details of polling configuration are covered in [IPv6 Device Tracking](device-tracking.md).

For **DHCPv4- and DHCPv6-originated bindings**, DHCP lease expiration takes precedence over the ordinary SISF stale/down lifetime. At lease expiration, the entry can be removed without additional polling.

---

## Configuring Binding Lifetimes

Global lifetime settings apply to both IPv4 and IPv6 bindings unless superseded by applicable policy/lease behavior.

Configure all three values in **one global configuration command**:

```text
SW1(config)# device-tracking binding reachable-lifetime 600 stale-lifetime 3600 down-lifetime 600
```

This sets `REACHABLE` to 10 minutes, `STALE` to one hour, and `DOWN` to 10 minutes.

The lifetime selectors can be configured individually or together, but **separate commands are not cumulative**. If you configure a different lifetime in a subsequent `device-tracking binding` command and omit the other selectors, their values revert to defaults. To preserve all three nondefault values, configure them together as shown above.

Policy-level settings can override global lifetime defaults. New global values apply to newly created entries rather than retroactively changing existing bindings. **DHCP lease lifetimes are not replaced by these stale/down settings.** Static bindings also have different aging behavior, described below.

---

## Basic Configuration and Verification Lab

Use the dual-stack topology from the beginning of the page. Both PCs are on VLAN 10:

| Device | MAC address | IPv4 | IPv6 |
| --- | --- | --- | --- |
| PC1 | `0011.2233.4455` | `192.0.2.10` (static) | `2001:DB8:10::10` (SLAAC or static) |
| PC2 | `0011.2233.4466` | `192.0.2.20` (DHCPv4) | `2001:DB8:10::20` (DHCPv6 or static) |

The addressing methods are chosen to demonstrate all four learning protocols. The actual binding **source** still depends on which messages SISF processes.

### Step 1: Create a Device-Tracking Policy

```text
SW1(config)# device-tracking policy LAN10_HOSTS
SW1(config-device-tracking)# security-level guard
SW1(config-device-tracking)# device-role node
SW1(config-device-tracking)# exit
```

`security-level guard` validates binding information and can reject unauthorized or conflicting claims. `security-level glean` learns information without address-ownership verification.

SISF can learn from ARP, DHCPv4, ND, and DHCPv6. The protocol selection commands in a device-tracking policy are `protocol arp`, `protocol dhcp4`, `protocol ndp`, and `protocol dhcp6`; the exact protocol set in effect can be confirmed in the policy output.

### Step 2: Apply the Policy to Host Ports

```text
SW1(config)# interface range GigabitEthernet1/0/1-2
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 10
SW1(config-if-range)# device-tracking attach-policy LAN10_HOSTS
```

An alternative for basic tracking is the built-in default policy:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# device-tracking
```

These are **alternatives**. Do not attach the custom policy and the default policy merely to perform the same test.

### Step 3: Generate IPv4 and IPv6 Learning Events

- From PC1, clear an IPv4 ARP entry and ping the gateway. Watch for an ARP Request that identifies `192.0.2.10` and PC1's MAC.
- From PC2, renew the DHCPv4 lease. Check whether SISF learns the assignment through DHCPv4 as `DH4`.
- From PC1, generate IPv6 ND traffic and look for `ND` bindings for the link-local and global IPv6 addresses.
- From PC2, perform a DHCPv6 address assignment where available and check for `DH6` entries.

Do not expect ICMP Echo packets alone to create bindings. SISF learns from the relevant control packets.

### Step 4: Inspect the Database

```text
SW1# show device-tracking database
SW1# show device-tracking database details
SW1# show device-tracking database vlanid 10
SW1# show device-tracking database interface GigabitEthernet1/0/1
SW1# show device-tracking policy LAN10_HOSTS
SW1# show device-tracking policies
```

Simplified **illustrative** output with both address families:

```text
SW1# show device-tracking database
Network Layer Address        Link Layer Address  Interface  vlan  prlvl  age  state      Time left
ARP 192.0.2.10               0011.2233.4455     Gi1/0/1    10    0004   12s  REACHABLE  288 s
ND  2001:DB8:10::10          0011.2233.4455     Gi1/0/1    10    0004   12s  REACHABLE  288 s
DH4 192.0.2.20               0011.2233.4466     Gi1/0/2    10    0024   26s  REACHABLE  lease
DH6 2001:DB8:10::20          0011.2233.4466     Gi1/0/2    10    0024   26s  REACHABLE  lease
```

The `lease` values are explanatory placeholders, not literal output. The actual `Time left` column can contain countdown values and additional retry/lease information.

### Useful Database Filters

```text
SW1# show device-tracking database address 192.0.2.10
SW1# show device-tracking database address 2001:DB8:10::10
SW1# show device-tracking database mac 0011.2233.4455
SW1# show device-tracking database prefix 2001:DB8:10::/64
SW1# show device-tracking database vlanid 10 details
```

- **Address:** Who owns this IPv4 or IPv6 address, and on which port?
- **MAC:** Which IP addresses are associated with this device?
- **Prefix:** Which IPv6 prefix entries match the specified IPv6 prefix?
- **VLAN/interface:** Where were the entries learned?
- **Details:** Which source, policy, state, and preference flags explain the binding?

---

## Static IPv4 and IPv6 Bindings

Static entries are useful for hosts whose addresses and locations are known but that do not send enough qualifying traffic to remain dynamically learned.

Suppose PC3 connects to `Gi1/0/3` in VLAN 10:

| Attribute | Value |
| --- | --- |
| IPv4 address | `192.0.2.50` |
| IPv6 address | `2001:DB8:10::50` |
| MAC address | `0050.56AA.0033` |
| Interface | `Gi1/0/3` |
| VLAN | 10 |

Configure **two separate static bindings**, one for each address:

```text
SW1(config)# device-tracking binding vlan 10 192.0.2.50 interface GigabitEthernet1/0/3 0050.56AA.0033
SW1(config)# device-tracking binding vlan 10 2001:DB8:10::50 interface GigabitEthernet1/0/3 0050.56AA.0033
```

The command family is:

```text
device-tracking binding vlan <vlan-id> <ipv4-address|ipv6-address|ipv6-prefix> [interface <interface>] [mac-address] [options]
```

The optional parameters include reachable-lifetime and tracking settings.

Verify both entries:

```text
SW1# show device-tracking database address 192.0.2.50 details
SW1# show device-tracking database address 2001:DB8:10::50 details
```

Static bindings have source code `S` and different persistence rules:

- A static entry is **not removed by ordinary dynamic database clearing**.
- A static entry can have a configurable reachable lifetime, but its `STALE` and `DOWN` lifetimes are effectively **infinite**.
- Remove it with the `no` form of the original configuration command.
- A local entry for an SVI IP address can supersede a static entry with the same IP address.

Remove the examples:

```text
SW1(config)# no device-tracking binding vlan 10 192.0.2.50 interface GigabitEthernet1/0/3 0050.56AA.0033
SW1(config)# no device-tracking binding vlan 10 2001:DB8:10::50 interface GigabitEthernet1/0/3 0050.56AA.0033
```

**Static bindings do not assign IPv4 or IPv6 addresses to hosts.** PC3 must independently have those addresses configured.

---

## Binding Table Limits

SISF supports global and policy-level limits to control database growth.

### Global Limits

Set a maximum number of bindings:

```text
SW1(config)# device-tracking binding max-entries 500
```

The global command also supports VLAN, port, and MAC limits:

```text
SW1(config)# device-tracking binding max-entries 500 vlan-limit 300 port-limit 50 mac-limit 20
```

These limits apply to the unified database. **IPv4 and IPv6 entries both contribute to binding counts.**

### Policy-Level Address Limit

```text
SW1(config)# device-tracking policy LAN10_HOSTS
SW1(config-device-tracking)# limit address-count 8
```

This limits the **total number of IP addresses per port** under the policy. It does not allow eight IPv4 addresses *plus* eight IPv6 addresses.

Example: On a port with a limit of `8`, a host with **one IPv4 address and seven IPv6 addresses** has reached the limit. Another IPv6 privacy address cannot create an additional binding.

SISF can also have separate **IPv4-per-MAC** and **IPv6-per-MAC** limits supplied by programmatically installed policies; inspect the effective policy to see these values.

**When an applicable limit is reached, SISF stops adding further bindings.** Reducing a limit generally affects new learning rather than immediately deleting existing entries. The limit affects binding-table population; whether packets are forwarded or dropped also depends on the policy and security feature involved.

---

## Multi-Switch Networks and Binding Location

Consider two access switches with a trunk carrying VLAN 10:

```text
            PC1                         PC2
        IPv4 / IPv6                 IPv4 / IPv6
             |                           |
          Gi1/0/1                     Gi1/0/1
          +-------+                   +-------+
          |  SW1  |===================|  SW2  |
          +-------+   VLAN 10 trunk   +-------+
                  Gi1/0/48   Gi1/0/48
```

Because **ARP, DHCPv4, ND, and DHCPv6** traffic can traverse the trunk, a switch could learn remote hosts as bindings on its uplink. That can unnecessarily duplicate host entries and cause repeated validation.

A common design is to learn **directly attached hosts on access ports** and avoid creating duplicate bindings for hosts behind other SISF-enabled switches.

Configure a switch-facing device-tracking policy:

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

Apply the equivalent configuration on SW2's uplink. Both settings are used together:

- `device-role switch` identifies the port as connecting to another switch rather than an end host.
- `trusted-port` marks the uplink as trusted for SISF binding validation and preference.

Together, these settings let the access switches maintain **distributed tables** containing directly attached hosts rather than learning every endpoint throughout the Layer 2 domain.

The alternative is a **centralized binding table** on a distribution switch, which can be useful when downstream switches do not run SISF. The appropriate design depends on where host-validation responsibility belongs.

These are SISF device-tracking policy commands, separate from RA Guard's similarly named `device-role` and `trusted-port` options.

---

## Binding Integrity and Address Movement

A binding ties an IP address not just to a MAC address but also to a **location** (VLAN and switch port). This matters when a host moves or an attacker claims another host's address.

### IPv4: Conflicting ARP Information

Suppose the existing binding is:

```text
192.0.2.10 -> 0011.2233.4455, VLAN 10, Gi1/0/1
```

SW1 then receives an ARP Reply on `Gi1/0/2` claiming:

```text
192.0.2.10 -> 0011.2233.9999, VLAN 10, Gi1/0/2
```

The ARP claim conflicts with the current mapping. With `security-level guard`, SISF can verify ownership and reject an invalid replacement. This is relevant to **ARP spoofing** and accidental duplicate IPv4 addresses.

### IPv6: Conflicting Neighbor Discovery Information

Suppose the existing IPv6 binding is:

```text
2001:DB8:10::10 -> 0011.2233.4455, VLAN 10, Gi1/0/1
```

An NA arriving on `Gi1/0/2` claims:

```text
2001:DB8:10::10 -> 0011.2233.9999, VLAN 10, Gi1/0/2
```

This is the corresponding IPv6 ownership conflict. SISF can validate a new claim rather than automatically replacing the legitimate binding.

For both address families:

- `security-level glean` learns without address-ownership validation.
- `security-level guard` performs stronger validation and can reject spoofed or conflicting information.
- A changed port or MAC is **not automatically proof of an attack**. A legitimate host move must also be supported.

**Binding integrity is not the same as filtering all data packets.** The table holds identity/location information; features such as IPv6 Source Guard and IPv4 source-verification mechanisms provide separate data-plane enforcement.

---

## Verification and Troubleshooting

### Show Commands

| Command | Purpose |
| --- | --- |
| `show device-tracking database` | List IPv4 and IPv6 bindings and states. |
| `show device-tracking database details` | Display additional binding/source/policy information. |
| `show device-tracking database vlanid 10` | Show bindings in VLAN 10. |
| `show device-tracking database interface GigabitEthernet1/0/1` | Show bindings on a particular port. |
| `show device-tracking database address 192.0.2.10` | Look up a specific IPv4 binding. |
| `show device-tracking database address 2001:DB8:10::10` | Look up a specific IPv6 binding. |
| `show device-tracking policy LAN10_HOSTS` | Display policy parameters and attachments. |
| `show device-tracking policies` | List device-tracking policies and targets. |
| `show device-tracking counters vlan 10` | Display SISF packet processing and drop counters. |
| `show device-tracking events` | Display SISF events. |
| `show device-tracking messages` | Display SISF messages. |

### Clearing Dynamic Entries

```text
SW1# clear device-tracking database
```

This clears dynamic bindings but **does not remove static entries**. Depending on the security features consuming the database, clearing entries can temporarily affect enforcement until they are relearned.

### Common Troubleshooting Cases

| Symptom | Likely explanation | What to check |
| --- | --- | --- |
| No IPv4 or IPv6 bindings appear | Device tracking is not active on the interface/VLAN. | Policy attachment and `show device-tracking policies`. |
| IPv4 host appears as `ARP` instead of `DH4` | SISF learned the host from ARP rather than a DHCP event. | Source codes and protocol activity. |
| IPv6 host appears as `ND` instead of `DH6` | Address was learned from ND, for example with SLAAC. | ND packets and DHCPv6 behavior. |
| IPv4 host has connectivity but no SISF entry | No qualifying ARP/DHCPv4 learning event occurred. | ARP activity and enabled learning protocols. |
| IPv6 host has connectivity but no SISF entry | No qualifying ND/DHCPv6 learning event occurred. | ND/DHCPv6 activity and enabled learning protocols. |
| One host has many entries | Dual-stack addressing and multiple IPv6 addresses. | MAC-based lookup and address-count limits. |
| Entry changes to `STALE` | Reachable lifetime expired without a refresh. | Lifetimes and polling policy. |
| Entry changes to `DOWN` | The associated interface is down. | Port and link status. |
| DHCP-originated entry disappears | DHCPv4 or DHCPv6 lease expired. | Lease expiration/renewal. |
| A new address cannot be learned | A global or policy-level count limit was reached. | `max-entries`, `limit address-count`, counters. |
| Remote hosts appear on trunk ports | Trunk learns ARP/ND/DHCP traffic from downstream devices. | Uplink policy and `device-role switch`/`trusted-port`. |
| Conflicting ARP or NA claim is rejected | `guard` validation detects inconsistent address ownership. | Existing binding, event logs, counters. |
| Clearing database does not remove an entry | It is static or local. | Source codes `S` and `L`. |

### Suggested Lab Tests

1. **ARP learning:** Configure a static IPv4 address, generate an ARP Request, and verify an `ARP` binding.
2. **DHCPv4 learning:** Renew an IPv4 DHCP lease and compare a DHCP-originated entry with an ARP-learned entry.
3. **SLAAC learning:** Observe a host's IPv6 link-local, stable global, and temporary addresses in the binding table.
4. **DHCPv6 learning:** Obtain an IPv6 address using DHCPv6 and inspect the source and lease behavior.
5. **Reachability:** Compare IPv4 ARP polling with IPv6 NS polling as entries move from `REACHABLE` to `VERIFY`/`STALE`.
6. **Static bindings:** Add one IPv4 and one IPv6 binding for the same MAC/port and confirm that clearing dynamic entries preserves both.
7. **Shared address limit:** Set a small `limit address-count` and exhaust it with a mixture of IPv4 and IPv6 addresses.
8. **Conflict validation:** Observe what happens when a second port claims an existing IPv4 address via ARP or an existing IPv6 address via NA.
9. **Multi-switch topology:** Confirm that host-facing ports learn their own endpoints while the trusted switch-facing trunk policy avoids duplicate bindings.

---

## Key Points

- The **SISF binding table is unified for IPv4 and IPv6**, even though this CCIE blueprint item is titled IPv6 Binding Table.
- Every binding associates an IP address with a **MAC, VLAN, port, source, and reachability state**.
- **ARP and DHCPv4** are principal IPv4 learning sources; **ND and DHCPv6** are principal IPv6 learning sources.
- A dual-stack host can have multiple bindings for one MAC, including multiple IPv6 addresses.
- `REACHABLE`, `STALE`, and `DOWN` are the principal stable states; `VERIFY`, `INCOMPLETE`, and `TENTATIVE` are transitional.
- Default dynamic lifetimes are **5 minutes** for `REACHABLE` and **24 hours** for `STALE`/`DOWN`. DHCP-originated entries follow their lease lifetimes.
- Reachability polling uses **ARP for IPv4** and **Neighbor Solicitations for IPv6**.
- Static IPv4 and IPv6 entries use `device-tracking binding vlan ...`, have source code `S`, and survive dynamic database clearing.
- The policy-level `limit address-count` counts **IPv4 and IPv6 entries together** on a port.
- `show device-tracking database` is the primary verification command for **both** address families.
- A binding table alone is not an ACL; **security policies determine how host information is validated and enforced**.

---

## References

- [Cisco: FHS and SISF Configuration Guide, SISF (IOS XE 17)](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/sec-crypto/fhs-sisf/fhs-and-sisf-configuration-guide/switch-integrated-security-features.html)
- [Cisco: Configuring Switch Integrated Security Features, Catalyst 9500](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9500/software/release/17-15/configuration_guide/sec/b_1715_sec_9500_cg/configuring_sisf_based_device_tracking.html)
- [Cisco: Security Command Reference, Catalyst 9200, IOS XE 17.15](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9200/software/release/17-15/command_reference/b_1715_9200_cr/security_commands.html)
