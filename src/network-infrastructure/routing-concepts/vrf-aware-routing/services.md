# VRF-Aware Services

VRF-aware services allow a router or Layer 3 switch to use a specific **VRF routing table** for locally generated management, monitoring, and control-plane traffic.

For example, an IOS XE router can reach its NTP, DNS, syslog, and TACACS+ servers through a dedicated `MGMT` VRF while forwarding ordinary user traffic through other VRFs.

**Configuring an interface in a VRF does not automatically make every service use that VRF.** Each service has its own VRF-aware configuration or operational syntax.

## Main Idea

Consider a router with a management VRF:

```text
                      R1
             +------------------+
             | Global table     |
             | VRF CUSTOMER_A   |
             | VRF CUSTOMER_B   |
             | VRF MGMT         |
             +---------+--------+
                       |
                  Ethernet0/0
                   VRF MGMT
                 192.0.2.1/24
                       |
               192.0.2.254
                MGMT Gateway
                       |
              Management Network
                       |
       +---------------+---------------+
       |               |               |
   NTP / DNS        Syslog         AAA / NMS
```

R1 needs to communicate with management servers through `MGMT` rather than through its global routing table.

There are two independent questions for most router-originated services:

1. **Which routing table is used to reach the destination?**
2. **Which local IP address is used as the packet's source?**

For example, configuring a service's source interface does not universally configure the VRF used for its route lookup. Some services require a separate VRF keyword or VRF-specific server group.

## Example Management VRF

The following examples assume this configuration already exists:

```text
R1(config)# vrf definition MGMT
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# exit-address-family
!
R1(config)# interface Ethernet0/0
R1(config-if)# vrf forwarding MGMT
R1(config-if)# ip address 192.0.2.1 255.255.255.0
R1(config-if)# no shutdown
!
R1(config)# ip route vrf MGMT 0.0.0.0 0.0.0.0 192.0.2.254
```

The management servers in the examples are reachable through `192.0.2.254`:

| Service | Destination |
|---|---|
| NTP | `203.0.113.10` |
| DNS | `203.0.113.20` |
| Syslog | `203.0.113.30` |
| TACACS+ | `203.0.113.40` |
| SNMP manager | `203.0.113.50` |
| NetFlow collector | `203.0.113.60` |
| IP SLA target | `203.0.113.70` |

> The addresses above are examples. They assume the management gateway and return routing are configured correctly.

Verify basic connectivity before configuring a management service:

```text
R1# show ip route vrf MGMT
R1# ping vrf MGMT 203.0.113.10 source Ethernet0/0
```

A successful VRF-aware ping confirms IP reachability, but does not prove that an individual service is using the correct VRF or that its server is accepting requests.

## VRF-Aware Ping and Traceroute

Specify the VRF when testing connectivity from the router.

```text
R1# ping vrf MGMT 203.0.113.10
R1# traceroute vrf MGMT 203.0.113.10
```

Specify a source interface when necessary:

```text
R1# ping vrf MGMT 203.0.113.10 source Ethernet0/0
```

Compare these commands:

```text
ping 203.0.113.10
ping vrf MGMT 203.0.113.10
```

- The first command performs the normal global routing lookup.
- The second uses the `MGMT` routing table.

The same destination can therefore be reachable in one context but not the other.

For IPv6, use the platform's VRF-aware IPv6 ping and traceroute syntax, for example:

```text
R1# ping vrf MGMT 2001:db8:100::10
```

The `MGMT` VRF must have an IPv6 address family, usable IPv6 interfaces, and an IPv6 route to the destination.

## VRF-Aware SSH and Telnet

### SSH Client

To initiate an SSH connection through a particular VRF, use the `-vrf` option:

```text
R1# ssh -vrf MGMT -l admin 203.0.113.80
```

This selects `MGMT` for the outgoing SSH connection.

On supported platforms, the SSH client can also use a configured source interface:

```text
R1(config)# ip ssh source-interface Ethernet0/0
```

The explicit `ssh -vrf` option makes the intended routing context clear and is preferable when initiating an occasional connection to a host in a particular VRF.

### SSH Server and VTY Access

A router can accept SSH sessions sent to an IP address on an interface in a VRF, subject to its SSH, AAA, VTY, ACL, and management-plane configuration.

For example:

```text
R1(config)# line vty 0 4
R1(config-line)# transport input ssh
```

