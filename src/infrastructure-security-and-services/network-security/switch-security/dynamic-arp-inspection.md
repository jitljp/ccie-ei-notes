# 4.2.a (v) Dynamic ARP Inspection

**Dynamic ARP Inspection (DAI)** is a Layer 2 security feature that prevents **ARP spoofing (ARP poisoning)** by checking ARP packets received on **untrusted switch ports**.

DAI checks whether an ARP packet's claimed **sender IPv4 address and MAC address** form a legitimate binding. It normally uses the **DHCP Snooping binding database**, but it can also use a manually configured **ARP ACL**. Packets that fail validation are dropped.

DAI protects **ARP requests and ARP replies**. It does not inspect ordinary IPv4 data packets; that is the role of [IP Source Guard](ip-source-guard.md).

---

## ARP Spoofing and ARP Poisoning

ARP resolves an IPv4 address to a MAC address on the local network. An ARP message contains fields including:

- **Sender hardware address (SHA):** The sender's MAC address.
- **Sender protocol address (SPA):** The sender's IPv4 address.
- **Target hardware address (THA):** The target's MAC address, if known.
- **Target protocol address (TPA):** The IPv4 address being resolved or contacted.

For example, a host at `192.0.2.10` might broadcast an ARP request asking for the MAC address of its default gateway, `192.0.2.1`.

ARP does not inherently authenticate the information in its messages. An attacker can send a forged ARP reply claiming that the gateway's IP address belongs to the attacker's MAC address.

```text
                  Legitimate gateway
              192.0.2.1 / 0000.0000.0001
                          |
                        Gi1/0/24
                       +--------+
                       |  SW1   |
                       +--------+
                      /          \
               Gi1/0/1          Gi1/0/2
                  |                 |
                 H1              Attacker
          192.0.2.10          192.0.2.99
          0011.2233.4455      00aa.bbcc.ddee
```

The attacker sends an ARP reply containing the false claim:

```text
Sender IP address:  192.0.2.1
Sender MAC address: 00aa.bbcc.ddee
```

If H1 accepts the message, it may update its ARP cache so that packets intended for `192.0.2.1` are sent to the attacker's MAC address. This can enable a **man-in-the-middle attack** or cause a denial of service.

With DAI enabled, SW1 inspects the forged ARP reply **as it enters Gi1/0/2**. Because the attacker does not have a valid binding for `192.0.2.1`, the switch drops the packet before forwarding it to H1.

**DAI also checks ARP requests.** A forged sender IP/MAC association in a request can be rejected even if the request is broadcast.

---

## How DAI Works

DAI is enabled **per VLAN** with `ip arp inspection vlan`. It has no separate global enable command on Catalyst switches.

For ARP traffic received in a DAI-enabled VLAN:

1. The switch identifies the **ingress interface** and its DAI trust state.
2. If the interface is **trusted**, the ARP packet bypasses DAI validation.
3. If the interface is **untrusted**, the switch checks the ARP packet against any applied **ARP ACL**.
4. If the ARP ACL does not make a final decision, the switch checks the **DHCP Snooping binding database**, provided ACL fallback is enabled.
5. The switch applies any enabled **additional validation checks**.
6. Valid ARP packets are forwarded; invalid packets are dropped and can be logged.

```text
             ARP received in DAI-enabled VLAN
                           |
                    Trusted ingress?
                      /           \
                    Yes            No
                     |              |
                  Forward      Applied ARP ACL?
                                  |
                     ACL permit / ACL deny / No match
                          |          |          |
                        Allow       Drop    DHCP binding
                                              lookup*
                                                 |
                                            Valid / Invalid
                                              |       |
                                            Allow    Drop

        * No DHCP fallback when the filter uses `static`.
        * Additional enabled validation checks must also pass.
```

DAI operates **on ingress only**. An ARP packet transmitted *out* an untrusted interface is not rechecked simply because that port is untrusted.

### DAI and DHCP Snooping Bindings

The DHCP Snooping binding database typically records:

