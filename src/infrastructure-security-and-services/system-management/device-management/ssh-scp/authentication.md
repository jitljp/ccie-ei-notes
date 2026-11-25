# SSH Authentication and Security Options

SSH provides several security mechanisms to protect remote management sessions:

- Server authentication.
- User authentication.
- Encryption.
- Data integrity protection.
- Session key exchange and rekeying.

Cisco IOS XE provides configuration options for controlling each of these mechanisms.

SSH security configuration involves both the cryptographic protection of the connection and the authentication and authorization of connecting users.

## Server Authentication vs. User Authentication

SSH performs two distinct types of authentication.

| Authentication Type | Purpose |
|---|---|
| Server authentication | Allows the client to verify the server's identity |
| User authentication | Allows the server to verify the user's identity |

Server authentication occurs before user authentication.

```text
SSH Client                         SSH Server
     |                                  |
     | ------ TCP Connection ---------> |
     |                                  |
     | <---- Algorithm Negotiation ---> |
     |                                  |
     | <---- Server Authentication ---- |
     |                                  |
     | ----- User Authentication -----> |
     |                                  |
     | <===== Secure SSH Session =====> |
```

Successful user authentication does not prove that the client is connected to the intended server.

Similarly, successful server authentication does not prove that the connecting user is authorized to access the server.

### Server Authentication

The SSH server uses a cryptographic host key to prove its identity to the client.

During key exchange, the server uses its private host key to generate a digital signature.

The client verifies the signature using the server's public host key.

However, verifying the signature alone only proves that the server possesses the corresponding private key.

The client must also determine whether the public key belongs to the intended server.

This can be accomplished by:

- Comparing the public key against a previously trusted host key.
- Comparing the public key's fingerprint against a trusted fingerprint.
- Validating a certificate, when certificate-based authentication is supported and configured.

### User Authentication

After establishing the secure SSH transport, the server authenticates the user.

User authentication can be performed using mechanisms such as passwords or public-key authentication.

The user's authentication credentials are distinct from the server's host key.

## SSH User Authentication Methods

IOS XE supports three primary SSH user authentication methods.

| Method | Description |
|---|---|
| `publickey` | The client proves possession of an authorized private key |
| `keyboard-interactive` | The server presents authentication challenges to the user |
| `password` | The client submits a username and password |

The traditional default order in which IOS XE advertises these methods is:

1. Public-key authentication.
2. Keyboard-interactive authentication.
3. Password authentication.

The SSH client determines which mutually supported authentication method to attempt.

The server's advertised order does not necessarily dictate the order in which the client attempts the methods.

### Password Authentication

Password authentication involves submitting a username and password to the SSH server.

The credentials are protected by the encrypted SSH transport.

For example, configure a local user:

```text
R1(config)# username ADMIN privilege 15 secret ADMIN-PASSWORD
!
R1(config)# line vty 0 15
R1(config-line)# login local
R1(config-line)# transport input ssh
```

From an SSH client:

```text
R2# ssh -l ADMIN 192.0.2.1
Password:

R1#
```

The server authenticates the submitted credentials against its local username database.

Password authentication can also use centralized AAA services such as TACACS+ and RADIUS.

### Keyboard-Interactive Authentication

Keyboard-interactive authentication allows the server to send one or more authentication challenges to the client.

The client presents the challenges to the user and returns the responses to the server.

For example:

```text
Username: ADMIN
Password:
Verification code:
```

Keyboard-interactive authentication can support mechanisms such as:

- Traditional password authentication.
- One-time passwords (OTPs).
- Challenge-response authentication.
- Multifactor authentication.

These capabilities depend on the authentication backend.

Keyboard-interactive authentication does not inherently provide multifactor authentication. It provides a mechanism through which authentication challenges can be exchanged.

A user might see an ordinary password prompt even when SSH is using keyboard-interactive authentication rather than the SSH password authentication method.

### Public-Key Authentication

Public-key authentication allows a user to authenticate by proving possession of a private key.

The user generates a key pair:

- The **private key** remains on the SSH client.
- The **public key** is configured on the SSH server.

The private key is never transmitted to the server.

During authentication, the client generates a digital signature using its private key.

The server verifies the signature using the client's authorized public key.

If the signature is valid and the key is authorized for the requested username, authentication can succeed.

## Public-Key Authentication Operation

Consider the following topology:

```text
Administrator                         R1
192.0.2.100                       192.0.2.1

+------------------+         +------------------+
| Client Key Pair  |         | Server Key Pair  |
|                  |         |                  |
| Client Private   |         | Server Private   |
| Client Public    |         | Server Public    |
+------------------+         +------------------+
                             | Authorized Keys  |
                             | ADMIN:           |
                             | Client Public    |
                             +------------------+
```

Two independent key pairs are involved:

1. The server's host key pair.
2. The administrator's authentication key pair.

The server's key pair is generated on R1.

The administrator's key pair is generated on the client, typically a workstation.

The administrator's public key is then configured on R1.

### Server and User Key Comparison

| Characteristic | Server Host Key | User Authentication Key |
|---|---|---|
| Purpose | Authenticate the SSH server | Authenticate the user |
| Private key location | SSH server | SSH client |
| Public key trusted by | SSH client | SSH server |
| Generated on | SSH server | SSH client |
| IOS XE configuration | `crypto key generate rsa` | `ip ssh pubkey-chain` to authorize a user's public key |

The two key pairs do not need to use the same cryptographic algorithm or key size.

Generating an RSA host key on R1 does not automatically configure public-key authentication for users.

Similarly, configuring a user's public key does not generate the server's host key.

## Configuring SSH Public-Key Authentication

The following example configures public-key authentication for an administrator connecting to R1 from a workstation running OpenSSH.

```text
Administrator                         R1
192.0.2.100 --------------------- 192.0.2.1

 SSH Client                        SSH Server
```

### Step 1: Configure the SSH Server

Generate an RSA host key on R1:

```text
R1# crypto key generate rsa label SSH-KEYS modulus 2048
```

Configure SSHv2 and the VTY lines:

```text
R1# configure terminal
R1(config)# ip ssh version 2
R1(config)# username ADMIN privilege 15 secret ADMIN-PASSWORD
!
R1(config)# line vty 0 15
R1(config-line)# login local
R1(config-line)# transport input ssh
```

The server's RSA host key is independent of the administrator's authentication key.

### Step 2: Generate the User's Key Pair

On the administrator's workstation, generate an RSA key pair using OpenSSH:

```text
ssh-keygen -t rsa -b 3072 -f ~/.ssh/r1_admin
```

The options have the following meanings:

| Option | Purpose |
|---|---|
| `-t rsa` | Generates an RSA key pair |
| `-b 3072` | Specifies a 3072-bit key size |
| `-f ~/.ssh/r1_admin` | Specifies the key file path |

Two files are generated:

```text
~/.ssh/r1_admin
~/.ssh/r1_admin.pub
```

| File | Contents |
|---|---|
| `r1_admin` | Private key |
| `r1_admin.pub` | Public key |

The private key should remain on the workstation and should be protected with a passphrase.

The public key will be configured on R1.

The passphrase protects the private key stored on the workstation. It is not the IOS XE user's password and is not transmitted to R1.

### Step 3: Obtain the User's Public Key

Display the contents of the public key file:

```text
cat ~/.ssh/r1_admin.pub
```

An OpenSSH RSA public key has the following format:

```text
ssh-rsa <base64-encoded-key-data> <comment>
```

The three components are:

- Key type.
- Base64-encoded public key data.
- Optional comment.

The comment is typically a username and hostname identifying the system on which the key was generated.

The comment is not used as the IOS XE username.

The `ssh-rsa` prefix identifies the RSA public key format. It does not necessarily mean that authentication must use SHA-1 signatures.

### Step 4: Configure the User's Public Key on R1

