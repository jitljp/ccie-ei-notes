# Rotary Groups and Custom TCP Ports

A **rotary group** is a logical grouping of terminal lines on a Cisco IOS XE device.

Rotary groups allow incoming connections to be directed to specific groups of lines, rather than relying solely on normal line allocation.

Rotary groups can be used with VTY lines and physical asynchronous lines.

Common applications include:

- Providing Telnet access through nonstandard TCP ports.
- Providing SSH access through custom TCP ports.
- Directing incoming connections to specific ranges of VTY lines.
- Applying different authentication and authorization policies to different management connections.
- Applying different access restrictions and session settings.
- Providing remote access to devices connected to a terminal server.

## Rotary Group Operation

Without rotary groups, incoming Telnet and SSH connections are normally assigned to the first eligible available VTY line.

Consider a router with 16 VTY lines:

```text
                         R1
                +-------------------+
                | VTY 0             |
                | VTY 1             |
                | ...               |
                | VTY 15            |
                +-------------------+
```

Rotary groups allow these lines to be divided into logical groups.

For example:

```text
                         R1
                +-------------------+
                | VTY 0-4           |
                | No Rotary Group   |
                +-------------------+
                | VTY 5-9           |
                | Rotary Group 1    |
                +-------------------+
                | VTY 10-15         |
                | Rotary Group 2    |
                +-------------------+
```

Incoming connections targeting rotary group 1 are assigned to an eligible available line within VTY lines 5 through 9.

Incoming connections targeting rotary group 2 are assigned to an eligible available line within VTY lines 10 through 15.

If all eligible lines in the requested rotary group are occupied, the connection cannot be assigned to a line in that group.

The connection does not automatically fall back to a line belonging to another rotary group.

## Rotary Group Configuration

The `rotary` line configuration command assigns a line to a rotary group.

The syntax is:

```text
rotary <group-number>
```

For example:

```text
R1(config)# line vty 5 9
R1(config-line)# rotary 1
!
R1(config)# line vty 10 15
R1(config-line)# rotary 2
```

In this example:

- VTY lines 5 through 9 belong to rotary group 1.
- VTY lines 10 through 15 belong to rotary group 2.
- VTY lines 0 through 4 do not belong to a rotary group.

Multiple lines can belong to the same rotary group.

The rotary group number is independent of the absolute and relative line numbers.

For example, rotary group 1 does not necessarily contain VTY line 1.

To remove a line from its rotary group:

```text
R1(config)# line vty 5 9
R1(config-line)# no rotary
```

### Rotary Group Verification

The `show line` command displays rotary group membership in the `Roty` column.

For example, the following simplified output assumes that VTY line 0 corresponds to absolute line 2:

```text
R1# show line
   Tty Typ     Tx/Rx    A Modem  Roty AccO AccI  Uses
*    0 CTY              -    -      -    -    -     0
     1 AUX   9600/9600  -    -      -    -    -     0
     2 VTY              -    -      -    -    -     0
     3 VTY              -    -      -    -    -     0
     4 VTY              -    -      -    -    -     0
     5 VTY              -    -      -    -    -     0
     6 VTY              -    -      -    -    -     0
     7 VTY              -    -      1    -    -     0
     8 VTY              -    -      1    -    -     0
     9 VTY              -    -      1    -    -     0
    10 VTY              -    -      1    -    -     0
    11 VTY              -    -      1    -    -     0
    12 VTY              -    -      2    -    -     0
    13 VTY              -    -      2    -    -     0
    14 VTY              -    -      2    -    -     0
    15 VTY              -    -      2    -    -     0
    16 VTY              -    -      2    -    -     0
    17 VTY              -    -      2    -    -     0
```

A dash (`-`) indicates that the line does not belong to a rotary group.

Remember that the `Tty` column displays absolute line numbers, not relative VTY line numbers.

## Telnet Rotary Groups

Telnet rotary groups use automatically calculated TCP port numbers.

The TCP port number is calculated as follows:

```text
TCP port = 3000 + rotary group number
```

For example:

| Rotary Group | Telnet TCP Port |
|---:|---:|
| 1 | 3001 |
| 2 | 3002 |
| 5 | 3005 |
| 10 | 3010 |
| 100 | 3100 |

Unlike SSH rotary groups, Telnet rotary groups do not require a separate global command to associate the rotary group with a TCP port.

### Telnet Rotary Group Configuration

Assume R1 has the IP address 192.0.2.1.

Configure VTY lines 5 through 9 as rotary group 1:

```text
R1(config)# username ADMIN secret ADMIN-PASSWORD
!
R1(config)# line vty 5 9
R1(config-line)# login local
R1(config-line)# transport input telnet
R1(config-line)# rotary 1
```

A remote client can connect to rotary group 1 using TCP port 3001:

```text
R2# telnet 192.0.2.1 3001
Trying 192.0.2.1, 3001 ... Open

User Access Verification

Username: ADMIN
Password:

R1>
```

