
# Command Authorization

**AAA command authorization** controls whether a user is permitted to execute individual CLI commands on a Cisco IOS XE device.

Unlike EXEC authorization, which determines whether a user can start an EXEC session, command authorization evaluates specific commands when the user attempts to execute them.

Command authorization is particularly useful with TACACS+, which supports centralized policies that permit or deny individual commands and their arguments.

For example, an administrator might be permitted to execute:

```text
show ip route
show running-config
```

But denied:

```text
configure terminal
reload
```

Command authorization is separate from login authentication, enable authentication, and EXEC authorization.

---

## Command Authorization Operation

When command authorization is enabled, IOS XE evaluates applicable commands against the configured AAA authorization method list.

For TACACS+, the operation is:

1. The user enters a CLI command.
2. IOS XE determines whether the command is subject to an enabled command authorization method list.
3. The router sends a TACACS+ authorization request containing the username, command, and arguments.
4. The TACACS+ server evaluates its configured command authorization policy.
5. The server returns a permit or deny decision.
6. IOS XE executes the command only if authorization succeeds.

For example:

```text
Administrator                R1                TACACS+ Server
      |                      |                       |
      |--- show ip route --->|                       |
      |                      |                       |
      |                      |--- Authorization ---->|
      |                      |    cmd=show           |
      |                      |    cmd-arg=ip         |
      |                      |    cmd-arg=route      |
      |                      |                       |
      |                      |<------ PASS ----------|
      |                      |                       |
      |<-- Routing Table ----|                       |
```

If the server denies the request, IOS XE does not execute the command.

A user can therefore have privilege level 15 but still be denied certain privilege level 15 commands.

**Privilege levels determine which commands a user is normally eligible to execute. AAA command authorization adds a separate authorization check.**

---

## Default Behavior

By default, AAA command authorization is disabled.

Without an `aaa authorization commands` method list:

- IOS XE does not contact an AAA server to authorize individual commands.
- Commands are permitted or denied according to normal CLI privilege levels and other applicable restrictions.
- A user at privilege level 15 can normally execute privilege level 15 commands.
- A user at privilege level 1 cannot normally execute privilege level 15 commands.

