# Console Line Configuration

The **console line** provides CLI access to a Cisco device without requiring IP connectivity.

Console access is particularly important for initial device configuration, password recovery, boot process monitoring, and troubleshooting problems that prevent remote management access.

Most Cisco IOS XE devices have a single console line, numbered 0.

## Console Interfaces

Depending on the platform, a Cisco device can provide one or more types of physical console interfaces:

- RJ45 serial console.
- USB console.
- Other serial console interfaces.

An RJ45 console port uses a serial communication protocol, not Ethernet, despite using the same connector type as many Ethernet interfaces.

A terminal emulator, such as SecureCRT, PuTTY, or Tera Term, can be used to access the console.

### Default Serial Settings

The traditional default serial console settings on Cisco devices are:

| Parameter | Default Value |
|---|---|
| Speed (baud rate) | 9600 |
| Data bits | 8 |
| Parity | None |
| Stop bits | 1 |
| Flow control | None |

These settings are commonly referred to as **9600 8N1**.

The terminal emulator's serial settings must match those of the device.

USB console connections do not necessarily use the configurable physical serial signaling parameters of an RJ45 console port.

## Console Line Configuration Mode

The console line is configured using the `line console 0` command.

```text
R1(config)# line console 0
R1(config-line)#
```

Commands entered in this configuration mode apply to the console line.

The console line configuration appears under `line con 0` in the running configuration.

```text
R1# show running-config | section line con
line con 0
 exec-timeout 10 0
 logging synchronous
```

## Console Authentication

Console access can be protected using several authentication methods.

### No Authentication

Without AAA, the `no login` command disables login authentication on the console line.

```text
R1(config)# line console 0
R1(config-line)# no login
```

A user can access the EXEC process without entering login credentials.

This does not necessarily grant privileged EXEC access. The user might still need to authenticate using the `enable` command.

### Line Password Authentication

A shared console password can be configured using the `password` and `login` commands.

```text
R1(config)# line console 0
R1(config-line)# password CISCO
R1(config-line)# login
```

When accessing the console, the user is prompted for the configured password.

```text
User Access Verification

Password:

R1>
```

The user is not required to enter a username.

The `password` command configures the line password, whereas the `login` command enables line password authentication when AAA is disabled.

### Local User Authentication

The `login local` command enables authentication using the local username database.

```text
R1(config)# username ADMIN privilege 15 secret ADMIN-PASSWORD
!
R1(config)# line console 0
R1(config-line)# login local
```

When accessing the console, the user is prompted for a username and password.

```text
User Access Verification

Username: ADMIN
Password:

R1#
```

In this example, the local user has privilege level 15 and enters privileged EXEC mode directly.

Local user authentication is preferable to shared line passwords because individual users can have separate credentials and privilege levels.

### AAA Authentication

