# Layer 3 Connectivity with Private VLANs

**Private VLANs (PVLANs)** restrict Layer 2 forwarding between host ports, but hosts still need **Layer 3 connectivity** to reach networks outside their subnet.

A PVLAN domain uses a common IP subnet, even when its host ports belong to different secondary VLANs. The primary VLAN provides the connection to a Layer 3 gateway.

Two common gateway designs are:

1. **Primary VLAN SVI:** A multilayer switch routes traffic for all associated secondary VLANs through an SVI on the primary VLAN.
2. **External gateway:** A router or firewall connects to a mapped promiscuous port or promiscuous PVLAN trunk.

In either design, **the PVLAN's Layer 2 isolation rules remain in effect**. Routing is a separate operation and must not be confused with direct Layer 2 communication between PVLAN hosts.

For the VLAN roles and host-port behavior, see [Primary and Secondary VLANs](vlan-types.md) and [PVLAN Port Types and Forwarding Rules](port-types.md).

---

## PVLAN IP Addressing

The examples use the same PVLAN domain as the preceding pages.

| VLAN | Type | Hosts |
|---|---|---|
| 100 | Primary | Gateway-facing VLAN |
| 101 | Isolated secondary | H1, H2 |
| 102 | Community secondary | H3, H4 |
| 103 | Community secondary | H5 |

All five hosts belong to **192.0.2.0/24**, regardless of their secondary VLAN.

| Host | IP address | Secondary VLAN | Default gateway |
|---|---|---|---|
| H1 | 192.0.2.11/24 | 101 (isolated) | 192.0.2.1 |
| H2 | 192.0.2.12/24 | 101 (isolated) | 192.0.2.1 |
| H3 | 192.0.2.13/24 | 102 (community) | 192.0.2.1 |
| H4 | 192.0.2.14/24 | 102 (community) | 192.0.2.1 |
| H5 | 192.0.2.15/24 | 103 (community) | 192.0.2.1 |

**Secondary VLANs do not require separate IP subnets or separate default gateways.**

For example, H1 and H3 are in different secondary VLANs but use the same subnet mask and default gateway. They cannot communicate directly at Layer 2 because the PVLAN forbids forwarding between their host ports.

A host's secondary VLAN determines its **Layer 2 forwarding permissions**, not its IP subnet.

---

## Two Ways to Provide a Default Gateway

| Design | Gateway location | PVLAN-specific configuration |
|---|---|---|
| Primary VLAN SVI | On the multilayer switch | `private-vlan mapping` under `interface Vlan100` |
| External router or firewall | Connected to the switch | Promiscuous port or promiscuous trunk mapped to the secondary VLANs |

Choose **one gateway design** for the same PVLAN subnet. The two configurations below are alternatives, not configurations to apply simultaneously.

---

## Method 1: Primary VLAN SVI

On a multilayer Catalyst switch, the **primary VLAN SVI** can act as the default gateway for all hosts in the PVLAN domain.

**Configure the SVI for the primary VLAN, not for each secondary VLAN.**

### Example Topology

```text
                              R1
                       Lo0: 203.0.113.10/32
                       Gi0/0: 198.51.100.2/30
                               |
                         Routed uplink
                               |
                       Gi1/0/24: 198.51.100.1/30
                              SW1
                        Vlan100 SVI
                         192.0.2.1/24
                               |
                    Primary VLAN 100
             +-----------------+-----------------+
             |                 |                 |
         Isolated 101     Community 102     Community 103
           /     \          /     \                |
          H1     H2        H3     H4               H5

       All hosts use gateway 192.0.2.1.
```

The SVI belongs to VLAN 100, but it serves host traffic received in secondary VLANs 101, 102, and 103 through the **Layer 3 PVLAN mapping**.

Gi1/0/24 is a regular **routed interface**, not a promiscuous PVLAN port.

### Step 1: Configure the PVLANs

```text
SW1(config)# vtp mode transparent
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
SW1(config-vlan)# exit
```

The association under `vlan 100` defines the **Layer 2 relationship** between the primary and secondary VLANs.

### Step 2: Configure the Host Ports

```text
SW1(config)# interface range GigabitEthernet1/0/2-3
SW1(config-if-range)# switchport mode private-vlan host
SW1(config-if-range)# switchport private-vlan host-association 100 101
SW1(config-if-range)# exit
!
SW1(config)# interface range GigabitEthernet1/0/4-5
SW1(config-if-range)# switchport mode private-vlan host
SW1(config-if-range)# switchport private-vlan host-association 100 102
SW1(config-if-range)# exit
!
SW1(config)# interface GigabitEthernet1/0/6
SW1(config-if)# switchport mode private-vlan host
SW1(config-if)# switchport private-vlan host-association 100 103
SW1(config-if)# exit
```