| Field | Example |
|---|---|
| MAC address | `0011.2233.4455` |
| IPv4 address | `192.0.2.10` |
| VLAN | `10` |
| Interface | `Gi1/0/1` |
| Lease | Remaining DHCP lease time |

For example:

```text
SW1# show ip dhcp snooping binding
MacAddress          IpAddress       Lease(sec)  Type           VLAN  Interface
------------------  --------------  ----------  -------------  ----  ---------
00:11:22:33:44:55   192.0.2.10      68000       dhcp-snooping   10    Gi1/0/1
```

The binding authorizes the host using MAC `0011.2233.4455` to claim sender IP `192.0.2.10` on VLAN 10 through `Gi1/0/1`.

| ARP packet received on Gi1/0/1 | DAI decision |
|---|---|
| Sender IP `192.0.2.10`, sender MAC `0011.2233.4455` | **Permit** |
| Sender IP `192.0.2.99`, sender MAC `0011.2233.4455` | **Drop**: incorrect IP |
| Sender IP `192.0.2.10`, sender MAC `00aa.bbcc.ddee` | **Drop**: incorrect MAC |
| Sender IP `192.0.2.10`, sender MAC `0011.2233.4455`, arriving on another untrusted port | **Drop**: incorrect ingress interface |

The binding check validates the **ARP sender**, not the ARP target. For example, an ARP request from the valid host asking *who has `192.0.2.1`?* does not require H1 to have a binding for `192.0.2.1`. The sender `192.0.2.10` is the address being validated.

A host using a **static IP address** has no dynamically learned DHCP Snooping entry. It therefore needs another means of authorization, such as an ARP ACL or an applicable static source binding.

See [DHCP Snooping](dhcp-snooping/overview.md) for binding creation, aging, and database persistence.

---

## Trusted and Untrusted Interfaces

DAI has two interface trust states:

| State | Incoming ARP traffic | Typical use |
|---|---|---|
| **Untrusted** | Subject to ARP ACL, binding, and enabled validation checks | Access ports connected to end hosts |
| **Trusted** | Bypasses DAI validation | Links carrying ARP from trusted infrastructure |

**All interfaces are untrusted by default.** This matters once DAI is enabled on a VLAN.

To configure a trusted port:

```text
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# ip arp inspection trust
```

To return it to untrusted:

```text
SW1(config-if)# no ip arp inspection trust
```

**DAI trust and DHCP Snooping trust are independent.** These commands control different traffic:

```text
SW1(config-if)# ip dhcp snooping trust
SW1(config-if)# ip arp inspection trust
```

For example, if the default gateway and DHCP server are upstream on `Gi1/0/24`, the port might require both commands. DHCP Snooping trust permits DHCP server messages; DAI trust permits incoming ARP from that upstream device without requiring a local DHCP binding for the gateway.

### DAI Across Multiple Switches

```text
       DHCP clients                       DHCP clients
             |                                 |
            SW1 ----------- trunk ----------- SW2
              DAI enabled                   DAI enabled
```

Each switch learns DHCP Snooping bindings for clients whose DHCP traffic enters its **untrusted** ports. Consequently, SW2 might not have local bindings for hosts attached to SW1.

If SW2 receives those hosts' ARP packets on an **untrusted trunk**, it may drop them because it has no matching local bindings. In a design where both switches inspect their own access ports, the interswitch trunk is commonly configured as **DAI trusted**.

A trusted trunk must lead to equipment that is itself trusted to enforce the security boundary. Otherwise, forged ARP messages could enter through the trusted link without inspection.

---

## Basic DAI Configuration

The following example enables DHCP Snooping and DAI for VLAN 10. H1 and H2 are DHCP clients, and the gateway/DHCP infrastructure is reached through `Gi1/0/24`.

```text
                    Gateway / DHCP infrastructure
                                |
                             Gi1/0/24
                                |
                              +-----+
                              | SW1 |
                              +-----+
                             /       \
                       Gi1/0/1     Gi1/0/2
                          |            |
                         H1           H2
                      DHCP client  DHCP client
```

