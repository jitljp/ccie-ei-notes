# PVLAN Port Types and Forwarding Rules

A **Private VLAN (PVLAN)** restricts Layer 2 communication between switch ports in the same PVLAN domain.

The forwarding behavior depends on the **PVLAN port type** and, for host ports, the **secondary VLAN** assigned to the port.

There are three main PVLAN port types:

- **Promiscuous port:** Can communicate with host ports in its mapped secondary VLANs.
- **Isolated host port:** Can communicate with mapped promiscuous ports, but not with other PVLAN host ports.
- **Community host port:** Can communicate with other host ports in the same community and with mapped promiscuous ports.

An isolated port and a community port are both configured as **PVLAN host ports**. Their different forwarding behavior comes from the **isolated or community VLAN type**, not from different interface modes.

---

## Example PVLAN Topology

Consider the following PVLAN domain:

```text
                          R1 (Default Gateway)
                                   |
                         Promiscuous Port (P)
                                   |
                                  SW1
                           Primary VLAN 100
                +------------------+------------------+
                |                  |                  |
           Isolated 101       Community 102      Community 103
             /     \            /      \                 |
            H1     H2          H3      H4                H5

                 All hosts use subnet 192.0.2.0/24.
                 P maps VLAN 100 to VLANs 101-103.
```

The port assignments are:

| Device | Port type | Primary VLAN | Secondary VLAN |
|---|---|---|---|
| R1 | Promiscuous | 100 | Mapped to 101, 102, 103 |
| H1 | Isolated host | 100 | 101 (isolated) |
| H2 | Isolated host | 100 | 101 (isolated) |
| H3 | Community host | 100 | 102 (community) |
| H4 | Community host | 100 | 102 (community) |
| H5 | Community host | 100 | 103 (community) |

Although these hosts share an IP subnet, they cannot all exchange Ethernet frames directly.

---

## Promiscuous Ports

A **promiscuous port** belongs to the primary VLAN and can communicate with host ports in the secondary VLANs **mapped to that port**.

Promiscuous ports commonly connect to:

- Default gateways and routers.
- Firewalls.
- Load balancers.
- Shared servers or management systems that must reach multiple PVLAN host groups.

In the example topology, P is mapped to all three secondary VLANs:

```text
Promiscuous port P
    Primary VLAN 100
       |
       +-- Isolated VLAN 101
       +-- Community VLAN 102
       +-- Community VLAN 103
```

Therefore, P can communicate with H1, H2, H3, H4, and H5, and all five hosts can communicate with P.

**Promiscuous-to-promiscuous traffic is also permitted** within the same primary VLAN, subject to normal Layer 2 forwarding.

### Promiscuous Port Mapping

A promiscuous port must be mapped to the secondary VLANs it will serve.

For example:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport mode private-vlan promiscuous
SW1(config-if)# switchport private-vlan mapping 100 101-103
```

The mapping associates the port with **primary VLAN 100** and **secondary VLANs 101 through 103**.

**A secondary VLAN's association with the primary VLAN is not the same as mapping that secondary VLAN to a promiscuous port.** Both relationships are required for the corresponding host-to-promiscuous connectivity.

For example, if P is mapped only to secondary VLANs 101 and 102, H5 in community VLAN 103 cannot communicate with P through that mapping, even if VLAN 103 is associated with primary VLAN 100.

A PVLAN domain can have multiple promiscuous ports. Each promiscuous port can have its own set of secondary VLAN mappings.

---

## Isolated Host Ports

An **isolated host port** is a PVLAN host port associated with an **isolated secondary VLAN**.

An isolated host port can:

- Send and receive traffic to or from appropriately mapped promiscuous ports.
- **Not communicate directly with other isolated host ports**, even if they belong to the same isolated secondary VLAN.
- **Not communicate directly with community host ports**, regardless of the community VLAN.

In the example:

```text
H1 (Isolated 101) <----X----> H2 (Isolated 101)

H1 (Isolated 101) <----------> P (Promiscuous)