The `ip ssh pubkey-chain` command enters SSH public-key configuration mode.

```text
R1(config)# ip ssh pubkey-chain
R1(conf-ssh-pubkey)# username ADMIN
R1(conf-ssh-pubkey-user)# key-string
R1(conf-ssh-pubkey-data)# <public-key-data>
R1(conf-ssh-pubkey-data)# exit
R1(conf-ssh-pubkey-user)# exit
R1(conf-ssh-pubkey)# exit
```

Replace `<public-key-data>` with the actual public key data generated on the workstation.

The Base64-encoded key data can be divided across multiple lines to accommodate IOS XE CLI input length limitations.

Do not insert additional characters into the key data.

The `username ADMIN` command associates the public key with the username `ADMIN`.

The private key is not configured on R1.

### Step 5: Verify the Public-Key Configuration

Display the SSH public-key chain configuration:

```text
R1# show running-config | section ip ssh pubkey-chain
```

IOS XE can represent the configured public key using a key hash in the running configuration.

For example:

```text
ip ssh pubkey-chain
 username ADMIN
  key-hash ssh-rsa <key-hash>
```

The hash identifies the authorized public key.

It is not the private key, and it is not a password hash.

### Step 6: Connect Using Public-Key Authentication

From the workstation:

```text
ssh -i ~/.ssh/r1_admin ADMIN@192.0.2.1
```

The `-i` option specifies the private key file used for authentication.

The client may prompt for the private key's passphrase.

This prompt is local to the workstation.

If public-key authentication succeeds, the administrator does not need to enter the password configured for `ADMIN` on R1.

### Testing Public-Key Authentication Exclusively

By default, an OpenSSH client might fall back to password or keyboard-interactive authentication if public-key authentication fails.

This can make it difficult to determine whether public-key authentication is actually working.

To explicitly test public-key authentication without password fallback:

```text
ssh -i ~/.ssh/r1_admin \
    -o IdentitiesOnly=yes \
    -o PreferredAuthentications=publickey \
    ADMIN@192.0.2.1
```

The options have the following purposes:

| Option | Purpose |
|---|---|
| `IdentitiesOnly=yes` | Restricts the client to explicitly selected identities |
| `PreferredAuthentications=publickey` | Restricts authentication attempts to public-key authentication |

Use verbose output to verify the authentication method:

```text
ssh -vvv -i ~/.ssh/r1_admin \
    -o IdentitiesOnly=yes \
    -o PreferredAuthentications=publickey \
    ADMIN@192.0.2.1
```

An illustrative successful authentication message is:

```text
Authenticated to 192.0.2.1 using "publickey".
```

## The key-string and key-hash Commands

IOS XE supports two methods of identifying authorized public keys.

### key-string

The `key-string` command allows the actual public key data to be entered.

```text
R1(config)# ip ssh pubkey-chain
R1(conf-ssh-pubkey)# username ADMIN
R1(conf-ssh-pubkey-user)# key-string
R1(conf-ssh-pubkey-data)# <public-key-data>
```

This is the preferred method when initially importing a public key.

IOS XE can calculate the corresponding key hash from the supplied public key data.

### key-hash

The `key-hash` command allows an authorized public key to be identified by its hash.

```text
R1(config)# ip ssh pubkey-chain
R1(conf-ssh-pubkey)# username ADMIN
R1(conf-ssh-pubkey-user)# key-hash ssh-rsa <key-hash>
```

An administrator can obtain the appropriate hash from another IOS XE device or calculate it using a compatible hashing tool.

During authentication, the client supplies its public key and proves possession of the corresponding private key.

The server can compare the supplied public key against the configured hash.

The expected fingerprint algorithm and representation depend on the IOS XE release.

Do not assume that the default OpenSSH SHA-256 fingerprint string can be entered directly into every IOS XE `key-hash` command.

### Command Comparison

| Command | Purpose |
|---|---|
| `key-string` | Configures the authorized public key using its actual key data |
| `key-hash` | Configures the authorized public key using its hash |

Either method can be used to identify an authorized public key.

## Public-Key Authentication and Local Users

The following two commands serve different purposes:

```text
R1(config)# username ADMIN privilege 15 secret ADMIN-PASSWORD
```

```text
R1(config)# ip ssh pubkey-chain
R1(conf-ssh-pubkey)# username ADMIN
```

The global `username` command creates an account in the local user database.

The `username` command under `ip ssh pubkey-chain` creates a public-key association for an SSH username.

It is not equivalent to creating a local user account.

Public-key authentication determines whether the client possesses an authorized private key.

Privilege levels and authorization determine what the authenticated user is permitted to do.

For a local-user configuration, configure the corresponding local account with the desired privilege level.

When centralized AAA is used, authentication and authorization behavior must be considered separately.

## Removing a User's Public Key

To remove a username and its associated public-key configuration from the SSH public-key chain:

```text
R1(config)# ip ssh pubkey-chain
R1(conf-ssh-pubkey)# no username ADMIN
```

This removes the SSH public-key association for `ADMIN`.

It does not delete the global local user account.

Similarly, deleting a global local user account is a separate operation:

```text
R1(config)# no username ADMIN
```

Both configurations should be considered when removing a user's management access.

## Controlling SSH Authentication Methods

IOS XE allows the administrator to control which SSH user authentication methods are offered to clients.

Two command families are relevant:

```text
ip ssh server algorithm authentication
```

```text
ip ssh server authenticate user
```

The available syntax depends on the IOS XE release.

### Configuring an Authentication Method List

The `ip ssh server algorithm authentication` command defines the permitted authentication methods and their advertised order.

For example:

```text
R1(config)# ip ssh server algorithm authentication publickey keyboard password
```

The keywords correspond to:

| CLI Keyword | SSH Authentication Method |
|---|---|
| `publickey` | `publickey` |
| `keyboard` | `keyboard-interactive` |
| `password` | `password` |

To permit only public-key authentication:

```text
R1(config)# ip ssh server algorithm authentication publickey
```

This prevents clients from authenticating using the other authentication methods.

To permit public-key and password authentication, but not keyboard-interactive authentication:

```text
R1(config)# ip ssh server algorithm authentication publickey password
```

To restore the default authentication method list:

```text
R1(config)# default ip ssh server algorithm authentication
```

### Individually Enabling and Disabling Authentication Methods

The `ip ssh server authenticate user` command can individually enable an authentication method.

Its `no` form disables an authentication method.

For example, disable password authentication:

```text
R1(config)# no ip ssh server authenticate user password
```

Disable keyboard-interactive authentication:

```text
R1(config)# no ip ssh server authenticate user keyboard
```

Assuming public-key authentication remains enabled, the server now permits only public-key authentication.

To re-enable password authentication:

```text
R1(config)# ip ssh server authenticate user password
```

To restore the default authentication methods:

```text
R1(config)# default ip ssh server authenticate user
```

These commands provide an alternative mechanism for controlling SSH authentication methods.

Verify the available syntax using context-sensitive CLI help.

### Disabling Public-Key Authentication

Public-key authentication can also be disabled.

```text
R1(config)# no ip ssh server authenticate user publickey
```

IOS XE may display the following warning:

```text
%SSH:Publickey disabled.Overriding RFC
```

RFC 4252 specifies public-key authentication as a mandatory SSH authentication mechanism to implement.

IOS XE allows administrators to override the default behavior and disable the method.

### Password-Only Authentication

To permit only the SSH password authentication method:

```text
R1(config)# ip ssh server algorithm authentication password
```

Alternatively:

```text
R1(config)# no ip ssh server authenticate user publickey
R1(config)# no ip ssh server authenticate user keyboard
```

The second example assumes that password authentication remains enabled.

### Public-Key-Only Authentication

To permit only public-key authentication:

```text
R1(config)# ip ssh server algorithm authentication publickey
```

Alternatively:

```text
R1(config)# no ip ssh server authenticate user password
R1(config)# no ip ssh server authenticate user keyboard
```

The second example assumes that public-key authentication remains enabled.

Notice that disabling password authentication alone is not sufficient to prevent users from authenticating with passwords.

Keyboard-interactive authentication can also present password-based challenges.

Both methods must be disabled if the objective is to enforce public-key-only SSH authentication.

### Disabling All Authentication Methods

Disabling every SSH user authentication method prevents new users from authenticating.

This is not equivalent to configuring unauthenticated SSH access.

The SSH server rejects incoming connection requests when no authentication methods are available.

For example:

```text
R1# show ip ssh
Authentication methods:NONE
```

An authentication method list containing no enabled methods does not bypass authentication.

### Authentication Method Verification

Use `show ip ssh` to display the enabled SSH authentication methods.

For example:

```text
R1# show ip ssh
SSH Enabled - version 2.0
Authentication methods:publickey,keyboard-interactive,password
```

After restricting the server to public-key authentication:

```text
R1# show ip ssh
SSH Enabled - version 2.0
Authentication methods:publickey
```

These outputs are abbreviated examples.

## SSH Authentication Methods vs. AAA Methods

SSH user authentication methods and AAA authentication methods operate at different levels.

An SSH authentication method determines how authentication information is exchanged between the SSH client and server.

An AAA authentication method determines which authentication mechanism or backend IOS XE uses.

| Configuration | Function |
|---|---|
| `ip ssh server algorithm authentication password` | Permits SSH password authentication |
| `login local` | Uses the local user database for line authentication |
| `aaa authentication login ADMIN group tacacs+ local` | Defines an AAA login authentication method list |
| `login authentication ADMIN` | Applies the named AAA authentication method list to a line |

For example:

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login SSH-LOGIN group tacacs+ local
!
R1(config)# ip ssh server algorithm authentication password keyboard
!
R1(config)# line vty 0 15
R1(config-line)# login authentication SSH-LOGIN
R1(config-line)# transport input ssh
```

The SSH server permits password and keyboard-interactive authentication.

The VTY lines use the `SSH-LOGIN` AAA authentication method list, which specifies TACACS+ followed by the local database.

The SSH authentication method configuration does not determine whether TACACS+ or the local database is used.

Similarly, configuring TACACS+ authentication does not automatically disable SSH public-key authentication.

### AAA Authentication Fallback

Consider the following method list:

```text
R1(config)# aaa authentication login SSH-LOGIN group tacacs+ local
```

The local method is used if the TACACS+ method returns an error, such as when no TACACS+ server is reachable.

The local method is not used simply because the TACACS+ server rejects the submitted credentials.

This distinction is important when designing emergency management access.

For more information, see [AAA Method Lists and Processing](../../../device-security/aaa/method-lists.md).

### Authentication vs. Authorization

Successful SSH authentication proves the user's identity.

AAA EXEC authorization determines whether the authenticated user is permitted to establish an EXEC session and can assign session attributes such as a privilege level.

AAA command authorization can determine which commands the user is permitted to execute.

Public-key authentication does not automatically grant privilege level 15 or bypass command authorization.

For more information, see [EXEC Authorization](../../../device-security/aaa/exec-authorization.md) and [Command Authorization](../../../device-security/aaa/command-authorization.md).

## SSH Server Authentication on an IOS XE Client

When an IOS XE device operates as an SSH client, it can verify the remote server's host key.

Consider the following topology:

```text
       R2                           R1
  192.0.2.2 ------------------ 192.0.2.1

   SSH Client                    SSH Server
```

R2 should verify that the SSH server it connects to is actually R1.

### Configuring the Trusted Server Public Key

The remote server's public key can be configured on R2 using `ip ssh pubkey-chain`.

Unlike user public-key authentication, the `server` keyword is used instead of `username`.

```text
R2(config)# ip ssh pubkey-chain
R2(conf-ssh-pubkey)# server 192.0.2.1
R2(conf-ssh-pubkey-server)# key-string
R2(conf-ssh-pubkey-data)# <R1-public-host-key-data>
R2(conf-ssh-pubkey-data)# exit
R2(conf-ssh-pubkey-server)# exit
R2(conf-ssh-pubkey)# exit
```

The public key must be the remote SSH server's host public key in a format accepted by IOS XE.

It should be obtained through a trusted mechanism.

The server name or address should correspond to the destination used by the SSH client.

### Enabling Strict Host Key Checking

Configure strict host key checking on R2:

```text
R2(config)# ip ssh stricthostkeycheck
```

The SSH client verifies the remote server's host key.

A connection is rejected if the server cannot be authenticated against the trusted host key configuration.

The available strict host key checking modes and defaults depend on the IOS XE release.

### User vs. Server Public-Key Chains

The same `ip ssh pubkey-chain` configuration mode supports two different applications.

| Subcommand | Configured On | Purpose |
|---|---|---|
| `username <username>` | SSH server | Trusts a user's public key |
| `server <server-name>` | SSH client | Trusts a remote server's public host key |

For example:

```text
R1(config)# ip ssh pubkey-chain
R1(conf-ssh-pubkey)# username ADMIN
```

This configures public-key authentication information for a user connecting to R1.

```text
R1(config)# ip ssh pubkey-chain
R1(conf-ssh-pubkey)# server 192.0.2.2
```

This configures trusted host key information for an SSH server to which R1 connects.

The first configuration is used by the local SSH server.

The second configuration is used by the local SSH client.

## SSH Cryptographic Algorithms

SSHv2 uses several categories of cryptographic algorithms.

Each category provides a different security function.

| Category | Purpose |
|---|---|
| Key Exchange (KEX) | Establishes shared session key material |
| Host Key | Authenticates the SSH server |
| Encryption | Provides confidentiality |
| Message Authentication Code (MAC) | Provides message integrity and authenticity |
| User Public Key | Specifies permitted public-key authentication signature algorithms |

SSH does not use a single cryptographic algorithm for all security functions.

For example, a single SSH connection might use:

| Category | Example |
|---|---|
| KEX | `ecdh-sha2-nistp256` |
| Host Key | `rsa-sha2-512` |
| Encryption | `aes256-ctr` |
| MAC | `hmac-sha2-256-etm@openssh.com` |
| User Public Key | `rsa-sha2-256` |

The supported combinations depend on the client, server, available keys, and software releases.

### Algorithm Negotiation

During connection establishment, the SSH client and server exchange lists of supported algorithms.

They must agree on a compatible algorithm for each required category.

For transport algorithms, SSH generally selects the first algorithm in the client's preference list that also appears in the server's list and satisfies any applicable requirements.

For example:

```text
Client's Encryption List:
1. aes128-ctr
2. aes256-ctr

