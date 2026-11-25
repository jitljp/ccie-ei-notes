# SSH Configuration and Verification

Cisco IOS XE supports both SSH server and SSH client functionality.

- The **SSH server** accepts incoming SSH connections.
- The **SSH client** initiates outgoing SSH connections.

These functions are independent. A router does not need to have its SSH server enabled to initiate an SSH connection to another device.

## SSH Server Configuration Requirements

Configuring an SSH server involves several independent tasks.

| Component | Purpose |
|---|---|
| Cryptographic host key | Provides the server's cryptographic identity |
| SSH version | Determines the supported SSH protocol version |
| Authentication | Verifies the identity of connecting users |
| VTY transport configuration | Permits incoming SSH connections on VTY lines |
| IP connectivity | Allows remote clients to reach the server |

Generating an appropriate cryptographic host key enables the SSH server.

The `transport input ssh` command permits incoming SSH connections on VTY lines but does not independently enable the SSH server.

## RSA Key Generation

RSA key pairs consist of two mathematically related keys:

- **Public key:** Can be distributed to other devices.
- **Private key:** Must remain confidential on the device.

The SSH server uses its private key to prove its identity to SSH clients.

The RSA host key is not directly used to encrypt all SSH session traffic. Symmetric session keys are established through the SSH key exchange process.

### Generating an RSA Key Pair

The `crypto key generate rsa` command generates an RSA key pair.

Starting with Cisco IOS XE 17.11.1a, this is a privileged EXEC command. Earlier releases use global configuration mode.

For example:

```text
R1# crypto key generate rsa modulus 2048
```

The `modulus` keyword specifies the RSA key size in bits.

Starting with Cisco IOS XE 17.11.1, RSA keys smaller than 2048 bits are not supported for SSH under the default cryptographic compliance settings.

### Default RSA Key Pair Name

If a key pair label is not explicitly specified, IOS XE uses the device's fully qualified domain name (FQDN) as the key pair name.

The FQDN consists of the hostname and domain name.

For example:

```text
R1(config)# hostname R1
R1(config)# ip domain name jitl.lab
R1(config)# end
R1# crypto key generate rsa modulus 2048
```

The resulting general-purpose RSA key pair is named `R1.jitl.lab`.

The hostname and domain name must be configured before generating an RSA key pair using the default key pair name.

However, neither a hostname nor a domain name needs to be explicitly configured when generating a named RSA key pair.


### Named RSA Key Pairs

The `label` keyword explicitly specifies the name of an RSA key pair.

For example:

```text
Router# crypto key generate rsa label SSH-KEYS modulus 2048
```

This generates a 2048-bit general-purpose RSA key pair named `SSH-KEYS`.

Because the key pair name is explicitly specified, IOS XE does not need to derive it from the device's FQDN.

This enables the SSH server without requiring a hostname or domain name to be explicitly configured.

The `ip ssh rsa keypair-name` command is not required to enable SSH.

### RSA Key Pair Types

The `crypto key generate rsa` command supports several RSA key pair types.

| Keyword | Purpose |
|---|---|
| `general-keys` | Generates a single general-purpose RSA key pair that can be used for signing and encryption |
| `usage-keys` | Generates two separate RSA key pairs: one for signing and one for encryption |
| `signature` | Generates an RSA key pair specifically for signing |
| `encryption` | Generates an RSA key pair specifically for encryption |

The `general-keys` keyword is optional. If no key pair type is specified, IOS XE generates a general-purpose RSA key pair by default.

Therefore, the following two commands are functionally equivalent:

```text
Router# crypto key generate rsa label SSH-KEYS modulus 2048
```

```text
Router# crypto key generate rsa general-keys label SSH-KEYS modulus 2048
```

Both commands generate a single 2048-bit general-purpose RSA key pair named `SSH-KEYS`.

Similarly, when using the default FQDN-based key pair name, the following commands are equivalent:

```text
R1# crypto key generate rsa modulus 2048
```

```text
R1# crypto key generate rsa general-keys modulus 2048
```

A general-purpose RSA key pair is sufficient for SSH server operation. There is no need to explicitly specify the `general-keys` keyword.

### Multiple RSA Key Pairs

Multiple RSA key pairs can exist on the same device.

For example:

```text
R1# crypto key generate rsa label SSH-KEYS modulus 2048
R1# crypto key generate rsa label PKI-KEYS modulus 2048
```

Different cryptographic applications can use different key pairs.

