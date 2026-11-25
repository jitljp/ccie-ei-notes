# 4.3.a (ii) SSH, SCP

**Secure Shell (SSH)** and **Secure Copy Protocol (SCP)** provide encrypted remote management and file transfer capabilities for Cisco IOS XE devices.

Both protocols use SSH to establish secure communication between devices.

Their primary purposes are different:

- **SSH:** Provides secure remote CLI access to network devices.
- **SCP:** Provides secure file transfers between network devices and other systems.

SSH replaces unencrypted remote management protocols such as Telnet, while SCP provides a secure alternative to file transfer protocols such as TFTP and FTP.

## SSH Overview

SSH is a client-server protocol that provides secure remote access over TCP.

By default, SSH servers listen on **TCP port 22**.

SSH provides four important security functions:

1. **Server authentication:** Allows the client to verify the identity of the SSH server.
2. **User authentication:** Verifies that the user is authorized to establish a connection.
3. **Confidentiality:** Encrypts transmitted data to prevent unauthorized disclosure.
4. **Integrity:** Protects transmitted data against unauthorized modification.

Unlike Telnet, SSH encrypts usernames, passwords, commands, and command output.

### SSH Client and Server

Cisco IOS XE devices can operate as both SSH clients and SSH servers.

An **SSH client** initiates a connection, while an **SSH server** accepts incoming connections.

For example:

```text
Administrator
192.0.2.100
     |
     | SSH (TCP/22)
     |
     v
    R1
192.0.2.1
```

In this example:

- The administrator's computer operates as the SSH client.
- R1 operates as the SSH server.
- The administrator establishes an encrypted CLI session with R1.

A Cisco IOS XE router can also initiate an SSH connection to another router.

```text
          SSH                SSH
Administrator ---> R1 ----------------> R2
                   |                    |
                SSH Server           SSH Server
                SSH Client
```

R1 acts as an SSH server for the administrator's connection and as an SSH client when connecting to R2.

The SSH client and server functions are independent.

### SSH Protocol Versions

There are two major versions of SSH:

| Version | Description |
|---|---|
| SSHv1 | Legacy protocol with known security weaknesses |
| SSHv2 | Modern SSH protocol with improved security and cryptographic capabilities |

**SSHv2 should be used for device management.**

Cisco IOS XE supports configuring the SSH protocol version using:

```text
R1(config)# ip ssh version 2
```

This explicitly restricts SSH connections to version 2.

The supported protocol versions and default behavior depend on the IOS XE release.

## SSH Connection Establishment

An SSH connection involves several stages.

### 1. TCP Connection

The SSH client establishes a TCP connection to the SSH server.

By default, the destination TCP port is 22.

### 2. Protocol and Algorithm Negotiation

The client and server exchange SSH protocol information and negotiate supported cryptographic algorithms.

SSH uses several categories of cryptographic algorithms:

| Algorithm Category | Purpose |
|---|---|
| Key Exchange (KEX) | Establish shared cryptographic key material |
| Host Key | Authenticate the SSH server |
| Encryption Cipher | Encrypt transmitted data |
| Message Authentication Code (MAC) | Protect message integrity |
| Public-Key Signature | Authenticate identities or protocol messages |

SSH uses key exchange to establish shared session keys without transmitting those keys directly across the network.

The negotiated algorithms must be supported by both devices.

### 3. Server Authentication

The SSH server proves its identity using a cryptographic host key.

The SSH client verifies the server's identity against a previously trusted host key or another configured trust mechanism.

For example, an administrator connecting to a device for the first time may be prompted to accept the server's host key.

On subsequent connections, the client can detect an unexpected host key change.

Server authentication helps protect against man-in-the-middle attacks.

### 4. User Authentication

After the secure transport has been established, the SSH server authenticates the connecting user.

Cisco IOS XE supports authentication mechanisms such as:

- Local username and password authentication.
- AAA authentication through TACACS+ or RADIUS.
- SSH public-key authentication.

For example, a router might authenticate SSH users against its local username database.

```text
R1(config)# username ADMIN secret ADMIN-PASSWORD
!
R1(config)# line vty 0 15
R1(config-line)# login local
```

User authentication and server authentication are separate processes.

### 5. Secure Session

After successful authentication, the SSH client can request a session.

