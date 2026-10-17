# EXEC Authorization

**AAA EXEC authorization** determines whether an authenticated user is permitted to start an EXEC session on a Cisco IOS XE device.

It can also control attributes of the EXEC session, including:

- The user's privilege level.
- Whether an EXEC shell is permitted.
- An automatically executed command (`autocommand`).
- Other supported session attributes.

EXEC authorization is separate from login authentication.

**Login authentication verifies the user's identity. EXEC authorization determines whether an EXEC session is permitted and which attributes are assigned to it.**

---

## EXEC Authorization Operation

When a user connects to a device through SSH or another terminal service, IOS XE can perform two AAA operations:

1. **Login authentication:** Verify the user's credentials.
2. **EXEC authorization:** Determine whether the user may start an EXEC session and obtain its authorization attributes.

For example:

```text
Administrator                R1                TACACS+ Server
      |                      |                       |
      |----- SSH Login ----->|                       |
      |                      |                       |
      |                      |--- Authentication --->|
      |                      |<------- PASS ---------|
      |                      |                       |
      |                      |--- EXEC Authorization >|
      |                      |<------ PASS_ADD -------|
      |                      |     priv-lvl=15       |
      |                      |                       |
      |<------- R1# ---------|                       |
```

In this example, the TACACS+ server authenticates the administrator and then separately authorizes the EXEC session.

The authorization response assigns privilege level 15.

The administrator therefore enters privileged EXEC mode directly.

If EXEC authorization assigns privilege level 1 instead, the administrator enters user EXEC mode.

### Default Behavior

By default, AAA EXEC authorization is disabled.

Without an `aaa authorization exec` method list:

- A successfully authenticated user can normally start an EXEC session.
- IOS XE does not perform a separate AAA EXEC authorization check.
- The user's privilege level is determined by other applicable settings.
- Command access remains restricted by privilege levels and any separately configured command authorization.

