# MSDP Peering and Originator ID

**Multicast Source Discovery Protocol (MSDP)** uses explicitly configured TCP sessions to exchange information about active multicast sources between Rendezvous Points (RPs).

In an IPv4 Anycast RP deployment, MSDP allows multiple RPs sharing the same Anycast RP address to learn about sources registered with other RPs.

Two important addresses are involved:

- **MSDP peering address:** The unique IPv4 address used to establish the TCP session between MSDP peers.
- **MSDP Originator ID:** The IPv4 address placed in the RP Address field of locally originated Source-Active (SA) messages.

In an Anycast RP deployment, both addresses must uniquely identify the RP. They are commonly configured using the same loopback interface.

## MSDP Peering Overview

Consider the following topology:

```text
Src --- R1 --- RP1 --- RP2 --- R4 --- Rec
```

RP1 and RP2 share the same Anycast RP address but have unique MSDP addresses.

| Address | RP1 | RP2 |
|---|---|---|
| Anycast RP (Loopback0) | `10.0.0.100/32` | `10.0.0.100/32` |
| MSDP (Loopback1) | `10.0.0.1/32` | `10.0.0.2/32` |

The Anycast RP address is used by PIM routers for:

- PIM Register messages sent toward the RP.
- PIM `(*,G)` Joins sent toward the RP.

The unique MSDP addresses are used for:

- Establishing the MSDP TCP session.
- Identifying the originating RP in SA messages.

MSDP peering occurs between the unique addresses:

```text
RP1                                     RP2
10.0.0.1                               10.0.0.2
   |                                      |
   |------------ MSDP (TCP 639) ----------|
   |                                      |
```

**The shared Anycast RP address must not be used for MSDP peering.**

## MSDP Transport and Session Establishment

MSDP uses **TCP port 639**.

Unlike PIM, which discovers neighbors dynamically through Hello messages, MSDP peers are explicitly configured.

MSDP peers do not need to be directly connected. They can establish their TCP session across multiple routed hops, provided their addresses are reachable.

### TCP Connection Initiation

RFC 3618 defines which MSDP peer initiates the TCP connection.

- The peer with the **lower IPv4 address** actively initiates the TCP connection.
- The peer with the **higher IPv4 address** passively listens on TCP port 639.

For the example:

```text
RP1: 10.0.0.1
RP2: 10.0.0.2
```

RP1 has the lower address, so it initiates the connection:

```text
RP1                                  RP2
10.0.0.1                             10.0.0.2
   |                                    |
   |------------- TCP SYN ------------->|
   |<----------- TCP SYN-ACK -----------|
   |------------- TCP ACK ------------->|
   |                                    |
   |         MSDP Established           |
```

This deterministic connection establishment mechanism prevents both peers from simultaneously attempting to establish competing TCP sessions.

**TCP connection initiation does not determine the direction of SA advertisements.**

Once the session is established, either RP can send SA messages to the other.

### MSDP Session States

RFC 3618 defines the following MSDP connection states:

| State | Description |
|---|---|
| **DISABLED** | MSDP peering is not enabled |
| **INACTIVE** | The peer is waiting before attempting a TCP connection |
| **CONNECTING** | The peer is attempting to establish a TCP connection |
| **LISTEN** | The peer is waiting for an incoming TCP connection |
| **ESTABLISHED** | The TCP connection is established and MSDP messages can be exchanged |

The important operational state is **ESTABLISHED**, which IOS XE reports as an MSDP peer being **Up**.

A TCP session becoming established does not necessarily mean SA messages will be accepted. SA processing is also subject to MSDP Peer-RPF and any configured filters.

## Basic IOS XE MSDP Configuration

### RP1 Configuration

Configure the shared Anycast RP address:

```text
RP1(config)# interface Loopback0
RP1(config-if)# ip address 10.0.0.100 255.255.255.255
RP1(config-if)# ip pim sparse-mode
```

Configure the unique MSDP address:

```text
RP1(config)# interface Loopback1
RP1(config-if)# ip address 10.0.0.1 255.255.255.255
```

Configure the MSDP peer:

```text
RP1(config)# ip msdp peer 10.0.0.2 connect-source Loopback1
```

### RP2 Configuration

Configure the shared Anycast RP address:

```text
RP2(config)# interface Loopback0
RP2(config-if)# ip address 10.0.0.100 255.255.255.255
RP2(config-if)# ip pim sparse-mode
```

Configure the unique MSDP address:

```text
RP2(config)# interface Loopback1
RP2(config-if)# ip address 10.0.0.2 255.255.255.255
```

Configure the MSDP peer:

```text
RP2(config)# ip msdp peer 10.0.0.1 connect-source Loopback1
```

The MSDP peering configuration is therefore:

| Router | Command |
|---|---|
| RP1 | `ip msdp peer 10.0.0.2 connect-source Loopback1` |
| RP2 | `ip msdp peer 10.0.0.1 connect-source Loopback1` |

No separate global `ip msdp enable` command is required.

**Configuring an MSDP peer implicitly enables MSDP.**

These commands assume that IPv4 multicast routing, PIM Sparse Mode, Group-to-RP mappings, and the necessary unicast routing are already configured.

## The `ip msdp peer` Command

The IOS XE command syntax is:

```text
ip msdp peer <peer-address> [connect-source <interface>] [remote-as <as-number>]
```

| Parameter | Purpose |
|---|---|
| `peer-address` | IPv4 address of the remote MSDP peer |
| `connect-source` | Specifies the local interface used as the TCP connection source |
| `remote-as` | Optionally identifies the remote peer's AS number for display purposes |

### The `connect-source` Keyword

The `connect-source` keyword specifies the local interface whose primary IPv4 address is used as the source of the MSDP TCP connection.

For example:

```text
RP1(config)# ip msdp peer 10.0.0.2 connect-source Loopback1
```

The TCP connection uses:

```text
Source IP:      10.0.0.1
Destination IP: 10.0.0.2
Destination TCP Port: 639
```

Without `connect-source`, IOS XE selects a source address based on the outgoing interface used to reach the peer.

This can cause problems when the remote router is configured to recognize a different address as its MSDP peer.

For example, RP2 expects a connection from `10.0.0.1`.

If RP1 establishes the connection using the address of its outgoing physical interface instead, the source address will not match the configured peer address on RP2.

**Using `connect-source Loopback1` ensures that the TCP connection uses RP1's unique MSDP address.**

### The `remote-as` Keyword

The optional `remote-as` keyword identifies the AS number associated with the remote MSDP peer.

For example:

```text
RP1(config)# ip msdp peer 10.0.0.2 connect-source Loopback1 remote-as 65000
```

Unlike BGP's `remote-as` parameter, this value is **not required to establish the MSDP TCP session**.

On IOS XE, it is used for informational and display purposes.

It is not necessary in the basic Anycast RP configuration.

## MSDP Originator ID

Every MSDP Source-Active (SA) message contains an **RP Address field**, which identifies the RP that originated the advertisement.

**In an Anycast RP deployment, this field must NOT contain the shared Anycast RP address.** Instead, it must contain a unique address identifying the originating RP.

For example, RP1 and RP2 share the following Anycast RP address:

```text
10.0.0.100
```

However, their SA messages must use different RP addresses:

| Router | Anycast RP Address | SA RP Address |
|---|---|---|
| RP1 | `10.0.0.100` | `10.0.0.1` |
| RP2 | `10.0.0.100` | `10.0.0.2` |

If both RPs used `10.0.0.100` in their SA messages, the originating RP would not be uniquely identifiable. This can cause MSDP Peer-RPF checks to fail, resulting in rejected SA messages.

The Anycast RP address is used for PIM operations, while the unique Originator ID identifies the RP in MSDP SA messages.

**RFC 3446 requires that the shared Anycast RP address not be used as the RP address in SA messages.**

### Configure the Originator ID

IOS XE provides the following command:

```text
ip msdp originator-id <interface>
```

The router uses the specified interface's primary IPv4 address as the RP Address field in locally originated SA messages.

On RP1:

```text
RP1(config)# ip msdp originator-id Loopback1
```

On RP2:

```text
RP2(config)# ip msdp originator-id Loopback1
```

The resulting SA originator addresses are:

| RP | SA Originator ID |
|---|---|
| RP1 | `10.0.0.1` |
| RP2 | `10.0.0.2` |

### Example: SA Message Originated by RP1

Suppose Src begins transmitting:

```text
Source (S): 192.0.2.10
Group (G):  239.1.1.1
```

R1 sends PIM Register messages toward the Anycast RP address, reaching RP1.

RP1 originates an MSDP SA message:

```text
MSDP SA

RP Address: 10.0.0.1
Source:     192.0.2.10
Group:      239.1.1.1
```

The message is sent to RP2 over the MSDP TCP connection:

```text
RP1 --------------------------> RP2
         MSDP SA Message

TCP Source:      10.0.0.1
TCP Destination: 10.0.0.2

SA RP Address:   10.0.0.1
SA (S,G):        (192.0.2.10, 239.1.1.1)
```

Although the TCP source address and SA RP Address are the same in this example, they are conceptually different fields.

| Field | Purpose |
|---|---|
| TCP source address | Identifies the router sending the MSDP message over the TCP connection |
| SA RP Address | Identifies the RP that originally advertised the source |

In a larger MSDP topology, an intermediate MSDP speaker may forward an SA message originated by another RP.

In that case, the TCP source address changes to the forwarding peer, but the SA RP Address continues identifying the original RP.

### `connect-source` vs. `originator-id`

These two commands perform different functions:

```text
ip msdp peer 10.0.0.2 connect-source Loopback1
ip msdp originator-id Loopback1
```

- **`connect-source`:** Controls the source IPv4 address of the MSDP TCP connection.
- **`originator-id`:** Controls the RP Address field in locally originated SA messages.

The same unique loopback interface is normally used for both.

Configuring `connect-source` alone does not replace the need to explicitly configure a unique Originator ID in an Anycast RP deployment.

## MSDP Keepalive and Hold Timers

MSDP uses application-level Keepalive messages to maintain peer liveness.

These are separate from TCP's own keepalive mechanism.

RFC 3618 recommends the following values, which are also the default values documented for IOS XE:

| Timer | Default | Purpose |
|---|---|---|
| **Keepalive Interval** | 60 seconds | Maximum interval between outgoing MSDP messages when the connection is idle |
| **Hold Time** | 75 seconds | Maximum interval without receiving an MSDP message before declaring the peer down |

### Keepalive Interval

An MSDP peer sends a Keepalive message when it has not sent another MSDP message during the Keepalive Interval.

For example:

```text
RP1                         RP2
 |                           |
 |------ MSDP Keepalive ---->|
 |                           |
 |       60 seconds          |
 |                           |
 |------ MSDP Keepalive ---->|
```

However, the timer is reset whenever the router sends another MSDP message, such as an SA message.

Therefore, Keepalive messages are not necessarily transmitted exactly every 60 seconds.

### Hold Time

The Hold Timer tracks how long the peer has gone without receiving an MSDP message.

It is reset whenever a valid MSDP message is received.

If the Hold Timer expires, the peer is considered down and the MSDP connection is closed.

For example:

```text
Time 0:
  RP1 receives an MSDP message from RP2.
  Hold Timer = 75 seconds.

Time 30:
  RP1 receives another MSDP message.
  Hold Timer resets to 75 seconds.

Time 105:
  No additional MSDP messages received.
  Hold Timer expires.
  RP1 closes the MSDP connection.
```

The Hold Time must be greater than the Keepalive Interval.

### Configure MSDP Timers

The IOS XE command syntax is:

```text
ip msdp keepalive <peer-address> <keepalive-interval> <hold-time>
```

For example, configure a 30-second Keepalive Interval and 45-second Hold Time:

On RP1:

```text
RP1(config)# ip msdp keepalive 10.0.0.2 30 45
```

On RP2:

```text
RP2(config)# ip msdp keepalive 10.0.0.1 30 45
```

Cisco recommends configuring matching timer values on both sides of the MSDP peering session.

