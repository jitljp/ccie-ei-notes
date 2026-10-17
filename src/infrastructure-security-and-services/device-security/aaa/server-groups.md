# AAA Server Groups and VRF-Aware Configuration

Cisco IOS XE supports **AAA server groups**, which organize TACACS+ and RADIUS servers into named collections.

Server groups allow different AAA method lists to use different authentication servers, and they support group-specific settings such as:

- Server membership.
- VRF selection.
- Source-interface selection.
- Group-specific private servers.

Server groups are especially useful when management traffic must use a dedicated **management VRF** instead of the global routing table.

---

## AAA Server Group Overview

An AAA server group is a logical collection of TACACS+ or RADIUS servers.

For example, a router might use:

- `TAC_ADMIN` for administrator login and command authorization.
- `RAD_USERS` for network access authentication.

Each AAA method list can reference the appropriate server group.

```text
                   R1
                    |
          +---------+---------+
          |                   |
      TAC_ADMIN           RAD_USERS
          |                   |
     +----+----+         +----+----+
     |         |         |         |
    TAC1      TAC2      RAD1      RAD2
```

The server group determines **which servers are eligible** to process AAA requests.

The AAA method list determines **when and how that group is used**.

Creating a server group does not automatically enable authentication, authorization, or accounting.

---

## Named TACACS+ Server Groups

Configure a named TACACS+ server group using:

```text
aaa group server tacacs+ <group-name>
```

Individual TACACS+ servers are added with:

```text
server name <server-name>
```

### Configuration Example

First, define two TACACS+ servers:

```text
R1(config)# tacacs server TAC1
R1(config-server-tacacs)# address ipv4 192.0.2.10
R1(config-server-tacacs)# key Cisco123
!
R1(config)# tacacs server TAC2
R1(config-server-tacacs)# address ipv4 192.0.2.20
R1(config-server-tacacs)# key Cisco123
```

Then create the server group:

```text
R1(config)# aaa group server tacacs+ TAC_ADMIN
R1(config-sg-tacacs+)# server name TAC1
R1(config-sg-tacacs+)# server name TAC2
```

The group `TAC_ADMIN` contains TAC1 and TAC2.

Reference the group in an AAA method list:

```text
R1(config)# aaa authentication login default group TAC_ADMIN local
```

This configures login authentication using the TACACS+ servers in `TAC_ADMIN`, with local authentication as a fallback if the group method returns an error.

### Default vs. Named TACACS+ Groups

IOS XE also supports the built-in TACACS+ group:

```text
group tacacs+
```

Compare:

```text
aaa authentication login default group tacacs+ local
```

```text
aaa authentication login default group TAC_ADMIN local
```

| Method | Server Selection |
|---|---|
| `group tacacs+` | All globally configured TACACS+ servers |
| `group TAC_ADMIN` | Only TACACS+ servers assigned to `TAC_ADMIN` |

**Adding a TACACS+ server to a named group does not remove it from the default TACACS+ group.**

Use a named group when the AAA operation must use a particular subset of servers or group-specific settings.

---

## Named RADIUS Server Groups

RADIUS server groups use similar configuration syntax:

```text
aaa group server radius <group-name>
```

### Configuration Example

Define two RADIUS servers:

```text
R1(config)# radius server RAD1
R1(config-radius-server)# address ipv4 192.0.2.30 auth-port 1812 acct-port 1813
R1(config-radius-server)# key Cisco123
!
R1(config)# radius server RAD2
R1(config-radius-server)# address ipv4 192.0.2.40 auth-port 1812 acct-port 1813
R1(config-radius-server)# key Cisco123
```

Create a server group:

```text
R1(config)# aaa group server radius RAD_USERS
R1(config-sg-radius)# server name RAD1
R1(config-sg-radius)# server name RAD2
```

Reference the group:

```text
R1(config)# aaa authentication login default group RAD_USERS local
```

### Default vs. Named RADIUS Groups

| Method | Server Selection |
|---|---|
| `group radius` | All globally configured RADIUS servers |
| `group RAD_USERS` | Only RADIUS servers assigned to `RAD_USERS` |

As with TACACS+, named RADIUS groups allow separate AAA services to reference different sets of servers.