For example:

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login default group tacacs+ local
R1(config)# aaa authorization exec default group tacacs+ local
```

These commands enable login authentication and EXEC authorization.

They do **not** enable command authorization.

Even if the TACACS+ server assigns privilege level 15, it is not asked to authorize individual commands unless the corresponding command authorization method list is configured.

**EXEC command authorization and configuration-mode command authorization are separate behaviors.** Configuring a command authorization method list does not necessarily mean that configuration-mode commands will be authorized.

---

## IOS XE Command Authorization Configuration

Configure command authorization using:

```text
aaa authorization commands <level> {default | <list-name>} <method1> [<method2> ...]
```

For example:

```text
R1(config)# aaa authorization commands 15 default group tacacs+ local
```

The parameters are:

| Parameter | Purpose |
|---|---|
| `commands` | Specifies command authorization |
| `<level>` | Privilege level of commands subject to the method list, from 0 through 15 |
| `default` | Creates the default command authorization method list for that privilege level |
| `<list-name>` | Creates a named command authorization method list |
| `group tacacs+` | Uses the default TACACS+ server group |
| `group <name>` | Uses a named AAA server group |
| `local` | Uses local authorization |
| `if-authenticated` | Permits authorization if the user has already authenticated |
| `none` | Permits authorization without another authorization check |

The example configures TACACS+ authorization for level 15 commands, with local authorization as a fallback if the TACACS+ method returns an error.

**An explicit TACACS+ authorization denial does not trigger fallback to the next method.**

To explicitly enable authorization of configuration-mode commands, also configure:

```text
R1(config)# aaa authorization config-commands
```

The behavior of this command is covered later in the page.

---

## Command Authorization Privilege Levels

Command authorization is configured separately for each privilege level.

IOS XE supports privilege levels **0 through 15**.

The most important levels are:

| Level | Typical Commands |
|---|---|
| 0 | Basic commands such as `enable`, `disable`, `exit`, `help`, and `logout` |
| 1 | User EXEC commands |
| 2-14 | Custom privilege levels |
| 15 | Privileged EXEC and configuration commands |

### Level 15 Authorization

For example:

```text
R1(config)# aaa authorization commands 15 default group tacacs+
```

This enables TACACS+ authorization for commands assigned to privilege level 15.

Typical examples include:

```text
show running-config
configure terminal
reload
```

Configuration-mode commands can also be subject to the level 15 authorization list when configuration-command authorization is enabled.

### Level 1 Authorization

Configure:

```text
R1(config)# aaa authorization commands 1 default group tacacs+
```

This enables command authorization for commands assigned to privilege level 1.

For example, a user may be allowed to execute `show version` but denied another level 1 command according to the TACACS+ server policy.

### Level 0 Authorization

Configure:

```text
R1(config)# aaa authorization commands 0 default group tacacs+
```

This applies authorization to commands assigned to privilege level 0.

By default, five commonly referenced level 0 commands are:

- `disable`
- `enable`
- `exit`
- `help`
- `logout`

Use caution when authorizing level 0 commands because denying commands such as `exit` or `logout` can interfere with normal session control.

### Privilege-Level Matching

**The level in `aaa authorization commands <level>` identifies the privilege level assigned to the command, not the user's current privilege level.**

For example, a privilege level 15 administrator can execute both level 1 and level 15 commands.

Consider:

```text
aaa authorization commands 15 default group tacacs+
```

This enables authorization for level 15 commands.

It does not automatically enable authorization for level 1 commands executed by the same administrator.

To authorize commands at both levels:

```text
R1(config)# aaa authorization commands 1 default group tacacs+
R1(config)# aaa authorization commands 15 default group tacacs+
```

Similarly, configuring level 15 authorization does not automatically authorize commands assigned to custom levels 2 through 14.

**Each privilege level that requires command authorization must have an applicable method list.**

### Changing Command Privilege Levels

IOS XE supports changing the privilege level assigned to a command.

For example:

```text
R1(config)# privilege exec level 5 show ip route
```

This assigns the specified command to privilege level 5.

If command authorization is configured for level 5, the command can be evaluated by that method list.

The privilege level assigned to a command affects which command authorization policy applies.

---

## Default and Named Command Authorization Lists

Command authorization supports default and named method lists.

### Default Method List

For example:

```text
R1(config)# aaa authorization commands 15 default group tacacs+
```

This defines the default authorization method list for commands at privilege level 15.

The default list applies automatically to applicable terminal lines unless overridden by a named list.

It is not normally necessary to configure:

```text
R1(config-line)# authorization commands 15 default
```

### Named Method Lists

For example:

```text
R1(config)# aaa authorization commands 15 SSH_COMMANDS group tacacs+ local
```

Apply the named method list to the relevant terminal lines:

```text
R1(config)# line vty 0 4
R1(config-line)# authorization commands 15 SSH_COMMANDS
```

The named list applies to level 15 commands executed through those VTY lines.

A different method list can be assigned to other lines.

### Different Policies for Different Lines

For example:

```text
R1(config)# aaa authorization commands 15 REMOTE_COMMANDS group tacacs+ local
R1(config)# aaa authorization commands 15 CONSOLE_COMMANDS local
!
R1(config)# line vty 0 4
R1(config-line)# authorization commands 15 REMOTE_COMMANDS
!
R1(config)# line console 0
R1(config-line)# authorization commands 15 CONSOLE_COMMANDS
```

The VTY and console configurations reference separate command authorization policies.

However, console AAA authorization requires an additional global command, discussed later.

---

## TACACS+ Command Authorization

TACACS+ is particularly suitable for command authorization because it supports separate authorization requests for individual commands.

An IOS XE device can send information such as:

```text
service=shell
cmd=show
cmd-arg=ip
cmd-arg=route
```

The server can evaluate both the command and its arguments.

For example:

```text
show ip route
```

and:

```text
show running-config
```

can receive different authorization decisions, even though both begin with `show`.

### Command and Argument Encoding

IOS XE sends the command name and its arguments as separate TACACS+ attribute-value pairs.

For example, the command:

```text
interface Ethernet0/1
```

can be represented as:

```text
service=shell
cmd=interface
cmd-arg=Ethernet
cmd-arg=0/1
cmd-arg=<cr>
```

The command name is identified by `cmd`.

The arguments are identified by `cmd-arg`.

The `<cr>` argument indicates the end of the command.

**A single CLI argument is not necessarily represented by a single `cmd-arg` attribute.**

For example, IOS XE can split an interface identifier into separate interface-type and interface-number arguments.

The TACACS+ server must account for the actual arguments sent by the device when matching its authorization policies.

### TACACS+ Server Policies

Consider a simplified TACACS+ server configuration using the commonly deployed `tac_plus` syntax:

```text
group = operators {
    default service = deny

    cmd = show {
        permit .*
    }

    cmd = ping {
        permit .*
    }

    cmd = configure {
        deny .*
    }
}
```

This policy permits `show` and `ping` commands but denies `configure` commands.

The `cmd` statements identify commands.

The `permit` and `deny` entries use regular expressions to match command arguments.

For example:

```text
cmd = show {
    permit ^ip route$
    deny .*
}
```

This policy permits `show ip route` while denying other `show` argument combinations.

The exact policy syntax and default matching behavior depend on the TACACS+ server implementation.

### Command Authorization Responses

The TACACS+ server can return these authorization statuses:

| Result | Meaning |
|---|---|
| PASS_ADD | Authorization succeeds |
| PASS_REPL | Authorization succeeds, with replacement attributes |
| FAIL | Authorization is denied |
| ERROR | Authorization could not be completed |

When TACACS+ returns `FAIL`, IOS XE denies the command.

When it returns `ERROR`, IOS XE can attempt the next authorization method.

**A command authorization request does not prompt the user for a new username and password.** The request uses the identity associated with the existing authenticated session.

---

## Command Authorization and Configuration Mode

Command authorization can apply to both EXEC commands and configuration-mode commands.

For example:

```text
R1# configure terminal
R1(config)# interface Ethernet0/0
R1(config-if)# shutdown
```

These are distinct command authorization operations.

A TACACS+ server could permit an administrator to enter configuration mode but deny the `shutdown` command.

### Enabling Configuration Command Authorization

The global command controlling authorization of configuration-mode commands is:

```text
aaa authorization config-commands
```

For example:

```text
R1(config)# aaa authorization commands 15 default group tacacs+ local
R1(config)# aaa authorization config-commands
```

The first command configures the authorization method list for privilege level 15 commands.

The second explicitly enables authorization of configuration-mode commands.

With configuration-command authorization enabled, IOS XE can send TACACS+ authorization requests for commands entered in global configuration mode and its submodes.

For example:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# switchport mode access
```

