# 4.2.a (iii) DHCP Snooping and DHCP Option 82

**DHCP snooping** is a Layer 2 switch security feature that inspects DHCP messages, blocks unauthorized DHCP server traffic, and records legitimate DHCP address assignments in a **DHCP snooping binding database**.

Its two main functions are:

1. **DHCP message filtering:** Prevents devices on untrusted switch ports from acting as DHCP servers and rejects certain invalid client messages.
2. **Binding database creation:** Records which DHCP-assigned IPv4 address belongs to which MAC address, VLAN, and switch interface.

The binding database can also be used by [IP Source Guard](../ip-source-guard.md) and [Dynamic ARP Inspection](../dynamic-arp-inspection.md).

**DHCP Option 82** lets a switch add information identifying the client's point of attachment to DHCP requests. It is introduced below and covered in detail on the [DHCP Option 82](option82.md) page.

---

## Why DHCP Snooping Is Needed

DHCP clients generally accept configuration information from DHCP servers without authenticating the servers. A device connected to an ordinary access port can therefore attempt to act as a **rogue DHCP server**.

A rogue server might distribute:

- An incorrect default gateway, redirecting clients' traffic through an attacker.
- Incorrect DNS server addresses, redirecting name resolution.
- Invalid IP configuration, preventing clients from communicating.

For example:

```text
                   Legitimate DHCP Server
                            |
                            | DHCPOFFER
                            |
                        Gi1/0/24
                       +--------+
                       |  SW1   |
                       +--------+
                      /          \
               Gi1/0/1          Gi1/0/2
                  |                 |
               Client1          Rogue DHCP
                                   Server
                                      |
                                      +-- DHCPOFFER
```

Without DHCP snooping, the client might receive competing offers from the legitimate server and rogue server.

With DHCP snooping, SW1 can allow the legitimate server's replies from `Gi1/0/24` while dropping DHCP server messages received from `Gi1/0/2`.

---

## Trusted and Untrusted Interfaces

DHCP snooping assigns switch interfaces one of two **trust states**:

| Trust state | Typical connection | DHCP behavior |
|---|---|---|
| **Trusted** | Legitimate DHCP server, DHCP relay, or uplink toward them | DHCP server messages are permitted on ingress. |
| **Untrusted** | End-user device or other untrusted network | DHCP server messages received on ingress are dropped; client messages are inspected. |

**All interfaces are untrusted by default.** Trust is configured explicitly on the interfaces that should receive legitimate DHCP server messages.

```text
               DHCP Server / DHCP Relay
                          |
                          |  Trusted
                          |
                       Gi1/0/24
                      +--------+
                      |  SW1   |
                      +--------+
                     /          \
              Gi1/0/1          Gi1/0/2
                 |                 |
              Client1          Client2
              Untrusted        Untrusted
```

The trust state applies to **packets entering an interface**. It does not mean that DHCP server messages cannot be transmitted *out* an untrusted interface toward a client.

For example:

- A DHCPDISCOVER enters `Gi1/0/1` from a client. It is permitted and forwarded toward the legitimate server.
- A DHCPOFFER enters `Gi1/0/24` from the legitimate server. It is permitted and can be forwarded out `Gi1/0/1` toward the client.
- A DHCPOFFER enters `Gi1/0/2` from a rogue server. It is dropped.

An uplink is not automatically trusted simply because it is a trunk. Its DHCP snooping trust state must be configured.

### DHCP Message Types

DHCP snooping distinguishes messages sent by DHCP clients from messages sent by DHCP servers.

| Message | Sender | Purpose | Received on untrusted port |
|---|---|---|---|
| DHCPDISCOVER | Client | Locate DHCP servers | Permitted if validation succeeds |
| DHCPOFFER | Server | Offer an address and options | **Dropped** |
| DHCPREQUEST | Client | Request or renew an address | Permitted if validation succeeds |
| DHCPACK | Server | Confirm an address assignment | **Dropped** |
| DHCPNAK | Server | Reject an address request | **Dropped** |
| DHCPDECLINE | Client | Report that an offered address is already in use | Checked against binding information |
| DHCPRELEASE | Client | Release an address | Checked against binding information |
| DHCPINFORM | Client | Request configuration options without requesting a lease | Permitted if validation succeeds |