Server's Encryption List:
1. aes256-ctr
2. aes128-ctr
```

Both devices support the same two algorithms.

The negotiated encryption algorithm is `aes128-ctr` because it is the first mutually supported algorithm in the client's list.

The server's algorithm list restricts which algorithms are acceptable, but the first entry in the server's list is not guaranteed to be selected.

If there is no mutually supported algorithm in a required category, SSH connection establishment fails.

## Configuring SSH Server Algorithms

IOS XE supports several commands for restricting SSH server algorithms.

| Command | Controls |
|---|---|
| `ip ssh server algorithm kex` | Key exchange algorithms |
| `ip ssh server algorithm hostkey` | Server host key algorithms |
| `ip ssh server algorithm encryption` | Encryption algorithms |
| `ip ssh server algorithm mac` | Message authentication algorithms |
| `ip ssh server algorithm publickey` | User public-key signature algorithms |
| `ip ssh server algorithm authentication` | User authentication methods |

These commands have different functions.

In particular, `hostkey`, `publickey`, and `authentication` should not be confused with one another.

Supported algorithms and default lists change between IOS XE releases.

Use context-sensitive CLI help and `show ip ssh` to determine the algorithms available on the device.

## Key Exchange Algorithms

Key exchange establishes shared cryptographic key material between the client and server.

The resulting key material is used to derive symmetric encryption and integrity keys.

Examples of SSH key exchange algorithms include:

| Algorithm | Description |
|---|---|
| `diffie-hellman-group14-sha256` | Finite-field Diffie-Hellman using a 2048-bit group and SHA-256 |
| `diffie-hellman-group16-sha512` | Finite-field Diffie-Hellman using a 4096-bit group and SHA-512 |
| `ecdh-sha2-nistp256` | Elliptic Curve Diffie-Hellman using the NIST P-256 curve |
| `ecdh-sha2-nistp384` | Elliptic Curve Diffie-Hellman using the NIST P-384 curve |
| `ecdh-sha2-nistp521` | Elliptic Curve Diffie-Hellman using the NIST P-521 curve |
| `curve25519-sha256` | Curve25519-based key exchange |

Support for individual algorithms depends on the IOS XE release.

### Configuring Server Key Exchange Algorithms

The `ip ssh server algorithm kex` command specifies the permitted SSH server key exchange algorithms.

For example:

```text
R1(config)# ip ssh server algorithm kex ecdh-sha2-nistp256 diffie-hellman-group14-sha256
```

Only the configured key exchange algorithms are permitted.

An SSH client that supports neither algorithm cannot establish a connection to R1.

To restore the default key exchange algorithm list:

```text
R1(config)# default ip ssh server algorithm kex
```

### Minimum Diffie-Hellman Group Size

The `ip ssh dh min size` command specifies the minimum acceptable modulus size for finite-field Diffie-Hellman key exchange.

For example:

```text
R1(config)# ip ssh dh min size 4096
```

The device requires a minimum 4096-bit Diffie-Hellman modulus for applicable DH key exchange algorithms.

A fixed 2048-bit Diffie-Hellman group no longer satisfies this requirement.

The command applies to both the IOS XE SSH client and server.

The traditional default minimum size is 2048 bits.

The configured minimum Diffie-Hellman size is independent of the RSA host key size.

For example:

```text
R1# crypto key generate rsa label SSH-KEYS modulus 2048
!
R1(config)# ip ssh dh min size 4096
```

The RSA host key is 2048 bits, while the minimum acceptable finite-field Diffie-Hellman modulus is 4096 bits.

These values control different cryptographic functions.

The DH modulus size does not represent an equivalent elliptic-curve key size requirement.

## Host Key Algorithms

Host key algorithms are used to authenticate the SSH server during key exchange.

Examples include:

| Algorithm | Description |
|---|---|
| `ssh-rsa` | RSA signatures using SHA-1 |
| `rsa-sha2-256` | RSA signatures using SHA-256 |
| `rsa-sha2-512` | RSA signatures using SHA-512 |
| `ssh-ed25519` | Ed25519 signatures |
| `ecdsa-sha2-nistp256` | ECDSA signatures using the NIST P-256 curve |

Algorithm support depends on the IOS XE release.

### Configuring Host Key Algorithms

The `ip ssh server algorithm hostkey` command specifies the permitted server host key algorithms.

For example:

```text
R1(config)# ip ssh server algorithm hostkey rsa-sha2-512 rsa-sha2-256
```

R1 permits RSA host authentication using SHA-512 or SHA-256 signatures.

It does not offer the legacy SHA-1-based `ssh-rsa` signature algorithm.

To restore the default host key algorithm list:

```text
R1(config)# default ip ssh server algorithm hostkey
```

### RSA Keys vs. RSA Signature Algorithms

An RSA key pair and an RSA signature algorithm are different concepts.

For example:

```text
R1# crypto key generate rsa label SSH-KEYS modulus 2048
```

This generates a 2048-bit RSA key pair.

The same suitable RSA key pair can be used with multiple RSA signature algorithms, including:

- `ssh-rsa`.
- `rsa-sha2-256`.
- `rsa-sha2-512`.

The signature algorithm determines how the RSA key is used to produce a digital signature.

Disabling the `ssh-rsa` signature algorithm does not necessarily prevent the use of RSA keys.

RSA keys can still be used with the stronger RSA SHA-2 signature algorithms.

This distinction is particularly important when troubleshooting interoperability with modern OpenSSH clients.

## Encryption Algorithms

SSH encryption algorithms provide confidentiality for transmitted data.

After key exchange, SSH uses symmetric encryption to protect traffic.

Examples include:

| Algorithm | Description |
|---|---|
| `aes128-ctr` | AES with a 128-bit key in Counter mode |
| `aes192-ctr` | AES with a 192-bit key in Counter mode |
| `aes256-ctr` | AES with a 256-bit key in Counter mode |
| `aes128-gcm@openssh.com` | AES with a 128-bit key in Galois/Counter Mode |
| `aes256-gcm@openssh.com` | AES with a 256-bit key in Galois/Counter Mode |

AES-GCM provides authenticated encryption, combining confidentiality and integrity protection.

Older algorithms such as 3DES and certain CBC-mode ciphers are generally avoided in modern SSH deployments.

### Configuring Server Encryption Algorithms

The `ip ssh server algorithm encryption` command specifies the permitted server encryption algorithms.

For example:

```text
R1(config)# ip ssh server algorithm encryption aes256-ctr aes128-ctr
```

R1 permits only the specified encryption algorithms.

An SSH client must support at least one of them.

To restore the default encryption algorithm list:

```text
R1(config)# default ip ssh server algorithm encryption
```

## Message Authentication Code Algorithms

A Message Authentication Code (MAC) provides integrity and message authenticity protection.

SSH supports keyed hash-based MAC algorithms, known as HMACs.

Examples include:

| Algorithm | Description |
|---|---|
| `hmac-sha2-256` | HMAC using SHA-256 |
| `hmac-sha2-512` | HMAC using SHA-512 |
| `hmac-sha2-256-etm@openssh.com` | SHA-256 HMAC using Encrypt-then-MAC |
| `hmac-sha2-512-etm@openssh.com` | SHA-512 HMAC using Encrypt-then-MAC |
| `hmac-sha1` | Legacy HMAC using SHA-1 |

### Encrypt-then-MAC

Encrypt-then-MAC (EtM) calculates the MAC over the encrypted packet data according to the applicable SSH packet format.

The receiver can verify the integrity of the protected packet before decrypting its encrypted payload.

The `-etm@openssh.com` suffix identifies OpenSSH Encrypt-then-MAC algorithm variants.

### Configuring Server MAC Algorithms

The `ip ssh server algorithm mac` command specifies the permitted SSH server MAC algorithms.

For example:

```text
R1(config)# ip ssh server algorithm mac hmac-sha2-512-etm@openssh.com hmac-sha2-256-etm@openssh.com
```

Only the specified MAC algorithms are permitted when a separate MAC is required.

To restore the default MAC algorithm list:

```text
R1(config)# default ip ssh server algorithm mac
```

### MAC Algorithms and Authenticated Encryption

Not all encryption algorithms require a separate MAC algorithm for packet protection.

For example, AES-GCM is an authenticated encryption algorithm that provides both confidentiality and integrity.

When authenticated encryption is negotiated, integrity protection is provided by the encryption algorithm itself.

When a non-authenticated encryption algorithm such as AES-CTR is negotiated, SSH uses a separate MAC for integrity protection.

## User Public-Key Algorithms

The `ip ssh server algorithm publickey` command controls the permitted signature algorithms for SSH user public-key authentication.

For example:

```text
R1(config)# ip ssh server algorithm publickey rsa-sha2-512 rsa-sha2-256
```

Users authenticating with RSA keys must use one of the permitted RSA SHA-2 signature algorithms.

This command does not configure authorized user public keys.

The authorized public keys are configured separately using `ip ssh pubkey-chain`.

It also does not restrict the server's host key algorithms.

### Authentication vs. Public-Key vs. Host Key Algorithms

Consider the following commands:

```text
R1(config)# ip ssh server algorithm authentication publickey
R1(config)# ip ssh server algorithm publickey rsa-sha2-512 rsa-sha2-256
R1(config)# ip ssh server algorithm hostkey rsa-sha2-512 rsa-sha2-256
```

Each command controls a different aspect of SSH authentication.

| Command | Effect |
|---|---|
| `algorithm authentication publickey` | Requires users to authenticate using public-key authentication |
| `algorithm publickey rsa-sha2-512 rsa-sha2-256` | Restricts user public-key signature algorithms |
| `algorithm hostkey rsa-sha2-512 rsa-sha2-256` | Restricts server authentication signature algorithms |

The `publickey` authentication method and the `publickey` algorithm category are related but distinct.

Enabling public-key authentication does not automatically authorize a user's key.

Restricting user public-key algorithms does not automatically disable password authentication.

## Configuring SSH Client Algorithms

When IOS XE operates as an SSH client, its outgoing connections can be restricted using client-specific algorithm configuration commands.

For example:

```text
R2(config)# ip ssh client algorithm encryption aes256-ctr aes128-ctr
R2(config)# ip ssh client algorithm mac hmac-sha2-256-etm@openssh.com
R2(config)# ip ssh client algorithm kex ecdh-sha2-nistp256 diffie-hellman-group14-sha256
```

These commands restrict algorithms used by SSH connections initiated by R2.

They do not configure the algorithm policy for incoming SSH connections.

The corresponding server commands must be used to restrict incoming connections.

### Client and Server Algorithm Configuration

| Command Prefix | Applies To |
|---|---|
| `ip ssh server algorithm` | Incoming SSH connections |
| `ip ssh client algorithm` | Outgoing SSH connections |

The local SSH client and server can have different algorithm policies.

For example, a router might accept incoming SSH connections using only modern algorithms while maintaining a separate client policy for connecting to older devices.

However, legacy algorithm support should be avoided unless specifically required.

## Removing Individual SSH Algorithms

The `no` form of supported SSH algorithm commands can remove individual algorithms from the configured list.

For example:

```text
R1(config)# ip ssh server algorithm encryption aes128-ctr aes192-ctr aes256-ctr
R1(config)# no ip ssh server algorithm encryption aes192-ctr
```

The remaining encryption algorithms are:

```text
aes128-ctr
aes256-ctr
```

Removing an individual algorithm does not necessarily restore the default algorithm list.

Use the `default` form of the command to restore the default list.

```text
R1(config)# default ip ssh server algorithm encryption
```

IOS XE prevents all algorithms from being removed from certain required algorithm categories.

For example, the SSH server must have at least one permitted encryption algorithm.

This differs from SSH user authentication methods, all of which can be disabled, preventing user authentication.

## SSH Algorithm Defaults and IOS XE Releases

SSH cryptographic defaults change as older algorithms are deprecated and new algorithms are introduced.

For example, IOS XE 17.10 removed several algorithms from the default SSH algorithm lists, including:

- `diffie-hellman-group14-sha1`.
- `hmac-sha1`.
- `hmac-sha2-256`.
- `hmac-sha2-512`.

The SHA-2 Encrypt-then-MAC variants provide alternatives to the traditional non-EtM SHA-2 HMAC variants.

An algorithm being removed from the default list does not necessarily mean that it is completely unsupported.

There are three distinct possibilities:

- The algorithm is supported and enabled by default.
- The algorithm is supported but must be explicitly enabled.
- The algorithm is no longer supported.

The behavior depends on the IOS XE release and applicable cryptographic compliance settings.

Do not assume that an algorithm list observed on an older IOS XE release represents the defaults of a newer release.

Use the following command to examine the current configuration:

```text
R1# show ip ssh
```

Use context-sensitive CLI help to determine which algorithms are available for configuration.

For example:

```text
R1(config)# ip ssh server algorithm kex ?
```

## SSH Authentication Timeout

The `ip ssh time-out` command specifies the SSH negotiation and authentication timeout in seconds.

For example:

```text
R1(config)# ip ssh time-out 30
```

The SSH connection must complete the applicable negotiation and authentication process within the configured timeout.

The traditional default is 120 seconds.

The SSH authentication timeout does not determine the inactivity timeout of an established EXEC session.

For example:

```text
R1(config)# ip ssh time-out 30
!
R1(config)# line vty 0 15
R1(config-line)# exec-timeout 10 0
```

The two timers serve different purposes:

- SSH negotiation and authentication must complete within the SSH timeout.
- After an EXEC session is established, ten minutes of EXEC inactivity causes the session to terminate.

### SSH Timeout vs. EXEC Timeout

| Command | Unit | Purpose |
|---|---|---|
| `ip ssh time-out 30` | Seconds | Controls SSH negotiation and authentication timeout |
| `exec-timeout 10 0` | Minutes and seconds | Controls established EXEC session inactivity timeout |

The SSH authentication timeout does not terminate an established SSH session simply because the session has existed longer than the configured value.

## SSH Authentication Retries

The `ip ssh authentication-retries` command controls the number of authentication retries permitted by the SSH server.

For example:

```text
R1(config)# ip ssh authentication-retries 2
```

The server permits two authentication retries after the initial attempt, subject to the client's behavior and authentication mechanism.

The traditional default value is 3.

### Retries vs. Password Prompts

The number of authentication retries is not necessarily equal to the number of password prompts displayed to the user.

For password authentication, Cisco documents the number of prompts as the lower of:

- The number requested by the SSH client.
- The configured authentication retries plus one.

For example:

```text
R1(config)# ip ssh authentication-retries 2
```

The server permits up to three password attempts, consisting of the initial attempt and two retries.

Assume an OpenSSH client is configured as follows:

```text
ssh -o NumberOfPasswordPrompts=5 ADMIN@192.0.2.1
```

When using password authentication, the server's limit is lower than the client's requested limit.

The user can therefore receive up to three password prompts.

If the client instead requests only one password prompt:

```text
ssh -o NumberOfPasswordPrompts=1 ADMIN@192.0.2.1
```

The client's limit is lower, so the user receives only one password prompt.

This assumes that the client reaches password authentication without other failures.

### Authentication Retries vs. Login Blocking

The `ip ssh authentication-retries` command controls retries within an SSH connection.

It does not provide a complete mechanism for blocking repeated login attempts across multiple TCP connections.

An attacker could initiate another SSH connection after the current connection is terminated.

Login attack mitigation features such as `login block-for` provide additional protection against repeated authentication failures.

## SSH Rekeying

SSH uses symmetric session keys to encrypt transmitted traffic and provide integrity protection.

These keys are established during the initial SSH key exchange.

Using the same session keys indefinitely is undesirable.

**Rekeying** allows the SSH client and server to perform another key exchange during an established connection.

```text
SSH Client                         SSH Server
     |                                  |
     | <------ Initial KEX -----------> |
     |                                  |
     | <=== Traffic Using Key Set 1 ==> |
     |                                  |
     | <-------- Rekeying ------------> |
     |                                  |
     | <=== Traffic Using Key Set 2 ==> |
     |                                  |