### Step 1: Configure DHCP Snooping

```text
SW1(config)# ip dhcp snooping
SW1(config)# ip dhcp snooping vlan 10
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# ip dhcp snooping trust
```

The switch learns the clients' IP-to-MAC-to-port associations as their DHCP assignments complete.

### Step 2: Enable DAI for VLAN 10

```text
SW1(config)# ip arp inspection vlan 10
```

This single command enables DAI on VLAN 10. A VLAN range or list can also be specified:

```text
SW1(config)# ip arp inspection vlan 10,20,30-35
```

### Step 3: Configure DAI Trust on the Uplink

```text
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# ip arp inspection trust
```

The two host-facing access ports remain **untrusted**, which is the default.

### Step 4: Verify Bindings and DAI

```text
SW1# show ip dhcp snooping binding
SW1# show ip arp inspection
SW1# show ip arp inspection interfaces
SW1# show ip arp inspection statistics vlan 10
```

If DHCP Snooping is newly enabled on a network where clients already hold leases, the snooping database may initially be empty. The clients must renew their leases or otherwise complete DHCP exchanges that the switch can observe before DAI can validate their ARP messages through those bindings.

To disable DAI for a VLAN:

```text
SW1(config)# no ip arp inspection vlan 10
```

---

## ARP ACLs for Statically Addressed Hosts

A host with a manually assigned IP address does not create a dynamic DHCP Snooping binding. DAI can authorize that host using an **ARP ACL**, which matches the **sender IP and sender MAC** inside ARP messages.

Suppose VLAN 10 contains a server with:

```text
IP address:  192.0.2.50
MAC address: 0011.2233.4477
Port:        Gi1/0/3
```

Configure:

```text
SW1(config)# arp access-list STATIC_HOSTS
SW1(config-arp-nacl)# permit ip host 192.0.2.50 mac host 0011.2233.4477
SW1(config-arp-nacl)# exit
SW1(config)# ip arp inspection filter STATIC_HOSTS vlan 10
```

This permits ARP messages with the authorized sender IP/MAC pair while keeping the port **DAI untrusted**. The ARP ACL is applied **to the VLAN**, not to the individual switch interface.

The ACL is not active until `ip arp inspection filter` applies it to a VLAN. An ARP ACL is **not an IPv4 ACL or an Ethernet MAC ACL**: the `arp access-list` command creates a separate ACL type used for DAI.

### ARP ACL Processing and DHCP Snooping Precedence

**ARP ACLs take precedence over DHCP Snooping bindings.** For packets on untrusted ports, DAI uses the following rules:

| ARP ACL result | Filter uses `static`? | DAI decision |
|---|---|---|
| Explicit `permit` | Either | **Permit** based on ARP ACL, subject to additional validations |
| Explicit `deny` | Either | **Drop**, even if DHCP Snooping has a valid binding |
| No ACL match | **No** | Fall back to the DHCP Snooping binding database |
| No ACL match | **Yes** | **Drop** without checking DHCP Snooping |

The ARP ACL has an **implicit deny** at the end, but when the filter is configured **without `static`**, an unmatched packet can still be validated against DHCP Snooping. By contrast, an **explicit ACL deny** always takes precedence and drops the packet.

This distinction lets one VLAN support both **static-IP devices through the ARP ACL** and **DHCP clients through DHCP Snooping**.

### The `static` Keyword

Compare:

```text
SW1(config)# ip arp inspection filter STATIC_HOSTS vlan 10
```

**Without `static`:** ARP ACL permits/denies are honored; packets not matching an ACE can use DHCP Snooping bindings.

```text
SW1(config)# ip arp inspection filter STATIC_HOSTS vlan 10 static
```

**With `static`:** The ARP ACL becomes authoritative. Packets not permitted by the ACL are dropped, and **DHCP Snooping bindings are not used** for those packets.

**Do not use `static` for a mixed DHCP/static VLAN unless the ARP ACL also explicitly permits all required hosts.** Otherwise, dynamically addressed clients can lose ARP connectivity despite valid DHCP Snooping bindings.

