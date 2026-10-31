# DHCP Option 82

**DHCP Option 82**, also called the **Relay Agent Information Option**, allows a network device to attach information about a DHCP client's point of attachment to a DHCPv4 request. The DHCP server can use this information to identify where the client is connected and apply address-assignment policies.

Option 82 is commonly inserted by either:

1. A **Layer 2 switch** running DHCP snooping, even if that switch is not acting as a DHCP relay agent.
2. A **Layer 3 DHCP relay agent**, such as a router or multilayer switch configured with `ip helper-address`.

These two functions use **different commands** and have **different trust controls**.

This page covers Option 82 in detail. General DHCP snooping operation and its binding database are covered in [DHCP Snooping](overview.md).

---

## Option 82 Fundamentals

A client normally knows nothing about Option 82. It sends ordinary DHCP messages, such as DHCPDISCOVER and DHCPREQUEST.

A switch or relay agent inserts Option 82 before forwarding the request toward the server:

```text
Client                  SW1                     R1                  DHCP Server
                       (L2 switch)            (Relay)
  |                       |                      |                       |
  |--- DHCPDISCOVER ----->|                      |                       |
  |   no Option 82        |                      |                       |
  |                       |--- DHCPDISCOVER ---->|                       |
  |                       |   + Option 82        |                       |
  |                       |                      |---- DHCPDISCOVER ---->|
  |                       |                      |    + Option 82        |
  |                       |                      |    giaddr = R1 IP     |
  |                       |                      |                       |
  |                       |                      |<----- DHCPOFFER ------|
  |                       |                      |    echoes Option 82   |
  |<----- DHCPOFFER ------|<----- DHCPOFFER -----|                       |
  |   no Option 82        |   no Option 82       |                       |
```

The upstream devices must be configured to accept Option 82 that was inserted farther downstream. If they are not, DHCP requests can be dropped before reaching the server.

### Option 82 Suboptions

Option 82 is DHCP **option code 82**. Its contents are made up of one or more *suboptions*.

| Suboption | Code | Purpose |
|---|---:|---|
| **Circuit ID** | 1 | Identifies the client's attachment circuit, typically its VLAN and switch port. |
| **Remote ID** | 2 | Identifies the switch or relay agent inserting Option 82. |
| **Subscriber ID** | 6 | Optional identifier for a particular subscriber, configured on a relay agent. |

The most important suboptions for switch-security labs are **Circuit ID** and **Remote ID**.

For example:

```text
Client connected to SW1 Gi1/0/3 in VLAN 10

Option 82
  Circuit ID: VLAN 10 / SW1 Gi1/0/3
  Remote ID:  SW1 MAC address
```

The server can distinguish two clients connected through different switch ports, even if both requests reach it through the same relay-agent address.

### Packet Format

Option 82 contains a code, a length, and a sequence of suboptions:

```text
+-----------+--------------+--------------------------------------+
| Code = 82 | Option length| Suboption 1 | Suboption 2 |   ...    |
+-----------+--------------+--------------------------------------+
                         
Each suboption:
+-------------+------------------+-------------------------+
| Suboption ID| Suboption length | Suboption value         |
+-------------+------------------+-------------------------+
```

---

## Option 82 vs. `giaddr`

The DHCP header contains a **Gateway IP Address** field, called `giaddr`. It is **not part of Option 82**.

| Field | Location | Purpose |
|---|---|---|
| **`giaddr`** | Fixed field in the DHCP/BOOTP header | Identifies the relay address used to select the client subnet and return DHCP replies. |
| **Option 82** | DHCP options area | Provides additional information about the client's attachment point. |

These two fields serve different purposes and are not interchangeable.

### Layer 2 Switch Insertion

When SW1 only performs Layer 2 switching and DHCP snooping, it can insert Option 82 without becoming a DHCP relay agent.