Each host port is associated with **one primary/secondary VLAN pair**.

### Step 3: Configure the Primary VLAN SVI

```text
SW1(config)# ip routing
SW1(config)# interface Vlan100
SW1(config-if)# ip address 192.0.2.1 255.255.255.0
SW1(config-if)# private-vlan mapping 101-103
SW1(config-if)# no shutdown
```

The critical command is:

```text
SW1(config-if)# private-vlan mapping 101-103
```

This command **maps secondary VLANs 101, 102, and 103 to the Layer 3 interface of primary VLAN 100**.

Traffic received from a mapped secondary VLAN can therefore be processed by the primary VLAN's SVI for Layer 3 forwarding.

The `ip routing` command enables routing between IP networks on the multilayer switch. It is not a PVLAN-specific command.

### Step 4: Configure the Routed Uplink

```text
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# no switchport
SW1(config-if)# ip address 198.51.100.1 255.255.255.252
SW1(config-if)# no shutdown
SW1(config-if)# exit
SW1(config)# ip route 0.0.0.0 0.0.0.0 198.51.100.2
```

R1 connects to the routed uplink and has a test loopback:

```text
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 198.51.100.2 255.255.255.252
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# interface Loopback0
R1(config-if)# ip address 203.0.113.10 255.255.255.255
R1(config-if)# exit
R1(config)# ip route 192.0.2.0 255.255.255.0 198.51.100.1
```

The return route on R1 is required so that replies can reach the PVLAN subnet.

### Packet Forwarding Through the SVI

Suppose H1 (192.0.2.11) sends an IP packet to R1's Loopback0 address (203.0.113.10).

1. H1 determines that 203.0.113.10 is outside 192.0.2.0/24.
2. H1 ARPs for its default gateway, 192.0.2.1. Its ARP Request enters SW1 through isolated VLAN 101 and can reach the mapped primary SVI.
3. H1 sends the IP packet with the **SVI's MAC address** as the destination MAC.
4. SW1 receives the packet through VLAN 101, and the `private-vlan mapping 101-103` configuration allows the **Vlan100 SVI** to route it.
5. SW1 routes the packet out Gi1/0/24 toward R1.
6. R1 replies using its route to 192.0.2.0/24 through SW1.
7. SW1 forwards the return traffic from the primary SVI toward H1 in the associated isolated VLAN 101.

```text
H1 (101) --> SW1 Vlan100 SVI --> Gi1/0/24 --> R1
                192.0.2.1

H1 (101) <-- SW1 Vlan100 SVI <-- Gi1/0/24 <-- R1
```

The SVI is the **Layer 3 gateway for all mapped secondary VLANs**. There is no need to create Vlan101, Vlan102, or Vlan103 SVIs.

---

## Primary VLAN SVI Mapping vs. Other PVLAN Mappings

PVLAN configurations use several similarly named commands with different functions.

| Configuration context | Command | Purpose |
|---|---|---|
| Primary VLAN | `private-vlan association 101-103` | Associates secondary VLANs with primary VLAN 100 at Layer 2 |
| PVLAN host port | `switchport private-vlan host-association 100 101` | Assigns a host port to a primary/secondary pair |
| Promiscuous port | `switchport private-vlan mapping 100 101-103` | Permits Layer 2 traffic between the promiscuous port and mapped secondary host ports |
| **Primary SVI** | **`private-vlan mapping 101-103`** | Allows secondary VLAN traffic to be handled by the primary SVI at Layer 3 |

**The SVI command does not contain `switchport` and does not repeat the primary VLAN ID.** The primary VLAN is already identified by `interface Vlan100`.

The SVI's `private-vlan mapping` is also **not a substitute for the Layer 2 `private-vlan association`** under the primary VLAN.

### Adding and Removing SVI Mappings

Add another eligible secondary VLAN without replacing the existing list:

```text
SW1(config)# interface Vlan100
SW1(config-if)# private-vlan mapping add 104
```

Remove one mapping:

```text
SW1(config-if)# private-vlan mapping remove 104
```

VLAN 104 must already be a secondary VLAN associated with primary VLAN 100 for the mapping to take effect.

---

## Method 2: External Router or Firewall

An external router or firewall can provide the default gateway through a **promiscuous PVLAN port**.