### Static IP Source Bindings

Another option on supported Catalyst switches is a manually configured **IP source binding**:

```text
SW1(config)# ip source binding 0011.2233.4477 vlan 10 192.0.2.50 interface GigabitEthernet1/0/3
SW1# show ip source binding
```

This adds the host's IP/MAC/VLAN/interface association to the switch's source-binding information. Cisco documents this as another way to authorize a static host for DAI. ARP ACLs remain the most direct way to configure a DAI-specific static exception.

Do not confuse an `ip source binding` with an `arp access-list`: the former identifies a host on a specific switch port and is also used by [IP Source Guard](ip-source-guard.md); the latter defines sender IP/MAC matches for DAI at the VLAN level.

---

## Additional ARP Validation Checks

Besides its fundamental IP-to-MAC binding validation, DAI supports **three optional checks** with the global command:

```text
SW1(config)# ip arp inspection validate [src-mac] [dst-mac] [ip]
```

These additional checks are **disabled by default**. Disabling them does **not** disable normal DHCP-binding or ARP-ACL validation.

| Keyword | Validation | Checked on |
|---|---|---|
| `src-mac` | Ethernet **source MAC** must match the ARP **sender hardware address (SHA)** | ARP requests and replies |
| `dst-mac` | Ethernet **destination MAC** must match the ARP **target hardware address (THA)** | ARP replies |
| `ip` | Reject invalid or unexpected IP addresses in relevant ARP fields | Sender IP in requests and replies; target IP in replies |

### `src-mac` Validation

An incoming ARP packet might contain:

```text
Ethernet source MAC: 0011.2233.4455
ARP sender MAC:     00aa.bbcc.ddee
```

With `ip arp inspection validate src-mac`, DAI rejects the mismatch. This check compares **two fields within the received frame**; it is distinct from looking up the sender IP/MAC pair in the DHCP Snooping database.

### `dst-mac` Validation

For an **ARP reply**, DAI can compare:

```text
Ethernet destination MAC
ARP target hardware address (THA)
```

If they differ, the reply is dropped. This check does **not** apply to ordinary ARP requests, where the target hardware address is often unknown.

### `ip` Validation

DAI can reject ARP packets containing invalid or unexpected IPv4 addresses, including:

- `0.0.0.0`
- `255.255.255.255`
- IPv4 multicast addresses

The check applies to the **sender IP** in ARP requests and replies, and to the **target IP** in ARP replies. It does **not** perform the same target-IP check on ordinary ARP requests.

### Enabling Multiple Checks

To enable all three:

```text
SW1(config)# ip arp inspection validate src-mac dst-mac ip
```

**The command replaces the previous selection rather than adding to it.** For example:

```text
SW1(config)# ip arp inspection validate src-mac dst-mac
SW1(config)# ip arp inspection validate ip
```

After the second command, **only `ip` validation is enabled**. To keep all three, configure them together in one command.

---

## ARP Rate Limiting

DAI also rate-limits incoming ARP requests and replies to protect switch resources. The ARP rate limit is an **interface-level** setting, not a separate limit for each VLAN.

| Interface | Default ARP rate limit |
|---|---|
| **Untrusted** | **15 packets per second (pps)** |
| **Trusted** | **Unlimited** |
| **Default burst interval** | **1 second** |

If the received ARP rate **exceeds the limit**, the switch places the interface in the **error-disabled** state. This is different from simply dropping an individual invalid ARP packet.

### Configure a Rate Limit

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# ip arp inspection limit rate 30
```

This permits an ARP arrival rate of up to **30 pps** according to the monitoring interval.

You can also set the burst interval:

```text
SW1(config-if)# ip arp inspection limit rate 30 burst interval 2
```

This sets a **30 pps** limit with a **2-second monitoring interval**.

To remove the rate limit entirely:

```text
SW1(config-if)# ip arp inspection limit none
```

To return to the default rate limit for the interface's trust state:

```text
SW1(config-if)# no ip arp inspection limit
```

An explicitly configured rate limit is retained if you later change the interface trust state. Without an explicit rate setting, the interface uses the default for its current trust state.

### Recovering from a Rate-Limit Violation

An ARP rate-limit violation causes error-disablement. Without automatic recovery, correct the cause and reset the interface administratively:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# shutdown
SW1(config-if)# no shutdown
```

