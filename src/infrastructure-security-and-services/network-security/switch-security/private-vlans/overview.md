# Private VLANs

A **Private VLAN (PVLAN)** divides a Layer 2 network into smaller groups with different communication permissions.

In an ordinary VLAN, switch ports in the same VLAN can generally communicate directly at Layer 2. A private VLAN introduces **port isolation within a VLAN domain**, restricting which hosts can exchange frames even when they share the same IP subnet.

Private VLANs are commonly used in environments where many hosts need access to a shared gateway or common services but should not be able to communicate directly with every other host.

Examples include:

- Server networks in which individual servers should be isolated from one another.
- Hosting environments where different customers share an IP subnet.
- DMZs where servers need access to a firewall but should not communicate directly.
- Networks where selected groups of hosts need to communicate internally while remaining isolated from other groups.

---

## Why Use Private VLANs?

Consider four hosts connected to a switch in VLAN 100:

```text
                    R1
                     |
                    SW1
                 VLAN 100
              /   /   \   \
             H1  H2   H3  H4
```

By default, all four hosts can communicate directly at Layer 2. They can send unicast, broadcast, and multicast frames to one another, subject to any other configured restrictions.

One way to isolate the hosts is to place them in separate VLANs. However, this normally requires separate IP subnets and Layer 3 gateway interfaces.

A **private VLAN** provides a different solution:

- The hosts can use addresses from **one shared IP subnet**.
- They can access a common default gateway.
- Layer 2 communication between hosts is restricted according to their **PVLAN port types and secondary VLAN membership**.

A private VLAN therefore offers more granular Layer 2 isolation without requiring a separate IP subnet for every isolated host or group.

---

## Primary and Secondary VLANs

A private VLAN consists of one **primary VLAN** and one or more **secondary VLANs**.

| VLAN type | Purpose |
|---|---|
| **Primary VLAN** | Defines the overall PVLAN domain and provides connectivity toward promiscuous ports and Layer 3 gateways |
| **Isolated secondary VLAN** | Prevents its host ports from communicating directly with other PVLAN host ports |
| **Community secondary VLAN** | Allows its host ports to communicate with other ports in the same community, but not with ports in other communities or isolated ports |

The following rules apply:

- A PVLAN domain has **one primary VLAN**.
- A primary VLAN can have **one isolated secondary VLAN**.
- A primary VLAN can have **multiple community secondary VLANs**.
- Each secondary VLAN is associated with **only one primary VLAN**.

For example:

```text
                   Primary VLAN 100
                           |
              +------------+------------+
              |            |            |
         Isolated       Community    Community
         VLAN 101        VLAN 102     VLAN 103
              |            |            |
            H1, H2       H3, H4       H5, H6
```

All six hosts can use the same IP subnet. However, they do not all have the same Layer 2 connectivity.

- **H1 and H2** cannot communicate directly, even though both belong to isolated VLAN 101.
- **H3 and H4** can communicate directly because they belong to the same community VLAN 102.
- **H5 and H6** can communicate directly because they belong to the same community VLAN 103.
- Hosts in VLAN 102 cannot communicate directly with hosts in VLAN 103 or isolated VLAN 101.

The primary and secondary VLAN IDs are distinct VLANs, but they operate together as one PVLAN domain.

---

## Private VLAN Port Types

Private VLANs use three principal port types:

| Port type | VLAN association | Layer 2 communication |
|---|---|---|
| **Promiscuous** | Primary VLAN, with mappings to associated secondary VLANs | Can communicate with all host ports in the mapped secondary VLANs |
| **Isolated host** | Primary VLAN and isolated secondary VLAN | Can communicate with promiscuous ports, but not other host ports |
| **Community host** | Primary VLAN and community secondary VLAN | Can communicate with promiscuous ports and host ports in the same community |

An **isolated** or **community** port is configured as a *PVLAN host port*. Its behavior is determined by the type of secondary VLAN associated with that port.