In this design, **SW1 does not need an SVI for VLAN 100**. It forwards Ethernet frames between the secondary host ports and the mapped promiscuous port, while R1 performs IP routing.

### Example Topology

```text
                        R1
                  Gi0/0: 192.0.2.1/24
                  Lo0: 203.0.113.10/32
                         |
                 Gi1/0/1 (Promiscuous)
                         |
                        SW1
                  Primary VLAN 100
             +-----------+-----------+
             |           |           |
         Isolated 101 Community 102 Community 103
           /    \       /    \           |
          H1    H2     H3    H4          H5

          All hosts: 192.0.2.0/24
          Gateway:   192.0.2.1
```

### Configure the PVLAN on SW1

Assume the primary/secondary VLANs and host ports are already configured as shown in Method 1.

Configure the gateway-facing interface as promiscuous:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport mode private-vlan promiscuous
SW1(config-if)# switchport private-vlan mapping 100 101-103
SW1(config-if)# no shutdown
```

The **promiscuous mapping** allows R1 to communicate with hosts in all three associated secondary VLANs.

Do not configure `private-vlan mapping 101-103` under an SVI on SW1 for this design. R1, not SW1, supplies the gateway.

### Configure R1

```text
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.0.2.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# interface Loopback0
R1(config-if)# ip address 203.0.113.10 255.255.255.255
```

All five hosts use **192.0.2.1** as their default gateway.

R1 does **not** need to configure or understand the isolated/community VLAN IDs. Its Ethernet interface exchanges **untagged frames** with the mapped promiscuous port, which carries those frames within the primary PVLAN domain.

### Packet Forwarding Through the Promiscuous Port

Suppose H2 (isolated VLAN 101) sends a packet to R1's Loopback0 address.

1. H2 ARPs for 192.0.2.1.
2. SW1 forwards H2's ARP Request from isolated VLAN 101 to the **mapped promiscuous port**.
3. R1 replies, and SW1 forwards the ARP Reply back to H2.
4. H2 sends the IP packet to R1's MAC address.
5. SW1 bridges the frame to the promiscuous port, and R1 receives it.
6. R1 processes the packet and returns the response to H2 through the same interface.

```text
H2 (101) --> SW1 --> Promiscuous Gi1/0/1 --> R1
H2 (101) <-- SW1 <-- Promiscuous Gi1/0/1 <-- R1
```

SW1 performs **Layer 2 forwarding** between H2 and R1; routing occurs on R1.

### External Gateway Over an 802.1Q Trunk

If the router or firewall must serve **multiple PVLAN domains** on one physical link, use a **promiscuous PVLAN trunk** rather than an untagged promiscuous port.

For primary VLAN 100 and secondary VLANs 101-103:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport mode private-vlan trunk promiscuous
SW1(config-if)# switchport private-vlan mapping trunk 100 101-103
SW1(config-if)# switchport private-vlan trunk allowed vlan 100
```

R1 uses an 802.1Q subinterface for **primary VLAN 100**:

```text
R1(config)# interface GigabitEthernet0/0
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# interface GigabitEthernet0/0.100
R1(config-subif)# encapsulation dot1Q 100
R1(config-subif)# ip address 192.0.2.1 255.255.255.0
```

The trunk presents the PVLAN domain to R1 using **VLAN 100 tags**, not tags 101-103.

For additional trunk configurations, see [Private VLAN Trunks](trunks.md).

---

## Layer 3 Routing Does Not Automatically Remove PVLAN Isolation

PVLAN forwarding rules are **Layer 2 restrictions**. Whether two hosts can communicate through a router is a separate question.

### Same-Subnet Traffic

Suppose H1 (192.0.2.11) tries to contact H2 (192.0.2.12).

Both hosts use `/24` masks, so H1 considers H2 **on-link**. H1 ARPs for **H2's MAC address**, not for the default gateway's MAC address.

```text
H1 (101) -- ARP for 192.0.2.12 --> SW1 --X--> H2 (101)
```

The isolated host port cannot forward the ARP Request to H2's isolated host port. **A working default gateway does not automatically make this direct same-subnet communication succeed.**

Similarly, H3 in community 102 cannot ARP directly for H5 in community 103, while H3 and H4 in the **same community 102** can exchange ARP and unicast frames normally.

### Traffic to Different Subnets

For an off-subnet destination, the host sends traffic to the default gateway. That traffic is permitted through the primary SVI or a mapped promiscuous port.

```text
H1 (101) --> Default gateway --> Another IP subnet
```

The gateway may route traffic between **different IP subnets**, including subnets served by other PVLAN domains, unless a routing policy, ACL, or firewall rule blocks the traffic.