Other server-originated messages, such as DHCPLEASEQUERY replies, are likewise not accepted from ordinary untrusted user ports.

---

## DHCP Snooping During DORA

A normal DHCP address assignment follows the **DORA** sequence:

1. **Discover:** The client broadcasts a DHCPDISCOVER.
2. **Offer:** The server sends a DHCPOFFER.
3. **Request:** The client sends a DHCPREQUEST.
4. **Acknowledgment:** The server sends a DHCPACK.

DHCP snooping inspects these messages as they cross the switch.

```text
       DHCP Client              SW1              DHCP Server
           |                     |                     |
           |--- DHCPDISCOVER --->|-------------------->|
           |                     |                     |
           |<--------------------|<---- DHCPOFFER -----|
           |                     |                     |
           |--- DHCPREQUEST ---->|-------------------->|
           |                     |                     |
           |<--------------------|<---- DHCPACK -------|
           |                     |                     |
           |                     |                     |
           |                     +-- Create binding --+
```

The client-facing interface is **untrusted**, and the interface receiving the server's messages is **trusted**.

After seeing the successful DHCP acknowledgment, the switch can create a dynamic binding for the client. A DHCPOFFER alone does **not** establish a completed address assignment.

DHCP snooping operates while the switch forwards DHCP traffic. It does **not** require SW1 to be the DHCP server or DHCP relay agent.

---

## DHCP Snooping Binding Database

The **DHCP snooping binding database** records legitimate DHCP address assignments learned on untrusted interfaces.

Each dynamic entry includes:

| Field | Meaning |
|---|---|
| **MAC address** | Client MAC address |
| **IP address** | IPv4 address assigned to the client |
| **Lease** | Remaining DHCP lease time |
| **Type** | Dynamic or another binding type |
| **VLAN** | VLAN where the client is connected |
| **Interface** | Switch interface where the client's DHCP traffic was received |

Example:

```text
SW1# show ip dhcp snooping binding
MacAddress          IpAddress       Lease(sec)  Type           VLAN  Interface
------------------  --------------  ----------  -------------  ----  ---------
00:11:22:33:44:55   192.0.2.10      68000       dhcp-snooping   10    Gi1/0/1
00:11:22:33:44:66   192.0.2.11      72000       dhcp-snooping   10    Gi1/0/2
```

The first row means that MAC address `0011.2233.4455` has been assigned `192.0.2.10` on VLAN 10 through `Gi1/0/1`.

### Learning and Removing Bindings

- The switch learns dynamic bindings from successful DHCP address assignments, particularly the DHCPACK.
- Renewals can update the lease information.
- Bindings expire when their DHCP leases expire.
- A valid DHCPRELEASE can remove the corresponding binding.
- Bindings associated with disconnected interfaces can also be removed or aged out.

The database normally contains **clients learned on untrusted interfaces**. It does not learn ordinary client bindings from traffic received on trusted interfaces.

Hosts configured with **static IPv4 addresses** do not create dynamic DHCP snooping bindings because they do not obtain their addresses through DHCP. Use static source bindings for **IP Source Guard** and ARP ACLs where needed for **Dynamic ARP Inspection** on statically addressed hosts.

### Persistence Across Reloads

Dynamic bindings are held in memory. Without a configured **DHCP snooping database agent**, the switch does not automatically preserve them across a reload.

The database agent writes bindings to a file so they can be restored when the switch restarts.

```text
SW1(config)# ip dhcp snooping database flash:/dhcp-snooping.db
```

The file can also be stored on a supported remote server, such as TFTP:

```text
SW1(config)# ip dhcp snooping database tftp://192.0.2.100/dhcp-snooping.db
```

Database writes are batched rather than necessarily performed after every individual binding change.