If an incoming `access-class` is applied to the VTY lines, the **`vrf-also`** keyword may be necessary to permit connections arriving through VRF interfaces.

```text
R1(config)# access-list 10 permit 203.0.113.0 0.0.0.255
R1(config)# line vty 0 4
R1(config-line)# access-class 10 in vrf-also
```

Without `vrf-also`, the VTY access-class feature can reject otherwise permitted connections arriving through VRF interfaces. 

Newer IOS XE platforms also offer `access-class <acl> in vrfname <vrf>` to apply a filter to a specific VRF.

> **VRF-aware incoming management** and **VRF-aware outgoing SSH** are different functions. The `ssh -vrf` option applies to an SSH client session initiated by the router, not to its SSH server.

Telnet also has a VRF-aware client option on supported IOS XE platforms:

```text
R1# telnet 203.0.113.80 /vrf MGMT
```

Use SSH instead of Telnet for normal device administration.

## VRF-Aware NTP

A device can synchronize its clock with an NTP server reachable through a VRF.

Configure the VRF on the NTP server association:

```text
R1(config)# ntp server vrf MGMT 203.0.113.10
```

On supported IOS XE releases, specify a source interface for this particular server association:

```text
R1(config)# ntp server vrf MGMT 203.0.113.10 source Ethernet0/0
```

The `vrf MGMT` option selects the routing table used to reach the NTP server. The `source Ethernet0/0` option selects the source interface.

The global `ntp source` command is another way to select a source interface for NTP packets:

```text
R1(config)# ntp source Ethernet0/0
```

Verify synchronization:

```text
R1# show ntp associations
R1# show ntp status
```

An NTP association may exist without the clock being synchronized. Check reachability, server responses, authentication (if configured), and synchronization state.

## VRF-Aware DNS

A router can use different DNS servers and hostname information in different VRFs.

### VRF-Specific DNS Server

```text
R1(config)# ip name-server vrf MGMT 203.0.113.20
```

This associates the DNS server with `MGMT`.

DNS lookup can be enabled for the VRF:

```text
R1(config)# ip domain lookup vrf MGMT
```

DNS lookup is normally enabled by default; this command is useful when it was previously disabled.

### VRF-Specific Domain Name and Static Host Entries

```text
R1(config)# ip domain name vrf MGMT example.net
R1(config)# ip host vrf MGMT nms.example.net 203.0.113.50
```

The domain name is used for resolving unqualified names in `MGMT`. The `ip host vrf` command creates a hostname mapping associated with that VRF.

A router can maintain different hostname mappings for different routing contexts, including cases where names or private IP addresses overlap.

> The service that uses a hostname must also select the intended VRF. Configuring a VRF-specific DNS server alone does not force every router-originated application to use it.

## VRF-Aware Syslog

Remote syslog messages can be sent to a server through a particular VRF.

```text
R1(config)# logging host 203.0.113.30 vrf MGMT
R1(config)# logging trap informational
```

The `vrf MGMT` option selects the routing table used to reach the syslog server.

The logging level determines which severities are forwarded to remote logging destinations.

Verify the configuration and current logging status:

```text
R1# show running-config | include logging
R1# show logging
```

A syslog server can be unreachable because of missing VRF routing even when the router successfully generates local log messages.

Source-interface behavior for syslog depends on the IOS XE platform and release. Do not assume that a generic source-interface command automatically configures the syslog VRF.

## VRF-Aware SNMP

SNMP has two distinct VRF-related use cases:

- **Sending SNMP notifications** through a VRF to a monitoring system.
- **Allowing SNMP polling and exposing VRF-specific information**, which can also involve SNMP contexts and access controls.

### Sending Traps Through a VRF

A VRF-aware SNMP notification destination is configured with the `vrf` option:

```text
snmp-server host <manager-address> vrf <vrf-name> version 3 priv <username>
```

For example, the following illustrates the transport selection for an existing SNMPv3 user:

```text
R1(config)# snmp-server host 203.0.113.50 vrf MGMT version 3 priv NMS-USER
```

The SNMPv3 group/user configuration and appropriate notifications must also be configured. For example:

```text
R1(config)# snmp-server enable traps
```

This causes enabled notifications to be sent to the manager through `MGMT` instead of relying on a global routing-table lookup.

### SNMP Polling and VRF Contexts

Receiving an SNMP request on a VRF interface is **not the same thing** as selecting which VRF's routing information an SNMP manager can inspect.

