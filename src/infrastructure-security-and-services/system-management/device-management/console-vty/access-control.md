# Session Timeouts and Access Restrictions

Cisco IOS XE provides several mechanisms for controlling the duration, availability, and accessibility of terminal sessions.

These mechanisms can be used to:

- Terminate inactive administrative sessions.
- Limit the maximum duration of administrative sessions.
- Detect and terminate abandoned TCP connections.
- Restrict the number of outgoing terminal sessions.
- Restrict incoming management connections based on source IP addresses.
- Restrict outgoing terminal connections based on destination IP addresses.
- Apply different access restrictions to different VRFs.

## EXEC Timeout

The `exec-timeout` command specifies how long the EXEC process waits for user input before terminating an inactive EXEC session.

The syntax is:

```text
exec-timeout <minutes> [seconds]
```

For example:

```text
R1(config)# line vty 0 15
R1(config-line)# exec-timeout 5 30
```

An inactive EXEC session is terminated after five minutes and 30 seconds.

The default EXEC timeout is 10 minutes.

The EXEC timeout is based on user input, not simply whether the underlying TCP connection is active.

For example, receiving unsolicited syslog messages does not constitute user input and does not reset the EXEC inactivity timer.

### Disabling the EXEC Timeout

The EXEC timeout can be disabled using either of the following commands:

```text
R1(config-line)# exec-timeout 0 0
```

```text
R1(config-line)# no exec-timeout
```

An EXEC session can remain inactive indefinitely without being terminated by the EXEC inactivity timer.

Other mechanisms, such as absolute timeouts and TCP keepalives, can still terminate the session.

To restore the default EXEC timeout:

```text
R1(config-line)# default exec-timeout
```

Disabling the EXEC timeout is convenient in lab environments but is generally undesirable in production networks.

## Session Timeout

The `session-timeout` command specifies how long a terminal connection can remain inactive before it is closed.

The syntax is:

```text
session-timeout <minutes> [output]
```

For example:

```text
R1(config)# line vty 0 15
R1(config-line)# session-timeout 5
```

By default, the session inactivity timer is based only on user input.

### The output Keyword

The optional `output` keyword causes both input and output traffic to reset the session inactivity timer.

```text
R1(config-line)# session-timeout 5 output
```

In this example, the connection must experience five minutes of inactivity in both directions before the session timeout occurs.

| Configuration | Activity That Resets the Timer |
|---|---|
| `session-timeout 5` | User input |
| `session-timeout 5 output` | User input or output traffic |

For example, consider a terminal connection in which the remote device continuously generates command output, but the user does not enter any commands.

- Without the `output` keyword, the session inactivity timer can expire.
- With the `output` keyword, the output traffic resets the session inactivity timer.

By default, no session timeout is configured.

To remove the session timeout:

```text
R1(config-line)# no session-timeout
```

### EXEC Timeout vs. Session Timeout

The `exec-timeout` and `session-timeout` commands control different aspects of terminal inactivity.

Consider the following topology:

```text
PC1 ---- SSH ----> R1 ---- Telnet ----> R2
```

PC1 establishes an SSH connection to R1 and then initiates a Telnet connection from R1 to R2.

On a VTY line:

- The session timeout can close the outgoing terminal connection and return the user to the local EXEC prompt.
- The EXEC timeout terminates the inactive local EXEC session.

On physical lines, such as console, AUX, and asynchronous TTY lines, a session timeout logs the user out and returns the line to an idle state.

The behavior of `session-timeout` therefore differs between physical and virtual lines.

| Characteristic | EXEC Timeout | Session Timeout |
|---|---|---|
| Command | `exec-timeout` | `session-timeout` |
| Default | 10 minutes | Disabled |
| Time unit | Minutes and seconds | Minutes |
| Based on inactivity | Yes | Yes |
| User input resets timer | Yes | Yes |
| Output can reset timer | No | Yes, with `output` |
| Effect on VTY | Terminates inactive EXEC session | Closes remote terminal connection and returns to EXEC |

Both mechanisms can be configured on the same line.

## Absolute Timeout

The `absolute-timeout` command limits the total duration of a terminal session, regardless of activity.

The syntax is:

```text
absolute-timeout <minutes>
```

For example:

```text
R1(config)# line vty 0 15
R1(config-line)# absolute-timeout 60
```

The session is terminated after 60 minutes, even if the user has been continuously entering commands.

Unlike the EXEC timeout and session timeout, the absolute timeout is not an inactivity timer.

User input and output do not reset it.

By default, no absolute timeout is configured.

To remove the absolute timeout:

```text
R1(config-line)# no absolute-timeout
```