**TACACS+ and RADIUS server groups are configured separately.** A named group contains servers of only its specified protocol.

---

## Using Different Groups for Different AAA Services

AAA method lists can reference different server groups independently.

For example:

```text
R1(config)# aaa authentication login default group TAC_ADMIN local
R1(config)# aaa authorization exec default group TAC_ADMIN local
R1(config)# aaa accounting exec default start-stop group RAD_USERS
```

In this example:

- Login authentication uses `TAC_ADMIN`.
- EXEC authorization uses `TAC_ADMIN`.
- EXEC accounting sends records to `RAD_USERS`.

The same group does not have to be used for every AAA service.

It is also possible to configure multiple server groups within a method list:

```text
R1(config)# aaa authentication login default group TAC_ADMIN group RAD_USERS local
```

The router attempts the methods in order:

1. TACACS+ servers in `TAC_ADMIN`.
2. RADIUS servers in `RAD_USERS`, if the preceding method returns an error.
3. Local authentication, if the RADIUS method also returns an error.

**An explicit authentication rejection stops method-list processing.** The router does not automatically continue to the next server group after a failed username/password check.

---

## Group-Specific Source Interfaces

A server group can specify a source interface for its AAA traffic.

This allows different groups to use different source IP addresses, even when they use the same protocol.

### TACACS+ Source Interface

The global command is:

```text
ip tacacs source-interface <interface>
```

For example:

```text
R1(config)# ip tacacs source-interface Loopback0
```

To configure a source interface specifically for a TACACS+ server group:

```text
R1(config)# aaa group server tacacs+ TAC_ADMIN
R1(config-sg-tacacs+)# ip tacacs source-interface Loopback1
```

TACACS+ requests using `TAC_ADMIN` are sourced from the address associated with Loopback1.

Other TACACS+ groups can use a different source interface.

### RADIUS Source Interface

The global RADIUS equivalent is:

```text
ip radius source-interface <interface>
```

For example:

```text
R1(config)# ip radius source-interface Loopback0
```

A group-specific source interface can be configured:

```text
R1(config)# aaa group server radius RAD_USERS
R1(config-sg-radius)# ip radius source-interface Loopback1
```

This makes the RADIUS group use Loopback1 as its source interface.

### Source-Interface Precedence

When multiple source-interface configurations exist, the group-specific setting takes precedence.

For example:

```text
R1(config)# ip radius source-interface Loopback0
!
R1(config)# aaa group server radius RAD_USERS
R1(config-sg-radius)# ip radius source-interface Loopback1
```

RADIUS requests using `RAD_USERS` use Loopback1 rather than Loopback0.

On supported IOS XE platforms, RADIUS also supports per-VRF global source-interface configuration:

```text
R1(config)# ip radius source-interface Loopback2 vrf MGMT
```

A RADIUS source-interface configured directly under the server group has higher priority than the global or per-VRF source-interface settings.

| Source-Interface Configuration | Priority |
|---|---|
| Source interface under the server group | Highest |
| Global source interface associated with the group's VRF | Used when no group-specific source interface exists |
| General global source interface | Used when no more specific setting applies |

The source IP address must be reachable from the AAA server.

---

## VRF-Aware AAA

By default, AAA server communication uses the applicable default routing context, normally the **global routing table**.

However, management networks often use a dedicated VRF.

For example:

```text
                R1
         +---------------+
         |               |
         | Global Table  |
         |               |
         | VRF MGMT      |
         +-------+-------+
                 |
            Ethernet0/0
                 |
         Management Network
                 |
        +--------+--------+
        |                 |
    TACACS+            RADIUS
    Server             Server
```

If the AAA servers are reachable only through `MGMT`, the router must use that VRF to communicate with them.

**Configuring an interface in a VRF does not automatically make AAA server traffic use that VRF.**

A VRF must be associated with the appropriate AAA server group.

### VRF Selection vs. Source-Interface Selection

These settings serve different purposes:

| Setting | Purpose |
|---|---|
| `ip vrf forwarding <vrf>` under the server group | Selects the routing table used to reach the AAA server |
| `ip tacacs source-interface <interface>` | Selects the source interface for TACACS+ packets |
| `ip radius source-interface <interface>` | Selects the source interface for RADIUS packets |

