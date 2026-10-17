
# TACACS+

**Terminal Access Controller Access-Control System Plus (TACACS+)** is a protocol used to provide centralized **Authentication, Authorization, and Accounting (AAA)** for network device administration.

TACACS+ allows Cisco IOS XE routers and switches to delegate administrative access decisions to external AAA servers.

It is commonly used to:

- Authenticate administrators connecting through SSH or the console.
- Assign EXEC privilege levels.
- Authorize individual CLI commands.
- Record administrative sessions and commands.
- Apply consistent administrative access policies across multiple devices.

TACACS+ is particularly suitable for network device administration because it supports **separate authentication, authorization, and accounting exchanges**, including granular per-command authorization.

---

## TACACS+ Characteristics

| Characteristic | TACACS+ |
|---|---|
| Transport protocol | TCP |
| Default destination port | 49 |
| AAA functions | Authentication, Authorization, Accounting |
| Authentication and authorization | Separate exchanges |
| Command authorization | Supported |
| Command accounting | Supported |
| Shared secret | Required for normal protected communication |
| Packet protection | Obfuscates most of the packet body using an MD5-based mechanism |

The network device acts as the **TACACS+ client**, while the external AAA server makes authentication and authorization decisions and receives accounting records.

For example:

```text
                  TCP Port 49
R1 -------------------------------- TACACS+ Server
   Authentication Requests
   Authorization Requests
   Accounting Records
```

The TACACS+ server maintains user accounts and authorization policies.

The router enforces the results returned by the server.

---

## TACACS+ Authentication

TACACS+ authentication verifies the identity of a user requesting access.

For example, when an administrator connects to R1 through SSH:

1. The administrator provides a username and password.
2. R1 sends a TACACS+ authentication request to the server.
3. The server validates the credentials.
4. The server returns an authentication result.
5. R1 permits or rejects the login according to the result.

A simplified exchange is:

```text
Administrator             R1                   TACACS+ Server
      |                   |                          |
      |---- SSH Login --->|                          |
      |                   |                          |
      |                   |---- Authentication ----->|
      |                   |                          |
      |                   |<------- PASS ------------|
      |                   |                          |
      |<-- Login Allowed -|                          |
```

The server can return different authentication results.

| Result | Meaning |
|---|---|
| PASS | Authentication succeeded |
| FAIL | Authentication was explicitly rejected |
| ERROR | Authentication could not be completed |
| GETUSER | Request a username |
| GETPASS | Request a password |
| GETDATA | Request additional information |

TACACS+ authentication supports multiple exchanges, allowing the server to request additional credentials or information when necessary.

### Authentication Failure vs. Server Unavailability

Consider:

```text
aaa authentication login default group tacacs+ local
```

If the TACACS+ server rejects the user's credentials, authentication fails.

If the TACACS+ method cannot complete authentication because the servers are unavailable, the router attempts local authentication.

**An explicit TACACS+ authentication rejection does not trigger local fallback.**

---

## TACACS+ Authorization

TACACS+ authorization determines which services and commands a user can access.

Unlike authentication, authorization does not verify the user's password.

Instead, the router sends information about the user and the requested service to the TACACS+ server.

The server returns an authorization decision, potentially including attributes that control the user's session.

### EXEC Authorization

When a user successfully authenticates, the router can request authorization to establish an EXEC session.

For example, the TACACS+ server might return:

```text
service=shell
priv-lvl=15
```

These attributes indicate that the user is authorized for an EXEC shell with privilege level 15.

The router can then create the EXEC session with that privilege level.

If the server assigns privilege level 5 instead, the user can enter the EXEC session with privilege level 5.


### Configuring EXEC Authorization

TACACS+ login authentication does not automatically enable EXEC authorization.

To authorize EXEC sessions through TACACS+, configure:

```text
R1(config)# aaa authorization exec default group tacacs+ local
```

After successful login authentication, the router sends a separate EXEC authorization request to the TACACS+ server.

