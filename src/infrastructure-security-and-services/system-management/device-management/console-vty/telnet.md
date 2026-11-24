# Telnet Configuration and Operation

**Telnet** is an application-layer protocol that provides remote terminal access to a device over a TCP connection.

Cisco IOS XE can operate as both a Telnet server and a Telnet client:

- **Telnet server:** Accepts incoming Telnet connections from remote devices.
- **Telnet client:** Initiates outgoing Telnet connections to remote devices.

Telnet uses TCP port 23 by default.

Unlike SSH, Telnet does not provide encryption, cryptographic server authentication, or cryptographic integrity protection. Usernames, passwords, commands, and command output are transmitted in cleartext.

Although SSH is preferred in production networks, Telnet is useful for understanding remote access, VTY line configuration, terminal servers, and TCP connectivity testing.

## Telnet Operation

Telnet provides bidirectional communication between a client and a server.

```text
 Telnet Client                         Telnet Server
      PC1                                    R1
       |                                      |
       |------- TCP SYN (Dst Port 23) ------->|
       |<------------ TCP SYN-ACK ------------|
       |--------------- TCP ACK ------------->|
       |                                      |
       |<------ Telnet Option Negotiation --->|
       |                                      |
       |<---------- Login Prompt -------------|
       |------------- Credentials ----------->|
       |                                      |
       |<---------- CLI Interaction --------->|
```

A typical Telnet session involves:

1. Establishing a TCP connection.
2. Negotiating Telnet options.
3. Performing user authentication, if required.
4. Starting an interactive EXEC session.
5. Exchanging commands and command output.
6. Terminating the session and closing the TCP connection.

Telnet itself does not define the authentication method used by the server. On IOS XE, authentication is determined by the line and AAA configuration.

### Network Virtual Terminal

Telnet uses the concept of a **Network Virtual Terminal (NVT)** to provide a standardized representation of terminal communication.

The NVT allows devices with different native terminal characteristics to communicate using a common format.

Telnet clients and servers can negotiate additional options to modify the behavior of the session.

### Telnet Option Negotiation

Telnet uses control commands embedded in the TCP data stream to negotiate session characteristics.

These commands are preceded by the **Interpret As Command (IAC)** byte, which has a value of 255 (0xFF).

The four primary option negotiation commands are:

| Command | Value | Purpose |
|---|---:|---|
| WILL | 251 | Indicates willingness to perform an option |
| WONT | 252 | Refuses to perform an option |
| DO | 253 | Requests that the peer perform an option |
| DONT | 254 | Requests that the peer not perform an option |

Common Telnet options include:

- Echo.
- Suppress Go Ahead (SGA).
- Terminal type.
- Negotiate About Window Size (NAWS).
- Binary transmission.

Option negotiation allows the client and server to agree on terminal behavior.

## Telnet Server Configuration

Incoming interactive Telnet connections use VTY lines.

Unlike SSH, basic Telnet server operation does not require cryptographic keys or a separate global server-enabling command.

The VTY lines must permit Telnet as an incoming transport protocol and have an appropriate authentication configuration.

### Line Password Authentication

When AAA is disabled, a shared line password can be used to authenticate Telnet users.

```text
R1(config)# line vty 0 15
R1(config-line)# password CISCO
R1(config-line)# login
R1(config-line)# transport input telnet
```

The `password` command configures the shared line password.

The `login` command enables line password authentication.

The `transport input telnet` command permits incoming Telnet connections.

When a client connects to R1, it is prompted for the line password.

```text
R2# telnet 192.0.2.1
Trying 192.0.2.1 ... Open

User Access Verification

Password:

R1>
```

A username is not required.

If `login` is configured but no line password is configured, incoming Telnet access is rejected.

```text
Password required, but none set
```

### Local User Authentication

The `login local` command enables authentication against the local username database when AAA is disabled.

```text
R1(config)# username ADMIN privilege 15 secret ADMIN-PASSWORD
!
R1(config)# line vty 0 15
R1(config-line)# login local
R1(config-line)# transport input telnet
```

The client is prompted for a username and password.

