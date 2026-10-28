# Private VLAN Trunks

**Private VLANs (PVLANs)** can extend across multiple switches using IEEE 802.1Q trunks. Cisco Catalyst switches also support specialized **isolated PVLAN trunks** and **promiscuous PVLAN trunks** for connecting devices that do not themselves support PVLANs.

These three trunk types have different purposes:

| Trunk type | Typical connection | Behavior |
|---|---|---|
| **Standard 802.1Q trunk** | Between PVLAN-aware switches | Carries the primary VLAN and all required secondary VLANs without changing the VLAN IDs in the tags |
| **Isolated PVLAN trunk** | Toward a downstream, non-PVLAN-aware switch or device | Extends the behavior of an isolated host port across a trunk, carrying isolated-side traffic with its **secondary VLAN ID** |
| **Promiscuous PVLAN trunk** | Toward an upstream router, firewall, or shared-service device | Extends the behavior of a promiscuous port across a trunk, carrying PVLAN-domain traffic with its **primary VLAN ID** |

A **PVLAN trunk is not the same as a promiscuous port**. The word *trunk* refers to transporting VLAN traffic, whereas *promiscuous* describes the forwarding permissions for traffic in a PVLAN domain.

For the underlying primary/secondary VLAN relationships, see [Primary and Secondary VLANs](vlan-types.md). For host-port forwarding restrictions, see [PVLAN Port Types and Forwarding Rules](port-types.md).

---

## VLANs Used in the Examples

The examples use this PVLAN domain:

| VLAN | Type | Purpose |
|---|---|---|
| 100 | Primary | PVLAN domain and gateway-facing VLAN |
| 101 | Isolated secondary | Hosts cannot communicate directly with other PVLAN hosts |
| 102 | Community secondary | Hosts in community 102 can communicate with each other |
| 103 | Community secondary | Hosts in community 103 can communicate with each other, but not with community 102 |

All hosts use the **192.0.2.0/24** subnet, with a gateway at **192.0.2.1**.

Where a switch needs to create the PVLAN domain locally, the configuration is:

```text
SW1(config)# vlan 100
SW1(config-vlan)# private-vlan primary
SW1(config-vlan)# exit
SW1(config)# vlan 101
SW1(config-vlan)# private-vlan isolated
SW1(config-vlan)# exit
SW1(config)# vlan 102
SW1(config-vlan)# private-vlan community
SW1(config-vlan)# exit
SW1(config)# vlan 103
SW1(config-vlan)# private-vlan community
SW1(config-vlan)# exit
SW1(config)# vlan 100
SW1(config-vlan)# private-vlan association 101-103
```

The primary VLAN must be associated with its secondary VLANs. The VLAN definitions alone do not establish host-port or promiscuous-port mappings.

---

## Standard 802.1Q Trunks Between PVLAN-Aware Switches

A normal **802.1Q trunk** can carry an entire PVLAN domain between two PVLAN-aware switches.

```text
                          R1 (192.0.2.1)
                                |
                       Gi1/0/23 (promiscuous)
                                |
                               SW1
                          Primary 100
                      _________|_________
                     |         |         |
                  Gi1/0/1   Gi1/0/24  Gi1/0/2
                     |         |         |
                 H1 (101)      |      H3 (102)
                 Isolated      |      Community 102
                               |
                               | Standard 802.1Q trunk
                               | VLANs 100, 101, 102, 103
                               |
                            Gi1/0/24
                               |
                              SW2
                       ________|________
                      |        |        |
                   Gi1/0/1  Gi1/0/2  Gi1/0/3
                      |        |        |
                  H2 (101)  H4 (102)  H5 (103)
                  Isolated  Community Community
                             102       103
```

SW1 and SW2 both understand the PVLAN relationships:

```text
Primary VLAN:   100
Isolated VLAN:  101
Community VLANs: 102, 103
```

Even though H1 and H2 are attached to different switches, **they cannot communicate directly at Layer 2**. The PVLAN isolation rules remain in force across the trunk.

**H3 is connected to SW1 and H4 is connected to SW2, but both are in community VLAN 102.** They can communicate directly at Layer 2 across the standard trunk.