When AAA is enabled, console login authentication can be controlled using AAA method lists.

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login CONSOLE local
!
R1(config)# line console 0
R1(config-line)# login authentication CONSOLE
```

In this example, the console uses the AAA login authentication method list named `CONSOLE`, which authenticates users against the local username database.

Different AAA authentication method lists can be applied to console and VTY lines.

For example:

```text
R1(config)# aaa authentication login CONSOLE local
R1(config)# aaa authentication login REMOTE group tacacs+ local
!
R1(config)# line console 0
R1(config-line)# login authentication CONSOLE
!
R1(config)# line vty 0 15
R1(config-line)# login authentication REMOTE
```

Console users authenticate against the local username database, whereas remote users authenticate using TACACS+, with local authentication as a fallback if the TACACS+ method returns an error.

Using local authentication for console access allows administrators to access the device even if external AAA servers are unreachable.

AAA authentication is covered in detail in [Login and Enable Authentication](../../../device-security/aaa/authentication.md).

## Console Authorization

By default, AAA EXEC and command authorization are not performed on the console line.

The `aaa authorization console` global configuration command enables AAA authorization on the console.

For example:

```text
R1(config)# aaa new-model
R1(config)# aaa authorization exec CONSOLE local
R1(config)# aaa authorization console
!
R1(config)# line console 0
R1(config-line)# authorization exec CONSOLE
```

Two components are required in this example:

- `aaa authorization console` enables AAA authorization on the console.
- `authorization exec CONSOLE` applies the specified EXEC authorization method list to the console line.

Enabling AAA authorization on the console should be done carefully because an incorrect configuration can prevent administrative access through the console.

## Default Privilege Level

The `privilege level` line configuration command specifies the default privilege level for EXEC sessions on the line.

```text
R1(config)# line console 0
R1(config-line)# privilege level 15
```

Users accessing the console can enter privileged EXEC mode directly without entering the `enable` command.

The default line privilege level is 1.

A user's effective initial privilege level can also be determined by local user configuration or AAA EXEC authorization.

Configuring privilege level 15 on the console does not disable login authentication. Authentication and privilege assignment are separate functions.

## EXEC Timeout

The `exec-timeout` command specifies how long an EXEC session can remain idle before it is terminated.

```text
R1(config)# line console 0
R1(config-line)# exec-timeout 5 30
```

In this example, the EXEC inactivity timeout is five minutes and 30 seconds.

The syntax is:

```text
exec-timeout <minutes> [seconds]
```

The default EXEC inactivity timeout is 10 minutes.

### Disabling the EXEC Timeout

The EXEC inactivity timeout can be disabled by configuring both arguments as 0.

```text
R1(config)# line console 0
R1(config-line)# exec-timeout 0 0
```

The EXEC session is not terminated due to inactivity.

This is convenient in lab environments, where administrators might leave console sessions open for extended periods.

However, leaving an authenticated console session unattended can allow unauthorized users to access the device.

The `no exec-timeout` command also disables the EXEC timeout.

To restore the default timeout, use the `default exec-timeout` command or explicitly configure `exec-timeout 10 0`.

## Logging Synchronous

System messages can be displayed on the console while an administrator is entering a command.

Without synchronous logging, these messages can visually interrupt the command being entered.

For example:

```text
R1(config)# interface Gigabit
*Nov 24 10:00:00.123: %LINK-3-UPDOWN: Interface GigabitEthernet2, changed state to down
Ethernet1
```

Although the system message interrupts the displayed command, it does not modify the command stored in the input buffer.

The `logging synchronous` command causes IOS XE to redisplay the command prompt and partially entered command after displaying an unsolicited message.

```text
R1(config)# line console 0
R1(config-line)# logging synchronous
```

For example:

```text
R1(config)# interface Gigabit
*Nov 24 10:00:00.123: %LINK-3-UPDOWN: Interface GigabitEthernet2, changed state to down
R1(config)# interface GigabitEthernet1
```

This makes CLI interaction easier when the device is generating system messages.

The `logging synchronous` command does not disable logging or prevent messages from being displayed.

### Console Logging vs. Synchronous Logging

The `logging console` and `logging synchronous` commands perform different functions.

| Command | Configuration Mode | Purpose |
|---|---|---|
| `logging console` | Global configuration | Controls system message logging to the console |
| `logging synchronous` | Line configuration | Controls how unsolicited messages interact with terminal input |

For example:

```text
R1(config)# logging console warnings
!
R1(config)# line console 0
R1(config-line)# logging synchronous
```

Console logging is limited to severity levels 0 through 4, and synchronous logging is enabled.

To disable system message logging to the console:

```text
R1(config)# no logging console
```

Disabling console logging does not disable the console line or prevent normal CLI command output.

Logging is covered in detail in [4.3.c Logging](../../logging/overview.md).


## Terminal Settings

Several line configuration commands control the behavior of the terminal.

### Screen Length and Width

The `length` command specifies the number of lines displayed before IOS XE pauses output with the `--More--` prompt.

```text
R1(config)# line console 0
R1(config-line)# length 24
```

Configuring `length 0` disables output pagination.

The `width` command specifies the number of character columns on the terminal.

```text
R1(config)# line console 0
R1(config-line)# width 120
```

These settings can also be modified temporarily for the current EXEC session:

```text
R1# terminal length 0
R1# terminal width 120
```

The `terminal` commands affect the current session, whereas line configuration commands establish settings for the line.

### Command History

IOS XE maintains a history buffer of previously entered commands.

The `history size` command specifies the size of the command history buffer.

```text
R1(config)# line console 0
R1(config-line)# history size 50
```

The default history buffer size is 10 commands.

Previously entered commands can be displayed using `show history` and recalled using the up and down arrow keys.

The history buffer size can also be modified for the current EXEC session:

```text
R1# terminal history size 50
```

### Preferred Transport Protocol

The `transport preferred` command specifies the preferred protocol for implicit outgoing terminal connections.

For example, if an administrator enters an unrecognized command, IOS XE might interpret it as a hostname to which a connection should be established.

Configuring `transport preferred none` disables implicit outgoing terminal connections.

```text
R1(config)# line console 0
R1(config-line)# transport preferred none
```

Explicitly initiated connections using commands such as `telnet` and `ssh` are still permitted, subject to the `transport output` configuration.

The `transport preferred none` and `transport output none` commands serve different purposes:

| Command | Purpose |
|---|---|
| `transport preferred none` | Disables implicit outgoing terminal connections |
| `transport output none` | Disables outgoing connections through supported terminal transport protocols |

The `no ip domain lookup` global configuration command serves a different purpose: it disables DNS-based hostname resolution.

## Serial Communication Parameters

The physical serial communication parameters of a console line can be modified on supported platforms.

For example:

```text
R1(config)# line console 0
R1(config-line)# speed 115200
R1(config-line)# databits 8
R1(config-line)# parity none
R1(config-line)# stopbits 1
R1(config-line)# flowcontrol none
```

| Command | Purpose |
|---|---|
| `speed` | Sets the transmit and receive speed |
| `databits` | Sets the number of data bits per character |
| `parity` | Configures parity checking |
| `stopbits` | Sets the number of stop bits |
| `flowcontrol` | Configures flow control |

After changing the console speed, the terminal emulator must be configured to use the new speed.

A mismatch in serial communication parameters can result in unreadable characters or an inability to communicate with the device.

Higher console speeds can improve the responsiveness of serial console sessions and reduce the time required for serial file transfers.

## Console Banners

Banners provide messages to users accessing the device.

Common banner types include:

| Banner | Purpose |
|---|---|
| `banner motd` | Displays a message of the day |
| `banner login` | Displays a message before login authentication |
| `banner exec` | Displays a message when an EXEC session starts |

For example:

```text
R1(config)# banner login #
Enter your authorized administrative credentials.
#
```

Banners are configured globally rather than under the console line.

The display of EXEC and MOTD banners can be suppressed on individual lines using the `no exec-banner` command.

```text
R1(config)# line console 0
R1(config-line)# no exec-banner
```

## Disabling Console EXEC Access

The `no exec` command prevents IOS XE from starting an EXEC process on the console line.

```text
R1(config)# line console 0
R1(config-line)# no exec
```

This prevents normal interactive CLI access through the console.

However, it does not disable the physical console interface itself or necessarily suppress system messages sent to the console.

EXEC access can be restored using the `exec` command.

```text
R1(config)# line console 0
R1(config-line)# exec
```

Disabling console EXEC access should be done carefully because it removes an important method of recovering from management connectivity problems.

## Complete Configuration Example

The following example configures console access using local user authentication without AAA:

```text
R1(config)# username ADMIN privilege 15 secret ADMIN-PASSWORD
!
R1(config)# line console 0
R1(config-line)# login local
R1(config-line)# exec-timeout 5 0
R1(config-line)# logging synchronous
R1(config-line)# history size 50
R1(config-line)# transport preferred none
R1(config-line)# transport output none
```

This configuration provides the following behavior:

- Users authenticate against the local username database.
- The ADMIN user enters privileged EXEC mode directly.
- Inactive EXEC sessions are terminated after five minutes.
- Unsolicited messages do not disrupt the displayed command prompt.
- The command history buffer stores up to 50 commands.
- Implicit outgoing terminal connections are disabled.
- Explicit outgoing terminal connections are also disabled.

## Verification

### show line console 0

The `show line console 0` command displays detailed information about the console line.

```text
R1# show line console 0
```

The output includes information about:

- Serial communication parameters.
- EXEC and session timeouts.
- Transport protocols.
- Line capabilities and status.
- Line usage statistics.
- Terminal settings.

The console line can also be specified using its absolute line number:

```text
R1# show line 0
```

### show terminal

The `show terminal` command displays the terminal settings associated with the current session.

```text
R1# show terminal
```

This is useful for verifying settings that can be modified on a per-session basis, such as terminal length, width, and history buffer size.

### show running-config all

Commands configured with their default values are not necessarily displayed in the standard running configuration.

The following command includes default console line settings:

```text
R1# show running-config all | section ^line con
```

### Verification Command Summary

| Command | Purpose |
|---|---|
| `show line` | Display information about all lines |
| `show line console 0` | Display detailed console line information |
| `show terminal` | Display the current session's terminal settings |
| `show users` | Display active user sessions |
| `show history` | Display previously entered commands |
| `show running-config \| section ^line con` | Display console line configuration |
| `show running-config all \| section ^line con` | Display console line configuration, including defaults |

## Key Concepts

- Console access does not require IP connectivity.
- The traditional serial console settings are 9600 baud, 8 data bits, no parity, and 1 stop bit (9600 8N1).
- `line console 0` enters console line configuration mode.
- Console authentication can use a shared line password, the local username database, or AAA.
- AAA EXEC and command authorization on the console must be explicitly enabled using `aaa authorization console`.
- The default EXEC inactivity timeout is 10 minutes.
- `exec-timeout 0 0` disables the EXEC inactivity timeout.
- `logging synchronous` redisplays the command prompt and partially entered command after unsolicited messages.
- `length 0` disables output pagination.
- Line configuration commands establish terminal settings, whereas EXEC-mode `terminal` commands modify the current session.
- `transport preferred none` disables implicit outgoing terminal connections.
- `transport output none` disables outgoing terminal connections through supported transport protocols.
- `no exec` prevents IOS XE from starting an EXEC process on the line.
