# Management Plane Protection

**Management Plane Protection (MPP)** is a Cisco IOS XE security feature that restricts which router interfaces can receive management traffic.

MPP allows administrators to designate specific interfaces as **management interfaces** and specify which management protocols are permitted on each interface.

For example, a router could be configured to:

- Accept SSH and SNMP through Ethernet0/0.
- Accept HTTPS through Ethernet0/1.
- Drop supported management traffic received through all other interfaces.

MPP operates within the **Control Plane Protection (CPPr)** framework and is configured under the **control-plane host subinterface**.

Unlike CoPP, MPP does not use token buckets or policing rates. Its purpose is to **restrict management access based on the incoming interface and management protocol**.

---

## MPP Operation

By default, MPP is disabled.

When MPP is disabled, it does not restrict which interfaces can receive management traffic. Access is controlled by other configured features, such as ACLs, VTY restrictions, and authentication.

When MPP is enabled:

1. One or more interfaces are designated as management interfaces.
2. Specific management protocols are permitted on each designated interface.
3. Supported management traffic arriving through an unauthorized interface is dropped.
4. Supported management traffic using a protocol not permitted on the incoming interface is also dropped.

```text
                       R1
                        |
           +------------+------------+
           |            |            |
       Ethernet0/0  Ethernet0/1  Ethernet0/2
           |            |            |
        SSH/SNMP       HTTPS       No Management
           |            |            |
           v            v            v
         Permit       Permit         Drop
```

MPP uses an **implicit deny** approach for supported management protocols once the feature is enabled.

A protocol must be explicitly permitted on at least one designated interface to be accessible through MPP.

### Example

Suppose R1 has the following configuration:

```text
control-plane host
 management-interface Ethernet0/0 allow ssh snmp
```

The resulting behavior is:

| Incoming Interface | Protocol | MPP Action |
|---|---|---|
| Ethernet0/0 | SSH | Permit |
| Ethernet0/0 | SNMP | Permit |
| Ethernet0/0 | Telnet | Drop |
| Ethernet0/0 | HTTPS | Drop |
| Ethernet0/1 | SSH | Drop |
| Ethernet0/1 | SNMP | Drop |
| Ethernet0/2 | SSH | Drop |

Only SSH and SNMP are permitted through Ethernet0/0.

Only protocols explicitly permitted on designated management interfaces are allowed.

For example, permitting SSH and SNMP on Ethernet0/0 does not permit those protocols on any other interface.

---

## Supported Management Protocols

Cisco documents the following management protocols for traditional MPP:

| Protocol | Keyword | Default Port(s) |
|---|---|---|
| SSH | `ssh` | TCP 22 |
| SNMP | `snmp` | UDP 161, 162 |
| Telnet | `telnet` | TCP 23 |
| HTTP | `http` | TCP 80 |
| HTTPS | `https` | TCP 443 |
| FTP | `ftp` | TCP 20, 21 |
| TFTP | `tftp` | UDP 69 |
| Blocks Extensible Exchange Protocol | `beep` | TCP 10288 |

MPP supports the documented management protocols using their recognized well-known ports.

For example, the `ssh` keyword covers both SSHv1 and SSHv2 in the documented implementation.

**SCP is also covered by the SSH keyword** and does not have a separate MPP keyword.

MPP does not provide arbitrary protocol or port filtering. Protocols outside its supported list are not automatically blocked by MPP.

> MPP should not be treated as a general-purpose control-plane firewall. For other protocols or nonstandard management ports, use suitable ACLs, CoPP policies, or other platform-supported controls.

---

## IOS XE Configuration

The configuration syntax is:

```text
control-plane host
 management-interface <interface> allow <protocols>
```

The arguments are:

| Parameter | Purpose |
|---|---|
| `<interface>` | Interface permitted to receive the specified management protocols |
| `allow` | Specifies the protocols permitted on the interface |
| `<protocols>` | One or more supported management protocol keywords |

### Example: Allow SSH and SNMP on One Interface

Configure R1 to accept SSH and SNMP only through Ethernet0/0.

```text
R1(config)# control-plane host
R1(config-cp-host)# management-interface Ethernet0/0 allow ssh snmp
```

This single command:

1. Designates Ethernet0/0 as a management interface.
2. Permits SSH and SNMP through Ethernet0/0.
3. Enables MPP if it was previously disabled.
4. Restricts all other supported management protocols unless explicitly permitted elsewhere.