H3 and H5 cannot communicate directly at Layer 2 because H5 belongs to a different community (VLAN 103). H1 and H2 cannot communicate directly because both use the isolated secondary VLAN (101).

| Host | Switch | Interface | Secondary VLAN | IP address |
|---|---|---|---|---|
| H1 | SW1 | Gi1/0/1 | 101 (isolated) | 192.0.2.11/24 |
| H2 | SW2 | Gi1/0/1 | 101 (isolated) | 192.0.2.12/24 |
| H3 | SW1 | Gi1/0/2 | 102 (community) | 192.0.2.13/24 |
| H4 | SW2 | Gi1/0/2 | 102 (community) | 192.0.2.14/24 |
| H5 | SW2 | Gi1/0/3 | 103 (community) | 192.0.2.15/24 |

All hosts use 192.0.2.1 as their default gateway.

### Trunk Configuration

Configure SW1's inter-switch interface:

```text
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 100-103
```

Configure the corresponding SW2 interface:

```text
SW2(config)# interface GigabitEthernet1/0/24
SW2(config-if)# switchport mode trunk
SW2(config-if)# switchport trunk allowed vlan 100-103
```

**These are ordinary trunk commands**, not `switchport mode private-vlan trunk` commands.

The allowed VLAN list includes both the **primary VLAN** and the **secondary VLANs** required across the link.

### Configure the PVLAN Domain on Both Switches

An ordinary trunk does not transmit the PVLAN's role definitions or primary-to-secondary associations as part of 802.1Q framing.

With VTP transparent mode, configure the same VLAN roles and associations on both switches:

```text
SW2(config)# vtp mode transparent
SW2(config)# vlan 100
SW2(config-vlan)# private-vlan primary
SW2(config-vlan)# exit
SW2(config)# vlan 101
SW2(config-vlan)# private-vlan isolated
SW2(config-vlan)# exit
SW2(config)# vlan 102
SW2(config-vlan)# private-vlan community
SW2(config-vlan)# exit
SW2(config)# vlan 103
SW2(config-vlan)# private-vlan community
SW2(config-vlan)# exit
SW2(config)# vlan 100
SW2(config-vlan)# private-vlan association 101-103
```

Configure the **SW1 host ports** for H1 (isolated 101) and H3 (community 102):

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport mode private-vlan host
SW1(config-if)# switchport private-vlan host-association 100 101
SW1(config-if)# exit
SW1(config)# interface GigabitEthernet1/0/2
SW1(config-if)# switchport mode private-vlan host
SW1(config-if)# switchport private-vlan host-association 100 102
```

Configure the **SW2 host ports** for H2 (isolated 101), H4 (community 102), and H5 (community 103):

```text
SW2(config)# interface GigabitEthernet1/0/1
SW2(config-if)# switchport mode private-vlan host
SW2(config-if)# switchport private-vlan host-association 100 101
SW2(config-if)# exit
SW2(config)# interface GigabitEthernet1/0/2
SW2(config-if)# switchport mode private-vlan host
SW2(config-if)# switchport private-vlan host-association 100 102
SW2(config-if)# exit
SW2(config)# interface GigabitEthernet1/0/3
SW2(config-if)# switchport mode private-vlan host
SW2(config-if)# switchport private-vlan host-association 100 103
```

SW1 must also have the VLAN roles/associations from the earlier configuration, and its gateway-facing port must be mapped as promiscuous:

```text
SW1(config)# interface GigabitEthernet1/0/23
SW1(config-if)# switchport mode private-vlan promiscuous
SW1(config-if)# switchport private-vlan mapping 100 101-103
```

**H3 and H4 use the same `host-association 100 102` configuration even though they are on different switches.** This is what allows them to communicate within community 102 across the trunk.

**VTPv1/v2 do not propagate PVLAN relationships.** VTPv3 supports propagation of PVLAN information, but the VLANs and associations still need to be consistent throughout the PVLAN domain.

### VLAN Tagging on a Standard Trunk

A standard trunk does **not** translate secondary VLAN IDs into the primary VLAN ID simply because the VLANs are part of a PVLAN.

The tag identifies the VLAN to which the forwarded frame belongs:

| Traffic | VLAN tag on the standard trunk |
|---|---|
| Isolated host traffic toward a gateway on another switch | Isolated secondary VLAN **101** |
| Community 102 host traffic toward a gateway on another switch | Community secondary VLAN **102** |
| H3 (SW1, community 102) to H4 (SW2, community 102) | Community secondary VLAN **102** |
| Promiscuous gateway traffic toward hosts on another switch | Primary VLAN **100** |

The receiving PVLAN-aware switch uses its primary/secondary VLAN associations to apply the correct forwarding restrictions.

For example:

```text
H2 (SW2, isolated 101)
   |
   | Frame enters SW2 as isolated VLAN 101
   v