For example, one key pair might be used for SSH, while another is used by a PKI trustpoint.

### Explicitly Selecting an RSA Key Pair for SSH

The `ip ssh rsa keypair-name` global configuration command explicitly specifies which RSA key pair the SSH server uses.

```text
R1(config)# ip ssh rsa keypair-name SSH-KEYS
```

This is particularly useful when multiple RSA key pairs exist on the device.

The command does not generate a key pair. It associates a specified key pair with the SSH server.

For example:

```text
R1# crypto key generate rsa label SSH-KEYS modulus 2048
R1# crypto key generate rsa label PKI-KEYS modulus 2048
R1# configure terminal
R1(config)# ip ssh rsa keypair-name SSH-KEYS
```

The SSH server uses `SSH-KEYS` as its RSA host key pair.

The distinction between the two commands is important.

| Command | Function |
|---|---|
| `crypto key generate rsa ... label SSH-KEYS` | Generates an RSA key pair named `SSH-KEYS` |
| `ip ssh rsa keypair-name SSH-KEYS` | Selects `SSH-KEYS` for use by the SSH server |

The second command is optional when only one suitable RSA key pair exists.

To remove the explicit association:

```text
R1(config)# no ip ssh rsa keypair-name
```

Removing the association does not delete the RSA key pair.

## RSA Key Verification

The `show crypto key mypubkey rsa` command displays locally generated RSA public keys.

```text
R1# show crypto key mypubkey rsa
```

The output includes information such as:

- Key pair name.
- Key generation time.
- Key type and usage.
- Key storage location.
- Whether the key is exportable.
- Public key data.

Despite displaying information about RSA key pairs, the command does not display their private keys.

Use this command to verify that the required RSA key pair exists.

If `ip ssh rsa keypair-name` is configured, verify that the specified name matches an existing, suitable RSA key pair.

## Deleting RSA Key Pairs

The `crypto key zeroize rsa` command deletes RSA key pairs from the device.

On modern IOS XE releases, this is a privileged EXEC command.

### Deleting a Specific RSA Key Pair

Specify the key pair name to delete only that key pair.

```text
R1# crypto key zeroize rsa SSH-KEYS
```

This deletes the RSA key pair named `SSH-KEYS`.

Other RSA key pairs are not deleted.

### Deleting All RSA Key Pairs

If no key pair name is specified, the command deletes all RSA key pairs after confirmation.

```text
R1# crypto key zeroize rsa
```

Deleting all RSA key pairs can affect multiple services, not just SSH.

Other potentially affected services include:

- HTTPS.
- PKI certificate-based authentication.
- IPsec certificate-based authentication.
- SCP.

Deleting the only suitable RSA host key disables RSA-based SSH server operation.

Care must be taken when deleting RSA keys through a remote SSH session. Doing so can prevent subsequent SSH connections.

### Replacing an RSA Key Pair

A new RSA key pair can be generated using the same label as an existing key pair.

```text
R1# crypto key generate rsa label SSH-KEYS modulus 3072
```

IOS XE prompts for confirmation before replacing an existing key pair with the same name.

Replacing an SSH host key changes the cryptographic identity presented to SSH clients.

Clients that previously trusted the old host key may display a host key verification warning.

For example, an OpenSSH client might display:

```text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

A legitimate RSA key replacement can cause this warning. However, an unexpected host key change can also indicate a man-in-the-middle attack.

Administrators should verify the reason for the change before accepting the new host key.

For example, after confirming that the router's host key was intentionally replaced, an OpenSSH client can remove the old cached host key:

```text
ssh-keygen -R 192.0.2.1
```

The client can then establish a new SSH connection and verify the replacement host key.

## SSH Version Configuration

The `ip ssh version` command explicitly configures the SSH protocol version.

```text
R1(config)# ip ssh version 2
```

SSHv2 should be used instead of SSHv1.

Older IOS and IOS XE releases can support SSHv1, SSHv2, or compatibility mode.

### SSH Version 1.99

On releases that support both SSHv1 and SSHv2, the following output may be displayed:

```text
R1# show ip ssh
SSH Enabled - version 1.99
```

Version 1.99 indicates compatibility with both SSHv1 and SSHv2.

It does not indicate a separate SSH protocol version.

Explicitly configuring SSHv2 prevents the server from accepting SSHv1 connections on releases supporting both versions.

```text
R1(config)# ip ssh version 2
```

The output then indicates version 2.0:

```text
R1# show ip ssh
SSH Enabled - version 2.0
```

Modern IOS XE releases may support only SSHv2.

The available protocol versions and default behavior depend on the software release.

## VTY Configuration

Incoming interactive SSH sessions use VTY lines.

To permit SSH connections on VTY lines:

```text
R1(config)# line vty 0 15
R1(config-line)# transport input ssh
```

This permits SSH and prohibits other incoming terminal transport protocols, including Telnet, on the specified lines.

To permit both Telnet and SSH:

```text
R1(config-line)# transport input telnet ssh
```

To prevent incoming terminal transport connections:

```text
R1(config-line)# transport input none
```

These commands control line access. They do not generate RSA keys or remove the SSH server's cryptographic configuration.

### Configuring All VTY Lines

It is important to configure all relevant VTY lines.

For example:

```text
R1(config)# line vty 0 4
R1(config-line)# transport input ssh
!
R1(config)# line vty 5 15
R1(config-line)# transport input telnet ssh
```

In this example:

- VTY lines 0 through 4 permit only SSH.
- VTY lines 5 through 15 permit both Telnet and SSH.

The device still permits Telnet connections through VTY lines 5 through 15.

Use `show line` to determine which lines exist on the device.

Use the following command to verify the configuration of the VTY lines:

```text
R1# show running-config | section line vty
```

## Basic SSH Server Configuration

Consider the following topology:

```text
       R1                          R2
  192.0.2.1/24               192.0.2.2/24
       G1--------------------------G1

   SSH Server                  SSH Client
```

R2 will establish an SSH session to R1.

### R1 Configuration

Configure the interface and generate an RSA key pair:

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# ip address 192.0.2.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# end
!
R1# crypto key generate rsa label SSH-KEYS modulus 2048
```

Configure SSHv2 and local user authentication:

```text
R1# configure terminal
R1(config)# ip ssh version 2
R1(config)# username ADMIN privilege 15 secret ADMIN-PASSWORD
!
R1(config)# line vty 0 15
R1(config-line)# login local
R1(config-line)# transport input ssh
```

This configuration does not require a domain name because the RSA key pair has an explicit label.

It also does not require the `ip ssh rsa keypair-name` command because there is only one suitable RSA key pair.

### R2 Configuration

Configure IP connectivity:

```text
R2(config)# interface GigabitEthernet1
R2(config-if)# ip address 192.0.2.2 255.255.255.0
R2(config-if)# no shutdown
```

Initiate an SSH connection to R1:

```text
R2# ssh -l ADMIN 192.0.2.1
Password:

R1#
```

The user enters privileged EXEC mode because the local user account on R1 is configured with privilege level 15.

R2 does not require its own RSA host key pair to initiate this password-authenticated SSH connection.

## SSH Client Configuration

The Cisco IOS XE SSH client is invoked using the `ssh` EXEC command.

The SSH client does not require the local SSH server to be enabled.

### Specifying a Username

The `-l` option specifies the username used to authenticate to the remote SSH server.

```text
R2# ssh -l ADMIN 192.0.2.1
```

The username must be recognized by the remote server's authentication system.

It does not need to exist in R2's local user database.

### Specifying an SSH Version

The `-v` option specifies the SSH protocol version for the outgoing connection.

```text
R2# ssh -v 2 -l ADMIN 192.0.2.1
```

This requests an SSHv2 connection.

The SSH versions that can be specified depend on the IOS XE release.

### Specifying a Destination TCP Port

By default, the SSH client connects to TCP port 22.

The `-p` option specifies an alternative destination TCP port.

```text
R2# ssh -l ADMIN -p 2222 192.0.2.1
```

This initiates an SSH connection to TCP port 2222 instead of TCP port 22.

The remote SSH server must be configured to accept connections on that port.

### Executing a Remote Command

The IOS XE SSH client can specify a command to execute on the remote device instead of starting an ordinary interactive CLI session.

For example:

```text
R2# ssh -l ADMIN 192.0.2.1 show ip interface brief
```

This requests execution of `show ip interface brief` on R1.

The remote command is subject to the user's privileges and any applicable command authorization policies.

After the command completes, control returns to the local device.

### SSH Client Options Summary

| Option | Purpose |
|---|---|
| `-l` | Specifies the username |
| `-p` | Specifies the destination TCP port |
| `-v` | Specifies the SSH protocol version |
| `-vrf` | Specifies the routing table used for the connection |
| `-c` | Specifies an encryption algorithm |
| `-m` | Specifies a message authentication algorithm |

