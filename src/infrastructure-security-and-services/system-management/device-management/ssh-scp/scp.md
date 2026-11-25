# SCP Configuration and File Transfers

**Secure Copy Protocol (SCP)** provides secure file transfers using SSH.

SCP can be used to transfer files between Cisco IOS XE devices and other devices supporting SCP, including Linux and Windows workstations.

Common applications include:

- Backing up device configurations.
- Restoring device configurations.
- Transferring IOS XE software images.
- Transferring diagnostic files.
- Copying files between network devices.

Because SCP uses SSH, it provides authentication, confidentiality, and integrity protection.

SCP normally uses TCP port 22, the same port used for SSH CLI access.

## File Transfer Protocol Comparison

Cisco IOS XE supports several file transfer protocols.

| Protocol | Default Transport | Encryption | Authentication |
|---|---|---|---|
| TFTP | UDP/69 (initial request) | No | No |
| FTP | TCP/21 (control connection) | No | Yes |
| RCP | TCP/514 | No | Host/user-based trust |
| SCP | TCP/22 | Yes | SSH authentication |
| SFTP | TCP/22 | Yes | SSH authentication |

SCP and SFTP are separate file transfer protocols, although both use SSH.

Support for SFTP depends on the IOS XE platform and software release.

An SCP server does not necessarily support SFTP.

## SCP Client and Server

Cisco IOS XE can operate as both an SCP client and an SCP server.

- The **SCP client** initiates the connection and requests a file transfer.
- The **SCP server** accepts the connection and responds to the file transfer request.

The direction of connection establishment determines which device is the client and which device is the server.

The direction of the file transfer does not determine their roles.

### SCP Upload

In the following example, a workstation initiates an SCP connection to R1 and uploads a file.

```text
Workstation                         R1
 SCP Client                      SCP Server
     |                               |
     | ---- Connection Request ----> |
     |                               |
     | ========= File Data ========> |
     |                               |
```

The workstation is the SCP client because it initiates the connection.

R1 is the SCP server because it accepts the connection.

### SCP Download

In the following example, the workstation initiates an SCP connection to R1 and downloads a file.

```text
Workstation                         R1
 SCP Client                      SCP Server
     |                               |
     | ---- Connection Request ----> |
     |                               |
     | <======== File Data ========= |
     |                               |
```

The workstation is still the SCP client, even though R1 sends the file.

An SCP client can both upload and download files.

## SCP Operation

SCP uses SSH to establish an authenticated and encrypted connection.

A simplified SCP operation consists of the following stages:

1. The SCP client establishes a TCP connection to the server.
2. The SSH client and server negotiate cryptographic algorithms.
3. The client authenticates the server.
4. The server authenticates the user.
5. The server authorizes the requested SCP operation.
6. The SCP file transfer occurs through the secure SSH connection.
7. The connection terminates after the transfer completes.

Unlike an interactive SSH CLI connection, an SCP connection is used to transfer file data.

The user does not need to manually enter IOS XE CLI commands on the remote device.

SCP relies on the underlying SSH implementation for:

- Cryptographic key exchange.
- Server authentication.
- User authentication.
- Encryption.
- Data integrity protection.
- Session rekeying.

Changing the SSH server's supported cryptographic algorithms or authentication methods can therefore affect SCP operation.

## Cisco IOS File System

Cisco IOS XE provides access to files through the **Cisco IOS File System (IFS)**.

Different file systems are identified using prefixes.

Common examples include:

| File System | Purpose |
|---|---|
| `flash:` | Local flash storage |
| `bootflash:` | Local boot flash storage |
| `nvram:` | Nonvolatile configuration storage |
| `system:` | System-related virtual file system |
| `usbflash0:` | USB flash storage |
| `scp:` | Remote SCP file transfers |
| `ftp:` | Remote FTP file transfers |
| `tftp:` | Remote TFTP file transfers |

The available file systems depend on the IOS XE platform.

On some platforms, `flash:` and `bootflash:` refer to the same physical storage.

### Displaying Available File Systems

Use the following command to display the available file systems:

```text
R1# show file systems
```

The output includes information such as:

- File system prefixes.
- Total storage capacity.
- Available storage capacity.
- File system type.
- Read and write permissions.

### Displaying Files

To display files in local flash storage:

```text
R1# dir flash:
```

Alternatively:

```text
R1# dir bootflash:
```

To display files in a particular directory:

```text
R1# dir bootflash:/backups/
```

### Configuration File Locations

IOS XE provides virtual file system references for the running and startup configurations.

| File | Equivalent Reference |
|---|---|
| `running-config` | `system:running-config` |
| `startup-config` | `nvram:startup-config` |

For example, the following commands can be used to display the running configuration:

```text
R1# show running-config
```

```text
R1# more system:running-config
```

These file references are useful when copying configurations using SCP.

## The copy Command

IOS XE uses the `copy` EXEC command to transfer files.

The general syntax is:

```text
copy <source> <destination>
```

The first argument specifies the source file or file system.

The second argument specifies the destination file or file system.

For example:

```text
R1# copy running-config bootflash:R1-backup.cfg
```

This copies the current running configuration to a file named `R1-backup.cfg` in boot flash.

The same command structure is used for remote file transfers.

```text
R1# copy running-config scp:
```

This copies the running configuration to a remote SCP server.

```text
R1# copy scp: bootflash:
```

This copies a file from a remote SCP server to local boot flash.

The position of `scp:` determines the direction of the file transfer.

| Command | Transfer Direction |
|---|---|
| `copy scp: bootflash:` | Remote server to local storage |
| `copy bootflash: scp:` | Local storage to remote server |
| `copy running-config scp:` | Running configuration to remote server |
| `copy startup-config scp:` | Startup configuration to remote server |
| `copy scp: running-config` | Remote configuration file into the running configuration |
| `copy scp: startup-config` | Remote configuration file to the startup configuration |

In all of these examples, the local IOS XE device operates as the SCP client.

## SCP Client Configuration

SCP client functionality does not require enabling the local SCP server.

The following command is not required for outgoing SCP transfers:

```text
ip scp server enable
```

Similarly, the local device does not need to generate an SSH host key merely to initiate a password-authenticated SCP transfer.

The SCP client requires:

- IP connectivity to the remote SCP server.
- Compatible SSH cryptographic algorithms.
- Valid authentication credentials.
- Sufficient permissions on the remote SCP server.
- Access to the relevant local file system.

The remote device must support and permit incoming SCP transfers.

## Copying Files Using Interactive Prompts

Consider the following topology:

```text
        R1                      Linux Server
   192.0.2.1                    192.0.2.100
        G1---------------------------eth0

    SCP Client                   SCP Server
```

Assume the Linux server has:

- A functioning SSH server.
- A user account named `backup`.
- Support for SCP file transfers.
- Appropriate file system permissions.

### Copying a File to an SCP Server

First, create a local backup file on R1:

```text
R1# copy running-config bootflash:R1-backup.cfg
```

Copy the file to the remote SCP server:

```text
R1# copy bootflash:R1-backup.cfg scp:
```