SW2 -- [802.1Q VLAN 101] --> SW1
                                |
                                | Allowed toward promiscuous port
                                v
                               R1
```

The reverse direction is:

```text
R1 --> SW1 -- [802.1Q VLAN 100] --> SW2 --> H2
```

The gateway-facing traffic uses the **primary VLAN**, while host-originated traffic uses the corresponding **secondary VLAN**.

For direct communication between H3 and H4, the frame stays in **secondary VLAN 102** on the inter-switch trunk:

```text
H3 (SW1, community 102)
   |
   v
SW1 -- [802.1Q VLAN 102] --> SW2 --> H4 (community 102)
```

This traffic is bridged at Layer 2 and does not need to pass through R1. If H3 instead sends a frame directly to H5 (community 103), the switches do not bridge that traffic between the communities.

### Isolation Must Be Preserved End to End

PVLAN isolation is preserved across ordinary trunks when the switches along the path apply compatible PVLAN definitions and forwarding rules.

If an intermediate switch treats the secondary VLANs as ordinary VLANs, it may forward traffic that a PVLAN-aware switch would have blocked.

Therefore, configure the PVLAN relationships on **intermediate switches**, even if they have no directly connected PVLAN host ports.

---

## Isolated PVLAN Trunks

An **isolated PVLAN trunk** allows a port to behave like an isolated host connection while carrying 802.1Q-tagged traffic.

A typical use is connecting a PVLAN-aware Catalyst switch to a downstream switch that **does not support PVLANs**.

```text
                          R1
                           |
                       Promiscuous
                           |
                          SW1
                    Primary VLAN 100
                      /         \
             H3 (isolated)     Gi1/0/23
                VLAN 101           |
                                   | Isolated PVLAN trunk
                                   | VLAN 101 tags on wire
                                   |
                                 Gi0/1
                                  SW2
                           (normal VLANs)
                              /      \
                        Gi0/2        Gi0/3
                          |            |
                         H1           H2
                        VLAN 101     VLAN 101
```

SW1 knows that VLAN 101 is the **isolated secondary VLAN** associated with primary VLAN 100.

SW2 is a regular Layer 2 switch that does not understand primary and secondary PVLAN relationships.

### Configure the Isolated PVLAN Trunk on SW1

Assuming primary VLAN 100 and isolated VLAN 101 have already been created and associated:

```text
SW1(config)# interface GigabitEthernet1/0/23
SW1(config-if)# switchport mode private-vlan trunk secondary
SW1(config-if)# switchport private-vlan association trunk 100 101
SW1(config-if)# switchport private-vlan trunk allowed vlan 100
```

The commands have distinct purposes:

| Command | Purpose |
|---|---|
| `switchport mode private-vlan trunk secondary` | Configures an isolated PVLAN trunk |
| `switchport private-vlan association trunk 100 101` | Associates primary VLAN 100 and isolated secondary VLAN 101 on this trunk |
| `switchport private-vlan trunk allowed vlan 100` | Allows primary VLAN 100 as a PVLAN domain on this trunk |

On supported Catalyst platforms, `switchport mode private-vlan trunk` without `secondary` is also used to select the isolated trunk mode.

**Important:** The special `switchport private-vlan trunk allowed vlan` command uses the **primary VLAN** to identify a PVLAN domain. It is not the same as a normal trunk's `switchport trunk allowed vlan` list, where the secondary VLAN IDs must also be permitted.

### Configure the Non-PVLAN Switch

SW2 treats VLAN 101 as an ordinary VLAN:

```text
SW2(config)# vlan 101
SW2(config-vlan)# exit
SW2(config)# interface GigabitEthernet0/1
SW2(config-if)# switchport mode trunk
SW2(config-if)# switchport trunk allowed vlan 101
SW2(config-if)# exit
SW2(config)# interface range GigabitEthernet0/2-3
SW2(config-if-range)# switchport mode access
SW2(config-if-range)# switchport access vlan 101
```

The link uses 802.1Q tagging, even though one side is an **isolated PVLAN trunk** and the other is an **ordinary trunk**.


### Isolated Trunk Frame Handling

Frames from H1 or H2 arrive at SW1 tagged with **secondary VLAN 101**.

SW1 applies the corresponding isolated-host forwarding restrictions and allows the frames toward mapped promiscuous destinations.

In the opposite direction, frames received from a promiscuous port in primary VLAN 100 are sent toward SW2 using the **isolated secondary VLAN 101** tag.

```text
Upstream:
H1 -> SW2 -- [VLAN 101] --> SW1 -> R1

