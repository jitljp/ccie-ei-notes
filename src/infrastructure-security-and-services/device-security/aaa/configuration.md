
# Complete AAA Configuration

This page brings together the IOS XE AAA features covered in the preceding sections.

It provides complete configuration examples for:

- TACACS+ authentication, authorization, and accounting.
- RADIUS authentication, authorization, and accounting.
- Named AAA method lists.
- AAA server groups and server failover.
- Local fallback.
- VRF-aware AAA.
- Console and VTY line configuration.

The examples use documentation IP addresses and illustrative credentials. Replace them with appropriate values for the actual deployment.

---

## AAA Configuration Overview

A complete AAA configuration can contain several independent components.

| Component | Purpose |
|---|---|
| Local users | Provide local credentials and privilege levels |
| AAA servers | Define TACACS+ or RADIUS servers |
| AAA server groups | Organize servers and control which servers are used |
| Authentication method lists | Determine how users authenticate |
| Authorization method lists | Determine EXEC access and command permissions |
| Accounting method lists | Determine which activities are recorded |
| Line configuration | Apply named method lists to console, VTY, and other lines |

**Authentication, authorization, and accounting must be configured independently.**

For example:

```text
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local
aaa authorization commands 15 default group tacacs+ local
aaa accounting exec default start-stop group tacacs+
aaa accounting commands 15 default stop-only group tacacs+
```

These commands configure five separate AAA functions.

A successful login does not automatically authorize the EXEC session, authorize individual commands, or generate accounting records.

### Recommended Configuration Order

When configuring AAA on a device, a practical order is:

1. Configure local administrative credentials.
2. Configure the remote AAA servers and server groups.
3. Enable AAA using `aaa new-model`.
4. Configure login authentication.
5. Test remote login and local fallback.
6. Configure EXEC authorization.
7. Configure command authorization, if required.
8. Configure accounting.
9. Apply named method lists to the relevant lines.
10. Verify normal operation and server-failure behavior.

**Maintain an alternative administrative session while changing AAA settings.**

Incorrect AAA configuration can prevent administrators from accessing the device.

---

## Complete TACACS+ Configuration

This example uses TACACS+ for centralized device administration.

### Requirements

R1 must:

- Authenticate SSH administrators using TACACS+.
- Use a second TACACS+ server if the first is unavailable.
- Fall back to local authentication if both servers are unavailable.
- Obtain EXEC authorization through TACACS+.
- Authorize privilege level 15 commands through TACACS+.
- Include configuration-mode commands in command authorization.
- Record EXEC sessions and commands.
- Preserve local access during a TACACS+ outage.

The topology is:

```text
             TACACS+ Servers
            +----------------+
            | TAC1           |
            | 192.0.2.10     |
            +----------------+
                    |
                    |
    R1 ------------+
                    |
                    |
            +----------------+
            | TAC2           |
            | 192.0.2.11     |
            +----------------+
```

Assume R1 already has IP connectivity to both servers.

### Step 1: Configure Local Credentials

```text
R1(config)# username admin privilege 15 secret Local123
R1(config)# enable secret Enable123
```

The local user provides login authentication fallback.

The local privilege level is also relevant to local EXEC authorization.

The enable secret provides local enable authentication when the `enable` method is configured.

### Step 2: Configure TACACS+ Servers

```text
R1(config)# tacacs server TAC1
R1(config-server-tacacs)# address ipv4 192.0.2.10
R1(config-server-tacacs)# key Cisco123
R1(config-server-tacacs)# timeout 5
!
R1(config)# tacacs server TAC2
R1(config-server-tacacs)# address ipv4 192.0.2.11
R1(config-server-tacacs)# key Cisco123
R1(config-server-tacacs)# timeout 5
```

The TACACS+ servers use TCP port 49 by default.

The server-side shared secrets must match.

### Step 3: Configure a TACACS+ Server Group

```text
R1(config)# aaa group server tacacs+ TAC_SERVERS
R1(config-sg-tacacs+)# server name TAC1
R1(config-sg-tacacs+)# server name TAC2
```