**Selecting a source interface does not, by itself, select the VRF used for routing.**

For example, if Loopback0 belongs to `MGMT`, configuring only:

```text
ip tacacs source-interface Loopback0
```

does not replace the need for a VRF-aware TACACS+ server group.

---

## VRF-Aware TACACS+ Configuration

Consider the following addressing:

| Device/Interface | Address | VRF |
|---|---|---|
| R1 Ethernet0/0 | 192.168.99.2/24 | MGMT |
| TACACS+ Server | 192.168.99.101 | Management network |

### Step 1: Configure the Management VRF

```text
R1(config)# vrf definition MGMT
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# exit-address-family
```

### Step 2: Assign an Interface to the VRF

```text
R1(config)# interface Ethernet0/0
R1(config-if)# vrf forwarding MGMT
R1(config-if)# ip address 192.168.99.2 255.255.255.0
R1(config-if)# no shutdown
```

If an interface already has an IPv4 address, assigning it to a VRF removes the existing IPv4 address configuration. Reconfigure the address after assigning the VRF.

### Step 3: Define the TACACS+ Server

```text
R1(config)# tacacs server TAC1
R1(config-server-tacacs)# address ipv4 192.168.99.101
R1(config-server-tacacs)# key Cisco123
```

### Step 4: Configure the VRF-Aware Server Group

```text
R1(config)# aaa group server tacacs+ TAC_MGMT
R1(config-sg-tacacs+)# server name TAC1
R1(config-sg-tacacs+)# ip vrf forwarding MGMT
R1(config-sg-tacacs+)# ip tacacs source-interface Ethernet0/0
```

The commands perform separate functions:

| Command | Purpose |
|---|---|
| `server name TAC1` | Adds TAC1 to the group |
| `ip vrf forwarding MGMT` | Uses the MGMT routing table to reach the server |
| `ip tacacs source-interface Ethernet0/0` | Uses Ethernet0/0 as the source interface |

### Step 5: Reference the Group

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login default group TAC_MGMT local
R1(config)# aaa authorization exec default group TAC_MGMT local
```

R1 now uses the `MGMT` VRF to reach TAC1 when performing the configured AAA operations.

---

## VRF-Aware RADIUS Configuration

RADIUS supports a similar configuration.

Assume:

| Device/Interface | Address | VRF |
|---|---|---|
| R1 Ethernet0/0 | 192.168.99.2/24 | MGMT |
| RADIUS Server | 192.168.99.100 | Management network |

The VRF and interface configuration are the same as in the preceding example.

### Configure the RADIUS Server

```text
R1(config)# radius server RAD1
R1(config-radius-server)# address ipv4 192.168.99.100 auth-port 1812 acct-port 1813
R1(config-radius-server)# key Cisco123
```

### Configure the VRF-Aware Server Group

```text
R1(config)# aaa group server radius RAD_MGMT
R1(config-sg-radius)# server name RAD1
R1(config-sg-radius)# ip vrf forwarding MGMT
R1(config-sg-radius)# ip radius source-interface Ethernet0/0
```

The two key commands are:

```text
ip vrf forwarding MGMT
ip radius source-interface Ethernet0/0
```

The first selects the routing table, and the second selects the source interface.

### Configure Authentication

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login default group RAD_MGMT local
```

R1 now uses the `MGMT` VRF to reach the RADIUS server for login authentication.

---

## IPv6 VRF-Aware AAA

TACACS+ and RADIUS can also use IPv6 transport on supported IOS XE platforms.

For example, create an IPv6-enabled management VRF:

```text
R1(config)# vrf definition MGMT
R1(config-vrf)# address-family ipv6
R1(config-vrf-af)# exit-address-family
```