### Logout Warning

The `logout-warning` command warns users before a session is terminated by the absolute timeout.

The syntax is:

```text
logout-warning [seconds]
```

For example:

```text
R1(config)# line vty 0 15
R1(config-line)# absolute-timeout 60
R1(config-line)# logout-warning 30
```

Users receive a warning 30 seconds before the 60-minute absolute timeout expires.

The `logout-warning` command does not extend the session duration.

It also does not provide a warning for the EXEC inactivity timeout.

### Combining EXEC and Absolute Timeouts

Multiple timeout mechanisms can be configured simultaneously.

```text
R1(config)# line vty 0 15
R1(config-line)# exec-timeout 10 0
R1(config-line)# absolute-timeout 60
R1(config-line)# logout-warning 30
```

This configuration provides the following behavior:

- An inactive EXEC session is terminated after 10 minutes.
- An active session is terminated after a maximum of 60 minutes.
- A warning is displayed 30 seconds before the absolute timeout expires.

```text
Session Start                                  Absolute Timeout
      |                                               |
      |<---------------- 60 Minutes ----------------->|
      |                                               |
      +---- User Activity ---- User Activity ---------X
                  ^                  ^
                  |                  |
           EXEC Timer Reset   EXEC Timer Reset
```

User activity resets the EXEC inactivity timer but does not reset the absolute timeout timer.

## TCP Keepalives

A remote device can lose connectivity without properly terminating an established TCP connection.

For example:

```text
PC1 -------- Network -------- R1
           SSH Established
```

If PC1 suddenly loses power or network connectivity, it might not transmit a TCP FIN or RST to R1.

Without a mechanism to detect the failure, R1 can retain the abandoned TCP connection and its associated VTY line.

**TCP keepalives** allow IOS XE to probe remote TCP endpoints and detect connections that are no longer functional.

### Incoming TCP Keepalives

The `service tcp-keepalives-in` global configuration command enables TCP keepalives for incoming TCP connections.

```text
R1(config)# service tcp-keepalives-in
```

For example, if a remote SSH client becomes unreachable, R1 can detect the failure and clear the abandoned connection.

```text
PC1 -------- SSH --------> R1
     <---- TCP Keepalive --
```

### Outgoing TCP Keepalives

The `service tcp-keepalives-out` global configuration command enables TCP keepalives for outgoing TCP connections.

```text
R1(config)# service tcp-keepalives-out
```

For example, if R1 initiates a Telnet connection to R2 and R2 becomes unreachable, R1 can detect the failure and clear the connection.

```text
R1 -------- Telnet -------> R2
   ------ TCP Keepalive --->
```

Incoming and outgoing refer to which endpoint initiated the TCP connection, not the direction of individual TCP packets.

Both mechanisms can be enabled simultaneously.

```text
R1(config)# service tcp-keepalives-in
R1(config)# service tcp-keepalives-out
```

These mechanisms are disabled by default.

### TCP Keepalive Parameters

The `ip tcp keepalive` command allows the TCP keepalive interval and retry count to be modified for TTY and VTY applications.

```text
R1(config)# ip tcp keepalive interval 30
R1(config)# ip tcp keepalive retries 3
```

| Parameter | Default | Range |
|---|---:|---:|
| Keepalive interval | 60 seconds | 1-7200 seconds |
| Keepalive retries | 4 | 1-10 |

The keepalive parameters can be configured independently of whether keepalive probing has been enabled.

The `service tcp-keepalives-in` and `service tcp-keepalives-out` commands enable the actual probing.

### TCP Keepalives vs. Inactivity Timeouts

TCP keepalives and inactivity timeouts serve different purposes.

A user can leave an SSH session idle while the SSH client remains connected and functional.

In this situation:

- The client can respond to TCP keepalive probes.
- The TCP connection remains functional.
- The EXEC inactivity timer can still expire.

Conversely, if the client loses connectivity:

- TCP keepalive probes go unanswered.
- IOS XE can detect the failed connection and terminate it.

TCP keepalives detect connection failure, not administrative inactivity.

## Timeout Mechanism Summary

| Mechanism | Purpose | Configuration Mode |
|---|---|---|
| `exec-timeout` | Terminate inactive EXEC sessions | Line |
| `session-timeout` | Close inactive terminal connections | Line |
| `absolute-timeout` | Limit total session duration | Line |
| `logout-warning` | Warn of an impending absolute timeout | Line |
| `service tcp-keepalives-in` | Detect abandoned incoming TCP connections | Global |
| `service tcp-keepalives-out` | Detect abandoned outgoing TCP connections | Global |
| `ip tcp keepalive` | Modify TCP keepalive parameters | Global |