H1 (Isolated 101) <----X----> H3 (Community 102)
```

The restriction applies to Layer 2 traffic, not just IP traffic.

### Isolated Host Port Configuration

```text
SW1(config)# interface GigabitEthernet1/0/2
SW1(config-if)# switchport mode private-vlan host
SW1(config-if)# switchport private-vlan host-association 100 101
```

The first command sets the interface to **PVLAN host mode**. The second associates it with **primary VLAN 100** and **isolated secondary VLAN 101**.

There is no separate `switchport mode private-vlan isolated` command for a standard PVLAN host port.

---

## Community Host Ports

A **community host port** is a PVLAN host port associated with a **community secondary VLAN**.

A community host port can:

- Communicate directly with other host ports in the **same community secondary VLAN**.
- Communicate with appropriately mapped promiscuous ports.
- **Not communicate directly with host ports in other community VLANs**.
- **Not communicate directly with isolated host ports**.

For example:

```text
H3 (Community 102) <----------> H4 (Community 102)

H3 (Community 102) <----------> P  (Promiscuous)

H3 (Community 102) <----X----> H5 (Community 103)

H3 (Community 102) <----X----> H1 (Isolated 101)
```

Hosts H3 and H4 can exchange Ethernet frames directly because they use **the same community VLAN, 102**. H3 and H5 cannot communicate directly, even though both are community host ports associated with primary VLAN 100.

### Community Host Port Configuration

```text
SW1(config)# interface GigabitEthernet1/0/4
SW1(config-if)# switchport mode private-vlan host
SW1(config-if)# switchport private-vlan host-association 100 102
```

These are **the same interface commands used for an isolated host port**. VLAN 102's configured type (`private-vlan community`) makes the interface a community host port.

---

## PVLAN Layer 2 Forwarding Matrix

The following table describes **direct Layer 2 communication within one primary VLAN domain**, assuming the promiscuous ports have the necessary secondary VLAN mappings.

| Source port | Promiscuous | Isolated host | Same community | Different community |
|---|---|---|---|---|
| **Promiscuous** | Allowed | Allowed | Allowed | Allowed |
| **Isolated host** | Allowed | Blocked | Blocked | Blocked |
| **Community host** | Allowed | Blocked | Allowed | Blocked |

**Same community** means another host port in the **same community secondary VLAN**. It does not mean any port configured as a community host.

Using the example topology:

| Source | Destination | Direct Layer 2 communication? | Reason |
|---|---|---|---|
| H1 | H2 | No | Both are isolated hosts |
| H1 | H3 | No | Isolated and community hosts cannot communicate directly |
| H3 | H4 | Yes | Same community VLAN 102 |
| H3 | H5 | No | Different community VLANs |
| H5 | P | Yes | P is mapped to VLAN 103 |
| P | H1 | Yes | P is mapped to VLAN 101 |

These are **bidirectional communication relationships**, not permissions that apply only to one direction. For example, H3 cannot send directly to H5, and H5 cannot send directly to H3.

> PVLAN port rules are enforced independently of MAC address learning. Learning a destination MAC address does not override the isolation between host ports.

---

## Unicast Forwarding

For **known unicast**, the switch uses its Layer 2 forwarding information but still applies the PVLAN port restrictions.

For example:

```text
H1 -> P:  Allowed
P  -> H1: Allowed
H1 -> H2: Blocked
H3 -> H4: Allowed
H3 -> H5: Blocked
```

Even if SW1 has learned H2's MAC address, it does not forward a unicast frame directly from H1's isolated port to H2's isolated port.

### Unknown Unicast

When a destination MAC address has not been learned, a normal VLAN may flood the frame to other ports in that VLAN. PVLANs restrict **which ports may receive that flooded traffic**.

- **From an isolated host:** Unknown unicast can be forwarded toward appropriately mapped promiscuous ports, not to other host ports.
- **From a community host:** Unknown unicast can be forwarded toward mapped promiscuous ports and other ports in the same community.
- **From a promiscuous port:** Unknown unicast can be forwarded to the associated PVLAN host ports and other eligible promiscuous ports.

Where a PVLAN spans switches, forwarding toward eligible trunk links must preserve the PVLAN restrictions. A trunk is not permission for isolated hosts to communicate with one another.

---

## Broadcast Forwarding

PVLAN isolation also applies to **broadcast traffic**, including ARP Requests.

Unlike a regular VLAN, a broadcast is **not automatically delivered to every port in the PVLAN domain**.

| Broadcast source | Eligible local recipients |
|---|---|
| **Promiscuous port** | Associated isolated and community hosts, and other eligible promiscuous ports |
| **Isolated host port** | Mapped promiscuous ports only |
| **Community host port** | Mapped promiscuous ports and other host ports in the same community |

Eligible inter-switch trunk links may also carry these broadcasts to their permitted destinations.

### Example: ARP from an Isolated Host

Suppose H1 and H2 are in isolated VLAN 101 and use these addresses:

```text
R1: 192.0.2.1
H1: 192.0.2.11
H2: 192.0.2.12
```

H1 sends an ARP Request for the default gateway:

```text
H1 -> Broadcast: Who has 192.0.2.1?
                       |
                       v
                      SW1
                       |
                       +---> P / R1   (forwarded)
                       |
                       X----> H2      (blocked)
