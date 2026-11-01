# 4.2.c (ii) DHCPv6 Guard

**DHCPv6 Guard** is an IPv6 First Hop Security (FHS) feature that prevents unauthorized **DHCPv6 servers and relay agents** from sending server-side DHCPv6 messages into a switched network. It inspects DHCPv6 traffic **as it enters a Layer 2 switch port** and blocks unauthorized messages before they reach clients.

The feature has two main functions:

1. **Server/relay protection:** Block DHCPv6 server messages received on interfaces that should connect only to clients.
2. **Server message validation:** Optionally restrict authorized DHCPv6 server/relay addresses, assigned IPv6 addresses and prefixes, and DHCPv6 server preference values.

DHCPv6 Guard uses **policy-based configuration** with `ipv6 dhcp guard policy` and `ipv6 dhcp guard attach-policy`. It is part of the IPv6 FHS architecture alongside [RA Guard](ra-guard.md), SISF device tracking, and [IPv6 Source Guard](source-guard.md). **It does not require an existing SISF binding-table entry** to block rogue DHCPv6 server messages.

---

## DHCPv6 and Rogue Server Attacks

IPv6 hosts can receive configuration through **Stateless Address Autoconfiguration (SLAAC)**, **stateful DHCPv6**, and **stateless DHCPv6**. DHCPv6 can supply IPv6 addresses, DNS servers, DNS search domains, and other options.

A rogue DHCPv6 server might:

- Assign IPv6 addresses that conflict with the intended addressing plan.
- Advertise malicious DNS servers or other configuration options.
- Compete with the legitimate DHCPv6 server, disrupting address assignment.
- Set a high **DHCPv6 server preference** to influence which DHCPv6 server clients select.

**DHCPv6 does not advertise the IPv6 default gateway.** Hosts learn their default routers through IPv6 Router Advertisements (RAs). DHCPv6 Guard therefore does not replace RA Guard.

### DHCPv6 Message Types

DHCPv6 uses **UDP port 546** for clients and **UDP port 547** for servers and relay agents. The important message types are:

| Message | Type | Typical sender | Purpose |
| --- | ---: | --- | --- |
| **Solicit** | 1 | Client | Discover available DHCPv6 servers. |
| **Advertise** | 2 | Server | Offer configuration and identify the server. |
| **Request** | 3 | Client | Request configuration from a selected server. |
| **Renew** | 5 | Client | Renew bindings with the original server. |
| **Rebind** | 6 | Client | Renew bindings without requiring the original server. |
| **Reply** | 7 | Server | Supply or confirm configuration and lease information. |
| **Release** | 8 | Client | Release assigned resources. |
| **Reconfigure** | 10 | Server | Ask a client to initiate a new DHCPv6 exchange. |
| **Information-request** | 11 | Client | Request configuration options without necessarily obtaining an address. |
| **Relay-forward** | 12 | Relay | Forward client-side messages toward servers. |
| **Relay-reply** | 13 | Relay | Carry a server's response back toward clients. |

The typical initial stateful DHCPv6 exchange is:

```text
Client                  SW1                 DHCPv6 Server
  |                      |                        |
  |------ Solicit ------>|----------------------->|
  |                      |                        |
  |<---------------------|<----- Advertise -------|
  |                      |                        |
  |------- Request ----->|----------------------->|
  |                      |                        |
  |<---------------------|<-------- Reply --------|
```

The initial exchange can also use **Rapid Commit**, in which a server replies directly to a Solicit when the feature is supported and negotiated. The fundamental DHCPv6 Guard rule is unchanged: **client-side messages are allowed, while server-side messages must enter through an authorized port**.

### DHCPv6 Guard Packet Processing

Cisco classifies DHCPv6 packets by the kind of message they carry:

- **Client-side messages:** Permitted by DHCPv6 Guard regardless of whether the receiving port has the `client` or `server` device role.
- **Server-side messages:** Blocked by the `client` role. With the `server` role, they can pass, subject to any additional policy checks.
- **Relay messages:** DHCPv6 Guard distinguishes client-to-server forwarding from server-to-client forwarding. **Relay-reply** traffic returning to clients must enter through an authorized server/relay-facing port.

DHCPv6 Guard is an **ingress feature**. Permitting a legitimate server's Reply on the server-facing port does not prevent the switch from forwarding it **out** a client-facing port.

---

## DHCPv6 Guard Device Roles

A DHCPv6 Guard policy supports two device roles:

| Role | Expected device | Server-side DHCPv6 messages received on the port |
| --- | --- | --- |
| `device-role client` | DHCPv6 client or untrusted endpoint | **Dropped** |
| `device-role server` | Authorized DHCPv6 server, relay, or uplink toward them | **Permitted**, subject to configured checks |

The default role **inside a DHCPv6 Guard policy** is `client`. A policy must be attached to an appropriate target before it enforces filtering. **DHCPv6 Guard is not automatically active on every switch interface merely because the default role is `client`.**

Unlike RA Guard, DHCPv6 Guard uses `client` and `server`, not `host` and `router`.

### Client Role

```text
SW1(config)# ipv6 dhcp guard policy CLIENTS
SW1(config-dhcp-guard)# device-role client
```

Apply this to ordinary host-facing ports. A client can send Solicit and Request messages, but a rogue server connected to the same port cannot send DHCPv6 Advertise or Reply messages through SW1.

### Server Role

```text
SW1(config)# ipv6 dhcp guard policy SERVERS
SW1(config-dhcp-guard)# device-role server
```

Apply this where legitimate DHCPv6 server or relay traffic **enters** the switch. The role is necessary to allow server-side messages when DHCPv6 Guard is enforced, but **does not identify or authenticate a particular DHCPv6 server**. Additional matching options can restrict the permitted messages.

---

## Basic DHCPv6 Guard Lab

Use one switch, one legitimate DHCPv6 server, one client, and one rogue server:

```text
                       R1
                Legitimate DHCPv6
                      Server
                       | Gi1
                       |
                    Gi1/0/24
                 +-----------+
                 |    SW1    |
                 |  VLAN 10  |
                 +-----------+
                   /       \
              Gi1/0/1   Gi1/0/2
                 |         |
                PC1        R2
               Client    Rogue DHCPv6
                           Server

             VLAN 10: 2001:DB8:10::/64
```

| SW1 interface | Connected device | Policy | Expected result |
| --- | --- | --- | --- |
| `Gi1/0/1` | PC1 | `CLIENTS` | Client requests allowed. |
| `Gi1/0/2` | R2, rogue server | `CLIENTS` | Server advertisements and replies dropped. |
| `Gi1/0/24` | R1, legitimate server | `SERVERS` | Server replies allowed. |

### Step 1: Configure R1 as a DHCPv6 Server

This example uses stateful DHCPv6, with R1 also providing Router Advertisements:

```text
R1(config)# ipv6 unicast-routing
R1(config)# ipv6 dhcp pool LAN10
R1(config-dhcpv6)# address prefix 2001:DB8:10::/64
R1(config-dhcpv6)# dns-server 2001:DB8:10::53
R1(config-dhcpv6)# domain-name example.test
!
R1(config)# interface GigabitEthernet1
R1(config-if)# ipv6 address FE80::1 link-local
R1(config-if)# ipv6 address 2001:DB8:10::1/64
R1(config-if)# ipv6 nd managed-config-flag
R1(config-if)# ipv6 nd prefix 2001:DB8:10::/64 no-autoconfig
R1(config-if)# ipv6 dhcp server LAN10
R1(config-if)# no shutdown
```

- The **M flag** in R1's RAs indicates that stateful DHCPv6 address configuration is available.
- The `no-autoconfig` prefix option suppresses SLAAC address formation from the advertised prefix. It does not prevent the prefix from being advertised as on-link.
- R1's RAs still provide default-router information. DHCPv6 does **not** provide the default gateway.

A host must support and use stateful DHCPv6 for this example. You can also test DHCPv6 Guard with stateless DHCPv6, because rogue servers can supply unauthorized DNS information even without assigning addresses.

### Step 2: Configure SW1

```text
SW1(config)# vlan 10
!
SW1(config)# ipv6 dhcp guard policy CLIENTS
SW1(config-dhcp-guard)# device-role client
SW1(config-dhcp-guard)# exit
!
SW1(config)# ipv6 dhcp guard policy SERVERS
SW1(config-dhcp-guard)# device-role server
SW1(config-dhcp-guard)# exit
!
SW1(config)# interface range GigabitEthernet1/0/1-2
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 10
SW1(config-if-range)# ipv6 dhcp guard attach-policy CLIENTS
!
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# ipv6 dhcp guard attach-policy SERVERS
```

**No global DHCPv6 Guard enable command, IPv6 ACL, or prefix list is required for this basic protection.** The policies enforce their roles once attached.

### Step 3: Observe the Results