The server can return attributes such as:

```text
service=shell
priv-lvl=15
```

The router uses the returned attributes to establish the EXEC session with the appropriate privileges.

If TACACS+ authorization is unavailable, the router falls back to local authorization.

**Authentication and EXEC authorization require separate AAA method lists.**

### Command Authorization

TACACS+ can also authorize individual commands.

For example:

```text
Administrator              R1                  TACACS+ Server
      |                    |                         |
      |--- show run ------>|                         |
      |                    |--- Authorize Command -->|
      |                    |                         |
      |                    |<-------- PASS ----------|
      |                    |                         |
      |<-- Command Output -|                         |
```

The router requests authorization before executing the command.

The TACACS+ server can permit or deny the request based on its configured policies.

For example:

| User | Command | Authorization |
|---|---|---|
| admin | `show running-config` | Permit |
| admin | `configure terminal` | Permit |
| operator | `show ip route` | Permit |
| operator | `configure terminal` | Deny |

TACACS+ command authorization is a major advantage over traditional RADIUS-based device administration.

**TACACS+ authorization must be explicitly configured.** Successfully authenticating a user through TACACS+ does not automatically enable EXEC or command authorization.

---

## TACACS+ Accounting

TACACS+ accounting records administrative activities.

Common accounting services include:

| Accounting Type | Records |
|---|---|
| EXEC | Administrative session start and stop events |
| Commands | Executed CLI commands at configured privilege levels |
| Network | Network service session activity |

For example, an accounting record might identify:

```text
User: admin
Device: R1
Command: configure terminal
```

Accounting records can include information such as the username, device, command, and timestamp.

TACACS+ accounting does not decide whether a command is permitted.

**Authorization controls whether a command may execute; accounting records the activity.**

The available accounting modes and IOS XE configuration commands are covered in the AAA Accounting page.

---

## TACACS+ Packet Format

TACACS+ messages contain a fixed-length **12-byte header** followed by a variable-length body.

The header includes:

| Field | Size | Purpose |
|---|---|---|
| Version | 1 byte | Identifies the TACACS+ protocol version |
| Type | 1 byte | Identifies authentication, authorization, or accounting |
| Sequence Number | 1 byte | Tracks messages within a TACACS+ session |
| Flags | 1 byte | Indicates protocol options, including whether the body is obfuscated |
| Session ID | 4 bytes | Identifies the TACACS+ session |
| Length | 4 bytes | Length of the packet body |

The three primary message types are:

| Type | Value |
|---|---|
| Authentication | 1 |
| Authorization | 2 |
| Accounting | 3 |

A TACACS+ authentication exchange may require multiple messages.

Authorization and accounting exchanges normally use a request and a corresponding reply.

Multiple TACACS+ sessions can use the same TCP connection.

---

## TACACS+ Packet Protection

Traditional TACACS+ uses a shared secret and an MD5-based mechanism to obfuscate most of the packet body.

The packet header remains visible.

For example:

| Packet Component | Protected by Traditional TACACS+ Obfuscation? |
|---|---|
| Header | No |
| Authentication data | Yes |
| Authorization data | Yes |
| Accounting data | Yes |

The shared secret is configured on both the IOS XE device and the TACACS+ server.

Both sides must use the same secret to communicate successfully.

**Traditional TACACS+ packet obfuscation is not equivalent to modern authenticated encryption.**

Its MD5-based protection has known security weaknesses and does not provide strong message integrity or confidentiality against capable attackers.

Consequently, traditional TACACS+ should be used over a trusted or otherwise protected management network.

TACACS+ over TLS 1.3 is separately specified in RFC 9887, but support must be verified for the particular IOS XE platform and release.

---

## IOS XE TACACS+ Server Configuration

Cisco IOS XE supports configuring individual TACACS+ servers using `tacacs server`.

The basic syntax is:

```text
tacacs server <name>
 address ipv4 <address>
 key <shared-secret>
```