This allows SW1 to **identify the client's physical attachment point** to the DHCP server. For example, the **Circuit ID** can identify the VLAN and switch port, while the **Remote ID** identifies the switch. The DHCP server can use this information to assign addresses or apply policies based on where the client is connected.

```text
DHCPDISCOVER leaving SW1:
  giaddr:     0.0.0.0
  Option 82:  Present (inserted by SW1)
```

The Layer 2 switch does **not** automatically set `giaddr` to an SVI address simply because it inserts Option 82. A subsequent DHCP relay agent normally supplies the appropriate nonzero `giaddr` when forwarding the packet to a server on another subnet.

Option 82 insertion is **not required for DHCP snooping** to function and can be disabled independently.

> **Important:** Option 82 insertion can cause DHCP failures when an upstream DHCP snooping switch or DHCP relay agent does not trust the received Option 82 information. By default, snooping switches reject DHCP packets containing Option 82 on untrusted ports, and relay agents may reject packets containing Option 82 with `giaddr = 0.0.0.0`. Configure the appropriate trust settings or disable Option 82 insertion on SW1.

### DHCP Relay Insertion

R1 receives the client's broadcast on `GigabitEthernet0/0/0` and relays it to a server:

```text
R1(config)# interface GigabitEthernet0/0/0
R1(config-if)# ip address 192.0.2.1 255.255.255.0
R1(config-if)# ip helper-address 198.51.100.10
```

The relayed request normally has:

```text
giaddr:     192.0.2.1
Option 82:  Present if inserted by an upstream switch or by R1
```

**`ip helper-address` enables DHCP relay forwarding. It does not, by itself, mean that the relay agent will insert Option 82.**

---

## DHCP Message Processing

For a normal DHCP address assignment:

1. The client sends DHCPDISCOVER without Option 82.
2. A switch or relay agent inserts Option 82 in the request.
3. The server receives Option 82 and can use it for address allocation or policy selection.
4. The server returns the DHCPOFFER, including the same Option 82 information.
5. The device responsible for the Option 82 insertion processes the reply and removes its Option 82 information before delivering the response to the client.
6. The same general processing applies to DHCPREQUEST and DHCPACK.

The DHCP server must **echo the relay-agent information** in its replies for the inserting device to validate and remove that information correctly.

---

## Option 82 on a Layer 2 Switch (DHCP Snooping)

On a switch, Option 82 insertion is controlled by the **DHCP snooping** feature, not by the DHCP relay configuration commands.

### Enable DHCP Snooping and Option 82

For clients in VLAN 10:

```text
SW1(config)# ip dhcp snooping
SW1(config)# ip dhcp snooping vlan 10
SW1(config)# ip dhcp snooping information option
!
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
!
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# switchport mode trunk
SW1(config-if)# ip dhcp snooping trust
```

- `ip dhcp snooping` enables the DHCP snooping feature globally.
- `ip dhcp snooping vlan 10` enables snooping for VLAN 10.
- `ip dhcp snooping information option` enables Option 82 insertion and removal for snooped DHCP traffic.
- `ip dhcp snooping trust` trusts DHCP server messages **arriving on the uplink**.

**Option 82 insertion is enabled by default when DHCP snooping is operating.** The explicit insertion command is included to make the configuration clear.

### Disable Option 82 Insertion

```text
SW1(config)# no ip dhcp snooping information option
```

This disables the snooping feature's Option 82 insertion and removal. **DHCP snooping itself remains enabled**, so untrusted-port filtering and dynamic binding learning can continue.

### Default Circuit ID

By default, the Circuit ID identifies the ingress VLAN and port using the switch's **`vlan-mod-port`** format. The module component corresponds to the switch/stack member.

For a client on VLAN 10, `Gi1/0/3`, the server can identify the client-facing switch attachment from this suboption.

The exact value is encoded in the Option 82 field. It is not necessarily the text `10-Gi1/0/3`.

### Default Remote ID

The default Remote ID is the switch's **MAC address**.

You can instead use the device hostname:

```text
SW1(config)# ip dhcp snooping information option format remote-id hostname
```

Or configure a string:

```text
SW1(config)# ip dhcp snooping information option format remote-id string ACCESS-SW1
```

To restore the default MAC-address-based Remote ID:

```text
SW1(config)# no ip dhcp snooping information option format remote-id string ACCESS-SW1
```

For the custom string, use **1 to 63 ASCII characters without spaces**.

### Custom Circuit ID

You can configure a **custom ASCII string** as the Circuit ID for a specific interface and VLAN.

```text
SW1(config)# interface GigabitEthernet1/0/3
SW1(config-if)# ip dhcp snooping vlan 10 information option format-type circuit-id string FLOOR1-PORT3
```

This uses the custom string instead of the default VLAN/module/port identifier, while retaining Cisco's structured Circuit ID encoding.

To **replace the default Circuit ID format entirely**, use `override`:

```text
SW1(config-if)# ip dhcp snooping vlan 10 information option format-type circuit-id override string FLOOR1-PORT3
```

With `override`, the specified ASCII string is used directly as the Circuit ID value, without the additional Cisco TLV structure.

Both commands are configured in **interface configuration mode** and apply to the specified VLAN.

---

## Switch DHCP Snooping Trust vs. Option 82 Trust

**The two settings below are not equivalent:**

| Command | Configured on | Meaning |
|---|---|---|
| `ip dhcp snooping trust` | Switch interface | Allows DHCP server messages to enter the port. Does not learn normal client bindings from that trusted ingress. |
| `ip dhcp snooping information option allow-untrusted` | Switch global configuration | Permits already-tagged DHCP client requests with Option 82 to enter **untrusted** ports. |

### Default Behavior: Untrusted Port Receives Option 82

By default, a switch with DHCP snooping enabled **drops client DHCP packets that already contain Option 82 when they arrive on an untrusted port**.

This prevents an untrusted endpoint from supplying its own Circuit ID or Remote ID and pretending to be connected somewhere else.

### Allow Existing Option 82 on Untrusted Ports

On an aggregation switch that receives DHCP requests from a downstream access switch:

```text
SW2(config)# ip dhcp snooping information option allow-untrusted
```

SW2 can now accept DHCP client requests that already contain Option 82 through an untrusted interface. It can also **learn bindings for clients behind that untrusted port**.

### Two-Switch Example: Preserve DHCP Snooping Bindings on SW2

```text
                    DHCP Server / Relay
                              |
                           Gi1/0/24
                             SW2
                           Gi1/0/1
                              |
                           Gi1/0/24
                             SW1
                           Gi1/0/1
                              |
                            Client
                            VLAN 10
```

SW1 inserts Option 82. SW2 must inspect the DHCP traffic and **learn bindings for clients behind SW1**.

**SW1:**

```text
ip dhcp snooping
ip dhcp snooping vlan 10
!
interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 10
!
interface GigabitEthernet1/0/24
 switchport mode trunk
 ip dhcp snooping trust
```

**SW2:**

```text
ip dhcp snooping
ip dhcp snooping vlan 10
ip dhcp snooping information option allow-untrusted
!
interface GigabitEthernet1/0/1
 switchport mode trunk
 no ip dhcp snooping trust
!
interface GigabitEthernet1/0/24
 switchport mode trunk
 ip dhcp snooping trust
```

SW2 learns clients on its untrusted `Gi1/0/1` interface. SW1 learns clients on its untrusted `Gi1/0/1` interface.

**Why not simply trust SW2's downlink?** If SW2 trusts `Gi1/0/1`, it will allow Option 82 traffic, but it will **not build ordinary DHCP snooping bindings for those downstream clients** on that interface.

Use `allow-untrusted` only on an aggregation switch whose downstream connection is controlled and expected to carry legitimate Option 82 information. Because it is global, it also changes acceptance behavior on other untrusted snooping ports.

---