The named server group contains both TACACS+ servers.

Method lists using `group TAC_SERVERS` can attempt another eligible server if the selected server is unavailable.

### Step 4: Enable AAA

```text
R1(config)# aaa new-model
```

This enables the IOS XE AAA framework.

### Step 5: Configure Authentication

```text
R1(config)# aaa authentication login default group TAC_SERVERS local
R1(config)# aaa authentication enable default group TAC_SERVERS enable
```

The login authentication method list uses:

1. TACACS+.
2. Local username database fallback.

The enable authentication method list uses:

1. TACACS+.
2. Local enable-secret fallback.

**Local fallback occurs only when the preceding method returns an error, not when the server explicitly rejects authentication.**

### Step 6: Configure EXEC Authorization

```text
R1(config)# aaa authorization exec default group TAC_SERVERS local
```

TACACS+ is used to authorize the EXEC session and can provide attributes such as privilege level.

If the TACACS+ method cannot complete authorization, IOS XE attempts local authorization.

For example, the TACACS+ server can return privilege level 15 using:

```text
service=shell
priv-lvl=15
```

The local user's `privilege 15` configuration supports the local fallback case.

### Step 7: Configure Command Authorization

```text
R1(config)# aaa authorization commands 15 default group TAC_SERVERS local
R1(config)# aaa authorization config-commands
```

The first command enables authorization for privilege level 15 commands.

The second explicitly enables configuration-mode command authorization.

For example, the TACACS+ server can independently authorize:

```text
configure terminal
interface Ethernet0/0
shutdown
```

**Cisco documentation suggests that configuration-command authorization may be enabled automatically when command authorization is configured. However, testing on IOS and IOS XE showed that `aaa authorization config-commands` was required to enable it.**

Explicitly configuring the command avoids depending on the platform default.

If level 1 commands must also be authorized, configure a separate method list:

```text
R1(config)# aaa authorization commands 1 default group TAC_SERVERS local
```

### Step 8: Configure Accounting

```text
R1(config)# aaa accounting exec default start-stop group TAC_SERVERS
R1(config)# aaa accounting commands 1 default stop-only group TAC_SERVERS
R1(config)# aaa accounting commands 15 default stop-only group TAC_SERVERS
R1(config)# aaa accounting system default start-stop group TAC_SERVERS
```

These method lists record:

| Accounting Type | Activity |
|---|---|
| EXEC | Start and end of each EXEC session |
| Commands 1 | Individual privilege level 1 commands |
| Commands 15 | Individual privilege level 15 commands |
| System | Supported system-level events |

Command accounting is separate from command authorization.

For example, even if command authorization is not configured, applicable commands can still be accounted for.

### Step 9: Configure SSH Access

```text
R1(config)# ip domain name example.com
R1(config)# crypto key generate rsa modulus 2048
R1(config)# ip ssh version 2
!
R1(config)# line vty 0 15
R1(config-line)# transport input ssh
R1(config-line)# login authentication default
R1(config-line)# authorization exec default
R1(config-line)# authorization commands 15 default
R1(config-line)# accounting exec default
R1(config-line)# accounting commands 15 default
```

The commands explicitly associate the VTY lines with the default AAA method lists.

**Default AAA method lists are generally applied automatically to applicable lines.** Therefore, the explicit `default` commands under the VTY lines are normally unnecessary.

Named method lists, however, require explicit line assignment.

### Complete Configuration

The resulting configuration is:

```text
hostname R1
!
username admin privilege 15 secret Local123
enable secret Enable123
!
aaa new-model
!
tacacs server TAC1
 address ipv4 192.0.2.10
 key Cisco123
 timeout 5
!
tacacs server TAC2
 address ipv4 192.0.2.11
 key Cisco123
 timeout 5
!
aaa group server tacacs+ TAC_SERVERS
 server name TAC1
 server name TAC2
!
aaa authentication login default group TAC_SERVERS local
aaa authentication enable default group TAC_SERVERS enable
!
aaa authorization exec default group TAC_SERVERS local
aaa authorization commands 15 default group TAC_SERVERS local
aaa authorization config-commands
!
aaa accounting exec default start-stop group TAC_SERVERS
aaa accounting commands 1 default stop-only group TAC_SERVERS
aaa accounting commands 15 default stop-only group TAC_SERVERS
aaa accounting system default start-stop group TAC_SERVERS
!
ip domain name example.com
ip ssh version 2
!
line vty 0 15
 transport input ssh
 login authentication default
 authorization exec default
 authorization commands 15 default
 accounting exec default
 accounting commands 15 default
```

RSA keys must also be generated for SSH operation. They are not represented as ordinary running-configuration commands.

### Expected Behavior

| Condition | Result |
|---|---|
| TAC1 authenticates the user | Authentication succeeds |
| TAC1 unavailable, TAC2 responds | TAC2 can complete authentication |
| Both TACACS+ servers unavailable | Local authentication fallback |
| TACACS+ explicitly rejects credentials | Authentication fails; no local fallback |
| TACACS+ authorizes privilege 15 | User can receive privilege level 15 |
| TACACS+ explicitly denies a command | Command denied |
| TACACS+ command authorization unavailable | Local command authorization attempted |
| Administrator enters a tracked command | Accounting record generated |
| Administrator logs out | EXEC STOP record generated |

---

## Complete RADIUS Configuration

RADIUS can also provide centralized authentication, EXEC authorization, and accounting for administrative access.

However, **TACACS+ is generally preferred for granular CLI command authorization**.

This example uses RADIUS without command authorization.

### Requirements

R1 must:

- Authenticate SSH administrators using RADIUS.
- Use two RADIUS servers.
- Fall back to the local username database if the RADIUS servers are unavailable.
- Obtain EXEC authorization through RADIUS.
- Record EXEC sessions using RADIUS.

### Step 1: Configure Local Credentials

```text
R1(config)# username admin privilege 15 secret Local123
R1(config)# enable secret Enable123
```

### Step 2: Configure RADIUS Servers

```text
R1(config)# radius server RAD1
R1(config-radius-server)# address ipv4 192.0.2.20 auth-port 1812 acct-port 1813
R1(config-radius-server)# key Cisco123
R1(config-radius-server)# timeout 3
R1(config-radius-server)# retransmit 2
!
R1(config)# radius server RAD2
R1(config-radius-server)# address ipv4 192.0.2.21 auth-port 1812 acct-port 1813
R1(config-radius-server)# key Cisco123
R1(config-radius-server)# timeout 3
R1(config-radius-server)# retransmit 2
```

The ports are specified explicitly:

| RADIUS Service | UDP Port |
|---|---|
| Authentication | 1812 |
| Accounting | 1813 |

Some IOS XE images may use legacy default ports 1645 and 1646 when the ports are omitted.

Specifying the ports avoids ambiguity.

### Step 3: Configure a RADIUS Server Group

```text
R1(config)# aaa group server radius RAD_SERVERS
R1(config-sg-radius)# server name RAD1
R1(config-sg-radius)# server name RAD2
```

### Step 4: Configure AAA

```text
R1(config)# aaa new-model
!
R1(config)# aaa authentication login default group RAD_SERVERS local
R1(config)# aaa authorization exec default group RAD_SERVERS local
R1(config)# aaa accounting exec default start-stop group RAD_SERVERS
```

Login authentication uses RADIUS, with local fallback.

EXEC authorization uses RADIUS, with local fallback.

EXEC accounting records session start and end events.

### Step 5: Configure the RADIUS Server

For privilege level 15, the RADIUS server can return the Cisco vendor-specific attribute:

```text
Cisco-AVPair = "shell:priv-lvl=15"
```

This permits IOS XE to assign the appropriate EXEC privilege level when the RADIUS authorization information is processed.

The exact server-side configuration syntax depends on the RADIUS implementation.

### Step 6: Configure SSH