## Session Limits

The `session-limit` command restricts the number of outgoing terminal sessions that can be established from an individual line.

For example:

```text
R1(config)# line vty 0 15
R1(config-line)# session-limit 2
```

A user connected through one of these VTY lines can establish a maximum of two simultaneous outgoing terminal sessions.

```text
                            +---- Telnet ----> R2
PC1 ---- SSH ----> R1 ------+
                            +---- SSH -------> R3

                            X---- Telnet ----> R4
                                 Limit Reached
```

The session limit applies independently to each configured line.

It does not limit the total number of incoming management connections to the device.

To remove the configured session limit:

```text
R1(config-line)# no session-limit
```

The number of simultaneous incoming interactive CLI sessions is primarily determined by the number of available and eligible VTY lines.

## IPv4 VTY Access Classes

The `access-class` line configuration command applies an IPv4 ACL to terminal connections.

The basic syntax is:

```text
access-class <acl-name | acl-number> {in | out}
```

An access class can be applied in either direction:

- **Inbound (`in`):** Restricts incoming terminal connections to the local device.
- **Outbound (`out`):** Restricts outgoing terminal connections initiated from the line.

```text
              Inbound               Outbound
             Access Class          Access Class
                  |                     |
                  v                     v
PC1 ---- SSH ----> R1 ---- Telnet ----> R2
```

The directions are relative to the local device and its terminal sessions.

An outbound access class does not filter packets transmitted back to the client of an incoming SSH or Telnet session.

### Access Classes vs. Interface ACLs

Although both mechanisms use ACLs, they serve different purposes.

| Characteristic | Access Class | Interface ACL |
|---|---|---|
| IPv4 command | `access-class` | `ip access-group` |
| IPv6 command | `ipv6 access-class` | `ipv6 traffic-filter` |
| Applied to | Terminal line | Network interface |
| Filters terminal connections | Yes | If matched by the ACL |
| Filters ordinary transit traffic | No | Yes |
| Independent of physical ingress interface | Yes | No |

An access class restricts access to terminal services rather than filtering all IP traffic traversing an interface.

For example, denying a host using an inbound VTY access class does not inherently prevent that host from pinging the router or forwarding traffic through it.

## Inbound IPv4 Access Classes

An inbound IPv4 access class restricts which remote hosts can establish incoming terminal connections.

When a standard IPv4 ACL is used, the ACL matches the remote client's source IP address.

For example:

```text
R1(config)# ip access-list standard VTY-IN
R1(config-std-nacl)# permit 192.0.2.0 0.0.0.255
!
R1(config)# line vty 0 15
R1(config-line)# access-class VTY-IN in
```

Only clients with source IP addresses in the 192.0.2.0/24 subnet are permitted by the access class.

```text
PC1 (192.0.2.100) ------ SSH ------> R1
                                  Permitted

PC2 (198.51.100.100) --- SSH ---X   R1
                                  Denied
```

All other IPv4 source addresses are denied by the ACL's implicit deny.

Permitting a client in the ACL does not bypass authentication, authorization, or transport protocol restrictions.

### Multiple Physical Interfaces

An inbound VTY access class applies regardless of which physical interface receives the connection.

```text
                              R1
                     +------------------+
PC1 ---------------->| G1               |
                     |         VTY 0-15 |
PC2 ---------------->| G2               |
                     +------------------+
```

Both clients are subject to the access class configured on the VTY line selected for their connection.

To restrict management access through a particular physical interface, additional mechanisms such as interface ACLs or Management Plane Protection (MPP) can be used.

### Applying Access Classes to All VTY Lines

Consider the following configuration on a router with 16 VTY lines:

```text
R1(config)# line vty 0 4
R1(config-line)# access-class VTY-IN in
```

Only VTY lines 0 through 4 have the access class configured.

VTY lines 5 through 15 do not inherit the configuration.

Depending on line allocation and the remaining line configurations, unrestricted VTY lines can provide an unintended management access path.

To apply the restriction to all 16 lines:

```text
R1(config)# line vty 0 15
R1(config-line)# access-class VTY-IN in
```

Always verify the configurations of all available VTY lines.

## Outbound IPv4 Access Classes

An outbound IPv4 access class restricts the remote destinations to which users can establish outgoing terminal connections.

When a standard IPv4 ACL is used with an outbound access class, the address in the ACL is interpreted as the **destination IP address**.

This differs from the normal interpretation of a standard IPv4 ACL applied to a network interface, where it matches the source IP address.

For example:

```text
R1(config)# ip access-list standard VTY-OUT
R1(config-std-nacl)# permit 198.51.100.0 0.0.0.255
!
R1(config)# line vty 0 15
R1(config-line)# access-class VTY-OUT out
```

Users connected through these VTY lines can initiate outgoing terminal connections only to addresses in the 198.51.100.0/24 subnet.

```text
                          +---- SSH ----> R2 (198.51.100.2)
                          |               Permitted
PC1 ---- SSH ----> R1 ----+
                          |               Denied
                          +---- SSH ---X  R3 (203.0.113.3)
```

### Standard ACL Address Matching Summary

| Application | Address Matched |
|---|---|
| `ip access-group ... in` | Source IP address |
| `ip access-group ... out` | Source IP address |
| `access-class ... in` | Remote client's source IP address |
| `access-class ... out` | Remote server's destination IP address |

The interpretation of a standard ACL applied using `access-class ... out` is an important distinction.

### Outbound Restrictions on the Console

Outgoing terminal connections initiated from the console are subject to the console line's configuration, not the VTY line configuration.

For example:

```text
R1(config)# line console 0
R1(config-line)# access-class VTY-OUT out
```

This applies the outgoing access restriction to terminal connections initiated from the console.

It does not restrict physical access to the console itself.

## Extended IPv4 Access Classes

Extended IPv4 ACLs can also be used as VTY access classes on supported IOS XE platforms.

For example, the following ACL permits SSH from the 192.0.2.0/24 subnet and Telnet from the 198.51.100.0/24 subnet:

```text
R1(config)# ip access-list extended VTY-IN
R1(config-ext-nacl)# permit tcp 192.0.2.0 0.0.0.255 any eq 22
R1(config-ext-nacl)# permit tcp 198.51.100.0 0.0.0.255 any eq 23
!
R1(config)# line vty 0 15
R1(config-line)# access-class VTY-IN in
R1(config-line)# transport input telnet ssh
```

| Client Source Address | Protocol | Result |
|---|---|---|
| 192.0.2.100 | SSH | Permitted |
| 192.0.2.100 | Telnet | Denied |
| 198.51.100.100 | SSH | Denied |
| 198.51.100.100 | Telnet | Permitted |
| 203.0.113.100 | SSH | Denied |

Both the transport protocol configuration and the access class must permit the connection.

### Destination Address Matching

Historically, IOS evaluated incoming VTY access classes without matching the actual local destination IP address.

Consequently, extended ACLs applied as incoming VTY access classes commonly specified `any` as the destination address.

Support for matching specific local destination addresses was added through Cisco bug fix CSCuw89081.

When configuring destination-specific VTY access restrictions, verify the behavior of the IOS XE release and platform.

If the objective is simply to restrict management clients by source address, a standard IPv4 ACL is sufficient.

## IPv6 VTY Access Classes

IPv6 management connections are restricted using the `ipv6 access-class` command.

The basic syntax is:

```text
ipv6 access-class <acl-name> {in | out}
```

Unlike IPv4, IOS XE IPv6 ACLs are inherently extended ACLs.

An IPv4 access class does not provide equivalent restrictions for IPv6 management connections.

### Inbound IPv6 Access Class

The following example permits incoming SSH connections from the 2001:db8:1::/64 prefix:

```text
R1(config)# ipv6 access-list VTY6-IN
R1(config-ipv6-acl)# permit tcp 2001:db8:1::/64 any eq 22
!
R1(config)# line vty 0 15
R1(config-line)# ipv6 access-class VTY6-IN in
R1(config-line)# transport input ssh
```

Incoming IPv6 SSH connections from other source prefixes are denied.

Alternatively, the ACL can permit all IPv6 traffic from the specified source prefix, leaving protocol selection to the `transport input` command.

```text
R1(config)# ipv6 access-list VTY6-IN
R1(config-ipv6-acl)# permit ipv6 2001:db8:1::/64 any
```

The ACL is still applied only to terminal connections, not ordinary IPv6 transit traffic.

### Outbound IPv6 Access Class

The following example restricts outgoing IPv6 terminal connections to destinations within the 2001:db8:2::/64 prefix:

```text
R1(config)# ipv6 access-list VTY6-OUT
R1(config-ipv6-acl)# permit ipv6 any 2001:db8:2::/64
!
R1(config)# line vty 0 15
R1(config-line)# ipv6 access-class VTY6-OUT out
```

### Dual-Stack Access Restrictions

IPv4 and IPv6 access classes can coexist on the same VTY lines.