Downstream:
R1 -> SW1 -- [VLAN 101] --> SW2 -> H1
```

This is different from a **standard trunk between two PVLAN-aware switches**, on which downstream gateway traffic is carried using the primary VLAN tag.

### Traffic from H3

H3 is connected directly to an **isolated host port** on SW1, associated with primary VLAN 100 and isolated VLAN 101.

Consider two scenarios.

**Scenario 1: H3 sends a frame toward R1**

1. H3 sends an untagged Ethernet frame to SW1.
2. SW1 classifies the frame as belonging to **isolated VLAN 101**.
3. SW1 permits the frame to be forwarded to the mapped **promiscuous port** connected to R1.
4. R1 receives the frame.

```text
H3 (101) --> SW1 --> R1
               |
               X
         Isolated trunk
               |
              SW2
```

SW1 does not forward H3's frame toward SW2 because the isolated trunk is treated as an **isolated-side connection**, not a promiscuous destination.

**Scenario 2: H3 sends a frame toward H1 or H2**

1. H3 sends an Ethernet frame to SW1.
2. SW1 classifies the frame as belonging to **isolated VLAN 101**.
3. SW1 determines that the destination is reachable only through another isolated-side connection.
4. SW1 blocks forwarding toward the isolated PVLAN trunk.

```text
H3 (101) --> SW1 --X-- [VLAN 101] --> SW2 --> H1/H2
```

The restriction also applies to broadcast traffic. For example, an **ARP Request from H3 for H1** is not forwarded across the isolated trunk.

Therefore:

- **H3 → R1:** Allowed.
- **H3 → H1/H2:** Blocked.
- **H1/H2 → R1:** Allowed.
- **R1 → H3/H1/H2:** Allowed.

**The isolated PVLAN trunk effectively behaves like another isolated host connection, while transporting frames with 802.1Q tags.**

---

## Promiscuous PVLAN Trunks

A **promiscuous PVLAN trunk** extends the behavior of a promiscuous port to a tagged trunk.

It is commonly used to connect a PVLAN-aware switch to an upstream router or firewall that needs to receive traffic from **multiple PVLAN domains** or additional ordinary VLANs.

```text
                      R1 (Router)
                  Gi0/0.100 / 802.1Q 100
                           |
                           | Promiscuous PVLAN trunk
                           | PVLAN traffic tagged VLAN 100
                           |
                       Gi1/0/24
                          SW1
                    Primary VLAN 100
                  /        |         \
              H1 (101)  H2 (102)   H3 (103)
              Isolated   Community  Community