The connection is assigned to an available eligible line in rotary group 1.

```text
                         R1
                +-------------------+
                | VTY 0-4           |
                | No Rotary Group   |
                +-------------------+
TCP 3001 ------>| VTY 5-9           |
                | Rotary Group 1    |
                +-------------------+
                | VTY 10-15         |
                | No Rotary Group   |
                +-------------------+
```

The rotary group determines which lines can be selected.

It does not bypass the authentication, authorization, or access restrictions configured on those lines.

### Multiple Telnet Rotary Groups

Different rotary groups can be assigned different line settings.

For example:

```text
R1(config)# line vty 5 9
R1(config-line)# login local
R1(config-line)# transport input telnet
R1(config-line)# rotary 1
R1(config-line)# exec-timeout 5 0
!
R1(config)# line vty 10 15
R1(config-line)# login local
R1(config-line)# transport input telnet
R1(config-line)# rotary 2
R1(config-line)# exec-timeout 15 0
```

The resulting behavior is:

| Destination TCP Port | Eligible VTY Lines | EXEC Timeout |
|---:|---|---|
| 3001 | 5-9 | 5 minutes |
| 3002 | 10-15 | 15 minutes |

```text
                         R1
                +-------------------+
                | VTY 0-4           |
                | No Rotary Group   |
                +-------------------+
TCP 3001 ------>| VTY 5-9           |
                | Rotary Group 1    |
                +-------------------+
TCP 3002 ------>| VTY 10-15         |
                | Rotary Group 2    |
                +-------------------+
```

The destination TCP port determines which rotary group is targeted.

## Standard Telnet Access and Rotary Groups

Assigning a VTY line to a rotary group does not inherently prevent that line from being selected by an ordinary Telnet connection to TCP port 23.

For example:

```text
R1(config)# line vty 0 4
R1(config-line)# rotary 1
R1(config-line)# transport input telnet
```

TCP port 3001 provides an additional means of accessing the lines in rotary group 1.

However, TCP port 23 can still provide access to those lines through normal VTY allocation.

This is an important distinction:

- A connection to TCP port 3001 explicitly targets rotary group 1.
- A connection to TCP port 23 uses normal VTY line allocation.

Configuring a rotary group does not, by itself, change the default Telnet port from TCP 23 to TCP 3001.

To prevent access through TCP port 23 while permitting access through TCP port 3001, additional filtering is required.

For example, an interface ACL can selectively deny TCP port 23 while permitting TCP port 3001.

## SSH Rotary Groups

Unlike Telnet, SSH does not automatically calculate a TCP port number based on the rotary group number.

An SSH TCP port must be explicitly associated with a rotary group using the `ip ssh port` global configuration command.

The syntax is:

```text
ip ssh port <port-number> rotary <group-number>
```

For example:

```text
R1(config)# ip ssh port 2222 rotary 1
```

This associates TCP port 2222 with rotary group 1.

There is no requirement for the TCP port number to be mathematically related to the rotary group number.

The rotary group number in the `ip ssh port` command must match the rotary group number configured on the relevant lines.

### SSH Rotary Group Configuration

Assume the SSH server is already enabled on R1.

Configure VTY lines 5 through 9 as rotary group 1:

```text
R1(config)# line vty 5 9
R1(config-line)# login local
R1(config-line)# transport input ssh
R1(config-line)# rotary 1
!
R1(config)# ip ssh port 2222 rotary 1
```

A remote IOS XE device can establish an SSH connection to rotary group 1 using the `-p` option:

```text
R2# ssh -l ADMIN -p 2222 192.0.2.1
```

An OpenSSH client can connect using the following command:

```text
ssh -p 2222 ADMIN@192.0.2.1
```

The incoming SSH connection is directed to an eligible available line in rotary group 1.

```text
                         R1
                +-------------------+
                | VTY 0-4           |
                | No Rotary Group   |
                +-------------------+
TCP 2222 ------>| VTY 5-9           |
                | Rotary Group 1    |
                +-------------------+
                | VTY 10-15         |
                | No Rotary Group   |
                +-------------------+
```

### Multiple SSH Rotary Groups

Different TCP ports can be associated with different rotary groups.

For example:

```text
R1(config)# line vty 5 9
R1(config-line)# rotary 1
R1(config-line)# transport input ssh
!
R1(config)# line vty 10 15
R1(config-line)# rotary 2
R1(config-line)# transport input ssh
!
R1(config)# ip ssh port 2222 rotary 1
R1(config)# ip ssh port 2223 rotary 2
```

The resulting mappings are:

| Destination TCP Port | Rotary Group | Eligible VTY Lines |
|---:|---:|---|
| 2222 | 1 | 5-9 |
| 2223 | 2 | 10-15 |

```text
                         R1
                +-------------------+
                | VTY 0-4           |
                | No Rotary Group   |
                +-------------------+
TCP 2222 ------>| VTY 5-9           |
                | Rotary Group 1    |
                +-------------------+
TCP 2223 ------>| VTY 10-15         |
                | Rotary Group 2    |
                +-------------------+
```


