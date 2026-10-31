# 4.2.a (iv) IP Source Guard

**IP Source Guard (IPSG)** is a Layer 2 switch security feature that prevents hosts from sending IPv4 traffic with unauthorized **source IP addresses**. It checks the source address of packets entering a switchport against the addresses permitted for that **interface and VLAN**.

IPSG uses **IP source bindings**, usually learned through [DHCP Snooping](dhcp-snooping/overview.md) or configured manually. It is commonly used on client-facing switchports to prevent **IPv4 source address spoofing**.

For example, if PC1 is permitted to use `192.0.2.11` on `Gi1/0/1`, IPSG prevents PC1 from sending packets with a source address of `192.0.2.12` or `192.0.2.100`.

---

## IP Source Guard Operation

Consider this topology:

```text
                   DHCP Server / Gateway
                            |
                         Gi1/0/24
                            |
                      +-----------+
                      |    SW1    |
                      +-----------+
                       /         \
                 Gi1/0/1         Gi1/0/2
                    |                |
                   PC1              PC2
              192.0.2.11       192.0.2.12
                  VLAN 10          VLAN 10
```

SW1 has learned these bindings:

| Interface | VLAN | Authorized IP | Client MAC |
|---|---:|---|---|
| `Gi1/0/1` | 10 | `192.0.2.11` | `0011.2233.4455` |
| `Gi1/0/2` | 10 | `192.0.2.12` | `0011.2233.4466` |

When IP Source Guard is enabled on `Gi1/0/1`, the switch checks packets **received on Gi1/0/1**:

| Ingress packet | Result with IP-only filtering | Reason |
|---|---|---|
| Source IP `192.0.2.11` | **Permit** | The IP address is authorized on Gi1/0/1, VLAN 10. |
| Source IP `192.0.2.12` | **Drop** | The address belongs to another interface. |
| Source IP `192.0.2.100` | **Drop** | No matching binding exists on Gi1/0/1. |
| DHCPDISCOVER before a lease exists | **Permit** | DHCP traffic necessary to obtain an address is handled by DHCP Snooping. |

IPSG checks **source** addresses, not destination addresses. It applies to packets **entering** the protected port, not packets leaving the port.

It protects both:

- **Intra-VLAN traffic:** PC1 sending IPv4 packets directly to another host in VLAN 10.
- **Inter-VLAN traffic:** PC1 sending IPv4 packets toward its default gateway for routing to another subnet.

The switch installs the permitted source addresses in **hardware forwarding filters**. These filters are updated as bindings are added or removed. IPSG is **disabled by default** and must be configured on the ports to be protected.

---

## IP Source Guard Filtering Modes

IPSG supports two primary filtering modes:

| Mode | Interface command | Validation |
|---|---|---|
| **IP-only** | `ip verify source` | Source IPv4 address, associated interface, and VLAN |
| **IP and MAC** | `ip verify source mac-check` | Source IPv4 address **and** source Ethernet MAC address, associated interface, and VLAN |

### Source IP Address Filtering

Enable IPSG with source IP filtering:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# ip verify source
```

This permits only IPv4 source addresses authorized for this interface and VLAN.

Suppose the binding is:

```text
IP:        192.0.2.11
MAC:       0011.2233.4455
VLAN:      10
Interface: Gi1/0/1
```

A frame entering `Gi1/0/1` with source IP `192.0.2.11` is permitted by the **IP-only** filter, even if the frame's Ethernet source MAC is different. The binding contains a MAC address, but this filtering mode does **not** compare the Ethernet source MAC with it.

**IP-only filtering prevents unauthorized source IP addresses from being used on the port; it does not, by itself, prevent Ethernet MAC address spoofing.**

### Source IP and MAC Address Filtering

Enable both source IP and source MAC verification:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# ip verify source mac-check
```

With the binding above:

| Packet source IP | Packet source MAC | Result |
|---|---|---|
| `192.0.2.11` | `0011.2233.4455` | **Permit** |
| `192.0.2.11` | `0011.2233.9999` | **Drop** |
| `192.0.2.12` | `0011.2233.4455` | **Drop** |
| `192.0.2.12` | `0011.2233.9999` | **Drop** |

Both values must match an authorized binding for the ingress port and VLAN.

`mac-check` is distinct from the DHCP Snooping command `ip dhcp snooping verify mac-address`:

- `ip dhcp snooping verify mac-address` compares the Ethernet source MAC with the DHCP **`chaddr`** field in **DHCP client messages**.
- `ip verify source mac-check` verifies the Ethernet source MAC, along with the source IP, for **traffic protected by IP Source Guard**.