```

R1 does **not** need to understand isolated or community VLANs. It sees the shared PVLAN subnet through **primary VLAN 100**.

### Configure the Promiscuous PVLAN Trunk on SW1

Assuming VLAN 100 is the primary and VLANs 101-103 are associated secondary VLANs:

```text
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# switchport mode private-vlan trunk promiscuous
SW1(config-if)# switchport private-vlan mapping trunk 100 101-103
SW1(config-if)# switchport private-vlan trunk allowed vlan 100
```

| Command | Purpose |
|---|---|
| `switchport mode private-vlan trunk promiscuous` | Configures a promiscuous PVLAN trunk |
| `switchport private-vlan mapping trunk 100 101-103` | Maps secondary VLANs 101-103 to primary VLAN 100 on the trunk |
| `switchport private-vlan trunk allowed vlan 100` | Allows the primary VLAN 100 PVLAN domain on the trunk |

The **`mapping trunk`** command is required to establish the appropriate PVLAN forwarding relationships. Merely allowing VLAN 100 is not enough.

### Configure the Router

R1 can use an ordinary 802.1Q subinterface:

```text
R1(config)# interface GigabitEthernet0/0
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# interface GigabitEthernet0/0.100
R1(config-subif)# encapsulation dot1Q 100
R1(config-subif)# ip address 192.0.2.1 255.255.255.0
```

All PVLAN hosts in this example use **192.0.2.1** as their default gateway.

The router does not require separate subinterfaces for VLANs 101, 102, and 103 because the promiscuous PVLAN trunk presents this PVLAN domain using VLAN **100**.

### Promiscuous Trunk Frame Handling

Consider H1, which belongs to isolated VLAN 101.

```text
Upstream:
H1 (101) -> SW1 -- [802.1Q VLAN 100] --> R1

Downstream:
R1 -- [802.1Q VLAN 100] --> SW1 -> H1 (101)
```

The switch uses the mapping `100 101-103` to bridge between the secondary host VLANs and the primary VLAN on the promiscuous trunk.

**The 802.1Q tag on this trunk is the primary VLAN ID, not the originating host's secondary VLAN ID.**

### Multiple PVLAN Domains and Normal VLANs

A promiscuous trunk can serve more than one PVLAN domain and may also carry regular VLANs.

For example, suppose another PVLAN domain uses primary VLAN 200 and secondary VLAN 201, while VLAN 300 is an ordinary VLAN.

The trunk can be configured with separate PVLAN mappings:

```text
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# switchport mode private-vlan trunk promiscuous
SW1(config-if)# switchport private-vlan mapping trunk 100 101-103
SW1(config-if)# switchport private-vlan mapping trunk 200 201
SW1(config-if)# switchport private-vlan trunk allowed vlan 100,200,300
```

This example assumes that all VLANs and both primary/secondary associations have already been configured.

The trunk carries the **primary VLAN IDs 100 and 200**, plus the **normal VLAN 300**, rather than requiring VLANs 101-103 and 201 in its special allowed list.

### Adding or Removing Trunk Mappings

The `add` and `remove` keywords let you adjust the secondary VLANs mapped to a promiscuous trunk:

```text
SW1(config-if)# switchport private-vlan mapping trunk 100 add 103
SW1(config-if)# switchport private-vlan mapping trunk 100 remove 103
```

These commands affect the **interface mapping**, not the global association configured under `vlan 100`.

---

## Trunk Modes and 802.1Q Tags Compared

For the same PVLAN domain (primary 100, isolated 101, community 102), the VLAN tags differ depending on which trunk is used.

| Connection | Host-to-gateway traffic | Gateway-to-host traffic |
|---|---|---|
| Standard trunk between PVLAN-aware switches | Secondary VLAN tag (101 or 102) | Primary VLAN tag (100) |
| Isolated PVLAN trunk toward non-PVLAN switch | Isolated VLAN tag (101) | Isolated VLAN tag (101) |
| Promiscuous PVLAN trunk toward router | Primary VLAN tag (100) | Primary VLAN tag (100) |

The standard-trunk rows describe traffic crossing the inter-switch link between the host-facing switch and the gateway-facing switch. The special trunk rows describe traffic on the corresponding non-PVLAN device link.

In short:

- **Standard trunk:** Preserves primary and secondary VLAN information across PVLAN-aware switches.
- **Isolated PVLAN trunk:** Presents the downstream connection as an isolated secondary VLAN.
- **Promiscuous PVLAN trunk:** Presents the upstream connection as the primary VLAN.

---

## Allowed VLANs and Native VLANs

Cisco IOS XE uses **different interface commands** for ordinary trunks and special PVLAN trunks.

| Function | Standard trunk | Special PVLAN trunk |
|---|---|---|
| Mode | `switchport mode trunk` | `switchport mode private-vlan trunk secondary` or `... trunk promiscuous` |
| Allowed VLANs | `switchport trunk allowed vlan ...` | `switchport private-vlan trunk allowed vlan ...` |
| Native VLAN | `switchport trunk native vlan ...` | `switchport private-vlan trunk native vlan ...` |
| Isolated-side association | Not an interface command | `switchport private-vlan association trunk ...` |
| Promiscuous-side mapping | Not an interface command | `switchport private-vlan mapping trunk ...` |

On **standard trunks**, allow the required primary and secondary VLANs.

On **special PVLAN trunks**, Cisco documents the `private-vlan trunk allowed vlan` command using the **primary VLANs** of the PVLAN domains, plus any regular VLANs that must traverse the link. The corresponding secondary VLANs are selected using the isolated-trunk association or promiscuous-trunk mapping.

The native VLAN for a special PVLAN trunk can be configured with:

```text
SW1(config-if)# switchport private-vlan trunk native vlan <vlan-id>
```

The normal 802.1Q native-VLAN concept still applies: frames may be transmitted without VLAN tags, depending on the native VLAN and tagging configuration. Both ends of the link must interpret tagged and untagged traffic consistently.

In the examples above, the native VLAN is left at the default and the PVLAN data uses explicit tags.

---

## STP and VTP Considerations

### Spanning Tree

Cisco Catalyst PVLANs use the **primary VLAN's STP topology** for the associated secondary VLANs.

When a secondary VLAN is associated with a primary VLAN, the primary VLAN's STP parameters are propagated to it. A single STP instance applies to that PVLAN domain.

For example:

```text
Primary VLAN 100
  +-- Isolated 101
  +-- Community 102
  +-- Community 103