### Mapping a Range of SSH Rotary Groups

Instead of configuring a separate SSH port mapping for each rotary group, a range of consecutive rotary groups can be specified using a single command.

The complete syntax is:

```text
ip ssh port <starting-port> rotary <low-group> [high-group]
```

The optional `high-group` argument specifies the last rotary group in the range.

For example:

```text
R1(config)# ip ssh port 3000 rotary 1 5
```

IOS XE sequentially maps consecutive TCP ports to consecutive rotary groups, starting with the specified TCP port and the lowest rotary group number.

| SSH TCP Port | Rotary Group |
|---:|---:|
| 3000 | 1 |
| 3001 | 2 |
| 3002 | 3 |
| 3003 | 4 |
| 3004 | 5 |

The preceding command is functionally equivalent to the following individual port mappings:

```text
R1(config)# ip ssh port 3000 rotary 1
R1(config)# ip ssh port 3001 rotary 2
R1(config)# ip ssh port 3002 rotary 3
R1(config)# ip ssh port 3003 rotary 4
R1(config)# ip ssh port 3004 rotary 5
```

The TCP port associated with a particular rotary group can be calculated as follows:

```text
TCP port = starting port + (group number - low group)
```

For example:

```text
R1(config)# ip ssh port 4000 rotary 10 13
```

The resulting mappings are:

| SSH TCP Port | Rotary Group |
|---:|---:|
| 4000 | 10 |
| 4001 | 11 |
| 4002 | 12 |
| 4003 | 13 |

Notice that the starting TCP port is associated with the lowest rotary group in the specified range, regardless of its group number.

#### Complete Configuration Example

```text
R1(config)# line vty 5 7
R1(config-line)# rotary 1
R1(config-line)# transport input ssh
!
R1(config)# line vty 8 10
R1(config-line)# rotary 2
R1(config-line)# transport input ssh
!
R1(config)# line vty 11 15
R1(config-line)# rotary 3
R1(config-line)# transport input ssh
!
R1(config)# ip ssh port 3000 rotary 1 3
```

The resulting port-to-line mappings are:

```text
                         R1
                +-------------------+
TCP 3000 ------>| VTY 5-7           |
                | Rotary Group 1    |
                +-------------------+
TCP 3001 ------>| VTY 8-10          |
                | Rotary Group 2    |
                +-------------------+
TCP 3002 ------>| VTY 11-15         |
                | Rotary Group 3    |
                +-------------------+
```

The `ip ssh port` command does not assign lines to rotary groups. Rotary group membership must still be configured using the `rotary` line configuration command.

Configuring a range of SSH rotary groups is particularly useful on terminal servers, where many asynchronous lines can each belong to a separate rotary group.

#### Single Group vs. Range of Groups

| Command | Result |
|---|---|
| `ip ssh port 3000 rotary 1` | Maps TCP port 3000 to rotary group 1 |
| `ip ssh port 3000 rotary 1 5` | Maps TCP ports 3000-3004 to rotary groups 1-5 |
| `ip ssh port 3000 rotary 1 127` | Maps TCP ports 3000-3126 to rotary groups 1-127 |

The range is inclusive of both the low and high rotary group numbers.

If the high rotary group number is omitted, only a single port-to-group mapping is configured.

### Removing an SSH Port Mapping

The following command removes the association between TCP port 2222 and rotary group 1:

```text
R1(config)# no ip ssh port 2222 rotary 1
```

Removing the SSH port mapping does not remove rotary group membership from the VTY lines.

Similarly, removing rotary group membership from the lines does not automatically remove the global SSH port mapping.

Both components must be considered when configuring or removing SSH rotary groups.

## Standard SSH Access and Rotary Groups

The `ip ssh port` command associates an additional SSH TCP port with a rotary group.

It does not inherently disable the standard SSH TCP port (22).

For example:

```text
R1(config)# ip ssh port 2222 rotary 1
```

This command does not globally replace TCP port 22 with TCP port 2222.

Ordinary SSH connections to TCP port 22 can still use normal VTY line allocation, including eligible lines assigned to rotary groups.

If the objective is to permit SSH exclusively through a custom TCP port, access to TCP port 22 must be restricted separately.

Possible filtering mechanisms include:

- Interface ACLs.
- Extended VTY access classes that match TCP destination ports.
- External firewalls.

The `transport input ssh` command permits SSH as a protocol. It does not distinguish between SSH connections using standard and custom TCP ports.

## Telnet vs. SSH Rotary Groups

| Characteristic | Telnet | SSH |
|---|---|---|
| Standard TCP port | 23 | 22 |
| Rotary membership command | `rotary` | `rotary` |
| Rotary TCP port selection | Automatically calculated | Explicitly configured |
| Rotary TCP port formula | 3000 + group number | N/A |
| Additional global port mapping required | No | Yes |
| Global port mapping command | N/A | `ip ssh port ... rotary ...` |
| Encryption | No | Yes |
| Standard TCP port automatically disabled | No | No |

