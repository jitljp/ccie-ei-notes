
# AAA Method Lists and Processing

**AAA method lists** define which authentication, authorization, or accounting methods Cisco IOS XE uses and the order in which they are attempted.

A method list can specify a single method or multiple methods for redundancy.

For example:

```text
R1(config)# aaa authentication login default group tacacs+ local
```

This configuration tells the router to:

1. Attempt authentication using TACACS+.
2. Use the local user database if TACACS+ authentication cannot be completed because of an error, such as server unavailability.

**The next method is attempted only if the current method returns an error, not if it explicitly rejects the request.**

Method lists are configured separately for authentication, authorization, and accounting.

---

## AAA Method List Types

Cisco IOS XE uses different method lists for different AAA services.

| Function | Method List | Purpose |
|---|---|---|
| Authentication | `aaa authentication login` | Authenticates users logging in |
| Authentication | `aaa authentication enable` | Authenticates requests for privileged EXEC access |
| Authorization | `aaa authorization exec` | Authorizes EXEC sessions and assigns session attributes |
| Authorization | `aaa authorization commands` | Authorizes individual commands at specified privilege levels |
| Authorization | `aaa authorization network` | Authorizes network services |
| Accounting | `aaa accounting exec` | Records EXEC session activity |
| Accounting | `aaa accounting commands` | Records commands executed at specified privilege levels |
| Accounting | `aaa accounting network` | Records network service activity |

Each AAA service has its own method lists.

For example, the following commands configure three independent lists:

```text
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local
aaa accounting exec default start-stop group tacacs+
```

The authentication list determines how the user is authenticated.

The authorization list determines how the EXEC session is authorized.

The accounting list determines where EXEC accounting records are sent.

**Configuring a method list for one AAA function does not automatically configure the other functions.**

---

## Default and Named Method Lists

AAA supports two kinds of method lists:

- **Default method lists:** Used automatically for the applicable service when no named method list overrides them.
- **Named method lists:** Explicitly assigned to supported lines, interfaces, or services.

### Default Method Lists

The keyword `default` identifies the default method list for a particular AAA service.

For example:

```text
R1(config)# aaa authentication login default group tacacs+ local
```

This configures the default login authentication list.

Unless a named authentication list overrides it, the default list is used for login authentication on applicable lines.

The keyword `default` is a reserved list identifier, not the name of an AAA server group.

### Named Method Lists

A named method list uses an administrator-defined name.

For example:

```text
R1(config)# aaa authentication login VTY_AUTH group tacacs+ local
```

This creates a login authentication method list named `VTY_AUTH`.

The list must be applied to the desired lines:

```text
R1(config)# line vty 0 4
R1(config-line)# login authentication VTY_AUTH
```

The VTY lines now use `VTY_AUTH` instead of the default login authentication list.

**Creating a named method list does not automatically activate it.**

### Default vs. Named List Example

Consider the following configuration:

```text
aaa authentication login default local
aaa authentication login VTY_AUTH group tacacs+ local
!
line console 0
 login authentication default
!
line vty 0 4
 login authentication VTY_AUTH
!
line vty 5 15
 login authentication default
```

| Access Method | Method List | Authentication Order |
|---|---|---|
| Console | `default` | Local |
| VTY 0–4 | `VTY_AUTH` | TACACS+, then local on error |
| VTY 5–15 | `default` | Local |

A named method list overrides the default list only where it is applied.

> Console authorization has additional behavior controlled by `aaa authorization console`. This is separate from login authentication and is covered in the authorization pages.

---

## AAA Method List Processing

Methods are evaluated from left to right.

The result of each method determines whether processing stops or continues.

For authentication and authorization, the three fundamental results are:

| Result | Meaning | Next Method Attempted? |
|---|---|---|
| **PASS** | Authentication or authorization succeeds | No |
| **FAIL** | Authentication or authorization is explicitly rejected | No |
| **ERROR** | The method cannot complete the request | Yes |

### PASS

A PASS result indicates successful authentication or authorization.

For example, a TACACS+ server accepts a user's credentials.

The router immediately stops processing the method list.

Any remaining methods are ignored.

### FAIL

A FAIL result indicates that authentication or authorization was explicitly rejected.

For example, a TACACS+ server responds that the username or password is invalid.

The router stops processing the method list and rejects the request.

**A FAIL result does not trigger fallback.**

### ERROR

An ERROR result means that the method could not complete the request.