MAC verification behavior depends on the platform's source-filtering implementation. The supported Catalyst IOS XE syntax is `mac-check`; some earlier Catalyst software uses the `port-security` keyword for the equivalent mode.

---

## DHCP Snooping and the Binding Database

The most common IPSG deployment uses bindings learned by **DHCP Snooping**.

The sequence is:

1. DHCP Snooping is enabled globally and for the client's VLAN.
2. The client sends DHCPDISCOVER and DHCPREQUEST messages on an untrusted port.
3. The DHCP server's responses arrive through a trusted port.
4. After the DHCPACK, the switch records the assigned IPv4 address, client MAC, VLAN, interface, and lease time.
5. IPSG uses the binding to permit the client's authorized source IP address.

```text
DHCP exchange
     |
     v
DHCP Snooping Binding Database
     |
     | IP + MAC + VLAN + Interface
     v
IP Source Guard filtering on client port
     |
     +-- Authorized source: PERMIT
     |
     +-- Other IPv4 source: DROP
```

A valid DHCP lease **does not automatically enable IP Source Guard**. DHCP Snooping provides the binding information; `ip verify source` enables enforcement on the interface.

### Behavior Before a Binding Exists

When IPSG is enabled but the port has **no valid IP source binding**, it denies ordinary IPv4 traffic sourced from that port. **DHCP traffic needed to obtain a lease remains permitted**, subject to DHCP Snooping's checks.

For example:

```text
PC1 boots
    |
    | No DHCP binding exists
    v
Ordinary IPv4 data: DROP
DHCPDISCOVER:       PERMIT
    |
    | DHCP completes
    v
Binding learned: 192.0.2.11 on Gi1/0/1, VLAN 10
    |
    v
IPv4 sourced from 192.0.2.11: PERMIT
```

This is an important troubleshooting point: **an enabled interface does not necessarily have an active permit entry**. The binding must be present and match the actual ingress port and VLAN.

### Binding Changes and Aging

IPSG uses the current bindings. If a binding disappears because the DHCP lease expires, the host releases its address, or the binding database is cleared, the corresponding authorization can be removed.

A client that continues using an address after its binding disappears may lose IPv4 connectivity until a valid binding is restored.

For DHCP Snooping binding persistence across reloads:

```text
SW1(config)# ip dhcp snooping database flash:/dhcp-snooping.db
```

See [DHCP Snooping](dhcp-snooping/overview.md) for database persistence and lease-related behavior.

---

## Basic IP Source Guard Configuration

The following example protects two DHCP clients in VLAN 10:

```text
                 DHCP Server
                      |
                 Gi1/0/24
                 SW1 (trunk)
                  /      \
           Gi1/0/1      Gi1/0/2
              |             |
             PC1           PC2
          (DHCP)         (DHCP)
            VLAN 10       VLAN 10
```

### Step 1: Enable DHCP Snooping

```text
SW1(config)# ip dhcp snooping
SW1(config)# ip dhcp snooping vlan 10
```

DHCP Snooping is the source of dynamic client bindings.

### Step 2: Trust the Uplink Toward the DHCP Server

```text
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10
SW1(config-if)# ip dhcp snooping trust
```

### Step 3: Enable IPSG on the Client Ports

```text
SW1(config)# interface range GigabitEthernet1/0/1-2
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 10
SW1(config-if-range)# ip verify source
```

These interfaces remain **untrusted** for DHCP Snooping, which is their default trust state.

IPSG does not require a separate global `ip source guard` enable command. It is activated using `ip verify source` under the interfaces.

### Complete Configuration

```text
ip dhcp snooping
ip dhcp snooping vlan 10
ip dhcp snooping database flash:/dhcp-snooping.db
!
interface range GigabitEthernet1/0/1-2
 switchport mode access
 switchport access vlan 10
 ip verify source
!
interface GigabitEthernet1/0/24
 switchport mode trunk
 switchport trunk allowed vlan 10
 ip dhcp snooping trust
```

Once the clients obtain DHCP addresses, check the bindings and active source filters:

```text
SW1# show ip dhcp snooping binding
SW1# show ip source binding
SW1# show ip verify source
```

**Do not apply IPSG to the uplink as though it were an ordinary single-client port.** The uplink can carry traffic from many different hosts and is trusted for DHCP Snooping.

---

## Static IP Source Bindings

A host configured with a **static IPv4 address** does not obtain a DHCP lease, so DHCP Snooping does not dynamically create a binding for it.

You can authorize that host with a **static IP source binding**.

For example, PC3 uses:

```text
IP address: 192.0.2.50/24
MAC:        0011.2233.4477
VLAN:       10
Port:       Gi1/0/3
```