Available options and algorithms depend on the IOS XE release.

## SSH Client Source Interface

By default, locally initiated SSH connections normally use the IP address of the outgoing interface as their source IP address.

The `ip ssh source-interface` command specifies a source interface for locally initiated SSH connections.

Consider the following topology:

```text
     Loopback0
     2.2.2.2/32
          |
          R2--------------------R1
    192.0.2.2/24          192.0.2.1/24
          G1                    G1
```

Without an explicitly configured source interface, an SSH connection from R2 to R1 normally uses the following addresses:

```text
Source IP:      192.0.2.2
Destination IP: 192.0.2.1
```

Configure R2 to use Loopback0 as the source interface for SSH:

```text
R2(config)# ip ssh source-interface Loopback0
```

Initiate a new SSH connection:

```text
R2# ssh -l ADMIN 192.0.2.1
```

The connection uses the following addresses:

```text
Source IP:      2.2.2.2
Destination IP: 192.0.2.1
```

The remote server must have a return route to the specified source address.

For example:

```text
R1(config)# ip route 2.2.2.2 255.255.255.255 192.0.2.2
```

The specified source interface does not necessarily have to be the interface through which packets are transmitted.

Routing determines the outgoing interface.

### Use Cases

Configuring an SSH source interface is useful for:

- Providing a consistent source IP address.
- Using a loopback address for management traffic.
- Simplifying management ACL configuration.
- Avoiding source IP address changes when the outgoing interface changes.
- Using a dedicated management interface.

The `ip ssh source-interface` command applies to SSH client connections initiated by the local device.

It does not restrict which local interface addresses remote SSH clients can connect to.

For example, configuring the following command does not limit incoming SSH connections to Loopback0:

```text
R1(config)# ip ssh source-interface Loopback0
```

Incoming SSH access must be restricted using other mechanisms, such as VTY access classes, interface ACLs, or Management Plane Protection (MPP).

## VRF-Aware SSH

SSH supports VRF-aware connectivity.

This is particularly relevant when a device uses a dedicated management VRF.

### VRF-Aware SSH Client

Consider the following topology:

```text
       R2                          R1
  VRF MANAGEMENT              VRF MANAGEMENT
  192.0.2.2/24               192.0.2.1/24
       G1--------------------------G1
```

Both routers have GigabitEthernet1 assigned to a VRF named `MANAGEMENT`.

For example, on R2:

```text
R2(config)# vrf definition MANAGEMENT
R2(config-vrf)# address-family ipv4
R2(config-vrf-af)# exit-address-family
!
R2(config)# interface GigabitEthernet1
R2(config-if)# vrf forwarding MANAGEMENT
R2(config-if)# ip address 192.0.2.2 255.255.255.0
R2(config-if)# no shutdown
```

An ordinary SSH command uses the global routing table by default:

```text
R2# ssh -l ADMIN 192.0.2.1
```

Because the destination is reachable through the `MANAGEMENT` VRF, specify the VRF using the `-vrf` option:

```text
R2# ssh -l ADMIN -vrf MANAGEMENT 192.0.2.1
```

The SSH client uses the specified VRF's routing table to reach the destination.

The VRF name is locally significant.

The remote device does not need to use the same VRF name.

### Combining a VRF and Source Interface

The SSH client can use a specified source interface in combination with VRF-aware connectivity.

For example:

```text
R2(config)# ip ssh source-interface GigabitEthernet1
```

```text
R2# ssh -l ADMIN -vrf MANAGEMENT 192.0.2.1
```

Ensure that the configured source interface is compatible with the VRF used for the connection.

The source address must be reachable from the remote SSH server.

### VRF-Aware SSH Server

The IOS XE SSH server can accept connections received through interfaces belonging to VRFs.

A separate SSH server does not need to be configured for each VRF.

However, VTY access classes require special attention when connections are received through VRF interfaces.

Consider the following configuration:

```text
R1(config)# ip access-list standard MGMT-IN
R1(config-std-nacl)# permit 192.0.2.0 0.0.0.255
!
R1(config)# line vty 0 15
R1(config-line)# access-class MGMT-IN in
```

By default, this IPv4 VTY access class does not permit incoming connections from non-default VRFs.

Even if the source address matches a permit statement, a connection received through a VRF can be rejected.