## Option 82 on a DHCP Relay Agent

A router or multilayer switch acting as the DHCP relay agent uses a separate family of commands beginning with **`ip dhcp relay information`**.

### Basic DHCP Relay Without Option 82

```text
R1(config)# interface GigabitEthernet0/0/0
R1(config-if)# ip address 192.0.2.1 255.255.255.0
R1(config-if)# ip helper-address 198.51.100.10
```

This configures DHCP relay forwarding from the client subnet to the server `198.51.100.10`.

Relay-agent Option 82 insertion is **disabled by default**. A normal relay can forward DHCP requests without adding the Circuit ID and Remote ID suboptions.

### Enable Relay-Agent Option 82 Globally

```text
R1(config)# ip dhcp relay information option
```

This enables the relay agent to insert Option 82 into forwarded DHCP client requests.

To disable it:

```text
R1(config)# no ip dhcp relay information option
```

### Enable Relay-Agent Option 82 on One Interface

```text
R1(config)# interface GigabitEthernet0/0/0
R1(config-if)# ip dhcp relay information option-insert
```

This enables Option 82 insertion for DHCP client messages relayed from that interface.

To explicitly disable insertion on one interface, **even when enabled globally**:

```text
R1(config-if)# ip dhcp relay information option-insert none
```

Interface configuration takes precedence over the global setting. A plain `no` form generally removes the interface override so it inherits the global setting; `none` explicitly disables the feature on that interface.

### Relay-Agent Insertion: Global vs. Interface

| Command | Mode | Effect |
|---|---|---|
| `ip dhcp relay information option` | Global | Enables insertion on relay interfaces that inherit global settings. |
| `no ip dhcp relay information option` | Global | Disables global insertion. |
| `ip dhcp relay information option-insert` | Interface | Explicitly enables insertion on this relay interface. |
| `ip dhcp relay information option-insert none` | Interface | Explicitly disables insertion here, overriding global insertion. |

A relay-agent interface must still be configured to forward DHCP broadcasts to the server, usually using `ip helper-address`.

---

## Relay-Agent Trust for Received Option 82

A DHCP relay agent can receive a client request in which a downstream access switch has **already inserted Option 82**, while `giaddr` is still `0.0.0.0`.

**By default, the relay drops this combination** because a client with `giaddr = 0.0.0.0` is not expected to supply relay-agent information on its own.

The relay agent must be configured to trust Option 82 from the downstream network device.

### Trust Option 82 on One Relay Interface

```text
R1(config)# interface GigabitEthernet0/0/0
R1(config-if)# ip dhcp relay information trusted
```

This marks the interface as a trusted source of existing Option 82 information.

### Trust Option 82 on All Relay Interfaces

```text
R1(config)# ip dhcp relay information trust-all
```

This accepts Option 82 from all relay interfaces. Use the **interface command** when only one downstream segment is expected to supply Option 82.

### Trust Option 82 on a Multilayer Switch SVI

If the **DHCP relay agent is a multilayer switch**, the same relay information commands can be applied to the client-facing SVI:

```text
SW2(config)# ip dhcp relay information option
SW2(config)# interface Vlan10
SW2(config-if)# ip address 192.0.2.1 255.255.255.0
SW2(config-if)# ip helper-address 198.51.100.10
SW2(config-if)# ip dhcp relay information trusted
SW2(config-if)# ip dhcp relay information policy-action keep
```

The SVI receives and relays DHCP traffic for VLAN 10. Trusting Option 82 on the SVI is **relay-agent trust**. It does not change the separate DHCP snooping trust state of the switch's Layer 2 ports.

### Verify Relay-Agent Trust

```text
R1# show ip dhcp relay information trusted-sources
```

### Important Distinction