```text
SW1(config)# ip dhcp snooping database write-delay 60
SW1(config)# ip dhcp snooping database timeout 300
```

- `write-delay 60`: Wait 60 seconds after binding changes before writing the database file.
- `timeout 300`: Allow up to 300 seconds for a database transfer to complete.

The default for both timers is **300 seconds**.

This persistence matters particularly when **IP Source Guard** or **Dynamic ARP Inspection** uses the bindings to permit traffic. Losing the bindings after a reload can temporarily cause valid hosts' traffic to be rejected until bindings are learned again.

### Static Bindings

For hosts with statically assigned addresses, configure a **static IP source binding** rather than relying on DHCP learning when using IP Source Guard.

For example:

```text
SW1(config)# ip source binding 0011.2233.4477 vlan 10 192.0.2.50 interface GigabitEthernet1/0/3
```

Use `show ip source binding` to inspect the combined source binding information, including configured static bindings. `show ip dhcp snooping binding` primarily displays dynamically learned DHCP snooping entries.

---

## DHCP Packet Validation

DHCP snooping performs checks beyond simply blocking DHCPOFFER and DHCPACK messages on untrusted interfaces.

### Source MAC Address Verification

By default, the switch compares:

- The **source MAC address** in the Ethernet header.
- The **client hardware address (`chaddr`)** inside the DHCP message.

A mismatch causes the DHCP client packet received on an untrusted port to be dropped.

```text
Ethernet source MAC: 0011.2233.4455
DHCP chaddr:         00aa.bbcc.ddee

Result: Dropped (MAC verification failure)
```

MAC verification is enabled by default. It can be explicitly configured or disabled globally:

```text
SW1(config)# ip dhcp snooping verify mac-address
SW1(config)# no ip dhcp snooping verify mac-address
```

This check helps prevent a host from sending DHCP messages on behalf of another MAC address.

### DHCPRELEASE and DHCPDECLINE Validation

A malicious device could attempt to release another client's address or report that the address is unusable.

For DHCPRELEASE and DHCPDECLINE, DHCP snooping checks the relevant binding information. If the packet arrives on a different interface from the one recorded for that client, the switch drops it.

For example:

```text
Binding database:
  MAC:       0011.2233.4455
  Interface: Gi1/0/1

DHCPRELEASE for that client arrives on Gi1/0/2.

Result: Dropped (binding/interface mismatch)
```

### Relay Information Validation

DHCP snooping also checks relay-related information, including the `giaddr` field and Option 82. Client-facing untrusted ports are not normally allowed to supply packets that claim to have come through a DHCP relay agent.

The rules and supported exceptions are covered in the [DHCP Option 82](option82.md) page.

---

## Configuring DHCP Snooping

The following example protects VLANs **10 and 20** on SW1. Client interfaces are `Gi1/0/1` and `Gi1/0/2`. The uplink toward the legitimate DHCP server or relay agent is `Gi1/0/24`.

### Step 1: Enable DHCP Snooping Globally

```text
SW1(config)# ip dhcp snooping
```

This enables the DHCP snooping feature globally, but **does not enable snooping on any VLAN by itself**.

### Step 2: Enable DHCP Snooping on the Required VLANs

```text
SW1(config)# ip dhcp snooping vlan 10,20
```

The switch inspects DHCP traffic in these VLANs according to the snooping configuration.

A range is also supported:

```text
SW1(config)# ip dhcp snooping vlan 10-20
```

### Step 3: Configure the Trusted Interface

```text
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,20
SW1(config-if)# ip dhcp snooping trust
```

**Only the interfaces on which legitimate DHCP server traffic can enter should be trusted.**

### Step 4: Configure DHCP Rate Limiting on Client Ports

```text
SW1(config)# interface range GigabitEthernet1/0/1-2
SW1(config-if-range)# ip dhcp snooping limit rate 15
```

This limits DHCP traffic received on each interface to **15 DHCP packets per second (pps)**.

The command is configured **per interface**, not per VLAN. There is no DHCP snooping rate limit configured by default.