The important distinction is that Telnet automatically derives its rotary group TCP port, whereas SSH requires an explicit port mapping.

## Using Telnet and SSH with the Same Rotary Group

A rotary group is a grouping of lines, not a grouping of protocols.

The same rotary group can support both Telnet and SSH if both protocols are permitted on its lines.

For example:

```text
R1(config)# line vty 5 9
R1(config-line)# rotary 1
R1(config-line)# login local
R1(config-line)# transport input telnet ssh
!
R1(config)# ip ssh port 2222 rotary 1
```

The following connections target rotary group 1:

```text
R2# telnet 192.0.2.1 3001
```

```text
R2# ssh -l ADMIN -p 2222 192.0.2.1
```

Both connections use the same pool of VTY lines.

```text
                               R1
                      +-------------------+
Telnet TCP 3001 ----->|                   |
                      | VTY 5-9           |
                      | Rotary Group 1    |
SSH TCP 2222 -------->|                   |
                      +-------------------+
```

Rotary group 1 contains five VTY lines in this example.

It can therefore accommodate up to five simultaneous interactive sessions through those lines, regardless of the combination of Telnet and SSH sessions.

## Rotary Groups and AAA

Different rotary groups can use different AAA authentication and authorization method lists.

This allows the destination TCP port to determine which group of lines, and therefore which management policy, is selected.

For example, assume the following requirements:

- SSH connections to TCP port 2222 must use TACACS+ authentication, with local authentication as a fallback.
- SSH connections to TCP port 2223 must use local authentication.

Configure two AAA login authentication method lists:

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login CENTRAL group tacacs+ local
R1(config)# aaa authentication login LOCAL local
```

Configure the rotary groups:

```text
R1(config)# line vty 5 9
R1(config-line)# rotary 1
R1(config-line)# login authentication CENTRAL
R1(config-line)# transport input ssh
!
R1(config)# line vty 10 15
R1(config-line)# rotary 2
R1(config-line)# login authentication LOCAL
R1(config-line)# transport input ssh
```

Configure the SSH port mappings:

```text
R1(config)# ip ssh port 2222 rotary 1
R1(config)# ip ssh port 2223 rotary 2
```

The resulting authentication policies are:

| TCP Port | Rotary Group | Authentication Method List |
|---:|---:|---|
| 2222 | 1 | CENTRAL |
| 2223 | 2 | LOCAL |

```text
                         R1
                +-------------------+
TCP 2222 ------>| VTY 5-9           |
                | Rotary Group 1    |
                | CENTRAL AAA       |
                +-------------------+
TCP 2223 ------>| VTY 10-15         |
                | Rotary Group 2    |
                | LOCAL AAA         |
                +-------------------+
