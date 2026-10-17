# AAA Accounting

**AAA accounting** records user activity and sends accounting information to a TACACS+ or RADIUS server.

Accounting allows administrators to track events such as:

- When a user starts and ends an EXEC session.
- Which CLI commands a user executes.
- When a device reloads.
- When a user establishes or terminates a network connection.
- How long a session lasts and which network resources it consumes.

Accounting is separate from authentication and authorization:

| AAA Function | Purpose |
|---|---|
| Authentication | Verifies the user's identity |
| Authorization | Determines which services or commands the user can access |
| Accounting | Records what the user does |

For example, a TACACS+ server might authenticate an administrator and authorize the `reload` command.

AAA accounting can then record that the administrator executed `reload`.

**Accounting does not determine whether a command is permitted.**

## What Counts as an Activity?

An **activity** is the unit of work tracked by the selected accounting type. It does **not** always mean an individual command or an entire login session.

| Accounting Type | What Counts as an Activity? | Example |
|---|---|---|
| `exec` | One EXEC session | An SSH user opens a CLI session, then logs out |
| `commands <level>` | One eligible CLI command | The user executes `show running-config` |
| `network` | One supported network-service session | A PPP session is established, then disconnected |
| `connection` | One supported outbound connection | An administrator initiates a connection from the device |
| `system` | One supported system-level event or process | A device reload |

For example, suppose a user connects by SSH, executes ten commands, and logs out:

- **EXEC accounting** can record one session START and one session STOP, not a START/STOP pair for each of the ten commands.
- **Command accounting** can generate separate records for each command whose privilege level has an accounting method list. Each command is the tracked activity, rather than the entire SSH session.
- **System accounting** is independent of that user session and tracks supported device-level events.

The terms **START** and **STOP** refer to the beginning and end of the *tracked activity*. Their meaning therefore depends on the accounting type and the record mode (`start-stop`, `stop-only`, or `wait-start`).

---

## AAA Accounting Operation

When accounting is enabled, IOS XE generates records for events covered by the configured accounting method lists.

For example, consider an administrator establishing an SSH session:

```text
Administrator                  R1                 TACACS+ Server
      |                       |                        |
      |------ SSH Login ----->|                        |
      |                       |--- Authentication ---->|
      |                       |<------ PASS -----------|
      |                       |                        |
      |                       |--- EXEC Accounting --->|
      |                       |        START           |
      |                       |                        |
      |<--- EXEC Session -----|                        |
      |                       |                        |
      |-------- logout -----> |                        |
      |                       |--- EXEC Accounting --->|
      |                       |        STOP            |
      |                       |                        |
```

A start record indicates that a tracked activity has begun.

A stop record indicates that it has ended.

Accounting records can include information such as:

| Attribute | Purpose |
|---|---|
| Username | Identifies the user |
| Source address | Identifies the remote client |
| Device address | Identifies the network device |
| Session ID | Associates records with a particular session |
| Start/stop status | Identifies the accounting event |
| Timestamp | Identifies when an event occurred |
| Command | Identifies an executed CLI command |
| Elapsed time | Indicates session duration |

The exact attributes depend on the accounting type, protocol, and server implementation.

### Default Behavior

AAA accounting is **disabled by default**.

Configuring AAA authentication or authorization does not automatically enable accounting.