```

Rekeying establishes fresh session key material without requiring the user to establish a new SSH connection.

It does not generate a new permanent RSA host key pair.

It also does not normally require the user to authenticate again.

Either SSH endpoint can initiate rekeying.

IOS XE supports configuring time-based and volume-based rekeying thresholds.

### Time-Based Rekeying

The `ip ssh rekey time` command specifies a rekeying interval in minutes.

For example:

```text
R1(config)# ip ssh rekey time 60
```

The configured time-based rekeying threshold is 60 minutes.

Cisco documents a supported range of 10 through 1440 minutes.

### Volume-Based Rekeying

The `ip ssh rekey volume` command specifies a data volume threshold in kilobytes.

For example:

```text
R1(config)# ip ssh rekey volume 102400
```

The configured volume-based rekeying threshold is 102400 KB.

Cisco documents a supported range of 100 through 4194303 KB.

Volume-based rekeying is particularly relevant to SSH-based file transfers, which can transmit considerably more data than ordinary interactive CLI sessions.

### Combining Rekeying Thresholds

Time-based and volume-based thresholds can be configured together on releases supporting both settings concurrently.

For example:

```text
R1(config)# ip ssh rekey time 60
R1(config)# ip ssh rekey volume 102400
```

Rekeying can occur when the applicable configured threshold is reached.

The peer can also initiate rekeying independently of the locally configured thresholds.

### Removing Rekeying Configuration

To remove the explicitly configured rekeying settings:

```text
R1(config)# no ip ssh rekey
```

Removing the configuration does not guarantee that an SSH session will never rekey.

Protocol limitations, implementation behavior, and peer-initiated key exchanges can still trigger rekeying.

### Rekeying vs. RSA Key Replacement

| Operation | Effect |
|---|---|
| SSH rekeying | Establishes fresh session key material |
| RSA key replacement | Replaces the server's persistent RSA key pair |

SSH rekeying does not change the server's persistent RSA identity.

Replacing the server's RSA host key can cause clients to report an unexpected host key change.

## SSH Access Restrictions

SSH authentication and encryption do not prevent unauthorized hosts from attempting to connect to the server.

Management access should also be restricted at the network and terminal-line levels.

### VTY Access Classes

Configure an ACL permitting trusted management hosts:

```text
R1(config)# ip access-list standard SSH-CLIENTS
R1(config-std-nacl)# permit 192.0.2.0 0.0.0.255
!
R1(config)# line vty 0 15
R1(config-line)# access-class SSH-CLIENTS in
R1(config-line)# transport input ssh
```

Only clients whose source addresses are permitted by the access class can establish incoming sessions on the configured VTY lines.

For incoming connections through non-default VRFs, the `vrf-also` keyword can be required:

```text
R1(config-line)# access-class SSH-CLIENTS in vrf-also
```

For more information, including VRF-specific and IPv6 access classes, see [Session Timeouts and Access Restrictions](../console-vty/access-control.md).

### Management Plane Protection

Management Plane Protection (MPP) can restrict which interfaces are permitted to receive management traffic.

For example, on a platform supporting MPP:

```text
R1(config)# control-plane host
R1(config-cp-host)# management-interface GigabitEthernet1 allow ssh
```

MPP restricts the interfaces through which supported management protocols can be accessed.

It does not replace SSH user authentication.

For more information, see [Management Plane Protection](../../../device-security/control-plane/mpp.md).

### Control Plane Policing

Control Plane Policing (CoPP) can rate-limit SSH traffic destined for the control plane.

This helps protect the device against excessive management traffic.

CoPP does not authenticate users or determine their privileges.

For more information, see [Control Plane Policing and Protection](../../../device-security/control-plane/overview.md).

## Login Attack Mitigation

IOS XE provides login security features that can help protect against repeated authentication failures.

For example:

```text
R1(config)# login block-for 120 attempts 3 within 60
```

If three qualifying failed login attempts occur within 60 seconds, the device enters quiet mode for 120 seconds.

During quiet mode, remote login attempts are restricted.

An ACL can define trusted hosts that are exempt from quiet mode restrictions:

```text
R1(config)# ip access-list standard TRUSTED-ADMINS
R1(config-std-nacl)# permit 192.0.2.100
!
R1(config)# login quiet-mode access-class TRUSTED-ADMINS
```

The SSH authentication retry limit and login attack mitigation provide different types of protection.

| Feature | Scope |
|---|---|
| `ip ssh authentication-retries` | Authentication retries within an SSH connection |
| `login block-for` | Repeated login failures across connections |

For more information, see [Session Timeouts and Access Restrictions](../console-vty/access-control.md).

## SSH Event Logging

The `ip ssh logging events` command enables logging of SSH connection-related events.

```text
R1(config)# ip ssh logging events
```

SSH event logging can help administrators identify connection attempts, authentication activity, and session termination events.

Starting with IOS XE 17.12.1a, this command is enabled by default.

Login-related logging can also be configured using:

```text
R1(config)# login on-success log
R1(config)# login on-failure log
```

These commands enable logging of successful and failed login attempts, respectively.

Display the logging buffer:

```text
R1# show logging
```

For example, SSH-related messages can be filtered using:

```text
R1# show logging | include SSH
```

Login-related messages can be filtered using:

```text
R1# show logging | include LOGIN
```

SSH event logging and general login event logging are separate features.

## SSH Security Mechanism Summary

Several security features can be combined to protect SSH management access.

```text
Incoming SSH Connection
           |
           v
