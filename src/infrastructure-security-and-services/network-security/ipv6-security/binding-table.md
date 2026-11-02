# 4.2.c (iii) IPv6 Binding Table

The **SISF (Switch Integrated Security Features) binding table** is a database of IP addresses and their associated MAC addresses, VLANs, and switch interfaces. It allows a switch to identify **which host is using an address and where that host is connected**.

Although this topic is listed under IPv6 infrastructure security, the SISF binding table supports **both IPv4 and IPv6**. It is populated by [device tracking](device-tracking.md) and can supply host information to features such as [ND Inspection](nd-inspection.md) and [IPv6 Source Guard](source-guard.md).

The binding table is **not** the IPv6 neighbor table, MAC address table, or DHCP snooping database. Its purpose is to maintain **IP-to-MAC-to-location bindings**, including information about how they were learned and whether they remain valid.

---

## Binding Table Entries

Consider two hosts connected to SW1:

```text
                     SW1 (VLAN 10)
                   /               \
                Gi1/0/1          Gi1/0/2
                   |                |
                  PC1              PC2
           0011.2233.4455    0011.2233.4466
           2001:DB8:10::10   2001:DB8:10::20
```

The switch can maintain bindings such as:

| IP address | MAC address | VLAN | Interface | Source |
| --- | --- | --- | --- | --- |
| `2001:DB8:10::10` | `0011.2233.4455` | 10 | `Gi1/0/1` | ND |
| `FE80::211:22FF:FE33:4455` | `0011.2233.4455` | 10 | `Gi1/0/1` | ND |
| `2001:DB8:10::20` | `0011.2233.4466` | 10 | `Gi1/0/2` | DHCPv6 |

An individual host can have **multiple binding entries**, for example one for its link-local address, another for its stable global unicast address, and additional entries for temporary IPv6 addresses. The same database can also contain the host's IPv4 address.

Each binding maintains information including:

- **Network-layer address:** IPv4 or IPv6 address (or, where applicable, an IPv6 prefix).
- **Link-layer address:** Associated MAC address.
- **Interface and VLAN:** Where the host is connected.
- **Source:** How the entry was created.
- **State:** Whether the binding is considered reachable, stale, down, or undergoing verification.
- **Age and remaining lifetime:** How long the entry has existed or been in its current state, and when its current state timer expires.
- **Preference-level flags (`prlvl`):** Attributes used when SISF evaluates and prioritizes binding information.

### What Creates an Entry?

SISF can learn entries from several sources:

| Source code | Meaning |
| --- | --- |
| `ND` | IPv6 Neighbor Discovery |
| `DH6` | DHCPv6 |
| `ARP` | IPv4 ARP |
| `DH4` | DHCPv4 |
| `PKT` | Other packet-based learning |
| `S` | Manually configured static binding |
| `L` | Locally configured IP address, such as an address on an SVI |
| `API` | Entry added through an API |

A source code describes **how information was learned or installed**. It does not necessarily tell you how the host originally configured its address. For example, an IPv6 address assigned by DHCPv6 could be discovered through ND and appear with the `ND` source code.

Device-tracking policy settings control which protocols the switch inspects. See [IPv6 Device Tracking](device-tracking.md) for discovery methods and policy configuration.

---

## Binding States

An entry has a **state**, which represents SISF's current assessment of the binding's reachability or validation status.

| State | Meaning |
| --- | --- |
| `REACHABLE` | Binding has been validated and the host is considered reachable. |
| `STALE` | The reachable lifetime has expired without renewed evidence of reachability. The entry is retained temporarily. |
| `DOWN` | The interface associated with the binding is down. |
| `TENTATIVE` | A newly discovered binding is not yet fully validated. |
| `INCOMPLETE` | SISF does not yet have all the information needed to complete the binding. |
| `VERIFY` | SISF is checking the binding's validity or reachability. |

The three principal **stable states** are `REACHABLE`, `STALE`, and `DOWN`. The others are transitional states during learning or verification.

### Typical Binding Lifecycle

```text
New host information
        |
        v
TENTATIVE / INCOMPLETE
        |
        | Successful validation
        v
    REACHABLE <-------------------+
        |                         |
        | Reachable timer expires | Successful verification
        v                         |
      VERIFY ---------------------+
        |
        | No successful verification
        v
      STALE
        |
        | Stale timer expires
        v
   Entry deleted
```

If active polling is disabled, a binding can transition from `REACHABLE` to `STALE` when its reachable lifetime expires without active verification. If polling is enabled, SISF can probe the host before declaring the entry stale.

**Interface-down events** can transition an entry into `DOWN`, rather than following the ordinary reachable-to-stale path. An entry may also be removed if validation detects invalid or conflicting ownership information.

### REACHABLE

