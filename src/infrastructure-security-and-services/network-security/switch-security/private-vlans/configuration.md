# Private VLAN Configuration

This page demonstrates how to configure a complete **Private VLAN (PVLAN)** domain on a Cisco Catalyst switch running IOS XE.

The configuration includes:

- A primary VLAN.
- An isolated secondary VLAN.
- Two community secondary VLANs.
- Isolated and community host ports.
- A promiscuous port connected to a shared default gateway.
- Primary-to-secondary VLAN associations and interface mappings.
- Verification and common configuration mistakes.

For the conceptual differences between primary and secondary VLANs, see [Primary and Secondary VLANs](vlan-types.md). For the forwarding rules associated with each port type, see [PVLAN Port Types and Forwarding Rules](port-types.md).

---

## Example Topology

SW1 connects five hosts and an external router, R1.

```text
                            R1
                     192.0.2.1/24
                             |
                       Gi1/0/1 (P)
                             |
                            SW1
                      Primary VLAN 100
                  /      /     |      \      \
                 /      /      |       \      \
           Gi1/0/2 Gi1/0/3 Gi1/0/4 Gi1/0/5 Gi1/0/6
              |       |       |       |       |
             H1      H2      H3      H4      H5
              \       /        \     /        |
           Isolated 101     Community 102   Community 103
```

**P** identifies the promiscuous port connected to R1.

The VLAN and interface assignments are:

| VLAN | PVLAN type | Function |
|---|---|---|
| 100 | Primary | Defines the PVLAN domain |
| 101 | Isolated secondary | H1 and H2 cannot communicate directly with each other or any community hosts |
| 102 | Community secondary | H3 and H4 can communicate directly with each other |
| 103 | Community secondary | H5 is isolated from the other community and isolated hosts |

| SW1 interface | Connected device | PVLAN mode | VLAN relationship |
|---|---|---|---|
| Gi1/0/1 | R1 | Promiscuous | Primary 100, mapped to 101-103 |
| Gi1/0/2 | H1 | Host (isolated) | 100 / 101 |
| Gi1/0/3 | H2 | Host (isolated) | 100 / 101 |
| Gi1/0/4 | H3 | Host (community) | 100 / 102 |
| Gi1/0/5 | H4 | Host (community) | 100 / 102 |
| Gi1/0/6 | H5 | Host (community) | 100 / 103 |

All devices use **192.0.2.0/24**. R1 is the default gateway at **192.0.2.1**.

| Device | IP address | Default gateway |
|---|---|---|
| R1 | 192.0.2.1/24 | N/A |
| H1 | 192.0.2.11/24 | 192.0.2.1 |
| H2 | 192.0.2.12/24 | 192.0.2.1 |
| H3 | 192.0.2.13/24 | 192.0.2.1 |
| H4 | 192.0.2.14/24 | 192.0.2.1 |
| H5 | 192.0.2.15/24 | 192.0.2.1 |

The hosts share an IP subnet, but their **direct Layer 2 connectivity is restricted by the PVLAN configuration**.

---

## Configuration Order

A practical configuration order is:

1. Configure a VTP mode that supports PVLANs.
2. Create the primary and secondary VLANs and assign their PVLAN types.
3. Associate the secondary VLANs with the primary VLAN.
4. Configure the isolated and community host ports.
5. Configure the promiscuous port and its secondary VLAN mappings.
6. Verify the VLAN relationships, interface modes, and forwarding behavior.

There are **three distinct relationships** to configure:

| Relationship | Configuration mode | Command |
|---|---|---|
| Primary VLAN to secondary VLANs | Primary VLAN configuration | `private-vlan association` |
| Host port to one primary/secondary pair | Interface configuration | `switchport private-vlan host-association` |
| Promiscuous port to a primary VLAN and selected secondary VLANs | Interface configuration | `switchport private-vlan mapping` |

These commands are not interchangeable. Configuring the VLAN association alone does not assign host ports or permit traffic through a promiscuous port.

---

## Step 1: Configure VTP