```text
R1(config)# ip domain name example.com
R1(config)# crypto key generate rsa modulus 2048
R1(config)# ip ssh version 2
!
R1(config)# line vty 0 15
R1(config-line)# transport input ssh
```

The default login authentication, EXEC authorization, and EXEC accounting lists apply automatically to applicable VTY lines.

### Complete RADIUS Configuration

```text
hostname R1
!
username admin privilege 15 secret Local123
enable secret Enable123
!
aaa new-model
!
radius server RAD1
 address ipv4 192.0.2.20 auth-port 1812 acct-port 1813
 key Cisco123
 timeout 3
 retransmit 2
!
radius server RAD2
 address ipv4 192.0.2.21 auth-port 1812 acct-port 1813
 key Cisco123
 timeout 3
 retransmit 2
!
aaa group server radius RAD_SERVERS
 server name RAD1
 server name RAD2
!
aaa authentication login default group RAD_SERVERS local
aaa authorization exec default group RAD_SERVERS local
aaa accounting exec default start-stop group RAD_SERVERS
!
ip domain name example.com
ip ssh version 2
!
line vty 0 15
 transport input ssh
```

The RSA keys must be generated separately.

### RADIUS Dead-Server Detection

RADIUS failover can be improved using dead-server detection:

```text
R1(config)# radius-server dead-criteria time 10 tries 3
R1(config)# radius-server deadtime 5
```

The first command configures the dead-server detection criteria.

Both conditions must be satisfied:

- The configured time since the last valid response must have elapsed.
- The required consecutive request timeouts must occur.

The second command specifies a deadtime of 5 minutes.

A RADIUS server marked dead is generally avoided until it becomes eligible again.

An explicit RADIUS Access-Reject is not a server failure and does not trigger local fallback.

---

## Using Named AAA Method Lists

The previous examples use default method lists.

Named lists allow different AAA policies to be assigned to different lines.

For example, suppose R1 should use TACACS+ on VTY lines 0 through 4 and local authentication on VTY lines 5 through 15.

### Define Named Lists

```text
R1(config)# aaa authentication login SSH_LOGIN group TAC_SERVERS local
R1(config)# aaa authorization exec SSH_EXEC group TAC_SERVERS local
R1(config)# aaa authorization commands 15 SSH_CMDS group TAC_SERVERS local
R1(config)# aaa accounting exec SSH_ACCT start-stop group TAC_SERVERS
R1(config)# aaa accounting commands 15 SSH_CMD_ACCT stop-only group TAC_SERVERS
!
R1(config)# aaa authentication login LOCAL_LOGIN local
R1(config)# aaa authorization exec LOCAL_EXEC local
```

### Apply TACACS+ Lists

```text
R1(config)# line vty 0 4
R1(config-line)# login authentication SSH_LOGIN
R1(config-line)# authorization exec SSH_EXEC
R1(config-line)# authorization commands 15 SSH_CMDS
R1(config-line)# accounting exec SSH_ACCT
R1(config-line)# accounting commands 15 SSH_CMD_ACCT
```

### Apply Local Lists

```text
R1(config)# line vty 5 15
R1(config-line)# login authentication LOCAL_LOGIN
R1(config-line)# authorization exec LOCAL_EXEC
```

The resulting behavior is:

| VTY Lines | Authentication | EXEC Authorization | Accounting |
|---|---|---|---|
| 0-4 | TACACS+, then local | TACACS+, then local | TACACS+ |
| 5-15 | Local only | Local only | No explicit list |

**Named method lists do not apply automatically.** They must be selected under the appropriate terminal lines or interfaces.

---

## VRF-Aware TACACS+ Configuration

AAA servers can be reached through a management VRF.

This is useful when management traffic is isolated from the global routing table.

### Requirements

R1 must:

- Reach the TACACS+ servers through VRF `MGMT`.
- Source TACACS+ packets from `Ethernet0/0`.
- Use local fallback if the servers cannot be reached.

Assume:

| Component | Address |
|---|---|
| R1 Ethernet0/0 | 192.0.2.2/24 |
| TAC1 | 192.0.2.10 |
| TAC2 | 192.0.2.11 |
| VRF | MGMT |