```text
R2# telnet 192.0.2.1
Trying 192.0.2.1 ... Open

User Access Verification

Username: ADMIN
Password:

R1#
```

Because the ADMIN user is configured with privilege level 15, the user enters privileged EXEC mode directly.

Although the `secret` keyword protects the password stored in the device configuration, it does not encrypt the password transmitted over the Telnet connection.

### AAA Authentication

When AAA is enabled, Telnet authentication can be controlled using AAA method lists.

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login TELNET local
!
R1(config)# line vty 0 15
R1(config-line)# login authentication TELNET
R1(config-line)# transport input telnet
```

The VTY lines use the AAA login authentication method list named TELNET.

AAA authentication can also use external TACACS+ or RADIUS servers.

Using an external AAA server does not encrypt the Telnet connection between the client and the router.

For more information, see [4.1.b AAA](../../../device-security/aaa/overview.md).

### Disabling Authentication

When AAA is disabled, the `no login` command disables login authentication on a line.

```text
R1(config)# line vty 0 15
R1(config-line)# no login
R1(config-line)# transport input telnet
```

A client can establish a Telnet session without entering credentials.

```text
R2# telnet 192.0.2.1
Trying 192.0.2.1 ... Open

R1>
```

Disabling login authentication does not necessarily grant privileged EXEC access.

This configuration is useful for certain controlled lab environments but should not be used on production devices.

### Permitting Both Telnet and SSH

Telnet and SSH can be permitted on the same VTY lines.

```text
R1(config)# line vty 0 15
R1(config-line)# login local
R1(config-line)# transport input telnet ssh
```

Both protocols use the same pool of VTY lines.

The SSH server must also be configured independently.

### Disabling Incoming Telnet Access

To permit SSH but prohibit Telnet:

```text
R1(config)# line vty 0 15
R1(config-line)# transport input ssh
```

To prohibit all incoming terminal transport connections:

```text
R1(config)# line vty 0 15
R1(config-line)# transport input none
```

These restrictions should be applied to all relevant VTY lines.

Other terminal access mechanisms, such as reverse Telnet on asynchronous lines, must be considered separately where applicable.

## Telnet Client Operation

Cisco IOS XE includes a Telnet client that can initiate TCP connections to remote devices.

### Initiating a Telnet Connection

The basic syntax is:

```text
telnet <ip-address | hostname> [port]
```

For example:

```text
R1# telnet 192.0.2.2
Trying 192.0.2.2 ... Open
```

Because no destination port is specified, TCP port 23 is used.

A hostname can also be specified:

```text
R1# telnet R2
```

The hostname must be resolvable using a local hostname mapping or DNS.

For example, a static hostname mapping can be configured as follows:

```text
R1(config)# ip host R2 192.0.2.2
```

### Implicit Telnet Connections

IOS XE can interpret a hostname entered at the EXEC prompt as a request to establish an outgoing terminal connection.

For example:

```text
R1# R2
```

If Telnet is the preferred transport protocol and R2 can be resolved, IOS XE attempts to establish a Telnet connection to R2.

This behavior is controlled by the `transport preferred` line configuration command.

```text
R1(config)# line console 0
R1(config-line)# transport preferred telnet
```

To disable implicit outgoing terminal connections:

```text
R1(config-line)# transport preferred none
```

Explicit Telnet connections using the `telnet` command remain permitted, provided Telnet is allowed by `transport output`.

### Outgoing Transport Restrictions

The `transport output` command controls whether users can initiate Telnet connections from an EXEC session on a line.

For example:

```text
R1(config)# line vty 0 15
R1(config-line)# transport input ssh
R1(config-line)# transport output telnet
```

Users must use SSH to access R1, but they can initiate Telnet connections from R1.

```text
Administrator ---- SSH ----> R1 ---- Telnet ----> R2
```

To prohibit outgoing Telnet connections while permitting SSH:

```text
R1(config-line)# transport output ssh
```

To prohibit all outgoing terminal transport connections:

```text
R1(config-line)# transport output none
```

Incoming and outgoing transport restrictions are independent.

### Specifying the Source Interface

By default, IOS XE uses the IP address of the outgoing interface as the source IP address of an outgoing Telnet connection.

The `ip telnet source-interface` global configuration command specifies a preferred source interface for outgoing Telnet connections.

```text
R1(config)# ip telnet source-interface Loopback0
```

For example, if Loopback0 has the IP address 1.1.1.1, outgoing Telnet connections use 1.1.1.1 as their source IP address.

If the specified interface is not up, IOS XE falls back to its default source address selection behavior.

A source interface can also be specified for an individual Telnet connection:

```text
R1# telnet 192.0.2.2 /source-interface Loopback0
```

This is useful for:

- Testing management access restrictions.
- Using a consistent source IP address for management connections.
- Testing connectivity from a specific source address.
- Satisfying source-based filtering on remote devices.

### VRF-Aware Telnet

An outgoing Telnet connection can use a specific VRF's routing table.

For example:

```text
R1# telnet 192.0.2.2 /vrf MGMT
```

The connection uses the MGMT VRF rather than the global routing table.

This is particularly useful when management connectivity is provided through a dedicated management VRF.

Incoming VRF-aware management access is covered in [Session Timeouts and Access Restrictions](access-control.md).

## Telnet to Nonstandard TCP Ports

Although Telnet uses TCP port 23 by default, the client can establish TCP connections to other destination ports.

For example:

```text
R1# telnet 192.0.2.2 179
```

This attempts to establish a TCP connection to port 179, the well-known BGP port.

Other examples include:

```text
R1# telnet 192.0.2.2 22
R1# telnet 192.0.2.2 80
R1# telnet 192.0.2.2 443
R1# telnet 192.0.2.2 830
```

| TCP Port | Associated Protocol |
|---:|---|
| 22 | SSH |
| 23 | Telnet |
| 80 | HTTP |
| 179 | BGP |
| 443 | HTTPS |
| 830 | NETCONF over SSH |

The Telnet client does not implement these other application-layer protocols simply because it connects to their associated TCP ports.

For example, connecting to TCP port 179 does not establish a BGP peering.

### Raw TCP Stream Mode

The `/stream` option disables Telnet protocol control sequences and allows the connection to operate as a raw TCP stream.

```text
R1# telnet 192.0.2.2 179 /stream
```

This is useful when connecting to a TCP service that does not implement the Telnet protocol.

The `/nostream` option disables stream mode.

## Telnet as a Troubleshooting Tool

Telnet can be used to test whether a TCP connection can be established to a remote device.

For example:

```text
R1# telnet 192.0.2.2 179
Trying 192.0.2.2, 179 ... Open
```

The `Open` message indicates that the TCP connection was successfully established.

This confirms TCP connectivity to the specified destination address and port, but it does not confirm that the application-layer protocol is operating correctly.

### Connection Refused

```text
R1# telnet 192.0.2.2 179
Trying 192.0.2.2, 179 ...
% Connection refused by remote host
```

A connection refusal typically indicates that a TCP RST was received in response to the connection attempt.

Possible causes include:

- No application is listening on the destination port.
- The remote device explicitly rejects the connection.
- An intermediate security device actively rejects the connection.

### Connection Timeout

```text
R1# telnet 192.0.2.2 179
Trying 192.0.2.2, 179 ...
% Connection timed out; remote host not responding
```

A timeout indicates that the TCP connection could not be established within the permitted time.

Possible causes include:

- Missing forward or return routing information.
- An ACL silently dropping packets.
- A firewall silently dropping packets.
- An unreachable destination.
- Packet loss.

### Interpreting Telnet Connection Results

| Result | Interpretation |
|---|---|
| Open | TCP connection established successfully |
| Connection refused | Connection actively rejected |
| Connection timed out | No successful TCP handshake before the timeout |
| Open, followed by immediate closure | TCP connection established, but subsequently terminated |

Telnet provides more specific information about TCP connectivity than ping.

A successful ping confirms ICMP connectivity, but it does not confirm that a specific TCP port is reachable.

For example, an ACL can permit ICMP echo requests while denying TCP connections to port 22.

## Telnet Session Management

IOS XE allows users to establish multiple outgoing terminal sessions from a single EXEC session.

Consider the following topology:

```text
PC1 ---- SSH ----> R1 ---- Telnet ----> R2
                    |
                    +---- Telnet ----> R3