Each command can generate a separate authorization request.

The TACACS+ server can therefore permit `interface Ethernet0/0` while denying `switchport mode access`, or apply different policies to other configuration commands.

**Authorization of `configure terminal` does not automatically authorize the configuration commands entered afterward.**

### Default Configuration Command Authorization Behavior

Cisco documentation describes configuration-command authorization as being enabled by default after configuring `aaa authorization commands <level>`.

However, **testing on both IOS and IOS XE showed that configuration-command authorization was disabled until `aaa authorization config-commands` was explicitly configured**.

With only:

```text
aaa authorization commands 15 default group tacacs+
```

EXEC commands were subject to TACACS+ authorization, but configuration-mode commands were not.

After configuring:

```text
aaa authorization config-commands
```

Configuration-mode commands generated TACACS+ authorization requests containing their command names and arguments.

Therefore, **explicitly configure `aaa authorization config-commands` when configuration-mode command authorization is required** rather than relying on the documented default.

The exact default may vary by platform and software release.

### Disabling Configuration Command Authorization

Use:

```text
R1(config)# no aaa authorization config-commands
```

This disables AAA command authorization for configuration-mode commands.

For example:

```text
R1(config)# aaa authorization commands 15 default group tacacs+
R1(config)# no aaa authorization config-commands
```

With this configuration:

- Eligible level 15 EXEC commands remain subject to TACACS+ authorization.
- Commands entered in global configuration mode and its submodes are not subject to AAA command authorization.

For example, the EXEC command:

```text
R1# configure terminal
```

can still require TACACS+ authorization.

