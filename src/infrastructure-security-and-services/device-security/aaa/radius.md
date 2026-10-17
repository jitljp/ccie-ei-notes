# RADIUS

**Remote Authentication Dial-In User Service (RADIUS)** is a protocol that provides centralized **Authentication, Authorization, and Accounting (AAA)** for network devices and network access services.

RADIUS allows Cisco IOS XE routers and switches to delegate authentication and authorization decisions to external AAA servers and send accounting information to those servers.

Common uses include:

- Authenticating administrators connecting through SSH.
- Assigning user privileges and session attributes.
- Authenticating network access using 802.1X.
- Authorizing access to network services.
- Recording user sessions and resource usage.

RADIUS is widely used with Cisco Identity Services Engine (ISE) and other centralized AAA platforms.

Unlike TACACS+, which separates authentication and authorization into distinct protocol operations, **RADIUS typically combines authentication and authorization in the same exchange**.

---

## RADIUS Characteristics

| Characteristic | RADIUS |
|---|---|
| Transport protocol | UDP |
| Authentication/authorization port | UDP 1812 |
| Accounting port | UDP 1813 |
| Legacy authentication port | UDP 1645 |
| Legacy accounting port | UDP 1646 |
| AAA functions | Authentication, Authorization, Accounting |
| Authentication and authorization | Typically combined |
| Authorization information | Returned as attributes |
| Shared secret | Used to protect and validate messages |
| Packet protection | Selected fields are protected; most attributes are not encrypted |

The network device acts as the **RADIUS client**, also called the **Network Access Server (NAS)**.

The external RADIUS server validates credentials, applies authorization policies, and receives accounting records.

For example:

```text
                 UDP 1812 / 1813
R1 -------------------------------- RADIUS Server
    Authentication / Authorization
    Accounting
```

The RADIUS client and server must have matching shared secrets.

---

## RADIUS Authentication and Authorization

RADIUS uses a request/response model.

When a user attempts to access a service:

1. The network device sends an **Access-Request** to the RADIUS server.
2. The server evaluates the credentials and applicable policies.
3. The server returns an authentication result.
4. If access is permitted, the response can include authorization attributes.
5. The network device enforces the result.

### RADIUS Authentication Messages

| Message | Code | Purpose |
|---|---|---|
| Access-Request | 1 | Requests authentication and authorization |
| Access-Accept | 2 | Accepts the request and may return authorization attributes |
| Access-Reject | 3 | Rejects the request |
| Access-Challenge | 11 | Requests additional authentication information |

For example:

```text
Administrator             R1                  RADIUS Server
      |                   |                         |
      |---- SSH Login --->|                         |
      |                   |                         |
      |                   |----- Access-Request --->|
      |                   |                         |
      |                   |<---- Access-Accept -----|
      |                   |      + Attributes       |
      |                   |                         |
      |<-- Login Allowed -|                         |
```

An Access-Accept can include attributes that determine which services the user may access.

For example, a RADIUS server can return a privilege-level attribute allowing an administrator to enter privileged EXEC mode.

### Access-Accept

An **Access-Accept** indicates that the RADIUS server has accepted the request.

The response may include authorization attributes such as:

- Service type.
- Privilege level.
- VLAN assignment.
- Access control policies.
- Session timeout.

The attributes used depend on the requested service and the RADIUS server's policy.

### Access-Reject

An **Access-Reject** explicitly denies the request.

For example, if a user enters an incorrect password, the server may return Access-Reject.

This produces an AAA authentication failure.

**An Access-Reject does not trigger fallback to the next authentication method.**

### Access-Challenge

An **Access-Challenge** requests additional authentication information.

It may be used in interactive or multi-step authentication processes.

For example, a server might request an additional credential before deciding whether to accept or reject the user.

Access-Challenge is not an authentication success or failure. The authentication exchange continues.

---

## RADIUS Attributes

RADIUS carries authentication, authorization, and accounting information using **Type-Length-Value (TLV) attributes**.

Each attribute identifies a particular piece of information.

Common RADIUS attributes include:

| Attribute | Number | Purpose |
|---|---|---|
| User-Name | 1 | Identifies the user |
| User-Password | 2 | Carries an obfuscated password for PAP-style authentication |
| NAS-IP-Address | 4 | Identifies the RADIUS client |
| Service-Type | 6 | Identifies the requested or authorized service |
| Framed-IP-Address | 8 | Specifies an IP address for certain network access services |
| Filter-Id | 11 | Identifies a filter or policy |
| Reply-Message | 18 | Carries a message from the server |
| State | 24 | Maintains state across authentication exchanges |
| Class | 25 | Carries server-provided information that can be echoed in accounting |
| Vendor-Specific | 26 | Carries vendor-defined attributes |
| Session-Timeout | 27 | Specifies a session duration |
| NAS-Identifier | 32 | Identifies the RADIUS client by name |
| NAS-Port-Type | 61 | Identifies the type of access port or service |
| EAP-Message | 79 | Carries EAP information |
| Message-Authenticator | 80 | Provides message authentication using an HMAC-based mechanism |

The attributes sent depend on the AAA service, authentication method, and IOS XE configuration.

### Vendor-Specific Attributes (VSAs)

RADIUS supports **Vendor-Specific Attributes (VSAs)** using Attribute 26.

VSAs allow vendors to define additional authorization information beyond the standard RADIUS attributes.

Cisco's vendor ID is:

```text
9
```

Cisco VSAs can communicate information such as administrative privileges and service-specific authorization parameters.

For example, Cisco's `Cisco-AVPair` attribute can carry:

```text
shell:priv-lvl=15
```

This indicates privilege level 15 for an IOS XE shell session.

A RADIUS Access-Accept containing an appropriate Cisco privilege attribute can therefore assign administrative privileges.

**RADIUS can assign privilege levels, but TACACS+ is generally better suited to granular per-command authorization.**

---

## RADIUS Packet Format

RADIUS messages contain a fixed-length **20-byte header**, followed by attributes.

The packet format is:

```text
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |     Code      |  Identifier   |            Length             |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                                                               |
 |                         Authenticator                         |
 |                           (16 bytes)                          |
 |                                                               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                         Attributes ...                        |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The fields are:

| Field | Length | Purpose |
|---|---|---|
| Code | 8 bits | Identifies the RADIUS message type |
| Identifier | 8 bits | Matches responses with requests |
| Length | 16 bits | Total RADIUS packet length |
| Authenticator | 128 bits | Used in authentication and integrity-related calculations |
| Attributes | Variable | Carries authentication, authorization, and accounting information |

The Identifier and Authenticator help the client associate a reply with the corresponding request.

Unlike TACACS+, RADIUS does not establish a TCP connection for each AAA exchange.

It uses UDP requests, responses, timeouts, and retransmissions.

---

## RADIUS Security

Traditional RADIUS does not encrypt the entire packet.

Instead, it uses a shared secret and protocol-specific mechanisms to protect selected information.

For example:

- The User-Password attribute is obfuscated using an MD5-based mechanism.
- The Response Authenticator is calculated using the shared secret.
- The Message-Authenticator attribute uses HMAC-MD5 to authenticate messages where applicable.
- Many other RADIUS attributes are transmitted without encryption.

**Traditional RADIUS does not provide full-packet confidentiality.**

This differs from traditional TACACS+, which obfuscates most of its packet body.

Neither traditional protocol provides the protection expected from modern authenticated encryption.

RADIUS should therefore operate over a trusted or otherwise protected management network.

RADIUS over TLS, often called **RadSec**, provides a separate transport-security mechanism, but its availability and configuration depend on the IOS XE platform and release.

---

## IOS XE RADIUS Server Configuration

Cisco IOS XE supports named RADIUS server definitions using the `radius server` command.

The basic configuration is:

```text
radius server <name>
 address ipv4 <address> [auth-port <port>] [acct-port <port>]
 key <shared-secret>