### Step 1: Configure the VRF

```text
R1(config)# vrf definition MGMT
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# exit-address-family
```

### Step 2: Configure the Management Interface

```text
R1(config)# interface Ethernet0/0
R1(config-if)# vrf forwarding MGMT
R1(config-if)# ip address 192.0.2.2 255.255.255.0
R1(config-if)# no shutdown
```

The interface belongs to the `MGMT` VRF.

### Step 3: Configure TACACS+ Servers

```text
R1(config)# tacacs server TAC1
R1(config-server-tacacs)# address ipv4 192.0.2.10
R1(config-server-tacacs)# key Cisco123
!
R1(config)# tacacs server TAC2
R1(config-server-tacacs)# address ipv4 192.0.2.11
R1(config-server-tacacs)# key Cisco123
```

### Step 4: Configure the VRF-Aware Server Group

```text
R1(config)# aaa group server tacacs+ TAC_MGMT
R1(config-sg-tacacs+)# server name TAC1
R1(config-sg-tacacs+)# server name TAC2
R1(config-sg-tacacs+)# ip vrf forwarding MGMT
R1(config-sg-tacacs+)# ip tacacs source-interface Ethernet0/0
```

The two important commands are:

| Command | Purpose |
|---|---|
| `ip vrf forwarding MGMT` | Specifies the VRF used for AAA server communication |
| `ip tacacs source-interface Ethernet0/0` | Specifies the source interface for TACACS+ packets |

**Configuring a source interface does not, by itself, select the VRF.**

The server group must be associated with the appropriate VRF.

### Step 5: Configure AAA Method Lists

```text
R1(config)# username admin privilege 15 secret Local123
R1(config)# aaa new-model
R1(config)# aaa authentication login default group TAC_MGMT local
R1(config)# aaa authorization exec default group TAC_MGMT local
R1(config)# aaa accounting exec default start-stop group TAC_MGMT
```

The TACACS+ method attempts server communication through `MGMT`.

If the selected server group cannot complete authentication, IOS XE can fall back to the local database.

### Verify VRF Reachability

```text
R1# ping vrf MGMT 192.0.2.10 source Ethernet0/0
R1# ping vrf MGMT 192.0.2.11 source Ethernet0/0
```

Successful ICMP reachability does not guarantee that TACACS+ will work, but it verifies basic IP connectivity through the correct VRF.

If the servers are on remote networks, the management VRF must also have appropriate routes.

---

## VRF-Aware RADIUS Configuration

RADIUS server groups support similar VRF-aware configuration.

Assume the `MGMT` VRF and management interface are already configured.

### Configure RADIUS Servers

```text
R1(config)# radius server RAD1
R1(config-radius-server)# address ipv4 192.0.2.20 auth-port 1812 acct-port 1813
R1(config-radius-server)# key Cisco123
!
R1(config)# radius server RAD2
R1(config-radius-server)# address ipv4 192.0.2.21 auth-port 1812 acct-port 1813
R1(config-radius-server)# key Cisco123
```

### Configure the Server Group

```text
R1(config)# aaa group server radius RAD_MGMT
R1(config-sg-radius)# server name RAD1
R1(config-sg-radius)# server name RAD2
R1(config-sg-radius)# ip vrf forwarding MGMT
R1(config-sg-radius)# ip radius source-interface Ethernet0/0
```

### Configure AAA

```text
R1(config)# aaa authentication login default group RAD_MGMT local
R1(config)# aaa authorization exec default group RAD_MGMT local
R1(config)# aaa accounting exec default start-stop group RAD_MGMT
```

The method lists use the RADIUS servers through the specified VRF.

---

## Console AAA Configuration

By default, console login authentication and console authorization have different behavior.

In particular, **AAA authorization is not applied to the console unless explicitly enabled**.

### Console Login Authentication

For example:

```text
R1(config)# aaa authentication login CONSOLE_LOGIN local
!
R1(config)# line console 0
R1(config-line)# login authentication CONSOLE_LOGIN
```