IOS XE prompts for information about the destination.

For example:

```text
R1# copy bootflash:R1-backup.cfg scp:
Address or name of remote host []? 192.0.2.100
Destination username []? backup
Destination filename [R1-backup.cfg]? R1-backup.cfg
Password:
```

The resulting transfer is:

```text
R1                                   Linux Server
bootflash:R1-backup.cfg -----------> R1-backup.cfg
```

The password is the password of the `backup` account on the Linux server.

It is not a password configured on R1.

### Copying a File from an SCP Server

To retrieve the backup file from the remote SCP server:

```text
R1# copy scp: bootflash:
```

For example:

```text
R1# copy scp: bootflash:
Address or name of remote host []? 192.0.2.100
Source username []? backup
Source filename []? R1-backup.cfg
Destination filename [R1-backup.cfg]? restored-backup.cfg
Password:
```

The resulting transfer is:

```text
Linux Server                              R1
R1-backup.cfg ----------------------> bootflash:restored-backup.cfg
```

The destination filename does not need to match the source filename.

Verify the transferred file:

```text
R1# dir bootflash:restored-backup.cfg
```

## SCP URL Syntax

Instead of entering the remote hostname, username, and filename through interactive prompts, these parameters can be included directly in the `copy` command.

The general SCP URL syntax is:

```text
scp://<username>@<host>/<path>
```

For example:

```text
scp://backup@192.0.2.100/R1-backup.cfg
```

### Uploading a File

Copy a local file to a remote SCP server:

```text
R1# copy bootflash:R1-backup.cfg scp://backup@192.0.2.100/R1-backup.cfg
```

### Downloading a File

Copy a remote file to local boot flash:

```text
R1# copy scp://backup@192.0.2.100/R1-backup.cfg bootflash:R1-backup.cfg
```

The SCP URL specifies the remote file.

Its position determines whether the remote file is the source or destination.

### Specifying a Remote Directory

A path can be included in the SCP URL.

For example:

```text
R1# copy running-config scp://backup@192.0.2.100/backups/R1.cfg
```

The remote SCP server interprets the requested path.

Be aware of the distinction between relative and absolute paths.

For example, when using an OpenSSH server:

```text
scp://backup@192.0.2.100/backups/R1.cfg
```

A relative path can refer to a directory under the user's home directory.

An absolute path can be specified using an additional slash:

```text
scp://backup@192.0.2.100//home/backup/backups/R1.cfg
```

The exact URL parsing and path handling should be verified on the IOS XE release being used.

The destination directory must already exist, and the remote user must have the necessary permissions.

### Including Credentials in a URL

IOS XE supports specifying credentials in certain remote file transfer URLs.

However, embedding passwords directly in commands is not recommended.

Passwords could be exposed through:

- CLI command history.
- Terminal session logs.
- AAA command accounting.
- Configuration files.
- Screen captures.

Prefer specifying the username in the URL and entering the password when prompted.

## SCP Client Credentials

Some IOS XE releases support configuring default SCP client credentials.

For example:

```text
R1(config)# ip scp username backup
R1(config)# ip scp password 0 BACKUP-PASSWORD
```

These commands configure credentials for locally initiated SCP transfers.

They do not create a local user account.

They also do not configure authentication credentials for incoming SCP connections.

The distinction is important.

| Command | Purpose |
|---|---|
| `username backup privilege 15 secret ...` | Creates a local user account |
| `ip scp username ...` | Configures a default SCP client username |
| `ip scp password ...` | Configures a default SCP client password |

Support for these client credential commands depends on the IOS XE release and platform.

Storing SCP credentials in the device configuration introduces additional security considerations.

Remove the configured credentials when they are no longer required:

```text
R1(config)# no ip scp username
R1(config)# no ip scp password
```

## SCP Server Configuration

Unlike SCP client functionality, SCP server functionality must be explicitly enabled.

```text
R1(config)# ip scp server enable
```

This allows remote SCP clients to initiate file transfer requests to R1. It does not enable SSH, create user accounts, or grant privileges.

### SCP Server Requirements

The IOS XE SCP server requires:

- A functioning SSH server.
- User authentication.
- A user with privilege level 15.
- SCP server functionality enabled using `ip scp server enable`.

**Explicit AAA configuration is not required.** Traditional local authentication using `login local` is sufficient when the local user has privilege level 15.

### Basic SCP Server Configuration

Generate a named RSA key pair to enable the SSH server:

```text
R1# crypto key generate rsa label SSH-KEYS modulus 2048
```

Starting with IOS XE 17.11.1a, RSA key generation is performed in privileged EXEC mode. The explicit label eliminates the need to configure a hostname and domain name for key generation.

Configure SSHv2, a local privilege level 15 user, the VTY lines, and the SCP server:

```text
R1# configure terminal
R1(config)# ip ssh version 2
R1(config)# username BACKUP privilege 15 secret BACKUP-PASSWORD
!
R1(config)# line vty 0 15
R1(config-line)# login local
R1(config-line)# transport input ssh
R1(config-line)# exit
!
R1(config)# ip scp server enable
```

Assuming IP connectivity and compatible SSH algorithms, R1 can now accept SCP transfers from the BACKUP user.

Neither `aaa new-model` nor `aaa authorization exec` is required for this configuration.

## SCP Server Authentication and Authorization

Authentication verifies the identity of the user. Authorization determines the privileges and access available to that user. Successful SSH authentication alone does not guarantee successful SCP access.

### Without AAA

Traditional local authentication uses the local username database:

```text
R1(config)# username BACKUP privilege 15 secret BACKUP-PASSWORD
!
R1(config)# line vty 0 15
R1(config-line)# login local
R1(config-line)# transport input ssh
```

The local account supplies the required privilege level. No explicit AAA EXEC authorization method list is needed.

### With AAA