For example:

```text
R1(config)# tacacs server TAC1
R1(config-server-tacacs)# address ipv4 192.0.2.10
R1(config-server-tacacs)# key Cisco123
```

This defines a TACACS+ server named `TAC1` at `192.0.2.10`.

The name `TAC1` is a locally significant identifier used by IOS XE to reference the server.

The shared secret must match the secret configured for R1 on the TACACS+ server.

**Defining a TACACS+ server does not automatically enable TACACS+ authentication, authorization, or accounting.**

AAA method lists must reference the server through an appropriate TACACS+ group.

### TACACS+ Server Parameters

The following options are commonly available under TACACS+ server configuration mode:

| Command | Purpose |
|---|---|
| `address ipv4 <address>` | Configures the server's IPv4 address |
| `address ipv6 <address>` | Configures the server's IPv6 address |
| `key <secret>` | Configures the shared secret |
| `port <number>` | Changes the destination TCP port |
| `timeout <seconds>` | Configures the server response timeout |
| `single-connection` | Enables TCP connection reuse |

Support for individual options can vary by IOS XE platform and release.

---

## TACACS+ Server Timeout

The `timeout` command controls how long IOS XE waits for a response from a particular TACACS+ server.

For example:

```text
R1(config)# tacacs server TAC1
R1(config-server-tacacs)# timeout 5
```

This configures a five-second timeout for TAC1.

If TAC1 does not respond within the timeout, IOS XE can attempt another available TACACS+ server, according to the server selection and failover behavior.

The router does not necessarily proceed immediately to the next AAA method when a single TACACS+ server times out.

For example:

```text
aaa authentication login default group tacacs+ local
```

If multiple TACACS+ servers are available, IOS XE can try another TACACS+ server before falling back to local authentication.

**Local fallback occurs when the TACACS+ method returns ERROR, not merely because one server fails to respond.**

### Global TACACS+ Timeout

A global timeout can also be configured:

```text
R1(config)# tacacs-server timeout 10
```

This sets the global TACACS+ server response timeout to 10 seconds.

A timeout configured under an individual `tacacs server` definition overrides the global setting for that server.

---

## TACACS+ Single-Connection Mode

By default, a separate TCP connection is established for each TACACS+ session.

For example, login authentication, EXEC authorization, and each individual command authorization request normally use separate TCP connections. If command accounting is enabled, each accounting exchange can also require its own connection.

The `single-connection` command allows multiple TACACS+ sessions to share a persistent TCP connection, reducing connection establishment overhead. Both the client and server must support this behavior.

Configure:

```text
R1(config)# tacacs server TAC1
R1(config-server-tacacs)# single-connection
```

With single-connection mode:

- Multiple AAA exchanges can use the same TCP connection.
- The router avoids repeatedly establishing new TCP connections.
- Connection establishment overhead is reduced.

Single-connection mode is particularly useful when frequent TACACS+ requests occur, such as with command authorization.

**The TACACS+ server must support single-connection mode.**

If the server does not support it correctly, communication problems can occur.

Single-connection mode does not combine authentication, authorization, and accounting into a single AAA operation.

They remain separate protocol exchanges.

---

## TACACS+ Source Interface

By default, the source IP address used for TACACS+ traffic depends on routing and interface selection.

IOS XE allows a specific source interface to be configured:

```text
R1(config)# ip tacacs source-interface Loopback0
```

For example:

```text
R1(config)# interface Loopback0
R1(config-if)# ip address 10.0.0.1 255.255.255.255
!
R1(config)# ip tacacs source-interface Loopback0
```

The TACACS+ packets sourced through this configuration use the IPv4 address of Loopback0.

This is useful because:

- The TACACS+ server can identify the client using a stable IP address.
- AAA policies can match a consistent device address.
- The source address does not change based on the outgoing physical interface.

The source address must be reachable from the TACACS+ server.

### IPv6 Source Interface

