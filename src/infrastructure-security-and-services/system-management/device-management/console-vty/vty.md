# VTY Line Configuration and Remote Access

**Virtual terminal (VTY) lines** provide remote access to the Cisco IOS XE CLI through network protocols such as Telnet and SSH.

Unlike console and AUX lines, VTY lines are not associated with physical communication ports. They are virtual resources allocated to incoming remote management sessions.

Each incoming interactive Telnet or SSH session occupies a VTY line until the session terminates.

## VTY Line Numbering

VTY lines are numbered starting at 0.

The number of available VTY lines depends on the platform. Many Cisco devices provide 16 VTY lines, numbered 0 through 15.

The following command configures all 16 VTY lines:

```text
R1(config)# line vty 0 15
R1(config-line)#
```

Individual VTY lines or ranges of VTY lines can be configured separately.

```text
R1(config)# line vty 0
```

```text
R1(config)# line vty 0 4
```

```text
R1(config)# line vty 5 15
```

The specified range is inclusive. For example, `line vty 0 4` selects five VTY lines.

### Absolute and Relative Line Numbers

The `show line` command displays absolute line numbers, which account for other line types, such as console and AUX lines.

For example:

```text
R1# show line
   Tty Typ     Tx/Rx    A Modem  Roty AccO AccI  Uses  Noise Overruns
*    0 CTY              -    -      -    -    -     0      0    0/0
     1 AUX   9600/9600  -    -      -    -    -     0      0    0/0
     2 VTY              -    -      -    -    -     1      0    0/0
     3 VTY              -    -      -    -    -     0      0    0/0
     4 VTY              -    -      -    -    -     0      0    0/0
     5 VTY              -    -      -    -    -     0      0    0/0
     6 VTY              -    -      -    -    -     0      0    0/0
```

In this example:

| Absolute Line Number | Relative Line Number |
|---:|---|
| 0 | Console 0 |
| 1 | AUX 0 |
| 2 | VTY 0 |
| 3 | VTY 1 |
| 4 | VTY 2 |
| 5 | VTY 3 |
| 6 | VTY 4 |

The `line vty` command uses relative VTY line numbers.

For example, the following command configures absolute line 2 in the preceding output:

```text
R1(config)# line vty 0
```

Absolute line numbering can differ between platforms.

## VTY Line Allocation

VTY lines are shared resources available to incoming remote management connections.

Incoming connections are normally assigned to the first eligible available VTY line.

For example, assume all VTY lines are initially available and have identical configurations:

```text
                            R1
                   +-----------------+
SSH Session 1 ---->| VTY 0           |
                   |                 |
Telnet Session 2 ->| VTY 1           |
                   |                 |
SSH Session 3 ---->| VTY 2           |
                   |                 |
                   | VTY 3 (Free)    |
                   | VTY 4 (Free)    |
                   | ...             |
                   | VTY 15 (Free)   |
                   +-----------------+
```

Both Telnet and SSH use the same pool of VTY lines for interactive CLI sessions.

When a session terminates, its VTY line becomes available for another connection.

If no eligible VTY line is available, a new interactive remote management connection cannot be established.

### VTY Lines and Physical Interfaces

VTY lines are independent of the physical interface through which a management connection arrives.

For example:

```text
                            R1
                   +------------------+
Administrator 1 -->| G1               |
                   |         VTY 0    |
Administrator 2 -->| G2               |
                   |         VTY 1    |
                   +------------------+
```

Both administrators can establish remote CLI sessions even though their connections arrive through different physical interfaces.

Multiple connections arriving through the same interface can also occupy different VTY lines.

### Separate VTY Line Configurations

Different VTY line ranges can have different configurations.

For example:

```text
R1(config)# line vty 0 4
R1(config-line)# exec-timeout 5 0
!
R1(config)# line vty 5 15
R1(config-line)# exec-timeout 15 0
```

In this example:

- VTY lines 0 through 4 have a five-minute EXEC inactivity timeout.
- VTY lines 5 through 15 have a 15-minute EXEC inactivity timeout.

Because incoming connections are dynamically allocated to VTY lines, administrators should ensure that security restrictions are consistently applied to all relevant lines.

Rotary groups can be used to explicitly direct connections to particular groups of lines. They are covered in [Rotary Groups and Custom TCP Ports](rotary-groups.md).

## Incoming Transport Protocols

The `transport input` command specifies which transport protocols can establish incoming connections to a line.

For example, to permit only SSH:

```text
R1(config)# line vty 0 15
R1(config-line)# transport input ssh
```

Common configurations include:

| Command | Effect |
|---|---|
| `transport input ssh` | Permit incoming SSH connections |
| `transport input telnet` | Permit incoming Telnet connections |
| `transport input telnet ssh` | Permit incoming Telnet and SSH connections |
| `transport input all` | Permit all supported incoming transport protocols |
| `transport input none` | Prevent incoming terminal transport connections |

The command replaces the existing list of permitted incoming transport protocols.

For example:

```text
R1(config-line)# transport input telnet
R1(config-line)# transport input ssh
```

The resulting configuration permits only SSH, not both Telnet and SSH.

To permit both protocols, specify them in the same command:

```text
R1(config-line)# transport input telnet ssh
```

The default incoming transport configuration varies between IOS and IOS XE releases and platforms. Administrators should explicitly configure the desired protocols.

### Transport Permissions vs. Server Configuration

Permitting a transport protocol on a VTY line does not necessarily enable the corresponding server.

For example:

```text
R1(config)# line vty 0 15
R1(config-line)# transport input ssh
```

This permits incoming SSH connections to the VTY lines, but the SSH server must also be configured appropriately.

Similarly, enabling the SSH server does not guarantee that SSH connections are permitted on the VTY lines.

Both global protocol configuration and line-level transport configuration must be considered.

SSH server configuration is covered in [SSH Configuration and Verification](../ssh-scp/ssh.md).

## Outgoing Transport Protocols

The `transport output` command specifies which outgoing terminal transport protocols a user can initiate from an EXEC session on a line.

For example:

```text
R1(config)# line vty 0 15
R1(config-line)# transport input ssh
R1(config-line)# transport output telnet
```

In this example, users can connect to R1 using SSH, but can initiate only Telnet terminal connections from R1.

```text
Administrator ---- SSH ----> R1 ---- Telnet ----> R2
```

The incoming and outgoing transport protocols are independent.

To prevent outgoing terminal connections:

```text
R1(config-line)# transport output none
```

This does not prevent the device from generating other network traffic, such as ICMP packets or routing protocol messages.

### Preferred Transport Protocol

The `transport preferred` command specifies the preferred protocol for implicit outgoing terminal connections.

To disable implicit outgoing connections:

```text
R1(config-line)# transport preferred none
```

Explicitly initiated connections remain subject to the `transport output` configuration.

| Command | Purpose |
|---|---|
| `transport input` | Controls incoming terminal connections |
| `transport output` | Controls outgoing terminal connections |
| `transport preferred` | Specifies the preferred protocol for implicit outgoing terminal connections |

## VTY Authentication

Authentication determines whether a user is permitted to access the CLI through a VTY line.

VTY authentication can use:

- A shared line password.
- The local username database.
- AAA authentication method lists.

### Line Password Authentication

Without AAA, the `password` and `login` commands configure traditional line password authentication.

```text
R1(config)# line vty 0 15
R1(config-line)# password CISCO
R1(config-line)# login
R1(config-line)# transport input telnet
```

A user establishing a Telnet connection is prompted for the configured line password.

```text
PC1> telnet 192.0.2.1

User Access Verification

Password:

R1>
```

A username is not required.

If line password authentication is enabled but no line password is configured, IOS XE rejects incoming Telnet access.