Examples include:

- The TACACS+ server is unreachable.
- The RADIUS server does not respond before the configured retries and timeouts are exhausted.
- The AAA method encounters an error that prevents a valid decision.

The router proceeds to the next configured method.

If all methods return errors and no method successfully completes the request, authentication or authorization fails.

---

## Authentication Fallback Example

Consider:

```text
aaa authentication login default group tacacs+ local
```

Assume the following local user exists:

```text
username admin secret Cisco123
```

The router evaluates the authentication list as follows:

| TACACS+ Result | Local Database Checked? | Outcome |
|---|---|---|
| TACACS+ accepts credentials | No | Login succeeds |
| TACACS+ rejects credentials | No | Login fails |
| All TACACS+ servers are unreachable | Yes | Local credentials are checked |
| TACACS+ unavailable, local credentials valid | Yes | Login succeeds |
| TACACS+ unavailable, local credentials invalid | Yes | Login fails |

For example:

**Scenario 1: TACACS+ rejects the password**

```text
TACACS+ → FAIL
```

Result: Login denied.

The router does not try local authentication.

**Scenario 2: TACACS+ is unavailable**

```text
TACACS+ → ERROR
Local   → PASS
```

Result: Login succeeds.

**Scenario 3: TACACS+ is unavailable and local authentication fails**

```text
TACACS+ → ERROR
Local   → FAIL
```

Result: Login denied.

The distinction between **FAIL and ERROR** is essential for understanding AAA fallback.

---

## Common Authentication Methods

The following methods are commonly used with `aaa authentication login`.

| Method | Behavior |
|---|---|
| `group tacacs+` | Uses all configured TACACS+ servers |
| `group radius` | Uses all configured RADIUS servers |
| `group <name>` | Uses a named AAA server group |
| `local` | Uses the local username database |
| `local-case` | Uses the local username database with case-sensitive username matching |
| `enable` | Uses the configured enable password or secret |
| `line` | Uses the password configured under the line |
| `none` | Allows authentication without checking credentials |

For example:

```text
aaa authentication login default group tacacs+ group radius local
```

Processing order:

1. TACACS+.
2. RADIUS, only if TACACS+ returns an error.
3. Local authentication, only if RADIUS also returns an error.

If any method returns PASS or FAIL, processing stops.

### The `none` Method

The `none` method succeeds without requiring authentication credentials.

For example:

```text
aaa authentication login default group tacacs+ none
```

If TACACS+ is unavailable, the router proceeds to `none` and allows the login without validating credentials.

However, if TACACS+ explicitly rejects the login, authentication fails before `none` is reached.

> Using `none` as an authentication fallback can create a serious security vulnerability. It is useful for understanding AAA processing but should not ordinarily be used for administrative login access.

---

## Authorization Method Lists

Authorization method lists follow the same general processing principle as authentication lists.

For example:

```text
aaa authorization exec default group tacacs+ local
```

The router first attempts EXEC authorization through TACACS+.

If the TACACS+ server returns an authorization rejection, access is denied.

If TACACS+ authorization cannot be completed because of an error, the router attempts local authorization.

### Authorization Methods

Common authorization methods include:

| Method | Behavior |
|---|---|
| `group tacacs+` | Requests authorization from TACACS+ servers |
| `group radius` | Requests authorization from RADIUS servers |
| `group <name>` | Uses a named AAA server group |
| `local` | Uses locally configured authorization information |
| `if-authenticated` | Allows authorization if the user has already authenticated |
| `none` | Allows the requested service without performing an authorization check |

Supported methods depend on the authorization service.

### `if-authenticated`

The `if-authenticated` method authorizes the requested service if the user has already been authenticated.

For example:

```text
aaa authorization exec default group tacacs+ if-authenticated
```

If TACACS+ authorization succeeds, its result is used.

If TACACS+ returns an error, the router proceeds to `if-authenticated`.

Because the user has already passed authentication, EXEC authorization succeeds.

However, if TACACS+ explicitly denies authorization, `if-authenticated` is not attempted.

**`if-authenticated` is a terminating authorization method.** Subsequent methods are not evaluated.

### `none`

The `none` method allows the authorization request without an authorization check.

For example:

```text
aaa authorization exec default group tacacs+ none
```

If TACACS+ returns an error, EXEC authorization succeeds without an additional authorization check.

Unlike `if-authenticated`, `none` does not itself require successful authentication as a condition for authorization.