| Device/function | Command | What it permits |
|---|---|---|
| Layer 2 switch, snooping | `ip dhcp snooping trust` | Ingress DHCP server messages. |
| Layer 2 switch, snooping | `ip dhcp snooping information option allow-untrusted` | Pre-existing Option 82 in requests on untrusted ports. |
| Router/multilayer switch, relay | `ip dhcp relay information trusted` | Pre-existing Option 82 from the specified ingress interface. |
| Router/multilayer switch, relay | `ip dhcp relay information trust-all` | Pre-existing Option 82 from all ingress relay interfaces. |

**Trusting an interface for DHCP snooping does not configure relay-agent trust, and configuring relay-agent trust does not automatically trust it for DHCP snooping.**

---

## Relay Agent Policy for Existing Option 82

A relay agent can receive a DHCP request that already contains Option 82. If it is configured to insert its own Option 82, it must decide what to do with the existing information.

The relay information **reforwarding policy** controls this behavior.

| Policy | Behavior |
|---|---|
| **`drop`** | Discard the DHCP request if it already contains Option 82. |
| **`keep`** | Preserve the existing Option 82 information rather than replacing it. |
| **`replace`** | Replace existing Option 82 information with the relay agent's own information. **Default policy.** |

Configure globally:

```text
R1(config)# ip dhcp relay information policy drop
R1(config)# ip dhcp relay information policy keep
R1(config)# ip dhcp relay information policy replace
```

Configure a different policy for an interface:

```text
R1(config)# interface GigabitEthernet0/0/0
R1(config-if)# ip dhcp relay information policy-action keep
```

The interface-specific `policy-action` takes precedence over the global policy.

### Default Behavior and Trust

**By default, the relay agent drops DHCP requests containing Option 82 with `giaddr = 0.0.0.0` if they arrive on an untrusted interface.** This occurs regardless of the configured reforwarding policy.

If the request passes this trust check, the relay agent applies its reforwarding policy. The default policy is **`replace`**.

**Trust and policy serve different purposes:**

- **Trust:** Determines whether the relay agent accepts DHCP requests containing Option 82 with `giaddr = 0.0.0.0`.
- **Policy:** Determines whether existing Option 82 information is dropped, preserved, or replaced.

For example, if an access switch inserts useful Circuit ID information and R1 should preserve it when forwarding DHCP requests to the server, configure:

```text
R1(config)# interface GigabitEthernet0/0/0
R1(config-if)# ip dhcp relay information trusted
R1(config-if)# ip dhcp relay information policy-action keep
```

This allows R1 to **accept the existing Option 82 information and preserve it** rather than replacing it.

### End-to-End Example: Access Switch Adds Option 82, Router Relays It

```text
Client --- SW1 (Layer 2) --- R1 (DHCP Relay) --- DHCP Server
VLAN 10                    192.0.2.1             198.51.100.10
```

**SW1:**

```text
ip dhcp snooping
ip dhcp snooping vlan 10
ip dhcp snooping information option
!
interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 10
!
interface GigabitEthernet1/0/24
 switchport mode access
 switchport access vlan 10
 ip dhcp snooping trust
```

**R1:**

```text
ip dhcp relay information option
!
interface GigabitEthernet0/0/0
 ip address 192.0.2.1 255.255.255.0
 ip helper-address 198.51.100.10
 ip dhcp relay information trusted
 ip dhcp relay information policy-action keep
```

Result:

1. SW1 adds its Circuit ID and Remote ID.
2. R1 accepts the already-present Option 82 because its client-facing interface is trusted for relay information.
3. R1 sets the appropriate `giaddr` and sends the request to the DHCP server while retaining SW1's Option 82.
4. The server can use **SW1's port information**, not just R1's interface, for its decision.

If `keep` is replaced with `replace`, R1 inserts its own relay information instead, and the server no longer receives SW1's original Circuit ID/Remote ID in that Option 82.

---

## Relay-Agent Reply Checking

A relay agent also validates Option 82 returned in the DHCP server's replies.

Reply checking is **enabled by default**. Invalid returned relay information causes the reply to be dropped.