However, commands entered afterward, such as:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# shutdown
```

are not subject to AAA command authorization.

**`no aaa authorization config-commands` does not disable EXEC command authorization.**

### The `do` Command

IOS XE supports executing EXEC commands from configuration mode using the `do` keyword.

For example:

```text
R1(config)# do show ip route
```

The interaction between `do` and AAA authorization depends on configuration-command authorization.

Cisco documents `aaa authorization config-commands` as necessary to authorize EXEC commands invoked using `do` from configuration mode.

For example, with `no aaa authorization config-commands` enabled:

```
SW2(config)#do sh ip int br
Interface              IP-Address      OK? Method Status                Protocol
Ethernet0/0            unassigned      YES unset  up                    up      
Ethernet0/1            unassigned      YES unset  up                    up      
Ethernet0/2            unassigned      YES unset  up                    up      
Ethernet0/3            unassigned      YES unset  up                    up      
Vlan1                  192.168.99.2    YES manual up                    up      
SW2(config)#
*Oct 17 00:25:27.515: AAA/AUTHOR: auth_need : user= 'jeremy' ruser= 'SW2'rem_addr= '192.168.99.1' priv= 15 list= '' AUTHOR-TYPE= 'commands'
*Oct 17 00:25:27.515: AAA/AUTHOR: config command authorization not enabled
*Oct 17 00:25:27.515: AAA/AUTHOR: auth_need : user= 'jeremy' ruser= 'SW2'rem_addr= '192.168.99.1' priv= 1 list= '' AUTHOR-TYPE= 'commands'
```

Notice that, although `show ip interface brief` itself is an EXEC command, the `debug` output says that `config command authorization not enabled`.

---

## Local Command Authorization

The `local` method uses locally available authorization information instead of consulting an external AAA server.

For example:

```text
R1(config)# aaa authorization commands 15 default group tacacs+ local
```

If the TACACS+ group returns an error, IOS XE attempts local authorization.

However, **local authorization is not equivalent to TACACS+ command authorization**.

The local username database does not provide the same granular per-command allow/deny policy capabilities as a TACACS+ server.

Local command authorization is primarily useful as a fallback that relies on locally configured user privileges and supported authorization information.

For example:

```text
R1(config)# username admin privilege 15 secret Local123
```

The locally configured privilege level can be relevant when local authorization is used.

Exact local command authorization behavior should be verified on the IOS XE platform and release.

---

## The `if-authenticated` Method

The `if-authenticated` method permits authorization if the user has already authenticated successfully.

For example:

```text
R1(config)# aaa authorization commands 15 default group tacacs+ if-authenticated
```

The router attempts TACACS+ authorization first.

If the TACACS+ method returns an error, `if-authenticated` permits the command for an already authenticated user.

It does not query the local username database for command-specific authorization information.

### Example

Consider:

```text
aaa authentication login default group tacacs+ local
aaa authorization commands 15 default group tacacs+ if-authenticated
```

Suppose the TACACS+ server becomes unavailable.

1. Login authentication can fall back to the local database.
2. The administrator authenticates successfully.
3. The administrator attempts a level 15 command.
4. TACACS+ command authorization returns an error because the server is unavailable.
5. `if-authenticated` permits the command.

This can preserve administrative access during an AAA server outage.

However, it also means that central command restrictions are not enforced during that outage.

**`if-authenticated` permits authorization when the preceding method returns an error, not when the server explicitly denies the command.**

### Security Consideration

Consider a TACACS+ server that normally denies:

```text
reload
```

With:

```text
aaa authorization commands 15 default group tacacs+ if-authenticated
```

If the TACACS+ server is available and denies `reload`, the command is rejected.

If the server is unavailable, `if-authenticated` can permit `reload` for an already authenticated user with sufficient CLI privileges.

This is a significant consequence of the fallback policy.

---

## The `none` Method

The `none` method permits authorization without performing another authorization check.

For example:

```text
R1(config)# aaa authorization commands 15 default group tacacs+ none
```

If TACACS+ returns an error, `none` permits the command.

However, if TACACS+ explicitly denies the command, processing stops.

Compare:

| Method | Behavior |
|---|---|
| `group tacacs+` | Requests a decision from the TACACS+ server |
| `local` | Uses local authorization information |
| `if-authenticated` | Permits authorization if the user has authenticated |
| `none` | Permits authorization without another authorization check |

The `none` method does not grant a user a higher privilege level.

It simply permits the AAA authorization check.

Normal CLI privilege restrictions still apply.

---

## Command Authorization Failover

Consider:

```text
aaa authorization commands 15 default group tacacs+ local
```

The behavior depends on the TACACS+ result.

| TACACS+ Result | Behavior |
|---|---|
| PASS_ADD / PASS_REPL | Command permitted |
| FAIL | Command denied; local not attempted |
| ERROR | Local authorization attempted |

If multiple TACACS+ servers belong to the selected server group, IOS XE can attempt another server before the group returns an error.

The next AAA method is not necessarily attempted when the first server fails to respond.

### Example with Two Server Groups

```text
R1(config)# aaa authorization commands 15 default group TAC_PRIMARY group TAC_BACKUP if-authenticated
```

The processing order is:

1. Attempt command authorization using `TAC_PRIMARY`.
2. If that group returns an error, attempt `TAC_BACKUP`.
3. If both groups return errors, permit the command if the user has authenticated successfully.

An explicit deny from either server group stops processing.

---

## Console Command Authorization

AAA authorization is normally disabled for console sessions by default.

To enable AAA authorization for console sessions, configure:

```text
R1(config)# aaa authorization console
```

For example:

```text
R1(config)# aaa authorization commands 15 default group tacacs+ local
R1(config)# aaa authorization console
```

This enables the configured command authorization policy for applicable EXEC commands entered through the console.

To also authorize configuration-mode commands, configure:

```text
R1(config)# aaa authorization config-commands
```

These two commands serve different purposes:

| Command | Purpose |
|---|---|
| `aaa authorization console` | Enables AAA authorization for console sessions |
| `aaa authorization config-commands` | Enables AAA authorization for configuration-mode commands |

**Enabling console authorization does not, by itself, enable configuration-command authorization.**

A named command authorization method list can also be assigned:

```text
R1(config)# aaa authorization commands 15 CONSOLE_CMDS group tacacs+ local
R1(config)# aaa authorization console
R1(config)# aaa authorization config-commands
!
R1(config)# line console 0
R1(config-line)# authorization commands 15 CONSOLE_CMDS
```

Without `aaa authorization console`, the console normally bypasses AAA command authorization, even when a command authorization method list exists.

Console AAA authorization is separate from console login authentication.

Be careful when enabling command authorization on the console, because a server failure or restrictive authorization policy can interfere with recovery access.

---

## Command Authorization vs. EXEC Authorization

EXEC authorization and command authorization serve different purposes.

| Feature | EXEC Authorization | Command Authorization |
|---|---|---|
| Configuration | `aaa authorization exec` | `aaa authorization commands <level>` |
| Evaluated | When the EXEC session starts | When a matching command is executed |
| Controls | EXEC access and session attributes | Individual CLI commands |
| Can assign initial privilege level | Yes | No |
| Supports granular command/argument policies | No | Yes, with TACACS+ |

For example:

```text
R1(config)# aaa authorization exec default group tacacs+ local
R1(config)# aaa authorization commands 15 default group tacacs+ local
R1(config)# aaa authorization config-commands
```

The first command authorizes the EXEC session.

The second configures authorization for privilege level 15 commands.

The third explicitly enables configuration-mode command authorization.

A user can receive privilege level 15 through EXEC authorization but still be denied `reload` or `shutdown` through command authorization.

Likewise, enabling command authorization does not automatically assign privilege level 15.

---

## TACACS+ vs. RADIUS Command Authorization

TACACS+ is the preferred protocol for granular IOS XE command authorization.

| Feature | TACACS+ | RADIUS |
|---|---|---|
| Separate command authorization requests | Yes | Not part of traditional RADIUS operation |
| Per-command authorization | Yes | Not normally supported |
| Command argument matching | Yes | Not normally supported |
| Administrative privilege assignment | Yes | Yes, using authorization attributes |
| Typical administrative use | Granular CLI access control | Login and session authorization |

RADIUS can return attributes that assign administrative privileges or other session properties.

However, traditional RADIUS does not provide the same separate per-command authorization exchanges as TACACS+.

**For CCIE EI command authorization tasks, TACACS+ is the primary protocol to understand.**

---

## Command Authorization and Accounting

Command authorization and command accounting are separate AAA functions.

- **Command authorization:** Determines whether the command is permitted.
- **Command accounting:** Records command execution activity.

For example:

```text
R1(config)# aaa authorization commands 15 default group tacacs+ local
R1(config)# aaa accounting commands 15 default start-stop group tacacs+
```

The first command configures authorization for level 15 commands.

The second configures accounting for level 15 command activity.

Command accounting does not determine whether a command may execute.

Likewise, command authorization does not automatically enable command accounting.

Both must be configured independently.

Detailed accounting operation and configuration are covered in the AAA Accounting page.

---

## Complete TACACS+ Command Authorization Example

Consider a router using a TACACS+ server at `192.0.2.10`.

Configure the server:

```text
R1(config)# tacacs server TAC1
R1(config-server-tacacs)# address ipv4 192.0.2.10
R1(config-server-tacacs)# key Cisco123
```

Configure a local administrator:

```text
R1(config)# username admin privilege 15 secret Local123
```

Enable AAA and configure method lists:

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login default group tacacs+ local
R1(config)# aaa authorization exec default group tacacs+ local
R1(config)# aaa authorization commands 1 default group tacacs+ local
R1(config)# aaa authorization commands 15 default group tacacs+ local
R1(config)# aaa authorization config-commands
```