The `vrf-also` keyword allows the access class to be applied to incoming connections from VRFs as well as the global routing instance.

```text
R1(config)# line vty 0 15
R1(config-line)# access-class MGMT-IN in vrf-also
```

The ACL still determines which source IP addresses are permitted.

The `vrf-also` keyword does not bypass the ACL.

| Configuration | Global Routing Instance | Non-Default VRFs |
|---|---|---|
| `access-class MGMT-IN in` | Evaluated against ACL | Rejected |
| `access-class MGMT-IN in vrf-also` | Evaluated against ACL | Evaluated against ACL |

The `vrf-also` keyword does not restrict access to one specific VRF.

When overlapping IP addressing is used in multiple VRFs, a source address permitted by the ACL might be permitted from multiple routing instances.

On releases supporting VRF-specific VTY access classes, more granular restrictions can be configured.

For additional information about VRF-aware services, see [VRF-Aware Services](../../../../network-infrastructure/routing-concepts/vrf-aware-routing/services.md).

## Custom SSH Ports

The default SSH server TCP port is 22.

Additional SSH ports can be associated with rotary groups.

For example:

```text
R1(config)# ip ssh port 2222 rotary 1
!
R1(config)# line vty 5 9
R1(config-line)# rotary 1
R1(config-line)# transport input ssh
```

Incoming SSH connections to TCP port 2222 are directed to eligible VTY lines in rotary group 1.

A client can connect using:

```text
R2# ssh -l ADMIN -p 2222 192.0.2.1
```

Configuring an additional SSH port does not disable the standard SSH port (TCP 22).

Additional filtering is required if connections to TCP port 22 must be prohibited.

Rotary group configuration, including sequential port mappings, is covered in [Rotary Groups and Custom TCP Ports](../console-vty/rotary-groups.md).

## SSH Connection Verification

Several commands can be used to verify SSH configuration and operation.

### show ip ssh

The `show ip ssh` command displays SSH server status and SSH configuration parameters.

```text
R1# show ip ssh
```

The following is an illustrative, abbreviated example:

```text
SSH Enabled - version 2.0
Authentication timeout: 120 secs; Authentication retries: 3
```

Depending on the IOS XE release, the output can also include:

- Supported authentication methods.
- Public-key algorithms.
- Host key algorithms.
- Encryption algorithms.
- MAC algorithms.
- Key exchange algorithms.
- Minimum Diffie-Hellman group size.
- SSH host public key information.

This command is useful for verifying the SSH server before attempting a connection.

If the SSH server is disabled, investigate whether a suitable cryptographic host key exists.

The command does not primarily display individual active SSH connections.

### show ssh

The `show ssh` command displays active SSH connections.

```text
R1# show ssh
```

The following is an illustrative example:

```text
Connection Version Mode Encryption  Hmac           State            Username
0          2.0     IN   aes128-ctr  hmac-sha2-256  Session started  ADMIN
0          2.0     OUT  aes128-ctr  hmac-sha2-256  Session started  ADMIN
```

The output includes information such as:

| Field | Description |
|---|---|
| Connection | SSH connection identifier |
| Version | Negotiated SSH protocol version |
| Mode | Direction of the SSH traffic |
| Encryption | Negotiated encryption algorithm |
| Hmac | Negotiated message authentication algorithm |
| State | Current connection state |
| Username | Authenticated username |

An SSHv2 connection can be represented by separate `IN` and `OUT` entries.

These entries represent the two communication directions of the SSH connection.

They do not indicate two independent SSH sessions.

SSH allows different cryptographic algorithms to be negotiated independently for each direction.

### show ip ssh vs. show ssh

The distinction between these two commands is important.

| Command | Purpose |
|---|---|
| `show ip ssh` | Displays SSH status and configuration parameters |
| `show ssh` | Displays operational information about active SSH connections |

For example, a router can have SSH enabled without any active SSH connections.

In that case, `show ip ssh` indicates that SSH is enabled, but `show ssh` does not list active connections.

### show users

The `show users` command displays active user sessions on the local device.

For example:

```text
R1# show users
    Line       User       Host(s)              Idle       Location
*  0 con 0                idle                 00:00:00
   2 vty 0     ADMIN      idle                 00:00:15 192.0.2.2
```

This output indicates that:

- The current CLI session is using the console.
- The user `ADMIN` is connected through VTY line 0.
- The remote user's source IP address is 192.0.2.2.