There is no separate global `enable` command for MPP.

**MPP becomes active when the first management interface is configured.**

A syslog message may indicate that the feature has been enabled:

```text
%CP-5-FEATURE: Management-Interface feature enabled on Control plane host path
```

### Example: Multiple Management Interfaces

Suppose R1 should accept:

- SSH and SNMP through Ethernet0/0.
- HTTPS through Ethernet0/1.
- No management traffic through Ethernet0/2.

Configure:

```text
R1(config)# control-plane host
R1(config-cp-host)# management-interface Ethernet0/0 allow ssh snmp
R1(config-cp-host)# management-interface Ethernet0/1 allow https
```

The resulting behavior is:

| Incoming Interface | SSH | SNMP | HTTPS | Telnet |
|---|---|---|---|---|
| Ethernet0/0 | Permit | Permit | Drop | Drop |
| Ethernet0/1 | Drop | Drop | Permit | Drop |
| Ethernet0/2 | Drop | Drop | Drop | Drop |

**Each management interface can have a different set of permitted protocols.**

The configuration is global under `control-plane host`, rather than under the individual physical interfaces.

---

## Incoming Interface vs. Destination Address

An important distinction is that **MPP restricts the ingress interface, not simply the destination IP address of the management session**.

Consider R1 with:

```text
Ethernet0/0: 10.0.12.1/24
Ethernet0/1: 10.0.13.1/24
Loopback0:   10.255.0.1/32
```

MPP permits SSH only through Ethernet0/0:

```text
R1(config)# control-plane host
R1(config-cp-host)# management-interface Ethernet0/0 allow ssh
```

Now consider two SSH connections to R1's Loopback0 address.

```text
PC1
 |
 | SSH to 10.255.0.1
 |
 v
Ethernet0/0
 |
 R1
 |
 ^ 
 | SSH to 10.255.0.1
 |
Ethernet0/1
 ^
 |
PC2
```

Assuming the packets follow the ordinary supported host processing path:

| Incoming Interface | Destination IP | Protocol | MPP Result |
|---|---|---|---|
| Ethernet0/0 | 10.0.12.1 | SSH | Permit |
| Ethernet0/0 | 10.255.0.1 | SSH | Permit |
| Ethernet0/1 | 10.0.13.1 | SSH | Drop |
| Ethernet0/1 | 10.255.0.1 | SSH | Drop |

Even though both PCs target the same Loopback0 address, the results differ because the traffic enters through different interfaces.

**Configuring Ethernet0/0 as an MPP interface does not mean SSH must be addressed to Ethernet0/0's IP address.**

It means supported SSH traffic must enter through Ethernet0/0 to pass the MPP restriction.

### Loopback Interface Restriction

Traditional MPP does not support configuring loopback interfaces directly as management interfaces.

For example, the following is not supported in the documented implementation:

```text
management-interface Loopback0 allow ssh
```

Instead, designate the supported physical interface through which management traffic enters.

This allows management sessions to use stable loopback IP addresses while MPP restricts the permitted ingress paths.

---

## MPP vs. Source-Based Access Control

MPP does not provide source IP address filtering.

For example:

```text
management-interface Ethernet0/0 allow ssh
```

Permits SSH traffic entering through Ethernet0/0 regardless of the source IP address, provided no other security feature blocks the traffic.

Suppose:

```text
PC1: 10.10.10.10
PC2: 10.10.10.20
```

Both PCs reach the router through Ethernet0/0.

MPP alone cannot distinguish between these two management sources.

To restrict which source IP addresses may establish management sessions, combine MPP with an appropriate ACL.

For example, configure an ACL to permit only PC1:

```text
R1(config)# ip access-list standard SSH-MANAGEMENT
R1(config-std-nacl)# permit host 10.10.10.10
```

Apply it to the VTY lines:

```text
R1(config)# line vty 0 15
R1(config-line)# access-class SSH-MANAGEMENT in
R1(config-line)# transport input ssh
```

Then configure MPP:

```text
R1(config)# control-plane host
R1(config-cp-host)# management-interface Ethernet0/0 allow ssh
```

The restrictions now operate together.

