# LDP Authentication

This page covers **LDP Authentication**.

LDP authentication protects the **LDP TCP session** between LDP peers.

LDP uses two main communication channels:

```text
LDP discovery:
- UDP 646
- Hellos
- Finds potential LDP peers

LDP session:
- TCP 646
- Initialization, Keepalive, Address, Label Mapping, Notification messages
- Exchanges label information
```

LDP authentication applies to the **TCP session**, not to the UDP Hello discovery messages.

Cisco IOS XE uses TCP MD5 authentication for LDP. When authentication is enabled, the router computes an MD5 digest for TCP segments sent to the peer and checks the MD5 digest for TCP segments received from the peer.

```text
LDP Hellos discover the peer.
The LDP TCP session is authenticated with MD5.
Labels are exchanged only after the authenticated TCP session is established.
```

---

## LDP RID

LDP authentication commands identify peers by **LDP router ID**.

> This is the same as in session protection configuration.

Example LDP peer:

```text
Peer LDP Ident: 3.3.3.3:0
                ^^^^^^^
                LDP router ID
```

The `:0` is the label space ID.

The first part, `3.3.3.3`, is the LDP router ID.

That is the value you use in LDP authentication commands and ACLs.

---

## Basic Topology

This is the lab topology. I'll focus on P1-P2-P3 in this page and leave PE1 (1.1.1.1) and PE2 (2.2.2.2) out of the examples.
We'll look at them in the practice lab.

```text
            P3
           /  \
          /    \
 PE1 --- P1 --- P2 --- PE2
```

Loopbacks:

```text
P1 Loopback0: 2.2.2.2/32
P2 Loopback0: 3.3.3.3/32
P3 Loopback0: 5.5.5.5/32
```

LDP IDs:

```text
P1: 2.2.2.2:0
P2: 3.3.3.3:0
P3: 5.5.5.5:0
```

Assume LDP is already working.

```text
P1# show mpls ldp neighbor 
    Peer LDP Ident: 3.3.3.3:0; Local LDP Ident 2.2.2.2:0
        TCP connection: 3.3.3.3.30080 - 2.2.2.2.646
        State: Oper; Msgs sent/rcvd: 130/122; Downstream
        Up time: 01:31:17
        LDP discovery sources:
          Targeted Hello 2.2.2.2 -> 3.3.3.3, active, passive
          Ethernet0/1, Src IP addr: 10.1.2.2
        Addresses bound to peer LDP Ident:
          10.2.2.1        3.3.3.3         10.1.2.2        10.2.3.1 
!output omitted
```

Now I'll add authentication.

---

## Per-Neighbor Password

The simplest option is to configure a password for a specific LDP neighbor.

Command:

```text
mpls ldp neighbor <peer-ldp-router-id> password <password>
```

Example requirement:

```text
Authenticate the P1-P2 LDP session with password JITL.
```

P1:

```text
mpls ldp neighbor 3.3.3.3 password 0 JITL
```

P2:

```text
mpls ldp neighbor 2.2.2.2 password 0 JITL
```

The password must match on both routers.

This protects the LDP TCP session between P1 and P2.

---

### Authenticating an Established Session

Be careful with an LDP session that is already established without authentication.

If you add a per-neighbor password after the LDP session is already up, the existing session remains unauthenticated.

Example:

```text
P1(config)# mpls ldp neighbor 3.3.3.3 password JITL
```

This configures the password that P1 should use for peer `3.3.3.3`.

However, by itself, this does not tear down the existing unauthenticated LDP session and rebuild it with MD5 authentication.

If you need to force an established unauthenticated session to use authentication, configure a password source and then require authentication.

Example:

P1: 

```text
mpls ldp neighbor 3.3.3.3 password 0 JITL
mpls ldp password required
```

P2: 

```text
mpls ldp neighbor 2.2.2.2 password 0 JITL
mpls ldp password required
```

After authentication is required, LDP must use a password for the session.

If the existing session was unauthenticated, it must be reestablished with MD5 authentication.