For example:

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login default group tacacs+ local
R1(config)# aaa authorization exec default group tacacs+ local
```

These commands configure authentication and authorization.

They do not configure accounting.

Accounting requires separate `aaa accounting` commands.

---

## Accounting Record Types

IOS XE supports several accounting record types.

The most important are:

| Record Type | Behavior |
|---|---|
| `start-stop` | Generates a START record when an activity begins and a STOP record when it ends |
| `wait-start` | Sends START and STOP records, waiting for acknowledgment of START before allowing the activity on platforms that support it |
| `stop-only` | Generates a STOP record when an activity ends |
| `none` | Disables accounting for the specified method list |

### Start-Stop Accounting

Configure:

```text
R1(config)# aaa accounting exec default start-stop group tacacs+
```

For an EXEC session:

1. The user starts an EXEC session.
2. IOS XE generates a START record.
3. The user works in the CLI.
4. The session ends.
5. IOS XE generates a STOP record.

START records are normally sent asynchronously. The device does not have to wait for confirmation that the accounting server received the START record before allowing the session to proceed.

### Wait-Start Accounting

The `wait-start` mode is designed to send both START and STOP records, but **the activity is not allowed to begin until the accounting server acknowledges its START record**.

For example, on a platform that supports it:

```text
R1(config)# aaa accounting exec default wait-start group tacacs+
```

For EXEC accounting, the user must wait for the START accounting acknowledgment before beginning the EXEC session. For command accounting, an eligible command waits for the START acknowledgment before executing.

Compare the modes:

| Mode | START Record | Must Wait for START Acknowledgment? | STOP Record |
|---|---|---|---|
| `start-stop` | Yes | No | Yes |
| `wait-start` | Yes | **Yes, if supported** | Yes |
| `stop-only` | No | No | Yes |

If the server is unreachable, supported `wait-start` behavior can delay or prevent the tracked activity from starting. This can affect administrative access, so use it cautiously.

**IOS XE support limitation:** `wait-start` is not universally supported. In my lab, I got the following warning message:

```text
*Oct 17 00:41:28.768: %PARSER-5-HIDDEN: Warning!!! ' aaa accounting commands 15 default wait-start group tacacs+ ' is a hidden command. Use of this command is not recommended/supported and will be removed in future.
```

### Stop-Only Accounting

Configure:

```text
R1(config)# aaa accounting exec default stop-only group tacacs+
```

The device sends a STOP record when the session ends.

No corresponding START record is generated by this method.

Stop-only accounting is useful when only the completed activity needs to be recorded.

### None

Configure:

```text
R1(config)# aaa accounting exec default none
```

This disables EXEC accounting for the default method list.

`none` is not a server group or fallback protocol. It specifies that accounting should not be performed.

---

## IOS XE Accounting Configuration

The general syntax is:

```text
aaa accounting {exec | commands <level> | network | connection | system} {default | <list-name>} {start-stop | stop-only | wait-start | none} <methods>
```

The exact available keywords depend on the accounting type, IOS XE platform, and release. In particular, some IOS XE versions do not support `wait-start` and map it to `start-stop`.

For example:

```text
R1(config)# aaa accounting exec default start-stop group tacacs+
```

The main parameters are:

| Parameter | Purpose |
|---|---|
| `exec` | Accounts for EXEC sessions |
| `commands <level>` | Accounts for commands assigned to a specified privilege level |
| `network` | Accounts for network services and connections, such as PPP |
| `connection` | Accounts for outbound connections initiated from the device |
| `system` | Accounts for system-level events |
| `default` | Configures the default accounting method list |
| `<list-name>` | Configures a named accounting method list |
| `start-stop` | Generates START and STOP records |
| `wait-start` | Sends START and STOP and waits for acknowledgment of START, where supported |
| `stop-only` | Generates STOP records |
| `none` | Disables accounting for that list |
| `group tacacs+` | Uses the default TACACS+ server group |
| `group radius` | Uses the default RADIUS server group |
| `group <name>` | Uses a named AAA server group |

**System accounting supports only the `default` method list.**

---

## EXEC Accounting

EXEC accounting records the beginning and end of user EXEC sessions.

Configure:

```text
R1(config)# aaa accounting exec default start-stop group tacacs+
```

For example, when an administrator connects through SSH, IOS XE generates an EXEC START record.

When the administrator logs out, IOS XE generates an EXEC STOP record.

EXEC accounting can record information such as:

- Username.
- Remote IP address.
- Terminal line.
- Session start time.
- Session end time.
- Session duration.

### EXEC Accounting with RADIUS

RADIUS also supports EXEC accounting:

```text
R1(config)# aaa accounting exec default start-stop group radius
```

RADIUS accounting uses accounting packets sent to a RADIUS server, normally on UDP port 1813.

### Named EXEC Accounting Lists

Create a named list:

```text
R1(config)# aaa accounting exec SSH_ACCT start-stop group tacacs+
```

Apply it to VTY lines:

```text
R1(config)# line vty 0 4
R1(config-line)# accounting exec SSH_ACCT
```

The named list is used for EXEC sessions established through those lines.

A default EXEC accounting list applies to applicable lines automatically unless overridden.

---

## Command Accounting

**Command accounting records CLI commands executed by users.**

It is particularly useful for tracking administrative changes and investigating configuration modifications.

For example, command accounting could record:

```text
configure terminal
interface Ethernet0/0
shutdown
```

Command accounting and command authorization are independent.

A command can be accounted for even if TACACS+ command authorization is not configured.

Likewise, command authorization does not automatically enable command accounting.

### Command Accounting Configuration

Configure command accounting for privilege level 15:

```text
R1(config)# aaa accounting commands 15 default start-stop group tacacs+
```

Alternatively:

```text
R1(config)# aaa accounting commands 15 default stop-only group tacacs+
```

For command accounting, `stop-only` is commonly used because the principal purpose is to record the completed command.

### Command Privilege Levels

Like command authorization, command accounting is configured separately for each privilege level.

For example:

```text
R1(config)# aaa accounting commands 0 default stop-only group tacacs+
R1(config)# aaa accounting commands 1 default stop-only group tacacs+
R1(config)# aaa accounting commands 15 default stop-only group tacacs+
```

These commands enable accounting for commands at privilege levels 0, 1, and 15.

**The configured level refers to the command's privilege level, not the user's current privilege level.**

For example, a privilege level 15 administrator can execute commands assigned to privilege levels 1 and 15.

If only the level 15 accounting list exists, level 1 commands are not automatically accounted for by that list.

Commands assigned to custom privilege levels 2 through 14 require corresponding accounting method lists if they must be tracked.

### Command Accounting and Configuration Mode

Command accounting can record configuration-mode commands when the applicable accounting configuration and platform support are present.

For example:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# description LAN
```