A `REACHABLE` entry represents a verified IP/MAC binding whose reachability has been recently confirmed.

- Default reachable lifetime: **300 seconds (5 minutes)**.
- Relevant incoming control traffic can refresh the reachable lifetime.
- With active tracking enabled, successful probes can also reconfirm reachability.

`REACHABLE` does not mean that the switch continuously receives traffic from the device. It means that the current binding has not yet lost its validated reachable status.

### STALE

A `STALE` entry retains the host's information even though its reachable lifetime has expired.

- Default stale lifetime: **86,400 seconds (24 hours)**.
- New valid evidence of reachability can return it to `REACHABLE`.
- If its stale lifetime expires without recovery, the dynamic entry is removed.

A stale binding is **not equivalent to proof that the host is currently reachable**. The information is retained so SISF can recover or reuse it if the host becomes active again.

### DOWN

A `DOWN` entry indicates that the host's associated switch interface is down.

- Default down lifetime: **86,400 seconds (24 hours)**.
- The binding can be retained while the interface is unavailable.
- A dynamic entry is removed if its down lifetime expires without recovery.

---

## Configuring Binding Lifetimes

The following commands configure the **global default lifetimes** for applicable SISF bindings:

```text
SW1(config)# device-tracking binding reachable-lifetime 600
SW1(config)# device-tracking binding stale-lifetime 3600
SW1(config)# device-tracking binding down-lifetime 1800
```

These settings configure:

| State | Example lifetime |
| --- | --- |
| `REACHABLE` | 600 seconds (10 minutes) |
| `STALE` | 3,600 seconds (1 hour) |
| `DOWN` | 1,800 seconds (30 minutes) |

To configure an unlimited lifetime, use `infinite` where supported by the corresponding command:

```text
SW1(config)# device-tracking binding reachable-lifetime infinite
```

**DHCP-learned bindings:** Their DHCP lease information takes precedence over the global SISF stale and down lifetime settings. Do not assume that changing these global timers overrides DHCP lease expiration.

**Binding lifetime vs. active probing:** Lifetimes determine how long a binding remains in a state. Device-tracking `tracking` settings separately determine whether and how SISF actively probes hosts. See [IPv6 Device Tracking](device-tracking.md).

---

## Static Binding Entries

You can manually create a binding when a host should have a known IP address and attachment location, or when a host sends too little control traffic for reliable dynamic discovery.

### Configure a Static IPv6 Binding

For example, configure the following association:

| Field | Value |
| --- | --- |
| VLAN | 10 |
| IPv6 address | `2001:DB8:10::10` |
| Interface | `GigabitEthernet1/0/1` |
| MAC address | `0011.2233.4455` |

```text
SW1(config)# device-tracking binding vlan 10 2001:DB8:10::10 interface GigabitEthernet1/0/1 0011.2233.4455
```

You can also specify a reachable lifetime for a static binding:

```text
SW1(config)# device-tracking binding vlan 10 2001:DB8:10::10 interface GigabitEthernet1/0/1 0011.2233.4455 reachable-lifetime 600
```

Static bindings appear with the source code **`S`**.

### How Static Bindings Age

A static binding is not necessarily always shown as `REACHABLE`. It can transition to `STALE` or `DOWN` based on host reachability and interface state.

However, static bindings have an important difference from dynamic bindings:

- Their **reachable lifetime can be configured**.
- Their **stale and down lifetimes are infinite**.
- They **are not removed merely because the host remains stale or down**.
- They **cannot be cleared with ordinary database clear commands**. Remove the static configuration to delete a static entry.

A static entry in `STALE` or `DOWN` may show `N/A` under `Time left` because no expiration timer is running for that state.

### Local Bindings

A `L` entry represents an address belonging to the switch itself, for example an IPv6 address configured on an SVI. If a local address conflicts with a manually created static binding, the **local entry takes priority**.

---

## Reading the Binding Table

The main verification command is:

```text
SW1# show device-tracking database
```

The following is an **illustrative, shortened output**:

```text
Binding Table has 3 entries, 2 dynamic (limit 200000)
Codes: L - Local, S - Static, ND - Neighbor Discovery,
       ARP - Address Resolution Protocol, DH4 - IPv4 DHCP,
       DH6 - IPv6 DHCP, PKT - Other Packet, API - API created

Network Layer Address     Link Layer Address  Interface  vlan  prlvl  age  state      Time left
ND 2001:DB8:10::10        0011.2233.4455     Gi1/0/1    10   0005   40s  REACHABLE  260 s
DH6 2001:DB8:10::20       0011.2233.4466     Gi1/0/2    10   0025   50s  REACHABLE  250 s
S  2001:DB8:10::30        0011.2233.4477     Gi1/0/3    10   0105   90s  STALE      N/A
```

