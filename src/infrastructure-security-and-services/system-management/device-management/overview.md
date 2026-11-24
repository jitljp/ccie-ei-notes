# 4.3.a Device Management

Device management involves accessing, configuring, maintaining, and troubleshooting network devices.

Cisco IOS XE supports both traditional CLI-based management and programmatic management using structured data.

The appropriate management method depends on the task being performed, the available connectivity, and security requirements.

## Management Access Methods

Network devices can be accessed through several management interfaces and protocols.

| Access Method | Interface | Requires IP Connectivity | Encrypted |
|---|---|---|---|
| Console | CLI | No | N/A |
| Telnet | CLI | Yes | No |
| SSH | CLI | Yes | Yes |
| SCP | File transfer | Yes | Yes |
| NETCONF over SSH | Programmatic | Yes | Yes |
| RESTCONF over HTTPS | Programmatic | Yes | Yes |

Console, Telnet, and SSH provide access to the Cisco IOS XE CLI.

SCP provides file transfer capabilities rather than interactive CLI access.

NETCONF and RESTCONF provide programmatic interfaces for retrieving and modifying device data.

## CLI-Based Management

The Cisco IOS XE CLI allows administrators to enter commands to configure, monitor, and troubleshoot a device.

CLI access is provided through logical interfaces called **lines**.

### Line Types

Cisco IOS XE supports several line types:

| Line Type | Description |
|---|---|
| Console (CON) | Provides local CLI access through a console port |
| Auxiliary (AUX) | Provides serial access, traditionally through an external modem |
| Virtual Terminal (VTY) | Provides remote CLI access through protocols such as Telnet and SSH |
| TTY | Provides access through asynchronous serial interfaces on supported platforms |

The available line types and number of lines depend on the device platform.

The `show line` command displays information about the lines available on a device.

```text
R1# show line
```

Each line or range of lines can be configured independently.

For example:

```text
R1(config)# line console 0
R1(config-line)# exec-timeout 10 0
!
R1(config)# line vty 0 4
R1(config-line)# transport input ssh
R1(config-line)# exec-timeout 5 0
```

In this example:

- The console line has an inactivity timeout of 10 minutes.
- VTY lines 0 through 4 accept incoming SSH connections.
- VTY lines 0 through 4 have an inactivity timeout of 5 minutes.

### Line Configuration

Line configuration controls various aspects of management sessions:

- Authentication and authorization.
- Incoming and outgoing transport protocols.
- Access restrictions.
- EXEC session availability.
- Session and inactivity timeouts.
- Terminal settings.
- Rotary group membership.

Different lines can have different configurations.

For example, an administrator might configure separate ranges of VTY lines with different authentication methods or access restrictions.

### Rotary Groups

A **rotary group** is a logical grouping of lines.

Incoming connections to specific TCP ports can be directed to lines belonging to a particular rotary group.

Rotary groups can be used to:

- Provide access to specific groups of VTY lines.
- Apply different authentication and access control policies to different management sessions.
- Provide SSH access through custom TCP ports.
- Access asynchronous serial lines on terminal servers.

Rotary groups are particularly useful when a device must provide multiple management access methods with different requirements.

## Remote Management Protocols

### Telnet

Telnet provides remote access to a device's CLI over TCP.

By default, a Telnet server listens on TCP port 23.

Telnet does not encrypt the contents of a session. Usernames, passwords, commands, and command output can be observed by an attacker capable of capturing the traffic.

Although SSH is preferred in production networks, Telnet remains relevant for understanding and troubleshooting IOS XE line configuration, transport protocols, rotary groups, and legacy management environments.

### SSH

**Secure Shell (SSH)** provides encrypted remote access to a device's CLI.

By default, an SSH server listens on TCP port 22.

SSH provides several security functions:

- Server authentication.
- User authentication.
- Confidentiality through encryption.
- Data integrity protection.

Cisco IOS XE can operate as both an SSH server and an SSH client.

An administrator can establish an SSH session to a router, and the router itself can initiate an SSH session to another device.

```text
                      SSH
Administrator --------------------> R1

                      SSH
      R1 -------------------------> R2
```

SSH server configuration includes several components:

- Cryptographic host keys.
- SSH protocol version.
- Authentication methods.
- Authentication timeouts and retries.
- Supported cryptographic algorithms.
- VTY transport configuration.

Enabling the SSH server and permitting SSH on VTY lines are distinct configuration tasks.

### SCP

**Secure Copy Protocol (SCP)** provides secure file transfers using SSH.

SCP can be used to transfer:

- Device configuration files.
- Software images.
- Backup files.
- Other files stored in device file systems.

Cisco IOS XE can operate as both an SCP client and an SCP server on supported platforms.

When operating as an SCP client, the device initiates file transfers to or from a remote SCP server.

When operating as an SCP server, the device accepts file transfer requests from remote SCP clients.