```

The standard RADIUS ports are UDP 1812 for authentication and UDP 1813 for accounting.

However, IOS XE's default ports depend on the platform and software release. Some versions default to the legacy ports 1645 and 1646.

For example:

```text
R1(config)# radius server RAD1
R1(config-radius-server)# address ipv4 192.0.2.10 auth-port 1812 acct-port 1813
R1(config-radius-server)# key Cisco123
```

This defines a RADIUS server named `RAD1`.

The parameters are:

| Parameter | Purpose |
|---|---|
| `RAD1` | Locally significant server name |
| `192.0.2.10` | RADIUS server IPv4 address |
| `auth-port 1812` | Authentication/authorization destination UDP port |
| `acct-port 1813` | Accounting destination UDP port |
| `key Cisco123` | Shared secret |

The shared secret must match the value configured for R1 on the RADIUS server.

The server name is a local IOS XE identifier, not a name transmitted to the RADIUS server as part of authentication.

### IPv6 RADIUS Server

An IPv6 RADIUS server can be configured using:

```text
R1(config)# radius server RAD6
R1(config-radius-server)# address ipv6 2001:DB8::10 auth-port 1812 acct-port 1813
R1(config-radius-server)# key Cisco123
```

RADIUS supports both IPv4 and IPv6 transport.

The RADIUS message format is independent of whether IPv4 or IPv6 is used to reach the server.

---

## RADIUS Server Parameters

Additional settings can be configured under a named RADIUS server.

| Command | Purpose |
|---|---|
| `timeout <seconds>` | Configures the response timeout |
| `retransmit <count>` | Configures the retransmission count |
| `key <secret>` | Configures the shared secret |
| `automate-tester username <user>` | Configures automated RADIUS server testing |

### Server Timeout

The `timeout` command controls how long the device waits for a response before treating the attempt as timed out.

For example:

```text
R1(config)# radius server RAD1
R1(config-radius-server)# timeout 5
```

This configures a five-second response timeout.

### Retransmissions

Because RADIUS uses UDP, it relies on application-layer retransmissions when requests do not receive responses.

Configure:

```text
R1(config)# radius server RAD1
R1(config-radius-server)# retransmit 3
```

This configures the number of retransmissions.

If the server does not respond, IOS XE can retry the request before trying another eligible server or returning an error to the AAA method list.

**A server timeout does not necessarily cause immediate fallback to the next AAA method.**

IOS XE may first attempt retransmissions and other servers in the RADIUS server group.

### Global RADIUS Settings

Global timeout and retransmission settings can also be configured:

```text
R1(config)# radius-server timeout 5
R1(config)# radius-server retransmit 3
```

Server-specific settings take precedence over the corresponding global settings.

---

## RADIUS Source Interface

IOS XE allows the source interface for RADIUS traffic to be specified.

For example:

```text
R1(config)# interface Loopback0
R1(config-if)# ip address 10.0.0.1 255.255.255.255
!
R1(config)# ip radius source-interface Loopback0
```

This makes RADIUS traffic use the IPv4 address associated with Loopback0.

A stable source address is useful because RADIUS servers commonly identify network devices by their client IP addresses.

For example, the RADIUS server might be configured with:

```text
Client: 10.0.0.1
Shared secret: Cisco123
```

If the router instead sends requests from an unexpected source IP, the server may reject or silently discard them.

The selected source address must also be reachable from the RADIUS server.

For IPv6 RADIUS traffic, IOS XE supports the corresponding IPv6 source-interface configuration:

```text
R1(config)# ipv6 radius source-interface Loopback0
```

Source-interface selection, server-group overrides, and VRF-aware communication are covered in the AAA Server Groups and VRF-Aware Configuration page.

---

## RADIUS Server Groups

Multiple RADIUS servers can be organized into a named AAA server group.

For example:

```text
R1(config)# radius server RAD1
R1(config-radius-server)# address ipv4 192.0.2.10 auth-port 1812 acct-port 1813
R1(config-radius-server)# key Cisco123
!
R1(config)# radius server RAD2
R1(config-radius-server)# address ipv4 192.0.2.20 auth-port 1812 acct-port 1813
R1(config-radius-server)# key Cisco123
!
R1(config)# aaa group server radius RAD_GROUP
R1(config-sg-radius)# server name RAD1
R1(config-sg-radius)# server name RAD2
```

This creates the named server group `RAD_GROUP`.

An authentication method list can reference it:

```text
R1(config)# aaa authentication login default group RAD_GROUP local
```

The router uses the RADIUS servers in `RAD_GROUP` for login authentication.

### Default RADIUS Group

The keyword `group radius` selects the default RADIUS server group.

For example:

```text
R1(config)# aaa authentication login default group radius local
```

The default group includes globally configured RADIUS servers, including those assigned to named server groups.

| Method | Server Selection |
|---|---|
| `group radius` | Default RADIUS server group |
| `group RAD_GROUP` | Servers assigned to the named group |

Named groups allow different AAA services to use different sets of RADIUS servers.

---

## Applying RADIUS Authentication

After configuring a RADIUS server, enable AAA and create an authentication method list.

For example:

```text
R1(config)# aaa new-model
!
R1(config)# radius server RAD1
R1(config-radius-server)# address ipv4 192.0.2.10 auth-port 1812 acct-port 1813
R1(config-radius-server)# key Cisco123
!
R1(config)# username admin privilege 15 secret Local123
!
R1(config)# aaa authentication login default group radius local
```

The authentication order is:

1. RADIUS.
2. Local authentication if the RADIUS method returns an error.

| RADIUS Result | Local Authentication Attempted? |
|---|---|
| Access-Accept | No |
| Access-Reject | No |
| Access-Challenge | No, authentication exchange continues |
| No usable response after server attempts are exhausted | Yes |

For example, if the RADIUS server explicitly rejects the password, the router denies login rather than trying the local user database.

If no configured RADIUS server can complete the request, local authentication can be attempted.

### EXEC Authorization

RADIUS authentication alone does not automatically configure EXEC authorization on IOS XE.

To use AAA EXEC authorization, configure a separate method list:

```text
R1(config)# aaa authorization exec default group radius local
```

Although authentication and authorization information are commonly combined in a RADIUS Access-Accept, IOS XE still uses separate AAA configuration for the relevant services.

A RADIUS server can return authorization attributes that affect EXEC privileges.

For example, a Cisco VSA containing:

```text
shell:priv-lvl=15
```

can assign privilege level 15 when correctly interpreted by the device.

---

## RADIUS Accounting

RADIUS accounting uses separate accounting messages from authentication.

| Message | Code | Purpose |
|---|---|---|
| Accounting-Request | 4 | Sends accounting information |
| Accounting-Response | 5 | Acknowledges an accounting request |

An Accounting-Request contains attributes describing the event.

The **Acct-Status-Type** attribute identifies the accounting record type.

Common values include:

| Acct-Status-Type | Value | Meaning |
|---|---|---|
| Start | 1 | Session started |
| Stop | 2 | Session ended |
| Interim-Update | 3 | Periodic session update |

For example:

```text
R1 -------------------------> RADIUS Server
       Accounting-Request
       Acct-Status-Type = Start