```

PC1 connects to R1 and initiates Telnet connections to R2 and R3.

The administrator can suspend, resume, and disconnect the outgoing sessions independently.

### Suspending a Telnet Session

The default Cisco escape sequence is:

```text
Ctrl+Shift+6, followed by X
```

This suspends the outgoing Telnet session and returns the user to the originating device's EXEC prompt.

For example:

```text
R1# telnet 192.0.2.2
Trying 192.0.2.2 ... Open

R2#
```

After entering the escape sequence:

```text
R1#
```

The Telnet connection to R2 remains established.

The user can now execute commands on R1 or initiate another outgoing connection.

### Displaying Suspended Sessions

The `show sessions` command displays outgoing terminal connections associated with the current EXEC session.

```text
R1# show sessions
Conn Host                Address             Byte  Idle Conn Name
   1 192.0.2.2           192.0.2.2              0     1 192.0.2.2
*  2 192.0.2.3           192.0.2.3              0     0 192.0.2.3
```

Each outgoing connection is assigned a connection number.

The asterisk identifies the current connection.

### Resuming a Session

The `resume` command resumes a suspended connection.

```text
R1# resume 1
```

Alternatively, the connection number can be entered directly:

```text
R1# 1
```

Pressing Enter at an empty EXEC prompt can also resume the current outgoing connection.

### Renaming a Session

The `name-connection` command assigns a descriptive name to an outgoing terminal connection.

```text
R1# name-connection 1 R2
R1# name-connection 2 R3
```

The assigned names are displayed in the `show sessions` output and can be used when resuming connections.

```text
R1# resume R2
```

### Disconnecting a Session

The `disconnect` command terminates an outgoing terminal connection.

```text
R1# disconnect 1
```

This terminates connection 1 without terminating the current EXEC session on R1.

An outgoing Telnet session can also be terminated normally by entering `exit` or `logout` on the remote device.

### Changing the Escape Character

The `escape-character` line configuration command modifies the escape character associated with a line.

The default escape character is Ctrl+^ (Ctrl+Shift+6 on a typical US keyboard), corresponding to ASCII decimal value 30.

For example:

```text
R1(config)# line vty 0 15
R1(config-line)# escape-character 30
```

The escape character can also be modified for the current terminal session:

```text
R1# terminal escape-character 30
```

The escape character is followed by an additional character to perform a particular terminal operation.

For example, following the default escape character with X suspends the outgoing connection.

## Reverse Telnet

**Reverse Telnet** allows a Telnet client to access a physical asynchronous serial line on a terminal server.

Unlike a normal Telnet connection, reverse Telnet does not provide access to the terminal server's own EXEC process.

Instead, the terminal server forwards communication between the TCP connection and the selected serial line.

```text
Administrator
      |
      | Telnet (TCP 2001)
      |