If AAA is enabled, configure appropriate login authentication and EXEC authorization. For example, to use the local database:

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login default local
R1(config)# aaa authorization exec default local
R1(config)# username BACKUP privilege 15 secret BACKUP-PASSWORD
!
R1(config)# line vty 0 15
R1(config-line)# login authentication default
R1(config-line)# transport input ssh
```

This is an alternative to traditional local authentication, not an additional prerequisite for SCP.

The login authentication method list verifies the user's credentials. The EXEC authorization method list provides the applicable authorization and privilege information. Configuring AAA login authentication alone does not configure AAA EXEC authorization.

Centralized AAA services such as TACACS+ or RADIUS can also be used to authenticate users and assign privilege level 15.

### Privilege Level Requirements

The IOS XE SCP server requires the authenticated user to have privilege level 15.

Consider the following two accounts:

```text
R1(config)# username USER1 privilege 1 secret USER1-PASSWORD
R1(config)# username USER2 privilege 15 secret USER2-PASSWORD
```

Assuming traditional local authentication (`login local`) is configured:

| User | Initial Privilege Level | SCP Server Access |
|---|---:|---|
| USER1 | 1 | Denied |
| USER2 | 15 | Permitted |

Both users can potentially authenticate successfully through SSH.

However, USER1 does not have the required privilege level for SCP server operations.

### Why the enable Secret Is Not Sufficient

Consider the following configuration:

```text
R1(config)# username BACKUP privilege 1 secret BACKUP-PASSWORD
R1(config)# enable secret ENABLE-PASSWORD
```

An administrator can establish an interactive SSH session using the `BACKUP` account and subsequently enter privileged EXEC mode using the `enable` command.

However, an incoming SCP connection does not provide a normal interactive CLI session in which the client can enter `enable`.

The required privilege level must be provided through the user's authentication and authorization configuration.

Configuring an enable secret is not a substitute for assigning privilege level 15 to an SCP user.

### The Default EXEC Authorization List

When using AAA for SCP server access, use the default EXEC authorization method list as the baseline:

```text
R1(config)# aaa authorization exec default local
```

SCP does not establish an ordinary interactive CLI session.

Do not assume that configuring only a named EXEC authorization method list under particular VTY lines provides equivalent authorization for every SCP implementation.

For example:

```text
R1(config)# aaa authorization exec ADMIN local
!
R1(config)# line vty 0 15
R1(config-line)# authorization exec ADMIN
```

This configures named EXEC authorization for the specified lines.

For AAA-based SCP configurations, Cisco documents the default EXEC authorization list. This list is not required when using traditional local authentication without AAA.

## SCP Server Using TACACS+

SCP authentication and authorization can also use centralized AAA services.

For example, assume a TACACS+ server is available at 192.0.2.200.

```text
Workstation
192.0.2.100
     |
     | SCP
     |
     v
     R1 ---------------- TACACS+ Server
192.0.2.1                  192.0.2.200
```

Configure the TACACS+ server:

```text
R1(config)# tacacs server TAC1
R1(config-server-tacacs)# address ipv4 192.0.2.200
R1(config-server-tacacs)# key TACACS-KEY
```

Configure a TACACS+ server group:

```text
R1(config)# aaa group server tacacs+ TACACS-SERVERS
R1(config-sg-tacacs+)# server name TAC1
```

Configure authentication and EXEC authorization:

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login default group TACACS-SERVERS local
R1(config)# aaa authorization exec default group TACACS-SERVERS local
```

Configure a local fallback user:

```text
R1(config)# username BACKUP privilege 15 secret BACKUP-PASSWORD
```

Enable the SCP server:

```text
R1(config)# ip scp server enable
```

The TACACS+ server must authenticate the SCP user and authorize the user with the required privilege level.

For example, TACACS+ shell authorization can return the following attribute:

```text
priv-lvl=15
```

A successful TACACS+ authentication result without appropriate authorization can still result in SCP failure.

### AAA Fallback Behavior

Consider the following configuration:

```text
aaa authentication login default group TACACS-SERVERS local
aaa authorization exec default group TACACS-SERVERS local
```

The local database provides a fallback mechanism when the TACACS+ method returns an error.

A definitive TACACS+ rejection does not cause IOS XE to try the next method simply because the requested access was denied.

Authentication and authorization method lists are processed independently.

For more information, see [AAA Server Failover and Local Fallback](../../../device-security/aaa/failover.md).

## SCP Server Using RADIUS

RADIUS can also provide centralized authentication and EXEC authorization.

For example, assuming an appropriately configured RADIUS server group:

```text
R1(config)# aaa authentication login default group RADIUS-SERVERS local
R1(config)# aaa authorization exec default group RADIUS-SERVERS local
```

The RADIUS server must provide appropriate authorization attributes for the SCP user.

Cisco privilege levels can be assigned using Cisco vendor-specific attributes, such as:

```text
Cisco-AVPair = "shell:priv-lvl=15"
```

For more information, see [RADIUS](../../../device-security/aaa/radius.md).

## SCP and Public-Key Authentication

Because SCP uses SSH, SCP connections can use SSH public-key authentication on implementations supporting it.

For example, configure the user's public key on R1:

```text
R1(config)# ip ssh pubkey-chain
R1(conf-ssh-pubkey)# username BACKUP
R1(conf-ssh-pubkey-user)# key-string
R1(conf-ssh-pubkey-data)# <public-key-data>
```

The user still requires appropriate authorization for SCP file operations.

Public-key authentication does not bypass the privilege level requirement.

From an OpenSSH workstation, specify the private key using the `-i` option:

```text
scp -O -i ~/.ssh/r1_backup backup.cfg BACKUP@192.0.2.1:bootflash:backup.cfg
```

The client proves possession of the private key during SSH authentication.

The private key is not transmitted to R1.

For more information, see [SSH Authentication and Security Options](authentication.md).

## SCP File Transfers from a Workstation

An administrator can use an external SCP client to transfer files to and from a Cisco IOS XE device.

Common SCP clients include:

- OpenSSH SCP.
- PuTTY PSCP.
- WinSCP.

When using a client that supports both SCP and SFTP, ensure that the selected file transfer protocol is supported by the IOS XE device.

### OpenSSH SCP Syntax

The general syntax for copying a local file to a remote server is:

```text
scp <local-file> <username>@<host>:<remote-file>
```

The general syntax for copying a remote file to the local device is:

```text
scp <username>@<host>:<remote-file> <local-file>
```

Unlike IOS XE SCP URLs, OpenSSH identifies a remote file using the following syntax:

```text
username@host:path
```

## OpenSSH 9.0 and SFTP

Starting with OpenSSH 9.0, the `scp` command uses SFTP rather than the traditional SCP protocol by default.

This is an important distinction.

The name of the client command does not necessarily identify the file transfer protocol being used.

For example:

```text
scp backup.cfg BACKUP@192.0.2.1:backup.cfg
```

With OpenSSH 9.0 or later, this command attempts an SFTP transfer by default.

If the IOS XE device supports SCP but does not support SFTP, the transfer fails.

SSH authentication may succeed before the connection is terminated.

### Forcing the SCP Protocol

The uppercase `-O` option forces OpenSSH to use the traditional SCP protocol.

```text
scp -O backup.cfg BACKUP@192.0.2.1:backup.cfg
```

This is useful when communicating with IOS XE devices configured as SCP servers.

| Command | OpenSSH 9.0 and Later |
|---|---|
| `scp` | Uses SFTP by default |
| `scp -O` | Uses the traditional SCP protocol |

The `-O` option does not disable SSH encryption.

It changes the file transfer protocol used within the SSH connection.

### Verifying the OpenSSH Version

```text
ssh -V
```

For example:

```text
OpenSSH_9.6p1
```

If using OpenSSH 9.0 or later with an SCP-only IOS XE server, specify `-O`.

## Uploading Files to an IOS XE SCP Server

Assume R1 has the following configuration:

- IP address: 192.0.2.1.
- SCP username: BACKUP.
- SCP server enabled.
- Appropriate SSH authentication and authorization.

The workstation has a file named `test.txt`.

### Uploading to the Default File System

```text
scp -O test.txt BACKUP@192.0.2.1:test.txt
```

The file is copied to the remote device's default file system.