More info on the `required` option in the next section:

---

### Require Passwords

Before configuring a password:

```text
P1# show mpls ldp neighbor 3.3.3.3 detail | i Password
        Password: not required, none, in use
```

After configuring a neighbor-specific password:

```text
P1(config)# mpls ldp neighbor 3.3.3.3 password JITL

P1# show mpls ldp neighbor 3.3.3.3 detail | i Password
        Password: not required, neighbor, stale
```

This indicates that a password is configured for the neighbor, but the current session is not using it.

If an LDP session is already established without authentication, configuring a neighbor-specific password does not automatically tear down the session and rebuild it with MD5 authentication.

The session remains up, but it is still unauthenticated.

To force LDP to use authentication, configure `mpls ldp password required`.

Example:

```text
P1(config)# mpls ldp password required
*Jun 15 01:52:30.129: %LDP-5-NBRCHG: LDP Neighbor 5.5.5.5:0 (1) is DOWN (Session's MD5 password changed)
*Jun 15 01:52:30.130: %LDP-5-NBRCHG: LDP Neighbor 3.3.3.3:0 (2) is DOWN (Session's MD5 password changed)
*Jun 15 01:52:31.491: %LDP-5-NBRCHG: LDP Neighbor 3.3.3.3:0 (4) is UP
*Jun 15 01:52:32.898: %LDP-4-PWD: MD5 protection is required for peer 5.5.5.5:0, no password configured
```

After `mpls ldp password required` is configured, P1 tears down the existing LDP sessions and attempts to reestablish them with authentication.

Because only P2 (`3.3.3.3`) also has a configured password for P1, only the P1-P2 session comes back up.

The other session fails because authentication is now required, but no password is configured for that peer.

Summary:

- Neighbor password configured:
  - Defines which password to use for a peer
  - Does not automatically authenticate an already established unauthenticated session

- `mpls ldp password required`:
  - Requires LDP sessions to use authentication
  - Tears down existing unauthenticated sessions
  - New sessions must be established with MD5 authentication

---

### Requiring Passwords Only for Selected Peers

Instead of requiring authentication for all LDP peers, you can require authentication only for selected peers.

Use the `for` keyword with a standard ACL:

```text
mpls ldp password required for <standard-acl>
```

The ACL matches the peer **LDP router ID**.

Example requirement:

```text
Require authentication only for the P1-P2 LDP session.
```

On P1, match only P2's LDP router ID:

```text
ip access-list standard LDP-AUTH-REQUIRED
 permit 3.3.3.3
!
mpls ldp neighbor 3.3.3.3 password 0 JITL
mpls ldp password required for LDP-AUTH-REQUIRED
```

This tells P1:

```text
Use password JITL for peer 3.3.3.3.
Require authentication only for peer 3.3.3.3.
```

The P1-P2 session must use MD5 authentication.

The P1-P3 session is not affected because `5.5.5.5` is not matched by the ACL.

On P2, configure the matching password for P1:

```text
ip access-list standard LDP-AUTH-REQUIRED
 permit 2.2.2.2
!
mpls ldp neighbor 2.2.2.2 password 0 JITL
mpls ldp password required for LDP-AUTH-REQUIRED
```

This tells P2:

```text
Use password JITL for peer 2.2.2.2.
Require authentication only for peer 2.2.2.2.
```

If the P1-P2 session was already established without authentication, the `password required for` command causes that affected session to be torn down and reestablished with MD5 authentication.

Unaffected sessions are not forced to restart.

Verification:

```text
P1# show mpls ldp neighbor 3.3.3.3 detail | i Password
        Password: required, neighbor, in use

P1# show mpls ldp neighbor 5.5.5.5 detail | i Password
        Password: not required, none, in use
```

`required, neighbor, in use` for `3.3.3.3` indicates that a password is required, the password comes from a neighbor-specific password command, and the current session is using it.

`not required, none, in use` for `5.5.5.5` indicates that a password is not required, no password is configured, and the current session is up without LDP authentication.