Configure an interface:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# vrf forwarding MGMT
R1(config-if)# ipv6 address 2001:DB8:99::2/64
```

An IPv6 TACACS+ server can be defined using:

```text
R1(config)# tacacs server TAC6
R1(config-server-tacacs)# address ipv6 2001:DB8:99::101
R1(config-server-tacacs)# key Cisco123
```

An IPv6 RADIUS server can be defined using:

```text
R1(config)# radius server RAD6
R1(config-radius-server)# address ipv6 2001:DB8:99::100 auth-port 1812 acct-port 1813
R1(config-radius-server)# key Cisco123
```

For IPv6 server groups, the VRF association is still configured under the AAA server group.

On platforms supporting IPv6 group-specific source-interface configuration, use:

```text
ipv6 tacacs source-interface <interface>
```

or:

```text
ipv6 radius source-interface <interface>
```

The IPv4 and IPv6 source-interface commands are separate.

Platform and release support should be verified when configuring IPv6 AAA transport and VRF-aware source interfaces.

---

## Private AAA Servers

IOS XE also supports configuring servers directly within a named group using **`server-private`**.

A private server is associated with its specific AAA server group instead of being configured as a globally named server.

This is useful when different groups must use independent server definitions or server-specific settings.

### Private TACACS+ Server

For example:

```text
R1(config)# aaa group server tacacs+ TAC_PRIVATE
R1(config-sg-tacacs+)# server-private 192.0.2.10 port 49 key Cisco123
```

This defines a TACACS+ server that belongs to `TAC_PRIVATE`.

A private TACACS+ server can also be configured in a VRF-aware group:

```text
R1(config)# aaa group server tacacs+ TAC_PRIVATE
R1(config-sg-tacacs+)# server-private 192.168.99.101 port 49 key Cisco123
R1(config-sg-tacacs+)# ip vrf forwarding MGMT
R1(config-sg-tacacs+)# ip tacacs source-interface Ethernet0/0
```

### Private RADIUS Server

RADIUS supports similar functionality:

```text
R1(config)# aaa group server radius RAD_PRIVATE
R1(config-sg-radius)# server-private 192.0.2.30 auth-port 1812 acct-port 1813 key Cisco123
```

The server is defined directly within `RAD_PRIVATE`.

### Named vs. Private Servers

| Feature | `server name` | `server-private` |
|---|---|---|
| Server defined separately | Yes | No |
| Server referenced by local name | Yes | No |
| Server defined directly within group | No | Yes |
| Suitable for VRF-aware groups | Yes | Yes |
| Can share the same server definition among multiple groups | Yes | No |

**A private server is not automatically part of the default `group tacacs+` or `group radius` collection.**

The `server name` approach is generally more convenient when the same server is shared among multiple groups.

---

## AAA Server Selection and Failover

When multiple servers belong to a group, IOS XE can attempt another eligible server if the current server does not respond.

For example:

```text
aaa group server tacacs+ TAC_ADMIN
 server name TAC1
 server name TAC2