1. PC1 sends a DHCPv6 Solicit through `Gi1/0/1`. The `CLIENTS` policy allows it.
2. R1 sends a DHCPv6 Advertise, entering SW1 on `Gi1/0/24`. The `SERVERS` policy allows it.
3. R2 attempts to send an Advertise through `Gi1/0/2`. The `CLIENTS` policy blocks it.
4. PC1 sends a Request and receives R1's Reply, completing the assignment.

| Packet entering SW1 | Ingress policy | Result |
| --- | --- | --- |
| PC1 Solicit on `Gi1/0/1` | `client` | **Permit** |
| PC1 Request on `Gi1/0/1` | `client` | **Permit** |
| R2 Advertise on `Gi1/0/2` | `client` | **Drop** |
| R2 Reply on `Gi1/0/2` | `client` | **Drop** |
| R1 Advertise on `Gi1/0/24` | `server` | **Permit**, absent further restrictions |
| R1 Reply on `Gi1/0/24` | `server` | **Permit**, absent further restrictions |

**R2 can still send DHCPv6 client messages.** The client role blocks unauthorized *server-side* messages, not all DHCPv6 traffic from a port.

---

## DHCPv6 Guard Policy Options

A `server` policy can optionally examine DHCPv6 server/relay addresses, returned addresses or prefixes, and the server's advertised preference.

| Command | Purpose |
| --- | --- |
| `device-role client` | Block ingress server-side DHCPv6 messages. |
| `device-role server` | Permit ingress server-side messages, subject to additional checks. |
| `match server access-list NAME` | Restrict authorized DHCPv6 server/relay IPv6 addresses. |
| `match reply prefix-list NAME` | Validate IPv6 addresses/prefixes advertised by the server against a prefix list. |
| `preference min N` | Reject DHCPv6 Advertise messages with a preference below N. |
| `preference max N` | Reject DHCPv6 Advertise messages with a preference above N. |
| `trusted-port` | Bypass DHCPv6 Guard policing. |

Additional checks are not enforced unless configured. The `server` role alone permits server messages without restricting their source or advertised contents.

### Server and Relay Address Filtering

Use `match server access-list` to restrict the DHCPv6 server or relay addresses authorized to send inspected messages.

For example, R1 uses link-local address `FE80::1`:

```text
SW1(config)# ipv6 access-list ALLOWED_DHCPV6_SERVERS
SW1(config-ipv6-acl)# permit ipv6 host FE80::1 any
SW1(config-ipv6-acl)# exit
!
SW1(config)# ipv6 dhcp guard policy SERVERS
SW1(config-dhcp-guard)# device-role server
SW1(config-dhcp-guard)# match server access-list ALLOWED_DHCPV6_SERVERS
```

This authorizes DHCPv6 server messages from the specified IPv6 address on interfaces using the policy. **The ACL checks IPv6 addresses**, not the DHCPv6 Server Identifier's DUID. The ACL is referenced by DHCPv6 Guard; it is **not** applied to the interface as an ordinary IPv6 traffic filter.

A DHCPv6 server or relay may use a **link-local source address** on the client-facing link, so an ACL permitting only the server's global unicast address can incorrectly reject legitimate messages.

The **destination** of the ACL entry should be `any`. If the matching command is omitted, the address check is skipped. Cisco also documents that an **empty referenced ACL is treated as permit-all** for this feature, so define the intended entries explicitly.

### Reply Prefix Filtering

Use `match reply prefix-list` to restrict the IPv6 addresses or prefixes that an authorized server may assign in DHCPv6 messages:

```text
SW1(config)# ipv6 prefix-list ALLOWED_DHCPV6_PREFIXES seq 5 permit 2001:DB8:10::/64 le 128
!
SW1(config)# ipv6 dhcp guard policy SERVERS
SW1(config-dhcp-guard)# device-role server
SW1(config-dhcp-guard)# match reply prefix-list ALLOWED_DHCPV6_PREFIXES
```

The prefix-list entry permits the `2001:DB8:10::/64` prefix and its more-specific prefixes**, including individual `/128` host addresses within that subnet.

**Why `le 128` matters:**

```text
permit 2001:DB8:10::/64
```

matches the exact `/64` prefix, while:

```text
permit 2001:DB8:10::/64 le 128
```

also matches more-specific routes/addresses. For DHCPv6 address assignments, that distinction is important because an assigned host address can be evaluated as a `/128`.

For example:

| DHCPv6 assignment | Prefix-list result |
| --- | --- |
| `2001:DB8:10::100` | **Permitted** |
| `2001:DB8:10::200` | **Permitted** |
| `2001:DB8:99::100` | **Denied** |

If the reply contains an unauthorized assignment that fails the configured validation, **DHCPv6 Guard drops the offending DHCPv6 message rather than rewriting it to remove the unauthorized address**.

This matches addresses and prefixes **inside DHCPv6 server messages**, not the server's own IPv6 source address. Combine it with `match server access-list` when both restrictions are required.

Cisco documents that an **empty referenced prefix list is treated as permit-all** for DHCPv6 Guard. The check is also bypassed when the command is not configured.

### DHCPv6 Server Preference

DHCPv6 servers can include a **Preference option (option 7)** in Advertise messages. The value ranges from **0 to 255**, and **higher values are preferred** by DHCPv6 clients during server selection.

- **0:** Lowest preference; an Advertise with no Preference option is treated as preference 0 by the client.
- **255:** Highest preference; a valid Advertise with this value can cause the client to proceed immediately with the selected server rather than waiting for more offers.

DHCPv6 Guard can limit the preference values accepted from a server:

```text
SW1(config)# ipv6 dhcp guard policy SERVERS
SW1(config-dhcp-guard)# device-role server
SW1(config-dhcp-guard)# preference min 50
SW1(config-dhcp-guard)# preference max 200
```

The accepted preference range is **50 through 200, inclusive**:

| Advertised preference | Result |
| ---: | --- |
| 0 | **Drop** |
| 49 | **Drop** |
| 50 | **Permit** |
| 150 | **Permit** |
| 200 | **Permit** |
| 201 | **Drop** |
| 255 | **Drop** |

Cisco's command reference describes the bounds as rejecting values **less than** the minimum or **greater than** the maximum.

**This filters the DHCPv6 Preference option in Advertise messages.** It does not check the *router preference* in IPv6 Router Advertisements, which is a separate RA Guard option. It also does not set the preference advertised by R1; it only decides which received advertisements are acceptable.

With no preference limits configured, this check is skipped.

### Trusted Port

The `trusted-port` command makes interfaces using the policy **exempt from DHCPv6 Guard policing**:

```text
SW1(config)# ipv6 dhcp guard policy TRUSTED
SW1(config-dhcp-guard)# trusted-port
!
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# ipv6 dhcp guard attach-policy TRUSTED
```

All DHCPv6 Guard validation is bypassed on the trusted interface. This includes server-role blocking, source-address matching, reply-prefix filtering, and preference filtering.

`device-role server` and `trusted-port` are different:**

- `device-role server` **allows server messages but can still validate them**.
- `trusted-port` **bypasses DHCPv6 Guard checks entirely**.

A practical use for `trusted-port` is to exempt a fully trusted infrastructure interface from a broader restrictive policy. If you want to enforce a server-address ACL, reply prefix list, or preference restriction, **do not use `trusted-port`.

---

## Applying DHCPv6 Guard to Interfaces and VLANs

Policies can be attached to switch ports or to VLANs. Keep the distinction between **where the policy is attached** and **where a message enters** in mind.

### Default Client Policy

A policy name is optional when attaching DHCPv6 Guard:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# ipv6 dhcp guard
```

This applies the **default DHCPv6 Guard policy**, which uses `device-role client` with no `trusted-port`. It blocks unauthorized server-side messages received on that port.

A named policy is clearer for complete configurations:

```text
SW1(config-if)# ipv6 dhcp guard attach-policy CLIENTS
```

### Trunk Interfaces

On a trunk, a policy can apply to all VLANs on the interface, or only selected VLANs:

```text
SW1(config)# interface GigabitEthernet1/0/48
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,20
SW1(config-if)# ipv6 dhcp guard attach-policy CLIENTS vlan 10,20
```

This blocks unauthorized server messages **received on the trunk in VLANs 10 and 20**. If the trunk instead carries legitimate responses from a DHCPv6 server or relay on the other side, attach an appropriate `server` policy for those VLANs.

If you omit the VLAN selection, the attached policy applies to **all VLANs carried on that interface**.

### VLAN-Wide Attachment

You can also apply a DHCPv6 Guard policy to VLAN 10 across the switch:

```text
SW1(config)# vlan configuration 10
SW1(config-vlan-config)# ipv6 dhcp guard attach-policy CLIENTS
```

This provides VLAN-wide client protection, but the interface through which **legitimate DHCPv6 server or relay messages enter** still needs an appropriate **server-facing exception**.

For example:

```text
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# ipv6 dhcp guard attach-policy SERVERS
```

**An interface policy is the usual way to make a port-specific exception to a VLAN policy.** Verify the effective attachment and settings using `show ipv6 dhcp guard policy` when policies overlap.

### Inter-Switch Links

Consider two switches extending VLAN 10, with the DHCPv6 server attached to SW1 and a client attached to SW2:

```text
       R1 (DHCPv6 Server)
                |
               SW1 -------- SW2
                |            |
               PC1          PC2
                     VLAN 10
