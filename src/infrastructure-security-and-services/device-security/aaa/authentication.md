
# Login and Enable Authentication

Cisco IOS XE supports two separate types of AAA authentication for administrative CLI access:

- **Login authentication:** Verifies a user's identity when accessing the device through SSH, Telnet, the console, or another supported terminal line.
- **Enable authentication:** Verifies a user's credentials when attempting to enter a higher privilege level using the `enable` command.

Both use AAA authentication method lists, but their configuration and supported methods differ.

**Successful login authentication does not necessarily grant privileged EXEC access.**

---

## Login vs. Enable Authentication

Login and enable authentication occur at different stages of administrative access.

For example:

```text
Administrator                    R1
      |                          |
      |------ SSH Connection --->|
      |                          |
      |<----- Username ----------|
      |<----- Password ----------|
      |                          |
      |    Login Authentication  |
      |                          |
      |<------ R1> --------------|
      |                          |
      |------ enable ----------->|
      |                          |
      |<----- Password ----------|
      |                          |
      |    Enable Authentication |
      |                          |
      |<------ R1# --------------|
```

In this example:

1. The administrator authenticates to establish an EXEC session.
2. The administrator enters user EXEC mode (privilege level 1).
3. The administrator enters `enable`.
4. IOS XE performs enable authentication.
5. If authentication succeeds, the administrator enters privileged EXEC mode (privilege level 15).

However, a user assigned privilege level 15 during EXEC authorization can enter privileged EXEC mode immediately after login, without entering `enable`.

| Feature | Login Authentication | Enable Authentication |
|---|---|---|
| Purpose | Authenticate a user accessing the device | Authenticate a user requesting higher privileges |
| Configuration | `aaa authentication login` | `aaa authentication enable` |
| Default and named lists | Supported | Only the `default` list |
| Applied to lines | Yes | No separate line application |
| Local username database | Supported | Not supported as a direct method |
| Local enable secret | Supported using `enable` | Supported using `enable` |
| TACACS+ / RADIUS | Supported | Supported |

---

## Login Authentication Method Lists

Configure login authentication using:

```text
aaa authentication login {default | <list-name>} <method1> [<method2> ...]
```

The command defines an ordered list of authentication methods.

