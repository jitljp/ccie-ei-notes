
# AAA Server Failover and Local Fallback

**AAA server failover** allows a Cisco IOS XE device to use another AAA server when the selected server is unavailable.

**AAA local fallback** allows the device to use locally configured credentials or authorization information when remote AAA methods cannot complete a request.

These are two distinct mechanisms:

- **Server failover:** IOS XE attempts another server within the selected AAA server group.
- **Method fallback:** IOS XE advances to the next method in an AAA method list, such as `local`.

For example:

```text
aaa authentication login default group tacacs+ local
```

This method list first attempts authentication using TACACS+ servers.

If the TACACS+ method cannot complete authentication because the servers are unavailable, IOS XE attempts authentication using the local username database.

However, **an explicit authentication rejection does not trigger local fallback**.

---

## AAA Failover Operation

AAA processing involves two levels of selection:

1. Select an AAA method from the configured method list.
2. If the method uses a server group, select an available server within that group.

Consider:

```text
R1(config)# aaa authentication login default group TAC_SERVERS local
```

Suppose the `TAC_SERVERS` group contains two TACACS+ servers:

```text
TAC_SERVERS
  |
  +-- TAC1: 192.0.2.10
  |
  +-- TAC2: 192.0.2.11
```

The general processing sequence is:

```text
                     Login Attempt
                           |
                           v
                   TAC_SERVERS Group
                           |
                           v
                     Try TAC1
                           |
                  +--------+--------+
                  |                 |
              Responds         Unavailable
                  |                 |
          +-------+-------+         v
          |               |      Try TAC2
        PASS             FAIL        |
          |               |     +----+----+
          v               v     |         |
       Permit            Deny Responds Unavailable
                                |         |
                           PASS/FAIL     v
                                |       Local
                                v      Fallback
                          Accept/Deny
```

**Local fallback is attempted only when the remote AAA method returns an error, not when a server explicitly rejects the request.**

---

## AAA Method Results

IOS XE distinguishes between successful authentication or authorization, an explicit rejection, and an error.

| Result | Meaning | Next Method Attempted? |
|---|---|---|
| PASS | Authentication or authorization succeeded | No |
| FAIL | Server explicitly rejected the request | No |
| ERROR | Method could not complete the request | Yes, if another method exists |

For RADIUS authentication:

| RADIUS Result | AAA Interpretation |
|---|---|
| Access-Accept | PASS |
| Access-Reject | FAIL |
| No usable response after applicable attempts | ERROR |

For TACACS+ authentication, the server likewise distinguishes between successful authentication, failed authentication, and error conditions.

### PASS

If the server accepts the request, processing stops.

For example:

```text
aaa authentication login default group tacacs+ local
```

If TACACS+ successfully authenticates the user, IOS XE does not check the local username database.

### FAIL

If the server explicitly rejects the request, processing stops.

For example, suppose the TACACS+ server is reachable but rejects the supplied password.

IOS XE denies the login.

It does not attempt `local`.

### ERROR

If the selected server group cannot complete the request, IOS XE can attempt the next configured method.

For example, if all eligible TACACS+ servers are unavailable, IOS XE can attempt `local`.

**The distinction between FAIL and ERROR is the most important rule in AAA fallback.**

---

## Server Failover Within a Group

An AAA server group can contain multiple servers.

For example:

```text
R1(config)# tacacs server TAC1
R1(config-server-tacacs)# address ipv4 192.0.2.10
R1(config-server-tacacs)# key Cisco123
!
R1(config)# tacacs server TAC2
R1(config-server-tacacs)# address ipv4 192.0.2.11
R1(config-server-tacacs)# key Cisco123
```

Create a named server group:

```text
R1(config)# aaa group server tacacs+ TAC_SERVERS
R1(config-sg-tacacs+)# server name TAC1
R1(config-sg-tacacs+)# server name TAC2
```

Use the group:

```text
R1(config)# aaa authentication login default group TAC_SERVERS local
```

If TAC1 cannot complete the request, IOS XE can attempt TAC2.

If TAC2 successfully authenticates the user, the login succeeds.

If TAC2 explicitly rejects the user, authentication fails without attempting `local`.