```text
R1(config)# ip access-list standard VTY4-IN
R1(config-std-nacl)# permit 192.0.2.0 0.0.0.255
!
R1(config)# ipv6 access-list VTY6-IN
R1(config-ipv6-acl)# permit ipv6 2001:db8:1::/64 any
!
R1(config)# line vty 0 15
R1(config-line)# access-class VTY4-IN in
R1(config-line)# ipv6 access-class VTY6-IN in
R1(config-line)# transport input ssh
```

The IPv4 access class controls incoming IPv4 terminal connections.

The IPv6 access class controls incoming IPv6 terminal connections.

Both address families must be considered when securing a dual-stack device.
## VRF-Aware VTY Access

VTY access restrictions require special consideration when a device uses Virtual Routing and Forwarding (VRF) instances.

Consider the following topology:

```text
PC1 (192.0.2.100) ------ G1 [Global] ----+
                                         |
PC2 (192.0.2.100) ------ G2 [MGMT] ------+ R1
                                         |
PC3 (192.0.2.100) ------ G3 [CUSTOMER] --+
```

Because VRFs maintain separate routing tables, identical IP addresses can exist in multiple VRFs.

An administrator might want to permit management access through the MGMT VRF while denying access through the CUSTOMER VRF, even though the clients use identical IP addresses.

IOS XE provides two mechanisms for VRF-aware VTY access:

- The `vrf-also` keyword.
- The `vrfname` keyword.

### Default Access-Class Behavior

Consider the following configuration:

```text
R1(config)# ip access-list standard VTY-IN
R1(config-std-nacl)# permit 192.0.2.0 0.0.0.255
!
R1(config)# line vty 0 15
R1(config-line)# access-class VTY-IN in
```

Without a VRF-related keyword, the inbound IPv4 access class permits qualifying connections associated with the global routing table.

Incoming connections associated with a VRF are rejected, even if their source IP addresses are permitted by the ACL.

| Client | Routing Table | Result |
|---|---|---|
| PC1 | Global | Permitted |
| PC2 | MGMT | Denied |
| PC3 | CUSTOMER | Denied |

This is a common cause of remote management connectivity problems after moving a management interface into a VRF.

### The vrf-also Keyword

The `vrf-also` keyword allows an inbound access class to accept connections associated with VRFs in addition to the global routing table.

```text
R1(config)# line vty 0 15
R1(config-line)# access-class VTY-IN in vrf-also
```

The same ACL is used for connections associated with the global routing table and any VRF.

| Client | Routing Table | Result |
|---|---|---|
| PC1 | Global | Permitted |
| PC2 | MGMT | Permitted |
| PC3 | CUSTOMER | Permitted |

The `vrf-also` keyword does not bypass the ACL.

The source IP address must still be permitted by the ACL, regardless of the routing table associated with the connection.

The limitation of `vrf-also` is that it does not distinguish between individual VRFs.

### The vrfname Keyword

The `vrfname` keyword associates an inbound access class with a specific VRF.

This feature was introduced in Cisco IOS XE Release 16.8.1.

The syntax is:

```text
access-class <acl-name | acl-number> in vrfname <vrf-name>
```

For example:

```text
R1(config)# line vty 0 15
R1(config-line)# access-class VTY-IN in vrfname MGMT
```

The VTY access class permits qualifying incoming connections associated with the MGMT VRF.

Assuming no additional inbound access classes are configured:

| Client | Routing Table | Result |
|---|---|---|
| PC1 | Global | Denied |
| PC2 | MGMT | Permitted |
| PC3 | CUSTOMER | Denied |

Unlike `vrf-also`, the `vrfname` keyword allows the administrator to select a specific VRF.

### Different ACLs for Different VRFs

Multiple VRF-specific access classes can coexist on the same VTY lines.

For example:

```text
R1(config)# ip access-list standard MGMT-IN
R1(config-std-nacl)# permit 192.0.2.0 0.0.0.255
!
R1(config)# ip access-list standard CUSTOMER-IN
R1(config-std-nacl)# permit host 198.51.100.100
!
R1(config)# line vty 0 15
R1(config-line)# access-class MGMT-IN in vrfname MGMT
R1(config-line)# access-class CUSTOMER-IN in vrfname CUSTOMER
```

In this example:

- Connections associated with the MGMT VRF are evaluated against MGMT-IN.
- Connections associated with the CUSTOMER VRF are evaluated against CUSTOMER-IN.
- Connections associated with other VRFs are denied.

Each VRF can have its own access policy.

Multiple VRFs cannot be specified in a single `access-class` command. A separate command must be configured for each VRF.

Only one inbound VRF-specific IPv4 access class can be configured per VRF on a line. Configuring another IPv4 access class for the same VRF replaces the previous one.