For **VTP version 1 or 2**, a switch must be in **transparent mode** to configure PVLANs.

```text
SW1(config)# vtp mode transparent
```

**VTP version 3 supports PVLANs in server and transparent modes**, so transparent mode is not required if a VTPv3 design already supports the intended PVLAN configuration.

Verify the current VTP configuration before changing it:

```text
SW1# show vtp status
```

The example uses VTP transparent mode and assumes SW1's VLAN configuration is managed locally.

---

## Step 2: Create the Primary and Secondary VLANs

### Primary VLAN

Create VLAN 100 and designate it as the primary VLAN:

```text
SW1(config)# vlan 100
SW1(config-vlan)# name PVLAN_PRIMARY
SW1(config-vlan)# private-vlan primary
SW1(config-vlan)# exit
```

The `private-vlan primary` command assigns the **primary PVLAN role** to VLAN 100.

### Isolated Secondary VLAN

Create VLAN 101 and designate it as isolated:

```text
SW1(config)# vlan 101
SW1(config-vlan)# name PVLAN_ISOLATED
SW1(config-vlan)# private-vlan isolated
SW1(config-vlan)# exit
```

Any host port associated with secondary VLAN 101 becomes an **isolated host port**.

An isolated secondary VLAN can contain many host ports, but a primary VLAN can have **only one associated isolated secondary VLAN**.

### Community Secondary VLANs

Create VLANs 102 and 103 as community VLANs:

```text
SW1(config)# vlan 102
SW1(config-vlan)# name PVLAN_COMMUNITY_A
SW1(config-vlan)# private-vlan community
SW1(config-vlan)# exit
!
SW1(config)# vlan 103
SW1(config-vlan)# name PVLAN_COMMUNITY_B
SW1(config-vlan)# private-vlan community
SW1(config-vlan)# exit
```

Hosts in **the same** community VLAN can communicate directly. Hosts in **different** community VLANs cannot communicate directly, even if they share the primary VLAN.

A primary VLAN can have multiple community secondary VLANs.

---

## Step 3: Associate the Secondary VLANs with the Primary VLAN

Under the **primary VLAN**, associate isolated VLAN 101 and community VLANs 102 and 103:

```text
SW1(config)# vlan 100
SW1(config-vlan)# private-vlan association 101-103
SW1(config-vlan)# exit
```

This establishes the following relationships:

```text
Primary VLAN 100
   |
   +-- Isolated VLAN 101
   +-- Community VLAN 102
   +-- Community VLAN 103
```

The `private-vlan association` command is entered under the **primary VLAN**, not under the secondary VLANs.

### Changing Associations

Add a secondary VLAN to an existing primary VLAN association:

```text
SW1(config)# vlan 100
SW1(config-vlan)# private-vlan association add 104
```

This assumes VLAN 104 already exists and is configured as a community secondary VLAN (or another eligible secondary VLAN, subject to the one-isolated-VLAN limit).

Remove a secondary VLAN association:

```text
SW1(config-vlan)# private-vlan association remove 104
```

The form without `add` or `remove` specifies the desired association list. When modifying an existing association, use **`add`** to preserve the existing associated VLANs.

Secondary VLAN lists can include individual IDs, comma-separated values, and ranges, such as `101,103,105-107`. They must not contain spaces.

---

## Step 4: Configure Isolated Host Ports

Configure Gi1/0/2 and Gi1/0/3 as PVLAN host ports associated with **primary VLAN 100** and **isolated VLAN 101**:

```text
SW1(config)# interface range GigabitEthernet1/0/2-3
SW1(config-if-range)# switchport
SW1(config-if-range)# switchport mode private-vlan host
SW1(config-if-range)# switchport private-vlan host-association 100 101
SW1(config-if-range)# spanning-tree portfast
SW1(config-if-range)# spanning-tree bpduguard enable
SW1(config-if-range)# no shutdown
```

The two essential PVLAN commands are:

```text
switchport mode private-vlan host
switchport private-vlan host-association 100 101
```

The first command configures **PVLAN host mode**.