Interface and Network Restrictions
           |
           v
Control Plane Protection
           |
           v
SSH Protocol and Algorithm Negotiation
           |
           v
Server Authentication
           |
           v
VTY Access and User Authentication
           |
           v
EXEC Authorization
           |
           v
Command Authorization
           |
           v
Established Management Session
```

This diagram illustrates the different protection layers conceptually. It does not represent the exact IOS XE packet processing order.

Each mechanism provides a different security function.

| Mechanism | Purpose |
|---|---|
| Interface ACL | Filters network traffic |
| MPP | Restricts management access to permitted interfaces |
| CoPP | Protects control plane resources |
| SSH algorithm policy | Restricts permitted cryptographic algorithms |
| SSH host key verification | Authenticates the server |
| VTY access class | Restricts incoming terminal access |
| SSH user authentication | Authenticates connecting users |
| EXEC authorization | Controls permission to establish an EXEC session |
| Command authorization | Controls permitted CLI commands |
| EXEC timeout | Terminates inactive EXEC sessions |
| SSH rekeying | Refreshes session key material |
| Login attack mitigation | Restricts repeated login attempts |

These mechanisms complement one another.

No single mechanism provides all aspects of SSH management security.

## Complete Configuration Example

Consider the following topology:

```text
Administrator                         R1
192.0.2.100 --------------------- 192.0.2.1
                                     G1
```

The requirements are:

- Use SSHv2.
- Generate a named 3072-bit RSA host key pair.
- Permit only public-key user authentication.
- Permit SSH access only from 192.0.2.0/24.
- Restrict the permitted cryptographic algorithms.
- Limit authentication negotiation to 30 seconds.
- Permit two authentication retries.
- Terminate inactive EXEC sessions after five minutes.
- Configure time-based and volume-based SSH rekeying.
- Log SSH and login events.

The following example assumes a modern IOS XE release supporting the specified algorithms.

### RSA Host Key Generation

```text
R1# crypto key generate rsa label SSH-KEYS modulus 3072
```

### SSH Configuration

```text
R1# configure terminal
R1(config)# ip ssh version 2
R1(config)# ip ssh time-out 30
R1(config)# ip ssh authentication-retries 2
!
R1(config)# ip ssh server algorithm kex ecdh-sha2-nistp256 diffie-hellman-group14-sha256
R1(config)# ip ssh server algorithm hostkey rsa-sha2-512 rsa-sha2-256
R1(config)# ip ssh server algorithm encryption aes256-ctr aes128-ctr
R1(config)# ip ssh server algorithm mac hmac-sha2-512-etm@openssh.com hmac-sha2-256-etm@openssh.com
R1(config)# ip ssh server algorithm publickey rsa-sha2-512 rsa-sha2-256
!
R1(config)# ip ssh rekey time 60
R1(config)# ip ssh rekey volume 102400
R1(config)# ip ssh logging events
```

### User Configuration

```text
R1(config)# username ADMIN privilege 15 secret ADMIN-PASSWORD
!
R1(config)# ip ssh pubkey-chain
R1(conf-ssh-pubkey)# username ADMIN
R1(conf-ssh-pubkey-user)# key-string
R1(conf-ssh-pubkey-data)# <ADMIN-public-key-data>
R1(conf-ssh-pubkey-data)# exit
R1(conf-ssh-pubkey-user)# exit
R1(conf-ssh-pubkey)# exit
```

The actual public key data must be obtained from the administrator's workstation.

### Management Access Configuration

```text
R1(config)# ip access-list standard SSH-CLIENTS
R1(config-std-nacl)# permit 192.0.2.0 0.0.0.255
!
R1(config)# line vty 0 15
R1(config-line)# login local
R1(config-line)# transport input ssh
R1(config-line)# access-class SSH-CLIENTS in
R1(config-line)# exec-timeout 5 0
```

### Login Logging

```text
R1(config)# login on-success log
R1(config)# login on-failure log
```

### Testing Public-Key Authentication

Before disabling password and keyboard-interactive authentication, verify that public-key authentication works.

From the administrator's workstation:

```text
ssh -i ~/.ssh/r1_admin \
    -o IdentitiesOnly=yes \
    -o PreferredAuthentications=publickey \
    ADMIN@192.0.2.1