Both `none` and `if-authenticated` can serve as authorization fallback methods, but they do not provide the same authorization policies or attributes as a successfully contacted TACACS+ server.

### `local` vs. `if-authenticated`

Consider:

```text
aaa authorization exec default group tacacs+ local
```

If TACACS+ is unavailable, the router attempts to obtain authorization using local information.

Now consider:

```text
aaa authorization exec default group tacacs+ if-authenticated
```

If TACACS+ is unavailable, the router permits the EXEC session based on successful prior authentication.

| Method | Fallback Behavior |
|---|---|
| `local` | Uses local authorization information |
| `if-authenticated` | Permits the service if the user was authenticated |
| `none` | Permits the service without an authorization check |

These methods do not necessarily assign identical privileges.

For example, `if-authenticated` does not retrieve a TACACS+ privilege-level assignment when the server is unavailable.

---

## Accounting Method Lists

Accounting method lists determine where accounting records are sent.

For example:

```text
aaa accounting exec default start-stop group tacacs+
```

This configures EXEC accounting using TACACS+.

Accounting method lists can specify multiple server groups:

```text
aaa accounting exec default start-stop group tacacs+ group radius
```

If the first accounting method cannot communicate with its servers, the router can attempt the next method.

Accounting differs from authentication and authorization because its main purpose is **record delivery**, not deciding whether a user is permitted to log in or execute commands.

The accounting record types, `start-stop`, `stop-only`, `wait-start`, and failure behavior are covered in the AAA Accounting page.

---

## AAA Method Lists vs. Server Groups

A method list and an AAA server group serve different purposes.

- **Method list:** Determines the sequence of AAA methods.
- **Server group:** Determines which external servers are used by a method.

For example:

```text
aaa group server tacacs+ TAC_SERVERS
 server name TAC1
 server name TAC2
!
aaa authentication login default group TAC_SERVERS local
```

The method list contains two methods:

1. `group TAC_SERVERS`
2. `local`

The first method refers to a server group containing TAC1 and TAC2.

The router attempts to use the TACACS+ servers according to the server group's selection and failover behavior.

The router does not proceed to `local` merely because TAC1 fails to respond.

It proceeds to `local` only when the TACACS+ method as a whole returns an error.

**Failover between servers within a group is different from fallback between methods in a method list.**

---

## Verification

Display the configured AAA method lists:

```text
R1# show running-config | section ^aaa
```

The `show aaa method-lists` command can also display configured method-list information on supported IOS XE platforms:

```text
R1# show aaa method-lists
```

To verify which method list is applied to VTY or console lines:

```text
R1# show running-config | section ^line
```

For authentication troubleshooting, useful commands include:

```text
R1# debug aaa authentication
R1# debug aaa authorization
R1# debug aaa accounting
```

Protocol-specific debugging commands can also help determine whether a TACACS+ or RADIUS server is accepting, rejecting, or failing to respond to requests.

Disable debugging after troubleshooting:

```text
R1# undebug all
```

---

## Key Points

- AAA method lists define the **methods and processing order** for authentication, authorization, and accounting.
- Authentication, authorization, and accounting use **separate method lists**.
- The `default` method list is used automatically for the applicable service unless overridden.
- Named method lists must be **explicitly applied** to supported lines or services.
- Methods are processed **from left to right**.
- **PASS:** The request succeeds and processing stops.
- **FAIL:** The request is explicitly rejected and processing stops.
- **ERROR:** The current method cannot complete the request, so the next method is attempted.
- An explicit TACACS+ or RADIUS rejection **does not trigger local fallback**.
- `local` uses locally configured authentication or authorization information.
- `if-authenticated` permits authorization when the user has already authenticated.
- `none` permits authentication or authorization without the corresponding security check.
- Server failover within a group is distinct from method-list fallback.
- If every method returns an error without a successful result, authentication or authorization fails.

---

## References

- [Cisco IOS XE - Configuring Authentication](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_aaa/configuration/xe-16-12/sec-usr-aaa-xe-16-12-book/sec-cfg-authentifcn.html)
- [Cisco IOS XE - Configuring Accounting](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_aaa/configuration/xe-16-11/sec-usr-aaa-xe-16-11-book/sec-cfg-accountg.html)
- [Cisco IOS Security Command Reference - AAA Commands](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/security/a1/sec-a1-cr-book/sec-cr-a1.html)