IOS XE supports SNMP VPN contexts and context-based access controls on supported platforms. These associate SNMP contexts with VRF-specific MIB information.

Keep the two concepts separate:

```text
SNMP notification VRF:  How the router reaches the NMS.
SNMP context:           Which VRF-related MIB data the NMS can access.
```

Verify SNMP notification destinations with:

```text
R1# show snmp host
```

## VRF-Aware TACACS+ and RADIUS

AAA server groups can select a VRF for communication with authentication, authorization, and accounting servers.

For example, define a TACACS+ server:

```text
R1(config)# tacacs server TAC1
R1(config-server-tacacs)# address ipv4 203.0.113.40
R1(config-server-tacacs)# key <shared-secret>
```

Then associate a TACACS+ server group with `MGMT`:

```text
R1(config)# aaa group server tacacs+ TAC_MGMT
R1(config-sg-tacacs+)# server name TAC1
R1(config-sg-tacacs+)# ip vrf forwarding MGMT
R1(config-sg-tacacs+)# ip tacacs source-interface Ethernet0/0
```

The two final commands are different:

| Command | Function |
|---|---|
| `ip vrf forwarding MGMT` | Uses the `MGMT` routing table to contact the server |
| `ip tacacs source-interface Ethernet0/0` | Selects the source interface for TACACS+ packets |

RADIUS server groups work similarly:

```text
R1(config)# aaa group server radius RAD_MGMT
R1(config-sg-radius)# server name RAD1
R1(config-sg-radius)# ip vrf forwarding MGMT
R1(config-sg-radius)# ip radius source-interface Ethernet0/0
```

The RADIUS server named `RAD1` must already be defined.

**Configuring the AAA server group does not itself enable AAA authentication or accounting.** A method list must reference the group, and the required AAA services must be enabled.

Full configurations, method lists, IPv6 support, verification, and server failover are covered in [AAA Server Groups and VRF-Aware Configuration](../../../infrastructure-security-and-services/device-security/aaa/server-groups.md).

## VRF-Aware Flexible NetFlow Export

Flexible NetFlow can export flow records to a collector through a management VRF.

The VRF for the **export packets** is configured under the flow exporter:

```text
R1(config)# flow exporter EXPORT_MGMT
R1(config-flow-exporter)# destination 203.0.113.60 vrf MGMT
R1(config-flow-exporter)# source Ethernet0/0
R1(config-flow-exporter)# transport udp 2055
R1(config-flow-exporter)# export-protocol netflow-v9
```

- `destination ... vrf MGMT` selects the routing table for the collector.
- `source Ethernet0/0` selects the source interface for export packets.

Verify the exporter:

```text
R1# show flow exporter EXPORT_MGMT
```

**Do not confuse the VRF being monitored with the VRF used to send exported data.** A flow monitor can observe traffic on an interface assigned to a customer VRF while its exporter sends records through `MGMT`.

A flow exporter alone does not collect or export flow data. A flow record and flow monitor must be configured and the monitor applied to the appropriate interface.

## VRF-Aware IP SLA

An IP SLA operation can use a particular VRF to test reachability independently of the global routing table.

Example:

```text
R1(config)# ip sla 10
R1(config-ip-sla)# icmp-echo 203.0.113.70 source-interface Ethernet0/0
R1(config-ip-sla-echo)# vrf MGMT
R1(config-ip-sla-echo)# frequency 10
R1(config-ip-sla-echo)# exit
R1(config-ip-sla)# exit
R1(config)# ip sla schedule 10 life forever start-time now
```

The operation sends ICMP Echo requests through `MGMT` every 10 seconds.

Verify the result:

```text
R1# show ip sla configuration 10
R1# show ip sla statistics 10
```

The `source-interface` option and `vrf` option have separate functions. The former selects the packet's source; the latter selects the routing table for the probe.

Different IP SLA operation types support different optional parameters. Check the configuration submode for the operation being used.

## VRF-Aware FTP and TFTP Clients

File-transfer clients may also need a management VRF to reach backup or software servers.

On supported IOS XE platforms, FTP and TFTP source-interface commands can direct client operations through the VRF-aware interface:

```text
R1(config)# ip ftp source-interface Ethernet0/0
R1(config)# ip tftp source-interface Ethernet0/0
```

Platform and release behavior varies, so verify the selected routing context before relying on these settings. Some file-transfer implementations also expose a VRF selection option directly in the copy operation.

