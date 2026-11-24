# 4.3.a (i) Console and VTY

Cisco IOS XE provides CLI access through **lines**. Lines represent the physical or logical communication channels through which users interact with the device's CLI.

The two primary line types used for device administration are:

- **Console (CON):** Provides local CLI access, typically through a physical console port.
- **Virtual Terminal (VTY):** Provides remote CLI access through network protocols such as Telnet and SSH.

Line configuration determines how users access the device, how they are authenticated, and how their sessions behave.

## Line Types

Cisco IOS XE supports several types of lines.

| Line Type | Description | Configuration Command |
|---|---|---|
| Console (CON) | Local access through the console port | `line console 0` |
| Auxiliary (AUX) | Serial access, traditionally through an external modem | `line aux 0` |
| Virtual Terminal (VTY) | Remote access through Telnet or SSH | `line vty 0 4` |
| TTY | Asynchronous serial access | `line tty <number>` |

The available line types and number of lines depend on the device platform.

Most Cisco IOS XE devices have a single console line, numbered 0, and multiple VTY lines.

### Physical and Virtual Lines

Console, AUX, and asynchronous TTY lines are associated with physical communication interfaces.

VTY lines are virtual resources. They are not associated with specific physical interfaces.

For example, a router with four GigabitEthernet interfaces and 16 VTY lines can potentially support 16 simultaneous remote CLI sessions, regardless of which interfaces receive the connections.

```text
                              R1
                     +-------------------+
                     |                   |
Console Cable ------>| Console Line 0    |
                     |                   |
SSH Session 1 ------>| VTY Line 0        |
                     |                   |
SSH Session 2 ------>| VTY Line 1        |
                     |                   |
                     | VTY Line 2        |
                     | ...               |
                     | VTY Line 15       |
                     |                   |
                     +-------------------+
```

Each incoming remote CLI session occupies an available VTY line for the duration of the session.

The VTY line is released when the session terminates.

## Console Access

Console access allows administrators to interact with a device without requiring IP connectivity to the device.

It is commonly used for:

- Initial device configuration.
- Password recovery.
- Boot process monitoring.
- ROMMON access.
- Troubleshooting management connectivity problems.
- Recovering from configuration errors that prevent remote access.

Console connections typically use a serial or USB console interface.

A console server can also provide remote access to a device's console port. In this case, the administrator establishes a network connection to the console server, which communicates with the managed device through a physical console connection.

```text
Administrator
      |
      | SSH
      |
Console Server
      |
      | Serial Console Cable
      |
      R1
```

R1 does not require functional IP connectivity for the administrator to access its console through the console server.

This is a common method of providing out-of-band management access.

## VTY Access

VTY lines provide remote access to the device's CLI.

Remote CLI access typically uses one of the following protocols:

| Protocol | Default TCP Port | Encryption |
|---|---:|---|
| Telnet | 23 | No |
| SSH | 22 | Yes |

Both Telnet and SSH sessions use VTY lines on the destination device.

By default, incoming connections to the standard management ports are generally assigned to the first eligible available VTY line.

For example, assume all VTY lines are available and configured identically:

```text
Connection 1 --> VTY 0
Connection 2 --> VTY 1
Connection 3 --> VTY 2
```

The assignment of incoming connections can also be influenced by rotary group configuration.

If no eligible VTY line is available, a new remote CLI session cannot be established.

The number of VTY lines supported by a device therefore limits the number of simultaneous incoming remote CLI sessions.

## Line Configuration Mode

Lines are configured in **line configuration mode**.

The `line` command specifies an individual line or a range of lines.

### Configuring the Console Line

```text
R1(config)# line console 0
R1(config-line)#
```

Commands entered in this mode apply to console line 0.

### Configuring an Individual VTY Line

```text
R1(config)# line vty 0
R1(config-line)#
```

Commands entered in this mode apply only to VTY line 0.

### Configuring a Range of VTY Lines

```text
R1(config)# line vty 0 15
R1(config-line)#
```

Commands entered in this mode apply to VTY lines 0 through 15, inclusive.

The first and last line numbers are specified using VTY-relative numbering, rather than the absolute line numbers displayed by some verification commands.

### Separate VTY Line Configurations

Different ranges of VTY lines can have different configurations.

For example:

```text
R1(config)# line vty 0 4
R1(config-line)# transport input ssh
R1(config-line)# exec-timeout 10 0
!
R1(config)# line vty 5 15
R1(config-line)# transport input ssh
R1(config-line)# exec-timeout 5 0
```

In this example:

- VTY lines 0 through 4 permit SSH and have a 10-minute EXEC inactivity timeout.
- VTY lines 5 through 15 permit SSH and have a 5-minute EXEC inactivity timeout.

Administrators should ensure that security restrictions are applied to all relevant VTY lines.

For example, restricting access to VTY lines 0 through 4 does not necessarily restrict access to VTY lines 5 through 15.

## Line Configuration Functions

Line configuration controls several aspects of management access.