Configure the binding **globally**, then enable IPSG on the client-facing interface:

```text
SW1(config)# ip source binding 0011.2233.4477 vlan 10 192.0.2.50 interface GigabitEthernet1/0/3
!
SW1(config)# interface GigabitEthernet1/0/3
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# ip verify source
```

A static source binding has four important identifying fields:

```text
ip source binding <mac-address> vlan <vlan-id> <ip-address> interface <interface-id>
```

Even in IP-only filtering mode, the static binding command includes the host's MAC address. **That does not mean IP-only IPSG verifies the packet's MAC address.**

To enable IP and MAC filtering for PC3:

```text
SW1(config-if)# ip verify source mac-check
```

### Static Binding vs. Static DHCP Reservation

A DHCP server can reserve an IP address for a particular client MAC address. That is still a **DHCP assignment**: the client requests the address through DHCP, and DHCP Snooping can learn a dynamic binding.

By contrast, a host configured manually with `192.0.2.50` does not exchange DHCP messages. A DHCP server reservation alone does **not** create an IPSG binding for that host.

---

## IP Source Guard for Static Hosts Using Device Tracking

IPSG also supports a **device-tracking mode** that can learn addresses used by non-DHCP hosts. This is different from manually configuring `ip source binding` entries.

With `ip verify source tracking`, the switch uses **device tracking** to discover host IP/MAC information through ARP or other IP traffic and maintain source-filter entries. A maximum number of learned host addresses can be configured on a port.

For example, enable tracking-based IPSG on an access port:

```text
SW1(config)# interface GigabitEthernet1/0/4
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# device-tracking
SW1(config-if)# ip verify source tracking mac-check
SW1(config-if)# ip device tracking maximum 4
```

- `device-tracking`: Enables device tracking on the interface.
- `ip verify source tracking`: Uses device-tracking information for source filtering, including static-IP hosts.
- `mac-check`: Also validates the source MAC address.
- `ip device tracking maximum 4`: Limits the number of tracked IP addresses to four on that port.

Device-tracking IPSG can also work with DHCP clients. This mode is especially useful where endpoints use manually assigned IP addresses and maintaining individual static bindings would be cumbersome.

**Do not confuse automatic device-tracking learning with an explicitly authorized static IP source binding.** A manual binding fixes the allowed IP/MAC/VLAN/port association. Tracking-based learning derives host information from observed traffic and has different operational and security implications.

To inspect device-tracking entries:

```text
SW1# show device-tracking database
```

Some platforms also provide:

```text
SW1# show ip device tracking all
```

---

## IP Source Guard on Trunk Ports and Multiple VLANs

IPSG is not restricted to access ports. It can also filter traffic on supported **Layer 2 trunk ports**.

If a port carries multiple VLANs, the permitted source IP addresses are evaluated in the context of the **ingress VLAN and interface**. For example:

| Interface | VLAN | Authorized source |
|---|---:|---|
| `Gi1/0/5` | 10 | `192.0.2.21` |
| `Gi1/0/5` | 20 | `198.51.100.21` |

A packet received on `Gi1/0/5` in VLAN 10 using source `198.51.100.21` is not authorized by the VLAN 20 binding.

For DHCP-based IPSG, enable DHCP Snooping for the VLANs being protected:

```text
SW1(config)# ip dhcp snooping
SW1(config)# ip dhcp snooping vlan 10,20
!
SW1(config)# interface GigabitEthernet1/0/5
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,20
SW1(config-if)# ip verify source
```

IPSG on a trunk can be useful for a controlled downstream endpoint, but it should **not** be applied indiscriminately to infrastructure uplinks or switch-to-switch trunks. For a trunk hosting clients, ensure legitimate bindings can be learned and that the port is not inadvertently configured as a trusted DHCP Snooping interface.

The separate **device-tracking IPSG mode for static hosts** should be used on access ports, not uplinks or trunks.

---

## Packet Types and Filtering Scope

IPSG is primarily an **IPv4 source-address security feature**.

| Traffic | IP-only IPSG behavior |
|---|---|
| Ordinary IPv4 packets | Source IP checked against allowed bindings for the ingress VLAN and interface |
| DHCP client traffic | Permitted as needed for address assignment, subject to DHCP Snooping |
| ARP | Not validated by the IPv4 source-IP filter |
| IPv6 | Not validated by IPv4 IPSG |
| Inbound traffic toward the client | Not source-filtered by IPSG on the client port's egress direction |

### Why IPSG Does Not Stop ARP Spoofing