R1 <------------------------- RADIUS Server
       Accounting-Response
```

When the session ends, R1 can send another Accounting-Request with `Acct-Status-Type = Stop`.

### IOS XE Example

Configure EXEC accounting:

```text
R1(config)# aaa accounting exec default start-stop group radius
```

The router sends accounting records to the RADIUS server group when an EXEC session starts and stops.

The accounting records may contain information such as the username, session duration, NAS address, and other session attributes.

Detailed accounting methods and options are covered in the AAA Accounting page.

---

## RADIUS Dead Server Detection

IOS XE can track RADIUS server availability.

A server that repeatedly fails to respond can be marked **dead**, allowing the device to avoid repeatedly attempting requests to an unavailable server.

Two important commands are:

```text
radius-server dead-criteria
radius-server deadtime
```


### Dead Criteria

The `radius-server dead-criteria` command controls when IOS XE considers a RADIUS server unavailable and marks it **dead**.

Two criteria can be configured:

- **Time:** The minimum time that must have elapsed since the last valid response from the RADIUS server.
- **Tries:** The number of consecutive request timeouts required before the server is marked dead.

For example:

```text
R1(config)# radius-server dead-criteria time 10 tries 3
```

This requires both conditions to be satisfied:

1. At least 10 seconds have elapsed since the last valid RADIUS response.
2. The server has reached the configured threshold of 3 consecutive timeouts.

**Both conditions must be satisfied before the server is marked dead.**

#### How the Criteria Work

Dead-server detection is based on actual RADIUS communication failures, not simply the absence of traffic.

Consider the following situations:

| Situation | Server Marked Dead? |
|---|---|
| No RADIUS requests have been sent for an hour | No |
| Last valid response was an hour ago, but there have been no subsequent requests | No |
| Last valid response was 5 seconds ago, and 3 timeouts have occurred | No, time criterion not met |
| Last valid response was 30 seconds ago, but only 2 timeouts have occurred | No, tries criterion not met |
| Last valid response was 30 seconds ago, and 3 timeouts have occurred | Yes |
| No valid response has been received since boot, and the tries criterion is met | Yes |

If no valid response has been received since the router booted, **the time criterion is considered satisfied when a request times out**. The configured tries criterion must still be met.

A successful RADIUS response resets the consecutive-timeout condition and updates the last-response time.

#### Example Timeline

Assume:

```text
radius-server dead-criteria time 10 tries 3
```

The router receives a valid RADIUS response at time 0.

| Time | Event | Result |
|---|---|---|
| 0s | Valid RADIUS response received | Server alive |
| 3s | First request timeout | Tries threshold not met |
| 6s | Second request timeout | Tries threshold not met |
| 9s | Third request timeout | Tries threshold met, but time criterion not met |
| 12s | Another request times out | Both criteria met; server marked dead |

The precise timing depends on request timeouts, retransmissions, and server traffic.

**The `time` parameter is not a periodic timer that automatically declares a server dead after inactivity.** It establishes a minimum elapsed-time condition that must be satisfied during dead-server detection.

The tries counter is based on timed-out RADIUS transmissions. The exact counting behavior can depend on the IOS XE implementation, including how retransmissions are counted.

Authentication and accounting failures can both contribute to dead-server detection when the server handles both services.

If only one criterion is explicitly configured, IOS XE uses its default behavior for the other criterion.

### Deadtime

The `deadtime` command controls how long a dead server remains excluded from normal server selection.

For example:

```text
R1(config)# radius-server deadtime 5
```

This configures a five-minute deadtime.

After the deadtime expires, the server becomes eligible for requests again.

Deadtime is different from the individual request timeout.

| Parameter | Purpose |
|---|---|
| `timeout` | How long to wait for a response |
| `retransmit` | How many times to retransmit requests |
| `dead-criteria` | When to consider a server unavailable |
| `deadtime` | How long a dead server remains excluded |

These settings can significantly affect how quickly a device falls back to another AAA server or method.

---


## Automated RADIUS Server Testing

IOS XE supports **automated RADIUS server testing** to monitor server availability without relying entirely on real user authentication requests.

Configure automated testing under an individual RADIUS server:

```text
R1(config)# radius server RAD1
R1(config-radius-server)# automate-tester username radius-probe idle-time 2
```

The router uses the specified username when sending test requests to the RADIUS server.

Automated testing is useful for:

- Detecting unavailable servers during periods of low authentication activity.
- Checking whether a server previously marked dead has recovered.
- Updating server availability before real users attempt authentication.

### Automated Testing Options

| Option | Behavior |
|---|---|
| `username <user>` | Specifies the username included in test authentication requests |
| `idle-time <minutes>` | Tests server availability after the specified period of inactivity |
| `probe-on` | Enables recovery probing for a server marked dead |
| `ignore-auth-port` | Excludes the RADIUS authentication port from automated testing |
| `ignore-acct-port` | Excludes the RADIUS accounting port from automated testing |

### Idle-Time Testing

The `idle-time` option specifies how long the RADIUS server can remain idle before IOS XE initiates automated testing.

For example:

```text
R1(config)# radius server RAD1
R1(config-radius-server)# automate-tester username radius-probe idle-time 2
```

If the server has not received normal RADIUS activity for approximately two minutes, IOS XE can send test requests to verify its availability.

The key behaviors are:

- Normal RADIUS activity resets the inactivity period.
- An idle server can be tested even if it has not previously been marked dead.
- A server does not become dead merely because the idle-time interval expires.
- The router evaluates the server's response to determine availability.

**The `idle-time` option helps detect server failures even when no real users are attempting authentication.**

The default idle-time value, when the option is omitted, is 60 minutes on platforms documenting this behavior.

### Probe-On Testing

The `probe-on` option focuses on testing whether a server that has been marked dead is responding again.

For example:

```text
R1(config)# radius server RAD1
R1(config-radius-server)# automate-tester username radius-probe probe-on
```

Consider this sequence:

1. RAD1 fails to respond to normal RADIUS requests.
2. IOS XE marks RAD1 dead according to the configured dead criteria.
3. RAD1 is excluded from normal server selection.
4. Automated probing checks whether RAD1 has recovered.
5. When RAD1 responds appropriately, IOS XE can update its state to alive.

On documented Catalyst IOS XE implementations, recovery probing is coordinated with the dead-server timers. The router may retry probes using the configured RADIUS timeout and deadtime settings.

**`probe-on` is not a command to send an immediate, one-time test request from the CLI.** It configures automated server-state verification.

### Ignoring Authentication or Accounting Ports

By default, automated testing can check the server's authentication and accounting services.

The `ignore-auth-port` and `ignore-acct-port` options exclude individual services from testing.

For example:

```text
R1(config)# radius server RAD1
R1(config-radius-server)# automate-tester username radius-probe ignore-acct-port idle-time 2
```

This tests the RADIUS authentication service but excludes the accounting port.

Alternatively:

```text
R1(config)# radius server RAD1
R1(config-radius-server)# automate-tester username radius-probe ignore-auth-port idle-time 2
```

This excludes the authentication port and tests the accounting service.

These options can be useful when a server is intended to provide only one of the two services.

### Test Username and Responses

The username configured with `automate-tester` identifies the synthetic authentication requests sent by the router.

For example:

```text
automate-tester username radius-probe idle-time 2
```

The RADIUS server does not necessarily need to accept this username.

For authentication-port testing, **a valid RADIUS Access-Reject can still demonstrate that the server is responding**. The purpose is to verify server availability, not to authenticate a real administrator.

Cisco recommends using a test username that is not defined as a valid user on the RADIUS server.

### Relationship to Dead Criteria and Deadtime

These commands perform different functions:

| Command | Purpose |
|---|---|
| `radius-server dead-criteria` | Determines when a nonresponsive server is marked dead |
| `radius-server deadtime` | Controls how long a dead server is excluded from normal selection |
| `automate-tester ... idle-time` | Tests availability after a period of server inactivity |
| `automate-tester ... probe-on` | Probes a dead server to detect recovery |

Automated testing supplements dead-server detection. It does not replace the configured timeout, retransmission, and dead-server mechanisms.

> Automated testing syntax and exact scheduling behavior vary by IOS XE platform and release.

---

### Key Points

- `automate-tester` enables automated RADIUS server availability checks.
- `username` specifies the identity used in synthetic test requests.
- `idle-time` tests server availability after a period without normal RADIUS activity.
- `probe-on` is primarily used to detect recovery of a server marked dead.
- `ignore-auth-port` excludes authentication-port testing.
- `ignore-acct-port` excludes accounting-port testing.
- An Access-Reject can still demonstrate that a RADIUS server is reachable and responding.
- Automated testing works alongside `dead-criteria` and `deadtime`, not instead of them.

---

## RADIUS vs. TACACS+

| Feature | RADIUS | TACACS+ |
|---|---|---|
| Transport | UDP | TCP |
| Default port | 1812 / 1813 | 49 |
| Authentication and authorization | Typically combined in one exchange | Separate exchanges |
| Administrative privilege assignment | Supported through attributes | Supported through authorization attributes |
| Granular command authorization | Not a typical use | Supported |
| Command accounting | Not the usual choice for CLI command auditing | Supported |
| Packet protection | Selected fields | Most of the packet body |
| Common deployment | 802.1X, network access, device administration | Device administration |

**TACACS+ is generally preferred for administrative command authorization, while RADIUS is more commonly used for network access authentication.**

Both protocols can authenticate administrators and assign session privileges.

---

## IOS XE Verification

### Display RADIUS Configuration

Show named RADIUS server definitions:

```text
R1# show running-config | section ^radius server
```

Show configured server groups:

```text
R1# show running-config | section ^aaa group server
```

Show AAA method lists:

```text
R1# show running-config | section ^aaa
```

### Display RADIUS Statistics

Use:

```text
R1# show aaa servers
```

This displays AAA server information and statistics, including server state and authentication activity on supported IOS XE platforms.

Also use:

```text
R1# show radius statistics
```

This displays RADIUS-related counters.

Useful information includes:

- Requests sent.
- Responses received.
- Timeouts.
- Retransmissions.
- Authentication failures.
- Server availability.

### Test RADIUS Authentication

IOS XE supports testing RADIUS authentication using:

```text
R1# test aaa group radius admin Cisco123 new-code
```

The command sends a test authentication request to the RADIUS server group.

The test can help distinguish server communication failures from incorrect user credentials or policy problems.

The syntax and supported options depend on the IOS XE release.

### Debug RADIUS

Enable RADIUS debugging:

```text
R1# debug radius authentication
```

For detailed packet processing, supported platforms also provide:

```text
R1# debug radius
```

General AAA debugging can help correlate RADIUS activity with the AAA method list:

```text
R1# debug aaa authentication
R1# debug aaa authorization
R1# debug aaa accounting
```

Disable debugging after troubleshooting:

```text
R1# undebug all
```

> RADIUS debugging can expose sensitive authentication and session information. Avoid leaving debugging enabled on production devices.

---

## Common RADIUS Problems

| Problem | Possible Cause |
|---|---|
| No response from RADIUS server | Routing failure, ACL filtering, incorrect UDP ports |
| Access-Reject received | Incorrect credentials or server policy |
| Shared-secret mismatch | Client and server have different secrets |
| Server does not recognize the client | Unexpected source IP or missing NAS definition |
| Authentication succeeds but privilege is wrong | Missing or incorrect authorization attribute |
| Local fallback does not occur | Server returned Access-Reject rather than a communication error |
| Authentication takes too long | Excessive timeouts or retransmissions |
| Server marked dead unexpectedly | Dead criteria too aggressive or intermittent connectivity |
| Accounting records missing | Incorrect accounting port, shared secret, or accounting method list |
| RADIUS works in global routing but not in a VRF | Incorrect VRF or source-interface configuration |

When troubleshooting, distinguish three separate conditions:

1. **Access-Accept:** The server accepted the request.
2. **Access-Reject:** The server explicitly denied the request.
3. **No usable response:** The AAA method may eventually return an error and permit fallback.

An Access-Reject is not equivalent to server unavailability.

---

## Key Points

- RADIUS provides centralized authentication, authorization, and accounting.
- RADIUS uses **UDP**, not TCP.
- The standard ports are **UDP 1812 for authentication/authorization** and **UDP 1813 for accounting**.
- Legacy ports are UDP 1645 and 1646.
- The network device acts as the RADIUS client or NAS.
- RADIUS typically combines authentication and authorization in the same protocol exchange.
- Access-Request (Code 1) requests authentication.
- Access-Accept (Code 2) permits access and can include authorization attributes.
- Access-Reject (Code 3) explicitly denies access.
- Access-Challenge (Code 11) requests additional authentication information.
- RADIUS packets contain a **20-byte header** followed by TLV attributes.
- Attribute 26 carries Vendor-Specific Attributes.
- Cisco's vendor ID is **9**.
- The Cisco VSA `shell:priv-lvl=15` can assign privilege level 15 for an EXEC session.
- Traditional RADIUS protects selected fields but does not encrypt the entire packet.
- Configure a named server with `radius server <name>`.
- Use `address ipv4 ... auth-port 1812 acct-port 1813` to configure the server address and ports.
- `timeout` and `retransmit` control request retry behavior.
- `ip radius source-interface` specifies the source interface for IPv4 RADIUS traffic.
- `group radius` references the default RADIUS server group.
- `group <name>` references a named AAA server group.
- **Access-Reject does not trigger fallback** to the next authentication method.
- Local fallback occurs when the RADIUS method returns an error.
- RADIUS accounting uses Accounting-Request (Code 4) and Accounting-Response (Code 5).
- `radius-server dead-criteria` controls when a server is considered dead.
- `radius-server deadtime` controls how long a dead server remains excluded.
- `automate-tester` can probe RADIUS server availability.
- Use `show aaa servers`, `show radius statistics`, and `test aaa group radius` for verification.

---

## References

- [Cisco IOS XE 17 - RADIUS Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/sec-crypto/radius/radius-configuration-guide/radius.html)
- [Cisco IOS XE 17 - AAA Server Groups](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-rad-aaa-server-groups.html)
- [Cisco IOS XE 17 - RADIUS Server Load Balancing and Automated Testing](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/sec-crypto/radius/radius-configuration-guide/radius-server-load-balancing.html)
- [Cisco - Understand AAA Dead Detection and Deadtime on IOS XE](https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/225882-understand-aaa-dead-detection-and.html)
- [RFC 2865: Remote Authentication Dial In User Service (RADIUS)](https://www.rfc-editor.org/rfc/rfc2865)
- [RFC 2866: RADIUS Accounting](https://www.rfc-editor.org/rfc/rfc2866)
- [RFC 6614: Transport Layer Security (TLS) Encryption for RADIUS](https://www.rfc-editor.org/rfc/rfc6614)