| Function | Example Command |
|---|---|
| Line password | `password CISCO` |
| Line password authentication | `login` |
| Local user authentication | `login local` |
| AAA authentication method list | `login authentication ADMIN` |
| Incoming transport protocols | `transport input ssh` |
| Outgoing transport protocols | `transport output ssh` |
| Incoming access restrictions | `access-class MGMT-IN in` |
| Outgoing access restrictions | `access-class MGMT-OUT out` |
| EXEC inactivity timeout | `exec-timeout 10 0` |
| Session inactivity timeout | `session-timeout 10` |
| Disable the EXEC process | `no exec` |
| Default privilege level | `privilege level 15` |
| Synchronous logging | `logging synchronous` |
| Rotary group membership | `rotary 1` |

Not all commands apply to all line types or platforms.

These functions are covered in detail in the subsequent pages.

## Line Authentication

Line authentication determines how a user proves their identity when accessing the CLI.

Without AAA, IOS XE supports traditional line password authentication and local username/password authentication.

### Line Password Authentication

```text
R1(config)# line console 0
R1(config-line)# password CISCO
R1(config-line)# login
```

The user must enter the configured line password to access the CLI.

This method uses a shared password rather than identifying individual users.

### Local User Authentication

```text
R1(config)# username ADMIN secret ADMIN-PASSWORD
!
R1(config)# line vty 0 15
R1(config-line)# login local
```

Users authenticate using usernames and passwords configured in the local user database.

Unlike traditional line password authentication, local user authentication identifies individual users.

### AAA Authentication

When AAA is enabled, authentication can be controlled using AAA method lists.

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login ADMIN local
!
R1(config)# line vty 0 15
R1(config-line)# login authentication ADMIN
```

The VTY lines use the AAA login authentication method list named `ADMIN`, which specifies the local user database as its authentication method.

AAA can also provide centralized authentication through TACACS+ and RADIUS, as well as authorization and accounting.

These mechanisms are covered in [4.1.b AAA](../../../device-security/aaa/overview.md).

## Transport Protocols

The `transport input` and `transport output` commands control the protocols permitted on a line.

### Incoming Transport Protocols

The `transport input` command controls which protocols can be used to establish incoming connections to a line.

For example:

```text
R1(config)# line vty 0 15
R1(config-line)# transport input ssh
```

Only SSH is permitted for incoming connections to these VTY lines.

Other common configurations include:

```text
transport input telnet
transport input telnet ssh
transport input all
transport input none
```

| Configuration | Effect |
|---|---|
| `transport input ssh` | Permits incoming SSH connections |
| `transport input telnet` | Permits incoming Telnet connections |
| `transport input telnet ssh` | Permits incoming Telnet and SSH connections |
| `transport input all` | Permits all supported incoming transport protocols |
| `transport input none` | Prevents incoming connections through transport protocols |

Permitting a protocol on a VTY line does not necessarily enable the associated server.

For example, permitting SSH with `transport input ssh` does not replace the global configuration required to enable the SSH server.

### Outgoing Transport Protocols

The `transport output` command controls the outgoing transport protocols that a user can initiate from an EXEC session on a line.

For example:

```text
R1(config)# line vty 0 15
R1(config-line)# transport input ssh
R1(config-line)# transport output telnet ssh
```

A user must use SSH to establish a connection to R1 through these VTY lines.

However, after connecting to R1, the user can initiate either Telnet or SSH connections from R1 to other devices.

```text
Administrator ---- SSH ----> R1 ---- Telnet ----> R2
                                |
                                +---- SSH ------> R3