Alternatively, enable automatic recovery:

```text
SW1(config)# errdisable recovery cause arp-inspection
SW1(config)# errdisable recovery interval 300
```

The default **recovery interval is 300 seconds**, and automatic recovery is disabled until configured.

To inspect the interface status and error-disable recovery configuration:

```text
SW1# show interfaces status err-disabled
SW1# show errdisable recovery
SW1# show ip arp inspection interfaces
```

On trunk links, multiple hosts' ARP traffic contributes to the same port rate. For EtherChannels, the aggregate ARP rate of member links is relevant to the port-channel limit; a violation can error-disable the entire EtherChannel.

---

## DAI Logging

DAI can record rejected ARP packets and report why they failed validation.

Default logging behavior on Catalyst switches includes:

- Logging **denied or dropped ARP packets**.
- A DAI log buffer of **32 entries**.
- A limit of **5 system messages per second**, with a **1-second interval**.

To display the buffer:

```text
SW1# show ip arp inspection log
```

To clear it:

```text
SW1# clear ip arp inspection log
```

### Configure the Log Buffer

```text
SW1(config)# ip arp inspection log-buffer entries 128
SW1(config)# ip arp inspection log-buffer logs 10 interval 5
```

- `entries 128`: Allow up to **128 entries** in the DAI log buffer.
- `logs 10 interval 5`: Generate system messages at a controlled rate of **10 over a 5-second interval**.

### Per-VLAN Logging

The switch can separately control logging for ARP ACL matches and DHCP-binding decisions:

```text
SW1(config)# ip arp inspection vlan 10 logging dhcp-bindings all
SW1(config)# ip arp inspection vlan 10 logging acl-match none
```

- `dhcp-bindings all`: Log both permitted and denied packets decided through DHCP bindings.
- `acl-match none`: Do not log packets decided through the ARP ACL.

Logging every permitted ARP message can generate substantial log volume. The defaults focus on denied or dropped packets.

---

## Verification and Troubleshooting

### Useful Show Commands

| Command | Purpose |
|---|---|
| `show ip arp inspection` | DAI-enabled VLANs, validation settings, ACL associations, and counters |
| `show ip arp inspection vlan 10` | DAI status and configuration for VLAN 10 |
| `show ip arp inspection interfaces` | Port trust states and ARP rate limits |
| `show ip arp inspection statistics vlan 10` | Forward, drop, ACL, DHCP, and validation counters |
| `show ip arp inspection log` | Logged ARP validation failures |
| `show arp access-list` | ARP ACL entries |
| `show ip dhcp snooping binding` | Dynamically learned DHCP bindings |
| `show ip source binding` | Dynamic and static IP source bindings |
| `show interfaces status err-disabled` | Interfaces shut down by error-disable conditions |
| `show errdisable recovery` | Automatic recovery settings |

Reset the DAI statistics when preparing a test:

```text
SW1# clear ip arp inspection statistics
```

### Example: DAI Status

An illustrative `show ip arp inspection` output:

```text
SW1# show ip arp inspection
Source Mac Validation      : Enabled
Destination Mac Validation : Enabled
IP Address Validation      : Enabled

 Vlan  Configuration  Operation  ACL Match     Static ACL
 ----  -------------  ---------  ------------  ----------
   10  Enabled        Active     STATIC_HOSTS  No
```

Interpretation:

- DAI is **enabled and active** in VLAN 10.
- The three optional validation checks are enabled.
- The `STATIC_HOSTS` ARP ACL is applied.
- `Static ACL: No` means that the filter **does not use the `static` keyword**, so unmatched packets can fall back to DHCP Snooping bindings.