If neither server can complete authentication, IOS XE can proceed to `local`.

Server selection can be affected by server availability, timers, and platform-specific selection or load-balancing mechanisms.

---

## Failover Between Server Groups

An AAA method list can contain multiple named server groups.

For example:

```text
R1(config)# aaa authentication login default group TAC_PRIMARY group TAC_BACKUP local
```

This defines three methods:

1. `group TAC_PRIMARY`
2. `group TAC_BACKUP`
3. `local`

IOS XE first attempts eligible servers in `TAC_PRIMARY`.

If that group returns an error, it attempts `TAC_BACKUP`.

If both groups return errors, it attempts `local`.

An explicit authentication rejection from either group stops processing.

### Server Failover vs. Group Failover

Consider two configurations.

**Multiple servers in one group:**

```text
aaa authentication login default group TAC_SERVERS local
```

IOS XE can try another server within `TAC_SERVERS` before returning an error to the method list.

**Multiple groups in one method list:**

```text
aaa authentication login default group TAC_PRIMARY group TAC_BACKUP local
```

IOS XE can move to the second group if the first group cannot complete authentication.

Both approaches provide redundancy, but they operate at different levels.

---

## Local Login Authentication Fallback

The most common local fallback configuration is:

```text
R1(config)# aaa authentication login default group tacacs+ local
```

Create a local user:

```text
R1(config)# username admin privilege 15 secret Local123
```

Enable AAA:

```text
R1(config)# aaa new-model
```

When TACACS+ is available, IOS XE uses TACACS+ authentication.

If the TACACS+ method returns an error, IOS XE attempts to authenticate the username against the local database.

### Local Fallback Requirements

For local fallback to succeed:

- A matching local username must exist.
- The supplied password must match the local user's password or secret.
- The applicable method list must contain `local`.
- Earlier methods must return ERROR rather than FAIL.

The local username is not used to override an explicit rejection from the TACACS+ server.

### Example: Incorrect Password

Suppose a user named `admin` exists on both the TACACS+ server and the router.

The user attempts to log in using the local password instead of the TACACS+ password.

If the TACACS+ server is reachable and rejects the password, authentication fails.

IOS XE does not try the local password.

### Example: Server Unavailable

Suppose the TACACS+ server is unreachable.

After the TACACS+ method returns an error, IOS XE attempts local authentication.

The user can log in using the locally configured credentials.

This is why local fallback is useful for management access during AAA outages.

---

## Local vs. Local-Case

Login authentication can use either `local` or `local-case`.

For example:

```text
R1(config)# aaa authentication login default group tacacs+ local
```

Or:

```text
R1(config)# aaa authentication login default group tacacs+ local-case
```

The difference is username case sensitivity.

| Method | Username Matching |
|---|---|
| `local` | Case-insensitive |
| `local-case` | Case-sensitive |

For example, suppose the locally configured username is:

```text
username Admin privilege 15 secret Local123
```

With `local`, the usernames `Admin` and `admin` are treated as equivalent.

With `local-case`, they are different.

The password or secret remains case-sensitive.

---

## Enable Authentication Fallback

AAA enable authentication has a separate method list.

For example:

```text
R1(config)# aaa authentication enable default group tacacs+ enable
```

Configure a local enable secret:

```text
R1(config)# enable secret Enable123
```

When a user enters:

```text
R1> enable
```

IOS XE first attempts TACACS+ enable authentication.

If the TACACS+ method returns an error, IOS XE can use the locally configured enable secret.

### The `enable` Method

**Enable authentication uses `enable`, not `local`, for local enable-secret fallback.**

For example:

```text
aaa authentication enable default group tacacs+ enable
```

The `enable` method checks the locally configured enable password or secret.

It does not authenticate a username against the local username database.

### RADIUS Enable Authentication

RADIUS enable authentication conventionally uses a special username such as:

```text
$enab15$
```

for privilege level 15.

For example:

```text
R1(config)# aaa authentication enable default group radius enable
```

The first method attempts RADIUS enable authentication.

The second method provides local enable-secret fallback when the RADIUS method returns an error.

A RADIUS Access-Reject stops the method list.

---

## EXEC Authorization Fallback

