
# 4.1.b AAA

**Authentication, Authorization, and Accounting (AAA)** is a framework for controlling access to network devices and services, determining what authenticated users are permitted to do, and recording their activities.

AAA provides three independent but related security functions:

1. **Authentication:** Verifies the identity of a user.
2. **Authorization:** Determines which services or operations the user is permitted to access.
3. **Accounting:** Records user activity and resource usage.

Cisco IOS XE supports AAA using a **local user database** or external **TACACS+** and **RADIUS** servers.

AAA is commonly used to secure administrative access to routers and switches through SSH and the console, enforce privilege levels and command permissions, and maintain an audit trail of administrative activity.

---

## AAA Functions

| Function | Purpose | Example |
|---|---|---|
| **Authentication** | Verifies who the user is | Validate an SSH user's username and password |
| **Authorization** | Determines what the user can do | Allow a user to execute `show` commands but deny configuration commands |
| **Accounting** | Records what the user does | Record when a user logs in and which commands they execute |

These functions can operate independently.

For example, a device can authenticate a user locally, authorize the user's EXEC session through TACACS+, and send accounting records to an external server.

### Authentication

Authentication determines whether a user is permitted to access the device or a particular service.

Cisco IOS XE supports several authentication methods, including:

- **Local authentication:** Uses usernames and passwords configured on the device.
- **TACACS+:** Sends authentication requests to a TACACS+ server.
- **RADIUS:** Sends authentication requests to a RADIUS server.
- **Enable authentication:** Verifies credentials when a user requests privileged EXEC access.

Authentication is typically performed when a user attempts to establish an administrative session.

Successful authentication does not necessarily mean that the user has permission to execute every command.

### Authorization

Authorization determines which services and operations an authenticated user is permitted to access.

Cisco IOS XE supports several types of authorization.

| Type | Purpose |
|---|---|
| **EXEC authorization** | Controls access to an EXEC shell and can assign the user's privilege level |
| **Command authorization** | Determines whether individual commands are permitted |
| **Network authorization** | Controls access to network services, such as PPP |
| **Configuration command authorization** | Allows configuration commands to be subjected to command authorization |

For example, two users might authenticate successfully through TACACS+ but receive different permissions:

- **Administrator:** Privilege level 15, with permission to execute configuration commands.
- **Operator:** Privilege level 1, with permission to execute only a restricted set of commands.

Authorization is separate from authentication and must be configured for the services that require it.

### Accounting

Accounting records events associated with user sessions and activities.

Common accounting functions include:

- **EXEC accounting:** Records the beginning and end of administrative EXEC sessions.
- **Command accounting:** Records commands executed by users at configured privilege levels.
- **Network accounting:** Records network service sessions and related information.

Accounting records can be sent to TACACS+ or RADIUS servers.

Accounting provides visibility into administrative activity, but it does not determine whether a user is permitted to execute a command.

---

## AAA Operation

A typical AAA-controlled SSH login follows this sequence:

1. A user establishes an SSH connection to a router.
2. The router authenticates the user using the configured authentication method list.
3. If authentication succeeds, the router performs EXEC authorization if configured.
4. If authorization succeeds, the user receives an EXEC session with the appropriate privileges.
5. If command authorization is configured, the router checks permission when the user executes commands covered by the policy.
6. If accounting is configured, the router generates records for the relevant sessions and activities.

Authentication, authorization, and accounting are separate processes.

For example, a successful TACACS+ authentication does not automatically enable TACACS+ command authorization or accounting.

Each function requires its own AAA configuration.

---

## Enabling AAA

AAA is enabled globally using:

```text
R1(config)# aaa new-model
```

This enables the Cisco IOS XE AAA framework.

After enabling AAA, configure the required authentication, authorization, and accounting method lists.

> Enabling `aaa new-model` changes how the device handles authentication. Before making AAA changes on a remotely managed device, ensure that a working authentication method is available to prevent administrative lockout.

---

## AAA Method Lists

AAA uses **method lists** to determine how authentication, authorization, and accounting requests are processed.

A method list specifies one or more methods, evaluated in a defined order.

For example:

```text
R1(config)# aaa authentication login default group tacacs+ local
```

This method list:

1. Attempts login authentication using TACACS+.
2. Falls back to the local user database if TACACS+ authentication cannot be completed because of an error or server unavailability.

**An explicit authentication rejection does not trigger fallback to the next method.**

For example, if the TACACS+ server responds that the user's credentials are invalid, authentication fails rather than continuing to the local database.

Method lists can be:

- **Default:** Automatically used for the applicable AAA service where no named list overrides it.
- **Named:** Explicitly applied to supported lines or services.

Authentication, authorization, and accounting use separate method lists.

---

## Local AAA vs. External AAA

Cisco IOS XE can perform AAA operations locally or communicate with external authentication servers.

| Feature | Local AAA | External AAA |
|---|---|---|
| User database | Stored on each device | Centralized on AAA servers |
| Authentication | Supported | Supported |
| Authorization | Local privileges and supported policies | Centralized policies and permissions |
| Accounting | Limited local logging capabilities | Centralized accounting records |
| Scalability | Best suited to smaller deployments or fallback access | Better suited to managing many devices |

Local authentication is commonly configured as a fallback when external AAA servers become unavailable.

External AAA provides centralized administration, consistent access policies, and auditing across multiple network devices.

---

## TACACS+ vs. RADIUS

The two primary external AAA protocols supported by Cisco IOS XE are **TACACS+** and **RADIUS**.

| Feature | TACACS+ | RADIUS |
|---|---|---|
| Transport | TCP | UDP |
| Default ports | TCP 49 | UDP 1812 (authentication), 1813 (accounting) |
| Authentication and authorization | Separate protocol operations | Authorization information returned with authentication |
| Encryption | Protects most of the packet body | Protects selected fields, such as passwords, rather than the entire packet |
| Command authorization | Granular per-command authorization | Not typically used for granular CLI command authorization |
| Common use | Network device administration | Network access authentication, such as 802.1X |

**TACACS+ is generally preferred for administrative access to network devices** because it supports granular command authorization and command accounting.

RADIUS is commonly used for network access authentication and authorization, although it can also be used for device administration.

Both protocols support centralized authentication, authorization, and accounting.

---

## AAA Server Groups

AAA servers can be organized into **server groups**.

Server groups allow administrators to:

- Specify which TACACS+ or RADIUS servers are used for a particular AAA method list.
- Use multiple servers for availability.
- Configure server communication parameters.
- Support VRF-aware AAA communication.

For example, an organization might use separate TACACS+ server groups for different administrative domains.

A method list can reference either all configured servers of a particular protocol or a specific server group.

---

## AAA and Privilege Levels

Cisco IOS XE supports **16 privilege levels**, numbered 0 through 15.

| Privilege Level | Purpose |
|---|---|
| **0** | Minimal commands, such as `enable`, `disable`, `exit`, and `help` |
| **1** | Default user EXEC privilege level |
| **2–14** | Customizable intermediate privilege levels |
| **15** | Highest privilege level, normally providing unrestricted CLI access |

Privilege levels can be assigned through local user configuration or external AAA authorization.

Command authorization provides additional control by allowing an external AAA server to approve or reject individual commands.

---

## What This Section Covers

The following pages examine:

- AAA method lists, processing order, and fallback behavior.
- Local users, passwords, secrets, and privilege levels.
- TACACS+ protocol operation and IOS XE configuration.
- RADIUS protocol operation and IOS XE configuration.
- AAA server groups and VRF-aware communication.
- Login and enable authentication.
- EXEC and command authorization.
- AAA accounting methods and record types.
- Server failover, deadtime, and local fallback.
- Complete AAA configurations combining authentication, authorization, and accounting.
- Verification and troubleshooting commands.

The primary objective is to understand **how IOS XE selects AAA methods, handles authentication and authorization results, enforces administrative permissions, and records user activity**.

---

## References

- [Cisco IOS XE 17.x - Configuring Authentication](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-cfg-authentifcn-0.html)
- [Cisco IOS XE 17.x - Configuring Authorization](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-cfg-authorization.html)
- [Cisco IOS XE 17.x - Configuring Accounting](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-cfg-accountg-0.html)
- [Cisco IOS XE 17.x - Configuring RADIUS](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-cfg-radius.html)