| Source | Incoming Interface | Result |
|---|---|---|
| 10.10.10.10 | Ethernet0/0 | Permitted by MPP and VTY ACL |
| 10.10.10.20 | Ethernet0/0 | Permitted by MPP, denied by VTY ACL |
| 10.10.10.10 | Ethernet0/1 | Denied by MPP |
| 10.10.10.20 | Ethernet0/1 | Denied by MPP and VTY ACL |

This illustrates the distinction:

- **MPP:** Which interfaces can receive management traffic?
- **VTY access-class:** Which source addresses can access the VTY lines?
- **AAA:** Who is permitted to authenticate and what are they authorized to do?

The restrictions complement one another.

---

## MPP vs. CoPP and CPPr

MPP, CoPP, and CPPr provide different forms of control-plane protection.

| Feature | Main Function |
|---|---|
| CoPP | Polices CPU-bound traffic based on traffic classes |
| CPPr | Separates control-plane traffic into host, transit, and CEF-exception processing paths |
| MPP | Restricts which ingress interfaces can receive supported management protocols |

For example:

```text
control-plane host
 management-interface Ethernet0/0 allow ssh snmp
 service-policy input CPPR-HOST
```

On a platform supporting both features, MPP restricts the permitted management ingress paths and protocols, while the host service policy can additionally police applicable CPU-bound traffic.

**MPP does not replace rate-based control-plane policing.**

A permitted SSH connection can still generate excessive traffic. A separate CoPP or CPPr policer may be needed to limit that traffic.

Likewise, CoPP does not inherently enforce MPP's interface-and-protocol restrictions.

---

## MPP Restrictions

Traditional MPP has several important restrictions documented by Cisco.

### IPv4 Only

**MPP protects IPv4 management traffic only.**

Cisco explicitly documents that IPv6 traffic is neither blocked nor denied by traditional MPP.

Therefore, a device configured to restrict IPv4 SSH access through Ethernet0/0 may still accept IPv6 SSH through other interfaces unless additional IPv6 security controls are configured.

This is a particularly important limitation on dual-stack routers.

### CEF Required

IP Cisco Express Forwarding must be enabled before configuring MPP.

Verify:

```text
R1# show ip cef
```

MPP operates within the CPPr infrastructure and relies on the supported packet-processing architecture.

### In-Band Management Interfaces

Traditional MPP supports designated **in-band management interfaces**, which carry both management and ordinary network traffic.

The feature does not support dedicated out-of-band management interfaces in the documented IOS XE implementation.

For example, a dedicated management Ethernet port that bypasses the normal forwarding path may not be controlled by MPP.

> IOS XR has a different MPP implementation with dedicated in-band and out-of-band configuration modes. Do not confuse IOS XR MPP syntax with IOS XE MPP.

### Additional Restrictions

| Restriction | Explanation |
|---|---|
| Stateless | Does not maintain management connection state |
| Well-known ports | Matches supported management protocols on recognized ports |
| Loopback interfaces | Cannot be directly designated as MPP management interfaces |
| Dedicated OOB interfaces | Not supported in the documented implementation |
| Standby/fallback management interfaces | No automatic management-interface failover |
| Hardware-switched/distributed platforms | Traditional MPP is not supported on these architectures |
| IPv6 | Not filtered by traditional MPP |

Support for specific interface types, including certain virtual-template interfaces, depends on the software release.

These restrictions are specific to the traditional IOS XE MPP implementation described here.

---

## Removing MPP Configuration

MPP is enabled when the first management interface is configured.

It is automatically disabled when the **last configured management interface is removed**.

This means that removing the final management-interface restriction does not result in all management traffic being blocked.

Instead, the MPP feature stops enforcing its restrictions.

Other configured management security controls remain in effect.

Before removing or changing an MPP configuration, inspect the active settings:

```text
R1# show running-config | section control-plane
R1# show management-interface
```

Use the relevant `no management-interface` form supported by the platform to remove the configured interface or protocol permissions.

> Modifying MPP remotely can immediately affect management connectivity. Verify that your current access path remains permitted before applying changes.

---

## IOS XE Verification

The primary MPP verification command is:

```text
R1# show management-interface
```

This displays the configured management interfaces and permitted protocols.

### Display All Management Interfaces

```text
R1# show management-interface
```

Example output:

```text
Management interface Ethernet0/0
        Protocol        Packets processed
             ssh                125
            snmp                 48

Management interface Ethernet0/1
        Protocol        Packets processed
           https                 32
```

This indicates that:

- Ethernet0/0 permits SSH and SNMP.
- Ethernet0/1 permits HTTPS.
- The router has processed packets for these management protocols.

The sample counters are illustrative.

### Display a Specific Management Interface

```text
R1# show management-interface Ethernet0/0
```

Displays the permitted protocols and available statistics for Ethernet0/0.

### Display Interfaces Permitting a Protocol

```text
R1# show management-interface protocol ssh
```

Example output:

```text
The following management-interfaces allow protocol ssh
        Ethernet0/0 Packets processed 125
```

This is particularly useful when multiple management interfaces are configured.

### Verify the Running Configuration

```text
R1# show running-config | section control-plane
```

Expected output:

```text
control-plane host
 management-interface Ethernet0/0 allow ssh snmp
 management-interface Ethernet0/1 allow https
```

### Confirm the Management Services

MPP controls permitted ingress paths, but it does not enable SSH, SNMP, or HTTPS itself.

For example, verify the SSH server:

```text
R1# show ip ssh
```

Verify active SSH sessions:

```text
R1# show ssh
```

The appropriate service must be configured and running independently of MPP.

---

## Troubleshooting MPP

### SSH Works on One Interface but Not Another

Check:

```text
R1# show management-interface protocol ssh
```

If SSH is permitted only through Ethernet0/0, connections arriving through Ethernet0/1 are blocked by MPP.

This remains true even if the SSH session targets a loopback IP address.

### SSH Does Not Work on the Permitted Interface

Check the following:

1. Confirm that MPP permits SSH on the incoming interface.
2. Verify that SSH is enabled with `show ip ssh`.
3. Check VTY `access-class` and `transport input` configuration.
4. Verify routing and IP reachability.
5. Check CoPP or CPPr policies for packet drops.
6. Confirm that authentication and authorization are configured correctly.

### IPv6 SSH Still Works Through an Unauthorized Interface

This is an expected limitation of traditional MPP.

MPP does not filter IPv6 traffic.

Use IPv6 ACLs or other supported control-plane security mechanisms to restrict IPv6 management access.

### All Management Access Is Lost After Enabling MPP

Verify that the interface receiving your management traffic was designated as a management interface and that the required protocol was explicitly permitted.

Enabling MPP with a restrictive interface selection can immediately block remote access through all other interfaces.

For production deployments, verify access through a supported management path before enabling MPP and maintain console or other recovery access.

---

## Key Points

- **Management Plane Protection (MPP)** restricts which interfaces can receive supported management traffic.
- MPP operates within the **CPPr host subinterface**.
- Configure MPP under **`control-plane host`**.
- Use **`management-interface <interface> allow <protocols>`** to designate permitted management interfaces and protocols.
- MPP is **disabled by default**.
- Configuring the first management interface **automatically enables MPP**.
- Once enabled, supported management traffic is denied unless explicitly permitted on its incoming interface.
- Different management interfaces can permit different sets of management protocols.
- MPP filters based on the **incoming interface**, not solely the management session's destination IP address.
- MPP does **not** restrict management access by source IP address.
- Use VTY `access-class` or other ACL controls for source-based filtering.
- Supported protocols include SSH, SNMP, Telnet, HTTP, HTTPS, FTP, TFTP, and BEEP.
- SCP is covered by the **SSH** keyword.
- MPP does **not** enable the management services themselves.
- MPP does **not** police traffic rates.
- Traditional MPP protects **IPv4 only**, leaving IPv6 traffic unaffected.
- **CEF must be enabled** for MPP.
- Loopback interfaces and dedicated out-of-band management interfaces are not supported as traditional MPP interfaces.
- Removing the final configured management interface automatically disables MPP.
- Use **`show management-interface`** to verify configuration and statistics.
- Use **`show management-interface protocol <protocol>`** to identify interfaces permitting a specific protocol.
- MPP support varies by IOS XE platform.

---

## References

- [Cisco IOS XE 17.x - Management Plane Protection](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-mgt-plane-prt.html)
- [Cisco IOS XE Software Hardening Guide](https://sec.cloudapps.cisco.com/security/center/resources/IOS_XE_hardening)
- [Cisco IOS Security Command Reference - show management-interface](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/security/s1/sec-s1-cr-book/sec-cr-s4.html)
- [Cisco IOS - Management Plane Protection](https://www.cisco.com/c/en/us/td/docs/ios/security/configuration/guide/sec_mgmt_plane_prot.html)