On many IOS XE routers, this is boot flash.

Verify the file on R1:

```text
R1# dir bootflash:test.txt
```

### Explicitly Specifying a File System

To explicitly specify boot flash:

```text
scp -O test.txt BACKUP@192.0.2.1:bootflash:test.txt
```

To specify flash storage:

```text
scp -O test.txt BACKUP@192.0.2.1:flash:test.txt
```

The specified file system must exist on the remote IOS XE device.

### Uploading to a Different Filename

The remote filename does not need to match the local filename.

```text
scp -O test.txt BACKUP@192.0.2.1:bootflash:received.txt
```

The local file `test.txt` is stored on R1 as `received.txt`.

### Uploading to a Directory

Assume the following directory exists on R1:

```text
bootflash:/backups/
```

Upload a file to that directory:

```text
scp -O test.txt BACKUP@192.0.2.1:bootflash:/backups/test.txt
```

The destination directory must exist before the transfer.

To create the directory on R1:

```text
R1# mkdir bootflash:/backups
```

## Downloading Files from an IOS XE SCP Server

Copy a file from R1 to the workstation:

```text
scp -O BACKUP@192.0.2.1:bootflash:test.txt test.txt
```

The remote file is the first argument because it is the source.

The local file is the second argument because it is the destination.

To save the file under a different local filename:

```text
scp -O BACKUP@192.0.2.1:bootflash:test.txt R1-test.txt
```

To download the file to the current working directory using the remote filename:

```text
scp -O BACKUP@192.0.2.1:bootflash:test.txt .
```

The period (`.`) represents the current working directory.

## SCP and PuTTY PSCP

PuTTY provides a command-line file transfer utility named PSCP.

The `-scp` option explicitly selects the SCP protocol.

For example, upload a file:

```text
pscp -scp test.txt BACKUP@192.0.2.1:bootflash:test.txt
```

Download a file:

```text
pscp -scp BACKUP@192.0.2.1:bootflash:test.txt test.txt
```

The OpenSSH `-O` option and PuTTY PSCP `-scp` option serve similar purposes.

They explicitly select SCP rather than SFTP.

## Copying Files Between IOS XE Devices

An IOS XE device can initiate SCP transfers to another IOS XE device.

This does not require an external workstation.

Consider the following topology:

```text
        R1                         R2
   192.0.2.1/24               192.0.2.2/24
        G1-------------------------G1

    SCP Client                  SCP Server
```

R2 is configured as an SCP server.

R1 initiates file transfers.

### R2 Configuration

Generate an RSA key pair:

```text
R2# crypto key generate rsa label SSH-KEYS modulus 2048
```

Configure SSH and SCP:

```text
R2# configure terminal
R2(config)# ip ssh version 2
!
R2(config)# username BACKUP privilege 15 secret BACKUP-PASSWORD
!
R2(config)# line vty 0 15
R2(config-line)# login local
R2(config-line)# transport input ssh
!
R2(config)# ip scp server enable
```

### Uploading a File from R1 to R2

Create a local backup file on R1:

```text
R1# copy running-config bootflash:R1-backup.cfg
```

Transfer the file to R2:

```text
R1# copy bootflash:R1-backup.cfg scp://BACKUP@192.0.2.2/bootflash:R1-backup.cfg
```

Verify the received file on R2:

```text
R2# dir bootflash:R1-backup.cfg
```

### Downloading a File from R2 to R1

Retrieve the file:

```text
R1# copy scp://BACKUP@192.0.2.2/bootflash:R1-backup.cfg bootflash:retrieved.cfg
```

Verify the received file on R1:

```text
R1# dir bootflash:retrieved.cfg
```

R1 operates as the SCP client in both examples.

R2 operates as the SCP server in both examples.

Only R2 requires the `ip scp server enable` command.

## Backing Up Configuration Files

SCP can be used to back up IOS XE configurations to remote servers.

### Backing Up the Running Configuration

```text
R1# copy running-config scp:
```

Alternatively, specify the remote server and filename:

```text
R1# copy running-config scp://backup@192.0.2.100/R1-running.cfg
```

The running configuration is transferred without requiring an intermediate file in local flash storage.

### Backing Up the Startup Configuration

```text
R1# copy startup-config scp:
```

Alternatively:

```text
R1# copy startup-config scp://backup@192.0.2.100/R1-startup.cfg
```

The startup configuration is transferred to the remote SCP server.

### Running vs. Startup Configuration Backups

| Source | Contents |
|---|---|
| `running-config` | The currently active configuration |
| `startup-config` | The configuration saved for use during startup |

The two configurations are not necessarily identical.

If configuration changes have been made without saving them, the running configuration contains changes that are absent from the startup configuration.

SCP does not automatically synchronize the running and startup configurations.

### Downloading Configuration Files from a Workstation

Instead of having R1 initiate the transfer, a workstation can initiate an SCP download from R1.

For example:

```text
scp -O BACKUP@192.0.2.1:system:running-config R1-running.cfg
```

To retrieve the startup configuration:

```text
scp -O BACKUP@192.0.2.1:nvram:startup-config R1-startup.cfg
```

Support for directly accessing these virtual file system entries through an external SCP client depends on the IOS XE implementation.

If direct access is not supported, first copy the configuration to a regular file on R1:

```text
R1# copy running-config bootflash:R1-running.cfg
R1# copy startup-config bootflash:R1-startup.cfg
```

Then download the files:

```text
scp -O BACKUP@192.0.2.1:bootflash:R1-running.cfg R1-running.cfg
```

```text
scp -O BACKUP@192.0.2.1:bootflash:R1-startup.cfg R1-startup.cfg
```

Configuration backups can contain sensitive information, including credentials, password hashes, authentication keys, and internal network addressing.

They must be protected appropriately after the transfer completes.

SCP protects files in transit, but it does not automatically encrypt files stored on the destination device.

## Restoring Configuration Files

SCP can retrieve configuration files for restoration.

However, copying a configuration file into the running configuration has different effects from replacing the running configuration.

### Copying to the Running Configuration

```text
R1# copy scp://backup@192.0.2.100/R1-backup.cfg running-config
```

IOS XE processes the retrieved configuration and merges it into the current running configuration.

Existing configuration commands that are absent from the retrieved file are not necessarily removed.

For example, assume the current running configuration contains:

```text
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
!
interface Loopback1
 ip address 11.11.11.11 255.255.255.255
```

The remote backup contains:

```text
interface Loopback0
 ip address 10.10.10.10 255.255.255.255
```

After copying the remote file into the running configuration:

- Loopback0's IP address changes to 10.10.10.10.
- Loopback1 remains configured.

The absence of Loopback1 from the backup does not automatically delete it.

### Copying to the Startup Configuration

```text
R1# copy scp://backup@192.0.2.100/R1-backup.cfg startup-config
```

This replaces the startup configuration with the retrieved configuration file.

It does not immediately replace the running configuration.

The startup configuration is normally applied during the next reload, assuming the device is configured to load it.

### Replacing the Running Configuration