```

Different rotary groups can also use different EXEC authorization, command authorization, and accounting method lists.

Rotary groups do not perform authentication or authorization themselves. They provide a mechanism for selecting lines with the desired configuration.

For more information about AAA, see [4.1.b AAA](../../../device-security/aaa/overview.md).

## Rotary Groups and Access Classes

Different rotary groups can have different inbound and outbound access restrictions.

Assume the following requirements:

- SSH connections to TCP port 2222 are permitted only from 192.0.2.0/24.
- SSH connections to TCP port 2223 are permitted only from 198.51.100.0/24.

Configure two standard IPv4 ACLs:

```text
R1(config)# ip access-list standard ADMIN-IN
R1(config-std-nacl)# permit 192.0.2.0 0.0.0.255
!
R1(config)# ip access-list standard SUPPORT-IN
R1(config-std-nacl)# permit 198.51.100.0 0.0.0.255
```

Apply the ACLs to the corresponding rotary groups:

```text
R1(config)# line vty 5 9
R1(config-line)# rotary 1
R1(config-line)# login local
R1(config-line)# transport input ssh
R1(config-line)# access-class ADMIN-IN in
!
R1(config)# line vty 10 15
R1(config-line)# rotary 2
R1(config-line)# login local
R1(config-line)# transport input ssh
R1(config-line)# access-class SUPPORT-IN in
```

Configure the SSH port mappings:

```text
R1(config)# ip ssh port 2222 rotary 1
R1(config)# ip ssh port 2223 rotary 2
```

The resulting behavior is:

| Client Source IP | TCP Port 2222 | TCP Port 2223 |
|---|---|---|
| 192.0.2.100 | Permitted | Denied |
| 198.51.100.100 | Denied | Permitted |
| 203.0.113.100 | Denied | Denied |

These results apply to connections targeting the two configured rotary ports.

Ordinary SSH connections to TCP port 22 must be considered separately.

### Restricting Access by Destination TCP Port

An extended IPv4 ACL can match both the remote client's source IP address and the destination TCP port.

For example:

```text
R1(config)# ip access-list extended ADMIN-IN
R1(config-ext-nacl)# permit tcp 192.0.2.0 0.0.0.255 any eq 2222
!
R1(config)# ip access-list extended SUPPORT-IN
R1(config-ext-nacl)# permit tcp 198.51.100.0 0.0.0.255 any eq 2223
```

Apply these ACLs to the corresponding VTY lines:

```text
R1(config)# line vty 5 9
R1(config-line)# access-class ADMIN-IN in
!
R1(config)# line vty 10 15
R1(config-line)# access-class SUPPORT-IN in
```

On IOS XE platforms supporting destination-port matching in VTY access classes, these ACLs restrict each group of lines to connections using its designated destination TCP port.

For example, a connection to TCP port 22 does not match either ACL's permit statement.

All remaining VTY lines must also be secured if ordinary SSH access is to be prohibited.

For more information, see [Session Timeouts and Access Restrictions](access-control.md).

## Rotary Groups and Session Settings

Because rotary groups select specific lines, they can also provide different session settings.

For example:

```text
R1(config)# line vty 5 9
R1(config-line)# rotary 1
R1(config-line)# exec-timeout 15 0
R1(config-line)# absolute-timeout 120
R1(config-line)# transport output ssh
!
R1(config)# line vty 10 15
R1(config-line)# rotary 2
R1(config-line)# exec-timeout 5 0
R1(config-line)# absolute-timeout 30
R1(config-line)# transport output none
```

The resulting session policies are:

| Setting | Rotary Group 1 | Rotary Group 2 |
|---|---|---|
| EXEC inactivity timeout | 15 minutes | 5 minutes |
| Absolute timeout | 120 minutes | 30 minutes |
| Outgoing terminal protocols | SSH | None |

Other line settings can also differ between rotary groups, including:

- Default privilege levels.
- AAA authorization and accounting.
- Session limits.
- Automatic commands (`autocommand`).
- Command history size.
- Logging synchronous settings.

The rotary group determines which set of lines is selected, and the selected line determines the session's configuration.

## Rotary Groups on Terminal Servers

Rotary groups were originally associated with asynchronous lines and access servers.

A terminal server can provide network-based access to physical serial lines connected to other devices.

For example:

```text
Administrator
      |
      | TCP/IP Network
      |
+-------------------------+
|     Terminal Server     |
|                         |
| Async Line 1 [Rotary 1] |---- Serial ---- R1
| Async Line 2 [Rotary 2] |---- Serial ---- R2
| Async Line 3 [Rotary 3] |---- Serial ---- R3
+-------------------------+
```

Each asynchronous line can be assigned to a different rotary group.

Alternatively, multiple asynchronous lines can belong to the same rotary group.

Grouping multiple lines is useful when the lines provide equivalent services, such as connections to a pool of modems. The client targets the group rather than a particular line.

### Reverse Telnet

Reverse Telnet provides access to a specific asynchronous line using the following TCP port formula:

```text
TCP port = 2000 + absolute line number
```

Alternatively, a client can target an asynchronous rotary group:

```text
TCP port = 3000 + rotary group number
```

For example:

```text
TS(config)# line 1
TS(config-line)# no exec
TS(config-line)# transport input telnet
TS(config-line)# rotary 1
```

Assuming the terminal server's IP address is 192.0.2.100, both of the following connections can target the same physical line:

```text
R4# telnet 192.0.2.100 2001
```

```text
R4# telnet 192.0.2.100 3001
```

However, the two connections use different selection mechanisms:

- TCP port 2001 targets absolute line 1 directly.
- TCP port 3001 targets an available eligible line in rotary group 1.

If rotary group 1 contained multiple asynchronous lines, TCP port 3001 could select any eligible available line in that group.

The `no exec` command prevents the terminal server from starting its own EXEC process on the asynchronous line.

Instead, communication is forwarded between the network connection and the attached device.

For more information, see [Telnet Configuration and Operation](telnet.md).

### Reverse SSH

Reverse SSH provides encrypted remote access to asynchronous terminal lines.

One method of configuring reverse SSH is to assign asynchronous lines to rotary groups and associate each rotary group with an SSH TCP port.

Assume the following terminal server configuration:

```text
TS(config)# line 1
TS(config-line)# no exec
TS(config-line)# transport input ssh
TS(config-line)# rotary 1
!
TS(config)# line 2
TS(config-line)# no exec
TS(config-line)# transport input ssh
TS(config-line)# rotary 2
!
TS(config)# line 3
TS(config-line)# no exec
TS(config-line)# transport input ssh
TS(config-line)# rotary 3
```

Associate each rotary group with an SSH TCP port:

```text
TS(config)# ip ssh port 2201 rotary 1
TS(config)# ip ssh port 2202 rotary 2
TS(config)# ip ssh port 2203 rotary 3
```

Assume the SSH server and appropriate authentication mechanisms have already been configured.

The resulting mappings are:

| SSH TCP Port | Rotary Group | Asynchronous Line | Connected Device |
|---:|---:|---:|---|
| 2201 | 1 | 1 | R1 |
| 2202 | 2 | 2 | R2 |
| 2203 | 3 | 3 | R3 |

An administrator can access the connected devices through the terminal server.

```text
                              Terminal Server
                           +----------------------+