Key point:

```text
mpls ldp password required
= require authentication for all LDP peers

mpls ldp password required for <acl>
= require authentication only for peers matched by the ACL
```

---

## Fallback Password

A fallback password is a default LDP password.

Instead of configuring a separate password for each LDP neighbor, you can configure one fallback password that LDP can use when there is no more specific password configured.

Command:

```text
mpls ldp password fallback <password>
```

Example:

```text
P1(config)# mpls ldp password fallback 0 JITL
```

This tells P1:

```text
Use JITL as the default LDP password for peers that
do not have a more specific password configured.
```

Example:

```text
P1 has LDP peers P2 and P3.
P1 has a neighbor-specific password for P2.
P1 has no neighbor-specific password for P3.
P1 has fallback password FALLBACK.
```

In this case, P1 can use the fallback password for both LDP peers.

```text
P1(config)# mpls ldp password fallback JITL
P1(config)# mpls ldp password required
```

The `fallback` command defines the default password.

The `password required` command requires LDP sessions to use authentication.

So together:

```text
fallback password = which default password to use
password required = force LDP sessions to use a password
```

You can use a combination of neighbor-specific and fallback passwords.

Example:

P1:

```text
mpls ldp password required
mpls ldp password fallback FALLBACK
mpls ldp neighbor 3.3.3.3 password JITL
```

P2:
```text
mpls ldp neighbor 2.2.2.2 password JITL
```

P3:

```text
mpls ldp neighbor 2.2.2.2 password FALLBACK
```

The fallback password is useful when all LDP peers use the same shared password.

The downside is that all affected peers use the same password, so it is less granular than neighbor-specific password configuration.

---

## Password Options with ACLs

Password options let you assign different passwords to different groups of LDP peers.

Command:

```text
mpls ldp password option <number> for <acl> <password>
```

The ACL matches peer LDP router IDs.

The option number controls the order in which IOS XE evaluates the password options.

```text
Lower option number = checked first
Higher option number = checked later
```

Example requirement:

```text
Use password Pass12 for the P1-P2 LDP session.
Use password Pass13 for the P1-P3 LDP session.
Require authentication for both sessions.
```

P1:

```text
ip access-list standard LDP-P2
 permit 3.3.3.3
!
ip access-list standard LDP-P3
 permit 5.5.5.5
!
mpls ldp password option 10 for LDP-P2 Pass12
mpls ldp password option 20 for LDP-P3 Pass13
mpls ldp password required
```

P2:

```text
ip access-list standard LDP-P1
 permit 2.2.2.2
!
mpls ldp password option 10 for LDP-P1 Pass12
mpls ldp password required
```

P3:

```text
ip access-list standard LDP-P1
 permit 2.2.2.2
!
mpls ldp password option 10 for LDP-P1 Pass13
mpls ldp password required
```

P1 uses:

```text
Pass12 for peer 3.3.3.3
Pass13 for peer 5.5.5.5
```

P2 uses:

```text
Pass12 for peer 2.2.2.2
```

P3 uses:

```text
Pass13 for peer 2.2.2.2
```

---

## Password Selection Order

IOS XE determines the LDP session password in this order:

```text
1. Per-neighbor password
2. Password option commands, checked in ascending option number
3. Fallback password
4. No password
```

Example:

```text
mpls ldp neighbor 3.3.3.3 password NEIGHBOR-PW
mpls ldp password option 10 for LDP-CORE OPTION-PW
mpls ldp password fallback 0 FALLBACK-PW
```

For peer `3.3.3.3`, IOS XE uses:

```text
NEIGHBOR-PW
```

If there is no per-neighbor password, IOS XE checks the password options.

If no option matches, IOS XE checks the fallback password.

If no fallback password exists, no password is used unless `password required` prevents the session from forming.

---

## Important Difference: Password Config vs Password Requirement

These are separate ideas:

| Command Type | Purpose |
| :--- | :--- |
| `mpls ldp neighbor ... password ...` | Configures a password for one peer |
| `mpls ldp password option ...` | Configures a password for peers matched by an ACL |
| `mpls ldp password fallback ...` | Configures a default password |
| `mpls ldp password required` | Requires LDP to use a password |

A password can exist without `password required`.

Example:

```text
mpls ldp neighbor 3.3.3.3 password 0 JITLAB
```

This authenticates the session to `3.3.3.3`.

But peers without a matching password may still establish unauthenticated sessions.

If you want to prevent unauthenticated LDP sessions, use `password required`.

```text
mpls ldp password required
```

or restrict the requirement with an ACL:

```text
mpls ldp password required for LDP-AUTH-REQUIRED
```

---

---

## Password Rollover

LDP **password rollover** allows the MD5 authentication password to be changed **without tearing down the existing TCP/LDP session**.

When a password changes, the router can temporarily support both the **old password** and the **new password**, allowing the two LDP peers to transition without simultaneously changing their configurations.

Without rollover, changing the password on only one router can cause TCP MD5 authentication failures, disrupting the LDP session.

### Configuring Password Rollover

Command:

```text
mpls ldp password rollover duration <minutes>
```

Example:

```text
mpls ldp password rollover duration 10
```

- Configures a **10-minute delay** before a newly configured password becomes active for transmission.
- During this delay, the router continues using the old password for outgoing TCP segments.
- The rollover mechanism allows the new password to be introduced without immediately invalidating the existing session.
- The configured duration is **5–65535 minutes**.

The rollover duration gives administrators time to update the passwords on multiple routers before the new passwords are used.

### Example: Changing an LDP Password

Requirement:

```text
Change the P1-P2 LDP password from OLD-PW to NEW-PW.
Allow 10 minutes to update both routers.
Preserve the existing LDP session.
```

Assume:

- P1's LDP router ID is `2.2.2.2`.
- P2's LDP router ID is `3.3.3.3`.
- Both routers currently authenticate their LDP session using `OLD-PW`.

**Step 1: Configure the new password on P1.**

```text
P1(config)# mpls ldp password rollover duration 10
P1(config)# mpls ldp neighbor 3.3.3.3 password NEW-PW
```

P1 does not immediately start transmitting TCP segments authenticated with `NEW-PW`.

Instead, it continues using `OLD-PW` during the configured rollover delay.

**Step 2: Configure the new password on P2.**

```text
P2(config)# mpls ldp password rollover duration 10
P2(config)# mpls ldp neighbor 2.2.2.2 password NEW-PW
```

P2 also continues using `OLD-PW` during its rollover delay.

**Step 3: The passwords roll over.**

After each router's rollover delay expires:

- The router begins transmitting TCP segments authenticated with `NEW-PW`.
- The receiving peer can accept the new password during the transition.
- When TCP successfully authenticates a segment using the new password, it switches to that password and discards the old password.

The existing TCP/LDP session can remain established throughout the process.

### Rollover Timeline

| Stage | P1 | P2 |
|---|---|---|
| Initial | Uses OLD-PW | Uses OLD-PW |
| P1 reconfigured | Continues sending OLD-PW | Uses OLD-PW |
| P2 reconfigured | Continues sending OLD-PW | Continues sending OLD-PW |
| Rollover begins | Transitions to NEW-PW | Transitions to NEW-PW |
| Complete | Uses NEW-PW | Uses NEW-PW |

The timers are local to each router, so their rollover times might not be identical.

The temporary acceptance of the old and new passwords allows the transition to occur without requiring perfectly synchronized configuration changes.

### Keychain-Based Rollover

LDP also supports MD5 authentication using **keychains**.

With keychains, password transitions are controlled using:

- **send-lifetime:** Determines when a key is used to authenticate outgoing TCP segments.
- **accept-lifetime:** Determines when a key can authenticate incoming TCP segments.

By overlapping the accept lifetimes of successive keys, a router can accept **both old and new passwords** during a transition period.