Enable or reenable it globally:

```text
R1(config)# ip dhcp relay information check
```

Disable it globally:

```text
R1(config)# no ip dhcp relay information check
```

Interface-specific equivalents:

```text
R1(config-if)# ip dhcp relay information check-reply
R1(config-if)# ip dhcp relay information check-reply none
```

`check-reply none` explicitly disables checking on that interface, overriding the global setting. Normally, **leave reply validation enabled**.

---

## Advanced Relay-Agent Suboptions

### Subscriber ID

A relay can attach an additional identifier identifying the subscriber:

```text
R1(config)# ip dhcp relay information option
R1(config)# interface GigabitEthernet0/0/0
R1(config-if)# ip dhcp relay information option subscriber-id SUBSCRIBER-100
```

This adds the **Subscriber ID suboption (6)**. It is not the same as the Circuit ID or Remote ID. The configured identifier can be used by a DHCP server that understands subscriber-level policies.

### VPN Information

For VRF-aware DHCP relay, Cisco supports VPN-related suboptions:

```text
R1(config)# ip dhcp relay information option vpn
```

This is used with VRF-aware relay configurations to convey VPN context to the DHCP server. It does not create a VRF or enable DHCP relay by itself.

### Encapsulating an Existing Option 82

With multiple relay agents, a subsequent relay can encapsulate earlier relay information rather than discarding it or replacing it outright:

```text
R1(config)# ip dhcp relay information option
R1(config)# ip dhcp relay information option vpn
R1(config)# ip dhcp relay information policy encapsulate
```

For an interface override:

```text
R1(config-if)# ip dhcp relay information policy-action encapsulate
```

This preserves information from an earlier relay inside an encapsulated suboption while allowing the next relay to add its own information. It is used primarily in more complex service-provider or VPN-aware relay designs.

---

## `giaddr` Validation on a DHCP Snooping Switch

DHCP snooping also inspects the **`giaddr` field** in requests entering an untrusted switchport.

Ordinary directly connected DHCP clients should send requests with `giaddr = 0.0.0.0`. A nonzero `giaddr` can indicate that the request came from a relay agent rather than a normal client.

The operational status of this check is displayed by:

```text
SW1# show ip dhcp snooping
```

Look for:

```text
Verification of giaddr field is enabled
```

A related global command is:

```text
ip dhcp snooping verify no-relay-agent-address
```

Cisco's command reference describes this positive form as disabling the nonzero-`giaddr` check and its `no` form as reenabling it. However, `show running-config all` can display the positive form among default settings, so **verify the operational check with `show ip dhcp snooping`** rather than inferring its state from the configuration line alone.

This is a **switch snooping validation check**, not the same as the relay-agent command `ip dhcp relay information trusted`.

---

## Default Settings

| Setting | Default |
|---|---|
| DHCP snooping globally | **Disabled** |
| DHCP snooping on each VLAN | **Disabled** |
| Snooping interface trust | **Untrusted** |
| Snooping Option 82 insertion | **Enabled** when DHCP snooping is operating |
| Accept existing Option 82 on an untrusted snooping port | **Disabled** |
| Snooping Remote ID | **Switch MAC address** |
| Snooping Circuit ID | **VLAN/module/port** format |
| Relay-agent Option 82 insertion | **Disabled** |
| Relay-agent acceptance of existing Option 82 with zero `giaddr` | **Not trusted by default** |
| Relay-agent policy for existing Option 82 | **Replace** |
| Relay-agent reply-information checking | **Enabled** |

---

## Verification and Troubleshooting

### On the Switch

```text
SW1# show ip dhcp snooping
SW1# show ip dhcp snooping binding
SW1# show ip dhcp snooping statistics
SW1# show ip dhcp snooping statistics detail
SW1# show running-config | include dhcp snooping
```

In `show ip dhcp snooping`, check:

- Snooping is enabled globally **and** on the client's VLAN.
- **Insertion of option 82 is enabled/disabled** as intended.
- **Option 82 on untrusted port is allowed/not allowed** as intended.
- The correct interfaces are trusted for DHCP server messages.
- The Remote ID and Circuit ID format settings are as expected.
- The `giaddr` verification line shows the actual operational setting.

### On the DHCP Relay Agent

```text
R1# show running-config | include dhcp relay information
R1# show running-config interface GigabitEthernet0/0/0
R1# show ip dhcp relay information trusted-sources
R1# debug ip dhcp server packet
```

The debug can help reveal relay processing and rejected packets. Use it during a controlled troubleshooting session.

### Packet Capture

Inspect a DHCPDISCOVER at the following points:

| Capture location | Expected observation |
|---|---|
| Client to access switch | No Option 82, `giaddr = 0.0.0.0` |
| Access switch to relay | Option 82 added by switch; `giaddr` usually still zero |
| Relay to server | Option 82 present if preserved or inserted; `giaddr` set by relay |
| Reply reaching client | Option 82 removed before delivery |

In Wireshark, inspect **DHCP Option 82**, **Agent Circuit ID**, **Agent Remote ID**, and the BOOTP **Relay agent IP address (`giaddr`)** field.

### Troubleshooting Scenarios

| Symptom | Likely cause | Check/fix |
|---|---|---|
| DHCPDISCOVER is dropped at SW2 after SW1 adds Option 82 | SW2 receives an Option 82 packet on an untrusted port | Use `ip dhcp snooping information option allow-untrusted` on SW2 when appropriate, or change the intended trust design. |
| DHCPDISCOVER is dropped at the relay | Relay receives Option 82 with `giaddr = 0.0.0.0` from an untrusted source | Configure `ip dhcp relay information trusted` on the client-facing relay interface. |
| DHCP server sees R1's identifiers instead of SW1's | Relay is replacing existing Option 82 | Configure `ip dhcp relay information policy-action keep` on the client-facing relay interface. |
| DHCP server receives no Option 82 | Neither switch nor relay inserts it, or an intermediate hop removes it | Check snooping insertion and relay `option` / `option-insert` settings. |
| Clients receive offers, but bindings do not appear on SW2 | The SW2 downlink was trusted, so downstream hosts were not learned on an untrusted ingress | If SW2 must learn the clients, keep the downlink untrusted and use `allow-untrusted` as appropriate. |
| DHCP request arrives at server but reply is rejected | Missing/altered Option 82 on return or reply-validation failure | Inspect server echo and relay/snooping validation counters. |
| DHCP works with Option 82 disabled on SW1, fails when enabled | Upstream relay does not accept existing Option 82 with zero `giaddr` | Configure relay information trust, or deliberately disable insertion on SW1. |

---

## Command Summary

### Layer 2 Switch: DHCP Snooping

| Command | Purpose |
|---|---|
| `ip dhcp snooping` | Enable global DHCP snooping. |
| `ip dhcp snooping vlan 10` | Enable DHCP snooping for VLAN 10. |
| `ip dhcp snooping information option` | Enable snooping Option 82 insertion/removal. |
| `no ip dhcp snooping information option` | Disable snooping Option 82 insertion/removal. |
| `ip dhcp snooping information option allow-untrusted` | Accept existing Option 82 on untrusted ports. |
| `no ip dhcp snooping information option allow-untrusted` | Restore rejection of existing Option 82 on untrusted ports. |
| `ip dhcp snooping trust` | Trust an interface for incoming DHCP server messages. |
| `no ip dhcp snooping trust` | Set an interface to untrusted. |
| `ip dhcp snooping information option format remote-id hostname` | Use the hostname as Remote ID. |
| `ip dhcp snooping information option format remote-id string ID` | Use a custom Remote ID. |
| `ip dhcp snooping vlan 10 information option format-type circuit-id string ID` | Set a custom Circuit ID on an interface for VLAN 10. |