SSH TCP 2201 ------------>| Line 1 [Rotary 1]    |---- Console ---- R1
                           |                      |
SSH TCP 2202 ------------>| Line 2 [Rotary 2]    |---- Console ---- R2
                           |                      |
SSH TCP 2203 ------------>| Line 3 [Rotary 3]    |---- Console ---- R3
                           +----------------------+
```

The connected devices do not require IP connectivity.

The SSH encryption protects the network connection between the administrator and the terminal server. It does not extend encryption to the physical serial connection.

IOS XE also supports enhanced reverse SSH methods that can identify terminal lines or rotary groups through SSH login parameters rather than requiring a separate TCP port mapping for every line.

## Additional Rotary TCP Port Ranges

Traditional Cisco IOS terminal server functionality defines several TCP port ranges for accessing individual lines and rotary groups.

| Service | Individual Line | Rotary Group |
|---|---|---|
| Telnet | 2000 + line number | 3000 + group number |
| Raw TCP | 4000 + line number | 5000 + group number |
| Binary-mode Telnet | 6000 + line number | 7000 + group number |
| XRemote | 9000 + line number | 10000 + group number |

For individual line access, the absolute line number is used.

For rotary group access, the rotary group number is used.

These additional services and port ranges are primarily associated with traditional terminal server functionality. Their availability depends on the platform and line type.

The most important port calculations for the CCIE EI blueprint are:

```text
Reverse Telnet:       2000 + absolute line number
Telnet rotary group:  3000 + rotary group number
SSH rotary group:     Explicitly configured TCP port
```

## Complete Configuration Example

Assume R1 has the following management requirements:

- Standard SSH connections use TCP port 22.
- Administrators use TCP port 2222 to access rotary group 1.
- Support personnel use TCP port 2223 to access rotary group 2.
- Administrators must authenticate using TACACS+, with local authentication as a fallback.
- Support personnel must authenticate using the local user database.
- Administrators must connect from 192.0.2.0/24.
- Support personnel must connect from 198.51.100.0/24.
- Administrators have a 15-minute EXEC inactivity timeout.
- Support personnel have a five-minute EXEC inactivity timeout.
- Support personnel cannot initiate outgoing terminal connections.
- Telnet is prohibited.

The VTY lines are divided as follows:

```text
                         R1
                +----------------------+
TCP 22 -------->| VTY 0-4              |
                | Standard SSH Access  |
                +----------------------+
TCP 2222 ------>| VTY 5-9              |
                | Rotary Group 1       |
                | Administrators       |
                +----------------------+
TCP 2223 ------>| VTY 10-15            |
                | Rotary Group 2       |
                | Support Personnel    |
                +----------------------+
```

Assume the SSH server, TACACS+ servers, and local users have already been configured.

### AAA Configuration

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login STANDARD local
R1(config)# aaa authentication login ADMIN group tacacs+ local
R1(config)# aaa authentication login SUPPORT local
```

### ACL Configuration

The following extended ACLs restrict both source IP addresses and destination TCP ports:

```text
R1(config)# ip access-list extended STANDARD-IN
R1(config-ext-nacl)# permit tcp 192.0.2.0 0.0.0.255 any eq 22
!
R1(config)# ip access-list extended ADMIN-IN
R1(config-ext-nacl)# permit tcp 192.0.2.0 0.0.0.255 any eq 2222
!
R1(config)# ip access-list extended SUPPORT-IN
R1(config-ext-nacl)# permit tcp 198.51.100.0 0.0.0.255 any eq 2223
```

This example requires support for destination-port matching in extended VTY access classes.

### Standard VTY Configuration

```text
R1(config)# line vty 0 4
R1(config-line)# login authentication STANDARD
R1(config-line)# transport input ssh
R1(config-line)# access-class STANDARD-IN in
R1(config-line)# exec-timeout 10 0
R1(config-line)# logging synchronous
```

### Administrator Rotary Group

```text
R1(config)# line vty 5 9
R1(config-line)# rotary 1
R1(config-line)# login authentication ADMIN
R1(config-line)# transport input ssh
R1(config-line)# transport output ssh
R1(config-line)# access-class ADMIN-IN in
R1(config-line)# exec-timeout 15 0
R1(config-line)# logging synchronous
```

### Support Rotary Group

```text
R1(config)# line vty 10 15
R1(config-line)# rotary 2
R1(config-line)# login authentication SUPPORT
R1(config-line)# transport input ssh
R1(config-line)# transport output none
R1(config-line)# access-class SUPPORT-IN in
R1(config-line)# exec-timeout 5 0
R1(config-line)# logging synchronous
```

### SSH Port Mappings