To inspect the packet counters, use:

```text
SW1# show ip arp inspection statistics vlan 10
```

The counters distinguish packets forwarded or dropped through **ARP ACL decisions**, **DHCP binding decisions**, and **additional validation failures**. This helps identify why DAI rejected a particular packet.

### Common Problems

| Symptom | Likely cause | What to check |
|---|---|---|
| DHCP client obtains an IP but cannot communicate | Missing or incorrect DHCP Snooping binding | `show ip dhcp snooping binding`; DHCP renewal |
| Static-IP host cannot communicate | No acceptable binding or ARP ACL entry | `show arp access-list`; `show ip source binding` |
| DHCP clients stop communicating after applying an ARP ACL | Filter uses `static` and ACL does not permit their ARP | `show ip arp inspection` and filter configuration |
| ARP from an upstream router is dropped | Ingress port untrusted and gateway lacks a binding | `show ip arp inspection interfaces`; upstream link trust |
| Port enters error-disabled state | ARP rate limit exceeded | `show interfaces status err-disabled`; ARP rate |
| ARP ACL permits a host but packets still drop | Additional `src-mac`, `dst-mac`, or `ip` validation fails | DAI validation configuration and statistics |
| DHCP Snooping works, but DAI doesn't inspect packets | DAI not enabled for the VLAN | `show ip arp inspection` |
| DAI works on one switch but blocks hosts on another | Untrusted interswitch link without matching local bindings | Trunk trust states and DAI logs |

### DAI vs. IP Source Guard

Both features use address-binding information but inspect different traffic:

| Feature | Traffic checked | Main validation |
|---|---|---|
| **DAI** | **ARP requests and replies** | ARP sender IP-to-MAC association |
| **IP Source Guard** | **IP packets** entering a protected Layer 2 port | Source IP, optionally source MAC |
| **DHCP Snooping** | **DHCP messages** | DHCP server/client message legitimacy and DHCP binding creation |

DAI does not replace IP Source Guard. A host might send an IPv4 packet with a forged **source IP address** without sending any ARP packet containing that forged address. Conversely, basic IP Source Guard does not itself inspect forged ARP replies.

---

## Key Points

- **DAI is enabled per VLAN** using `ip arp inspection vlan <vlan-list>`; there is no separate global DAI enable command.
- **All interfaces are DAI untrusted by default.** `ip arp inspection trust` bypasses ARP inspection on the specified ingress port.
- DAI validates **ARP requests and replies**, normally using **DHCP Snooping bindings**.
- **ARP ACLs** authorize or deny specific sender IP/MAC combinations and have precedence over DHCP bindings.
- With `ip arp inspection filter <name> vlan <vlans>` **without `static`**, unmatched ACL traffic can fall back to DHCP Snooping.
- Adding **`static`** means unmatched traffic is dropped **without DHCP Snooping fallback**.
- The optional `src-mac`, `dst-mac`, and `ip` checks are **disabled by default**. A new `ip arp inspection validate` command **replaces** the previous selection.
- **Untrusted interfaces default to 15 ARP packets per second**; trusted interfaces default to unlimited. Exceeding a rate limit **error-disables** the port.
- DAI trust and DHCP Snooping trust are **separate configurations**.
- Use **DAI for ARP validation** and **IP Source Guard for IPv4 source filtering**.

---

## References

- [Cisco: Catalyst 9300 Security Configuration Guide, Configuring Dynamic ARP Inspection](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-18/configuration_guide/sec/b_1718_sec_9300_cg/configuring_dynamic_arp_inspection.html)
- [Cisco: FHS and SISF Configuration Guide, Dynamic ARP Inspection](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/sec-crypto/fhs-sisf/fhs-and-sisf-configuration-guide/dynamic-arp-inspection.html)
- [Cisco: Troubleshoot Dynamic ARP Inspection and IP Source Guard in Catalyst Switches](https://www.cisco.com/c/en/us/support/docs/switches/lan-switch-software/222274-troubleshoot-dynamic-arp-inspection-dai.html)