For example:

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login default group tacacs+ local
```

This configures:

1. TACACS+ authentication.
2. Local authentication if the TACACS+ method returns an error.

An authentication method list can use different authentication sources.

### Login Authentication Methods

| Method | Behavior |
|---|---|
| `group tacacs+` | Authenticates through the default TACACS+ server group |
| `group radius` | Authenticates through the default RADIUS server group |
| `group <name>` | Authenticates through a named AAA server group |
| `local` | Authenticates against the local username database |
| `local-case` | Authenticates against the local username database with case-sensitive usernames |
| `enable` | Authenticates using the locally configured enable password or secret |
| `line` | Authenticates using the password configured on the terminal line |
| `none` | Allows authentication without checking credentials |

Other authentication methods exist on certain IOS XE platforms, but these are the primary methods relevant to administrative CLI authentication.

**The `local` method uses local usernames and passwords, whereas the `enable` method uses the enable password or secret.**

---

## Default Login Authentication

The `default` keyword creates the default login authentication method list.

For example:

```text
R1(config)# aaa authentication login default group tacacs+ local
```

The default list is automatically used by applicable terminal lines that do not explicitly reference another authentication method list.

It is not necessary to configure `login authentication default` on every line when the default list already applies.

However, it can be configured explicitly:

```text
R1(config)# line vty 0 4
R1(config-line)# login authentication default
```

This applies the default authentication method list to VTY lines 0 through 4.

The `default` keyword is a reserved method-list name, not an arbitrary identifier.

### Default List Example

```text
R1(config)# username admin privilege 15 secret Local123
R1(config)# aaa new-model
R1(config)# aaa authentication login default group tacacs+ local
!
R1(config)# line vty 0 4
R1(config-line)# login authentication default
R1(config-line)# transport input ssh
```

With this configuration:

- TACACS+ is the primary authentication method.
- Local authentication provides a fallback if TACACS+ is unavailable.
- An explicit TACACS+ authentication rejection prevents local fallback.

Note that the local user's configured privilege level is separate from login authentication. EXEC authorization determines how session privileges are assigned when AAA EXEC authorization is enabled.

---

## Named Login Authentication Method Lists

Named method lists allow different terminal lines to use different authentication methods.

For example:

```text
R1(config)# aaa authentication login SSH_LOGIN group tacacs+ local
R1(config)# aaa authentication login CONSOLE_LOGIN local
```

Apply the appropriate method list to each line.

### SSH Authentication

```text
R1(config)# line vty 0 4
R1(config-line)# login authentication SSH_LOGIN
R1(config-line)# transport input ssh
```

### Console Authentication

```text
R1(config)# line console 0
R1(config-line)# login authentication CONSOLE_LOGIN
```

The resulting authentication behavior is:

| Access Method | Method List | Authentication |
|---|---|---|
| SSH through VTY 0-4 | `SSH_LOGIN` | TACACS+, then local on error |
| Console | `CONSOLE_LOGIN` | Local only |

This is useful when remote administrators must authenticate through a centralized AAA server, but the physical console must remain available during an AAA server outage.

**Named login authentication method lists do not become active until they are applied to the appropriate terminal lines.**

A named list overrides the default list on the lines where it is applied.

---

## Login Authentication Using the Local Database

The `local` method authenticates users against the IOS XE local username database.

For example:

```text
R1(config)# username admin privilege 15 secret Local123
R1(config)# aaa authentication login default local
```

When a user attempts to log in:

1. IOS XE prompts for a username and password.
2. The username is checked against the local database.
3. The supplied password is checked against the stored secret.
4. Authentication succeeds or fails.

### Local vs. Local-Case

Two related methods are available:

```text
aaa authentication login default local
```

```text
aaa authentication login default local-case
```

| Method | Username Comparison | Password Comparison |
|---|---|---|
| `local` | Case-insensitive | Case-sensitive |
| `local-case` | Case-sensitive | Case-sensitive |

For example, if the local username is `Admin`, the `local` method can accept `admin`, whereas `local-case` requires the username to match the configured capitalization.

**Passwords remain case-sensitive with both methods.**

---

## Login Authentication Using a Line Password

The `line` method authenticates using the password configured on the terminal line.

For example:

```text
R1(config)# aaa authentication login LINE_LOGIN line
!
R1(config)# line vty 0 4
R1(config-line)# password Line123
R1(config-line)# login authentication LINE_LOGIN
```

The line password is shared by users accessing that line.

Unlike `local`, this method does not authenticate against an individual username account.

Line passwords are generally less suitable for administrative access because they do not provide individual user identities.

---

## Login Authentication Using the Enable Secret

The `enable` authentication method uses the locally configured enable password or secret to authenticate a login attempt.

For example:

```text
R1(config)# enable secret Enable123
R1(config)# aaa authentication login CONSOLE_LOGIN enable
!
R1(config)# line console 0
R1(config-line)# login authentication CONSOLE_LOGIN
```

A user accessing the console can authenticate using the enable secret.

**Using `enable` as a login authentication method does not mean the user is automatically assigned privilege level 15.**

Authentication establishes whether the user is permitted to log in. Session privileges are a separate matter.

---

## Authentication Method Processing

IOS XE processes authentication methods in the order configured.

However, **the next authentication method is attempted only when the previous method returns an error, not when it explicitly rejects the user.**

For example:

```text
R1(config)# aaa authentication login default group tacacs+ local
```

The behavior is:

| TACACS+ Result | Behavior |
|---|---|
| PASS | Login authentication succeeds |
| FAIL | Login authentication fails; local is not attempted |
| ERROR | IOS XE attempts local authentication |

A TACACS+ method can return an error when all eligible servers are unavailable or unable to complete the authentication request.

Likewise, with RADIUS:

| RADIUS Result | Behavior |
|---|---|
| Access-Accept | Login authentication succeeds |
| Access-Reject | Login authentication fails; local is not attempted |
| No usable response after server attempts are exhausted | IOS XE attempts local authentication |

If a method references multiple AAA servers, IOS XE can attempt other eligible servers before the group returns an error.

The next method in the method list is not necessarily attempted after the first server times out.

### The `none` Method

The `none` method allows authentication to succeed without checking credentials.

For example:

```text
R1(config)# aaa authentication login default group tacacs+ none
```

This configuration attempts TACACS+ authentication first.

If the TACACS+ method returns an error, authentication succeeds through `none`.

However, if TACACS+ explicitly rejects the user, authentication fails.

**Using `none` as a fallback can permit administrative access without credential verification when the primary authentication method is unavailable.**

Avoid this configuration for normal production management access.

---

## Enable Authentication

Enable authentication controls access to a higher privilege level when a user executes the `enable` command.

For example:

```text
R1> enable
Password:
R1#
```

If the user is already at privilege level 15, enable authentication is unnecessary.

To configure AAA enable authentication, use:

```text
aaa authentication enable default <method1> [<method2> ...]
```

Unlike login authentication, **AAA enable authentication supports only the `default` method list**.

There is no named enable authentication method list applied under individual console or VTY lines.

### Enable Authentication Methods

| Method | Behavior |
|---|---|
| `group tacacs+` | Uses the default TACACS+ server group |
| `group radius` | Uses the default RADIUS server group |
| `group <name>` | Uses a named TACACS+ or RADIUS server group |
| `enable` | Uses the locally configured enable password or secret |
| `line` | Uses the terminal line password |
| `none` | Allows enable authentication without checking credentials |

**The `local` and `local-case` methods are not supported for `aaa authentication enable`.**

To provide a local fallback for enable authentication, use the `enable` method.

---

## Default Enable Authentication Behavior

Without an explicitly configured AAA enable authentication method list, IOS XE normally uses the locally configured enable password or secret.

For example:

```text
R1(config)# enable secret Enable123
```

A user at privilege level 1 can execute:

```text
R1> enable
Password:
```

The user must provide the enable secret to enter privilege level 15.

When AAA enable authentication is explicitly configured, the method list determines how that authentication request is processed.

For example:

```text
R1(config)# aaa authentication enable default group tacacs+ enable
```

The router first uses TACACS+ and falls back to the local enable secret if the TACACS+ method returns an error.

---

## TACACS+ Enable Authentication

TACACS+ can authenticate enable requests through a centralized AAA server.

For example:

```text
R1(config)# enable secret Enable123
R1(config)# aaa authentication enable default group tacacs+ enable
```

When a user executes `enable`:

1. IOS XE sends an enable authentication request to the TACACS+ server group.
2. The TACACS+ server accepts or rejects the request.
3. If the group returns an error, IOS XE attempts authentication using the local enable secret.

The results are:

| TACACS+ Result | Enable Authentication |
|---|---|
| PASS | Succeeds |
| FAIL | Fails; local enable secret is not attempted |
| ERROR | Local enable secret is attempted |

### TACACS+ Enable Username

For TACACS+ enable authentication, IOS XE can include the username associated with the existing login session.

This allows the TACACS+ server to apply user-specific enable authentication policies.

For example, a TACACS+ server can permit enable authentication for one administrator but reject it for another.

The server can therefore make enable authentication decisions based on the administrator's identity.

---

## RADIUS Enable Authentication

RADIUS can also authenticate enable requests.

For example:

```text
R1(config)# enable secret Enable123
R1(config)# aaa authentication enable default group radius enable
```

The router attempts RADIUS enable authentication first.

If the RADIUS group returns an error, the router uses the locally configured enable secret.

### RADIUS Enable Username

Unlike TACACS+, traditional IOS XE RADIUS enable authentication does not authenticate enable requests using each administrator's login username.

For privilege level 15, IOS XE uses the special username:

```text
$enab15$
```

The RADIUS server must recognize and authenticate this identity according to its configured policy.

For example, a RADIUS server may define a user called `$enab15$` with the appropriate password.

The distinction is:

| Protocol | Enable Authentication Identity |
|---|---|
| TACACS+ | Username associated with the login session |
| RADIUS | Special enable username, such as `$enab15$` for privilege level 15 |

This is one reason TACACS+ is generally better suited to per-user network device administration.

---

## Local Enable Authentication Fallback

The `enable` method is commonly used as a fallback when centralized enable authentication is unavailable.

For example:

```text
R1(config)# enable secret Enable123
R1(config)# aaa authentication enable default group TAC_ADMIN enable
```

The method list contains:

1. The named AAA server group `TAC_ADMIN`.
2. The local enable secret.

If the TACACS+ server explicitly rejects the request, the enable attempt fails.

If the TACACS+ method returns an error, the user can authenticate using the local enable secret.

**The `enable` fallback does not check the local username database.**

For example, this configuration:

```text
R1(config)# username admin secret Local123
R1(config)# enable secret Enable123
R1(config)# aaa authentication enable default group tacacs+ enable
```

uses `Enable123`, not `Local123`, when the `enable` fallback is reached.

### Missing Enable Secret

If the `enable` method is configured but no usable enable secret or password exists, behavior depends on the platform and access context.

Do not rely on an absent enable secret as an authentication control or fallback mechanism.

For predictable recovery behavior, explicitly configure a strong enable secret when using the `enable` fallback method.

---

## Login Authentication vs. EXEC Authorization

Login authentication and EXEC authorization are separate AAA functions.

Consider:

```text
R1(config)# aaa authentication login default group tacacs+ local
```

This authenticates users through TACACS+, with local fallback.

It does not automatically enable AAA EXEC authorization.

Configure EXEC authorization separately:

```text
R1(config)# aaa authorization exec default group tacacs+ local
```

The difference is:

| AAA Function | Purpose |
|---|---|
| Login authentication | Verifies the user's identity |
| EXEC authorization | Determines whether the user may establish an EXEC session and can assign session attributes |
| Enable authentication | Verifies credentials when requesting a higher privilege level |

For example, a TACACS+ server might authenticate an administrator successfully, but the administrator may still enter user EXEC mode if privilege level 15 is not assigned through an applicable mechanism.

Alternatively, EXEC authorization can assign privilege level 15 immediately after login, eliminating the need to execute `enable`.

Detailed EXEC authorization behavior is covered in the EXEC Authorization page.

---

## Console Authentication Considerations

Console access is commonly used for recovery when remote AAA servers are unavailable.

A dedicated local console authentication list is often useful.

For example:

```text
R1(config)# username admin privilege 15 secret Local123
R1(config)# aaa new-model
!
R1(config)# aaa authentication login default group tacacs+ local
R1(config)# aaa authentication login CONSOLE_LOGIN local
!
R1(config)# line console 0
R1(config-line)# login authentication CONSOLE_LOGIN
!
R1(config)# line vty 0 4
R1(config-line)# login authentication default
R1(config-line)# transport input ssh
```

In this example:

- SSH login uses TACACS+ with local fallback.
- Console login always uses local authentication.
- The console remains independent of TACACS+ server availability for login authentication.

If AAA EXEC authorization is configured, console EXEC authorization must also be considered.

On IOS XE, console authorization behavior can differ from VTY authorization. The `aaa authorization console` command is used on platforms requiring explicit enablement of AAA authorization for console sessions.

**Protect console access before making changes to AAA authentication methods that could lock out remote administrators.**

---

## IOS XE Verification

### Display Authentication Method Lists

Use:

```text
R1# show running-config | include ^aaa authentication
```

Example output:

```text
aaa authentication login default group tacacs+ local
aaa authentication login CONSOLE_LOGIN local
aaa authentication enable default group tacacs+ enable
```

This displays configured login and enable authentication method lists.

### Verify Line Authentication

Display the relevant line configuration:

```text
R1# show running-config | section ^line con
```

```text
R1# show running-config | section ^line vty
```

Verify that the intended authentication method list is applied.

### Verify the Local User and Enable Secret

Use:

```text
R1# show running-config | include ^username
```

```text
R1# show running-config | include ^enable secret
```

The enable secret is stored as a hash in the configuration.

### Test TACACS+ Authentication

For example:

```text
R1# test aaa group tacacs+ admin Password123 legacy
```

This tests authentication against the TACACS+ server group.

### Test RADIUS Authentication

For example:

```text
R1# test aaa group radius admin Password123 new-code
```

The command tests RADIUS authentication.

**These commands test AAA authentication requests, not necessarily the actual `enable` authentication process or EXEC authorization.**

To verify enable authentication, test an actual privilege-level transition using `enable`.

### Debug Authentication

General AAA authentication debugging:

```text
R1# debug aaa authentication
```

TACACS+ authentication debugging:

```text
R1# debug tacacs authentication
```

RADIUS authentication debugging:

```text
R1# debug radius authentication
```

These commands can help determine which authentication method is being attempted and whether it succeeds, fails, or returns an error.

Disable debugging afterward:

```text
R1# undebug all
```

---

## Common Authentication Problems

| Problem | Possible Cause |
|---|---|
| SSH login fails | Incorrect method list, server configuration, or credentials |
| Named authentication list is not used | Method list not applied to the VTY or console line |
| Local fallback does not occur | Remote server explicitly rejected authentication |
| Console login fails after enabling AAA | Incorrect console authentication method list |
| Login succeeds but user enters privilege level 1 | No applicable privilege-level assignment |
| Login succeeds but EXEC session is denied | EXEC authorization failure |
| `enable` rejects the user's password | Incorrect AAA enable authentication policy |
| Local username password does not work for enable fallback | `enable` uses the enable secret, not the local username database |
| RADIUS enable authentication fails | RADIUS server does not recognize `$enab15$` or the configured credentials |
| TACACS+ enable authentication fails for a particular user | TACACS+ server policy rejects that administrator |
| Authentication takes too long | Server timeouts, retransmissions, or unreachable AAA servers |

The most important troubleshooting distinction is between:

1. **Authentication failure:** The configured authentication method explicitly rejects the credentials.
2. **Authentication error:** The method cannot complete authentication, allowing the next configured method to be attempted.

---

## Key Points

- Login authentication controls access to IOS XE terminal sessions.
- Enable authentication controls access to higher privilege levels when using `enable`.
- Configure login authentication with `aaa authentication login {default | <name>} <methods>`.
- Configure enable authentication with `aaa authentication enable default <methods>`.
- Login authentication supports both default and named method lists.
- Enable authentication supports only the `default` method list.
- Named login method lists must be applied to lines using `login authentication <name>`.
- The default login method list applies to applicable lines unless overridden.
- `local` authenticates against the local username database.
- `local-case` performs case-sensitive username authentication.
- `line` authenticates using the line password.
- `enable` authenticates using the locally configured enable password or secret.
- `none` permits authentication without credential verification.
- The `local` method is not supported for AAA enable authentication.
- Authentication methods are tried in order, but **fallback occurs only after an error, not an explicit rejection**.
- TACACS+ enable authentication can use the administrator's login username.
- RADIUS enable authentication uses a special username such as `$enab15$` for privilege level 15.
- `aaa authentication enable default group tacacs+ enable` uses TACACS+ first and the local enable secret as a fallback.
- Successful login authentication does not automatically assign privilege level 15.
- EXEC authorization is configured separately using `aaa authorization exec`.
- A dedicated local console authentication method list can preserve recovery access if centralized AAA servers are unavailable.
- Use `show running-config`, `test aaa group`, and `debug aaa authentication` for verification.

---

## References

- [Cisco IOS XE - Configuring Authentication](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_aaa/configuration/xe-16-12/sec-usr-aaa-xe-16-12-book/sec-cfg-authentifcn.html)
- [Cisco IOS Security Command Reference - AAA Authentication Commands](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/security/a1/sec-a1-cr-book/sec-cr-a1.html)
- [Cisco IOS - Configuring Authentication](https://www.cisco.com/c/en/us/td/docs/ios/sec_user_services/configuration/guide/convert/aaa/sec_cfg_authentifcn.html)