These commands can generate individual accounting records.

**`aaa authorization config-commands` controls configuration-mode command authorization, not command accounting.**

It is not a prerequisite for AAA command accounting.

Therefore, configuration commands can be recorded even when their execution is not being authorized individually by TACACS+.

### Named Command Accounting Lists

Configure a named method list:

```text
R1(config)# aaa accounting commands 15 SSH_CMDS stop-only group tacacs+
```

Apply it:

```text
R1(config)# line vty 0 4
R1(config-line)# accounting commands 15 SSH_CMDS
```

The named list applies to privilege level 15 commands executed through those VTY lines.

### Command Authorization vs. Command Accounting

Consider:

```text
R1(config)# aaa authorization commands 15 default group tacacs+ local
R1(config)# aaa accounting commands 15 default stop-only group tacacs+
```

The first command determines whether applicable level 15 commands are permitted.

The second records command activity.

| Feature | Command Authorization | Command Accounting |
|---|---|---|
| Main command | `aaa authorization commands` | `aaa accounting commands` |
| Purpose | Permit or deny commands | Record commands |
| Uses privilege levels | Yes | Yes |
| Requires the other feature | No | No |
| TACACS+ role | Evaluates command policies | Receives accounting records |

An administrator can receive permission to execute a command through authorization and then generate an accounting record for its execution.

---

## System Accounting

System accounting records system-level events not necessarily associated with individual users.

For example:

- Device reloads.
- Device startup.
- Other supported system-level events.

Configure:

```text
R1(config)# aaa accounting system default start-stop group tacacs+
```

Or:

```text
R1(config)# aaa accounting system default stop-only group tacacs+
```