The `show users` command is particularly useful when verifying:

- Which VTY line was allocated.
- Whether the expected username authenticated.
- The remote client's source IP address.
- Whether SSH source-interface configuration is working.

Unlike `show ssh`, this command does not display negotiated cryptographic algorithms.

### show sessions

The `show sessions` command displays outgoing terminal connections associated with the current EXEC session.

Consider the following topology:

```text
Administrator --------> R2 --------> R1
                 SSH          SSH
```

The administrator connects to R2 and then initiates an SSH connection from R2 to R1.

The administrator can suspend the connection to R1 and return to R2 using the default IOS XE escape sequence:

```text
Ctrl-Shift-6, then x
```

On R2:

```text
R2# show sessions
```

This displays the suspended outgoing terminal connection.

To resume connection 1:

```text
R2# resume 1
```

To disconnect connection 1:

```text
R2# disconnect 1
```

The distinction between `show users` and `show sessions` is:

- `show users` displays user sessions on the local device.
- `show sessions` displays outgoing terminal connections associated with the current EXEC session.

### show tcp brief all

SSH uses TCP as its transport protocol.

The `show tcp brief all` command displays TCP connection information, including listening sockets and established connections.

```text
R1# show tcp brief all
```

An illustrative, abbreviated example is:

```text
TCB       Local Address       Foreign Address     (state)
...       *.22                *.*                 LISTEN
...       192.0.2.1.22        192.0.2.2.49152     ESTAB
```

The first entry represents a TCP listening socket on port 22.

The second entry represents an established TCP connection between:

- R1: TCP port 22.
- R2: TCP port 49152.

This command is useful for verifying TCP connectivity and determining which ports are being used.

An established TCP connection does not necessarily indicate successful SSH authentication.

TCP connection establishment occurs before SSH negotiation and authentication.

### show control-plane host open-ports

On platforms supporting this command, locally available TCP and UDP services can be displayed using:

```text
R1# show control-plane host open-ports
```

This can be useful for checking whether the router is listening on TCP port 22 or additional configured SSH ports.

An open SSH port does not guarantee that incoming users can successfully authenticate or obtain an EXEC session.

## Terminating SSH Connections

### Terminating the Current Session

An interactive SSH session can normally be terminated by entering:

```text
R1# exit
```

Alternatively:

```text
R1# logout
```

The remote VTY line is released after the session terminates.

### Terminating Another User's Session

An administrator can use `show users` to identify the line associated with another user's session.

```text
R1# show users
    Line       User       Host(s)              Idle       Location
*  0 con 0                idle                 00:00:00
   2 vty 0     ADMIN      idle                 00:02:30 192.0.2.2
```

To terminate the session associated with absolute line 2:

```text
R1# clear line 2
```

Alternatively, on supported platforms:

```text
R1# clear line vty 0
```

Clearing the line terminates the associated session and makes the VTY line available for subsequent connections.

## Troubleshooting SSH

SSH troubleshooting should distinguish between several stages of connection establishment.

```text
IP Connectivity
       |
       v
TCP Connection
       |
       v
SSH Version and Algorithm Negotiation
       |
       v
Server Authentication
       |
       v
User Authentication
       |
       v
Session Authorization and CLI Access
```

A failure at one stage prevents subsequent stages from completing.

### IP Connectivity Problems

Verify that the client can reach the SSH server.

For example:

```text
R2# ping 192.0.2.1
```

If the destination is in a VRF:

```text
R2# ping vrf MANAGEMENT 192.0.2.1
```

Check routing in the appropriate routing instance:

```text
R2# show ip route 192.0.2.1
```

```text
R2# show ip route vrf MANAGEMENT 192.0.2.1
```

If an SSH source interface is configured, verify connectivity using the corresponding source address.

```text
R2# ping 192.0.2.1 source Loopback0
```

A successful ping using the default source address does not necessarily prove that connectivity exists using the configured SSH source address.

### TCP Connectivity Problems

Successful ICMP communication does not guarantee that TCP port 22 is reachable.

An SSH connection can be prevented by:

- Interface ACLs.
- Firewalls.
- Management Plane Protection.
- Control Plane Policing.
- A missing SSH listening socket.
- Incorrect destination TCP port.

From another IOS XE device, a TCP connection attempt can be made using Telnet with a specified destination port:

```text
R2# telnet 192.0.2.1 22
```

If the connection succeeds, the SSH server may display its SSH identification string.