The `vrf-also` and `vrfname` mechanisms are mutually exclusive.

### VRF-Aware IPv6 Access Classes

IPv6 also supports VRF-specific access classes using the `vrfname` keyword.

For example:

```text
R1(config)# ipv6 access-list MGMT6-IN
R1(config-ipv6-acl)# permit ipv6 2001:db8:1::/64 any
!
R1(config)# line vty 0 15
R1(config-line)# ipv6 access-class MGMT6-IN in vrfname MGMT
```

Different IPv6 ACLs can be associated with different VRFs.

```text
R1(config)# line vty 0 15
R1(config-line)# ipv6 access-class MGMT6-IN in vrfname MGMT
R1(config-line)# ipv6 access-class CUSTOMER6-IN in vrfname CUSTOMER
```

IPv4 and IPv6 VRF-specific access classes can coexist.

### IPv4 VRF-Aware Access Class Summary

| Configuration | Global Routing Table | VRFs |
|---|---|---|
| No inbound access class | No ACL restriction | No ACL restriction |
| `access-class ACL in` | Evaluated against ACL | Denied |
| `access-class ACL in vrf-also` | Evaluated against ACL | All VRFs evaluated against ACL |
| `access-class ACL in vrfname MGMT` | Denied unless separately permitted | MGMT evaluated against ACL |
| Multiple `vrfname` commands | Denied unless separately permitted | Each configured VRF evaluated against its respective ACL |

These results assume no other management access restrictions are configured.

## Login Attack Mitigation

In addition to access classes, IOS XE provides mechanisms for mitigating repeated login attempts.

### Login Block-For

The `login block-for` global configuration command temporarily blocks remote login attempts when a specified number of login failures occurs within a specified interval.

The syntax is:

```text
login block-for <seconds> attempts <number> within <seconds>
```

For example:

```text
R1(config)# login block-for 120 attempts 3 within 60
```

If three failed login attempts occur within 60 seconds, remote login attempts are blocked for 120 seconds.

The period during which remote login attempts are blocked is called **quiet mode**.

Existing authenticated sessions are not terminated merely because the device enters quiet mode.

### Quiet Mode Access Class

The `login quiet-mode access-class` command specifies an ACL identifying trusted hosts that are exempt from quiet mode restrictions.

For example:

```text
R1(config)# ip access-list standard QUIET-MODE
R1(config-std-nacl)# permit host 192.0.2.100
!
R1(config)# login block-for 120 attempts 3 within 60
R1(config)# login quiet-mode access-class QUIET-MODE
```

When the device enters quiet mode, the trusted management host at 192.0.2.100 can continue attempting to log in.

The quiet mode access class does not bypass ordinary VTY access restrictions or authentication requirements.

### Login Delay

The `login delay` command introduces a delay between successive login attempts.

```text
R1(config)# login delay 3
```

This slows repeated login attempts.

Configuring `login block-for` automatically introduces a one-second login delay unless another delay is explicitly configured.

### Login Event Logging

Successful and unsuccessful login attempts can be logged using the following global configuration commands:

```text
R1(config)# login on-failure log
R1(config)# login on-success log
```

These commands improve visibility into administrative login activity.

Logging every successful login can generate additional system messages, depending on the management activity of the device.

### Login Verification

The `show login` command displays login attack mitigation settings and the current login state.

```text
R1# show login
```

The `show login failures` command displays information about failed login attempts.

```text
R1# show login failures
```

These commands are useful when an otherwise valid management connection is rejected because the device has entered quiet mode.

## Complete Configuration Example

Consider a dual-stack router with the following management requirements:

- Permit only SSH connections.
- Permit management access only through the MGMT VRF.
- Permit IPv4 management clients from 192.0.2.0/24.
- Permit IPv6 management clients from 2001:db8:1::/64.
- Terminate inactive EXEC sessions after five minutes.
- Limit administrative sessions to a maximum of 60 minutes.
- Warn users 30 seconds before the absolute timeout expires.
- Prevent users from initiating outgoing terminal connections.
- Enable TCP keepalives.
- Temporarily block repeated failed login attempts.

Assume the MGMT VRF, SSH server, and local user database have already been configured.