For example:

```text
Password required, but none set

[Connection to 192.0.2.1 closed by foreign host]
```

Traditional line password authentication is not sufficient for SSH authentication. SSH requires an authentication mechanism that supports a username, such as local user authentication or an appropriate AAA method list.

### Local User Authentication

The `login local` command enables authentication against the local username database when AAA is disabled.

```text
R1(config)# username ADMIN privilege 15 secret ADMIN-PASSWORD
!
R1(config)# line vty 0 15
R1(config-line)# login local
R1(config-line)# transport input telnet ssh
```

Users must provide valid local user credentials.

The same local user database can be used for both Telnet and SSH authentication.

In this example, the ADMIN user enters privileged EXEC mode directly because the username is configured with privilege level 15.

### AAA Authentication

When AAA is enabled, VTY authentication can be controlled using AAA method lists.

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login REMOTE group tacacs+ local
!
R1(config)# line vty 0 15
R1(config-line)# login authentication REMOTE
R1(config-line)# transport input ssh
```

The REMOTE authentication method list attempts TACACS+ authentication first.

If the TACACS+ method returns an error, such as when no TACACS+ server is reachable, IOS XE attempts local authentication.

If TACACS+ explicitly rejects the user's credentials, IOS XE does not proceed to local authentication.

AAA authentication is covered in detail in [Login and Enable Authentication](../../../device-security/aaa/authentication.md).

### Disabling Authentication

Without AAA, the `no login` command disables line login authentication.

```text
R1(config)# line vty 0 15
R1(config-line)# no login
R1(config-line)# transport input telnet
```

Users can establish Telnet sessions without entering login credentials.

This configuration should not be used in production networks.

The `no login` command does not provide a mechanism for bypassing SSH protocol authentication.

## VTY Authorization and Privilege Levels

Authentication and authorization serve different purposes:

- **Authentication:** Verifies the identity of a user.
- **Authorization:** Determines what an authenticated user is permitted to do.

The default line privilege level is 1.

The `privilege level` command changes the default privilege level associated with a line.

```text
R1(config)# line vty 0 15
R1(config-line)# privilege level 15
```

The effective initial privilege level can also be determined by the user's local account or AAA EXEC authorization.

### AAA EXEC Authorization

AAA EXEC authorization determines whether a user is permitted to start an EXEC session and can assign session attributes such as a privilege level.

For example:

```text
R1(config)# aaa new-model
R1(config)# aaa authorization exec REMOTE group tacacs+ local
!
R1(config)# line vty 0 15
R1(config-line)# authorization exec REMOTE
```

Unlike console authorization, VTY authorization does not require the global `aaa authorization console` command.

AAA command authorization and accounting can also be applied to VTY lines.

For more information, see [4.1.b AAA](../../../device-security/aaa/overview.md).

## Restricting VTY Access

The `access-class` command applies an ACL to incoming or outgoing terminal connections associated with a line.

### Incoming Access Restrictions

The following example permits incoming management connections only from the 192.0.2.0/24 subnet:

```text
R1(config)# ip access-list standard VTY-IN
R1(config-std-nacl)# permit 192.0.2.0 0.0.0.255
!
R1(config)# line vty 0 15
R1(config-line)# access-class VTY-IN in
```

Incoming terminal connections from other source addresses are denied by the ACL's implicit deny.

The ACL applies to incoming terminal connections regardless of which physical interface receives them.

Unlike an interface ACL, a VTY access class does not filter ordinary transit traffic.

### Outgoing Access Restrictions

An outgoing access class can restrict the destinations of terminal connections initiated from an EXEC session.

For example:

```text
R1(config)# ip access-list standard VTY-OUT
R1(config-std-nacl)# permit 198.51.100.0 0.0.0.255
!
R1(config)# line vty 0 15
R1(config-line)# access-class VTY-OUT out
```

Users connected through these VTY lines can initiate outgoing terminal connections only to destinations permitted by the ACL.

VTY access classes, IPv6 access restrictions, and VRF-aware management access are covered in detail in [Session Timeouts and Access Restrictions](access-control.md).

## VTY Session Management

Several commands control the lifetime and behavior of VTY sessions.

### EXEC Timeout

The `exec-timeout` command controls how long an EXEC session can remain inactive before it is terminated.

```text
R1(config)# line vty 0 15
R1(config-line)# exec-timeout 5 30
```

In this example, an inactive EXEC session is terminated after five minutes and 30 seconds.

The default EXEC inactivity timeout is 10 minutes.

To disable the EXEC inactivity timeout:

```text
R1(config-line)# exec-timeout 0 0
```

### Absolute Timeout

The `absolute-timeout` command limits the total duration of a session, regardless of user activity.

```text
R1(config)# line vty 0 15
R1(config-line)# absolute-timeout 60
```

The session is terminated after 60 minutes, even if the user is actively entering commands.

### TCP Keepalives

TCP keepalives help detect connections that are no longer functional, allowing abandoned connections to be cleared.

TCP keepalives are enabled using global configuration commands:

```text
R1(config)# service tcp-keepalives-in
R1(config)# service tcp-keepalives-out
```

| Command | Purpose |
|---|---|
| `service tcp-keepalives-in` | Enables TCP keepalives for incoming connections |
| `service tcp-keepalives-out` | Enables TCP keepalives for outgoing connections |

TCP keepalives differ from inactivity timeouts. An idle but functional TCP connection can respond to keepalives and remain established.

Session management mechanisms are covered in detail in [Session Timeouts and Access Restrictions](access-control.md).

## VTY Logging

Unlike the console, VTY sessions do not display unsolicited system logging messages by default.

The `terminal monitor` EXEC command enables the display of system logging messages in the current terminal session.

```text
R1# terminal monitor
```

To disable the display of these messages in the current session:

```text
R1# terminal no monitor
```

The global `logging monitor` command controls the severity threshold for monitor logging.

For example:

```text
R1(config)# logging monitor informational
```

Monitor logging includes messages with severity levels 0 through 6.

The `logging synchronous` line configuration command prevents unsolicited messages from visually disrupting CLI input.

```text
R1(config)# line vty 0 15
R1(config-line)# logging synchronous
```

Note the distinction between the following commands:

| Command | Purpose |
|---|---|
| `terminal monitor` | Enables monitor logging for the current terminal session |
| `logging monitor` | Controls the global monitor logging configuration |
| `logging synchronous` | Redisplays the prompt and partially entered command after unsolicited messages |

Logging synchronous does not enable monitor logging.

## Outgoing Session Management

An administrator connected to a device through a VTY line can initiate additional terminal connections from that device.

For example:

```text
Administrator ---- SSH ---- R1 ---- Telnet ---- R2
```

The administrator can suspend the outgoing Telnet session to R2 and return to R1's EXEC prompt without terminating either connection.

The default escape sequence is:

```text
Ctrl+Shift+6, followed by X
```

This is particularly useful when managing multiple devices through an intermediate router.

### Displaying Outgoing Sessions

The `show sessions` command displays outgoing terminal connections associated with the current EXEC session.

```text
R1# show sessions
Conn Host                Address             Byte  Idle Conn Name
*  1 192.0.2.2           192.0.2.2              0     0 192.0.2.2
   2 192.0.2.3           192.0.2.3              0     1 192.0.2.3