**PVLAN isolation does not replace Layer 3 access control.**

### Local Proxy ARP and Same-Subnet Routing

Some Layer 3 devices support **local proxy ARP**, which lets a gateway answer ARP Requests for another host in the **same IP subnet** using the gateway's own MAC address.

For example, if a supported gateway uses local proxy ARP:

```text
H1 ARPs for H2 (192.0.2.12)
              |
              v
Gateway replies with its own MAC
              |
              v
H1 sends the IP packet to the gateway
              |
              v
Gateway may route the packet back toward H2
```

The command on platforms supporting this function is:

```text
SW1(config)# interface Vlan100
SW1(config-if)# ip local-proxy-arp
```

**This is not part of the basic PVLAN gateway configuration.** If enabled and supported with the intended forwarding design, it can permit same-subnet IP communication via the gateway even though direct host-to-host Layer 2 forwarding remains prohibited.

That may be useful for inspecting or controlling traffic centrally, but it can also defeat the intended **host-to-host isolation at the IP layer** if appropriate Layer 3 filtering is absent.

Do not confuse **ordinary proxy ARP** with **local proxy ARP**. Ordinary proxy ARP does not generally answer ARP Requests for another destination on the same directly connected subnet.

---

## Layer 3 Policy and Shared Subnets

Because all secondary VLANs in one PVLAN domain share a subnet and primary SVI, an ordinary Layer 3 interface ACL does not inherently identify a host's **secondary VLAN**.

For example, this ACL filters by source and destination IP addresses, not by community membership:

```text
SW1(config)# ip access-list extended PVLAN_POLICY
SW1(config-ext-nacl)# deny ip host 192.0.2.11 host 203.0.113.10
SW1(config-ext-nacl)# permit ip any any
SW1(config-ext-nacl)# exit
SW1(config)# interface Vlan100
SW1(config-if)# ip access-group PVLAN_POLICY in
```

This policy prevents routed IP traffic from H1 to 203.0.113.10, while allowing other traffic matched by the second statement.

The ACL applies to **routed traffic passing through the SVI**. It does not replace the PVLAN's Layer 2 host-to-host isolation rules or filter all traffic that is bridged locally within a community VLAN.

If different secondary VLANs require different Layer 3 access policies, consider whether separate IP subnets, VLANs, VRFs, or an upstream firewall would provide clearer control.

---

## DHCP and Other Gateway Services

The shared IP subnet also affects DHCP and network services:

- **DHCP addressing:** Hosts in secondary VLANs normally receive addresses from the **same subnet/pool** associated with the primary VLAN.
- **DHCP relay:** If DHCP relay is required, configure the relay function on the **primary VLAN SVI** in the SVI-gateway design, not on secondary VLAN SVIs.
- **External gateway:** If a router or firewall provides gateway and DHCP relay services, broadcasts from hosts can reach that device through the mapped promiscuous connection.
- **Shared services:** A shared server on an appropriately mapped promiscuous port can communicate with all secondary host groups even without acting as their default gateway.

For example, when the primary SVI is the default gateway and the DHCP server is at 203.0.113.20:

```text
SW1(config)# interface Vlan100
SW1(config-if)# ip helper-address 203.0.113.20
```

The switch must have routing reachability to the DHCP server, and the DHCP server must have an appropriate scope for the shared PVLAN subnet.

---

## Verification

### Verify the Primary/Secondary VLAN Relationship

```text
SW1# show vlan private-vlan
```

Confirm that VLAN 100 is primary and has secondary VLANs 101, 102, and 103 associated with it.

### Verify the SVI Mapping

For Method 1:

```text
SW1# show interfaces private-vlan mapping
SW1# show running-config interface Vlan100
SW1# show ip interface brief
```

Confirm that the SVI contains:

```text
interface Vlan100
 ip address 192.0.2.1 255.255.255.0
 private-vlan mapping 101-103
```

Also verify that `Vlan100` is **up/up** and that the SVI is the correct gateway for all five hosts.

### Verify IP Routing

```text
SW1# show ip route
SW1# show ip route 203.0.113.10
SW1# show ip arp
```

For Method 1, SW1 should have its connected PVLAN subnet and routed-uplink subnet, plus a route to the test destination.

For Method 2, check R1's route table instead:

```text
R1# show ip route
R1# show ip arp
```

### Verify the Promiscuous Port

For Method 2:

```text
SW1# show interfaces GigabitEthernet1/0/1 switchport
SW1# show running-config interface GigabitEthernet1/0/1
```