For example:

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login default group tacacs+ local
```

This enables TACACS+ login authentication, but does not enable AAA EXEC authorization.

A successfully authenticated user can normally enter EXEC mode, typically at privilege level 1 unless another configuration assigns a higher level.

**Disabling EXEC authorization does not mean every command is permitted.** IOS XE still enforces command privilege levels.

---

## Configuring EXEC Authorization

Configure an EXEC authorization method list using:

```text
aaa authorization exec {default | <list-name>} <method1> [<method2> ...]
```

For example:

```text
R1(config)# aaa authorization exec default group tacacs+ local
```

This configures the following methods:

1. TACACS+ EXEC authorization.
2. Local EXEC authorization if TACACS+ returns an error.

The `default` method list applies automatically to applicable EXEC sessions unless a named authorization list overrides it.

### EXEC Authorization Methods

| Method | Behavior |
|---|---|
| `group tacacs+` | Uses the default TACACS+ server group |
| `group radius` | Uses the default RADIUS server group |
| `group <name>` | Uses a named AAA server group |
| `local` | Uses the local username database for authorization |
| `if-authenticated` | Permits authorization if the user has already authenticated |
| `none` | Permits authorization without performing a further authorization check |

Methods are evaluated in their configured order.

**The next method is attempted only when the current method returns an error. An explicit authorization rejection stops method-list processing.**

---

## Default and Named EXEC Authorization Lists

As with login authentication, EXEC authorization supports both default and named method lists.

### Default Method List

For example:

```text
R1(config)# aaa authorization exec default group tacacs+ local
```

This defines the default EXEC authorization policy.

It applies automatically to applicable EXEC sessions without requiring:

```text
R1(config-line)# authorization exec default
```

The command can be entered explicitly, but is not normally necessary when using the default list.

### Named Method Lists

Named authorization lists allow different terminal lines to use different authorization policies.

For example:

```text
R1(config)# aaa authorization exec SSH_AUTH group tacacs+ local
R1(config)# aaa authorization exec CONSOLE_AUTH local
```

Apply a named list under the appropriate line:

```text
R1(config)# line vty 0 4
R1(config-line)# authorization exec SSH_AUTH
```

For the console:

```text
R1(config)# line console 0
R1(config-line)# authorization exec CONSOLE_AUTH
```

**Named EXEC authorization method lists must be explicitly assigned using `authorization exec <list-name>`.**

The console also has additional authorization behavior, discussed later.

---

## TACACS+ EXEC Authorization

TACACS+ performs authentication and authorization using separate protocol exchanges.

When EXEC authorization is enabled, IOS XE sends an authorization request to the TACACS+ server after successful login authentication.

For example:

```text
R1(config)# aaa authentication login default group tacacs+ local
R1(config)# aaa authorization exec default group tacacs+ local
```

The first command controls login authentication.

The second controls EXEC authorization.

### TACACS+ EXEC Authorization Attributes

A TACACS+ server can authorize an EXEC shell using attributes such as:

```text
service=shell
priv-lvl=15
```

The attributes identify the authorized service and the requested privilege level.

For example, the following TACACS+ server configuration assigns privilege level 15:

```text
group = netadmin {
    default service = permit
    service = exec {
        priv-lvl = 15
    }
}
```

A user belonging to this group can receive privilege level 15 during EXEC authorization.

### TACACS+ Authorization Responses

The primary TACACS+ authorization statuses are:

| Status | Meaning |
|---|---|
| PASS_ADD | Authorization succeeds; returned attributes are added to the session |
| PASS_REPL | Authorization succeeds; returned attributes replace the supplied attributes |
| FAIL | Authorization is explicitly denied |
| ERROR | Authorization cannot be completed |

If TACACS+ returns `FAIL`, IOS XE denies the EXEC authorization request.

If TACACS+ returns `ERROR`, IOS XE can attempt the next authorization method.

For example:

```text
aaa authorization exec default group tacacs+ local
```

An explicit TACACS+ authorization denial does **not** cause fallback to `local`.

---

## RADIUS EXEC Authorization

RADIUS normally supplies authorization attributes in the authentication **Access-Accept** response.

Unlike TACACS+, it does not require a separate authorization protocol exchange in the typical login workflow.

However, IOS XE still uses a separate AAA EXEC authorization method list to enable EXEC authorization processing.

For example:

```text
R1(config)# aaa authentication login default group radius local
R1(config)# aaa authorization exec default group radius local
```

The RADIUS server can return authorization attributes during login authentication, and IOS XE can use those attributes to determine the EXEC session's properties.

### Cisco RADIUS Privilege Attribute

Cisco supports privilege assignment through a Vendor-Specific Attribute (VSA).

For example:

```text
Cisco-AVPair = "shell:priv-lvl=15"
```

This specifies privilege level 15 for the shell session.

When the attribute is accepted and applied during EXEC authorization, the administrator enters privileged EXEC mode.

### RADIUS Authorization and Local Login

Consider:

```text
aaa authentication login default group radius local
aaa authorization exec default group radius local
```

If RADIUS login authentication succeeds, the authorization process can use the attributes returned by RADIUS.

If RADIUS is unavailable and login authentication falls back to the local database, RADIUS authentication attributes are not available for that session.

The `local` EXEC authorization method allows IOS XE to use the local username database instead.

This is particularly important when local users have explicitly configured privilege levels.

---

## Local EXEC Authorization

The `local` method authorizes the EXEC session using information in the local username database.

For example:

```text
R1(config)# username admin privilege 15 secret Local123
R1(config)# aaa authorization exec default local
```

The configured username includes:

```text
privilege 15
```

When local EXEC authorization succeeds for this user, IOS XE can assign privilege level 15.

### Local Authentication and Authorization

Consider:

```text
R1(config)# username admin privilege 15 secret Local123
R1(config)# aaa authentication login default local
R1(config)# aaa authorization exec default local
```

The operations are separate:

| Operation | Method | Result |
|---|---|---|
| Login authentication | `local` | Validates the username and password |
| EXEC authorization | `local` | Authorizes the EXEC session using the local user's attributes |

The administrator can enter privileged EXEC mode immediately after login.

### Local Authorization as Fallback

For example:

```text
R1(config)# username backup privilege 15 secret Backup123
R1(config)# aaa authentication login default group tacacs+ local
R1(config)# aaa authorization exec default group tacacs+ local
```

When the TACACS+ servers are unavailable:

1. Login authentication falls back to the local database.
2. The `backup` user authenticates successfully.
3. EXEC authorization attempts TACACS+.
4. If TACACS+ authorization is unavailable, the `local` method is attempted.
5. The local user's privilege level can be applied.

**A local authentication fallback should be accompanied by an appropriate EXEC authorization fallback when EXEC authorization is enabled.**

Otherwise, a user may authenticate successfully but fail to obtain an EXEC session.

---

## The `if-authenticated` Method

The `if-authenticated` authorization method permits the requested service if the user has already been successfully authenticated.

For example:

```text
R1(config)# aaa authorization exec default group tacacs+ if-authenticated
```

The router first attempts TACACS+ EXEC authorization.

If TACACS+ authorization returns an error, IOS XE attempts `if-authenticated`.

Because the user has already authenticated, the method permits the EXEC session.

### What `if-authenticated` Does Not Do

The `if-authenticated` method does not retrieve authorization attributes from the local username database or from an unavailable AAA server.

For example, it does not automatically assign:

```text
priv-lvl=15
```

Instead, the session receives its privilege level from other applicable settings.

This typically means privilege level 1 unless a different privilege level is established by the line configuration or other applicable mechanisms.

### Example

```text
R1(config)# aaa authentication login default group tacacs+ local
R1(config)# aaa authorization exec default group tacacs+ if-authenticated
```

Suppose the TACACS+ server is unavailable.

Login authentication falls back to the local database.

After successful local authentication, EXEC authorization also attempts TACACS+.

Because TACACS+ is unavailable, `if-authenticated` permits the EXEC session.

However, the method does not itself retrieve the local user's configured privilege level.

Compare:

| Fallback Method | Behavior |
|---|---|
| `local` | Uses the local username database for authorization attributes |
| `if-authenticated` | Permits the EXEC session based on successful authentication, without retrieving attributes |

**Use `local` when local authorization attributes, such as a locally configured privilege level, must be applied.**

### Authentication Method Independence

`if-authenticated` does not require the user to have authenticated using the same AAA protocol used by the authorization method list.

For example:

```text
aaa authentication login default group radius local
aaa authorization exec default group tacacs+ if-authenticated
```

A user who successfully authenticated through RADIUS can be permitted by `if-authenticated` if TACACS+ authorization is unavailable.

The method checks whether the user has authenticated successfully, not which authentication method was used.

### Authorization Rejection

`if-authenticated` is not a bypass for an explicit authorization denial.

For example:

```text
aaa authorization exec default group tacacs+ if-authenticated
```

If TACACS+ explicitly denies EXEC authorization, the router does not proceed to `if-authenticated`.

The EXEC session is denied.

---

## The `none` Method

The `none` method permits authorization without requesting approval from an external server or consulting the local username database.

For example:

```text
R1(config)# aaa authorization exec default none
```

This allows EXEC authorization to succeed without another authorization check.

It does not perform authentication or assign privilege level 15.

### Using `none` as Fallback

For example:

```text
R1(config)# aaa authorization exec default group tacacs+ none
```

The router first attempts TACACS+ authorization.

If TACACS+ returns an error, `none` permits the EXEC session.

However, an explicit TACACS+ authorization denial still stops processing.

### `none` vs. `if-authenticated`

| Method | Behavior |
|---|---|
| `none` | Authorizes without performing an additional authorization check |
| `if-authenticated` | Authorizes only if the user has already authenticated successfully |
| `local` | Uses the local username database and its authorization attributes |

`if-authenticated` provides a clearer fallback policy when access should depend on successful prior authentication.

Neither `none` nor `if-authenticated` independently retrieves a privilege level from the AAA server.

---

## Authorization Failure vs. Authentication Failure

Authentication and EXEC authorization can fail independently.

For example:

```text
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local
```

Consider these situations:

| Login Authentication | EXEC Authorization | Result |
|---|---|---|
| Success | Success | EXEC session permitted |
| Failure | Not reached | Login denied |
| Success | Explicitly denied | EXEC session denied |
| Success | Server error, local authorization succeeds | EXEC session permitted |
| Success | All authorization methods fail | EXEC session denied |

The important distinction is:

**Successful authentication does not override an explicit EXEC authorization denial.**

If authentication succeeds but EXEC authorization fails, the user may be disconnected or prevented from entering an EXEC shell.

---

## Privilege Levels and EXEC Authorization

EXEC authorization can assign the privilege level at which a user's session begins.

Cisco IOS XE supports privilege levels 0 through 15.

The two most important levels are:

| Level | Typical Purpose |
|---|---|
| 1 | User EXEC |
| 15 | Privileged EXEC |

For example:

```text
R1>
```

indicates user EXEC mode at privilege level 1.

```text
R1#
```

indicates privileged EXEC mode, typically privilege level 15.

### TACACS+ Privilege Assignment

A TACACS+ authorization response might contain:

```text
priv-lvl=15
```

### RADIUS Privilege Assignment

A RADIUS Access-Accept might contain:

```text
Cisco-AVPair = "shell:priv-lvl=15"
```

### Local Privilege Assignment

A locally configured user might have:

```text
username admin privilege 15 secret Local123
```

In each case, the resulting session privilege level depends on the applicable EXEC authorization behavior.

**EXEC authorization can assign the initial privilege level, but it does not replace the normal CLI privilege system or command authorization.**

A user assigned privilege level 5 does not automatically gain access to privilege level 15 commands.

---

## Line Privilege Levels

A terminal line can also have a configured privilege level.

For example:

```text
R1(config)# line vty 0 4
R1(config-line)# privilege level 15
```

This configures the line's default EXEC privilege level.

The line privilege level is separate from user-specific attributes returned through AAA EXEC authorization.

A server-provided authorization privilege can override the line's default privilege level for the session.

When troubleshooting unexpected privileges, check both:

```text
username <name> privilege <level>
```

and:

```text
line vty 0 4
 privilege level <level>