```

The connection numbers identify individual outgoing sessions.

The asterisk identifies the current connection.

### Resuming an Outgoing Session

The `resume` command resumes a suspended outgoing session.

```text
R1# resume 1
```

Pressing Enter at the EXEC prompt can also resume the current outgoing session when one exists.

### Disconnecting an Outgoing Session

The `disconnect` command terminates an outgoing terminal connection.

```text
R1# disconnect 2
```

This terminates outgoing connection 2 without terminating the administrator's incoming connection to R1.

### Incoming vs. Outgoing Connections

Consider the following topology:

```text
PC1 ---- SSH ----> R1 ---- Telnet ----> R2
                    |
                    +----- SSH -------> R3
```

From PC1's EXEC session on R1:

- `show users` displays active user sessions on R1, including PC1's incoming SSH session.
- `show sessions` displays outgoing terminal sessions initiated from the current EXEC session, including the connections to R2 and R3.
- `disconnect` terminates a specified outgoing session.
- `clear line` clears a specified local line and its associated session.

## Automatically Executing Commands

The `autocommand` line configuration command specifies a command to execute automatically when an EXEC session starts on a line.

For example:

```text
R1(config)# line vty 10 15
R1(config-line)# autocommand show ip interface brief
```

When a user accesses one of these VTY lines, IOS XE automatically executes `show ip interface brief`.

The connection is normally terminated when the specified command completes.

The `autocommand` feature can also be used to automatically initiate a terminal connection to another device.

```text
R1(config)# line vty 10 15
R1(config-line)# autocommand telnet 192.0.2.2
```

This can be combined with rotary groups to provide dedicated access to specific devices or functions.

The `no autocommand` command removes the automatic command configuration.

## Complete Configuration Example

The following configuration provides SSH-only management access from a trusted management subnet.

Assume the SSH server has already been enabled.

```text
R1(config)# username ADMIN privilege 15 secret ADMIN-PASSWORD
!
R1(config)# ip access-list standard VTY-IN
R1(config-std-nacl)# permit 192.0.2.0 0.0.0.255
!
R1(config)# service tcp-keepalives-in
R1(config)# service tcp-keepalives-out
!
R1(config)# line vty 0 15
R1(config-line)# login local
R1(config-line)# transport input ssh
R1(config-line)# transport output none
R1(config-line)# transport preferred none
R1(config-line)# access-class VTY-IN in
R1(config-line)# exec-timeout 5 0
R1(config-line)# logging synchronous
```

Assuming AAA is disabled, this configuration provides the following behavior:

- Only incoming SSH connections are permitted.
- Incoming connections must originate from the 192.0.2.0/24 subnet.
- Users authenticate against the local username database.
- The ADMIN user enters privileged EXEC mode directly.
- Inactive EXEC sessions are terminated after five minutes.
- Outgoing terminal connections are prohibited.
- Synchronous logging is enabled.
- TCP keepalives help detect abandoned connections.

The IPv4 access class in this example does not provide equivalent IPv6 access restrictions. IPv6 management access must be secured separately where applicable.

## Verification

### show line

The `show line` command displays information about the device's available lines.

```text
R1# show line
```

Detailed information about a particular VTY line can be displayed using its relative line number:

```text
R1# show line vty 0
```

This includes information about:

- Line status and capabilities.
- Permitted transport protocols.
- Configured timeouts.
- Access classes.
- Rotary group membership.
- Line usage.

### show users

The `show users` command displays active user sessions.

For example:

```text
R1# show users
    Line       User       Host(s)              Idle       Location