To replace the running configuration rather than merge commands into it, use `configure replace`.

First, retrieve the backup file:

```text
R1# copy scp://backup@192.0.2.100/R1-backup.cfg bootflash:R1-backup.cfg
```

Then replace the running configuration:

```text
R1# configure replace bootflash:R1-backup.cfg force
```

The `configure replace` command attempts to make the running configuration match the specified replacement configuration, including removing configuration commands that should no longer be present.

The replacement file should contain a complete and valid IOS XE configuration.

The `force` keyword suppresses the confirmation prompt.

Care must be taken when replacing configurations remotely, because the replacement configuration might remove management connectivity or authentication settings.

### Configuration Restoration Comparison

| Command | Effect |
|---|---|
| `copy scp: running-config` | Merges the retrieved configuration into the running configuration |
| `copy scp: startup-config` | Replaces the startup configuration |
| `copy scp: bootflash:` | Stores the retrieved file without applying it |
| `configure replace bootflash:<file>` | Attempts to replace the running configuration |

SCP determines how the file is transferred.

The destination or subsequent configuration command determines how the file is used.

## Transferring IOS XE Software Images

SCP can securely transfer IOS XE software images to a network device.

For example:

```text
R1# copy scp://backup@192.0.2.100/iosxe-image.bin bootflash:iosxe-image.bin
```

Before initiating the transfer, verify that sufficient storage is available:

```text
R1# dir bootflash:
```

Alternatively:

```text
R1# show file systems
```

After the transfer, verify that the file exists and has the expected size:

```text
R1# dir bootflash:iosxe-image.bin
```

### File Integrity Verification

SCP protects data integrity during the SSH transfer.

However, administrators should also verify that the received file matches the expected software image.

For example:

```text
R1# verify /md5 bootflash:iosxe-image.bin
```

On releases supporting SHA-512 verification:

```text
R1# verify /sha512 bootflash:iosxe-image.bin
```

Compare the calculated hash against the expected hash obtained from a trusted source, such as Cisco's software download portal.

An authenticated SSH transfer does not prove that the source file is the intended Cisco software image.

### SCP Does Not Install Software

Copying an IOS XE software image to boot flash does not automatically install or activate it.

For example:

```text
R1# copy scp: bootflash:
```

This only transfers the file.

Additional installation or boot configuration commands are required to use the software image.

The required procedure depends on the platform and installation mode.

## SCP Source Interface

SCP uses the SSH client infrastructure for outgoing connections.

The `ip ssh source-interface` command can therefore control the source interface used by locally initiated SCP transfers.

Consider the following topology:

```text
    Loopback0
    1.1.1.1/32
         |
         R1 ------------------ SCP Server
    192.0.2.1                  192.0.2.100
         G1
```

By default, a locally initiated SCP connection normally uses the outgoing interface's IP address as its source address.

For example:

```text
Source IP:      192.0.2.1
Destination IP: 192.0.2.100
```

Configure R1 to use Loopback0 as the SSH source interface:

```text
R1(config)# ip ssh source-interface Loopback0
```

Initiate an SCP transfer:

```text
R1# copy running-config scp://backup@192.0.2.100/R1.cfg
```

The resulting connection uses:

```text
Source IP:      1.1.1.1
Destination IP: 192.0.2.100
```

The SCP server must have a return route to the configured source address.

### Shared SSH and SCP Source Interface

The same command controls the source interface for both locally initiated SSH and SCP connections.

```text
R1(config)# ip ssh source-interface Loopback0
```

This is different from protocols with their own source-interface configuration commands, such as:

| Protocol | Source Interface Command |
|---|---|
| SSH | `ip ssh source-interface` |
| SCP | `ip ssh source-interface` |
| TFTP | `ip tftp source-interface` |
| FTP | `ip ftp source-interface` |

The SSH source-interface command does not restrict incoming SSH or SCP connections to the specified interface.

It controls the source address used for locally initiated connections.

## VRF-Aware SCP

SCP supports VRF-aware file transfers.

This is particularly useful when a dedicated management VRF provides connectivity to a backup server.

Consider the following topology:

```text
         R1                         Backup Server
    VRF MANAGEMENT
    192.0.2.1 ------------------------ 192.0.2.100
         G1
```

GigabitEthernet1 belongs to the `MANAGEMENT` VRF.

The backup server is reachable through the `MANAGEMENT` routing table.

### Configuring the Management VRF

```text
R1(config)# vrf definition MANAGEMENT
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# exit-address-family
!
R1(config)# interface GigabitEthernet1
R1(config-if)# vrf forwarding MANAGEMENT
R1(config-if)# ip address 192.0.2.1 255.255.255.0
R1(config-if)# no shutdown
```

Verify connectivity:

```text
R1# ping vrf MANAGEMENT 192.0.2.100
```

Verify the route to the server:

```text
R1# show ip route vrf MANAGEMENT 192.0.2.100
```

### Specifying the VRF in the copy Command

On supported IOS XE releases, the `vrf` keyword can be specified at the end of the `copy` command.

For example:

```text
R1# copy scp: bootflash: vrf MANAGEMENT
```

The SCP connection uses the `MANAGEMENT` VRF.

IOS XE prompts for the remote server and file information.

For example:

```text
R1# copy scp: bootflash: vrf MANAGEMENT
Address or name of remote host []? 192.0.2.100
Source username []? backup
Source filename []? test.txt
Destination filename [test.txt]?
Password:
```

The `vrf` keyword specifies the routing instance used by the file transfer.

It does not change the VRF membership of any interface.

### Using the SSH Source Interface

Another method is to configure the SSH source interface as an interface belonging to the management VRF.

```text
R1(config)# ip ssh source-interface GigabitEthernet1
```

On implementations supporting VRF-aware SSH and SCP through source-interface selection, the configured interface also determines the routing context used by the connection.

The explicit `vrf` option in the `copy` command provides per-transfer VRF selection and avoids changing the global SSH source-interface configuration.

When combining these features, ensure that the selected source interface is compatible with the intended VRF.

### Incoming SCP Connections Through a VRF

The SCP server can accept connections through VRF interfaces on supported IOS XE platforms.

A separate SCP server does not need to be enabled for each VRF.

If VTY access restrictions are configured, verify whether VRF-aware access-class configuration is required.

For example:

```text
R1(config)# ip access-list standard MGMT-IN
R1(config-std-nacl)# permit 192.0.2.0 0.0.0.255
!
R1(config)# line vty 0 15
R1(config-line)# access-class MGMT-IN in vrf-also
```

The `vrf-also` keyword permits the VTY access class to evaluate incoming connections from non-default VRFs.

For more information, see [Session Timeouts and Access Restrictions](../console-vty/access-control.md).

## SCP Access Restrictions

An SCP server provides powerful file access capabilities.

An authorized SCP user with privilege level 15 can potentially access many files stored on the device.

Therefore, SCP server access should be restricted to trusted management systems.

### Restricting Incoming SSH and SCP Traffic

Because SCP uses SSH, network-level filtering can restrict access to the SSH transport.

For example:

```text
R1(config)# ip access-list extended MANAGEMENT-IN
R1(config-ext-nacl)# permit tcp host 192.0.2.100 host 192.0.2.1 eq 22
R1(config-ext-nacl)# deny tcp any host 192.0.2.1 eq 22
R1(config-ext-nacl)# permit ip any any
!
R1(config)# interface GigabitEthernet1
R1(config-if)# ip access-group MANAGEMENT-IN in
```

This ACL permits connections to TCP port 22 on 192.0.2.1 from 192.0.2.100.

It denies other sources from connecting to that address and port through GigabitEthernet1.

Other IP traffic is permitted.

The ACL must be applied wherever the relevant traffic can enter the device.

Other local IP addresses and custom SSH ports must also be considered if they are reachable.

### Distinguishing SSH and SCP with an ACL

Ordinary packet-filtering ACLs cannot distinguish interactive SSH sessions from SCP sessions based solely on TCP port numbers.

Both normally use TCP port 22.

```text
Interactive SSH ---- TCP/22 ----> R1
SCP ---------------- TCP/22 ----> R1
```

The distinction between an interactive CLI request and an SCP file transfer request is made within the SSH connection.

The application-specific information is protected by SSH encryption.

To allow ordinary SSH CLI access but prevent incoming SCP transfers, disable the SCP server rather than blocking TCP port 22.

```text
R1(config)# no ip scp server enable
```

This disables SCP server functionality without disabling ordinary SSH CLI access.

Outgoing SCP client functionality remains available.

## SSH Bulk Mode

SCP performance can be limited by the amount of data that can be transmitted before additional window space becomes available.

IOS XE supports **SSH bulk data transfer mode** to improve the performance of large SSH-based file transfers.

The feature can improve SCP performance when IOS XE operates as either an SCP client or server.

It is particularly useful when transferring large files, such as IOS XE software images.

### SSH Bulk Mode Configuration

The `ip ssh bulk-mode` command enables and configures SSH bulk-mode operation.

For example, to configure a bulk-mode window size of 2 MiB:

```text
R1(config)# ip ssh bulk-mode 2097152
```

The window size argument is expressed in bytes.

Relevant IOS XE release changes include:

| Release | Change |
|---|---|
| 17.2.1 | SSH bulk-mode performance enhancements introduced |
| 17.6.1 | Configurable bulk-mode window size introduced |
| 17.10.1 | Bulk mode enabled by default with a 128-KB window |

Feature availability depends on the platform.

### Bandwidth-Delay Product

The appropriate window size depends partly on the bandwidth-delay product (BDP) of the network.

The bandwidth-delay product represents the amount of data required to fill a network path.

```text
BDP (bits) = Bandwidth (bps) x RTT (seconds)
```

To calculate the result in bytes:

```text
BDP (bytes) = Bandwidth (bps) x RTT (seconds) / 8
```

For example, assume:

- Available bandwidth: 100 Mbps.
- Round-trip time: 100 milliseconds.

```text
BDP = 100,000,000 x 0.1 / 8
    = 1,250,000 bytes
```

Approximately 1.25 MB of data must be in flight to fully utilize the available bandwidth under ideal conditions.

A relatively small SSH channel window can restrict throughput on such a path.

### Window Size and Throughput

A simplified theoretical window-limited throughput calculation is:

```text
Throughput (bps) = Window size (bytes) x 8 / RTT (seconds)
```

For a 128-KiB window and an RTT of 100 milliseconds:

```text
Throughput = 131,072 x 8 / 0.1
           = 10,485,760 bps
           = Approximately 10.5 Mbps
```

Increasing the window size can allow more data to be transmitted before window limitations become a bottleneck.

Actual SCP throughput also depends on:

- TCP flow control and congestion control.
- SSH channel window behavior.
- Available network bandwidth.
- Packet loss.
- CPU utilization.
- Encryption performance.
- Storage read and write speeds.
- The remote SSH implementation.

Increasing the bulk-mode window size does not guarantee a proportional throughput increase.

Larger windows can also increase resource utilization.

## SCP Verification

### Verifying SSH Server Status

Use `show ip ssh` to verify the underlying SSH server.

```text
R1# show ip ssh
```

Verify that:

- SSH is enabled.
- The expected SSH version is supported.
- Compatible encryption and authentication algorithms are configured.
- Authentication timeouts and retries are appropriate.

A functioning SSH server is a prerequisite for incoming SCP transfers.

However, `show ip ssh` alone does not prove that SCP server functionality is enabled.

### Verifying SCP Server Configuration

Display the SCP-related configuration:

```text
R1# show running-config | include ^ip scp
```

For example:

```text
ip scp server enable
```

The presence of this command indicates that SCP server functionality is enabled.

Configured SCP client credentials may also appear in this output, so care should be taken when sharing it.

### Verifying Authentication Configuration

For traditional local authentication, verify the VTY lines:

```text
R1# show running-config | section line vty
```

Verify that `login local` and `transport input ssh` are configured and that the local user has privilege level 15.

### Verifying AAA Configuration

If AAA is enabled, display AAA authentication and authorization configuration:

```text
R1# show running-config | include ^aaa
```

For example:

```text
aaa new-model
aaa authentication login default local
aaa authorization exec default local
```

When using AAA, verify that the intended login authentication and EXEC authorization methods are configured. Without AAA, these commands are not required.

### Verifying User Privileges

Display the local user configuration:

```text
R1# show running-config | include ^username
```

For a local SCP user, verify that privilege level 15 is configured, whether using traditional local authentication or local AAA.

Alternatively, establish an ordinary SSH session using the account and display the current privilege level:

```text
R1# show privilege
Current privilege level is 15
```

A successful interactive SSH login at privilege level 15 is a useful diagnostic test, although it does not by itself prove that the complete SCP authorization path is functioning.

### Verifying Active Connections

Use the following command while a transfer is in progress:

```text
R1# show ssh
```

Because SCP uses SSH, active SCP connections can appear in SSH connection information.

TCP connection information can also be displayed:

```text
R1# show tcp brief all
```

A TCP connection to port 22 indicates that the underlying transport connection has been established.

It does not prove that a file transfer has successfully completed.

### Verifying Transferred Files

After transferring a file to the device, verify its existence and size:

```text
R1# dir bootflash:backup.cfg
```

For a text file, its contents can be displayed using:

```text
R1# more bootflash:backup.cfg
```

For binary files, use an appropriate hash verification command.

```text
R1# verify /md5 bootflash:iosxe-image.bin
```

Successful authentication, successful file transfer, and successful file integrity verification are distinct results.

## SCP Debugging

The `debug ip scp` command provides diagnostic information about incoming SCP operations.

```text
R1# debug ip scp
```

For example:

```text
Incoming SCP debugging is on
```

SCP debugging can help identify:

- SCP authorization failures.
- Insufficient user privileges.
- File transfer requests.
- File system errors.
- SCP protocol errors.

Because SCP relies on SSH, SSH debugging is also useful:

```text
R1# debug ip ssh
```

If AAA is involved:

```text
R1# debug aaa authentication
R1# debug aaa authorization
```

These commands help distinguish authentication failures from authorization failures.

Disable debugging when troubleshooting is complete:

```text
R1# undebug all
```

### OpenSSH SCP Verbose Output

When initiating an SCP transfer from a workstation, the `-v` option enables verbose output.

For example:

```text
scp -v -O test.txt BACKUP@192.0.2.1:bootflash:test.txt
```

Verbose output can help identify:

- SSH negotiation failures.
- Server host key verification failures.
- Authentication failures.
- The selected file transfer protocol.
- Remote file path errors.
- Unexpected connection termination.

For additional debugging information:

```text
scp -vvv -O test.txt BACKUP@192.0.2.1:bootflash:test.txt
```

## SCP Troubleshooting

SCP troubleshooting should distinguish between the following components:

```text
IP Connectivity
       |
       v
TCP Connectivity
       |
       v
SSH Negotiation
       |
       v
SSH User Authentication
       |
       v
SCP Authorization
       |
       v
SCP Protocol Operation
       |
       v
File System Access
       |
       v
Successful File Transfer
```

A failure can occur at any stage.

### Scenario 1: SSH Works but SCP Does Not

The administrator can establish an ordinary SSH CLI session, but SCP transfers fail.

Check whether SCP server functionality is enabled:

```text
R1# show running-config | include ^ip scp
```

If SCP server functionality is disabled:

```text
R1(config)# ip scp server enable
```

Remember that enabling SSH does not automatically enable SCP server functionality.

If the SCP server is already enabled, verify user privileges, the client's selected file transfer protocol, and AAA authorization if AAA is enabled.

### Scenario 2: Authentication Succeeds but SCP Immediately Fails

The user successfully authenticates, but the SCP connection closes.

Possible causes include:

- Missing or incorrect AAA EXEC authorization (if AAA is enabled).
- Insufficient user privilege level.
- Centralized AAA authorization rejection (if centralized AAA is used).
- Unsupported file transfer protocol.

For traditional local authentication, verify the following configuration:

```text
username BACKUP privilege 15 secret BACKUP-PASSWORD
!
line vty 0 15
 login local
 transport input ssh
```

If local AAA is enabled instead, verify the following configuration:

```text
aaa authentication login default local
aaa authorization exec default local
username BACKUP privilege 15 secret BACKUP-PASSWORD
```

If AAA is enabled, use AAA authorization debugging if necessary:

```text
R1# debug aaa authorization
```

### Scenario 3: An Ordinary User Cannot Transfer Files

Assume the following configuration:

```text
R1(config)# username BACKUP privilege 1 secret BACKUP-PASSWORD
```

The user can authenticate through SSH but cannot perform incoming SCP file transfers.

The user does not have the required privilege level.

Correct the user configuration:

```text
R1(config)# username BACKUP privilege 15 secret BACKUP-PASSWORD
```

Only grant privilege level 15 to users who are authorized for administrative access.

### Scenario 4: SCP Fails After Upgrading OpenSSH

The workstation was previously able to transfer files using:

```text
scp test.txt BACKUP@192.0.2.1:test.txt
```

After upgrading OpenSSH, the client displays:

```text
scp: Connection closed
```

SSH authentication succeeds, but the file is not transferred.

Use verbose output:

```text
scp -v test.txt BACKUP@192.0.2.1:test.txt
```

The output might include:

```text
debug1: Sending subsystem: sftp
```

The client is attempting an SFTP transfer.

The IOS XE device is configured as an SCP server but does not support the requested SFTP operation.

Force the traditional SCP protocol:

```text
scp -O test.txt BACKUP@192.0.2.1:test.txt
```

If `debug ip scp` produces no output despite successful SSH authentication, this can also indicate that the client is requesting SFTP rather than SCP.

### Scenario 5: Outgoing SCP Fails Despite Enabling the SCP Server

Assume R1 initiates the following transfer:

```text
R1# copy running-config scp://backup@192.0.2.100/R1.cfg
```

The transfer fails.

The administrator configures:

```text
R1(config)# ip scp server enable
```

However, the transfer still fails.

R1 is the SCP client in this example.

Enabling R1's SCP server does not correct a problem with its outgoing SCP connection.

Investigate:

- IP connectivity to the remote server.
- TCP port 22 connectivity.
- Remote SCP server functionality.
- Remote user credentials and permissions.
- SSH algorithm compatibility.
- Source interface configuration.
- VRF selection.

### Scenario 6: SCP Through a Management VRF Fails

The remote backup server is reachable through a management VRF.

An ordinary ping fails:

```text
R1# ping 192.0.2.100
```

A VRF-aware ping succeeds:

```text
R1# ping vrf MANAGEMENT 192.0.2.100
```

The SCP client must use the correct routing instance.

On supported releases:

```text
R1# copy scp: bootflash: vrf MANAGEMENT
```

Also verify the SSH source-interface configuration:

```text
R1# show running-config | include ^ip ssh source-interface
```

The configured source interface must be compatible with the selected routing instance.

### Scenario 7: File Not Found

The SCP client authenticates successfully, but the server reports that the file cannot be found.

Possible causes include:

- Incorrect filename.
- Incorrect capitalization.
- Incorrect file system prefix.
- Incorrect directory path.
- Confusion between relative and absolute paths.

Verify the file location on the remote device.

For example:

```text
R2# dir bootflash:
```

Then specify the correct remote file reference:

```text
R1# copy scp://BACKUP@192.0.2.2/bootflash:test.txt bootflash:test.txt
```

### Scenario 8: Insufficient Storage

An IOS XE image transfer fails because the destination device does not have enough available storage.

Verify available storage:

```text
R1# dir bootflash:
```

Alternatively:

```text
R1# show file systems
```

Ensure sufficient storage exists before initiating the transfer.

A failed or interrupted transfer can leave an incomplete destination file.

Verify the file size and integrity before using the received file.

### Scenario 9: SCP Transfer Is Unexpectedly Slow

A large file transfers successfully but at a much lower rate than expected.

Possible causes include:

- High round-trip time.
- Packet loss.
- TCP or SSH channel window limitations.
- CPU limitations.
- Encryption processing overhead.
- Storage performance.
- Network congestion.
- Control plane policing.

Verify the SSH bulk-mode configuration on supported releases.

For example:

```text
R1(config)# ip ssh bulk-mode 2097152
```

Select an appropriate window size based on the network bandwidth-delay product and available device resources.

### Scenario 10: SCP Fails After Restricting SSH Algorithms

The administrator changes the SSH server's permitted algorithms.

Ordinary SSH connections from certain clients and SCP transfers from other clients begin failing.

SCP uses the same underlying SSH cryptographic mechanisms.

For example, the following command restricts the SSH server's key exchange algorithms:

```text
R1(config)# ip ssh server algorithm kex ecdh-sha2-nistp256
```

An SCP client that does not support the configured algorithm cannot establish the underlying SSH connection.

Verify the server configuration:

```text
R1# show ip ssh
```