STP topology determined by VLAN 100
```

A standard trunk carrying the PVLAN must participate in the correct STP topology. A PVLAN configuration does not override an STP-blocking port.

### VTP

- **VTPv1/v2:** PVLAN configuration requires transparent mode; PVLAN roles and associations are not propagated through VTP.
- **VTPv3:** Supports PVLAN configuration propagation and PVLAN operation in the appropriate server/client/transparent modes.
- On a manually configured multi-switch PVLAN, keep the primary/secondary VLAN associations consistent on the switches carrying the PVLAN.

### EtherChannel

Some Catalyst platforms support **isolated or promiscuous PVLAN trunk modes on EtherChannels**, but support and configuration restrictions differ by switch family and release.

Do not assume every PVLAN host port or every special PVLAN trunk can be added to an EtherChannel. For an EtherChannel design, check the target platform's PVLAN and port-channel restrictions.

---

## Verification

### Check Primary and Secondary VLAN Associations

```text
SW1# show vlan private-vlan
```

Confirm that VLAN 100 is associated with the expected isolated and community VLANs.

### Check Standard Trunking

```text
SW1# show interfaces trunk
SW1# show interfaces GigabitEthernet1/0/24 switchport
```

For the **standard trunk** scenario, verify that the primary VLAN and all required secondary VLANs are present in the allowed VLAN list and can forward over the link.

### Check PVLAN Trunk Mode and Mappings

```text
SW1# show interfaces GigabitEthernet1/0/24 switchport
SW1# show running-config interface GigabitEthernet1/0/24
```

For an **isolated PVLAN trunk**, look for fields such as:

```text
Administrative Mode: private-vlan trunk secondary
Administrative private-vlan trunk associations:
  100 ... 101 ...
```

For a **promiscuous PVLAN trunk**, look for fields such as:

```text
Administrative Mode: private-vlan trunk promiscuous
Administrative private-vlan trunk mappings:
  100 ... 101 ...
  102 ...
  103 ...
