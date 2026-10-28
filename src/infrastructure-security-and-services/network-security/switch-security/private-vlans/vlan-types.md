# Primary and Secondary VLANs

A **Private VLAN (PVLAN)** uses multiple VLAN IDs to create a single Layer 2 domain with controlled communication between hosts.

Instead of placing every host in an ordinary VLAN where direct Layer 2 communication is generally permitted, a PVLAN divides the domain into a **primary VLAN** and one or more **secondary VLANs**.

The secondary VLAN assigned to a host determines which other hosts it can communicate with directly.

---

## PVLAN Structure

A PVLAN consists of:

- **One primary VLAN:** Defines the overall PVLAN domain and provides connectivity to promiscuous ports, such as ports connected to a shared gateway.
- **Zero or one isolated secondary VLAN:** Provides Layer 2 isolation between its host ports and all other PVLAN host ports.
- **Zero or more community secondary VLANs:** Allows host ports in the same community to communicate with each other while isolating them from other communities and isolated host ports.

A primary VLAN must have at least one associated secondary VLAN to provide PVLAN host connectivity.

For example:

```text
                         Primary VLAN 100
                                |
                +---------------+---------------+
                |               |               |
             Isolated        Community       Community
             VLAN 101        VLAN 102        VLAN 103
                |             /   \           /   \
              H1, H2         H3   H4         H5   H6

         Shared gateway / promiscuous port
                    Primary VLAN 100
```

All six hosts can use IP addresses from the **same subnet**, even though their direct Layer 2 connectivity differs.

| VLAN | PVLAN role | Hosts | Direct host-to-host communication |
|---|---|---|---|
| 100 | Primary | Shared gateway connected through a promiscuous port | Gateway can communicate with all mapped secondary VLANs |
| 101 | Isolated secondary | H1, H2 | Neither host can communicate directly with any other PVLAN host |
| 102 | Community secondary | H3, H4 | H3 and H4 can communicate with each other |
| 103 | Community secondary | H5, H6 | H5 and H6 can communicate with each other |

H3 cannot communicate directly with H5, even though both are attached to community ports within the same primary VLAN domain.

---

## Primary VLAN

The **primary VLAN** identifies the PVLAN domain and associates its secondary VLANs with the ports that provide shared connectivity.

A primary VLAN has several important functions:

- It associates one or more secondary VLANs into a single PVLAN domain.
- **Promiscuous ports** belong to the primary VLAN and can communicate with host ports in their mapped secondary VLANs.
- Traffic from promiscuous ports toward hosts is carried using the primary VLAN within the PVLAN forwarding model.
- When the switch provides the default gateway using an SVI, the **SVI is configured on the primary VLAN**, not on a secondary VLAN.
- The primary VLAN provides the Spanning Tree Protocol (STP) parameters used by its associated secondary VLANs.

For example, VLAN 100 could be the primary VLAN for an entire server subnet:

```text
Primary VLAN: 100
IP subnet:    192.0.2.0/24
Gateway:      192.0.2.1

Secondary VLANs:
  101 (isolated)
  102 (community)
  103 (community)
```

The primary VLAN does not automatically permit every attached host to communicate with every other host. **Host-to-host communication is controlled by the secondary VLANs and PVLAN port roles.**

> A primary VLAN is not simply a regular VLAN containing every PVLAN host as an ordinary access port. PVLAN host ports require a primary/secondary VLAN association, while promiscuous ports use primary-to-secondary mappings.

---

## Isolated Secondary VLAN

An **isolated secondary VLAN** is used when hosts should reach a shared gateway or service but should not communicate directly with other PVLAN host ports.

An isolated host port:

- Can communicate with appropriately mapped promiscuous ports.
- Cannot communicate directly with another isolated host port, **even within the same isolated VLAN**.
- Cannot communicate directly with community host ports.

For example:

```text
               R1 (Gateway)
                    |
             Promiscuous Port
                    |
                   SW1
                    |
              Primary 100
                 /    \
             H1          H2
              \          /
              Isolated 101

H1 -> R1: Allowed
H2 -> R1: Allowed
H1 -> H2: Blocked at Layer 2
```

This behavior is different from a regular VLAN. Two ordinary access ports in VLAN 101 could normally exchange frames, but **two isolated PVLAN host ports cannot**.

### One Isolated VLAN per Primary VLAN

A primary VLAN can be associated with **only one isolated secondary VLAN**.

This does **not** mean that the isolated VLAN can contain only one host port. Any number of supported host ports can be assigned to that isolated VLAN, and each remains isolated from the others.

For example:

```text
Primary VLAN 100
  Isolated VLAN 101
    H1, H2, H3, H4, H5 ...
```

All of these hosts can use the same default gateway without being allowed to communicate directly with one another.

---

## Community Secondary VLAN

A **community secondary VLAN** allows a selected group of hosts to communicate directly with one another while preventing direct Layer 2 communication with other PVLAN host groups.

A community host port:

- Can communicate with other host ports in the **same community secondary VLAN**.
- Can communicate with appropriately mapped promiscuous ports.
- Cannot communicate directly with host ports in a **different community secondary VLAN**.
- Cannot communicate directly with isolated host ports.