```text
R1(config)# ip access-list standard MGMT4-IN
R1(config-std-nacl)# permit 192.0.2.0 0.0.0.255
!
R1(config)# ipv6 access-list MGMT6-IN
R1(config-ipv6-acl)# permit ipv6 2001:db8:1::/64 any
!
R1(config)# service tcp-keepalives-in
R1(config)# service tcp-keepalives-out
R1(config)# ip tcp keepalive interval 30
R1(config)# ip tcp keepalive retries 3
!
R1(config)# login block-for 120 attempts 3 within 60
R1(config)# login on-failure log
R1(config)# login on-success log
!
R1(config)# line vty 0 15
R1(config-line)# login local
R1(config-line)# transport input ssh
R1(config-line)# transport output none
R1(config-line)# transport preferred none
R1(config-line)# access-class MGMT4-IN in vrfname MGMT
R1(config-line)# ipv6 access-class MGMT6-IN in vrfname MGMT
R1(config-line)# exec-timeout 5 0
R1(config-line)# absolute-timeout 60
R1(config-line)# logout-warning 30
R1(config-line)# logging synchronous
```

This configuration assumes AAA is disabled and that no other inbound VTY access classes are configured.

The IPv4 and IPv6 access classes provide address-family-specific and VRF-specific access restrictions.

The transport configuration determines which terminal protocols are permitted, while the timeout and keepalive mechanisms control session lifetime and connection failure detection.

## Verification

### show line

The `show line` command displays information about available terminal lines.

```text
R1# show line
```

The output includes incoming and outgoing access class information.

Detailed line settings can be displayed by specifying a particular line:

```text
R1# show line vty 0
```

Relevant information includes:

- EXEC timeout.
- Session timeout.
- Absolute timeout.
- Incoming and outgoing access classes.
- Permitted transport protocols.
- Line status.

### show users

The `show users` command displays active user sessions.

```text
R1# show users
```

This command is useful for identifying occupied VTY lines, remote source addresses, usernames, and session inactivity.

### show access-lists

The `show access-lists` command displays configured ACLs and their available match counters.

```text
R1# show access-lists
```

A specific IPv4 ACL can be displayed using:

```text
R1# show ip access-lists MGMT4-IN
```

A specific IPv6 ACL can be displayed using:

```text
R1# show ipv6 access-list MGMT6-IN
```

ACL match counters can assist in identifying whether an access class is permitting or denying connection attempts. Counter behavior can vary between platforms and ACL applications.

### show tcp brief all

The `show tcp brief all` command displays TCP connections, including listening sockets and established connections.

```text
R1# show tcp brief all
```

This is useful for identifying established or abandoned management connections.

### show running-config all

The following command displays VTY configuration, including commands configured with their default values:

```text
R1# show running-config all | section ^line vty
```

TCP keepalive configuration can be examined using:

```text
R1# show running-config all | include keepalive
```

Login attack mitigation configuration can be examined using:

```text
R1# show running-config | include ^login
```

## Troubleshooting Access Restrictions

When a remote management connection fails, consider each of the following components:

| Potential Problem | Verification |
|---|---|
| No IP connectivity | Routing table, ping, traceroute |
| Incorrect VRF | VRF configuration and routing table |
| Transport protocol not permitted | `show line`, VTY configuration |
| IPv4 access class denies connection | `show ip access-lists` |
| IPv6 access class denies connection | `show ipv6 access-list` |
| Missing `vrf-also` or `vrfname` | VTY configuration |
| Incorrect VRF-specific ACL | VTY configuration and ACL contents |
| Inconsistent restrictions across VTY lines | `show running-config all` |
| All eligible VTY lines occupied | `show users`, `show line` |
| Authentication failure | Local user database or AAA configuration |
| Device in quiet mode | `show login` |
| Repeated login failures | `show login failures` |
| Abandoned TCP connections | `show tcp brief all`, `show users` |
| Unexpected session termination | EXEC, session, and absolute timeout settings |

### Troubleshooting Scenario: Management VRF

Assume R1 has a management interface belonging to the MGMT VRF.

A remote administrator can successfully ping the management interface but cannot establish an SSH session.

The VTY configuration is:

```text
line vty 0 15
 login local
 transport input ssh
 access-class MGMT-IN in
```

The administrator's source IP address is permitted by MGMT-IN.

However, the inbound access class does not permit connections associated with a VRF by default.

One solution is to configure:

```text
R1(config)# line vty 0 15
R1(config-line)# access-class MGMT-IN in vrf-also
```

Alternatively, to specifically permit connections associated with the MGMT VRF:

```text
R1(config)# line vty 0 15
R1(config-line)# no access-class MGMT-IN in
R1(config-line)# access-class MGMT-IN in vrfname MGMT
```

The second solution provides more granular control because it does not extend the ACL's permissions to every VRF.

### Troubleshooting Scenario: Unexpected Session Termination

Assume the following configuration:

```text
line vty 0 15
 exec-timeout 0 0
 absolute-timeout 30
```