*  0 con 0                idle                 00:00:00
   2 vty 0     ADMIN      idle                 00:01:15   192.0.2.100
   3 vty 1     ADMIN2     idle                 00:00:30   192.0.2.200
```

In this example:

- The current session is associated with console line 0.
- ADMIN is connected through VTY line 0.
- ADMIN2 is connected through VTY line 1.
- The Location column displays the remote source addresses of the VTY sessions.

### show ssh

The `show ssh` command displays SSH connection information.

```text
R1# show ssh
```

This command provides SSH-specific information, such as the SSH protocol version, encryption algorithms, authentication state, and username.

Unlike `show users`, it does not display ordinary Telnet or console sessions.

### show running-config

The following command displays the VTY configuration:

```text
R1# show running-config | section ^line vty
```

To include commands configured with their default values:

```text
R1# show running-config all | section ^line vty
```

These commands are particularly useful when checking for inconsistent configurations across different VTY line ranges.

## Clearing VTY Sessions

The `clear line` command clears a specified line.

For example:

```text
R1# clear line vty 0
```

Alternatively, the absolute line number can be specified.

Assuming VTY line 0 corresponds to absolute line 2:

```text
R1# clear line 2
```

Clearing an occupied VTY line terminates its associated session and makes the line available for another connection.

This can be useful when:

- All VTY lines are occupied.
- A management session has become unresponsive.
- An administrator must terminate an unauthorized session.
- A disconnected client has left behind an established session.

Use `show users` to identify the correct line before clearing it.

## Troubleshooting Remote Access

When troubleshooting failed Telnet or SSH connections, verify each component of the management access configuration.

| Potential Problem | Verification |
|---|---|
| No IP connectivity | Routing table, ping, traceroute |
| Management protocol disabled | Global protocol configuration |
| Transport protocol not permitted | `show line`, VTY configuration |
| All eligible VTY lines occupied | `show users`, `show line` |
| Access class denies the connection | `show access-lists`, VTY configuration |
| Incorrect login authentication | VTY and AAA configuration |
| AAA server unreachable | AAA server status and connectivity |
| EXEC authorization failure | AAA EXEC authorization configuration |
| EXEC process disabled | Check for `no exec` |
| SSH server configuration problem | `show ip ssh`, `show ssh` |
| VRF-related access restriction | VRF-aware access-class configuration |
| Inconsistent VTY configurations | `show running-config all` |

A successful TCP connection does not necessarily indicate successful CLI access.

For example, an incoming connection can reach the device but fail during authentication or EXEC authorization.

Similarly, a successful login does not necessarily indicate that the user has sufficient privileges to execute the required commands.

## Verification Command Summary

| Command | Purpose |
|---|---|
| `show line` | Display available lines and line information |
| `show line vty <number>` | Display detailed information about a VTY line |
| `show users` | Display active user sessions |
| `show sessions` | Display outgoing terminal connections for the current EXEC session |
| `show terminal` | Display current terminal settings |
| `show ssh` | Display SSH connection information |
| `show ip ssh` | Display SSH server status and settings |
| `show access-lists` | Display ACL configuration and match counters |
| `show running-config \| section ^line vty` | Display VTY configuration |
| `show running-config all \| section ^line vty` | Display VTY configuration, including defaults |
| `clear line vty <number>` | Clear a specified VTY line |
| `resume <number>` | Resume a suspended outgoing session |
| `disconnect <number>` | Terminate an outgoing session |

## Key Concepts

- VTY lines provide remote CLI access through protocols such as Telnet and SSH.
- Each incoming interactive CLI session occupies one VTY line.
- Telnet and SSH share the same pool of VTY lines.
- VTY lines are independent of physical network interfaces.
- Absolute line numbers and relative VTY line numbers can differ.
- `transport input` controls incoming terminal transport protocols.
- `transport output` controls outgoing terminal transport protocols.
- `transport preferred none` disables implicit outgoing terminal connections.
- SSH requires a username-capable authentication mechanism.
- `access-class ... in` restricts incoming terminal connections.
- `access-class ... out` restricts outgoing terminal connections.
- `exec-timeout` controls EXEC inactivity timeouts.
- `absolute-timeout` limits the total duration of a session.
- TCP keepalives help detect nonfunctional TCP connections.
- `terminal monitor` enables monitor logging for the current session.
- `autocommand` automatically executes a command when an EXEC session starts.
- `show users` displays active user sessions on the local device.
- `show sessions` displays outgoing terminal connections associated with the current EXEC session.
- `clear line` clears a local line and its associated session.
- Security restrictions should be applied consistently to all relevant VTY lines.