For example:

```text
Trying 192.0.2.1, 22 ... Open
SSH-2.0-Cisco-1.25
```

This confirms that a TCP connection to the SSH service can be established.

It does not establish a valid SSH session.

The exact identification string depends on the software version.

### SSH Server Not Enabled

Verify SSH server status:

```text
R1# show ip ssh
```

If SSH is disabled, verify the RSA keys:

```text
R1# show crypto key mypubkey rsa
```

Possible causes include:

- No RSA host key has been generated.
- The required RSA key pair has been deleted.
- The explicitly selected RSA key pair does not exist.
- The RSA key size does not meet the release's cryptographic requirements.

Generate or select an appropriate key pair as necessary.

### VTY Configuration Problems

Verify that the VTY lines permit SSH.

```text
R1# show running-config | section line vty
```

For example, the following configuration prevents incoming SSH connections on the specified lines:

```text
line vty 0 15
 transport input telnet
```

Correct the configuration:

```text
R1(config)# line vty 0 15
R1(config-line)# transport input ssh
```

Also verify that:

- An eligible VTY line is available.
- Incoming VTY access classes permit the client.
- VRF-related access class requirements are satisfied.
- EXEC access has not been disabled with `no exec`.

Use `show line` and `show users` to identify occupied VTY lines.

### SSH Negotiation Problems

The SSH client and server must support mutually compatible algorithms.

For example, an OpenSSH client might report:

```text
Unable to negotiate with 192.0.2.1 port 22:
no matching key exchange method found.
```

Other negotiation failures can involve:

- Host key algorithms.
- Encryption algorithms.
- Message authentication algorithms.
- SSH protocol versions.

A TCP connection can be established successfully while SSH algorithm negotiation fails.

Compare the client's supported algorithms with the server's configuration.

Use:

```text
R1# show ip ssh
```

From an OpenSSH client, verbose output can provide additional information:

```text
ssh -vvv ADMIN@192.0.2.1
```

Prefer updating or correcting the SSH server's cryptographic configuration rather than unnecessarily enabling weak algorithms on the client.

Algorithm configuration is covered in [SSH Authentication and Security Options](authentication.md).

### Host Key Verification Problems

If an SSH client rejects the server's host key, verify whether:

- The server's RSA keys have been replaced.
- The destination IP address now belongs to a different device.
- The client has an outdated cached host key.
- The connection is reaching an unexpected SSH server.

Server authentication occurs before user authentication.

Correct user credentials do not resolve a server host key verification failure.

### User Authentication Problems

If SSH negotiation succeeds but the user cannot authenticate, check:

- Username and password.
- Local username configuration.
- VTY login configuration.
- AAA authentication method lists.
- TACACS+ or RADIUS server availability.
- Public-key authentication configuration.

For example, if local authentication is intended:

```text
R1(config)# username ADMIN secret ADMIN-PASSWORD
!
R1(config)# line vty 0 15
R1(config-line)# login local
```

SSH requires user authentication. A traditional VTY line password configured with `password` is not sufficient by itself for SSH authentication.

### Authorization Problems

Successful authentication does not necessarily guarantee access to the CLI or permission to execute commands.

AAA EXEC authorization determines whether the authenticated user is permitted to start an EXEC session.

Command authorization can restrict which commands the user is permitted to execute.

For more information, see [4.1.b AAA](../../../device-security/aaa/overview.md).

## SSH Debugging

The `debug ip ssh` command enables SSH debugging.

```text
R1# debug ip ssh
```

SSH debugging can provide information about:

- Incoming and outgoing SSH connections.
- Protocol version negotiation.
- Algorithm negotiation.
- User authentication.
- Session establishment.
- Connection termination.
- SSH errors.

If troubleshooting from a remote CLI session, use `terminal monitor` to display debugging messages in that session when necessary.

```text
R1# terminal monitor
```

After troubleshooting, disable SSH debugging:

```text
R1# undebug ip ssh
```

Alternatively, disable all active debugging:

```text
R1# undebug all
```

Debugging should be used carefully on production devices.

## Configuration Command Summary