```

R1 receives the request and can send the ARP Reply back to H1 through the promiscuous port.

Now suppose H1 sends an ARP Request for **H2's address**, `192.0.2.12`. H2 does not receive the request, even though H1 and H2 use the same IP subnet and the same isolated secondary VLAN.

**The isolated host is not permitted to send broadcasts directly to another isolated host.**

### Example: ARP from a Community Host

If H3 broadcasts an ARP Request:

```text
H3 (Community 102)
       |
       v
      SW1
       +---> H4 (Community 102)  Forwarded
       +---> P  (Promiscuous)   Forwarded
       X----> H5 (Community 103) Blocked
       X----> H1 (Isolated 101)  Blocked
```

The same PVLAN separation applies even though all the devices use addresses in the same IP subnet.

---

## Multicast Forwarding

Multicast forwarding is also constrained by the PVLAN port rules.

- An **isolated host** cannot send multicast traffic directly to other PVLAN host ports.
- A **community host** can send multicast traffic to eligible receivers in the same community and toward mapped promiscuous ports.
- A **promiscuous port** can send multicast traffic toward host ports in its mapped secondary VLANs.

Multicast is **not forwarded directly between isolated host ports or between different secondary VLANs**.

IGMP snooping and other multicast features can further determine which eligible receivers actually receive traffic. They do not remove the PVLAN isolation restrictions.

---

## PVLAN Isolation and Layer 3 Forwarding

PVLAN restrictions control **direct Layer 2 forwarding**. They do not inherently prevent an external router or firewall connected to a promiscuous port from forwarding traffic between hosts.

For example:

```text
H1 (Isolated 101)
        |
        v
       SW1
        |
        v
       R1 (Promiscuous)
        |
        v
       SW1
        |
        v