A **promiscuous port** is commonly used to connect a router, firewall, or other shared service that must be reachable from all the associated secondary VLANs.

### Example Topology

```text
                         R1 (Gateway)
                          192.0.2.1
                              |
                       Promiscuous Port
                              |
                             SW1
                       Primary VLAN 100
                +-------------+-------------+
                |             |             |
           Isolated       Community     Community
           VLAN 101        VLAN 102      VLAN 103
             /   \           /   \           /   \
            H1   H2         H3   H4         H5   H6
```

Assume every host uses the `192.0.2.0/24` subnet and R1 is its default gateway.

- Every host can communicate with **R1**, through the promiscuous port.
- H1 and H2 cannot communicate directly with each other or with hosts in either community.
- H3 can communicate directly with H4, but not H5 or H6.
- H5 can communicate directly with H6, but not H3 or H4.

**The promiscuous port is the shared point of connectivity for otherwise isolated host ports.**

---

## PVLAN Forwarding Rules

The following table summarizes direct **Layer 2 communication within one PVLAN domain**.

| Source port | Promiscuous port | Isolated host | Host in same community | Host in different community |
|---|---|---|---|---|
| **Promiscuous** | Allowed | Allowed | Allowed | Allowed |
| **Isolated host** | Allowed | Blocked | Blocked | Blocked |
| **Community host** | Allowed | Blocked | Allowed | Blocked |

For a community host, *same community* means the destination belongs to the **same community secondary VLAN**. Another community host in a different community secondary VLAN is blocked.

The key rules are:

1. **Promiscuous ports** can communicate with all host ports in their mapped secondary VLANs.
2. **Isolated host ports** can communicate only with promiscuous ports, not other host ports.
3. **Community host ports** can communicate with promiscuous ports and other host ports in the same community.
4. **Different communities** cannot communicate directly at Layer 2.

These rules govern forwarding in both directions. A host in an isolated VLAN cannot receive traffic directly from another isolated or community host simply because the destination MAC address is known.

### Broadcast and Multicast Traffic

Private VLAN isolation also affects flooded traffic:

- A broadcast sent by an **isolated host** can reach promiscuous ports, but it is not delivered to other PVLAN host ports.
- A broadcast sent by a **community host** can reach promiscuous ports and other hosts in the same community, but not other communities or isolated hosts.
- A broadcast sent by a **promiscuous port** can reach the associated isolated and community host ports.

Multicast forwarding is subject to corresponding PVLAN isolation rules. In particular, multicast traffic is not forwarded directly between isolated host ports or between different secondary VLANs.

For example, when H1 sends an ARP request for its default gateway, the request can reach R1. However, H2 does not receive that ARP request simply because it shares isolated VLAN 101 with H1.

---

## Private VLANs and IP Addressing

Unlike ordinary VLAN segmentation, the secondary VLANs in a PVLAN domain can share **one IP subnet and one default gateway**.

For example:

```text
Primary VLAN 100
IP subnet: 192.0.2.0/24
Gateway:   192.0.2.1

H1 (Isolated VLAN 101): 192.0.2.11/24
H2 (Isolated VLAN 101): 192.0.2.12/24
H3 (Community VLAN 102): 192.0.2.13/24
H4 (Community VLAN 102): 192.0.2.14/24
```

All four hosts use `192.0.2.1` as their default gateway.

H1 can reach R1 through the promiscuous port. However, if H1 tries to communicate directly with H2, H1 treats H2 as an on-link destination and attempts to resolve H2's MAC address using ARP. PVLAN isolation prevents that ARP request from reaching H2.

This demonstrates an important distinction:

> **Sharing an IP subnet does not guarantee direct Layer 2 connectivity when private VLANs are configured.**

For Layer 3 switching, an **SVI is configured on the primary VLAN**, not on the secondary VLANs. The secondary VLANs must be mapped appropriately to the primary VLAN SVI to obtain Layer 3 connectivity.