| Command | Mode | Purpose |
|---|---|---|
| `crypto key generate rsa modulus 2048` | Privileged EXEC | Generates a 2048-bit RSA key pair using the default name |
| `crypto key generate rsa label SSH-KEYS modulus 2048` | Privileged EXEC | Generates a named RSA key pair |
| `crypto key zeroize rsa SSH-KEYS` | Privileged EXEC | Deletes a specific RSA key pair |
| `crypto key zeroize rsa` | Privileged EXEC | Deletes all RSA key pairs |
| `ip ssh rsa keypair-name SSH-KEYS` | Global configuration | Explicitly selects an RSA key pair for SSH |
| `ip ssh version 2` | Global configuration | Configures SSHv2 |
| `ip ssh source-interface Loopback0` | Global configuration | Specifies the SSH client's source interface |
| `ip ssh port 2222 rotary 1` | Global configuration | Maps an SSH port to a rotary group |
| `transport input ssh` | Line configuration | Permits incoming SSH connections |
| `transport output ssh` | Line configuration | Permits outgoing SSH terminal connections |
| `login local` | Line configuration | Enables local user authentication |
| `access-class MGMT-IN in vrf-also` | Line configuration | Applies the access class to global and VRF connections |
| `ssh -l ADMIN 192.0.2.1` | EXEC | Initiates an SSH connection |
| `ssh -l ADMIN -p 2222 192.0.2.1` | EXEC | Initiates an SSH connection using a custom port |
| `ssh -l ADMIN -vrf MGMT 192.0.2.1` | EXEC | Initiates an SSH connection using a specified VRF |

The RSA key generation and deletion commands use privileged EXEC mode starting with IOS XE 17.11.1a. Earlier releases use global configuration mode.

## Verification Command Summary

| Command | Purpose |
|---|---|
| `show ip ssh` | Displays SSH server status and configuration |
| `show ssh` | Displays active SSH connections |
| `show crypto key mypubkey rsa` | Displays local RSA public key information |
| `show running-config \| include ^ip ssh` | Displays explicitly configured global SSH commands |
| `show running-config \| section line vty` | Displays VTY configuration |
| `show users` | Displays active user sessions |
| `show line` | Displays terminal line information |
| `show sessions` | Displays outgoing terminal connections |
| `show tcp brief all` | Displays TCP sockets and connections |
| `show control-plane host open-ports` | Displays locally available services on supported platforms |
| `debug ip ssh` | Enables SSH debugging |
| `clear line <number>` | Terminates the session associated with a specified line |

## Key Concepts

- SSH client and server functionality are independent.
- An appropriate cryptographic host key enables the SSH server.
- RSA key generation uses privileged EXEC mode starting with IOS XE 17.11.1a.
- The default RSA key pair name is derived from the device's FQDN.
- Explicitly naming an RSA key pair eliminates the hostname and domain name configuration requirements.
- The `ip ssh rsa keypair-name` command explicitly selects an RSA key pair but does not generate one.
- The `ip ssh rsa keypair-name` command is not required to enable SSH when only one suitable RSA key pair exists.
- Deleting an RSA key pair can affect multiple cryptographic services.
- Replacing an SSH host key can trigger host key verification warnings on clients.
- `transport input ssh` permits incoming SSH connections on VTY lines but does not independently enable the SSH server.
- The SSH client does not require an enabled local SSH server.
- The `-l`, `-p`, and `-vrf` SSH client options specify the username, destination port, and VRF, respectively.
- The `ip ssh source-interface` command controls the source address of locally initiated SSH connections.
- The `ip ssh source-interface` command does not restrict incoming SSH connections.
- The `vrf-also` keyword allows an inbound IPv4 VTY access class to evaluate connections received through VRFs.
- Configuring SSH rotary ports does not automatically disable TCP port 22.
- `show ip ssh` displays SSH status and configuration, whereas `show ssh` displays active SSH connections.
- TCP connection establishment, SSH negotiation, server authentication, user authentication, and authorization are distinct stages.

## References

- [Cisco IOS Security Command Reference: RSA Key Generation and Deletion](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/security/a1/sec-a1-cr-book/sec-cr-c4.html)
- [Cisco IOS Security Command Reference: ip ssh source-interface](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/security/d1/sec-d1-cr-book/sec-cr-i3.html)
- [Cisco IOS XE Secure Shell Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/sec-crypto/ssh/secure-shell-configuration-guide/m-secure-shell.html)
- [Cisco IOS XE Secure Shell Version 2 Support](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-secure-shell-v2-0.html)
- [Cisco Field Notice FN-72511: RSA Keys Less Than 2048 Bits Are Not Supported for SSH](https://www.cisco.com/c/en/us/support/docs/field-notices/725/fn72511.html)