Interpret the columns as follows:

| Column | Meaning |
| --- | --- |
| Source prefix (`ND`, `DH6`, `S`, etc.) | Origin of the binding information. |
| `Network Layer Address` | IP address associated with the host. |
| `Link Layer Address` | Host's MAC address. |
| `Interface` | Associated switch interface. |
| `vlan` / `vlan/bd` | VLAN or bridge-domain identifier. |
| `prlvl` | Hexadecimal preference-level bitmask. |
| `age` | Age shown for the binding. |
| `state` | Current binding state. |
| `Time left` | Remaining time in the current state; `N/A` means no applicable expiration timer. |

### Preference-Level Flags (`prlvl`)

The `prlvl` field is a **hexadecimal bitmask**, not a decimal score or a single enumerated value. Multiple flags can be present simultaneously.

| Flag | Meaning |
| --- | --- |
| `0001` | MAC and link-local address match |
| `0002` | Originally learned on a trunk |
| `0004` | Originally learned on an access port |
| `0008` | Originally learned on a trusted trunk |
| `0010` | Originally learned on a trusted access port |
| `0020` | DHCP-assigned address |
| `0040` | Cryptographically Generated Address (CGA) authenticated |
| `0080` | Certificate authenticated |
| `0100` | Statically assigned |

Examples:

- `0005` = `0001` + `0004`: MAC/link-local match and origin on an access port.
- `0025` = `0020` + `0004` + `0001`: DHCP-assigned, access-port origin, and MAC/link-local match.
- `0105` = `0100` + `0004` + `0001`: Statically assigned, access-port origin, and MAC/link-local match.

These flags describe the binding's attributes and contribute to how SISF evaluates entries. The word **preference** should not be interpreted as a simple universal ranking based on the numerical total.

### Useful Verification Commands

```text
SW1# show device-tracking database
SW1# show device-tracking database details
SW1# show device-tracking policies
SW1# show device-tracking policies interface GigabitEthernet1/0/1
SW1# show device-tracking policy default
```

`show device-tracking database details` provides expanded information for analyzing how bindings were learned and maintained. The policy commands help confirm that device tracking is actually attached to the intended port or VLAN.

---

## Relationship to Other Security Features

| Feature | Relationship to the binding table |
| --- | --- |
| **Device Tracking** | Learns, validates, and refreshes entries. |
| **ND Inspection** | Uses binding information to help detect invalid address ownership claims. |
| **IPv6 Source Guard** | Uses authorized IP-to-location information when enforcing source-address restrictions. |
| **RA Guard** | Filters unauthorized Router Advertisements independently of whether a host binding exists. |
| **DHCPv6 Guard** | Filters unauthorized DHCPv6 server/relay messages independently of whether a host binding exists. |

The existence of an entry does **not automatically mean that all packets from that host are permitted**, or that the address is cryptographically authenticated. What a security feature does with an entry depends on the policy and the entry's attributes and validation status.

Also distinguish the SISF table from other databases:

- **MAC address table:** Maps MAC addresses to switch ports for Layer 2 forwarding.
- **IPv6 neighbor table:** Maps on-link IPv6 addresses to link-layer information for IPv6 neighbor resolution.
- **DHCP snooping binding table:** Primarily records DHCPv4 lease-based IP/MAC/VLAN/port information.
- **SISF binding table:** Records IPv4 and IPv6 address-to-MAC-to-attachment information obtained from several sources and maintains validity and reachability state.

---

## Key Points

- SISF maintains a **unified IPv4/IPv6 binding table**.
- A host can have **multiple IPv6 bindings**, all associated with the same MAC address and port.
- `ND`, `DH6`, `ARP`, and `DH4` identify dynamic learning sources; `S` and `L` identify static and local entries.
- Stable states are **`REACHABLE`, `STALE`, and `DOWN`**; `VERIFY`, `INCOMPLETE`, and `TENTATIVE` are transitional states.
- Default lifetimes are **300 seconds reachable**, **24 hours stale**, and **24 hours down** for applicable non-DHCP dynamic bindings.
- Static bindings **remain in the database indefinitely while stale or down**, unless their configuration is removed.
- The `prlvl` field is a **hexadecimal combination of flags**.
- Use **`show device-tracking database`** and **`show device-tracking database details`** to inspect entries.

---

## References

- [Cisco FHS and SISF Configuration Guide: Switch Integrated Security Features](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/sec-crypto/fhs-sisf/fhs-and-sisf-configuration-guide/switch-integrated-security-features.html)
- [Cisco: Troubleshoot SISF on Catalyst 9000 Series Switches](https://www.cisco.com/c/en/us/support/docs/switches/catalyst-9300-series-switches/221562-troubleshoot-sisf-on-catalyst-9000-serie.html)