Unlike EXEC and command accounting, **system accounting supports only the default method list**.

### System Accounting and Server Availability

System accounting has an important availability consideration.

IOS XE supports a mechanism that attempts to guarantee that system accounting is performed before other accounting activity.

The relevant command is:

```text
aaa accounting system guarantee-first
```

This behavior is enabled by default on platforms that support it.

If system accounting is enabled but the accounting server is unreachable during device startup, this behavior can delay access to the device.

To disable it:

```text
R1(config)# no aaa accounting system guarantee-first
```

This can help prevent management access delays when the accounting server is unavailable.

**The command does not disable system accounting itself.**

---

## Connection Accounting

Connection accounting records outbound connections initiated from an IOS XE device.

For example, an administrator connected to R1 might initiate a connection from R1 to another device.

Connection accounting can track these outbound connection activities.

Configure:

```text
R1(config)# aaa accounting connection default start-stop group tacacs+
```

This differs from EXEC accounting:

- EXEC accounting tracks the user's EXEC session on the device.
- Connection accounting tracks supported outbound connections initiated through the device.

Connection accounting is generally less important for enterprise device administration than EXEC and command accounting, but it is useful to recognize the configuration and distinction.

---

## Network Accounting

Network accounting tracks user access to network services.

It is commonly associated with access technologies such as PPP and other authenticated network sessions.

For example:

```text
R1(config)# aaa accounting network default start-stop group radius
```

Network accounting can record:

- Username.
- Session start and end.
- Assigned IP address.
- Session duration.
- Traffic counters, when supported.

### PPP Accounting

For example:

```text
R1(config)# aaa authentication ppp default group radius
R1(config)# aaa authorization network default group radius
R1(config)# aaa accounting network default start-stop group radius
```

Apply PPP authentication and accounting to an appropriate interface:

```text
R1(config)# interface Serial0/0
R1(config-if)# encapsulation ppp
R1(config-if)# ppp authentication chap
R1(config-if)# ppp accounting default
```

The exact interface configuration depends on the platform and access service.

Network accounting is more closely associated with RADIUS-based access services than with granular CLI command tracking.

---

## TACACS+ vs. RADIUS Accounting

Both TACACS+ and RADIUS support accounting, but they are commonly used for different purposes.

| Feature | TACACS+ | RADIUS |
|---|---|---|
| Transport | TCP | UDP |
| Standard port | TCP 49 | UDP 1813 for accounting |
| EXEC accounting | Yes | Yes |
| Command accounting | Yes | Platform-dependent support |
| Network session accounting | Yes | Yes |
| Common use | Network device administration | Network access and subscriber sessions |

TACACS+ is especially useful for administrator command accounting.

RADIUS is frequently used to track authenticated network access sessions and resource usage.

### TACACS+ Accounting Records

TACACS+ uses accounting messages to report the activity.

A TACACS+ accounting request contains attributes describing the event.

For example, a command accounting record may include:

```text
service=shell
cmd=show ip route
```

The accounting server stores the information according to its configuration.

### RADIUS Accounting Packets

RADIUS accounting uses two particularly important status values:

| RADIUS Acct-Status-Type | Meaning |
|---|---|
| `Start` | Session has begun |
| `Stop` | Session has ended |
| `Interim-Update` | Session is still active; updated information is provided |

RADIUS accounting packets are distinct from RADIUS authentication packets.

A RADIUS Access-Accept is not an accounting record.

An accounting request is acknowledged by a RADIUS Accounting-Response.

---

## Accounting Method Lists and Server Groups

Accounting method lists can specify one or more server groups.

For example:

```text
R1(config)# aaa accounting exec default start-stop group TAC_PRIMARY group TAC_BACKUP
```

The device attempts to send accounting records through the first configured server group.

If that group is unavailable, it can attempt the next group.

### Named Server Groups

Define a TACACS+ server:

```text
R1(config)# tacacs server TAC1
R1(config-server-tacacs)# address ipv4 192.0.2.10
R1(config-server-tacacs)# key Cisco123
```

Create a named server group:

```text
R1(config)# aaa group server tacacs+ TAC_ACCT
R1(config-sg-tacacs+)# server name TAC1
```

Use the group for accounting:

```text
R1(config)# aaa accounting exec default start-stop group TAC_ACCT
R1(config)# aaa accounting commands 15 default stop-only group TAC_ACCT
```

This directs those accounting records to servers belonging to `TAC_ACCT`.

### Accounting Server Failover

With:

```text
aaa accounting exec default start-stop group TAC_PRIMARY group TAC_BACKUP
```

The server groups are attempted in order.

An unavailable server can trigger another server attempt according to the method-list and server-group behavior.

This is different from sending the same record to both groups simultaneously.

### Broadcast Accounting

IOS XE supports sending accounting records to multiple groups using `broadcast`.

For example:

```text
R1(config)# aaa accounting exec default start-stop broadcast group TAC_PRIMARY group TAC_BACKUP
```

With broadcast accounting, records are sent to both server groups rather than using the second group only when the first fails.

Within each group, normal server failover behavior still applies.

This is useful when accounting records must be delivered to multiple independent accounting systems.

---

## Interim Accounting Updates

Some accounting sessions can remain active for long periods.

IOS XE supports interim accounting updates to provide new information while a session is still active.

Configure:

```text
R1(config)# aaa accounting update periodic 10
```

This enables periodic interim updates at **10-minute intervals** for supported accounting sessions.

The periodic value is specified in minutes.

### Updates When New Information Is Available

Configure:

```text
R1(config)# aaa accounting update newinfo
```

This sends interim accounting records when new accounting information becomes available.

For example, in a network access session, new information might include an IP address assigned during negotiation.

### Combining Update Types

Where supported, the two options can be combined:

```text
R1(config)# aaa accounting update newinfo periodic 10
```

This allows updates both when new information becomes available and periodically.

Interim updates are mainly useful for network/session accounting.

They should not be confused with individual command accounting records.

**Frequent interim updates can increase accounting traffic and server load.**

---

## Suppressing Accounting for Sessions Without Usernames

Some sessions do not have an associated username.

For example, a line might use an authentication method that does not establish an individual username.

IOS XE can still generate accounting records for these sessions.

To suppress accounting records for sessions with a null username:

```text
R1(config)# aaa accounting suppress null-username
```

This prevents accounting records from being generated for sessions whose username is NULL.

It does not disable accounting for users with valid usernames.

---

## Accounting Records for Authentication Failures

By default, ordinary accounting records are not necessarily generated for login attempts that fail authentication.

IOS XE supports generating accounting stop records for authentication failures:

```text
R1(config)# aaa accounting send stop-record authentication failure
```

This is relevant when accounting information is needed for unsuccessful login or session-establishment attempts.

Another supported command is:

```text
R1(config)# aaa accounting send stop-record always
```

This causes accounting STOP records to be sent even when no corresponding START record was previously sent.

These commands change accounting-record generation behavior.

They do not change authentication success or failure decisions.

---

## Accounting and Sensitive Information

Command accounting records can contain information about commands entered by administrators.

Commands sometimes include sensitive arguments.

For security reasons, IOS XE supports masking certain sensitive information in command accounting records.

On supported platforms, the following command controls whether sensitive keys are visible in accounting records:

```text
aaa accounting commands visible-keys
```

This allows sensitive command arguments to be included without the normal masking.

**Avoid enabling `aaa accounting commands visible-keys` unless explicitly required.**

Accounting records should be protected because they can reveal:

- Administrative usernames.
- Device addresses.
- Configuration changes.
- Commands containing sensitive parameters.

Access to accounting server logs should be restricted appropriately.

---

## Complete TACACS+ Accounting Example

Consider a router using a TACACS+ server at `192.0.2.10`.