This authenticates console users using the local username database.

### Console Authorization

To enable AAA authorization on console sessions:

```text
R1(config)# aaa authorization console
```

For example:

```text
R1(config)# aaa authorization exec CONSOLE_EXEC local
R1(config)# aaa authorization commands 15 CONSOLE_CMDS local
R1(config)# aaa authorization console
!
R1(config)# line console 0
R1(config-line)# authorization exec CONSOLE_EXEC
R1(config-line)# authorization commands 15 CONSOLE_CMDS
```

The console now uses the corresponding authorization lists.

### Console Configuration-Mode Commands

To enable configuration-mode command authorization:

```text
R1(config)# aaa authorization config-commands
```

`aaa authorization console` and `aaa authorization config-commands` are different commands:

| Command | Purpose |
|---|---|
| `aaa authorization console` | Enables AAA authorization on the console |
| `aaa authorization config-commands` | Enables authorization of configuration-mode commands |

The first does not independently enable the second.

### Console Recovery Considerations

A common recovery-oriented design is to authenticate console users locally while using centralized AAA for SSH access.

This avoids depending on TACACS+ or RADIUS connectivity for basic console login.

If console authorization is enabled, ensure that the authorization method lists do not prevent local recovery during an AAA outage.

---

## AAA Configuration with Only Local Authentication

Remote AAA servers are not required to use the AAA framework.

For example:

```text
R1(config)# username admin privilege 15 secret Local123
R1(config)# aaa new-model
R1(config)# aaa authentication login default local
R1(config)# aaa authorization exec default local
!
R1(config)# line vty 0 15
R1(config-line)# transport input ssh
```

This configuration:

- Authenticates users against the local username database.
- Uses local EXEC authorization.
- Does not require TACACS+ or RADIUS connectivity.

The local username and privilege level are configured using standard IOS XE local-user commands.

SSH still requires the usual prerequisites, including device addressing, RSA keys, and appropriate SSH configuration.

---

## Configuration and Fallback Summary

The following table summarizes common AAA configuration patterns.

| Requirement | Configuration |
|---|---|
| TACACS+ login with local fallback | `aaa authentication login default group tacacs+ local` |
| RADIUS login with local fallback | `aaa authentication login default group radius local` |
| Local-only login | `aaa authentication login default local` |
| TACACS+ enable authentication with local enable fallback | `aaa authentication enable default group tacacs+ enable` |
| TACACS+ EXEC authorization with local fallback | `aaa authorization exec default group tacacs+ local` |
| TACACS+ command authorization | `aaa authorization commands 15 default group tacacs+ local` |
| Configuration command authorization | `aaa authorization config-commands` |
| Console AAA authorization | `aaa authorization console` |
| TACACS+ EXEC accounting | `aaa accounting exec default start-stop group tacacs+` |
| TACACS+ level 15 command accounting | `aaa accounting commands 15 default stop-only group tacacs+` |
| RADIUS EXEC accounting | `aaa accounting exec default start-stop group radius` |

### Fallback Rules

Remember the distinction:

| Remote AAA Result | Behavior |
|---|---|
| PASS | Accept the result; stop processing |
| FAIL | Reject the request; stop processing |
| ERROR | Attempt the next configured method, if present |

For authentication, an explicit TACACS+ rejection or RADIUS Access-Reject prevents local fallback.

For authorization, an explicit server denial also prevents fallback to `local`, `if-authenticated`, or `none`.

**Local fallback is intended for AAA server or method failures, not for overriding centralized authentication or authorization decisions.**

---

## Verification Checklist

After configuring AAA, verify both normal operation and fallback behavior.

### Check Server and Method Configuration

```text
R1# show running-config | section ^tacacs server
R1# show running-config | section ^radius server
R1# show running-config | section ^aaa group server
R1# show running-config | include ^aaa authentication
R1# show running-config | include ^aaa authorization
R1# show running-config | include ^aaa accounting
```

### Check Server Status