The second identifies the port's **primary and secondary VLAN pair**.

The host port's isolated behavior is determined by VLAN 101 being configured with `private-vlan isolated`.

`spanning-tree portfast` and `spanning-tree bpduguard enable` are appropriate for ports connected directly to end hosts. They are not what makes the ports isolated.

---

## Step 5: Configure Community Host Ports

Configure H3 and H4 in **community VLAN 102**:

```text
SW1(config)# interface range GigabitEthernet1/0/4-5
SW1(config-if-range)# switchport mode private-vlan host
SW1(config-if-range)# switchport private-vlan host-association 100 102
SW1(config-if-range)# spanning-tree portfast
SW1(config-if-range)# spanning-tree bpduguard enable
SW1(config-if-range)# no shutdown
```

Configure H5 in **community VLAN 103**:

```text
SW1(config)# interface GigabitEthernet1/0/6
SW1(config-if)# switchport
SW1(config-if)# switchport mode private-vlan host
SW1(config-if)# switchport private-vlan host-association 100 103
SW1(config-if)# spanning-tree portfast
SW1(config-if)# spanning-tree bpduguard enable
SW1(config-if)# no shutdown
```

**Isolated and community host ports use the same commands.** The secondary VLAN's type determines the forwarding rules.

For example:

```text
Host association 100 101 -> Isolated host
Host association 100 102 -> Community host (community A)
Host association 100 103 -> Community host (community B)
```

Each PVLAN host port is associated with **one primary VLAN and one secondary VLAN**, not a list of secondary VLANs.

Do not use `switchport access vlan 101` to assign a PVLAN host port to isolated VLAN 101. PVLAN hosts must use `switchport private-vlan host-association`.

---

## Step 6: Configure the Promiscuous Port

R1 connects to SW1 through Gi1/0/1. That switch interface must allow traffic to and from all three secondary VLANs.

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# description R1_DEFAULT_GATEWAY
SW1(config-if)# switchport
SW1(config-if)# switchport mode private-vlan promiscuous
SW1(config-if)# switchport private-vlan mapping 100 101-103
SW1(config-if)# no shutdown
```

The two essential PVLAN commands are:

```text
switchport mode private-vlan promiscuous
switchport private-vlan mapping 100 101-103
```

- `switchport mode private-vlan promiscuous` assigns the **promiscuous port role**.
- `switchport private-vlan mapping 100 101-103` maps **secondary VLANs 101, 102, and 103** to the promiscuous port in **primary VLAN 100**.

This allows R1 to exchange Layer 2 traffic with all five hosts, subject to other forwarding or security policies.

### Promiscuous Port Mapping vs. VLAN Association

The following commands serve different purposes:

```text
SW1(config)# vlan 100
SW1(config-vlan)# private-vlan association 101-103
```

Associates secondary VLANs 101-103 with **primary VLAN 100**.

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport private-vlan mapping 100 101-103
```

Allows **that promiscuous interface** to communicate with the selected secondary VLANs.

If the promiscuous port were mapped only to VLANs 101 and 102, hosts in VLAN 103 would not be able to communicate with R1 through that port, even though VLAN 103 remained associated with primary VLAN 100.

### Changing a Promiscuous Port Mapping

To add another already-associated secondary VLAN to the port's existing mapping:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport private-vlan mapping 100 add 104
```

To remove the mapping for VLAN 104:

```text
SW1(config-if)# switchport private-vlan mapping 100 remove 104
```

**VLAN association and promiscuous-port mapping are independent configuration steps.** Adding a secondary VLAN to the primary VLAN does not automatically map it to every promiscuous port.

---

## Step 7: Configure the External Gateway

R1 connects to SW1's promiscuous port through an ordinary routed Ethernet interface:

```text
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.0.2.1 255.255.255.0
R1(config-if)# no shutdown
```

Because Gi1/0/1 on SW1 is a **promiscuous host-facing switchport**, R1 does not need a separate VLAN subinterface for VLANs 101, 102, or 103 in this topology.

R1 simply serves as the default gateway for **192.0.2.0/24**.

The promiscuous port is not the same as an IEEE 802.1Q trunk. PVLAN trunk configuration is covered separately in [Private VLAN Trunks](trunks.md).

Alternatively, the switch itself can provide the default gateway through an SVI of the **primary VLAN**, with appropriate secondary VLAN mappings on the SVI. That design is covered in [Layer 3 Connectivity with Private VLANs](layer3.md).

---

## Complete SW1 Configuration

The following combines the required configuration into one example. R1's routed interface is configured as shown above.

```text
vtp mode transparent
!
vlan 100
 name PVLAN_PRIMARY
 private-vlan primary
 private-vlan association 101-103