For IPv6 TACACS+ communication, IOS XE supports:

```text
R1(config)# ipv6 tacacs source-interface Loopback0
```

This specifies the source interface for IPv6 TACACS+ packets.

**The IPv4 and IPv6 TACACS+ source-interface commands are separate.**

VRF-specific source-interface configuration and routing behavior are covered in the AAA Server Groups and VRF-Aware Configuration page.

---

## TACACS+ Server Groups

Multiple TACACS+ servers can be placed into a named server group.

For example:

```text
R1(config)# tacacs server TAC1
R1(config-server-tacacs)# address ipv4 192.0.2.10
R1(config-server-tacacs)# key Cisco123
!
R1(config)# tacacs server TAC2
R1(config-server-tacacs)# address ipv4 192.0.2.20
R1(config-server-tacacs)# key Cisco123
!
R1(config)# aaa group server tacacs+ TAC_GROUP
R1(config-sg-tacacs+)# server name TAC1
R1(config-sg-tacacs+)# server name TAC2
```

This defines a named server group containing TAC1 and TAC2.

A method list can reference the group:

```text
R1(config)# aaa authentication login default group TAC_GROUP local
```

The router uses the servers in `TAC_GROUP` for authentication.

### Default TACACS+ Group

The keyword `group tacacs+` references the default TACACS+ server group.

For example:

```text
R1(config)# aaa authentication login default group tacacs+ local
```

The default TACACS+ group includes the globally configured TACACS+ servers, including servers that also belong to named server groups.

A named group allows an AAA method list to select a specific subset of servers.

| Method | Server Selection |
|---|---|
| `group tacacs+` | Default TACACS+ server group |
| `group TAC_GROUP` | TACACS+ servers assigned to `TAC_GROUP` |

Server selection, failover, and VRF-aware configuration are covered in the AAA Server Groups and VRF-Aware Configuration page.

---

## Applying TACACS+ Authentication

After defining a server, create an AAA authentication method list.

For example:

```text
R1(config)# aaa new-model
!
R1(config)# tacacs server TAC1
R1(config-server-tacacs)# address ipv4 192.0.2.10
R1(config-server-tacacs)# key Cisco123
!
R1(config)# username admin privilege 15 secret Local123
!
R1(config)# aaa authentication login default group tacacs+ local
```

This configures login authentication using TACACS+, with local authentication as a fallback.

The router attempts:

1. TACACS+ authentication.
2. Local authentication if TACACS+ returns an error.

If TACACS+ explicitly rejects the user, the router does not attempt the local method.

The default login authentication method list applies to applicable lines unless overridden by a named method list or line-specific authentication behavior.

For example, the VTY lines can explicitly reference the default method list:

```text
R1(config)# line vty 0 4
R1(config-line)# login authentication default
R1(config-line)# transport input ssh
```

> This example configures login authentication only. TACACS+ EXEC authorization, command authorization, and accounting require additional AAA configuration.

---

## Legacy TACACS+ Server Configuration

Older Cisco IOS and IOS XE configurations use the global `tacacs-server host` command.

For example:

```text
R1(config)# tacacs-server host 192.0.2.10 key Cisco123
```

The modern named-server syntax is:

```text
R1(config)# tacacs server TAC1
R1(config-server-tacacs)# address ipv4 192.0.2.10
R1(config-server-tacacs)# key Cisco123
```

The named-server configuration model is preferred for current IOS XE deployments.

The legacy syntax may still appear in older documentation and existing network configurations.

---

## IOS XE Verification

### Display TACACS+ Configuration

Use:

```text
R1# show running-config | section ^tacacs server
```

This displays the named TACACS+ server definitions.

To examine AAA server groups:

```text
R1# show running-config | section ^aaa group server
```

To view configured AAA method lists:

```text
R1# show running-config | section ^aaa
```

### Display TACACS+ Server Statistics

Use:

```text
R1# show tacacs
```