This configuration provides:

| AAA Function | Primary Method | Fallback |
|---|---|---|
| Login authentication | TACACS+ | Local |
| EXEC authorization | TACACS+ | Local |
| Level 1 command authorization | TACACS+ | Local |
| Level 15 command authorization | TACACS+ | Local |

Configuration-mode commands are explicitly included in command authorization.

When TACACS+ is available, it can independently authorize each applicable command.

If TACACS+ becomes unavailable, IOS XE can use the configured local authorization fallback.

The configuration of individual permitted and denied commands takes place on the TACACS+ server.

---

## IOS XE Verification

### Display Command Authorization Configuration

Use:

```text
R1# show running-config | include ^aaa authorization commands
```

Example output:

```text
aaa authorization commands 1 default group tacacs+ local
aaa authorization commands 15 default group tacacs+ local
```

This verifies that command authorization method lists are configured.

It does not establish whether configuration-mode commands are subject to authorization.

### Verify Configuration Command Authorization

Check the configuration:

```text
R1# show running-config | include aaa authorization config-commands
```

The explicit configuration:

```text
aaa authorization config-commands
```

enables configuration-mode command authorization.

The command:

```text
no aaa authorization config-commands
```

disables it.

However, because documentation and tested platform behavior differ regarding the default, **the most reliable verification is to observe whether a configuration-mode command generates an authorization request**.