Private VLANs control Layer 2 forwarding. They are **not a replacement for Layer 3 security policies**. Routing, firewalls, proxy ARP, or other forwarding paths can affect whether traffic between otherwise isolated hosts is possible through a Layer 3 device.

---

## Private VLANs Across Multiple Switches

A private VLAN can extend across multiple switches using **802.1Q trunks**.

The trunk must carry the required **primary and secondary VLANs**, and the switches must preserve the appropriate PVLAN associations and forwarding behavior.

For example:

```text
     H1                     H2
      |                      |
 Isolated VLAN 101     Isolated VLAN 101
      |                      |
     SW1 ===== 802.1Q ===== SW2
            Trunk carrying
            VLANs 100, 101
```

H1 and H2 remain isolated even though they are connected to different switches, assuming the switches are configured to maintain PVLAN isolation.

Cisco also supports specialized **isolated PVLAN trunk ports** and **promiscuous PVLAN trunk ports** for particular connectivity designs. These are different from ordinary 802.1Q trunks and are covered in the dedicated trunking page.

---

## Private VLANs vs. Protected Ports

The `switchport protected` feature can also prevent direct Layer 2 communication between protected ports on the same switch.

However, **protected ports** and **private VLANs** are not equivalent:

| Feature | Protected ports | Private VLANs |
|---|---|---|
| Basic purpose | Isolate selected switch ports | Create an isolated or community-based Layer 2 domain |
| Multiple communication groups | No community VLAN structure | Supports isolated and multiple community VLANs |
| Isolation across switches | Not inherently maintained across a trunk | Can be maintained across switches with PVLAN configuration |
| Primary and secondary VLANs | Not used | Required |

Private VLANs provide more flexible isolation when hosts span multiple switches or need different communication permissions.

---

## What This Section Covers

The following pages examine:

- **Primary and Secondary VLANs:** Primary, isolated, and community VLANs and their associations.
- **PVLAN Port Types and Forwarding Rules:** Host and promiscuous ports, permitted traffic flows, and isolation behavior.
- **Private VLAN Configuration:** Creating VLANs, associating secondary VLANs, and configuring host and promiscuous ports.
- **Private VLAN Trunks:** Standard 802.1Q trunks, isolated trunks, and promiscuous trunks.
- **Layer 3 Connectivity with Private VLANs:** Default gateways, primary VLAN SVIs, and secondary VLAN mappings.
- **Verification and Troubleshooting:** Relevant show commands, configuration problems, and connectivity tests.

---

## Key Points

- Private VLANs provide **Layer 2 isolation within a shared PVLAN domain**.
- A PVLAN has **one primary VLAN**, **at most one isolated secondary VLAN**, and **multiple possible community secondary VLANs**.
- Each secondary VLAN is associated with **one primary VLAN**.
- **Promiscuous ports** can communicate with host ports in their mapped secondary VLANs.
- **Isolated host ports** cannot communicate directly with other PVLAN host ports, including other isolated ports.
- **Community host ports** can communicate directly with hosts in the same community, but not with isolated ports or other communities.
- Hosts in different secondary VLANs can **share an IP subnet and default gateway**.
- Private VLAN isolation applies to Layer 2 unicast, broadcast, and multicast forwarding.
- Layer 3 SVIs belong to **primary VLANs**, not secondary VLANs.
- PVLANs can span multiple switches when trunks and PVLAN associations are configured appropriately.
- PVLAN isolation does not automatically prevent communication through a Layer 3 forwarding path.

---

## References

- [Cisco Catalyst 9300, VLAN Configuration Guide: Configuring Private VLANs](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-14/configuration_guide/vlan/b_1714_vlan_9300_cg/configuring_private_vlans.html)
- [Cisco Catalyst 9300, VLAN Commands: switchport mode private-vlan](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-18/command_reference/b_1718_9300_cr/vlan_commands.html)