H2 (Isolated 101)
```

The direct **H1-to-H2 Layer 2 path** is blocked. However, a Layer 3 device might permit traffic to pass between them if it provides a suitable forwarding mechanism and return path.

Because H1 and H2 share an IP subnet, H1 ordinarily treats H2 as an **on-link destination** and sends an ARP Request for H2 rather than sending traffic to its default gateway. That request is blocked by PVLAN isolation.

Therefore, routed communication between same-subnet isolated hosts requires additional behavior, such as an appropriate proxy ARP mechanism or another design that directs the traffic through the Layer 3 device. PVLAN configuration alone does not make that routed path work.

See [Layer 3 Connectivity with Private VLANs](layer3.md) for gateway and routing details.

---

## PVLAN Port Types vs. Protected Ports

The **`switchport protected`** feature also restricts Layer 2 communication, but its behavior differs from PVLAN host isolation.

| Feature | Protected ports (`switchport protected`) | PVLAN host ports |
|---|---|---|
| Same-switch isolation | Protected ports cannot communicate directly with one another | Determined by isolated/community rules |
| Communication groups | No community groups | Multiple community VLANs are supported |
| Scope across switches | Protection is local to the switch | PVLAN isolation can extend across multiple switches |
| VLAN configuration | No primary/secondary VLAN associations | Primary/secondary VLANs and mappings are required |
| Shared gateway access | Through non-protected ports | Through mapped promiscuous ports |

For example, two ports configured with `switchport protected` on **different switches** are not automatically isolated from each other by that feature. With PVLANs, isolated host ports can remain isolated across switches when the primary and secondary VLANs and trunking are configured correctly.

---

## Port Configuration Summary

The following examples assume VLAN 100 is the primary VLAN, VLAN 101 is isolated, and VLANs 102 and 103 are communities. The VLAN types and primary-to-secondary associations must already be configured.

### Isolated Host Port

```text
SW1(config)# interface GigabitEthernet1/0/2
SW1(config-if)# switchport mode private-vlan host
SW1(config-if)# switchport private-vlan host-association 100 101
```

### Community Host Port

```text
SW1(config)# interface GigabitEthernet1/0/4
SW1(config-if)# switchport mode private-vlan host
SW1(config-if)# switchport private-vlan host-association 100 102
```

### Promiscuous Port

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport mode private-vlan promiscuous
SW1(config-if)# switchport private-vlan mapping 100 101-103
```

| Interface command | Purpose |
|---|---|
| `switchport mode private-vlan host` | Configures a PVLAN host port (isolated or community) |
| `switchport private-vlan host-association <primary> <secondary>` | Selects the host port's primary/secondary VLAN pair |
| `switchport mode private-vlan promiscuous` | Configures a promiscuous port |
| `switchport private-vlan mapping <primary> <secondary-list>` | Maps the promiscuous port to selected secondary VLANs |

A **promiscuous PVLAN trunk** uses different configuration commands and is covered in [Private VLAN Trunks](trunks.md).

### Verify the Port Mode

```text
SW1# show vlan private-vlan
SW1# show interfaces GigabitEthernet1/0/2 switchport
SW1# show interfaces GigabitEthernet1/0/1 switchport
```

These commands help verify the VLAN roles, host associations, and promiscuous mappings. Additional verification examples are provided in [Verification and Troubleshooting](verification.md).

---

## Key Points

- PVLAN ports have three principal roles: **promiscuous**, **isolated host**, and **community host**.
- **Promiscuous ports** can communicate with host ports in their **mapped secondary VLANs** and with other promiscuous ports in the same primary VLAN.
- **Isolated host ports** can communicate directly only with mapped promiscuous ports, not with other PVLAN host ports.
- **Community host ports** can communicate with mapped promiscuous ports and with **other host ports in the same community VLAN**.
- Two different community VLANs cannot communicate directly at Layer 2.
- Both isolated and community ports use **`switchport mode private-vlan host`**. The secondary VLAN type determines their behavior.
- **Known unicast, unknown unicast, broadcast, and multicast** traffic are subject to PVLAN forwarding restrictions.
- A broadcast from an isolated host does **not** reach another isolated host, even within the same isolated VLAN.
- A broadcast from a promiscuous port can reach host ports in mapped secondary VLANs.
- PVLAN isolation can be maintained across switches, unlike the local isolation provided by `switchport protected`.
- **PVLANs restrict direct Layer 2 forwarding**, but they do not automatically prohibit traffic forwarded by a Layer 3 device.

---

## References

- [Cisco Catalyst 9300, VLAN Configuration Guide: Configuring Private VLANs](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-14/configuration_guide/vlan/b_1714_vlan_9300_cg/configuring_private_vlans.html)
- [Cisco: Configure Isolated Private VLANs on Catalyst Switches](https://www.cisco.com/c/en/us/support/docs/lan-switching/private-vlans-pvlans-promiscuous-isolated-community/40781-194.html)