For example:

```text
                   Primary VLAN 100
                          |
                   +------+------+
                   |             |
              Community 102  Community 103
                 /    \         /    \
                H1    H2       H3    H4

H1 <-> H2: Allowed
H3 <-> H4: Allowed
H1 <-> H3: Blocked at Layer 2
H2 <-> H4: Blocked at Layer 2
```

A primary VLAN can have **multiple community secondary VLANs**, allowing multiple distinct groups to share a common default gateway without permitting direct communication between groups.

For example, different application-server clusters could use separate community VLANs within the same IP subnet.

---

## Primary-to-Secondary VLAN Associations

The primary VLAN must be **associated** with its secondary VLANs. This establishes which isolated and community VLANs belong to the same PVLAN domain.

The following rules apply:

| Relationship | Allowed? |
|---|---|
| One primary VLAN associated with one isolated VLAN | Yes |
| One primary VLAN associated with multiple community VLANs | Yes |
| One primary VLAN associated with two isolated VLANs | No |
| One secondary VLAN associated with two primary VLANs | No |
| A secondary VLAN configured as both isolated and community | No |
| Multiple independent primary VLANs on the same switch | Yes |

**Each secondary VLAN belongs to exactly one primary VLAN when associated.** A secondary VLAN cannot be shared between separate PVLAN domains.

### Valid Association Example

```text
Primary VLAN 100
  Isolated: 101
  Community: 102, 103

Primary VLAN 200
  Isolated: 201
  Community: 202, 203
```

These are two independent PVLAN domains. Hosts in the VLAN 100 domain are not placed in the VLAN 200 domain simply because both domains use private VLANs.

### Invalid Association Examples

```text
Primary VLAN 100
  Isolated: 101, 104    <-- Invalid: two isolated VLANs
```

```text
Primary VLAN 100
  Community: 102

Primary VLAN 200
  Community: 102        <-- Invalid: secondary VLAN shared
```

An isolated or community VLAN must also have a **different VLAN ID** from its primary VLAN.

### VLAN Association vs. Port Association

Two different associations are involved in a working PVLAN:

1. **VLAN-level association:** Identifies the secondary VLANs that belong to a primary VLAN.
2. **Interface-level association or mapping:** Identifies which primary and secondary VLANs a host port or promiscuous port uses.

Creating the VLAN-level association does **not** automatically configure switch interfaces as PVLAN host or promiscuous ports.

Interface-level configuration is covered in [PVLAN Port Types and Forwarding Rules](port-types.md) and [Private VLAN Configuration](configuration.md).

---

## Traffic and VLAN Identification

Primary and secondary VLANs are separate VLAN IDs, but they work together as one PVLAN domain.

Cisco describes their traffic roles as follows:

| VLAN type | Traffic role |
|---|---|
| Primary | Carries traffic downstream from promiscuous ports toward associated host ports |
| Isolated secondary | Carries traffic upstream from isolated host ports toward promiscuous ports |
| Community secondary | Carries traffic from community host ports toward promiscuous ports and other host ports in the same community |

For example:

```text
H1 (Isolated VLAN 101)
        |
        | Traffic toward gateway
        | Associated with secondary VLAN 101
        v
       SW1
        |
        | Traffic toward promiscuous port
        v
R1 (Primary VLAN 100)
```

Traffic traveling back from the gateway is forwarded from the primary VLAN toward the relevant secondary host port.

**The terms "upstream" and "downstream" describe how the PVLAN forwards traffic. They do not mean that communication with the gateway is one-way.** Hosts can exchange traffic bidirectionally with an appropriately mapped promiscuous port.

On a standard 802.1Q trunk between PVLAN-capable switches, the primary and secondary VLAN IDs must be carried as required by the PVLAN design. PVLAN trunk modes and VLAN-tag handling are covered in [Private VLAN Trunks](trunks.md).

---

## IP Subnets and Secondary VLANs

In an ordinary VLAN design, hosts in separate VLANs typically use separate IP subnets and communicate through a router.

A PVLAN differs because **hosts in the primary and associated secondary VLANs can use the same IP subnet and default gateway**.

For example:

| Host | Secondary VLAN | IP address | Default gateway |
|---|---|---|---|
| H1 | Isolated 101 | 192.0.2.11/24 | 192.0.2.1 |
| H2 | Isolated 101 | 192.0.2.12/24 | 192.0.2.1 |
| H3 | Community 102 | 192.0.2.13/24 | 192.0.2.1 |
| H4 | Community 102 | 192.0.2.14/24 | 192.0.2.1 |
| H5 | Community 103 | 192.0.2.15/24 | 192.0.2.1 |

All five hosts regard one another as being on the same IP subnet, but the PVLAN restricts which Layer 2 frames reach which host ports.

For example, an ARP request from H1 for H2's IP address does not reach H2, even though the hosts use addresses from the same subnet.

A common Layer 3 gateway is associated with the **primary VLAN**. On a Layer 3 switch, the SVI is configured under the primary VLAN and secondary VLANs are mapped to it as required.