Local fallback is also available for EXEC authorization.

For example:

```text
R1(config)# aaa authorization exec default group tacacs+ local
```

When TACACS+ is available, IOS XE requests EXEC authorization from the TACACS+ server.

The server can return attributes such as the user's privilege level.

If the TACACS+ method returns an error, IOS XE attempts local authorization.

### Local EXEC Authorization

Configure:

```text
R1(config)# username admin privilege 15 secret Local123
```

If local EXEC authorization succeeds using this user's local information, the user can receive privilege level 15.

This is separate from login authentication.

A complete configuration might include:

```text
R1(config)# aaa authentication login default group tacacs+ local
R1(config)# aaa authorization exec default group tacacs+ local
```

The first method list controls login authentication.

The second method list controls EXEC authorization.

**Successful login authentication does not automatically mean EXEC authorization will succeed.**

### Authentication Fallback Without Authorization Fallback

Consider:

```text
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+
```

Suppose the TACACS+ server is unavailable.

Login authentication can succeed through `local`.

However, EXEC authorization has no local fallback.

If TACACS+ cannot complete EXEC authorization, the user may be denied an EXEC session even though the password was accepted.

This is a common AAA configuration mistake.

---

## The `if-authenticated` Authorization Method

Authorization method lists can use `if-authenticated` as a fallback.

For example:

```text
R1(config)# aaa authorization exec default group tacacs+ if-authenticated
```

The router attempts TACACS+ EXEC authorization first.

If TACACS+ returns an error, `if-authenticated` permits EXEC authorization for a previously authenticated user.

Unlike `local`, it does not retrieve privilege attributes from the local username database.

### Local vs. If-Authenticated

| Method | Fallback Behavior |
|---|---|
| `local` | Uses locally configured authorization information |
| `if-authenticated` | Permits authorization if the user has already authenticated |
| `none` | Permits authorization without another authorization check |

For example:

```text
aaa authorization exec default group tacacs+ local
```

can obtain a locally configured privilege level.

By contrast:

```text
aaa authorization exec default group tacacs+ if-authenticated
```

does not independently assign privilege level 15.

The resulting privilege level depends on other session and line settings, commonly defaulting to level 1.

**`if-authenticated` provides authorization fallback, but not equivalent local privilege assignment.**

---

## Command Authorization Fallback

Command authorization can also use local fallback.

For example:

```text
R1(config)# aaa authorization commands 15 default group tacacs+ local
```

If TACACS+ returns PASS, the command is permitted.

If TACACS+ returns FAIL, the command is denied.

If TACACS+ returns ERROR, IOS XE attempts local authorization.

### Local Command Authorization

Local command authorization does not provide the same granular command and argument policies as TACACS+.

It primarily relies on local authorization information and the applicable privilege-level restrictions.

For example:

```text
R1(config)# username admin privilege 15 secret Local123
```

A locally authorized privilege level 15 user can normally execute commands available at that privilege level.

### If-Authenticated Command Fallback

Configure:

```text
R1(config)# aaa authorization commands 15 default group tacacs+ if-authenticated
```

This attempts TACACS+ command authorization first.

If the TACACS+ method returns an error, `if-authenticated` permits authorization for an already authenticated user.

However, it does not bypass normal CLI privilege restrictions.

### Security Implications

Suppose the TACACS+ server normally denies:

```text
reload
```

If the TACACS+ server is available, an explicit denial prevents the command.

If the server is unavailable and `if-authenticated` is configured as the fallback, the command can be permitted.

Therefore, centralized command restrictions may not be enforced during an AAA outage.

This is an important consideration when choosing a fallback method.

---

## TACACS+ Timeout Configuration

Server timeouts determine how long IOS XE waits for a response before considering an attempt unsuccessful.

Configure a timeout for an individual TACACS+ server:

```text
R1(config)# tacacs server TAC1
R1(config-server-tacacs)# timeout 5
```

This configures a 5-second timeout for that server.

A global TACACS+ timeout can also be configured on supported IOS XE platforms:

```text
R1(config)# tacacs-server timeout 5
```

A server-specific timeout can override the global timeout.

### Timeout and Failover

Suppose TAC1 is unavailable.