```

After confirming that public-key authentication succeeds, restrict the permitted authentication methods.

```text
R1(config)# ip ssh server algorithm authentication publickey
```

Verify that a new public-key-authenticated SSH session can still be established.

Do not terminate the existing management session until the replacement authentication mechanism has been verified.

Console access provides an additional recovery mechanism if the SSH configuration prevents remote management.

## Verification

### show ip ssh

The `show ip ssh` command displays SSH configuration and server status.

```text
R1# show ip ssh
```

The following is an illustrative, abbreviated example:

```text
SSH Enabled - version 2.0
Authentication methods:publickey
Authentication Publickey Algorithms:rsa-sha2-512,rsa-sha2-256
Hostkey Algorithms:rsa-sha2-512,rsa-sha2-256
Encryption Algorithms:aes256-ctr,aes128-ctr
MAC Algorithms:hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
KEX Algorithms:ecdh-sha2-nistp256,diffie-hellman-group14-sha256
Authentication timeout: 30 secs; Authentication retries: 2
Minimum expected Diffie Hellman key size : 2048 bits
```

Important information includes:

- Whether SSH is enabled.
- SSH protocol version.
- Enabled authentication methods.
- User public-key algorithms.
- Server host key algorithms.
- Encryption algorithms.
- MAC algorithms.
- Key exchange algorithms.
- Authentication timeout.
- Authentication retries.
- Minimum Diffie-Hellman modulus size.

The displayed algorithms are the configured algorithm policy, not necessarily the algorithms negotiated for an individual SSH connection.

### show ssh

The `show ssh` command displays active SSH connections.

```text
R1# show ssh
```

For example:

```text
Connection Version Mode Encryption  Hmac                            State           Username
0          2.0     IN   aes256-ctr  hmac-sha2-256-etm@openssh.com  Session started ADMIN
0          2.0     OUT  aes256-ctr  hmac-sha2-256-etm@openssh.com  Session started ADMIN
```

This displays operational information about established SSH connections, including the negotiated encryption and integrity algorithms.

Use `show ip ssh` to verify the configured policy.

Use `show ssh` to verify the operational algorithms used by active connections.

### show running-config

Display explicitly configured global SSH commands:

```text
R1# show running-config | include ^ip ssh
```

Display the SSH public-key chain:

```text
R1# show running-config | section ip ssh pubkey-chain
```

Display VTY configuration:

```text
R1# show running-config | section line vty
```

Commands retaining their default values may not appear in the normal running configuration output.

Use `show ip ssh` to verify effective SSH settings.

### show users

Display active management sessions:

```text
R1# show users
```

This is useful for verifying the authenticated username, allocated VTY line, and remote source IP address.

It does not display the complete SSH authentication exchange.

### OpenSSH Verbose Output

From an OpenSSH workstation:

```text
ssh -vvv ADMIN@192.0.2.1
```

Verbose output can display:

- Remote SSH protocol version.
- Negotiated key exchange algorithm.
- Negotiated host key algorithm.
- Negotiated encryption algorithms.
- Negotiated MAC algorithms.
- Host key verification information.
- Available user authentication methods.
- Authentication methods attempted by the client.
- Public keys offered by the client.
- Authentication results.

For example:

```text
debug1: kex: algorithm: ecdh-sha2-nistp256
debug1: kex: host key algorithm: rsa-sha2-512
debug1: Authentications that can continue: publickey
```

This is particularly useful when troubleshooting interoperability between IOS XE and third-party SSH clients.

## Troubleshooting SSH Authentication and Security

### Scenario 1: Public-Key Authentication Fails but Password Authentication Works

Possible causes include:

- The user's public key is not configured on the server.
- The public key is associated with the wrong username.
- The client is using the wrong private key.
- The configured public key data is incomplete or incorrect.
- Public-key authentication is disabled.
- The client and server do not support a mutually acceptable user public-key signature algorithm.

Verify the authorized public key configuration:

```text
R1# show running-config | section ip ssh pubkey-chain
```

Verify the permitted authentication methods and algorithms:

```text
R1# show ip ssh
```

Explicitly test public-key authentication from the client:

```text
ssh -vvv -i ~/.ssh/r1_admin \
    -o IdentitiesOnly=yes \
    -o PreferredAuthentications=publickey \
    ADMIN@192.0.2.1
```

### Scenario 2: Password Authentication Still Works After Being Disabled

Assume the following command is configured:

```text
R1(config)# no ip ssh server authenticate user password
```

The user might still receive a password prompt through keyboard-interactive authentication.

Verify the enabled authentication methods:

```text
R1# show ip ssh
```

If the objective is to enforce public-key-only authentication, also disable keyboard-interactive authentication:

```text
R1(config)# no ip ssh server authenticate user keyboard
```

Alternatively, configure the permitted authentication method list explicitly:

```text
R1(config)# ip ssh server algorithm authentication publickey
```

### Scenario 3: No Matching Key Exchange Algorithm

The client displays:

```text
Unable to negotiate with 192.0.2.1 port 22:
no matching key exchange method found.
```

The client and server have no mutually supported key exchange algorithm.

Verify the server's configured algorithms:

```text
R1# show ip ssh
```

From an OpenSSH workstation, display locally supported key exchange algorithms:

```text
ssh -Q kex
```

The `ssh -Q` command displays algorithms supported by the OpenSSH implementation. It does not necessarily indicate which algorithms are enabled by default.

Correct the algorithm policy so that both devices support at least one acceptable key exchange algorithm.

### Scenario 4: No Matching Host Key Type

The client displays:

```text
Unable to negotiate with 192.0.2.1 port 22:
no matching host key type found.
```

The client and server cannot agree on an acceptable server host key algorithm.

An older server might offer only the SHA-1-based `ssh-rsa` signature algorithm.

Modern OpenSSH clients generally disable this signature algorithm by default.

If supported by IOS XE, configure RSA SHA-2 host key algorithms:

```text
R1(config)# ip ssh server algorithm hostkey rsa-sha2-512 rsa-sha2-256
```

A new RSA key pair is not necessarily required. An existing suitable RSA key pair can be used with RSA SHA-2 signatures.

### Scenario 5: Server Authentication Succeeds but User Public-Key Authentication Fails

Host key algorithms and user public-key signature algorithms are configured independently.

For example:

```text
R1(config)# ip ssh server algorithm hostkey rsa-sha2-512 rsa-sha2-256
```

This controls server authentication.

It does not configure the signature algorithms accepted for user authentication.

Check the user public-key algorithm configuration:

```text
R1(config)# ip ssh server algorithm publickey rsa-sha2-512 rsa-sha2-256
```

Also verify that the user's public key is authorized for the requested username.

### Scenario 6: SSH Host Key Has Changed

The client displays:

```text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