An administrator's SSH session is terminated after 30 minutes, even though the administrator is actively entering commands.

The cause is the absolute timeout.

Disabling the EXEC inactivity timeout does not disable other timeout mechanisms.

To remove the absolute timeout:

```text
R1(config)# line vty 0 15
R1(config-line)# no absolute-timeout
```

### Troubleshooting Scenario: Outgoing Access Class

Assume the following configuration:

```text
ip access-list standard VTY-OUT
 permit 192.0.2.0 0.0.0.255
!
line vty 0 15
 transport input ssh
 transport output ssh
 access-class VTY-OUT out
```

An administrator connected to R1 through SSH cannot initiate an SSH connection to 198.51.100.2.

However, R1 can successfully ping 198.51.100.2.

The outgoing access class denies the destination IP address of the attempted SSH connection.

The access class does not prohibit ICMP echo requests because they are not outgoing terminal connections.

## Verification Command Summary

| Command | Purpose |
|---|---|
| `show line` | Display line information |
| `show line vty <number>` | Display detailed VTY line settings |
| `show users` | Display active user sessions |
| `show sessions` | Display outgoing terminal sessions |
| `show ip access-lists` | Display IPv4 ACLs |
| `show ipv6 access-list` | Display IPv6 ACLs |
| `show tcp brief all` | Display TCP connections and listening sockets |
| `show login` | Display login protection configuration and state |
| `show login failures` | Display failed login attempts |
| `show running-config all` | Display configuration, including defaults |
| `clear line vty <number>` | Clear a specified VTY line |

## Key Concepts

- `exec-timeout` controls how long an EXEC session can remain inactive.
- The default EXEC timeout is 10 minutes.
- `exec-timeout 0 0` disables the EXEC inactivity timeout.
- `session-timeout` controls terminal connection inactivity.
- The `output` keyword allows both input and output to reset the session inactivity timer.
- On a VTY line, a session timeout can return the user to the local EXEC prompt rather than terminate the entire EXEC session.
- `absolute-timeout` limits total session duration regardless of activity.
- `logout-warning` warns users before an absolute timeout expires.
- TCP keepalives detect failed TCP connections rather than inactive users.
- `service tcp-keepalives-in` enables keepalives for incoming TCP connections.
- `service tcp-keepalives-out` enables keepalives for outgoing TCP connections.
- The default configurable TCP keepalive interval is 60 seconds, and the default retry count is 4.
- `session-limit` restricts the number of outgoing terminal sessions per line.
- `access-class ... in` restricts incoming IPv4 terminal connections.
- `access-class ... out` restricts outgoing IPv4 terminal connections.
- A standard IPv4 ACL used as an outbound access class matches the remote destination address.
- IPv6 terminal connections are restricted using `ipv6 access-class`.
- IPv4 and IPv6 access classes can coexist on the same lines.
- An ordinary inbound IPv4 access class does not permit VRF-associated connections by default.
- The `vrf-also` keyword extends an inbound access class to connections associated with any VRF.
- The `vrfname` keyword associates an inbound access class with a specific VRF.
- Multiple VRF-specific access classes can coexist on the same VTY lines.
- `vrf-also` and `vrfname` are mutually exclusive mechanisms.
- `login block-for` temporarily blocks remote login attempts after repeated authentication failures.
- `login quiet-mode access-class` identifies trusted hosts exempt from quiet mode restrictions.
- Access classes restrict terminal connections, not ordinary transit traffic.
- Management access restrictions should be applied consistently across all relevant VTY lines.

## References

- [Cisco IOS XE Software Hardening Guide](https://sec.cloudapps.cisco.com/security/center/resources/IOS_XE_hardening)
- [Cisco IOS Terminal Services Command Reference - absolute-timeout through rxspeed](https://www.cisco.com/c/en/us/td/docs/ios/termserv/command/reference/tsv_book/tsv_a1.html)
- [Cisco IOS Terminal Services Command Reference - service exec-callback through x3](https://www.cisco.com/c/en/us/td/docs/ios/termserv/command/reference/tsv_book/tsv_s1.html)
- [Cisco IOS IP Application Services Command Reference - ip tcp keepalive](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipapp/command/iap-cr-book/iap-i2.html)
- [Cisco IOS Security Command Reference - access-class](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/security/a1/sec-a1-cr-book/sec-cr-a2.html)
- [Cisco IOS XE 17.x - VRF Awareness Access Class Line](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/application-services/b-application-services/m-bba-vrf-aware-access-class-line.html)
- [Cisco IOS XE 17.x - IPv6 Access Control Lists](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_ip6-acls-xe.html)