!
vlan 101
 name PVLAN_ISOLATED
 private-vlan isolated
!
vlan 102
 name PVLAN_COMMUNITY_A
 private-vlan community
!
vlan 103
 name PVLAN_COMMUNITY_B
 private-vlan community
!
interface GigabitEthernet1/0/1
 description R1_DEFAULT_GATEWAY
 switchport mode private-vlan promiscuous
 switchport private-vlan mapping 100 101-103
 no shutdown
!
interface range GigabitEthernet1/0/2-3
 switchport
 switchport mode private-vlan host
 switchport private-vlan host-association 100 101
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
!
interface range GigabitEthernet1/0/4-5
 switchport
 switchport mode private-vlan host
 switchport private-vlan host-association 100 102
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
!
interface GigabitEthernet1/0/6
 switchport
 switchport mode private-vlan host
 switchport private-vlan host-association 100 103
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
```

The `interface range` lines are convenient for entering configuration. A normal `show running-config` displays the resulting commands **under each individual interface**.

---

## Verification

### Check Primary and Secondary VLAN Relationships

```text
SW1# show vlan private-vlan
```

The expected relationships are:

| Primary | Secondary | Type |
|---|---|---|
| 100 | 101 | Isolated |
| 100 | 102 | Community |
| 100 | 103 | Community |

The command can also show associated ports, depending on platform and output format.

### Check Host Port Configuration

```text
SW1# show interfaces GigabitEthernet1/0/2 switchport
```

Look for the following information:

```text
Administrative Mode: private-vlan host
Administrative private-vlan host-association: 100 101
Operational private-vlan:
  100 101
```

The output above is an abbreviated illustration of the relevant fields.

For a community port, the association should identify its community secondary VLAN, such as `100 102`.

### Check the Promiscuous Port

```text
SW1# show interfaces GigabitEthernet1/0/1 switchport
```

Check that the operational mode is **private-vlan promiscuous** and that primary VLAN 100 is mapped to VLANs 101, 102, and 103.

Additional useful commands include:

```text
SW1# show interfaces status
SW1# show running-config interface GigabitEthernet1/0/1
SW1# show running-config interface GigabitEthernet1/0/2
```

### Test Connectivity

Assuming the connected hosts and R1 are operational and no additional security policies interfere, expect the following **direct Layer 2** results:

| Source | Destination | Result | Reason |
|---|---|---|---|
| H1 | R1 | Permitted | R1's promiscuous port maps VLAN 101 |
| H2 | R1 | Permitted | R1's promiscuous port maps VLAN 101 |
| H3 | R1 | Permitted | R1's promiscuous port maps VLAN 102 |
| H5 | R1 | Permitted | R1's promiscuous port maps VLAN 103 |
| H1 | H2 | Blocked | Both hosts are isolated |
| H1 | H3 | Blocked | Isolated-to-community forwarding is prohibited |
| H3 | H4 | Permitted | Both hosts belong to community VLAN 102 |
| H3 | H5 | Blocked | Different community VLANs |

The table describes Layer 2 forwarding. It does **not** guarantee that a router or firewall cannot forward traffic between hosts at Layer 3.

If testing with `ping`, remember that **a failed ping can also result from ARP, host firewalls, IP addressing, or routing**, not necessarily from PVLAN isolation. Confirm Layer 2 behavior before diagnosing a failed test as a PVLAN issue.

---

## Common Configuration Mistakes

### 1. Forgetting the Primary-to-Secondary VLAN Association

Configuring VLAN 101 as isolated does **not** automatically associate it with VLAN 100:

```text
vlan 101
 private-vlan isolated