If the configured rate is exceeded, the switch can place the interface into the **err-disabled** state, interrupting traffic on that interface.

To remove the configured rate limit:

```text
SW1(config-if)# no ip dhcp snooping limit rate
```

Do not use a low limit on an uplink that aggregates DHCP traffic from many clients.

### Step 5: Configure Binding Database Persistence

```text
SW1(config)# ip dhcp snooping database flash:/dhcp-snooping.db
```

This enables the database agent to maintain a binding file for restoration after a reload.

### Complete Example

```text
ip dhcp snooping
ip dhcp snooping vlan 10,20
ip dhcp snooping database flash:/dhcp-snooping.db
!
interface range GigabitEthernet1/0/1-2
 switchport mode access
 switchport access vlan 10
 ip dhcp snooping limit rate 15
!
interface GigabitEthernet1/0/24
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 ip dhcp snooping trust
```

The client interfaces remain **untrusted** because that is the default.

---

## DHCP Starvation Attacks

A **DHCP starvation attack** sends large numbers of DHCP requests using different client identities to exhaust a DHCP server's available address pool.

**DHCP snooping rate limiting** helps constrain the rate of requests from a user-facing switch port. **MAC address verification** also prevents a device from using one Ethernet source MAC address while supplying different `chaddr` values in the DHCP messages.

However, an attacker who changes both the Ethernet source MAC and DHCP client hardware address can satisfy the MAC verification check. Combining DHCP snooping with **port security** can further limit the number of source MAC addresses learned on a port.

---

## DHCP Snooping Rate-Limit Recovery

When a rate-limit violation places an interface into the err-disabled state, it normally requires intervention unless automatic recovery has been enabled.

To enable automatic recovery for DHCP rate-limit violations:

```text
SW1(config)# errdisable recovery cause dhcp-rate-limit
SW1(config)# errdisable recovery interval 300
```

The switch can then attempt to recover the affected interface after **300 seconds**.

To check the error state and recovery configuration:

```text
SW1# show interfaces status err-disabled
SW1# show errdisable recovery
```

After correcting the underlying problem, an interface can also be recovered manually:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# shutdown
SW1(config-if)# no shutdown
```

A large DHCP burst may be an attack, but it may also occur when many clients reconnect through one interface. Set the limit according to the expected number of clients behind that port.

---

## DHCP Option 82: Basic Relationship to Snooping

**DHCP Option 82**, also called the **Relay Agent Information Option**, carries information identifying the network attachment point through which a client's DHCP message arrived.

Two important suboptions are:

| Suboption | Purpose |
|---|---|
| **Circuit ID** | Identifies the client-facing circuit or switch port, typically including VLAN and port information. |
| **Remote ID** | Identifies the switch or other device inserting Option 82. |

When DHCP snooping is enabled, **Option 82 insertion is enabled by default**.

```text
SW1(config)# ip dhcp snooping information option
```

To disable insertion and removal of Option 82 information by the snooping feature:

```text
SW1(config)# no ip dhcp snooping information option
```

Option 82 and DHCP snooping work together, but they are not the same feature:

- **DHCP snooping** filters DHCP messages and learns client bindings.
- **Option 82** supplies client attachment information to DHCP servers for identification and policy decisions.

A DHCP relay agent's `ip dhcp relay information option` configuration belongs to the DHCP **relay** feature. Do not confuse it with `ip dhcp snooping information option`, which controls snooping-related Option 82 processing.

See [DHCP Option 82](option82.md) for insertion/removal behavior, `giaddr`, trusted and untrusted interfaces, `allow-untrusted`, and DHCP relay interactions.

---

## Verification and Troubleshooting

### Show DHCP Snooping Configuration

```text
SW1# show ip dhcp snooping
```

Check:

- Whether DHCP snooping is enabled globally.
- Which VLANs have snooping configured and operational.
- Which interfaces are trusted.
- Whether rate limits are configured.
- Whether Option 82 insertion and MAC verification are enabled.

An enabled global feature is insufficient if the client VLAN is absent from the operational VLAN list.

### Show DHCP Snooping Bindings

```text
SW1# show ip dhcp snooping binding
```

This shows dynamically learned DHCP snooping bindings. For a successfully configured DHCP client, check the **MAC address, IPv4 address, VLAN, interface, and lease time**.

For combined dynamic and static source bindings:

```text
SW1# show ip source binding
```

### Show Binding Database Agent Status

```text
SW1# show ip dhcp snooping database
SW1# show ip dhcp snooping database detail
```

Use these commands to verify database file transfers and persistence.

### Show DHCP Snooping Statistics

```text
SW1# show ip dhcp snooping statistics
SW1# show ip dhcp snooping statistics detail
```

These commands help identify DHCP packets dropped because of message-type filtering, MAC verification, relay-information validation, or other snooping checks.

To clear the counters before reproducing a problem:

```text
SW1# clear ip dhcp snooping statistics
```

---

## DHCP Snooping, IP Source Guard, and DAI

The three features provide different types of protection:

| Feature | What it checks | Main purpose |
|---|---|---|
| **DHCP Snooping** | DHCP messages and client address assignments | Prevent rogue DHCP servers and build the binding database |
| **IP Source Guard** | Source IPv4 address, and optionally MAC address, on incoming traffic | Prevent hosts from using unauthorized source addresses |
| **Dynamic ARP Inspection** | ARP sender information against trusted bindings or other permitted entries | Prevent ARP spoofing |

For a DHCP client:

```text
DHCPDISCOVER / DHCPOFFER / DHCPREQUEST / DHCPACK
                         |
                         v
                DHCP Snooping
                         |
                         v
                Binding Database
               /                \
              v                  v
       IP Source Guard    Dynamic ARP Inspection
       (IP filtering)        (ARP validation)