```

The excerpts are illustrative; the exact display format varies by platform.

### Check Spanning Tree

```text
SW1# show spanning-tree vlan 100
```

Verify that the required trunks are forwarding in the primary VLAN's spanning-tree topology.

### Test Forwarding Behavior

For a standard trunk between two PVLAN-aware switches, test:

| Test | Expected result |
|---|---|
| H1 (SW1, isolated 101) to H2 (SW2, isolated 101) | Blocked at Layer 2 |
| H3 (SW1, community 102) to H4 (SW2, community 102) | Allowed across the standard trunk |
| H3 (SW1, community 102) to H5 (SW2, community 103) | Blocked at Layer 2 |
| H1, H2, H3, H4, or H5 to R1 (mapped promiscuous gateway) | Allowed |

For an isolated PVLAN trunk to a non-PVLAN switch, additionally test **host-to-host connectivity on that downstream switch**. Those hosts may communicate locally unless the downstream switch enforces its own isolation.

For a promiscuous PVLAN trunk, verify that the upstream router receives traffic in the **primary VLAN**, not the secondary VLANs.

---

## Common Troubleshooting Problems

| Symptom | Likely cause | Check |
|---|---|---|
| Hosts on different PVLAN switches cannot reach the gateway | A required VLAN is missing from a standard trunk | `show interfaces trunk` on each inter-switch link |
| Isolated hosts communicate unexpectedly across switches | An intermediate switch lacks consistent PVLAN configuration | PVLAN definitions and trunk path on every switch |
| Isolated trunk does not forward as expected | Missing `association trunk` or incorrect special allowed VLAN list | `show interfaces ... switchport` |
| Gateway does not receive traffic through a promiscuous trunk | Missing `mapping trunk`, primary VLAN not allowed, or router subinterface uses the wrong VLAN ID | Switchport mappings and router 802.1Q encapsulation |
| Hosts behind a non-PVLAN downstream switch can reach each other | That downstream switch uses ordinary VLAN bridging | Configure protected ports or another isolation feature there |
| One part of the PVLAN domain is unreachable | An STP-blocking port, VLAN pruning, or mismatched VLAN definitions | STP state, trunk allowed VLANs, and PVLAN associations |

**Do not confuse a VLAN's global association with its per-interface association or mapping.** A correct `private-vlan association` under VLAN configuration does not replace `switchport private-vlan association trunk` or `switchport private-vlan mapping trunk` on a specialized trunk.

---

## Key Points

- **Standard 802.1Q trunks** extend a PVLAN across PVLAN-aware switches and carry the required primary and secondary VLANs.
- Hosts in the **same community VLAN can communicate across different switches** when a standard trunk carries that secondary VLAN and the switches have consistent PVLAN associations.
- A normal trunk uses `switchport mode trunk` and `switchport trunk allowed vlan`.
- PVLAN roles and primary/secondary associations must be consistent along the inter-switch path to preserve isolation.
- **Isolated PVLAN trunks** connect isolated-side traffic over 802.1Q, often to a downstream switch without PVLAN support.
- Configure an isolated PVLAN trunk with `switchport mode private-vlan trunk secondary` and `switchport private-vlan association trunk <primary> <isolated-secondary>`.
- Traffic to and from a non-PVLAN switch on an isolated trunk uses the **isolated secondary VLAN tag**.
- An isolated PVLAN trunk **does not isolate hosts from one another inside an ordinary downstream switch**.
- **Promiscuous PVLAN trunks** connect gateway-side traffic over 802.1Q, often to a router or firewall without PVLAN support.
- Configure a promiscuous PVLAN trunk with `switchport mode private-vlan trunk promiscuous` and `switchport private-vlan mapping trunk <primary> <secondary-list>`.
- Traffic to and from a router on a promiscuous trunk uses the **primary VLAN tag**.
- Special PVLAN trunks use `switchport private-vlan trunk allowed vlan` and `switchport private-vlan trunk native vlan`, not the ordinary trunk versions of those commands.
- For the special allowed-VLAN list, include the **primary VLANs** and any additional normal VLANs; the secondary relationships are specified by association or mapping commands.
- The **primary VLAN determines the STP topology** for its associated secondary VLANs.

---

## References

- [Cisco Catalyst 9300, IOS XE 17.18.x: Configuring Private VLANs](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-18/configuration_guide/vlan/b_1718_vlan_9300_cg/configuring_private_vlans.html)
- [Cisco Catalyst 9500, IOS XE 17.7.x: Configuring Private VLANs](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9500/software/release/17-7/configuration_guide/vlan/b_177_vlan_9500_cg/configuring_private_vlans.html)
- [Cisco: Configure Isolated Private VLANs on Catalyst Switches](https://www.cisco.com/c/en/us/support/docs/lan-switching/private-vlans-pvlans-promiscuous-isolated-community/40781-194.html)