Use verbose output on the SCP client to identify the incompatible algorithm category.

## Configuration Command Summary

| Command | Mode | Purpose |
|---|---|---|
| `crypto key generate rsa label SSH-KEYS modulus 2048` | Privileged EXEC | Generates an RSA host key for SSH |
| `ip ssh version 2` | Global configuration | Configures SSHv2 |
| `ip scp server enable` | Global configuration | Enables incoming SCP transfers |
| `no ip scp server enable` | Global configuration | Disables SCP server functionality |
| `aaa new-model` | Global configuration | Enables AAA (optional for SCP) |
| `aaa authentication login default local` | Global configuration | Configures login authentication when using local AAA |
| `aaa authorization exec default local` | Global configuration | Configures EXEC authorization when using local AAA |
| `username BACKUP privilege 15 secret ...` | Global configuration | Creates a local privilege 15 user |
| `ip scp username BACKUP` | Global configuration | Configures a default SCP client username on supported releases |
| `ip scp password ...` | Global configuration | Configures a default SCP client password on supported releases |
| `ip ssh source-interface Loopback0` | Global configuration | Specifies the source interface for outgoing SSH and SCP connections |
| `ip ssh bulk-mode 2097152` | Global configuration | Configures the SSH bulk-mode window size |
| `transport input ssh` | Line configuration | Permits incoming SSH connections |
| `login local` | Line configuration | Configures traditional local user authentication without AAA |
| `copy scp: bootflash:` | EXEC | Downloads a file from an SCP server |
| `copy bootflash:<file> scp:` | EXEC | Uploads a file to an SCP server |
| `copy running-config scp:` | EXEC | Backs up the running configuration to an SCP server |
| `copy startup-config scp:` | EXEC | Backs up the startup configuration to an SCP server |
| `copy scp: running-config` | EXEC | Retrieves and merges a remote configuration |
| `copy scp: startup-config` | EXEC | Retrieves and replaces the startup configuration |
| `copy scp: bootflash: vrf MANAGEMENT` | EXEC | Downloads a file using a specified VRF |
| `configure replace bootflash:<file>` | Privileged EXEC | Attempts to replace the running configuration using a local file |

## Verification Command Summary

| Command | Purpose |
|---|---|
| `show ip ssh` | Displays SSH server status and configuration |
| `show ssh` | Displays active SSH connections |
| `show running-config \| include ^ip scp` | Displays explicitly configured SCP commands |
| `show running-config \| include ^aaa` | Displays AAA configuration |
| `show privilege` | Displays the current EXEC privilege level |
| `show file systems` | Displays available file systems and storage capacity |
| `dir bootflash:` | Displays files and storage information |
| `more bootflash:<file>` | Displays the contents of a text file |
| `verify /md5 bootflash:<file>` | Calculates or verifies a file's MD5 hash |
| `verify /sha512 bootflash:<file>` | Calculates or verifies a SHA-512 hash on supported releases |
| `show tcp brief all` | Displays TCP connection information |
| `debug ip scp` | Enables incoming SCP debugging |
| `debug ip ssh` | Enables SSH debugging |
| `debug aaa authentication` | Enables AAA authentication debugging |
| `debug aaa authorization` | Enables AAA authorization debugging |

## External SCP Client Command Summary

| Command or Option | Purpose |
|---|---|
| `scp -O` | Forces the traditional SCP protocol in OpenSSH |
| `scp -v` | Enables verbose SCP output |
| `scp -i <key-file>` | Specifies a private key for authentication |
| `scp -P <port>` | Specifies an alternative SSH destination TCP port |
| `ssh -V` | Displays the installed OpenSSH version |
| `pscp -scp` | Forces SCP when using PuTTY PSCP |

Notice that OpenSSH uses different options to specify the destination TCP port for SSH and SCP.

```text
ssh -p 2222 ADMIN@192.0.2.1
```

```text
scp -O -P 2222 test.txt ADMIN@192.0.2.1:bootflash:test.txt
```

The SSH command uses lowercase `-p`, while the SCP command uses uppercase `-P`.

## Key Concepts

- SCP provides secure file transfers using SSH.
- SCP normally uses TCP port 22.
- SCP and SFTP are different file transfer protocols.
- The SCP client initiates the connection, regardless of the direction of the file transfer.
- An SCP client can upload and download files.
- Cisco IOS XE can operate as both an SCP client and server.
- The `ip scp server enable` command enables incoming SCP transfers.
- SCP client functionality does not require enabling the local SCP server.
- SCP server operation does not require explicit AAA configuration. Traditional `login local` authentication with a privilege level 15 user is sufficient.
- When AAA is used, configure appropriate login authentication and EXEC authorization.
- IOS XE SCP server users require privilege level 15.
- An enable secret is not a substitute for assigning the required privilege level to an SCP user.
- Successful SSH authentication does not necessarily indicate successful SCP authorization.
- The `copy` command specifies the source before the destination.
- The position of `scp:` determines the direction of the file transfer.
- SCP URLs can specify the remote username, server, and filename.
- OpenSSH 9.0 and later use SFTP by default when invoking `scp`.
- The OpenSSH `scp -O` option forces the traditional SCP protocol.
- The OpenSSH `scp -P` option specifies a custom destination TCP port.
- Copying a configuration file into the running configuration merges configuration commands.
- Copying a configuration file to the startup configuration replaces the startup configuration.
- The `configure replace` command attempts to replace the running configuration instead of merging commands.
- Transferring an IOS XE image does not automatically install it.
- The `ip ssh source-interface` command controls the source interface for both outgoing SSH and SCP connections.
- The `vrf` keyword in the `copy` command provides per-transfer VRF selection on supported releases.
- Ordinary packet-filtering ACLs cannot distinguish SCP from interactive SSH based solely on TCP port numbers.
- Disabling the SCP server does not disable interactive SSH access or outgoing SCP client functionality.
- SSH bulk mode can improve SCP performance, particularly over high-bandwidth, high-latency networks.
- The `debug ip scp` command provides diagnostic information about incoming SCP operations.
- SCP troubleshooting must consider SSH connectivity, authentication, authorization, protocol compatibility, and file system access.

## References

- [Cisco IOS XE 17.x: Secure Copy](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-usr-ssh-sec-copy-0.html)
- [Cisco IOS XE 17.17.x: Secure Copy and SSH Bulk Mode](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9600/software/release/17-17/configuration_guide/sys_mgmt/b_1717_sys_mgmt_9600_cg/secure_copy.html)
- [Cisco: SCP Interoperability with OpenSSH 9.0 and Later](https://www.cisco.com/c/en/us/support/docs/troubleshooting/220371-scp-from-clients-on-openssh9-0-to-ios-xe.html)
- [Cisco IOS XE 17.x: Managing Configuration Files](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/syst-mgmt/b-system-management/m_cm-config-files-0.html)
- [Cisco IOS Configuration Fundamentals Command Reference: copy](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/fundamentals/command/cf_command_ref/C_commands.html)
- [OpenSSH: scp Manual](https://man.openbsd.org/scp)