Enable:

```text
R1# debug aaa authorization
R1# debug tacacs authorization
```

Then execute an applicable configuration-mode command:

```text
R1# configure terminal
R1(config)# interface Ethernet0/0
```

When configuration-command authorization is enabled, IOS XE can generate a TACACS+ request containing:

```text
service=shell
cmd=interface
cmd-arg=Ethernet
cmd-arg=0/0
cmd-arg=<cr>
```

The command is executed only if authorization succeeds.

When configuration-command authorization is disabled, no TACACS+ authorization request is generated for the configuration command.

Debug output may explicitly indicate that configuration-command authorization is not enabled.

This provides a direct method of distinguishing EXEC command authorization from configuration-mode command authorization.

### Verify Console Authorization

Use:

```text
R1# show running-config | include ^aaa authorization console
```

The presence of:

```text
aaa authorization console
```

indicates that AAA authorization has been enabled for console sessions.

### Display Line Authorization

For VTY lines:

```text
R1# show running-config | section ^line vty
```

For console lines:

```text
R1# show running-config | section ^line con
```

Verify any named command authorization lists assigned to the lines.

### Debug Command Authorization

Enable general AAA authorization debugging:

```text
R1# debug aaa authorization
```

Enable TACACS+ authorization debugging:

```text
R1# debug tacacs authorization
```

Then execute a command that should be subject to authorization.

For example:

```text
R1# show running-config
```

The debugging output can reveal the authorization request and server response.

Important information includes:

- Username associated with the session.
- Command privilege level.
- Command name.
- Command arguments.
- Selected authorization method list.
- Authorization method attempted.
- Final authorization status.

For example, the statuses may indicate:

| Status | Meaning |
|---|---|
| `PASS_ADD` | Authorization succeeded |
| `FAIL` | Authorization was denied |
| `ERROR` | Authorization failed to complete; another method may be attempted |

A command that executes successfully was not necessarily permitted by TACACS+.

For example, if TACACS+ returns `ERROR`, a subsequent `local` method may authorize the command.

After troubleshooting:

```text
R1# undebug all
```

> AAA debugging can generate substantial output and may reveal sensitive information about commands and user sessions.

### Verify TACACS+ Server Logs

TACACS+ server logs can provide useful information about command authorization requests.

For example:

```text
authorize_cmd: user=admin, cmd=show
```

Depending on the server's logging configuration, additional messages may indicate whether the command was permitted or denied.

These logs help distinguish an IOS XE configuration problem from a TACACS+ server policy problem.

---

## Common Command Authorization Problems

| Problem | Possible Cause |
|---|---|
| Command authorization never occurs | No `aaa authorization commands` method list configured |
| EXEC commands are authorized but configuration-mode commands are not | `aaa authorization config-commands` not enabled |
| Level 15 commands are authorized but level 1 commands are not | Only level 15 authorization is configured |
| Configuration-mode commands bypass authorization | Configuration-command authorization disabled |
| Command denied despite privilege level 15 | TACACS+ command authorization policy denies it |
| Named authorization list is not used | List not applied under the terminal line |
| Commands are not authorized on the console | `aaa authorization console` is not enabled |
| Command authorization unexpectedly falls back | TACACS+ server or server group returned an error |
| Command denied without fallback | TACACS+ returned an explicit authorization denial |
| User can execute commands during a TACACS+ outage | `if-authenticated`, `none`, or permissive local fallback |
| Server policy matches the command but not its arguments | Incorrect TACACS+ argument-matching policy |
| Different commands use different authorization policies | Commands belong to different privilege levels |

### Troubleshooting Sequence

When a command is unexpectedly permitted or denied:

1. Verify the privilege level assigned to the command.
2. Confirm that an authorization method list exists for that level.
3. Check whether the default or a named method list applies.
4. Verify whether configuration-command authorization is explicitly enabled.
5. Check console authorization if using the console.
6. Verify TACACS+ connectivity and server-group selection.
7. Check whether an authorization request was generated.
8. Identify the authorization method and final result.
9. Inspect the TACACS+ server's command and argument policies.
10. Verify fallback methods and their security implications.

---

## Key Points

- AAA command authorization controls whether individual CLI commands are permitted.
- AAA command authorization is disabled by default.
- Configure it with `aaa authorization commands <level> {default | <name>} <methods>`.
- Command authorization is configured separately for privilege levels **0 through 15**.
- **The configured level refers to the command's privilege level, not the user's current privilege level.**
- Level 15 command authorization does not automatically authorize commands assigned to level 1.
- The default method list applies automatically to applicable lines.
- Named method lists must be applied using `authorization commands <level> <name>`.
- TACACS+ supports authorization of individual commands and their arguments.
- TACACS+ command authorization uses separate authorization exchanges.
- IOS XE sends the command name using `cmd` and its arguments using `cmd-arg`.
- An explicit authorization denial stops method-list processing.
- An authorization error allows the next configured method to be attempted.
- `local` uses local authorization information but does not provide the same granular per-command policies as TACACS+.
- `if-authenticated` permits authorization for an already authenticated user when the preceding method returns an error.
- `none` permits authorization without another authorization check.
- **EXEC command authorization and configuration-mode command authorization are separate behaviors.**
- `aaa authorization config-commands` explicitly enables authorization of configuration-mode commands.
- Although Cisco documentation suggests configuration-command authorization is enabled automatically when `aaa authorization commands` is configured, testing on IOS and IOS XE showed it disabled until explicitly enabled.
- **Explicitly configure `aaa authorization config-commands` rather than relying on the default.**
- `no aaa authorization config-commands` disables configuration-mode command authorization without disabling EXEC command authorization.
- Console command authorization requires `aaa authorization console`.
- `aaa authorization console` does not independently enable configuration-mode command authorization.
- EXEC authorization and command authorization are separate AAA functions.
- Command authorization and command accounting are configured independently.
- **TACACS+ is preferred over RADIUS for granular CLI command authorization.**
- Use `debug aaa authorization` and `debug tacacs authorization` to verify individual command authorization requests and results.

---

## References

- [Cisco IOS XE - Configuring Authorization](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_aaa/configuration/xe-16-11/sec-usr-aaa-xe-16-11-book/sec-cfg-authorizatn.html)
- [Cisco IOS Security Command Reference - AAA Authorization Commands](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/security/a1/sec-a1-cr-book/sec-cr-a1.html)
- [Cisco IOS XE - Configuring Authorization](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-cfg-authorizatn-0.html)
- [RFC 8907: The TACACS+ Protocol](https://www.rfc-editor.org/rfc/rfc8907)