IOS XE attempts TAC1 and waits according to the applicable timeout.

If the request cannot be completed, IOS XE can try another eligible server in the group.

Only after the group cannot complete the request can processing advance to the next method.

### Timeout vs. Deadtime

These terms are different:

| Mechanism | Purpose |
|---|---|
| Timeout | Determines how long IOS XE waits for a server response |
| Retransmit | Determines how many RADIUS requests are retransmitted |
| Dead detection | Determines when an unresponsive server is declared dead |
| Deadtime | Determines how long a dead server is avoided |

The total delay before local fallback depends on the server selection, timeout, retransmission, and dead-server behavior.

It is not necessarily equal to one configured timeout value.

---

## RADIUS Timeout and Retransmission

RADIUS servers support configurable timeouts and retransmission counts.

For example:

```text
R1(config)# radius server RAD1
R1(config-radius-server)# address ipv4 192.0.2.20 auth-port 1812 acct-port 1813
R1(config-radius-server)# key Cisco123
R1(config-radius-server)# timeout 3
R1(config-radius-server)# retransmit 2
```

This configures:

- A response timeout of 3 seconds.
- Two retransmissions after the initial request, subject to the applicable server behavior.

If the server does not respond, IOS XE can retry the request and then attempt another eligible server.

A RADIUS Access-Reject is not a timeout.

**An Access-Reject indicates that the server responded and explicitly denied authentication.**

It does not trigger local fallback.

---

## RADIUS Dead-Server Detection

IOS XE supports detecting unresponsive RADIUS servers and marking them as dead.

Configure dead-server criteria:

```text
R1(config)# radius-server dead-criteria time 10 tries 3
```

The parameters are:

| Parameter | Purpose |
|---|---|
| `time 10` | Requires at least 10 seconds since the last valid server response |
| `tries 3` | Requires the configured number of consecutive request timeouts |

**Both conditions must be satisfied before the RADIUS server is marked dead.**

For example:

```text
radius-server dead-criteria time 10 tries 3
```

A server is not marked dead merely because ten seconds have elapsed without any RADIUS traffic.

The device must also experience the required consecutive timeouts.

Likewise, satisfying the timeout-count criterion alone is insufficient if the time criterion has not been met.

Cisco documents that retransmissions, rather than the initial requests, contribute to the dead-criteria timeout count.

Malformed responses can also be treated as timeouts. <https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/225882-understand-aaa-dead-detection-and.html>

### RADIUS Deadtime

Configure:

```text
R1(config)# radius-server deadtime 5
```

This specifies a deadtime of **5 minutes**.

While a server is marked dead, IOS XE normally avoids sending ordinary RADIUS requests to it and instead selects another eligible server.

Once deadtime expires, the server becomes eligible to receive requests again.

**Deadtime is measured in minutes, while dead-criteria `time` is measured in seconds.**

### Default Deadtime

The default RADIUS deadtime is 0 minutes on the platforms described by Cisco's IOS XE dead-detection documentation.

Without a configured nonzero deadtime, a server can become eligible again immediately after being marked dead.

This can result in repeated attempts to use an unresponsive server.

Configuring deadtime can improve failover efficiency.

### Server-Group Deadtime

IOS XE also supports configuring deadtime within an AAA server group on supported platforms.

For example:

```text
R1(config)# aaa group server radius RAD_SERVERS
R1(config-sg-radius)# server name RAD1
R1(config-sg-radius)# server name RAD2
R1(config-sg-radius)# deadtime 5
```

Group-specific deadtime allows different AAA server groups to use different deadtime policies.

A server may belong to multiple groups with different deadtime settings.

Therefore, a server's eligibility can differ between groups.

---

## RADIUS Automated Server Testing

IOS XE supports RADIUS automated testing to check server availability.

This is useful when RADIUS servers may become unresponsive or recover after an outage.

### Idle-Time Testing

For example:

```text
R1(config)# radius server RAD1
R1(config-radius-server)# automate-tester username radius-probe idle-time 2
```

This configures automated testing using the specified test username after the server has been idle for the configured period.

The idle-time value is measured in minutes.

### Probe-On Testing

Configure:

```text
R1(config)# radius server RAD1
R1(config-radius-server)# automate-tester username radius-probe probe-on
```

This configures proactive probing associated with server availability and recovery.

When the server responds successfully to a recovery probe, IOS XE can mark it available before its deadtime expires.

### Authentication and Accounting Probe Ports

Automated testing supports the following keywords:

| Keyword | Purpose |
|---|---|
| `ignore-auth-port` | Excludes the authentication port from automated testing |
| `ignore-acct-port` | Excludes the accounting port from automated testing |
| `idle-time <minutes>` | Specifies the inactivity interval for testing |
| `probe-on` | Enables probe-based server verification |

The precise available syntax depends on the IOS XE platform and release.

Automated test credentials and server-side policies must be configured appropriately for the selected test method.

### Automated Testing vs. Ordinary Authentication

Automated testing differs from normal user authentication.

Ordinary authentication is triggered by a user or network-access event.

Automated testing generates test traffic to determine whether a RADIUS server is responding.

It can help reduce recovery time after a server outage.

---

## Server Recovery and Failback

Failover and failback are related but distinct operations.

- **Failover:** IOS XE stops using an unavailable server and selects another server or method.
- **Failback:** A previously unavailable server becomes eligible for new requests again.

For RADIUS, failback can occur when:

- Deadtime expires.
- Automated testing determines that the server is responding.

A recovered server can then be considered for subsequent AAA requests.

This does not necessarily mean that existing authenticated sessions are reauthenticated immediately.

Server recovery generally affects future requests.

---

## VRF-Aware AAA Failover

AAA server groups can use a specific VRF for server communication.

For example:

```text
R1(config)# vrf definition MGMT
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# exit-address-family
!
R1(config)# aaa group server tacacs+ TAC_MGMT
R1(config-sg-tacacs+)# server name TAC1
R1(config-sg-tacacs+)# server name TAC2
R1(config-sg-tacacs+)# ip vrf forwarding MGMT
```

Configure login fallback:

```text
R1(config)# aaa authentication login default group TAC_MGMT local
```

The TACACS+ group communicates with its servers using the configured VRF.

If the group cannot complete authentication, the method list can fall back to `local`.

### VRF Failures

Potential causes of AAA server unavailability include:

- Missing routes in the management VRF.
- Incorrect source-interface configuration.
- Incorrect VRF association under the server group.
- ACLs blocking TACACS+ or RADIUS.
- Incorrect routing toward the AAA server.

A failed AAA request does not necessarily mean the server itself is down.

For example, the server might be reachable through the global routing table but not through the configured management VRF.

In that case, requests using the management VRF can fail.

**Local fallback can still function because the local username database does not require network connectivity.**

---

## Complete TACACS+ Failover Configuration

Consider a router with two TACACS+ servers:

| Server | Address |
|---|---|
| TAC1 | 192.0.2.10 |
| TAC2 | 192.0.2.11 |

The router should:

- Authenticate administrators using TACACS+.
- Try an alternate TACACS+ server if the selected server is unavailable.
- Fall back to the local username database if all TACACS+ servers are unavailable.
- Use local EXEC authorization when TACACS+ cannot authorize the session.
- Use local command authorization when TACACS+ command authorization is unavailable.

### Configure Local Credentials

```text
R1(config)# username admin privilege 15 secret Local123
R1(config)# enable secret Enable123
```

### Configure TACACS+ Servers

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

### Configure a Named Server Group

```text
R1(config)# aaa group server tacacs+ TAC_SERVERS
R1(config-sg-tacacs+)# server name TAC1
R1(config-sg-tacacs+)# server name TAC2
```