```

**DHCP snooping does not automatically enable IP Source Guard or Dynamic ARP Inspection.** Those features require their own configuration.

DHCP snooping also does not generally filter ordinary non-DHCP IP traffic based on the binding database. That is the purpose of features such as IP Source Guard.

---

## Default Settings

| Setting | Default |
|---|---|
| Global DHCP snooping | **Disabled** |
| DHCP snooping per VLAN | **Disabled** |
| Interface trust | **Untrusted** |
| DHCP snooping rate limit | **No limit configured** |
| DHCP client MAC verification | **Enabled** |
| Option 82 insertion | **Enabled** when snooping is operating |
| Acceptance of pre-existing Option 82 on untrusted ports | **Disabled** |
| Binding database persistence | **No destination configured** |
| Database `write-delay` | **300 seconds** |
| Database transfer `timeout` | **300 seconds** |

---

## Key Points

- DHCP snooping is enabled **globally and per VLAN**.
- Switch ports are **untrusted by default**. Trust the interfaces receiving legitimate DHCP server traffic.
- Server-originated DHCP messages such as DHCPOFFER, DHCPACK, and DHCPNAK are **dropped when received on untrusted ports**.
- DHCP snooping builds bindings from successful DHCP assignments on **untrusted client-facing interfaces**.
- DHCP client MAC verification is **enabled by default**.
- DHCP snooping rate limiting is **optional**. A violation can err-disable the interface.
- Database persistence allows bindings to be **restored after reloads**.
- Option 82 insertion is **enabled by default**, but can be disabled independently of DHCP snooping.
- IP Source Guard and Dynamic ARP Inspection use binding information but are **separately configured features**.

---

## References

- [Cisco: Configuring DHCP, Catalyst 9300 IOS XE](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-13/configuration_guide/ip/b_1713_ip_9300_cg/configuring_dhcp.html)
- [Cisco: Operate and Troubleshoot DHCP Snooping on Catalyst 9000 Switches](https://www.cisco.com/c/en/us/support/docs/ip/dynamic-host-configuration-protocol-dhcp-dhcpv6/217055-operate-and-troubleshoot-dhcp-snooping.html)