This allows the peers to start transmitting with the new password at slightly different times.

For keychain-based rollover, the transition window is determined by the overlapping accept lifetimes rather than the `mpls ldp password rollover duration` timer.

### Verification

```text
show mpls ldp neighbor
show mpls ldp neighbor password
debug mpls ldp transport connections
```

Useful syslog messages include:

```text
%LDP-5-PWDCFG
%LDP-5-PWDRO
```

`PWDCFG` indicates a password configuration change, while `PWDRO` indicates a completed password rollover.

### Key Points

- Password rollover allows LDP authentication passwords to change **without resetting the TCP/LDP session**.
- `mpls ldp password rollover duration` delays activation of a newly configured password.
- During the delay, the router continues transmitting using the old password.
- During the rollover transition, TCP can accept both the old and new passwords.
- After successfully authenticating a segment using the new password, TCP switches to the new password.
- **Keychain-based rollover** uses overlapping accept lifetimes to accommodate password transitions.
- Password rollover is distinct from the normal LDP password-selection process, which does not automatically try alternative password options when authentication fails.

---

## Key Chains

IOS XE also supports LDP passwords through key chains with the newer lossless MD5 authentication behavior.

Key chains are useful for:

```text
Password rotation
Multiple keys
Timed key changes
Overlapping send and accept lifetimes
Reducing disruption during password changes
```

The concept is the same as OSPF and other keychain-based authentication.

Example key chain:

```text
key chain LDP-KEYS
 key 10
  key-string OLD-LDP-PW
  send-lifetime 00:00:00 Jan 1 2026 23:59:59 Jun 30 2026
  accept-lifetime 00:00:00 Jan 1 2026 23:59:59 Jul 7 2026
 key 20
  key-string NEW-LDP-PW
  send-lifetime 00:00:00 Jul 1 2026 infinite
  accept-lifetime 00:00:00 Jun 24 2026 infinite
```

Then reference the key chain from an LDP password option:

```text
ip access-list standard LDP-CORE
 permit 3.3.3.3
 permit 5.5.5.5
!
mpls ldp password option 10 for LDP-CORE key-chain LDP-KEYS
mpls ldp password required for LDP-CORE
```

Or use the key chain as a fallback password source:

```text
mpls ldp password fallback key-chain LDP-KEYS
mpls ldp password required
```

---

## Key Chain vs Rollover

There are two related but different methods for safer password changes.

| Method | Main Idea |
| :--- | :--- |
| Key chain | Use timed send and accept lifetimes for one or more keys |
| Password rollover duration | Delay the application of a new password for a configured number of minutes |

Key chains are more flexible.

Rollover is simpler.

---

## Key Points

* LDP authentication protects the LDP TCP session using the TCP MD5 authentication option.
* Configure a per-neighbor password: `mpls ldp neighbor <peer-rid> password <password>`
  * New sessions will require a matching password from the peer.
  * Established unauthenticated sessions will not be re-established and will remain unauthenticated.
* `mpls ldp password required` requires authentication for all peers.
  * Established unauthenticated sessions will be torn down and re-established with authentication.
  * Configure `mpls ldp password required for <standard-acl>` to require authentication only for specific peers.
* Configure a fallback password: `mpls ldp password fallback <password>`.
  * This is a default password used for peers without a more specific password configured.
* Configure password options to use different passwords for different peer groups.
  * `mpls ldp password option <number> for <acl> <password>`
  * Lower option numbers are checked first.
  * The ACL matches peer LDP router IDs.
* IOS XE selects the password in this order:
  * Per-neighbor password
  * Password options
  * Fallback password
  * No password
* Password rollover can delay when a new password takes effect.
  * `mpls ldp password rollover duration <minutes>`
* Key chains can be used for timed password changes.
  * `mpls ldp password option 10 for <acl>> key-chain <key-chain>`
  * or `mpls ldp password fallback key-chain <key-chain>`
* Verify authentication with `show mpls ldp neighbor detail`.