```

Also verify whether EXEC authorization is enabled and which authorization method succeeded.

---

## Console EXEC Authorization

Console EXEC authorization requires special consideration.

On Cisco IOS XE, **AAA authorization for console access is normally disabled by default**, even when an EXEC authorization method list is configured.

To enable AAA authorization for console sessions, use:

```text
R1(config)# aaa authorization console
```

For example:

```text
R1(config)# aaa authorization exec default group tacacs+ local
R1(config)# aaa authorization console
```

This allows the configured EXEC authorization policy to apply to console sessions.

Without `aaa authorization console`, the console is generally exempt from AAA authorization checks.

### Console Authentication vs. Authorization

Console authentication and console authorization are separate.

For example:

```text
R1(config)# aaa authentication login CONSOLE_LOGIN local
R1(config)# aaa authorization exec CONSOLE_EXEC local
R1(config)# aaa authorization console
!
R1(config)# line console 0
R1(config-line)# login authentication CONSOLE_LOGIN
R1(config-line)# authorization exec CONSOLE_EXEC
```

This configuration uses:

- Local login authentication.
- Local EXEC authorization.
- Explicitly enabled console AAA authorization.

The local username database can provide both credentials and privilege-level attributes.

### Console Lockout Risk

Enabling console authorization without an appropriate fallback can make recovery more difficult during an AAA outage.

For example, a console user may authenticate successfully but fail EXEC authorization if the configured server group is unavailable and no usable fallback is configured.

**Configure and verify console authorization carefully before relying on centralized AAA for recovery access.**

---

## EXEC Authorization and Command Authorization

EXEC authorization and command authorization are different AAA functions.

| Feature | EXEC Authorization | Command Authorization |
|---|---|---|
| Configuration | `aaa authorization exec` | `aaa authorization commands` |
| Timing | When the EXEC session is established | When a user executes a matching command |
| Primary purpose | Permit the EXEC shell and assign session attributes | Permit or deny individual CLI commands |
| TACACS+ | Supported | Supported |
| RADIUS | Supported | Not normally used for granular CLI command authorization |

For example:

```text
R1(config)# aaa authorization exec default group tacacs+ local
R1(config)# aaa authorization commands 15 default group tacacs+ local
```

The first command enables EXEC authorization.

The second enables authorization for commands at privilege level 15.

A user can be authorized to enter privileged EXEC mode but still be denied a particular command by command authorization.

Likewise, enabling EXEC authorization does not automatically enable command authorization.

Detailed command authorization configuration is covered in the Command Authorization page.

---

## IOS XE Verification

### Display EXEC Authorization Method Lists

Use:

```text
R1# show running-config | include ^aaa authorization exec
```

Example:

```text
aaa authorization exec default group tacacs+ local
aaa authorization exec CONSOLE_EXEC local
```

### Display Line Authorization

For VTY lines:

```text
R1# show running-config | section ^line vty
```

For the console:

```text
R1# show running-config | section ^line con
```

Verify whether named EXEC authorization method lists are applied.

### Verify Console Authorization

Use:

```text
R1# show running-config | include ^aaa authorization console
```

If console AAA authorization is enabled, the configuration contains:

```text
aaa authorization console
```

### Check the Current Privilege Level

After login:

```text
R1# show privilege
```

Example output:

```text
Current privilege level is 15
```

This verifies the privilege level assigned to the current session.

It does not identify which AAA method provided that privilege level.

### Debug AAA Authorization

Enable:

```text
R1# debug aaa authorization
```

For TACACS+ authorization:

```text
R1# debug tacacs authorization
```

General authentication debugging can help correlate the two stages:

```text
R1# debug aaa authentication
```

For RADIUS attribute processing, supported platforms provide additional RADIUS debugging:

```text
R1# debug radius
```

Disable debugging after testing:

```text
R1# undebug all
```

> AAA debugging can expose sensitive user, session, and authorization information.

---

## Common EXEC Authorization Problems

| Problem | Possible Cause |
|---|---|
| Login succeeds but EXEC session fails | EXEC authorization denied or all methods failed |
| TACACS+ login succeeds but user receives privilege level 1 | EXEC authorization is disabled or privilege attributes are missing |
| Local fallback login succeeds but EXEC authorization fails | No suitable authorization fallback |
| Local fallback login succeeds with the wrong privilege level | `if-authenticated` used instead of `local`, or another privilege setting applies |
| A named authorization list is ignored | It was not applied to the terminal line |
| Console does not use AAA EXEC authorization | `aaa authorization console` is not configured |
| Unexpected privilege level | Conflicting local, line, or server-provided privilege settings |
| `if-authenticated` does not permit access | An earlier method explicitly denied authorization |
| RADIUS login succeeds but privilege level is wrong | Missing or incorrectly interpreted RADIUS authorization attributes |
| Commands are denied despite privilege level 15 | Separate AAA command authorization policy |

### Troubleshooting Sequence

When login succeeds but the resulting session is incorrect:

1. Verify that login authentication succeeded.
2. Identify the EXEC authorization method list.
3. Confirm whether the default or a named list applies.
4. Check whether the server returned an explicit authorization denial or an error.
5. Verify any privilege-level attributes returned by the server.
6. Check local username privileges and line privilege configuration.
7. Verify console-specific authorization behavior if troubleshooting console access.
8. Use `show privilege` to confirm the final privilege level.

---

## Key Points

- EXEC authorization controls whether an authenticated user may start an EXEC session.
- It can also assign EXEC session attributes, including privilege levels.
- **AAA EXEC authorization is disabled by default.**
- Without EXEC authorization, successfully authenticated users can normally enter EXEC mode, but normal privilege restrictions still apply.
- Configure EXEC authorization using `aaa authorization exec {default | <name>} <methods>`.
- The default EXEC authorization method list applies automatically to applicable sessions.
- Named EXEC authorization method lists must be explicitly applied using `authorization exec <name>`.
- TACACS+ uses a separate EXEC authorization exchange after login authentication.
- TACACS+ can return `service=shell` and `priv-lvl=15`.
- RADIUS can provide authorization attributes in an Access-Accept.
- RADIUS can assign privilege levels using the Cisco VSA `shell:priv-lvl=15`.
- The `local` method uses locally configured username attributes for EXEC authorization.
- `if-authenticated` permits EXEC authorization if the user has already authenticated successfully.
- `if-authenticated` does **not** independently retrieve authorization attributes or assign privilege level 15.
- `none` permits authorization without performing another authorization check.
- An explicit authorization denial does not trigger fallback to the next authorization method.
- A server error can allow the next configured authorization method to be attempted.
- Successfully authenticating through one AAA protocol does not prevent `if-authenticated` from permitting authorization after another method returns an error.
- **Console AAA authorization is normally disabled unless `aaa authorization console` is configured.**
- EXEC authorization and command authorization are separate functions.
- Use `show privilege` to verify the current session privilege level.
- Use `debug aaa authorization` and `debug tacacs authorization` for troubleshooting.

---

## References

- [Cisco IOS XE - Configuring Authorization](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_aaa/configuration/xe-16-11/sec-usr-aaa-xe-16-11-book/sec-cfg-authorizatn.html)
- [Cisco IOS XE - Configuring Authentication](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_aaa/configuration/xe-16-12/sec-usr-aaa-xe-16-12-book/sec-cfg-authentifcn.html)
- [Cisco IOS XE - RADIUS Configuration Guide](https://www.cisco.com/c/en/us/td/docs/routers/asr920/configuration/guide/sec-usr-rad/17-1-1/b-sec-usr-rad-17-1-asr920.html)
- [RFC 8907: The TACACS+ Protocol](https://www.rfc-editor.org/rfc/rfc8907)
- [RFC 2865: RADIUS](https://www.rfc-editor.org/rfc/rfc2865)