Unlike an interactive SSH session, SCP is used to transfer files rather than provide CLI access.

## Programmatic Device Management

Traditional CLI-based management is designed primarily for human interaction.

Although scripts can automate CLI commands through SSH, this approach often requires parsing command output that is designed to be read by humans.

For example:

```text
R1# show ip interface brief
Interface              IP-Address      OK? Method Status    Protocol
GigabitEthernet1       192.0.2.1       YES manual up        up
GigabitEthernet2       198.51.100.1    YES manual up        up
Loopback0              1.1.1.1         YES manual up        up
```

A script must interpret the formatting of the output to extract the interface names, IP addresses, and interface states.

Model-driven management provides an alternative by exposing structured configuration and operational data.

### YANG Data Models

**YANG** is a data modeling language used to define the structure and characteristics of network device data.

YANG models can define:

- Configuration parameters.
- Operational state information.
- Data types and permitted values.
- Relationships between data elements.
- Remote procedure calls.
- Notifications.

NETCONF and RESTCONF use YANG models to provide structured access to device data.

YANG is a data modeling language, not a management protocol or data encoding format.

### NETCONF

The **Network Configuration Protocol (NETCONF)** provides mechanisms for installing, modifying, deleting, and retrieving network device configuration data.

NETCONF also supports retrieving operational data.

NETCONF uses XML-encoded messages and is typically transported over SSH using TCP port 830.

NETCONF uses a remote procedure call (RPC) mechanism. A client sends an RPC request, and the server returns an RPC reply.

```text
NETCONF Client                       NETCONF Server
                                             
      | -------- RPC Request -------------> |
      |                                     |
      | <------- RPC Reply ---------------- |
```

NETCONF supports operations such as:

- Retrieving configuration and operational data.
- Modifying configuration data.
- Locking configuration datastores.
- Copying configuration data.
- Validating and committing configuration changes, when supported by the server.

NETCONF capabilities are exchanged between the client and server when a session is established.

### RESTCONF

**RESTCONF** provides an HTTP-based interface to YANG-modeled device data.

RESTCONF uses standard HTTP methods to perform operations on resources.

Common HTTP methods include:

| HTTP Method | Purpose |
|---|---|
| GET | Retrieve data |
| POST | Create resources or invoke operations |
| PUT | Create or replace a resource |
| PATCH | Modify an existing resource |
| DELETE | Delete a resource |

RESTCONF supports XML and JSON data encoding and uses HTTPS to secure communication.

RESTCONF is particularly useful for integrating network devices with applications and automation tools that use HTTP-based APIs.

### NETCONF and RESTCONF Comparison

| Characteristic | NETCONF | RESTCONF |
|---|---|---|
| Data modeling language | YANG | YANG |
| Data encoding | XML | XML or JSON |
| Typical transport | SSH | HTTPS |
| Standard TCP port | 830 | 443 |
| Operation mechanism | RPCs | HTTP methods |
| Communication model | Persistent NETCONF session | HTTP request and response |
| Configuration datastore locking | Supported | No explicit locking operation |
| Candidate configuration and commit | Supported if advertised by the server | No explicit candidate/commit workflow |

Both protocols provide structured access to device data, but they differ in how clients interact with that data and manage configuration changes.

## Management Access Security

Management access must be protected because it allows administrators and applications to interact with the device's configuration and operational state.

Important security mechanisms include:

- Using encrypted management protocols instead of unencrypted alternatives.
- Restricting management access to trusted source addresses.
- Configuring appropriate authentication and authorization.
- Limiting the privileges of management users.
- Disabling unnecessary management services.
- Configuring session inactivity timeouts.
- Isolating management traffic using dedicated management interfaces or VRFs.
- Protecting the management plane from excessive or unauthorized traffic.

Authentication, Authorization, and Accounting (AAA) is covered in [4.1.b AAA](../../device-security/aaa/overview.md).

Control plane protection and Management Plane Protection (MPP) are covered in [4.1 Device Security](../../device-security/overview.md).

## Key Concepts

- Console access provides CLI access without requiring IP connectivity.
- VTY lines provide remote CLI access through protocols such as Telnet and SSH.
- Line configuration controls authentication, transport protocols, access restrictions, and session behavior.
- Rotary groups allow incoming connections to be directed to specific groups of lines.
- Telnet provides unencrypted remote CLI access using TCP port 23.
- SSH provides encrypted remote CLI access using TCP port 22 by default.
- SCP provides secure file transfers using SSH.
- Cisco IOS XE can operate as both a client and a server for supported remote management protocols.
- YANG defines the structure and characteristics of device data.
- NETCONF uses XML-encoded RPC messages, typically over SSH using TCP port 830.
- RESTCONF uses HTTP methods and XML or JSON encoding over HTTPS.
- NETCONF and RESTCONF provide structured, model-driven alternatives to traditional CLI-based management.