For interactive device management, this normally provides access to the IOS XE CLI.

Commands and command output are encrypted during transmission.

## SSH Configuration Overview

Configuring an SSH server on Cisco IOS XE involves several components.

| Component | Purpose |
|---|---|
| Hostname and domain name | Identify the device and support conventional RSA key generation |
| Cryptographic host keys | Enable SSH server authentication |
| SSH version | Specify the permitted SSH protocol version |
| User authentication | Authenticate connecting users |
| VTY configuration | Permit incoming SSH connections |
| IP connectivity | Allow clients to reach the SSH server |

### Basic SSH Server Configuration

The following example configures R1 as an SSH server using local username authentication.

```text
R1(config)# hostname R1
R1(config)# ip domain name jitl.lab
R1(config)# ip ssh version 2
R1(config)# username ADMIN secret ADMIN-PASSWORD
R1(config)# end
R1# crypto key generate rsa modulus 2048
!
R1(config)# line vty 0 15
R1(config-line)# login local
R1(config-line)# transport input ssh
```

> Prior to Cisco IOS XE 17.11.1a, the `crypto key generate rsa` command was entered in global configuration mode. Starting with IOS XE 17.11.1a, it is entered in privileged EXEC mode.

The configuration performs several functions:

1. Configures the hostname and domain name.
2. Configures SSH version 2.
3. Creates a local user.
4. Generates a 2048-bit RSA key pair for SSH.
5. Configures the VTY lines to authenticate users against the local database.
6. Permits SSH connections while denying Telnet connections on those VTY lines.

**Generating an appropriate cryptographic host key enables the SSH server.**

The `transport input ssh` command permits incoming SSH connections on the specified VTY lines. It does not independently enable the SSH server.


### Enabling SSH Without a Hostname and Domain Name

By default, IOS XE uses the device's fully qualified domain name (FQDN) as the RSA key pair name. Therefore, a hostname and domain name must be configured before generating an RSA key pair without an explicit label.

Alternatively, an explicitly named RSA key pair can be generated and associated with the SSH server.

```text
Router# crypto key generate rsa label SSH-KEYS modulus 2048
```

- The `label` keyword specifies the name of the generated RSA key pair.

Neither a hostname nor a domain name needs to be explicitly configured when using this method.

### Initiating an SSH Connection

An SSH client can connect to R1 using:

```text
ssh ADMIN@192.0.2.1
```

From another Cisco IOS XE router:

```text
R2# ssh -l ADMIN 192.0.2.1
```

Both commands initiate an SSH connection to R1 using the username `ADMIN`.

### SSH Verification

Cisco IOS XE provides several commands for verifying SSH configuration and operation.

| Command | Purpose |
|---|---|
| `show ip ssh` | Display SSH server status, version, and configured parameters |
| `show ssh` | Display active SSH connections |
| `show users` | Display active user sessions |
| `show crypto key mypubkey rsa` | Display information about local RSA public keys |
| `show running-config \| section line vty` | Display VTY configuration |

For example:

```text
R1# show ip ssh
```

This command can confirm whether the SSH server is enabled and which protocol version is configured.

SSH configuration and verification are covered in detail in [SSH Configuration and Verification](ssh.md).

## SSH Security Options

Cisco IOS XE provides additional mechanisms for controlling and securing SSH connections.

These include:

- Restricting supported key exchange algorithms.
- Restricting encryption and integrity algorithms.
- Configuring SSH public-key authentication.
- Controlling authentication timeouts and retries.
- Configuring SSH session rekeying.
- Restricting SSH access to trusted IP addresses.
- Using AAA authentication and authorization.
- Using management VRFs.
- Configuring custom SSH ports.

### Authentication Timeout and Retries

SSH authentication behavior can be controlled using:

```text
R1(config)# ip ssh time-out 60
R1(config)# ip ssh authentication-retries 3
```

The `ip ssh time-out` command controls the SSH authentication negotiation timeout.

It is distinct from the VTY `exec-timeout` command, which controls the inactivity timeout of an established EXEC session.

The `ip ssh authentication-retries` command limits authentication retry attempts.

### Restricting SSH Access

SSH traffic can be restricted using an access class on VTY lines.

For example:

```text
R1(config)# ip access-list standard SSH-CLIENTS
R1(config-std-nacl)# permit 192.0.2.100
!
R1(config)# line vty 0 15
R1(config-line)# access-class SSH-CLIENTS in
```

This restricts incoming VTY connections to the permitted source address.

Additional security mechanisms, such as Control Plane Policing (CoPP) and Management Plane Protection (MPP), can also protect management access.

SSH authentication and security features are covered in [SSH Authentication and Security Options](authentication.md).

## SCP Overview

**Secure Copy Protocol (SCP)** provides encrypted file transfers using SSH.

SCP uses SSH for authentication and secure transport.

Like SSH, SCP normally uses **TCP port 22**.

Unlike an interactive SSH session, SCP transfers files rather than providing an interactive CLI.

SCP is commonly used to transfer:

- IOS XE software images.
- Startup and running configuration backups.
- Configuration files for restoration.
- Other files stored in the device's file systems.

### SCP Client and Server

Cisco IOS XE devices can operate as SCP clients or SCP servers.

The SCP client initiates a file transfer, while the SCP server responds to file transfer requests.

For example:

```text
Administrator                     R1
192.0.2.100                   192.0.2.1
     |                            |
     |     SCP (TCP/22)           |
     | -------------------------> |
     |                            |
     |        File Upload         |
     | =========================> |
     |                            |
```

In this example, the administrator uploads a file to R1.

The same connection model also supports downloading files from the device.

### SCP Client Operation

When Cisco IOS XE operates as an SCP client, it initiates file transfers to or from a remote SCP server.

For example, R1 can copy a file from its local flash storage to a remote SCP server.

```text
R1# copy flash:backup.cfg scp:
```

The router prompts for information such as the remote server address, username, and destination filename.

A router can also retrieve a file from a remote SCP server:

```text
R1# copy scp: flash:
```

**The `ip scp server enable` command is not required for client-initiated SCP transfers.**

The device must have a working SSH client, connectivity to the destination, and valid authentication credentials.

### SCP Server Operation

When Cisco IOS XE operates as an SCP server, it accepts file transfer requests from remote clients.

SCP server functionality must be explicitly enabled.

```text
R1(config)# ip scp server enable
```

However, this command alone is not sufficient.

The device must also have:

- A functioning SSH server.
- Appropriate user authentication.
- A user with sufficient privileges to perform the transfer.

## SCP Authentication and Authorization

SCP uses SSH to authenticate the connecting user, but authorization determines whether the user is permitted to perform the requested file operation.

For an IOS XE SCP server, **AAA EXEC authorization** is required.