```text
R1(config)# ip ssh port 2222 rotary 1
R1(config)# ip ssh port 2223 rotary 2
```

### Resulting Behavior

| Setting | Standard SSH | Administrator | Support |
|---|---|---|---|
| TCP port | 22 | 2222 | 2223 |
| VTY lines | 0-4 | 5-9 | 10-15 |
| Rotary group | None | 1 | 2 |
| Authentication | Local | TACACS+, local fallback | Local |
| Permitted source subnet | 192.0.2.0/24 | 192.0.2.0/24 | 198.51.100.0/24 |
| EXEC timeout | 10 minutes | 15 minutes | 5 minutes |
| Outgoing transport | Default | SSH | None |

The rotary groups determine which VTY lines are targeted by connections to the custom SSH ports.

The extended access classes prevent connections using unintended destination TCP ports from accessing those lines.

IPv6 access restrictions must be configured separately if IPv6 management connectivity is available.

## Verification

### show line

The `show line` command displays line information, including rotary group membership.

```text
R1# show line
```

Examine the `Roty` column to determine which rotary group each line belongs to.

Detailed information about an individual VTY line can be displayed as follows:

```text
R1# show line vty 5
```

### show users

The `show users` command displays active user sessions and their associated lines.

```text
R1# show users
```

For example:

```text
R1# show users
    Line       User       Host(s)              Idle       Location
*  0 con 0                idle                 00:00:00
   7 vty 5     ADMIN      idle                 00:00:12   192.0.2.100
  12 vty 10    SUPPORT    idle                 00:00:30   198.51.100.100
```

Assuming the preceding complete configuration example:

- ADMIN is connected through VTY line 5, which belongs to rotary group 1.
- SUPPORT is connected through VTY line 10, which belongs to rotary group 2.

The `show users` command confirms which line has been allocated to each connection.

### show tcp brief all

The `show tcp brief all` command displays TCP connections and listening sockets.

```text
R1# show tcp brief all
```

For example, the following simplified output shows established SSH connections using standard and custom TCP ports:

```text
TCB       Local Address         Foreign Address        (state)
XXXXXXXX  192.0.2.1.22          192.0.2.100.49150       ESTAB
XXXXXXXX  192.0.2.1.2222        192.0.2.101.49151       ESTAB
XXXXXXXX  192.0.2.1.2223        198.51.100.100.49152    ESTAB
```

This command can be used to verify the local TCP port associated with each connection.

### show ssh

The `show ssh` command displays information about established SSH sessions.

```text
R1# show ssh
```

This provides information about SSH protocol versions, cryptographic algorithms, session status, and usernames.

### show running-config

The following command displays configured SSH rotary port mappings:

```text
R1# show running-config | include ^ip ssh port
ip ssh port 2222 rotary 1
ip ssh port 2223 rotary 2
```

The following command displays VTY line configuration:

```text
R1# show running-config | section ^line vty
```

Both the global SSH port mappings and line-level rotary group memberships must be verified.

### Verification Command Summary

| Command | Purpose |
|---|---|
| `show line` | Display lines and rotary group memberships |
| `show line vty <number>` | Display detailed VTY line information |
| `show users` | Display active sessions and allocated lines |
| `show ssh` | Display SSH session information |
| `show ip ssh` | Display SSH server information |
| `show tcp brief all` | Display TCP connections and listening sockets |
| `show running-config \| include ^ip ssh port` | Display SSH rotary port mappings |
| `show running-config \| section ^line vty` | Display VTY configuration |
| `clear line <number>` | Clear a specified absolute line |
| `clear line vty <number>` | Clear a specified VTY line |

## Troubleshooting Rotary Groups

### Incorrect Telnet TCP Port

Assume the following configuration:

```text
line vty 5 9
 rotary 2
 transport input telnet
```

A client attempts the following connection:

```text
R2# telnet 192.0.2.1 3001
```

However, TCP port 3001 targets rotary group 1.

The correct TCP port for rotary group 2 is 3002:

```text
R2# telnet 192.0.2.1 3002
```

### Missing SSH Port Mapping

Assume the following configuration:

```text
line vty 5 9
 rotary 1
 transport input ssh
```

A client attempts to establish an SSH connection to TCP port 2222.

However, assigning lines to rotary group 1 does not automatically associate TCP port 2222 with that group.

The following global configuration command is required:

```text
R1(config)# ip ssh port 2222 rotary 1
```

### Mismatched Rotary Group Numbers

Assume the following configuration:

```text
ip ssh port 2222 rotary 2
!
line vty 5 9
 rotary 1
 transport input ssh
```

TCP port 2222 targets rotary group 2, but VTY lines 5 through 9 belong to rotary group 1.

To correct the mapping:

```text
R1(config)# no ip ssh port 2222 rotary 2
R1(config)# ip ssh port 2222 rotary 1
```

### Incorrect Transport Protocol

Assume the following configuration:

```text
ip ssh port 2222 rotary 1
!
line vty 5 9
 rotary 1
 transport input telnet
```