PVLANs are a **Layer 2 isolation mechanism**. They do not, by themselves, prevent traffic from being routed between hosts through a gateway that permits such traffic. Layer 3 behavior is covered in [Layer 3 Connectivity with Private VLANs](layer3.md).

---

## STP and VLAN Associations

Although a PVLAN domain contains multiple VLAN IDs, Cisco Catalyst PVLAN implementations use **one STP instance for the PVLAN domain**.

When a secondary VLAN is associated with a primary VLAN, the primary VLAN's STP parameters are propagated to the secondary VLAN.

For example:

```text
Primary VLAN 100
  |
  +-- Isolated VLAN 101
  +-- Community VLAN 102
  +-- Community VLAN 103

STP parameters are derived from primary VLAN 100.
```

Do not assume that secondary VLANs behave like fully independent regular VLANs with separately configurable STP topologies.

---

## Configuring VLAN Types and Associations

The following commands define primary VLAN 100, isolated VLAN 101, and community VLANs 102 and 103.

```text
SW1(config)# vlan 101
SW1(config-vlan)# private-vlan isolated
SW1(config-vlan)# exit
!
SW1(config)# vlan 102
SW1(config-vlan)# private-vlan community
SW1(config-vlan)# exit
!
SW1(config)# vlan 103
SW1(config-vlan)# private-vlan community
SW1(config-vlan)# exit
!
SW1(config)# vlan 100
SW1(config-vlan)# private-vlan primary
SW1(config-vlan)# private-vlan association 101-103
SW1(config-vlan)# exit
```

The commands are:

| Command | Effect |
|---|---|
| `private-vlan primary` | Designates the VLAN as primary |
| `private-vlan isolated` | Designates the VLAN as an isolated secondary |
| `private-vlan community` | Designates the VLAN as a community secondary |
| `private-vlan association <vlan-list>` | Associates secondary VLANs with the primary VLAN |

The `private-vlan association` command is entered **under the primary VLAN**, not under each secondary VLAN.

For example, to add another community VLAN after defining it:

```text
SW1(config)# vlan 104
SW1(config-vlan)# private-vlan community
SW1(config-vlan)# exit
SW1(config)# vlan 100
SW1(config-vlan)# private-vlan association add 104
SW1(config-vlan)# exit
```

The `add` keyword adds a VLAN to the existing association list. Without `add`, specifying a new list replaces the existing association list.

> The VLAN-level `private-vlan` commands take effect when you exit VLAN configuration mode. These commands define the VLAN relationships only; they do not configure any physical switch port.

### Verification

Use:

```text
SW1# show vlan private-vlan
```

Expected relationships:

```text
Primary  Secondary  Type
-------  ---------  ---------
100      101        isolated
100      102        community
100      103        community
100      104        community
```

This output verifies the primary-to-secondary associations. It does not prove that individual host or promiscuous ports have been configured or mapped correctly.

---

## VLAN ID and VTP Considerations

On Cisco Catalyst switches:

- VLAN **1** and VLANs **1002 through 1005** cannot be configured as primary or secondary PVLANs.
- VLANs **2 through 1001** and **1006 through 4094** can be used, subject to platform restrictions.
- With **VTPv1 or VTPv2**, private VLANs require VTP transparent mode; those versions do not advertise PVLAN configurations.
- **VTPv3** supports PVLAN configuration propagation and supports PVLANs in server/client modes.
- When PVLANs extend across switches, the relevant primary/secondary VLAN definitions and associations must be preserved throughout the Layer 2 network.

Detailed trunking and multi-switch considerations are covered in [Private VLAN Trunks](trunks.md).

---

## Key Points

- A PVLAN domain consists of **one primary VLAN and associated secondary VLANs**.
- Secondary VLANs are either **isolated** or **community** VLANs.
- A primary VLAN can have **one isolated VLAN** and **multiple community VLANs**.
- One isolated VLAN can contain **multiple isolated host ports**. Those ports cannot communicate directly with one another.
- Hosts in the **same community VLAN** can communicate directly; hosts in **different communities** cannot.
- Each secondary VLAN can be associated with **only one primary VLAN**.
- The primary VLAN provides access to promiscuous ports and is used for the **Layer 3 SVI** when the switch acts as the gateway.
- All secondary VLANs in a PVLAN domain can use the **same IP subnet** as the primary VLAN.
- The primary VLAN's STP parameters are shared with its associated secondary VLANs.
- `private-vlan association` defines the **VLAN-level relationship**. Port associations and promiscuous mappings are configured separately.
- Use `show vlan private-vlan` to verify the configured primary-to-secondary VLAN associations.

---

## References

- [Cisco Catalyst 9300, Cisco IOS XE 17.12.x: Configuring Private VLANs](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-12/configuration_guide/vlan/b_1712_vlan_9300_cg/configuring_private_vlans.html)
- [Cisco: Configure Isolated Private VLANs on Catalyst Switches](https://www.cisco.com/c/en/us/support/docs/lan-switching/private-vlans-pvlans-promiscuous-isolated-community/40781-194.html)