Possible causes include:

- The server's host key was intentionally replaced.
- The router was replaced or reinstalled.
- The destination IP address now belongs to another device.
- The client is connecting to an unexpected SSH server.
- A man-in-the-middle attack is occurring.

Verify the cause through a trusted mechanism before accepting the new host key.

After confirming that the key change is legitimate, remove the outdated OpenSSH known-host entry:

```text
ssh-keygen -R 192.0.2.1
```

Reconnect and verify the replacement host key.

Do not automatically disable host key checking to bypass the warning.

### Scenario 7: SSH Authentication Succeeds but No EXEC Session Is Established

Possible causes include:

- AAA EXEC authorization failure.
- Incorrect privilege or authorization attributes.
- No eligible VTY line.
- EXEC access disabled using `no exec`.

User authentication and EXEC session authorization are distinct stages.

Verify the VTY configuration and applicable AAA method lists.

### Scenario 8: Increasing the DH Minimum Size Breaks SSH

Assume the server is configured as follows:

```text
R1(config)# ip ssh server algorithm kex diffie-hellman-group14-sha256
R1(config)# ip ssh dh min size 4096
```

The only permitted finite-field Diffie-Hellman algorithm uses a 2048-bit group.

However, the configured minimum modulus size is 4096 bits.

The configured key exchange algorithm cannot satisfy the minimum size requirement.

Use an appropriate key exchange algorithm with a sufficiently large group, such as `diffie-hellman-group16-sha512`, where supported.

Alternatively, correct the minimum DH size to match the intended cryptographic policy.

## Debugging SSH Authentication

The `debug ip ssh` command can provide information about SSH negotiation, authentication, and session establishment.

```text
R1# debug ip ssh
```

If AAA is involved, additional debugging commands can be useful:

```text
R1# debug aaa authentication
R1# debug aaa authorization
```

These commands help distinguish between SSH protocol problems and AAA authentication or authorization problems.

Disable debugging after troubleshooting:

```text
R1# undebug all
```

Debugging should be used carefully on production devices.

## Configuration Command Summary

| Command | Purpose |
|---|---|
| `ip ssh pubkey-chain` | Enters SSH public-key configuration mode |
| `username <username>` | Associates a public key with an SSH username under the public-key chain |
| `server <server-name>` | Configures trusted remote SSH server public-key information |
| `key-string` | Configures public key data |
| `key-hash ssh-rsa <hash>` | Identifies an RSA public key by its hash |
| `ip ssh stricthostkeycheck` | Enables strict SSH server host key verification by the client |
| `ip ssh server algorithm authentication` | Configures permitted SSH user authentication methods |
| `ip ssh server authenticate user` | Individually enables an SSH user authentication method |
| `no ip ssh server authenticate user` | Individually disables an SSH user authentication method |
| `ip ssh server algorithm kex` | Configures permitted server key exchange algorithms |
| `ip ssh server algorithm hostkey` | Configures permitted server host key algorithms |
| `ip ssh server algorithm encryption` | Configures permitted server encryption algorithms |
| `ip ssh server algorithm mac` | Configures permitted server MAC algorithms |
| `ip ssh server algorithm publickey` | Configures permitted user public-key signature algorithms |
| `ip ssh client algorithm` | Configures SSH client algorithm policies |
| `ip ssh dh min size <bits>` | Specifies the minimum finite-field DH modulus size |
| `ip ssh time-out <seconds>` | Configures the SSH negotiation and authentication timeout |
| `ip ssh authentication-retries <number>` | Configures the maximum number of authentication retries |
| `ip ssh rekey time <minutes>` | Configures a time-based rekeying threshold |
| `ip ssh rekey volume <kilobytes>` | Configures a volume-based rekeying threshold |
| `ip ssh logging events` | Enables SSH event logging |

## Verification Command Summary

| Command | Purpose |
|---|---|
| `show ip ssh` | Displays SSH configuration, enabled authentication methods, and algorithm lists |
| `show ssh` | Displays active SSH connections and negotiated algorithms |
| `show crypto key mypubkey rsa` | Displays local RSA public key information |
| `show running-config \| section ip ssh pubkey-chain` | Displays SSH public-key chain configuration |
| `show running-config \| section line vty` | Displays VTY authentication and access configuration |
| `show users` | Displays active user sessions |
| `show login` | Displays login attack mitigation status |
| `show login failures` | Displays recorded login failures |
| `show logging` | Displays logged events |
| `debug ip ssh` | Enables SSH protocol and authentication debugging |
| `debug aaa authentication` | Enables AAA authentication debugging |
| `debug aaa authorization` | Enables AAA authorization debugging |

## Key Concepts

- SSH server authentication and user authentication are separate processes.
- Server authentication occurs before user authentication.
- The server's host private key remains on the server.
- The user's authentication private key remains on the client.
- Public-key authentication proves possession of an authorized private key.
- The user's private key is never transmitted to the SSH server.
- IOS XE supports public-key, keyboard-interactive, and password authentication.
- Keyboard-interactive authentication can present password-based authentication challenges.
- Disabling password authentication alone does not necessarily prevent password-based user authentication.
- `ip ssh pubkey-chain` configures trusted public keys for user and server authentication.
- The `username` subcommand configures a user's public-key association.
- The `server` subcommand configures trusted remote server host key information.
- `ip ssh stricthostkeycheck` enables strict server host key verification by the IOS XE SSH client.
- SSH authentication methods and AAA authentication methods serve different purposes.
- Public-key authentication does not automatically grant privileged EXEC access.
- SSH negotiates separate algorithms for key exchange, server authentication, encryption, and integrity protection.
- SSH transport algorithm selection generally follows the client's preference order among mutually supported algorithms.
- The RSA key size and the Diffie-Hellman modulus size are independent.
- RSA keys can be used with SHA-2 signature algorithms without using the legacy `ssh-rsa` SHA-1 signature algorithm.
- The `hostkey` algorithm category controls server authentication.
- The `publickey` algorithm category controls user public-key signature algorithms.
- The `authentication` algorithm category controls permitted user authentication methods.
- SSH client and server algorithm policies can be configured independently.
- Removing an algorithm from the default list does not necessarily mean that the algorithm is unsupported.
- The `ip ssh time-out` command controls SSH negotiation and authentication timeout, not established EXEC session inactivity.
- The `ip ssh authentication-retries` command controls retries after an initial authentication attempt.
- SSH rekeying refreshes session key material without replacing the persistent host key pair.
- Time-based SSH rekeying is configured in minutes.
- Volume-based SSH rekeying is configured in kilobytes.
- VTY access classes, MPP, CoPP, and AAA provide complementary management security functions.
- `show ip ssh` displays the configured SSH policy.
- `show ssh` displays operational information about active SSH connections.

## References

- [Cisco IOS XE 17.x: Secure Shell Version 2 Support](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-secure-shell-v2-0.html)
- [Cisco IOS XE: Configuring SSH User Authentication Methods](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-usr-ssh-config-auth-xe.html)
- [Cisco IOS XE 17.x: SSH Algorithms for Common Criteria Certification](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-secure-shell-algorithm-ccc.html)
- [Cisco IOS XE: Secure Shell Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/sec-crypto/ssh/secure-shell-configuration-guide/m-secure-shell.html)
- [Cisco IOS Security Command Reference: SSH Commands](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/security/d1/sec-d1-cr-book/sec-cr-i3.html)
- [RFC 4252: The Secure Shell (SSH) Authentication Protocol](https://www.rfc-editor.org/rfc/rfc4252)
- [RFC 4253: The Secure Shell (SSH) Transport Layer Protocol](https://www.rfc-editor.org/rfc/rfc4253)
- [RFC 8332: Use of RSA Keys with SHA-256 and SHA-512 in SSH](https://www.rfc-editor.org/rfc/rfc8332)