Shorter timers can provide faster detection of a peer failure, but they also reduce tolerance for control-plane delays.

In most deployments, the default values are appropriate.

## MSDP Peer Authentication

IOS XE supports **TCP MD5 authentication** for MSDP sessions.

Authentication requires both peers to use the same password.

The command syntax is:

```text
ip msdp password peer <peer-address> [encryption-type] <password>
```

For example, on RP1:

```text
RP1(config)# ip msdp password peer 10.0.0.2 0 MSDPpass123
```

On RP2:

```text
RP2(config)# ip msdp password peer 10.0.0.1 0 MSDPpass123
```

The `0` indicates that the password is being entered in plaintext. It does not disable TCP MD5 authentication.

If the passwords do not match, the MSDP TCP connection cannot be established successfully.

> **Important:** When configuring or changing an MSDP authentication password, an existing established connection is not necessarily reset automatically. Cisco documents that the connection must be manually cleared to activate the new password.

For example:

```text
RP1# clear ip msdp peer 10.0.0.2
```

The peers then attempt to reestablish their session using the configured authentication.

TCP MD5 is a legacy authentication mechanism. Support and any additional security requirements depend on the IOS XE platform and release.

## MSDP Peer Administration

### Configure a Peer Description

A description can be associated with an MSDP peer:

```text
RP1(config)# ip msdp description 10.0.0.2 Anycast RP2
```

This helps identify peers in the running configuration and verification output.

### Administratively Shut Down a Peer

Use:

```text
RP1(config)# ip msdp shutdown 10.0.0.2
```

This administratively disables the specified MSDP peering relationship.

The TCP connection is terminated, but the peer configuration is retained.

Reenable the peer:

```text
RP1(config)# no ip msdp shutdown 10.0.0.2
```

This is useful when testing MSDP behavior or temporarily disabling a peer without removing its configuration.

### Remove an MSDP Peer

To remove the peer configuration entirely:

```text
RP1(config)# no ip msdp peer 10.0.0.2
```

This removes the configured MSDP peering relationship rather than merely shutting it down.

## IOS XE Verification

### Verify MSDP Peer Status

Use:

```text
RP1# show ip msdp summary
```

Illustrative output:

```text
MSDP Peer Status Summary

Peer Address   AS  State  Uptime/  Reset SA    Peer Name
                            Downtime Count Count
10.0.0.2       ?   Up     00:12:30 0     1     ?
```

The important fields include:

| Field | Meaning |
|---|---|
| Peer Address | Remote MSDP peer address |
| AS | Remote peer AS information, if known |
| State | MSDP peer status |
| Uptime/Downtime | Time the session has been up or down |
| Reset Count | Number of MSDP connection resets |
| SA Count | SA entries associated with the peer |

The expected state is **Up**.

The SA Count does not need to be greater than zero for the peer to be operational. An established MSDP session can exist even when no active multicast sources have been advertised.

### Verify Detailed Peer Information

Use:

```text
RP1# show ip msdp peer 10.0.0.2
```

Illustrative excerpt:

```text
MSDP Peer 10.0.0.2
  Connection status:
    State: Up, Resets: 0, Connection source: Loopback1 (10.0.0.1)
    Uptime(Downtime): 00:12:30
```

Check that:

- The peer address is correct.
- The session state is **Up**.
- The connection source is `Loopback1 (10.0.0.1)`.
- The reset count is not continually increasing.
- MSDP message counters increase as expected.

The detailed output also provides information about configured authentication, timers, and SA filtering.

### Verify the TCP Connection

On supported IOS XE platforms:

```text
RP1# show tcp brief
```

Look for a TCP connection between:

```text
10.0.0.1 <---- TCP 639 ----> 10.0.0.2
```

The TCP connection should be established.

### Verify MSDP Configuration

```text
RP1# show running-config | include ip msdp
```

For the basic RP1 configuration, the important commands are:

```text
ip msdp peer 10.0.0.2 connect-source Loopback1
ip msdp originator-id Loopback1
```

### Verify SA Reception

On RP2:

```text
RP2# show ip msdp sa-cache
```

This can display learned source information, assuming SA caching is enabled and SA messages have been accepted.

For example, the expected source information is:

```text
Source: 192.0.2.10
Group:  239.1.1.1
RP:     10.0.0.1
Peer:   10.0.0.1
```

A functioning MSDP peer relationship does not guarantee that every SA message will be accepted.

Peer-RPF failures, SA filters, and other conditions can prevent SA information from being accepted or retained.

SA cache behavior is covered in [Source-Active Messages and SA Cache](sa-messages.md).

## Troubleshooting MSDP Peering

| Problem | Possible Cause |
|---|---|
| MSDP peer remains down | No unicast connectivity to the peer address |
| TCP connection cannot establish | TCP port 639 blocked by an ACL or firewall |
| Unexpected source IP | Missing or incorrect `connect-source` |
| Peer refuses connection | Remote peer has not configured the expected local address |
| Authentication failure | TCP MD5 passwords differ |
| Peer repeatedly resets | Keepalive/Hold Timer expiration, unstable routing, or TCP connection problems |
| Session Up but no SA messages | No active locally registered sources, SA filtering, or SA origination problems |
| Session Up but SA messages rejected | Peer-RPF failure or SA filtering |
| SA messages identify the shared Anycast RP address | Missing or incorrect `ip msdp originator-id` |

### Verify Unicast Reachability

From RP1, test reachability using the expected MSDP source address:

```text
RP1# ping 10.0.0.2 source 10.0.0.1
```

From RP2:

```text
RP2# ping 10.0.0.1 source 10.0.0.2
```

These tests verify IP reachability between the two unique MSDP addresses.

Successful ping tests do not guarantee that TCP port 639 is permitted, but failed ping tests indicate a likely underlying connectivity issue, assuming ICMP is permitted.

### Clear the MSDP Session

To manually reset the connection:

```text
RP1# clear ip msdp peer 10.0.0.2
```

This terminates the existing TCP session and allows the peers to establish a new connection.

It is useful after changing authentication or while troubleshooting connection establishment.

### MSDP Debugging

IOS XE provides MSDP debugging commands.

For example:

```text
RP1# debug ip msdp resets
```

This can help identify why MSDP sessions are being reset.

Disable debugging when finished:

```text
RP1# undebug all
```

## Key Points

- **MSDP uses TCP port 639** and requires explicitly configured peers.
- MSDP peers do not need to be directly connected.
- The peer with the **lower IPv4 address initiates** the TCP connection; the higher-addressed peer listens.
- `ip msdp peer <address>` configures an MSDP peer and implicitly enables MSDP.
- **`connect-source`** specifies the local interface whose address is used as the TCP source.
- In Anycast RP deployments, MSDP peers must use **unique IPv4 addresses**, not the shared Anycast RP address.
- **`ip msdp originator-id`** specifies the RP Address field in locally originated SA messages.
- The shared Anycast RP address must not be used as the SA Originator ID.
- The same unique loopback interface is commonly used for both `connect-source` and `originator-id`.
- The default MSDP Keepalive Interval is **60 seconds**, and the Hold Time is **75 seconds**.
- `ip msdp keepalive` modifies these timers for a specific peer.
- `ip msdp password peer` enables TCP MD5 authentication.
- `ip msdp shutdown` administratively disables a peer without removing its configuration.
- Use `show ip msdp summary` and `show ip msdp peer` to verify peering status.
- **An established MSDP session does not guarantee successful SA processing**, which also depends on Peer-RPF and filtering.

## References

- [Cisco IOS XE: Using MSDP to Interconnect Multiple PIM-SM Domains](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_imc_msdp_im_pim_sim.html)
- [Cisco IOS XE: Configuring Basic IP Multicast](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-multicast/ip-multicast/m_imc_basic_cfg-0.html)
- [RFC 3446: Anycast RP Using PIM and MSDP](https://www.rfc-editor.org/rfc/rfc3446)
- [RFC 3618: Multicast Source Discovery Protocol](https://www.rfc-editor.org/rfc/rfc3618)