This command displays TACACS+ server information and statistics on supported IOS XE platforms.

It can help identify communication attempts, responses, and server-related errors.

### Debug TACACS+ Authentication

Use:

```text
R1# debug tacacs authentication
```

This enables debugging for TACACS+ authentication.

For TACACS+ authorization:

```text
R1# debug tacacs authorization
```

For TACACS+ accounting:

```text
R1# debug tacacs accounting
```

General AAA debugging is also available:

```text
R1# debug aaa authentication
R1# debug aaa authorization
R1# debug aaa accounting
```

Disable debugging after troubleshooting:

```text
R1# undebug all
```

> TACACS+ debugging may expose sensitive authentication and authorization information. Use debugging carefully, especially on production devices.

---

## Common TACACS+ Problems

| Problem | Possible Cause |
|---|---|
| TACACS+ server unreachable | Routing issue, ACL filtering, incorrect destination address |
| TCP connection fails | TCP port 49 blocked or server not listening |
| Authentication fails | Incorrect credentials or server authentication policy |
| Shared-secret mismatch | TACACS+ client and server use different keys |
| Authentication works but EXEC access fails | EXEC authorization is denied or misconfigured |
| User receives unexpected privilege level | Incorrect authorization attributes or local fallback behavior |
| Commands are denied | TACACS+ command authorization policy |
| TACACS+ works from one interface but not another | Incorrect source interface, VRF, or routing |
| Connection errors with `single-connection` | Server does not properly support connection reuse |
| Local fallback does not occur | TACACS+ returned FAIL rather than ERROR |

When troubleshooting TACACS+, distinguish:

1. **Connectivity:** Can the device communicate with the TACACS+ server?
2. **Authentication:** Does the server accept the credentials?
3. **Authorization:** Does the server permit the requested service or command?
4. **Accounting:** Are the expected records being delivered?

A successful TACACS+ authentication does not guarantee successful authorization.

---

## Key Points

- TACACS+ provides centralized authentication, authorization, and accounting for network device administration.
- TACACS+ uses **TCP port 49** by default.
- The network device acts as the TACACS+ client.
- TACACS+ separates authentication, authorization, and accounting into distinct exchanges.
- TACACS+ supports **granular per-command authorization and accounting**.
- TACACS+ authentication may involve multiple message exchanges.
- TACACS+ authorization and accounting normally use request/reply exchanges.
- Traditional TACACS+ protects most of the packet body using an MD5-based obfuscation mechanism, but its cryptographic protection is weak.
- The TACACS+ packet header is not obfuscated.
- Configure servers using `tacacs server <name>`.
- `address ipv4` and `address ipv6` configure the server's address.
- `key` configures the shared secret.
- `timeout` configures how long IOS XE waits for a server response.
- `single-connection` allows multiple TACACS+ sessions to reuse one TCP connection.
- `ip tacacs source-interface` specifies the source interface for IPv4 TACACS+ traffic.
- `ipv6 tacacs source-interface` specifies the source interface for IPv6 TACACS+ traffic.
- `group tacacs+` references the default TACACS+ server group.
- `group <name>` references a named AAA server group.
- A TACACS+ FAIL result does not trigger fallback to the next AAA method.
- A TACACS+ ERROR result allows the next configured method to be attempted.
- Defining a TACACS+ server does not automatically enable AAA authentication, authorization, or accounting.
- Use `show tacacs` and the appropriate `debug tacacs` commands for verification and troubleshooting.

---

## References

- [Cisco IOS XE 17.x - Configuring TACACS+](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-cfg-tacacs-0.html)
- [Cisco IOS XE 17.x - TACACS+ over IPv6](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_ip6-tacacs-xe.html)
- [RFC 8907: The TACACS+ Protocol](https://www.rfc-editor.org/rfc/rfc8907)
- [RFC 9887: TACACS+ over TLS 1.3](https://www.rfc-editor.org/rfc/rfc9887)