Confirm that the promiscuous port has the correct **primary-to-secondary mapping**.

### Connectivity Tests

Assuming normal host IP configuration and no additional filtering:

| Test | Expected behavior |
|---|---|
| H1 (isolated 101) to default gateway | Allowed |
| H2 (isolated 101) to default gateway | Allowed |
| H3 (community 102) to default gateway | Allowed |
| H5 (community 103) to default gateway | Allowed |
| H1 to external R1 Loopback0 (203.0.113.10) | Allowed when routing/return path is correct |
| H1 directly to H2 in the same subnet | Blocked at Layer 2 |
| H1 directly to H3 in the same subnet | Blocked at Layer 2 |
| H3 directly to H4 in community 102 | Allowed |
| H3 directly to H5 in another community | Blocked at Layer 2 |

These direct host-to-host expectations assume normal on-link ARP behavior without a local proxy ARP or other mechanism that intentionally routes traffic between hosts in the same subnet.

---

## Common Troubleshooting Problems

| Symptom | Likely cause | Check |
|---|---|---|
| A host cannot reach the primary SVI | Secondary VLAN is missing from the SVI mapping, VLAN association is wrong, or SVI is down | `show interfaces private-vlan mapping`, `show vlan private-vlan`, SVI status |
| One secondary VLAN has no Layer 3 connectivity | Its VLAN ID is omitted from `private-vlan mapping` | `show running-config interface Vlan100` |
| Hosts can reach the gateway but not remote networks | IP routing disabled, missing upstream route, or missing return route | `show ip route` on gateway and upstream router |
| Hosts cannot reach an external gateway | Promiscuous port lacks secondary VLAN mapping | `show interfaces ... switchport` |
| A secondary SVI stays down | An SVI was created for an isolated or community VLAN | Configure only the primary VLAN SVI |
| A secondary VLAN cannot reach the primary SVI | Only the global VLAN association was configured, without the Layer 3 SVI mapping | Compare `private-vlan association` and `private-vlan mapping` |
| Two same-subnet isolated hosts cannot ping each other | Expected PVLAN isolation prevents direct ARP | Examine ARP behavior and the PVLAN forwarding rules |
| Two same-subnet isolated hosts unexpectedly communicate | The gateway or another mechanism may be routing the traffic, or PVLAN isolation is not enforced end to end | Inspect local proxy ARP, routed path, and PVLAN definitions |
| Hosts can reach some secondary VLANs but not others through an external gateway | Promiscuous mapping omits one or more secondary VLANs | Check interface mapping and gateway interface |

---

## Key Points

- **All secondary VLANs associated with a primary VLAN normally share the same IP subnet and default gateway.**
- Configure a **Layer 3 SVI only for the primary VLAN**, not the secondary VLANs.
- Map secondary VLANs to the primary SVI with **`private-vlan mapping <secondary-vlan-list>`**.
- The SVI mapping allows **Layer 3 forwarding** for traffic received in the associated secondary VLANs.
- The SVI mapping and the primary VLAN's `private-vlan association` are **different configurations**.
- When using an **external gateway**, connect it through a mapped **promiscuous port** or **promiscuous PVLAN trunk** instead of configuring a gateway SVI on the switch.
- A regular promiscuous port carries untagged frames; a **promiscuous PVLAN trunk** carries PVLAN traffic using the **primary VLAN ID**.
- **PVLANs enforce Layer 2 isolation, not a universal Layer 3 security policy.**
- Hosts in the same subnet attempt direct ARP by default. A gateway does not automatically enable communication between isolated hosts in the same subnet.
- **Local proxy ARP** may allow same-subnet hosts to communicate through a Layer 3 gateway, so additional filtering may be required to preserve IP-level isolation.
- A routed path requires **both forward and return reachability**.

---

## References

- [Cisco Catalyst 9300 IOS XE 17.18.x: Configuring Private VLANs](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-18/configuration_guide/vlan/b_1718_vlan_9300_cg/configuring_private_vlans.html)
- [Cisco Catalyst 9300 IOS XE 17.18.x: VLAN Command Reference](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-18/command_reference/b_1718_9300_cr/vlan_commands.html)
- [Cisco: Configure Isolated Private VLANs on Catalyst Switches](https://www.cisco.com/c/en/us/support/docs/lan-switching/private-vlans-pvlans-promiscuous-isolated-community/40781-194.html)
- [Cisco Catalyst 4500 IOS XE 3.11.xE: `ip local-proxy-arp`](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst4500/XE3-11-0E/command/reference/xe-311-cmd/int_sess.html)