```

The primary VLAN also needs:

```text
vlan 100
 private-vlan association 101
```

A host association referencing an invalid primary/secondary relationship cannot operate correctly.

### 2. Confusing Host Associations with Promiscuous Mappings

A host port uses:

```text
switchport private-vlan host-association 100 101
```

A promiscuous port uses:

```text
switchport private-vlan mapping 100 101-103
```

**`host-association` specifies one secondary VLAN. `mapping` specifies which secondary VLANs the promiscuous port can reach.**

### 3. Forgetting the Promiscuous Port Mapping

This is incomplete:

```text
interface GigabitEthernet1/0/1
 switchport mode private-vlan promiscuous
```

The interface also needs a valid mapping:

```text
switchport private-vlan mapping 100 101-103
```

On Cisco Catalyst switches, a PVLAN port without a valid host association or promiscuous mapping can become **inactive**.

### 4. Configuring a Regular Access Port in a PVLAN

Do not configure an isolated host like this:

```text
interface GigabitEthernet1/0/2
 switchport mode access
 switchport access vlan 101
```

Instead, use the PVLAN host mode and association:

```text
interface GigabitEthernet1/0/2
 switchport mode private-vlan host
 switchport private-vlan host-association 100 101
```

Ordinary access ports assigned to VLANs configured as PVLAN primary or secondary VLANs are not a substitute for PVLAN host ports.

### 5. Using an Unsupported VLAN or Interface Feature

On Catalyst platforms, VLAN **1** and reserved VLANs **1002-1005** cannot be configured as PVLANs. Supported VLAN IDs are generally **2-1001** and **1006-4094**.

For ordinary PVLAN host and promiscuous ports, avoid incompatible features such as **EtherChannel (LACP/PAgP), voice VLAN, port security, and `switchport protected`**. Refer to the specific platform's restrictions for additional combinations.

### 6. Misunderstanding Layer 3 Isolation

PVLANs enforce **Layer 2** forwarding restrictions. Hosts may still communicate through a Layer 3 gateway if that gateway routes or otherwise forwards their traffic.

When complete isolation is required, use appropriate ACLs or firewall policies on the gateway in addition to PVLANs.

---

## Key Points

- Configure a **primary VLAN** with `private-vlan primary`.
- Configure **isolated** and **community** secondary VLANs with `private-vlan isolated` and `private-vlan community`.
- Under the primary VLAN, associate secondary VLANs with `private-vlan association <secondary-vlan-list>`.
- A primary VLAN supports **one isolated secondary VLAN** and **multiple community secondary VLANs**.
- An isolated or community host port uses `switchport mode private-vlan host`.
- Assign a host port to one primary/secondary VLAN pair with `switchport private-vlan host-association <primary> <secondary>`.
- Configure a promiscuous port with `switchport mode private-vlan promiscuous`.
- Map eligible secondary VLANs to that port with `switchport private-vlan mapping <primary> <secondary-list>`.
- **Host association, primary VLAN association, and promiscuous port mapping are three separate configuration relationships.**
- The isolated/community distinction comes from the **secondary VLAN type**, not a different host port mode.
- Use `show vlan private-vlan` and `show interfaces <interface> switchport` to verify the configuration.
- A PVLAN restricts **direct Layer 2 communication**; it does not inherently prevent Layer 3 forwarding through the gateway.

---

## References

- [Cisco Catalyst 9300, IOS XE 17.10.x: Configuring Private VLANs](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-10/configuration_guide/vlan/b_1710_vlan_9300_cg/configuring_private_vlans.html)
- [Cisco: Configure Isolated Private VLANs on Catalyst Switches](https://www.cisco.com/c/en/us/support/docs/lan-switching/private-vlans-pvlans-promiscuous-isolated-community/40781-194.html)