The following example enables SCP server functionality using the local user database.

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login default local
R1(config)# aaa authorization exec default local
!
R1(config)# username ADMIN privilege 15 secret ADMIN-PASSWORD
!
R1(config)# ip scp server enable
```

This example assumes SSH is already configured and functioning, and the VTY lines permit SSH using the configured authentication method.

The `aaa authorization exec default local` command authorizes the user against the local database.

The `ADMIN` account has privilege level 15, allowing administrative SCP file operations.

SCP server authentication and authorization can also use centralized AAA services such as TACACS+.

### File Transfer from a Workstation

Assume R1 is configured as an SCP server and has the IP address `192.0.2.1`.

From a workstation with an OpenSSH SCP client, an administrator can upload a file:

```text
scp -O backup.cfg ADMIN@192.0.2.1:backup.cfg
```

To download the file:

```text
scp -O ADMIN@192.0.2.1:backup.cfg backup.cfg
```

The `-O` option forces the OpenSSH client to use the legacy SCP file transfer protocol instead of SFTP.

**Important:** Starting with OpenSSH 9.0, the `scp` command uses SFTP for file transfers by default.

Cisco IOS XE SCP servers that do not support SFTP can reject these transfers even when SSH authentication succeeds.

Using `scp -O` forces the client to use the SCP protocol, allowing interoperability with those devices.

SCP configuration, authentication requirements, file transfer commands, and troubleshooting are covered in [SCP Configuration and File Transfers](scp.md).

## SSH and SCP Comparison

Although SSH and SCP are closely related, their functions are different.

| Characteristic | SSH | SCP |
|---|---|---|
| Primary purpose | Secure remote access | Secure file transfers |
| Default TCP port | 22 | 22 |
| Encryption | Yes | Yes, through SSH |
| Authentication | SSH authentication | SSH authentication |
| Interactive CLI access | Yes | No |
| File transfers | Not the primary purpose | Yes |
| IOS XE client support | Yes | Yes |
| IOS XE server support | Yes | Yes |
| VTY SSH configuration | Required for incoming CLI SSH | SSH server accessibility is required |
| Additional server configuration | SSH host keys and authentication | `ip scp server enable` and appropriate AAA authorization |

**SCP depends on SSH, but enabling SSH does not automatically enable SCP server functionality.**

A router can support incoming SSH CLI sessions without permitting incoming SCP file transfers.

Similarly, a router can initiate SCP file transfers as a client without being configured as an SCP server.

## Troubleshooting SSH and SCP

SSH and SCP failures often result from configuration problems rather than basic IP connectivity problems.

### SSH Troubleshooting

Common SSH problems include:

| Problem | Possible Cause |
|---|---|
| TCP connection cannot be established | Routing, ACL, firewall, or TCP port restriction |
| SSH server is disabled | Missing or unusable cryptographic host keys |
| Connection rejected | VTY transport configuration or access restrictions |
| Authentication fails | Incorrect credentials or AAA configuration |
| SSH algorithm negotiation fails | No mutually supported cryptographic algorithms |
| SSH session disconnects | Authentication timeout, EXEC timeout, or network interruption |

A useful troubleshooting sequence is:

1. Verify IP connectivity to the device.
2. Verify TCP port 22 is reachable.
3. Check SSH server status using `show ip ssh`.
4. Verify the cryptographic host keys.
5. Check the VTY transport configuration.
6. Verify authentication and authorization.
7. Check supported SSH algorithms and session status.

### SCP Troubleshooting

SCP introduces additional potential failure points.

| Problem | Possible Cause |
|---|---|
| SSH works, but SCP fails | SCP server is not enabled |
| SCP authentication succeeds, but transfer fails | Missing AAA authorization or insufficient privileges |
| File transfer cannot start | Incorrect source or destination path |
| Transfer fails after SSH authentication | Client attempts SFTP instead of SCP |
| Transfer stops or fails | Storage capacity, connectivity, or file system problems |

Because SCP depends on SSH, troubleshoot the underlying SSH connection first.

For SCP-specific troubleshooting:

```text
R1# debug ip scp
```

Debugging can help identify SCP connection and file transfer problems.

Disable debugging after troubleshooting.

## Key Concepts

- SSH provides encrypted remote CLI access over TCP port 22 by default.
- SCP provides encrypted file transfers using SSH.
- Cisco IOS XE can operate as both a client and server for SSH and SCP.
- SSH provides server authentication, user authentication, confidentiality, and integrity.
- SSHv2 should be explicitly configured using `ip ssh version 2`.
- SSH clients and servers must negotiate mutually supported cryptographic algorithms.
- SSH server authentication uses cryptographic host keys.
- SSH user authentication can use local credentials, centralized AAA, or public keys.
- Generating an appropriate SSH host key enables the SSH server.
- `transport input ssh` permits incoming SSH connections on VTY lines.
- `ip ssh time-out` controls SSH authentication negotiation, not established EXEC session inactivity.
- SCP server functionality is enabled using `ip scp server enable`.
- SCP server operation requires appropriate AAA authentication and EXEC authorization.
- SCP server file operations require sufficient user privileges.
- SCP client functionality does not require enabling the SCP server.
- Modern OpenSSH SCP clients may require the `-O` option when connecting to IOS XE SCP servers that do not support SFTP.

## Further Reading

- [SSH Configuration and Verification](ssh.md)
- [SSH Authentication and Security Options](authentication.md)
- [SCP Configuration and File Transfers](scp.md)
- [Console and VTY](../console-vty/overview.md)
- [AAA](../../../device-security/aaa/overview.md)

### Cisco Documentation

- [Secure Shell Version 2, Cisco IOS XE](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/sec-crypto/ssh/secure-shell-configuration-guide/m-secure-shell-version-2.html)
- [Secure Copy, Cisco IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-usr-ssh-sec-copy-0.html)
- [SCP Interoperability with OpenSSH 9.0 and Later](https://www.cisco.com/c/en/us/support/docs/troubleshooting/220371-scp-from-clients-on-openssh9-0-to-ios-xe.html)