### Configure AAA Method Lists

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login default group TAC_SERVERS local
R1(config)# aaa authentication enable default group TAC_SERVERS enable
R1(config)# aaa authorization exec default group TAC_SERVERS local
R1(config)# aaa authorization commands 15 default group TAC_SERVERS local
R1(config)# aaa authorization config-commands
```

The resulting behavior is:

| AAA Function | Primary Method | Backup Method |
|---|---|---|
| Login authentication | TAC_SERVERS | Local username database |
| Enable authentication | TAC_SERVERS | Local enable secret |
| EXEC authorization | TAC_SERVERS | Local authorization |
| Level 15 command authorization | TAC_SERVERS | Local authorization |

If TAC1 fails, IOS XE can try TAC2.

If both servers are unavailable, the relevant method list can use local fallback.

If either TACACS+ server explicitly rejects authentication or authorization, fallback is not attempted.

### Configure VTY Access

For example:

```text
R1(config)# line vty 0 15
R1(config-line)# transport input ssh
R1(config-line)# login authentication default
```

The default login authentication method list is applied automatically to applicable VTY lines, so the explicit `login authentication default` command is generally unnecessary.

Named authentication and authorization lists require explicit line assignment.

---

## Local Fallback and Device Recovery

A local fallback account can preserve management access when centralized AAA services are unavailable.

However, fallback behavior depends on the complete AAA configuration.

For example, the following configuration is incomplete for resilient management access:

```text
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+
```

Login authentication can fall back to the local database, but EXEC authorization has no local fallback.

A more resilient configuration is:

```text
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local
```

Command authorization should also be considered:

```text
aaa authorization commands 15 default group tacacs+ local
```

### Console Recovery

Console authorization is disabled by default.

It can be enabled using:

```text
R1(config)# aaa authorization console
```

When console authorization is enabled, ensure that appropriate fallback methods are configured.

Otherwise, AAA server failures can interfere with console-based recovery.

Avoid making changes to AAA settings without maintaining an alternative administrative session.

---

## IOS XE Verification

### Verify TACACS+ Server Configuration

Use:

```text
R1# show running-config | section ^tacacs server
```

Verify:

- Server IP addresses.
- Timeout values.
- Shared secrets.
- Configured servers.

### Verify AAA Method Lists

Use:

```text
R1# show running-config | include ^aaa authentication
R1# show running-config | include ^aaa authorization
```

Check whether the required fallback methods are present.

For example:

```text
aaa authentication login default group TAC_SERVERS local
aaa authorization exec default group TAC_SERVERS local
```

### Verify Server Groups

Use:

```text
R1# show running-config | section ^aaa group server
```

Verify the servers associated with each named group.

### Verify RADIUS Server State

Use:

```text
R1# show aaa servers
```

This displays AAA server information, including server states and statistics on supported IOS XE platforms.

Also use:

```text
R1# show radius statistics
```

The available fields depend on the platform and release.

Look for information such as:

- Server status.
- Requests and responses.
- Timeouts.
- Retransmissions.
- Dead-server events.

### Verify TACACS+ Connectivity

Use:

```text
R1# show tacacs
```

This provides TACACS+ server information and counters on supported platforms.

### Debug AAA Authentication

Enable:

```text
R1# debug aaa authentication
```

This is useful for identifying the authentication methods attempted and their results.

### Debug AAA Authorization

Enable:

```text
R1# debug aaa authorization
```

This is useful for determining whether the router used TACACS+, local authorization, or another method.

### Debug TACACS+

Enable:

```text
R1# debug tacacs authentication
R1# debug tacacs authorization
```

Check:

- Which AAA server was contacted.
- Whether the server responded.
- Whether an error or explicit denial occurred.
- Whether the next method was attempted.

### Debug RADIUS

Enable:

```text
R1# debug radius authentication
```

On platforms supporting detailed AAA transaction debugging, additional debug or platform-trace facilities may be available.

After troubleshooting:

```text
R1# undebug all
```

### Testing Failover

A practical lab test is to:

1. Configure at least two AAA servers.
2. Verify successful authentication using the primary server.
3. Make the primary server unavailable.
4. Verify authentication through an alternate server.
5. Make all AAA servers unavailable.
6. Verify local authentication fallback.
7. Restore the servers.
8. Verify subsequent requests can use remote AAA again.

A separate test should deliberately use incorrect credentials while the server is reachable.

**That test should fail without local fallback.**

This confirms the difference between a remote AAA rejection and an AAA server outage.

---

## Common AAA Failover Problems

| Problem | Possible Cause |
|---|---|
| Local fallback not attempted after invalid password | AAA server explicitly rejected authentication |
| Local fallback not attempted after server failure | Another server remains available, or the server group has not returned an error |
| Local authentication succeeds but EXEC is denied | EXEC authorization lacks an appropriate fallback |
| User logs in locally but receives insufficient privileges | Local authorization or privilege configuration is incomplete |
| Enable fallback does not work | `enable` method or local enable secret not configured |
| Command authorization fails during AAA outage | Command authorization method list lacks a suitable fallback |
| Authentication takes too long during an outage | Excessive timeout or retransmission settings |
| RADIUS server is not marked dead | Dead-criteria conditions have not both been satisfied |
| Dead RADIUS server is retried repeatedly | Deadtime set to 0 or server state changing rapidly |
| RADIUS server remains unused after recovery | Deadtime has not expired or recovery has not been detected |
| AAA servers reachable globally but unavailable to AAA | Incorrect VRF or source-interface configuration |
| Login denied even though the local username exists | Remote server returned FAIL instead of ERROR |
| Console access fails during AAA outage | Console authorization enabled without suitable fallback |

---

## Key Points

- **AAA server failover** occurs between servers in an AAA server group.
- **AAA method fallback** occurs between methods in an AAA method list.
- IOS XE attempts the next AAA method only when the current method returns ERROR.
- **PASS and FAIL both stop method-list processing.**
- A TACACS+ authentication rejection does not trigger local fallback.
- A RADIUS Access-Reject does not trigger local fallback.
- Server timeouts and communication errors can cause an AAA server group to return ERROR.
- `aaa authentication login default group tacacs+ local` enables local username authentication fallback.
- `local` uses the local username database.
- `local-case` uses the local username database with case-sensitive username matching.
- Enable authentication uses `enable` for local enable-secret fallback, not `local`.
- `aaa authentication enable default group tacacs+ enable` provides local enable authentication fallback.
- EXEC authorization requires its own fallback method list.
- `aaa authorization exec default group tacacs+ local` allows local EXEC authorization fallback.
- `if-authenticated` allows an already authenticated user to pass authorization after an earlier method returns ERROR.
- `if-authenticated` does not independently supply local privilege attributes.
- Command authorization requires its own fallback method list.
- `aaa authorization commands 15 default group tacacs+ local` allows local command authorization fallback.
- Local authorization does not provide the same granular command policies as TACACS+.
- A TACACS+ authorization denial prevents local or `if-authenticated` fallback.
- TACACS+ server-specific timeouts can be configured with `timeout <seconds>` under `tacacs server`.
- RADIUS server-specific timeouts and retransmissions can be configured with `timeout` and `retransmit`.
- **Timeout and deadtime are different mechanisms.**
- `radius-server dead-criteria time <seconds> tries <count>` controls RADIUS dead-server detection.
- **Both configured dead-criteria conditions must be satisfied** before a RADIUS server is marked dead.
- `radius-server deadtime <minutes>` controls how long a dead RADIUS server is avoided.
- The documented default RADIUS deadtime is 0 minutes.
- RADIUS automated testing can help determine whether an unavailable server has recovered.
- A recovered server becomes eligible for subsequent AAA requests.
- VRF and source-interface configuration can affect AAA server reachability and failover.
- Local fallback can preserve access during AAA outages, but authentication, EXEC authorization, and command authorization must be considered separately.
- Use `show aaa servers`, `show tacacs`, `show radius statistics`, and AAA debugging commands to verify failover behavior.

---

## References

- [Cisco IOS XE: AAA Dead-Server Detection](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_aaa/configuration/xe-3s/sec-usr-aaa-xe-3s-book/sec-aaa-dead-server.html)
- [Cisco: Understand AAA Dead Detection and Deadtime on IOS XE](https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/225882-understand-aaa-dead-detection-and.html)
- [Cisco IOS XE: AAA Server Groups](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-rad-aaa-server-groups.html)
- [Cisco IOS XE: RADIUS Server Load Balancing](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9400/software/release/17-9/configuration_guide/sec/b_179_sec_9400_cg/radius_server_load_balancing.html)
- [RFC 8907: The TACACS+ Protocol](https://www.rfc-editor.org/rfc/rfc8907)
- [RFC 2865: Remote Authentication Dial In User Service (RADIUS)](https://www.rfc-editor.org/rfc/rfc2865)