+---------------------+
|   Terminal Server   |
|                     |
| Async Line 1        |----- Console Cable ----- R1
| Async Line 2        |----- Console Cable ----- R2
| Async Line 3        |----- Console Cable ----- R3
+---------------------+
```

The administrator can remotely access the console of R1, R2, or R3 without requiring IP connectivity to those devices.

Only the terminal server requires network connectivity.

### Reverse Telnet Port Numbering

The standard reverse Telnet TCP port is calculated as follows:

```text
TCP port = 2000 + absolute line number
```

For example:

| Absolute Line Number | Reverse Telnet TCP Port |
|---:|---:|
| 1 | 2001 |
| 2 | 2002 |
| 3 | 2003 |
| 33 | 2033 |

The absolute line number must be used in the calculation.

Depending on the platform, the absolute line number might differ from the physical asynchronous port number.

Use `show line` to determine the correct line number.

### Reverse Telnet Configuration

Assume that absolute lines 1 through 3 correspond to asynchronous serial interfaces connected to the console ports of R1, R2, and R3.

Configure the terminal server as follows:

```text
TS(config)# line 1 3
TS(config-line)# no exec
TS(config-line)# transport input telnet
TS(config-line)# exec-timeout 0 0
```

The commands perform the following functions:

- `no exec` prevents the terminal server from starting its own EXEC process on the asynchronous lines.
- `transport input telnet` permits incoming Telnet connections to the lines.
- `exec-timeout 0 0` disables the EXEC inactivity timeout. However, because EXEC is disabled on these lines, it does not control the inactivity timeout of the reverse Telnet connection.

Appropriate authentication, access restrictions, and connection timeouts should also be configured as required.

Assuming the terminal server has the IP address 192.0.2.100, an administrator can access R1 using:

```text
PC1> telnet 192.0.2.100 2001
```

To access R2:

```text
PC1> telnet 192.0.2.100 2002
```

To access R3:

```text
PC1> telnet 192.0.2.100 2003
```

### Hostname Mappings for Reverse Telnet

The `ip host` command can associate a hostname with an IP address and a nonstandard Telnet TCP port.

For example:

```text
TS(config)# ip host R1 2001 192.0.2.100
TS(config)# ip host R2 2002 192.0.2.100
TS(config)# ip host R3 2003 192.0.2.100
```

The syntax is:

```text
ip host <hostname> [tcp-port] <ip-address>
```

An administrator connected to the terminal server can now initiate a reverse Telnet connection using the configured hostname.

```text
TS# telnet R1
Trying R1 (192.0.2.100, 2001) ... Open