Although TCP port 2222 is associated with rotary group 1, the lines do not permit incoming SSH connections.

To permit SSH:

```text
R1(config)# line vty 5 9
R1(config-line)# transport input ssh
```

A rotary port mapping does not override line-level transport restrictions.

### Exhausted Rotary Group

Assume rotary group 1 contains VTY lines 5 through 9.

If all five lines are occupied, a new connection targeting rotary group 1 cannot obtain a line.

Free VTY lines outside rotary group 1 do not increase its available capacity.

Possible solutions include:

- Terminating an existing session.
- Clearing an abandoned line.
- Adding more VTY lines to the rotary group.

### Standard Port Remains Accessible

Assume an administrator configures:

```text
R1(config)# ip ssh port 2222 rotary 1
```

The administrator expects TCP port 22 to become unavailable, but ordinary SSH connections to TCP port 22 still succeed.

This is expected behavior.

The command adds a custom SSH port mapping rather than replacing the standard SSH port.

Additional access restrictions are required to prevent connections to TCP port 22.

### Troubleshooting Summary

| Problem | Potential Cause |
|---|---|
| Telnet rotary connection fails | Incorrect TCP port calculation |
| Custom SSH connection fails | Missing `ip ssh port` mapping |
| Connection targets the wrong group | Mismatched rotary group numbers |
| SSH connection rejected | `transport input` does not permit SSH |
| Telnet connection rejected | `transport input` does not permit Telnet |
| Connection rejected from a particular host | Access-class restriction |
| Connection rejected through a VRF | VRF-aware access restriction |
| Connection fails during authentication | Incorrect authentication configuration |
| New connection cannot obtain a line | All eligible lines in the rotary group are occupied |
| Standard management port remains accessible | Rotary configuration does not disable the standard port |
| Reverse Telnet or SSH starts a local CLI | EXEC process not disabled on the asynchronous line |

## Security Considerations

Using a nonstandard TCP port can reduce exposure to automated scanning that targets standard management ports.

However, changing the destination TCP port does not provide strong security by itself.

Rotary groups and custom TCP ports should be combined with appropriate security mechanisms:

- SSH encryption.
- Strong authentication.
- AAA authorization and accounting.
- IPv4 and IPv6 access classes.
- VRF-aware access restrictions.
- Management network isolation.
- Session timeouts.
- Login attack mitigation.

A custom Telnet TCP port does not encrypt Telnet traffic.

A custom SSH TCP port provides the same fundamental SSH security mechanisms as the standard SSH TCP port.

Rotary groups should primarily be viewed as a line-selection and management-policy mechanism, rather than an independent security mechanism.

## Key Concepts

- A rotary group is a logical grouping of terminal lines.
- The `rotary` line configuration command assigns a line to a rotary group.
- Multiple lines can belong to the same rotary group.
- Rotary group numbers are independent of absolute and relative line numbers.
- Connections targeting a rotary group select eligible available lines within that group.
- Telnet rotary groups use TCP port 3000 + rotary group number.
- Reverse Telnet targets a specific line using TCP port 2000 + absolute line number.
- SSH rotary groups require an explicit TCP port mapping.
- `ip ssh port <port-number> rotary <group-number>` associates an SSH TCP port with a rotary group.
- SSH rotary group port numbers are not automatically calculated.
- The same rotary group can support both Telnet and SSH.
- Telnet and SSH connections targeting the same rotary group share its pool of lines.
- Rotary group membership does not inherently prevent normal VTY line allocation.
- Configuring a rotary TCP port does not automatically disable the standard Telnet or SSH TCP port.
- Different rotary groups can use different AAA method lists, access classes, timeouts, and other line settings.
- Rotary group selection does not bypass line-level authentication, authorization, transport, or access restrictions.
- If all eligible lines in a rotary group are occupied, lines outside the group are not automatically used as a fallback.
- `show line` displays rotary group membership in the `Roty` column.
- `show users` displays the lines allocated to active user sessions.
- `show tcp brief all` can verify the local TCP ports used by management connections.
- Rotary groups provide line selection, not encryption or authentication.

## References

- [Cisco IOS Dial Technologies Command Reference - rotary](https://www.cisco.com/c/en/us/td/docs/ios/dial/command/reference/dia-cr-book/dia_p3.html)
- [Cisco IOS Secure Shell Configuration Guide - SSH Terminal-Line Access](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_ssh/configuration/15-mt/sec-usr-ssh-15-mt-book/sec-ssh-term-line.html)
- [Cisco IOS XE Security and VPN Configuration Guide - Reverse SSH Enhancements](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-rev-ssh-enhanmt-0.html)
- [Cisco - Troubleshooting Dialup Connections](https://www.cisco.com/en/US/docs/internetworking/troubleshooting/guide/tr1916.html)
- [Cisco IOS XE Software Hardening Guide](https://sec.cloudapps.cisco.com/security/center/resources/IOS_XE_hardening)