ARP is an Ethernet protocol with EtherType `0x0806`, **not an IPv4 data packet**. IP-only Source Guard does not examine ARP sender information to validate address ownership.

For ARP spoofing protection, use [Dynamic ARP Inspection](dynamic-arp-inspection.md), which can validate ARP messages using DHCP Snooping bindings or configured ARP ACLs.

### IPv6 Is a Separate Security Feature

IPv4 IPSG does not protect IPv6 source addresses. IPv6 source protection uses **IPv6 Source Guard** and related IPv6 First Hop Security mechanisms, typically working with device tracking and Neighbor Discovery inspection.

---

## DHCP Snooping, IPSG, DAI, and Port Security

These mechanisms protect different aspects of host traffic:

| Feature | Main inspection | Prevents |
|---|---|---|
| **DHCP Snooping** | DHCP message types, sender validation, and bindings | Rogue DHCP servers and invalid DHCP messages |
| **IP Source Guard** | IPv4 source address; optionally source MAC | Unauthorized IPv4 source addresses on an interface |
| **Dynamic ARP Inspection (DAI)** | ARP sender IP/MAC information | ARP spoofing and ARP poisoning |
| **Port Security** | Ethernet source MAC addresses and learned MAC count | Unauthorized source MACs or excessive MAC addresses |

These features complement one another:

```text
                 DHCP Snooping
                       |
                  Binding Table
                   /       \
                  /         \
                IPSG        DAI
          (IPv4 data)   (ARP messages)
                  
          Port Security
       (Ethernet source MAC)
```

For example:

- **DHCP Snooping alone:** PC1 may still forge a source IPv4 address in ordinary data packets.
- **IP Source Guard alone:** PC1 is blocked from using unauthorized IPv4 addresses, but forged ARP messages require separate protection.
- **DAI alone:** ARP spoofing can be blocked, but the source IPv4 addresses in ordinary packets are not automatically checked by DAI.
- **Port Security alone:** A permitted MAC address can still be used to send packets with forged source IPv4 addresses.

---

## Verification

### Check IP Source Guard Enforcement

```text
SW1# show ip verify source
SW1# show ip verify source interface GigabitEthernet1/0/1
```

The output identifies the configured filtering mode and authorized source addresses. The exact columns can include:

```text
Interface  Filter-type  Filter-mode  IP-address    Mac-address        Vlan
---------  -----------  -----------  ------------  -----------------  ----
Gi1/0/1    ip           active       192.0.2.11                       10
Gi1/0/2    ip           active       deny-all                         10
Gi1/0/3    ip-mac       active       192.0.2.50    0011.2233.4477     10
```

Interpretation:

- `ip`: IP-only filtering.
- `ip-mac`: IP and MAC filtering.
- `active`: An active source filter is applied.
- `deny-all`: No authorized source binding exists for that interface/VLAN, so ordinary IPv4 traffic is denied.

Some output may show an inactive filter, for example where the interface is trusted for DHCP Snooping or DHCP Snooping is not enabled for the relevant VLAN. **A configured `ip verify source` line is not proof that the desired filter is active.**

### Check the Source Binding Table

```text
SW1# show ip source binding
```

This displays the source bindings available to IP Source Guard, including both **dynamic DHCP Snooping bindings** and **manually configured static bindings**.

Example:

```text
MacAddress          IpAddress      Lease(sec)  Type           VLAN  Interface
------------------  -------------  ----------  -------------  ----  ---------
00:11:22:33:44:55   192.0.2.11     68000       dhcp-snooping   10    Gi1/0/1
00:11:22:33:44:77   192.0.2.50     infinite    static          10    Gi1/0/3
```

Compare the binding with the **actual source IP, physical input interface, and VLAN** of the traffic being dropped.

### Check DHCP Snooping

```text
SW1# show ip dhcp snooping
SW1# show ip dhcp snooping binding
SW1# show ip dhcp snooping statistics
```

If the dynamic binding is missing, determine whether DHCP Snooping is enabled on the correct VLAN and whether DHCP traffic is passing successfully.

### Check the Interface Configuration

```text
SW1# show running-config interface GigabitEthernet1/0/1
SW1# show interfaces GigabitEthernet1/0/1 switchport
SW1# show vlan brief
```

These commands help verify that IPSG is attached to the expected **Layer 2 port** and VLAN.

---

## Troubleshooting Scenarios

### Scenario 1: DHCP Works, but IPv4 Traffic Fails

A client has a valid DHCP-assigned address, but pings and other IPv4 traffic are dropped.

Check:

```text
SW1# show ip dhcp snooping binding
SW1# show ip source binding
SW1# show ip verify source interface GigabitEthernet1/0/1
```