### Configure the TACACS+ Server

```text
R1(config)# tacacs server TAC1
R1(config-server-tacacs)# address ipv4 192.0.2.10
R1(config-server-tacacs)# key Cisco123
```

### Configure a Local Administrator

```text
R1(config)# username admin privilege 15 secret Local123
```

### Configure AAA

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login default group tacacs+ local
R1(config)# aaa authorization exec default group tacacs+ local
R1(config)# aaa authorization commands 15 default group tacacs+ local
R1(config)# aaa authorization config-commands
```

### Configure Accounting

```text
R1(config)# aaa accounting exec default start-stop group tacacs+
R1(config)# aaa accounting commands 1 default stop-only group tacacs+
R1(config)# aaa accounting commands 15 default stop-only group tacacs+
R1(config)# aaa accounting system default start-stop group tacacs+
```

This configuration provides:

| Event | Accounting Behavior |
|---|---|
| Administrator starts an EXEC session | EXEC START record |
| Administrator ends an EXEC session | EXEC STOP record |
| Administrator executes a level 1 command | Command accounting record |
| Administrator executes a level 15 command | Command accounting record |
| Device experiences a supported system event | System accounting record |

Login authentication, EXEC authorization, command authorization, and accounting operate independently.

A successful accounting operation is not required to grant a user additional privileges or permissions.

---

## IOS XE Verification

### Display Accounting Configuration

Use:

```text
R1# show running-config | include ^aaa accounting
```

Example output:

```text
aaa accounting exec default start-stop group tacacs+
aaa accounting commands 1 default stop-only group tacacs+
aaa accounting commands 15 default stop-only group tacacs+
aaa accounting system default start-stop group tacacs+
```

### Show Active Accounting Sessions

Use:

```text
R1# show accounting
```

This displays active accounting information tracked by the device.

It is particularly useful for checking active accountable sessions.

It is not a complete historical accounting database.

For historical activity, check the accounting server logs.

### Debug AAA Accounting

Enable:

```text
R1# debug aaa accounting
```

Then perform an activity covered by an accounting method list.

For example:

```text
R1# show running-config
```

Check whether an accounting event is generated for the command's privilege level.

### Debug TACACS+ Accounting

On supported IOS XE images, use:

```text
R1# debug tacacs accounting
```

This helps identify TACACS+ accounting requests and responses.

### Debug RADIUS Accounting

For RADIUS accounting:

```text
R1# debug radius accounting
```

Use the available RADIUS debugging options on the specific platform to inspect accounting exchanges.

After testing:

```text
R1# undebug all
```

### Verify Server Connectivity

For TACACS+:

```text
R1# show tacacs
```

For RADIUS:

```text
R1# show radius statistics
```

These commands can help identify server communication failures and accounting statistics.

The exact commands and displayed counters vary by platform.

### Verify Server Logs

The accounting server is the authoritative place to examine historical accounting records.

For TACACS+ command accounting, verify:

- The correct username is recorded.
- The intended command is recorded.
- The correct device is identified.
- The accounting record is accepted and stored.

For RADIUS session accounting, verify START, STOP, and any interim records.

---

## Common Accounting Problems

| Problem | Possible Cause |
|---|---|
| No accounting records generated | No applicable `aaa accounting` method list configured |
| EXEC sessions are recorded but commands are not | EXEC accounting configured without command accounting |
| Some commands are recorded but others are not | Accounting configured only for certain privilege levels |
| Named accounting list is not used | List not applied to the relevant line |
| Accounting records do not reach the server | Server unreachable, incorrect key, port, or server group |
| Only STOP records appear | `stop-only` configured |
| `wait-start` does not delay activity | Platform maps unsupported `wait-start` to `start-stop` |
| No records appear for failed login attempts | Authentication-failure stop accounting not enabled |
| Records have no username | Session has no associated username |
| System accounting delays startup access | System accounting server unreachable with `guarantee-first` behavior |
| Accounting records appear on only one server group | Sequential failover rather than broadcast accounting |
| Commands are permitted despite accounting failure | Accounting records activity; it is not command authorization |

### Troubleshooting Sequence

When accounting does not work as expected:

1. Verify that `aaa new-model` is enabled.
2. Verify that the correct accounting type is configured.
3. Check whether the command or session belongs to an accounted privilege level.
4. Verify the default or named method list.
5. Check the relevant line configuration for named lists.
6. Verify TACACS+ or RADIUS server-group configuration.
7. Check server reachability, ports, and shared secrets.
8. Enable `debug aaa accounting` and appropriate protocol debugging.
9. Check whether the server receives and stores the records.
10. Confirm the expected record type: START, STOP, or interim update.

---

## Key Points

- AAA accounting records user activity; it does not authenticate or authorize users.
- AAA accounting is **disabled by default**.
- Authentication and authorization do not automatically enable accounting.
- Configure accounting with `aaa accounting`.
- The meaning of an **activity** depends on its accounting type: a whole EXEC session, a single eligible command, a network/connection session, or a supported system event.
- `start-stop` generates START and STOP records asynchronously; the activity does not wait for START acknowledgment.
- `wait-start` is designed to wait for acknowledgment of START before allowing the tracked activity and also sends a STOP record.
- Some IOS XE releases do not support `wait-start` and explicitly map it to `start-stop`.
- `stop-only` generates STOP records.
- `none` disables accounting for the specified method list.
- EXEC accounting tracks the beginning and end of EXEC sessions.
- Configure EXEC accounting using `aaa accounting exec`.
- Command accounting tracks CLI commands executed by users.
- Configure command accounting using `aaa accounting commands <level>`.
- Command accounting is configured separately for privilege levels **0 through 15**.
- **The configured command accounting level refers to the command's privilege level, not the user's current privilege level.**
- Command accounting and command authorization are independent.
- `aaa authorization config-commands` is not required to enable configuration-command accounting.
- System accounting records system-level events such as device reloads.
- System accounting supports only the `default` method list.
- `no aaa accounting system guarantee-first` disables the system-accounting-first requirement, helping avoid startup access delays if the accounting server is unreachable.
- Connection accounting tracks supported outbound connections.
- Network accounting tracks authenticated network service sessions and resource usage.
- TACACS+ and RADIUS both support AAA accounting.
- RADIUS accounting normally uses UDP port 1813.
- `broadcast` allows accounting records to be sent to multiple server groups.
- `aaa accounting update periodic <minutes>` configures periodic interim accounting updates.
- `aaa accounting update newinfo` sends interim updates when new information becomes available.
- `aaa accounting suppress null-username` suppresses records for sessions with null usernames.
- `aaa accounting send stop-record authentication failure` enables stop records for supported authentication failures.
- `aaa accounting send stop-record always` permits stop records even without corresponding start records.
- `aaa accounting commands visible-keys` permits normally masked sensitive command information to appear in accounting records.
- Use `show accounting` to inspect active accounting information.
- Use `debug aaa accounting` to troubleshoot accounting events.
- The accounting server logs are the primary source for historical accounting information.

---

## References

- [Cisco IOS XE - Configuring Accounting](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_aaa/configuration/xe-3e/sec-usr-aaa-xe-3e-book/sec-cfg-accountg.html)
- [Cisco IOS XE Security Command Reference - AAA Accounting](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/security/a1/sec-a1-cr-book/sec-cr-a1.html)
- [RFC 8907: The TACACS+ Protocol](https://www.rfc-editor.org/rfc/rfc8907)
- [RFC 2866: RADIUS Accounting](https://www.rfc-editor.org/rfc/rfc2866)
- [Cisco IOS XE 17.14 System Message Guide: Unsupported wait-start warning](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/17_xe/syslogs/17-14-x/b-system-message-guide-17-14-x.html)