```

- The port **receiving R1's messages on SW1** uses a `server` policy.
- The **SW2 inter-switch port** receives legitimate server messages coming from SW1, so it needs a `server` policy for VLAN 10.
- PC1 and PC2 access ports use `client` policies.
- If SW2 has no legitimate DHCPv6 servers behind it, **SW1's inter-switch ingress port** can use a `client` policy to stop rogue messages coming from SW2.

**Do not automatically mark both ends of every trunk as `server`.** The role depends on the direction from which legitimate server/relay messages can enter.

---

## DHCPv6 Relay Agents

When the DHCPv6 server is on another IPv6 network, a **DHCPv6 relay agent** can forward messages between the client VLAN and the server.

```text
                DHCPv6 Server
                      |
                 Routed network
                      |
                  R1 (Relay)
                      |
                   Gi1/0/24
                      |
                     SW1
                      |
                   Gi1/0/1
                      |
                     PC1
```

The client initially sends Solicit messages toward the all-DHCP-servers-and-relays multicast destination `FF02::1:2` (UDP 547). The relay forwards requests toward the configured DHCPv6 server. The server's responses return through the relay, which delivers the response on the client-facing link.

**From SW1's perspective, R1 is the authorized source of server-side DHCPv6 messages** on the local client VLAN. Therefore, `Gi1/0/24` should use a `server` policy even though R1 is a **relay**, not the actual DHCPv6 server.

```text
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# ipv6 dhcp guard attach-policy SERVERS
```

On a separate Layer 2 segment carrying DHCPv6 **Relay-reply** messages from another relay, apply the same principle: authorize the ingress path carrying legitimate server-to-client responses.

**DHCPv6 Guard does not configure the relay agent.** The DHCPv6 relay is configured separately on the router. DHCPv6 Guard controls which messages are allowed to traverse the protected switch.

---

## DHCPv6 Guard and SISF Device Tracking

DHCPv6 Guard and SISF device tracking have related but different functions:

| Feature | Function |
| --- | --- |
| **DHCPv6 Guard** | Reject server-side DHCPv6 messages from unauthorized ingress ports, and optionally validate approved server messages. |
| **SISF device tracking** | Learn and maintain host IPv6 address, MAC, VLAN, and interface information from supported protocols. |
| **SISF binding table** | Store host address bindings used by tracking and applicable security features. |
| **IPv6 Source Guard** | Validate source IPv6 addresses in ordinary data traffic against authorized bindings. |

DHCPv6 Guard can help prevent **forged DHCPv6 server assignments from being accepted as legitimate host information**, but **configuring DHCPv6 Guard alone is not the same as configuring SISF device tracking or IPv6 Source Guard**.

For example, a host may use SLAAC rather than DHCPv6. DHCPv6 Guard does not prevent that host from using an unauthorized source IPv6 address in ordinary data packets. That is a separate security problem addressed through device tracking, binding validation, and Source Guard.

Also, DHCPv6 Guard does **not** inspect Router Advertisements or enforce the RA M/O flags. Those functions belong to [RA Guard](ra-guard.md) and router configuration.

---

## Verification and Troubleshooting

### Verify Policies and Attachments

Check the configured DHCPv6 Guard policies and where they are applied:

```text
SW1# show ipv6 dhcp guard policy
SW1# show ipv6 dhcp guard policy CLIENTS
SW1# show ipv6 dhcp guard policy SERVERS
SW1# show running-config interface GigabitEthernet1/0/1
SW1# show running-config interface GigabitEthernet1/0/24
SW1# show running-config | section ipv6 dhcp guard
```

The `show ipv6 dhcp guard policy` output identifies the configured role, policy checks, and policy attachment targets. Confirm both the **policy content** and the **actual ingress interface/VLAN**.

To inspect the related DHCPv6 server state on R1:

```text
R1# show ipv6 dhcp interface
R1# show ipv6 dhcp binding
```

These commands help confirm that R1 is operating as a DHCPv6 server and that client leases have been assigned.

### Debug DHCPv6 Guard

Cisco documents:

```text
SW1# debug ipv6 snooping dhcp-guard
```

Debug messages can help identify packets inspected and rejected by DHCPv6 Guard. Disable debugging after troubleshooting:

```text
SW1# undebug all
```

For packet-level inspection, capture **UDP 546 and UDP 547** and compare DHCPv6 exchanges entering the legitimate and rogue server-facing switch ports.

### Common Problems

| Symptom | Likely cause | What to check |
| --- | --- | --- |
| PC1 receives no DHCPv6 Advertise | Legitimate server-facing port has `device-role client`. | Check the policy on the server's **ingress** interface. |
| PC1 receives replies from R2 | R2's ingress port is unprotected or has `device-role server` / `trusted-port`. | Verify the client-facing policy attachment. |
| Legitimate Advertise messages are rejected | Source ACL denies R1 or the preference is outside the allowed range. | Compare actual source address and Preference option with the policy. |
| Solicit reaches the server, but Reply is rejected | Reply prefix list does not permit the assigned address or prefix. | Inspect the address/prefix in the Reply and prefix-list range. |
| Traffic works on one VLAN but fails on another | Policy attached to the wrong trunk VLAN selection. | Verify `vlan` scope and allowed VLANs. |
| DHCPv6 works, but there is no IPv6 default route | Missing or blocked RAs. | Check R1's RAs and RA Guard, **not** just DHCPv6 Guard. |
| DHCPv6 works, but SLAAC clients are not covered by address validation | DHCPv6 Guard is not a source-address binding enforcement feature. | Check SISF device tracking and IPv6 Source Guard. |
| DHCPv6 Guard policy is defined but does nothing | Policy was never attached. | Check interface or VLAN attachment. |

### Troubleshooting Sequence

1. Verify PC1 can receive RAs and is configured to request DHCPv6 information. For stateful DHCPv6, inspect the RA **M flag**.
2. Verify PC1 sends a **Solicit** or **Information-request** as appropriate.
3. Confirm the message reaches R1 or its DHCPv6 relay.
4. Check that the legitimate response arrives on an interface with a `server` policy**.
5. If it is dropped, examine the **server-address ACL**, **reply prefix list**, and **preference bounds**.
6. Verify the server's Reply reaches PC1 and that PC1 installs the intended address or options.
7. Independently verify that messages from R2 are dropped at **R2's ingress port**.

Do not confuse a failed IPv6 default route with a failed DHCPv6 exchange: **router discovery and DHCPv6 configuration are separate processes**.

---

## Key Points

- **DHCPv6 Guard** blocks unauthorized DHCPv6 server and relay replies at **ingress** on a Layer 2 switch.
- `device-role client` is the default policy role and drops incoming server-side DHCPv6 messages; ordinary client messages are allowed.
- `device-role server` permits server-side messages, subject to optional validation.
- `match server access-list` restricts authorized server/relay IPv6 addresses.
- `match reply prefix-list` restricts the IPv6 addresses or prefixes assigned in DHCPv6 server messages. Use `le 128` when permitting host addresses within a `/64`.
- `preference min` and `preference max` restrict the DHCPv6 **Advertise Preference option**, ranging from 0 through 255. Higher values are preferred by clients.
- `trusted-port` bypasses DHCPv6 Guard policing rather than simply permitting messages with additional checks.
- **Policies must be attached** to an interface or VLAN to enforce filtering.
- **The correct role depends on the direction of server-side message ingress**, including on inter-switch links and relay-facing ports.
- DHCPv6 Guard is distinct from **RA Guard**, **SISF device tracking**, and **IPv6 Source Guard**.

---

## References

- [Cisco: DHCPv6 Guard, IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_ip6-dhcpv6-guard.html)
- [Cisco: Configuring IPv6 First Hop Security, Catalyst 9600 IOS XE 26.x](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9600/software/release/26-x/configuration_guide/sec/b_26x_sec_9600_cg/configuring_ipv6_first_hop_security.html)
- [Cisco: IPv6 DHCPv6 Guard Command Reference (preference, match server, match reply)](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr/command/ipaddr-cr-book/ipaddr-l1.html)
- [Cisco: IPv6 DHCP Guard Attachment Command Reference](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipv6/command/ipv6-cr-book/ipv6-i1.html)
- [RFC 9915: Dynamic Host Configuration Protocol for IPv6 (DHCPv6)](https://www.rfc-editor.org/rfc/rfc9915.html)