```

With:

```text
aaa authentication login default group TAC_ADMIN local
```

The router processes the TACACS+ group as a single AAA method.

If TAC1 does not respond, IOS XE can attempt TAC2 before returning an error for the group.

Only after the group method returns an error can the method list attempt `local`.

An explicit authentication rejection does not trigger fallback to the next method.

**Failover between servers within a group is different from fallback between methods in an AAA method list.**

Detailed timeout, dead-server detection, server-selection, and fallback behavior is covered in the AAA Server Failover and Local Fallback page.

---

## IOS XE Verification

### Display AAA Server Groups

Use:

```text
R1# show running-config | section ^aaa group server
```

This displays the configured groups and their settings.

You can verify:

- Group names.
- Server membership.
- VRF assignments.
- Group-specific source interfaces.
- Private server definitions.

### Display AAA Server Definitions

For TACACS+:

```text
R1# show running-config | section ^tacacs server
```

For RADIUS:

```text
R1# show running-config | section ^radius server
```

### Display AAA Method Lists

```text
R1# show running-config | section ^aaa
```

Verify that the appropriate method lists reference the intended server groups.

### Verify VRF Connectivity

For an IPv4 management VRF:

```text
R1# show ip route vrf MGMT
```

Verify the connected routes and any additional routes required to reach the AAA servers.

Test reachability:

```text
R1# ping vrf MGMT 192.168.99.101
```

On supported platforms, explicitly select the source address if necessary:

```text
R1# ping vrf MGMT 192.168.99.101 source Ethernet0/0
```

For IPv6:

```text
R1# show ipv6 route vrf MGMT
```

Use the platform's VRF-aware IPv6 ping syntax to verify IPv6 reachability.

### Test AAA Authentication

Test the named TACACS+ group:

```text
R1# test aaa group TAC_MGMT admin Cisco123 legacy
```

Test the named RADIUS group:

```text
R1# test aaa group RAD_MGMT admin Cisco123 new-code
```

These commands test authentication through the selected groups.

They do not necessarily test EXEC authorization, command authorization, or accounting.

### Display Server Statistics

For general AAA server information:

```text
R1# show aaa servers
```

For TACACS+:

```text
R1# show tacacs
```

For RADIUS:

```text
R1# show radius statistics
```

### Debug AAA Communication

Useful debugging commands include:

```text
R1# debug aaa authentication
R1# debug tacacs authentication
R1# debug radius authentication
```

Additional packet or server-state debugging may be available depending on the platform.

Disable debugging after troubleshooting:

```text
R1# undebug all
```

> AAA debugging can expose sensitive information and generate substantial output. Use caution on production devices.

---

## Common Problems

| Problem | Possible Cause |
|---|---|
| AAA server reachable globally but not through the group | Incorrect group VRF |
| AAA server reachable in the VRF but authentication times out | Incorrect source interface, server port, shared secret, or ACL |
| A named group uses the wrong servers | Incorrect `server name` entries |
| Different AAA services use unexpected servers | Method list references the wrong group |
| RADIUS server ignores requests | Source IP is not defined as a valid RADIUS client |
| Global TACACS+ communication works but VRF-aware TACACS+ fails | Missing `ip vrf forwarding` under the group |
| Authentication succeeds but EXEC authorization fails | Authorization method list references a different or unavailable group |
| Local fallback occurs unexpectedly | All eligible servers in the selected group returned errors or were unavailable |

### Troubleshooting Checklist

1. Confirm that the correct AAA server definitions exist.
2. Confirm that the servers belong to the intended named group.
3. Confirm that the AAA method list references that group.
4. Verify the VRF configured under the server group.
5. Verify the group's source-interface selection.
6. Confirm that the VRF contains a route to the AAA server.
7. Confirm that the AAA server permits requests from the selected source IP.
8. Verify transport ports and shared secrets.
9. Test authentication and inspect AAA debugging output.

**Successful ping connectivity does not guarantee successful AAA communication.** TACACS+ requires TCP port 49, while traditional RADIUS uses UDP authentication and accounting ports.

---

## Key Points

- AAA server groups organize TACACS+ or RADIUS servers into named collections.
- Configure TACACS+ groups with `aaa group server tacacs+ <name>`.
- Configure RADIUS groups with `aaa group server radius <name>`.
- Use `server name <name>` to add a globally defined server to a group.
- `group tacacs+` references the default TACACS+ server group.
- `group radius` references the default RADIUS server group.
- `group <name>` references the specified named server group.
- Named groups allow different AAA method lists to use different sets of servers.
- A server can belong to multiple named groups.
- Adding a globally configured server to a named group does not remove it from the default group.
- Group-specific source-interface settings override general source-interface settings.
- `ip vrf forwarding <vrf>` under an AAA server group selects the VRF used to reach its servers.
- **Source-interface selection and VRF selection are separate operations.**
- `ip tacacs source-interface` configures the IPv4 TACACS+ source interface.
- `ip radius source-interface` configures the IPv4 RADIUS source interface.
- IPv6 uses separate source-interface commands.
- `server-private` defines a server directly within a named group.
- Private servers are not included in the default global TACACS+ or RADIUS group.
- A named server group is treated as one method within an AAA method list.
- IOS XE can attempt another eligible server within a group before falling back to the next AAA method.
- An explicit authentication rejection does not trigger method-list fallback.
- Use `show running-config`, VRF-aware ping commands, and `test aaa group` to verify server-group configuration.

---

## References

- [Cisco IOS XE - Per VRF for TACACS Servers](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-vrf-tacas-svrs-0.html)
- [Cisco IOS XE 17 - RADIUS Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/sec-crypto/radius/radius-configuration-guide/radius.html)
- [Cisco IOS XE - Per VRF AAA](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_aaa/configuration/xe-3s/sec-usr-aaa-xe-3s-book/sec-per-vrf-aaa.html)