R1#
```

This avoids having to remember the individual TCP port numbers associated with each asynchronous line.

If implicit Telnet connections are enabled, entering the hostname alone can also initiate the connection.

```text
TS# R1
```

### Clearing a Reverse Telnet Connection

An asynchronous line can remain occupied by an existing reverse Telnet connection.

Use `show line` to identify the line and `clear line` to clear it.

```text
TS# clear line 1
```

This terminates the connection occupying absolute line 1 and makes the line available for another connection.

## Telnet Rotary Groups

Reverse Telnet can target an individual asynchronous line using the corresponding TCP port.

Alternatively, Telnet rotary groups allow incoming connections to target a group of lines.

The standard Telnet rotary group TCP port is calculated as follows:

```text
TCP port = 3000 + rotary group number
```

For example:

```text
R1(config)# line vty 5 9
R1(config-line)# rotary 1
R1(config-line)# transport input telnet
```

An incoming Telnet connection to TCP port 3001 is directed to an available eligible line in rotary group 1.

### Telnet Port Number Summary

| Connection Type | TCP Port |
|---|---|
| Standard Telnet | 23 |
| Reverse Telnet to a specific line | 2000 + absolute line number |
| Telnet to a rotary group | 3000 + rotary group number |

Rotary groups, including their use with VTY lines and SSH, are covered in detail in [Rotary Groups and Custom TCP Ports](rotary-groups.md).

## Telnet Security

Telnet does not encrypt session traffic.

An attacker capable of capturing Telnet traffic can potentially recover:

- Usernames.
- Passwords.
- Enable passwords.
- Configuration commands.
- Configuration output.
- Other sensitive information displayed during the session.

The following mechanisms improve access control but do not encrypt Telnet traffic:

- Local user authentication.
- TACACS+ authentication.
- RADIUS authentication.
- VTY access classes.
- Management VRFs.
- Custom TCP ports.
- Rotary groups.

SSH should be used instead of Telnet when confidentiality, server authentication, and cryptographic integrity protection are required.

## Verification

### show users

Displays active user sessions on the local device.

```text
R1# show users
```

This can be used to identify incoming Telnet sessions and their associated VTY lines.

### show sessions

Displays outgoing terminal connections associated with the current EXEC session.

```text
R1# show sessions
```

This can be used to identify active and suspended outgoing Telnet connections.

### show tcp brief all

Displays TCP connections, including listening sockets and established connections.

```text
R1# show tcp brief all
```

For example, the following simplified output shows a Telnet listening socket and an established incoming Telnet connection:

```text
TCB       Local Address       Foreign Address     (state)
XXXXXXXX  *.23                *.*                 LISTEN
XXXXXXXX  192.0.2.1.23        192.0.2.2.49152      ESTAB
```

The established connection uses:

- Local IP address: 192.0.2.1.
- Local TCP port: 23.
- Remote IP address: 192.0.2.2.
- Remote TCP port: 49152.

This command can also be used to identify connections using reverse Telnet ports and rotary group ports.

### show hosts

Displays configured and cached hostname mappings.

```text
R1# show hosts
```

This is useful for verifying hostname mappings configured using `ip host`, including mappings with nonstandard Telnet TCP ports.

### Verification Command Summary

| Command | Purpose |
|---|---|
| `show users` | Display active user sessions on the local device |
| `show sessions` | Display outgoing terminal connections |
| `show line` | Display line status and configuration information |
| `show tcp brief all` | Display TCP connections and listening sockets |
| `show hosts` | Display hostname mappings |
| `show running-config \| section ^line vty` | Display VTY configuration |
| `resume <connection>` | Resume a suspended connection |
| `disconnect <connection>` | Terminate an outgoing connection |
| `name-connection <number> <name>` | Assign a name to an outgoing connection |
| `clear line <number>` | Clear a specified local line |

## Troubleshooting Telnet

| Symptom | Potential Cause |
|---|---|
| Connection times out | Routing problem, ACL, firewall, or unreachable destination |
| Connection refused | No listening service or active connection rejection |
| Password required, but none set | Line password authentication enabled without a configured password |
| Authentication failure | Incorrect credentials or AAA configuration |
| Connection opens but closes immediately | Authentication, authorization, or line configuration problem |
| Unable to initiate outgoing Telnet | `transport output` restriction |
| Connection denied from a particular source | Incoming access class |
| Connection denied to a particular destination | Outgoing access class |
| Connection denied through a VRF | VRF-aware access restriction |
| Unable to establish additional sessions | All eligible VTY lines occupied |
| Reverse Telnet connection fails | Incorrect absolute line number, occupied line, or transport restriction |
| Reverse Telnet connects to the terminal server's CLI | EXEC process not disabled on the asynchronous line |
| Connection to a rotary group fails | Incorrect TCP port, rotary group membership, or line availability |

## Key Concepts

- Telnet provides unencrypted remote terminal access using TCP port 23 by default.
- Cisco IOS XE can operate as both a Telnet client and server.
- Incoming interactive Telnet connections occupy VTY lines.
- Telnet authentication is controlled by line configuration and AAA.
- `transport input telnet` permits incoming Telnet connections.
- `transport output telnet` permits outgoing Telnet connections.
- `transport preferred none` disables implicit outgoing terminal connections.
- `ip telnet source-interface` specifies a preferred source interface for outgoing Telnet connections.
- The `/source-interface` option specifies a source interface for an individual Telnet connection.
- The `/vrf` option selects a VRF for an outgoing Telnet connection.
- The `/stream` option enables raw TCP stream operation without Telnet control sequences.
- Telnet can test TCP connectivity to nonstandard destination ports.
- A successful TCP connection does not guarantee correct application-layer operation.
- Ctrl+Shift+6 followed by X suspends an outgoing Telnet session.
- `show users` displays local user sessions, whereas `show sessions` displays outgoing terminal connections.
- Reverse Telnet provides remote access to asynchronous serial lines.
- Reverse Telnet uses TCP port 2000 + absolute line number.
- Telnet rotary groups use TCP port 3000 + rotary group number.
- `no exec` prevents the terminal server from starting its own EXEC process on an asynchronous line.
- SSH is preferred over Telnet for secure remote device administration.

## References

- [RFC 854 - Telnet Protocol Specification](https://www.rfc-editor.org/rfc/rfc854)
- [RFC 855 - Telnet Option Specifications](https://www.rfc-editor.org/rfc/rfc855)
- [Cisco IOS Configuration Fundamentals Command Reference](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/fundamentals/command/cf_command_ref/F_through_K.html)
- [Cisco IOS Security Command Reference - Telnet](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/security/s1/sec-s1-cr-book/sec-cr-t1.html)