```text
R1# show aaa servers
R1# show tacacs
R1# show radius statistics
```

The available output depends on the platform and software release.

### Check AAA Debugging

```text
R1# debug aaa authentication
R1# debug aaa authorization
R1# debug aaa accounting
```

For TACACS+:

```text
R1# debug tacacs authentication
R1# debug tacacs authorization
R1# debug tacacs accounting
```

For RADIUS:

```text
R1# debug radius authentication
R1# debug radius accounting
```

Use the supported commands for the specific IOS XE image.

Disable debugging when finished:

```text
R1# undebug all
```

### Functional Tests

Verify the following behaviors:

1. A valid remote AAA username and password allow login.
2. Invalid credentials are rejected without local fallback.
3. An unavailable primary server causes an alternate server to be attempted.
4. Unavailable remote AAA servers allow local login fallback.
5. EXEC authorization supplies the expected privilege level.
6. Local EXEC authorization works during an AAA outage.
7. TACACS+ permits or denies commands according to its policies.
8. Configuration-mode command authorization works when explicitly enabled.
9. EXEC accounting records session starts and stops.
10. Command accounting records commands assigned to the configured privilege levels.
11. Console recovery access remains available.

For command authorization troubleshooting, remember that **successful command execution does not prove TACACS+ authorized the command**.

The server might have returned an error, allowing `local` or another fallback method to authorize it.

Check the actual AAA method result.

---

## Key Points

- A complete AAA configuration can include local users, remote servers, server groups, authentication, authorization, accounting, and line assignments.
- Configure local administrative credentials before enabling remote AAA.
- `aaa new-model` enables the AAA framework.
- Authentication, authorization, and accounting require separate method lists.
- TACACS+ is preferred for granular CLI command authorization.
- RADIUS supports centralized authentication, EXEC authorization, and accounting.
- Named AAA server groups are configured using `aaa group server tacacs+` or `aaa group server radius`.
- `server name <name>` associates a globally defined server with a named group.
- `group tacacs+` and `group radius` refer to the respective generic server groups.
- Named method lists must be explicitly assigned to the relevant lines.
- Default method lists are normally applied automatically to applicable lines.
- `local` provides local username authentication fallback.
- Enable authentication uses `enable` for local enable-secret fallback.
- EXEC authorization requires its own fallback method list.
- Command authorization requires its own fallback method list.
- **AAA method fallback occurs on ERROR, not on explicit FAIL.**
- A TACACS+ or RADIUS authentication rejection does not trigger local fallback.
- Command authorization is configured separately for each command privilege level.
- Explicitly configure `aaa authorization config-commands` to enable configuration-mode command authorization.
- Command accounting is independent of command authorization.
- EXEC accounting records an entire EXEC session.
- Command accounting records individual commands belonging to the configured privilege levels.
- VRF-aware AAA requires a VRF association under the server group.
- Configuring a TACACS+ or RADIUS source interface does not independently select the VRF.
- Console AAA authorization requires `aaa authorization console`.
- A local console recovery configuration can help preserve administrative access during AAA outages.
- Test authentication, authorization, accounting, server failover, and local fallback separately.

---

## References

- [Cisco IOS XE: Configuring Authentication](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_aaa/configuration/xe-16-11/sec-usr-aaa-xe-16-11-book/sec-cfg-authentn.html)
- [Cisco IOS XE: Configuring Authorization](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_aaa/configuration/xe-16-11/sec-usr-aaa-xe-16-11-book/sec-cfg-authorizatn.html)
- [Cisco IOS XE: AAA Command Reference](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/security/a1/sec-a1-cr-book/sec-cr-a1.html)
- [Cisco IOS XE: Per VRF for TACACS Servers](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-vrf-tacas-svrs-0.html)
- [RFC 8907: The TACACS+ Protocol](https://www.rfc-editor.org/rfc/rfc8907)
- [RFC 2865: RADIUS](https://www.rfc-editor.org/rfc/rfc2865)
- [RFC 2866: RADIUS Accounting](https://www.rfc-editor.org/rfc/rfc2866)