TFTP does not provide encryption or authentication. Prefer an appropriate secure transfer method when supported.

## Troubleshooting VRF-Aware Services

### The Server Is Reachable in a VRF, but the Service Fails

```text
R1# ping vrf MGMT 203.0.113.10
```

A successful ping only verifies IP reachability. Check whether the service itself was configured to use `MGMT`, and whether the server permits the relevant protocol and source address.

### A Source Interface Was Configured, but the Service Uses the Wrong VRF

A source-interface setting and a VRF setting are not interchangeable.

For example, for TACACS+, the following command alone does not replace the VRF configuration on the server group:

```text
ip tacacs source-interface Ethernet0/0
```

Confirm that the appropriate AAA server group includes:

```text
ip vrf forwarding MGMT
```

### The VRF Has No Route to the Server

Verify the selected table:

```text
R1# show ip route vrf MGMT 203.0.113.10
R1# show ip cef vrf MGMT 203.0.113.10
```

Also verify the return route from the server network to the router's source IP address.

### Incoming SSH Fails Only Through a VRF

Check SSH configuration, VTY ACLs, AAA, interface VRF membership, and applicable management-plane restrictions.

If using an inbound VTY `access-class`, check whether `vrf-also` or the supported VRF-specific ACL syntax is needed.

### A NetFlow Collector Does Not Receive Exports

Check both parts of the flow exporter:

```text
destination 203.0.113.60 vrf MGMT
source Ethernet0/0
```

Also verify that a flow monitor is attached to an interface and generates records.

## Verification Commands

| Command | Purpose |
|---|---|
| `show vrf` | Display defined VRFs and interfaces |
| `show ip route vrf MGMT` | Display the management VRF routing table |
| `show ip cef vrf MGMT` | Display VRF forwarding information |
| `ping vrf MGMT <address>` | Test reachability from the management VRF |
| `traceroute vrf MGMT <address>` | Trace a path using the management VRF |
| `show ntp associations` | Inspect NTP associations |
| `show ntp status` | Check NTP synchronization |
| `show logging` | Inspect logging status and destinations |
| `show snmp host` | Display SNMP notification recipients |
| `show flow exporter EXPORT_MGMT` | Inspect the Flexible NetFlow exporter |
| `show ip sla statistics 10` | Inspect the IP SLA operation result |

## Key Points

- VRF-aware services let locally generated management and monitoring traffic use a specific VRF.
- **An interface's VRF membership does not automatically select the VRF for every router-originated service.**
- VRF selection and source-interface selection are different operations, although some services can infer their routing context from the source interface.
- Use `ping vrf` and `traceroute vrf` to test reachability in the correct table.
- The SSH client can explicitly select a VRF using `ssh -vrf`.
- Inbound VTY ACLs may require `vrf-also` or VRF-specific access-class syntax.
- NTP uses `ntp server vrf`; DNS uses `ip name-server vrf`; syslog uses `logging host ... vrf`.
- SNMP notification transport VRFs and SNMP contexts serve different purposes.
- TACACS+ and RADIUS server groups use `ip vrf forwarding` to select the routing table.
- Flexible NetFlow uses `destination ... vrf` under the flow exporter.
- IP SLA operations can use the `vrf` command to perform probes in a chosen routing table.
- Service-specific options and supported syntax vary by IOS XE platform and release.

## References

- [Cisco: VRF-Aware Management on ASR Configuration Examples](https://www.cisco.com/c/en/us/support/docs/routers/asr-1000-series-aggregation-services-routers/116093-configure-vrf-00.html)
- [Cisco: VRF-Aware DNS](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_vrf_aware_dns.html)
- [Cisco: VRF Aware System Message Logging](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/mp_l3_vpns/configuration/xe-16-9/mp-l3-vpns-xe-16-9-book/vrf-aware-system-message-logging.html)
- [Cisco: Network Time Protocol](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/time-sync/ntp-configuration-guide/m-network-time-protocol.html)
- [Cisco: SNMP Support over VPNs, Context-Based Access Control](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/snmp/configuration/xe-17-x/snmp-xe-17-book/nm-snmp-vpn-context.html)
- [Cisco: Flexible NetFlow IPv4 Unicast Flows](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/network-services/network-services/m_fnf-ipv4-uni.html)
- [Cisco: Configuring IP SLAs ICMP Echo Operations](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_sla_icmp_echo.html)