If the DHCP lease is visible on the client but **no matching binding** appears on the switch, IPSG may have only a deny-all filter. Check whether the DHCPACK traversed SW1 on the expected VLAN and through the expected trusted uplink.

### Scenario 2: A Static-IP Host Cannot Communicate

The host did not use DHCP, so there is no dynamic binding.

Solutions include:

- Configure a **static `ip source binding`** for the host's IP/MAC/VLAN/port.
- Use the **device-tracking mode** of IPSG where it meets the security requirement.

Do not assume that adding a static IP address to the host automatically adds an authorization to the switch.

### Scenario 3: Source IP Is Correct, but Traffic Is Dropped

If the port uses `ip verify source mac-check`, the source **MAC** must also match the binding.

Check the host's actual Ethernet MAC address and its binding. A host using a different NIC, a changed virtual NIC address, or a spoofed MAC might pass an IP-only check but fail the IP-and-MAC check.

### Scenario 4: The Port Shows an Inactive Filter

Check that:

1. DHCP Snooping is enabled globally and for the client's VLAN.
2. The port is a supported Layer 2 interface.
3. The client port is **untrusted** for DHCP Snooping.
4. The active source binding is associated with the correct interface and VLAN.

### Scenario 5: Connectivity Fails After a Reload

If DHCP Snooping bindings are not restored, the IPSG permit filters may be missing until clients renew their leases.

Configure a DHCP Snooping database agent when dynamic binding persistence is required:

```text
SW1(config)# ip dhcp snooping database flash:/dhcp-snooping.db
```

### Scenario 6: Removing IPSG Restores Connectivity

To temporarily disable IPSG on a port while troubleshooting:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# no ip verify source
```

This removes the IPSG source filter from the port. It does **not** disable DHCP Snooping globally or delete the source binding database.

---

## Commands Summary

| Command | Mode | Purpose |
|---|---|---|
| `ip dhcp snooping` | Global | Enable DHCP Snooping. |
| `ip dhcp snooping vlan 10` | Global | Enable DHCP Snooping in VLAN 10. |
| `ip dhcp snooping trust` | Interface | Permit DHCP server messages on the incoming port. |
| `ip verify source` | Interface | Enable IPSG with source IPv4 address filtering. |
| `ip verify source mac-check` | Interface | Enable IPSG with source IPv4 and MAC address filtering. |
| `no ip verify source` | Interface | Disable IPSG on a port. |
| `ip source binding ...` | Global | Configure a static IP/MAC/VLAN/interface binding. |
| `ip verify source tracking` | Interface | Use device-tracking bindings for IPSG, including static hosts. |
| `ip verify source tracking mac-check` | Interface | Enable tracking-based IPSG with IP and MAC filtering. |
| `ip device tracking maximum 4` | Interface | Set a maximum of four tracked IP addresses on the port. |
| `show ip verify source` | Privileged EXEC | Verify IPSG filters and their operational state. |
| `show ip source binding` | Privileged EXEC | View dynamic and static IP source bindings. |
| `show ip dhcp snooping binding` | Privileged EXEC | View learned DHCP Snooping bindings. |
| `show device-tracking database` | Privileged EXEC | Inspect device-tracking bindings. |

---

## Key Points

- **IP Source Guard prevents IPv4 source address spoofing** on protected Layer 2 switchports.
- IPSG checks traffic **entering the port**, not traffic leaving it.
- `ip verify source` enables **IP-only** filtering; `ip verify source mac-check` enables **IP and MAC** filtering.
- DHCP Snooping normally supplies dynamic bindings. **Enabling DHCP Snooping does not automatically enable IPSG.**
- If no valid binding exists, ordinary IPv4 packets from the protected port are **dropped**, while DHCP can still obtain a lease.
- Static-IP hosts can be authorized with **`ip source binding`** or supported **device-tracking IPSG**.
- IPSG does **not** replace **Dynamic ARP Inspection** for ARP spoofing protection.
- IPv4 IPSG does **not** protect IPv6 source addresses.
- Verify actual enforcement with **`show ip verify source`** and binding state with **`show ip source binding`**.

---

## References

- [Cisco: IP Source Guard, FHS and SISF Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/sec-crypto/fhs-sisf/fhs-and-sisf-configuration-guide/ip-source-guard.html)
- [Cisco: Configuring IP Source Guard, Catalyst 9300 IOS XE](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-15/configuration_guide/sec/b_1715_sec_9300_cg/configuring_ip_source_guard.html)
- [Cisco: IPv6 Source Guard and Prefix Guard](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_ipv6-src-guard-xe.html)