```

To prevent users from initiating outgoing connections through supported terminal transport protocols:

```text
R1(config-line)# transport output none
```

This does not prevent all forms of locally generated network traffic. It restricts outgoing terminal connections controlled by the line's transport configuration.

## EXEC Sessions

An **EXEC process** provides the IOS XE command-line interface through which a user enters commands.

The EXEC process supports two primary privilege modes:

- **User EXEC mode:** Provides limited access to operational commands.
- **Privileged EXEC mode:** Provides access to privileged commands, subject to privilege and authorization restrictions.

```text
R1>
R1> enable
R1#
```

By default, ordinary users typically begin in user EXEC mode at privilege level 1.

Users can also be assigned a higher initial privilege level through local user configuration, AAA authorization, or line configuration.

### Disabling the EXEC Process

The `no exec` command prevents IOS XE from starting an EXEC process on a line.

```text
R1(config)# line console 0
R1(config-line)# no exec
```

The line no longer provides normal interactive EXEC access.

The `no exec` command is particularly relevant on terminal servers, where asynchronous serial lines are used to access the consoles of other devices rather than provide access to the terminal server's own CLI.

The EXEC process can be re-enabled using the `exec` command.

Care must be taken when disabling EXEC access to a management line, particularly the console line.

## Session Management

IOS XE provides several mechanisms for controlling the lifetime and behavior of management sessions.

These include:

- **EXEC timeout:** Terminates an inactive EXEC session.
- **Session timeout:** Controls inactivity timeouts for terminal sessions.
- **Absolute timeout:** Limits the total duration of a connection, regardless of activity.
- **Access classes:** Restrict incoming or outgoing connections based on ACLs.
- **TCP keepalives:** Help detect connections that are no longer functional.

For example:

```text
R1(config)# line vty 0 15
R1(config-line)# exec-timeout 5 0
```

An EXEC session is terminated after five minutes of inactivity.

The `exec-timeout` command accepts two arguments:

```text
exec-timeout <minutes> [seconds]
```

Configuring `exec-timeout 0 0` disables the EXEC inactivity timeout.

Session timeouts and access restrictions are covered in detail in [Session Timeouts and Access Restrictions](access-control.md).

## Rotary Groups

A **rotary group** is a logical grouping of lines.

Rotary groups allow incoming connections to be directed to particular groups of lines rather than relying solely on normal line allocation.

A line is assigned to a rotary group using the `rotary` command.

```text
R1(config)# line vty 5 9
R1(config-line)# rotary 1
```

In this example, VTY lines 5 through 9 belong to rotary group 1.

On supported platforms, Telnet connections to TCP port 3001 can be directed to rotary group 1.

SSH can also be associated with rotary groups using explicitly configured TCP ports.

Rotary groups can be used to:

- Provide management access through alternative TCP ports.
- Select specific groups of VTY lines.
- Apply different authentication policies to different management connections.
- Apply different access restrictions to different groups of users.
- Provide access to asynchronous serial lines on terminal servers.

Rotary group membership alone does not necessarily prevent a line from being selected by an ordinary incoming connection.

Rotary group operation and configuration are covered in detail in [Rotary Groups and Custom TCP Ports](rotary-groups.md).

## Verification

Several IOS XE commands are available for verifying line configuration and management sessions.

### show line

The `show line` command displays information about the device's lines.

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

The output includes information such as:

- Absolute line number.
- Line type.
- Rotary group membership.
- Incoming and outgoing access classes.
- Line usage statistics.

The asterisk indicates the line associated with the current terminal session.

Notice that the first VTY line has an absolute line number of 2 in this example. However, it is configured using `line vty 0`, not `line vty 2`.

Absolute line numbering depends on the device's available line types and platform.

### show users

The `show users` command displays information about active user sessions.

```text
R1# show users
    Line       User       Host(s)              Idle       Location
*  0 con 0                idle                 00:00:00
   2 vty 0     ADMIN      idle                 00:01:15 192.0.2.100
   3 vty 1     ADMIN2     idle                 00:00:30 192.0.2.200
```

This command can be used to identify:

- Active console and VTY sessions.
- Authenticated usernames.
- Session inactivity.
- Remote source IP addresses.
- Absolute and relative line numbers.

### show sessions

The `show sessions` command displays outgoing terminal connections associated with the current EXEC session.

For example, if an administrator connects to R1 and then initiates a Telnet session from R1 to R2, the outgoing connection can be displayed using `show sessions` after returning to R1's EXEC prompt.

The distinction between `show users` and `show sessions` is important:

- `show users` displays user sessions on the local device.
- `show sessions` displays outgoing terminal connections associated with the current EXEC session.

### show running-config | section line

This command displays line configuration in the running configuration.

```text
R1# show running-config | section line
line con 0
 exec-timeout 10 0
 logging synchronous
line aux 0
line vty 0 4
 exec-timeout 5 0
 login local
 transport input ssh
line vty 5 15
 exec-timeout 5 0
 login local
 transport input ssh
```

This is useful for identifying inconsistent configurations across different ranges of VTY lines.

Commands that retain their default values might not appear in the standard running configuration output.

### clear line

The `clear line` command can be used to terminate a session associated with a specified line.

For example, assuming absolute line 2 corresponds to VTY line 0:

```text
R1# clear line 2
```

On supported platforms, a relative line type and number can also be specified:

```text
R1# clear line vty 0
```

This is useful for terminating abandoned or unauthorized sessions and releasing occupied VTY lines.

## Verification Command Summary

| Command | Purpose |
|---|---|
| `show line` | Display available lines and line information |
| `show line <number>` | Display detailed information about a specified line |
| `show users` | Display active user sessions |
| `show sessions` | Display outgoing terminal connections |
| `show terminal` | Display settings for the current terminal |
| `show running-config \| section line` | Display line configuration |
| `clear line <number>` | Clear a specified line |

## Key Concepts

- Lines provide the communication channels through which users access the IOS XE CLI.
- Console lines provide access without requiring IP connectivity to the managed device.
- VTY lines are virtual resources used for remote CLI sessions.
- Each incoming Telnet or SSH CLI session occupies a VTY line.
- Different ranges of VTY lines can have different configurations.
- Absolute line numbering and relative line numbering are not necessarily the same.
- `transport input` controls incoming transport protocols.
- `transport output` controls outgoing terminal transport protocols.
- `no exec` prevents the line from providing a normal EXEC session.
- Line authentication can use shared passwords, the local user database, or AAA method lists.
- Rotary groups allow incoming connections to be directed to particular groups of lines.
- `show line` displays line information, whereas `show users` displays active user sessions.
- `show sessions` displays outgoing terminal connections associated with the current EXEC session.