### DHCP Relay Agent

| Command | Purpose |
|---|---|
| `ip helper-address A.B.C.D` | Forward DHCP broadcasts to the specified server. |
| `ip dhcp relay information option` | Globally enable relay-agent Option 82 insertion. |
| `no ip dhcp relay information option` | Disable global relay-agent insertion. |
| `ip dhcp relay information option-insert` | Enable insertion on an interface. |
| `ip dhcp relay information option-insert none` | Explicitly disable insertion on an interface. |
| `ip dhcp relay information trusted` | Trust incoming Option 82 on one relay interface. |
| `ip dhcp relay information trust-all` | Trust incoming Option 82 on all relay interfaces. |
| `ip dhcp relay information policy keep` | Keep existing Option 82 (global). |
| `ip dhcp relay information policy drop` | Drop requests with existing Option 82 (global). |
| `ip dhcp relay information policy replace` | Replace existing Option 82 (global). |
| `ip dhcp relay information policy-action keep` | Set the keep policy on an interface. |
| `ip dhcp relay information check` | Enable reply Option 82 validation (global). |
| `ip dhcp relay information check-reply none` | Explicitly disable reply validation on an interface. |
| `ip dhcp relay information option subscriber-id ID` | Add the Subscriber ID suboption on an interface. |

---

## Key Points

- **Option 82** identifies a DHCP client's attachment point. The most important fields are **Circuit ID (1)** and **Remote ID (2)**.
- **`giaddr` is a separate DHCP header field**, not part of Option 82.
- A **Layer 2 switch** can insert Option 82 without being a DHCP relay agent or changing `giaddr`.
- Switch insertion uses **`ip dhcp snooping information option`**; relay insertion uses **`ip dhcp relay information option`** or **`option-insert`**.
- **Snooping insertion is enabled by default when DHCP snooping runs**; **relay-agent insertion is disabled by default**.
- **`ip dhcp snooping trust`**, **`allow-untrusted`**, and **`ip dhcp relay information trusted`** are **three different trust controls**.
- An aggregation switch that must learn downstream client bindings can use an **untrusted downlink with `allow-untrusted`**, rather than trusting the downlink for all DHCP messages.
- A relay receiving Option 82 from an access switch with `giaddr = 0.0.0.0` needs **relay-information trust**.
- Relay **`drop` / `keep` / `replace`** policies determine what happens when a forwarded DHCP request already contains Option 82; **replace is the default**.
- The DHCP server can use Option 82 for address assignment policies, and the relay/switch removes the relay information before it reaches the client.

---

## References

- [Cisco: Configuring DHCP, Catalyst 9300 IOS XE](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-13/configuration_guide/ip/b_1713_ip_9300_cg/configuring_dhcp.html)
- [Cisco: DHCP Options Support, Catalyst 9300 IOS XE](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-13/configuration_guide/ip/b_1713_ip_9300_cg/dhcp_options_support.html)
- [Cisco: Operate and Troubleshoot DHCP Snooping on Catalyst 9000](https://www.cisco.com/c/en/us/support/docs/ip/dynamic-host-configuration-protocol-dhcp-dhcpv6/217055-operate-and-troubleshoot-dhcp-snooping.html)
- [Cisco: Configuring the Cisco IOS XE DHCP Relay Agent](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_dhcp/configuration/xe-3s/dhcp-xe-3s-book/dhcp-relay-agent-xe.html)
- [Cisco: DHCP Snooping Interactions with GIADDR and Option 82 on CAT9000](https://www.cisco.com/c/en/us/support/docs/switches/catalyst-9600-series-switches/225929-dhcp-snooping-interactions-with-giaddr.html)
- [Cisco: IOS XE Catalyst 9300 Security Command Reference](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/26-x/command_reference/b_26x_9300_cr/security_commands.html)
- [RFC 3046: DHCP Relay Agent Information Option](https://www.rfc-editor.org/rfc/rfc3046)
