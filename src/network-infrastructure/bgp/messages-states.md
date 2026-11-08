
# BGP Messages and Neighbor States

BGP uses **TCP port 179** to establish sessions and exchange routing information between peers.

BGP defines five message types:

| Type | Message | Purpose |
|---|---|---|
| 1 | **OPEN** | Establishes a BGP session and negotiates parameters |
| 2 | **UPDATE** | Advertises and withdraws routes |
| 3 | **NOTIFICATION** | Reports errors and terminates a BGP session |
| 4 | **KEEPALIVE** | Maintains the BGP session |
| 5 | **ROUTE-REFRESH** | Requests that a neighbor resend routing information |

The original BGP-4 specification defined four messages. ROUTE-REFRESH was added through RFC 2918.

BGP also uses a **Finite State Machine (FSM)** consisting of six states to manage the establishment and maintenance of neighbor relationships.

---

## BGP Message Header

Every BGP message begins with a **19-byte common header**.

```text
+-------------------------------------+
| Marker (16 bytes)                   |
+-------------------------------------+
| Length (2 bytes)                    |
+-------------------------------------+
| Type (1 byte)                       |
+-------------------------------------+
| Message-specific data (variable)    |
+-------------------------------------+
```

| Field | Size | Purpose |
|---|---|---|
| Marker | 16 bytes | Synchronization and message validation |
| Length | 2 bytes | Total BGP message length, including the header |
| Type | 1 byte | Identifies the BGP message type |

The Marker field is normally filled with all binary 1s (`0xFF`).

The standard maximum BGP message length is **4096 bytes**, although the Extended Message capability allows larger messages.

The minimum BGP message length is **19 bytes**.

---

## BGP OPEN Message

The **OPEN** message is the first BGP message exchanged after a TCP connection is established.

Both peers send an OPEN message to negotiate the parameters of the BGP session.

```text
R1                              R2
 |                               |
 |------- TCP connection ------->|
 |                               |
 |---------- OPEN -------------->|
 |<--------- OPEN ---------------|
 |                               |
```

The OPEN message contains:

| Field | Size | Purpose |
|---|---|---|
| Version | 1 byte | BGP version, normally 4 |
| My Autonomous System | 2 bytes | Sender's ASN, using AS_TRANS when necessary for 4-byte ASNs |
| Hold Time | 2 bytes | Proposed hold time |
| BGP Identifier | 4 bytes | Sender's BGP Router ID |
| Optional Parameters Length | 1 byte | Length of the optional parameters |
| Optional Parameters | Variable | Parameters such as BGP capabilities |

The minimum OPEN message length is **29 bytes**, including the common header.

### AS Number Validation

The receiver checks whether the ASN in the OPEN message matches the expected remote ASN.

For example:

```text
R1 local AS: 65001
R2 local AS: 65002
```

R1 expects R2's OPEN message to identify AS 65002.

If R2 instead identifies itself as AS 65003, R1 rejects the session.

The receiving router sends a NOTIFICATION message indicating a **Bad Peer AS** error.

### Hold Time Negotiation

Both peers advertise their configured hold times in their OPEN messages.

The negotiated hold time is the **lower of the two advertised values**.

For example:

```text
R1 advertised hold time: 180 seconds
R2 advertised hold time: 90 seconds

Negotiated hold time: 90 seconds
```

Both peers use the negotiated hold time.

A hold time of 0 disables the hold timer. Otherwise, the hold time must be at least 3 seconds.

### BGP Capabilities

BGP peers also exchange supported capabilities in their OPEN messages.

Common capabilities include:

| Capability | Purpose |
|---|---|
| Multiprotocol Extensions | Support additional address families, such as IPv6 unicast |
| Route Refresh | Request route advertisements without resetting the session |
| 4-Byte ASN | Support 32-bit autonomous system numbers |
| Graceful Restart | Preserve routing information temporarily during supported control-plane restarts |
| ADD-PATH | Advertise multiple paths for the same prefix when negotiated |

These capabilities determine which optional BGP features the peers can use.

Verify negotiated capabilities with:

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2
```

Look for the `Neighbor capabilities` section.

---

## BGP UPDATE Message

The **UPDATE** message advertises new routes, changes existing routes, and withdraws routes.

UPDATE messages are exchanged after the BGP session reaches the **Established** state.

An UPDATE message contains three main components:

| Component | Purpose |
|---|---|
| Withdrawn Routes | Identifies previously advertised routes being withdrawn |
| Path Attributes | Describes the characteristics of advertised routes |
| Network Layer Reachability Information (NLRI) | Identifies the prefixes being advertised |

### Advertising Routes

For example, R2 advertises `192.168.1.0/24` to R1.

```text
R2                              R1
 |                               |
 |------ UPDATE ---------------->|
 |                               |
 |  NLRI: 192.168.1.0/24         |
 |  AS_PATH: 65002               |
 |  NEXT_HOP: 10.0.12.2          |
 |  ORIGIN: IGP                  |
 |                               |
```

R1 receives the prefix together with its path attributes.

BGP evaluates the route and determines whether to accept it and whether it should become the best path.

An UPDATE can advertise multiple prefixes if they share the same path attributes.

### Withdrawing Routes

Suppose R2 previously advertised:

```text
192.168.1.0/24
```

The destination becomes unreachable.

R2 sends an UPDATE withdrawing the route.

```text
R2                              R1
 |                               |
 |------ UPDATE ---------------->|
 |                               |
 |  Withdrawn Routes:            |
 |  192.168.1.0/24               |
 |                               |
```

R1 removes the path learned from R2 from consideration and recalculates its best path if necessary.

If an alternative path exists, R1 may select it.

If no usable alternative exists, the destination is removed from BGP's selected routes.

### Multiprotocol UPDATE Messages

MP-BGP uses special path attributes to advertise and withdraw routes belonging to additional address families.

| Attribute | Purpose |
|---|---|
| `MP_REACH_NLRI` | Advertises reachable destinations and their next-hop information |
| `MP_UNREACH_NLRI` | Withdraws previously advertised destinations |

For example, IPv6 unicast routes are advertised through `MP_REACH_NLRI`, rather than the original IPv4-specific NLRI field.

### Incremental Updates

BGP does not periodically advertise its entire routing table.

When a session is first established, peers exchange their permitted routing information.

Afterward, UPDATE messages are sent when routing information changes.

```text
Initial session establishment
          |
          v
Initial route exchange
          |
          v
Incremental UPDATEs
          |
          +--- New route
          |
          +--- Changed path
          |
          +--- Withdrawn route
```

This reduces unnecessary control-plane traffic.

---

## BGP KEEPALIVE Message

**KEEPALIVE** messages maintain an established BGP session.

They let a peer know that the neighbor is still functioning even when no routing updates are being exchanged.

A KEEPALIVE message contains only the **19-byte BGP common header**.

It does not contain any routing information.

### Default Timers

Cisco IOS XE uses these default BGP timers:

| Timer | Default |
|---|---|
| Keepalive | 60 seconds |
| Hold Time | 180 seconds |

By default, a router normally sends a KEEPALIVE every 60 seconds when no other BGP messages have recently been sent.

```text
R1                              R2
 |                               |
 |---------- KEEPALIVE --------->|
 |                               |
 |          60 seconds           |
 |                               |
 |---------- KEEPALIVE --------->|
 |                               |
```

### Hold Timer Operation

The **Hold Timer** measures how long a router can go without receiving a valid BGP KEEPALIVE or UPDATE message.

For example:

```text
Negotiated hold time = 180 seconds
```

If R1 receives no KEEPALIVE or UPDATE message from R2 for 180 seconds, the hold timer expires.

R1 declares the BGP session down.

```text
No KEEPALIVE or UPDATE received
                |
                v
        Hold Timer expires
                |
                v
      Send NOTIFICATION
                |
                v
       Close TCP connection
                |
                v
         BGP session down
```

Receiving a KEEPALIVE or UPDATE resets the hold timer.

Therefore, **KEEPALIVE messages are not required if sufficiently frequent UPDATE messages are being received**.

### Configuring BGP Timers

Configure the keepalive and hold timers for the BGP process:

```text
R1(config)# router bgp 65001
R1(config-router)# timers bgp 30 90
```

This configures:

```text
Keepalive: 30 seconds
Hold Time: 90 seconds
```

Configure timers for a specific neighbor:

```text
R1(config-router)# neighbor 10.0.12.2 timers 10 30
```

Neighbor-specific timers override the process-wide timer settings.

The advertised hold times are negotiated during session establishment. Cisco adjusts the keepalive interval based on the negotiated hold time and configured values.

---

## BGP NOTIFICATION Message

The **NOTIFICATION** message reports a BGP error or session termination.

After sending a NOTIFICATION, the router closes the BGP connection.

A NOTIFICATION message contains:

| Field | Purpose |
|---|---|
| Error Code | Identifies the general error |
| Error Subcode | Provides more specific error information |
| Data | Additional information about the error, when applicable |

### NOTIFICATION Error Codes

| Error Code | Meaning |
|---|---|
| 1 | Message Header Error |
| 2 | OPEN Message Error |
| 3 | UPDATE Message Error |
| 4 | Hold Timer Expired |
| 5 | Finite State Machine Error |
| 6 | Cease |

Examples include:

- Receiving an OPEN message containing an unexpected remote ASN
- Receiving an OPEN message with an unacceptable hold time
- Receiving a malformed BGP message
- Hold timer expiration
- Administrative session shutdown

### Example: Incorrect Remote AS

Suppose R1 is configured with:

```text
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
```

But R2 uses AS 65003.

```text
R1                              R2
 |                               |
 |<--------- OPEN ---------------|
 |          AS 65003             |
 |                               |
 |-------- NOTIFICATION -------->|
 |          Bad Peer AS          |
 |                               |
 |------ TCP connection closed --|
```

R1 rejects the OPEN message because the ASN does not match the configured neighbor.

### Example: Hold Timer Expiration

If R1 does not receive a KEEPALIVE or UPDATE before the hold timer expires, it sends a NOTIFICATION with:

```text
Error Code: 4
Hold Timer Expired
```

The BGP session is then terminated.

NOTIFICATION messages are especially useful for troubleshooting because they can indicate why a BGP session was rejected or terminated.

---

## BGP ROUTE-REFRESH Message

The **ROUTE-REFRESH** message requests that a neighbor resend its routing information for a particular address family.

It is identified by BGP message type **5**.

Route refresh is particularly useful after modifying an inbound routing policy.

For example:

```text
R1                              R2
 |                               |
 |------- ROUTE-REFRESH -------->|
 |                               |
 |<--------- UPDATE -------------|
 |<--------- UPDATE -------------|
 |                               |
```

R1 requests fresh routing information from R2.

R2 responds by resending the applicable routes according to its current outbound policy.

R1 can then apply its current inbound policy to the received advertisements.

### Route Refresh Capability

Peers advertise support for route refresh in their OPEN messages.

Route refresh allows inbound policies to be reapplied **without resetting the BGP session**.

For example:

```text
R1# clear bgp ipv4 unicast 10.0.12.2 soft in
```

If route refresh has been negotiated, this requests a new advertisement of IPv4 unicast routes from the neighbor.

The session remains Established.

Route refresh is covered in greater detail in the soft reconfiguration and route refresh section.

---

# BGP Neighbor States

BGP uses a **Finite State Machine (FSM)** to establish and maintain neighbor relationships.

The FSM consists of six states.

| State | Description |
|---|---|
| **Idle** | BGP is waiting to begin or restart the connection process |
| **Connect** | BGP is waiting for TCP connection establishment |
| **Active** | BGP is attempting to establish a TCP connection after an earlier attempt was unsuccessful |
| **OpenSent** | TCP is established; BGP has sent OPEN and is waiting for the peer's OPEN |
| **OpenConfirm** | BGP has accepted the peer's OPEN and is waiting for KEEPALIVE |
| **Established** | BGP session is established; routing information can be exchanged |

A successful session commonly progresses through:

```text
          +----------+
          |   Idle   |
          +----+-----+
               |
               v
          +----------+
          | Connect  |<------+
          +----+-----+       |
               |             |
          TCP success     Retry timer
               |          expires
               |             |
               |       +-----+----+
               |       |  Active  |
               |       +-----+----+
               |             ^
               |             |
               |        TCP failure
               |             |
               +-------------+
               |
               v
          +-----------+
          | OpenSent  |
          +-----+-----+
                |
          Valid OPEN received
                |
                v
          +-------------+
          | OpenConfirm |
          +------+------+
                 |
         KEEPALIVE received
                 |
                 v
          +-------------+
          | Established |
          +-------------+
```

The exact transitions depend on TCP events, timers, received messages, and configuration.

---

## Idle State

**Idle** is the initial BGP neighbor state.

In this state, the router is waiting to begin or restart the BGP connection process.

When BGP starts the connection process, it initializes resources and moves toward establishing a TCP connection.

A neighbor may also return to Idle after a session failure or administrative reset.

### Troubleshooting Idle

If a neighbor remains in Idle, check:

- Whether the neighbor is correctly configured
- Whether the neighbor is administratively disabled
- Whether the router has a valid BGP Router ID
- Whether BGP is repeatedly resetting due to errors

Use:

```text
R1# show bgp ipv4 unicast summary
R1# show bgp ipv4 unicast neighbors 10.0.12.2
```

The neighbor details may show recent reset reasons or error information.

---

## Connect State

In the **Connect** state, BGP is attempting to establish a TCP connection with the configured neighbor.

The router waits for TCP's three-way handshake to complete.

```text
R1                              R2
 |                               |
 |------------ SYN ------------->|
 |<--------- SYN/ACK ------------|
 |------------ ACK ------------->|
 |                               |
 |      TCP Established          |
```

If TCP connection establishment succeeds, BGP sends an OPEN message and transitions to **OpenSent**.

If the connection attempt fails, BGP normally transitions to **Active**.

A router can also accept an incoming TCP connection initiated by its peer.

### Troubleshooting Connect

If a neighbor remains in Connect or alternates between Connect and Active, investigate TCP connectivity.

Common causes include:

- No route to the neighbor
- Incorrect neighbor IP address
- Interface or link failure
- ACL blocking TCP port 179
- Incorrect TCP source address
- Incorrect EBGP multihop configuration
- TTL expiration

Verify connectivity:

```text
R1# ping 10.0.12.2
R1# show ip route 10.0.12.2
R1# show tcp brief
```

For loopback-based peering, use an extended ping with the intended source address to verify reachability.

---

## Active State

In the **Active** state, BGP is attempting to establish a TCP connection after an unsuccessful connection attempt.

Despite its name, **Active does not mean that the BGP session is established or working**.

BGP continues trying to establish the session.

The ConnectRetry timer controls the retry process.

Cisco IOS uses a default **ConnectRetry timer of 120 seconds**.

When the timer expires, BGP can transition back to Connect and make another connection attempt.

```text
Connect
   |
   | TCP connection fails
   v
Active
   |
   | Retry timer expires
   v
Connect
```

A successful incoming or outgoing TCP connection can allow BGP to proceed directly to OpenSent.

### Active State vs. Active TCP Open

The term *Active* has two different meanings in BGP discussions.

| Term | Meaning |
|---|---|
| BGP Active state | FSM state in which BGP is trying to establish a session |
| Active TCP open | Initiating a TCP connection by sending a SYN |

An active TCP open can occur while BGP is in Connect or Active.

The peer accepting the incoming TCP connection performs a **passive TCP open**.

BGP's Active FSM state should not be confused with the active/passive TCP connection roles.

### Troubleshooting Active

A neighbor repeatedly showing Active generally indicates a TCP connectivity problem.

Verify:

```text
R1# show bgp ipv4 unicast summary
R1# show ip route 10.0.12.2
R1# ping 10.0.12.2
R1# show tcp brief
```

An incorrect remote ASN usually causes a failure **after TCP establishment**, when the OPEN message is checked.

---

## OpenSent State

In the **OpenSent** state:

1. The TCP connection has been established.
2. The router has sent its BGP OPEN message.
3. The router is waiting for an acceptable OPEN message from its neighbor.

```text
R1                              R2
 |                               |
 |-------- TCP established ------|
 |                               |
 |---------- OPEN -------------->|
 |                               |
 |       R1: OpenSent            |
 |                               |
```

When R1 receives R2's OPEN message, it validates the parameters.

Important checks include:

- BGP version compatibility
- Expected remote ASN
- Valid BGP Router ID
- Acceptable hold time
- Supported and properly encoded optional parameters

If the OPEN message is accepted, R1 sends a KEEPALIVE and transitions to **OpenConfirm**.

If the OPEN message contains an unacceptable parameter, R1 sends a NOTIFICATION and terminates the session.

### Troubleshooting OpenSent

If a neighbor repeatedly reaches OpenSent but fails to progress, check:

- Remote ASN configuration
- BGP Router IDs
- Hold-time configuration
- BGP capabilities
- NOTIFICATION messages

Use:

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2
```

Look for session reset reasons and NOTIFICATION statistics.

---

## OpenConfirm State

In the **OpenConfirm** state:

1. TCP is established.
2. The router has received and accepted the peer's OPEN message.
3. The router has sent a KEEPALIVE.
4. The router is waiting for a KEEPALIVE from the peer.

```text
R1                              R2
 |                               |
 |---------- OPEN -------------->|
 |<--------- OPEN ---------------|
 |                               |
 |---------- KEEPALIVE --------->|
 |                               |
 |       R1: OpenConfirm         |
 |                               |
 |<--------- KEEPALIVE ----------|
 |                               |
 |       R1: Established         |
```

After receiving a valid KEEPALIVE, the router transitions to **Established**.

If the hold timer expires or an error occurs, the session is terminated.

### Troubleshooting OpenConfirm

If a session remains in OpenConfirm, check:

- Whether KEEPALIVE messages are being received
- Whether the TCP session remains operational
- Whether NOTIFICATION messages are being generated
- Whether the negotiated hold time is appropriate

---

## Established State

**Established** is the final BGP neighbor state.

In this state:

- The TCP connection is established.
- OPEN messages have been exchanged successfully.
- Initial KEEPALIVE messages have been received.
- The peers can exchange UPDATE and ROUTE-REFRESH messages.
- KEEPALIVE messages maintain the session when necessary.

```text
R1                              R2
 |                               |
 |        Established            |
 |                               |
 |<--------- UPDATE ------------>|
 |                               |
 |<--------- KEEPALIVE --------->|
 |                               |
 |<--------- UPDATE ------------>|
 |                               |
```

An Established session does not guarantee that the peers are exchanging routes.

For example, a session may be Established but exchange no prefixes because:

- No routes are being originated or redistributed.
- Outbound policies deny all advertisements.
- Inbound policies deny received routes.
- The required address family is not activated.

Therefore, **Established confirms the BGP session is operational, not that the expected routes are present**.

---

## Complete BGP Session Establishment

Consider this EBGP topology:

```text
       AS 65001                AS 65002

         R1 --------------------- R2
       E0/0                     E0/0
    10.0.12.1                10.0.12.2
```

A successful session establishment follows this general sequence:

```text
R1                               R2
 |                                |
 |---- TCP SYN ------------------>|
 |<--- TCP SYN/ACK ---------------|
 |---- TCP ACK ------------------>|
 |                                |
 |          TCP Established       |
 |                                |
 |---- BGP OPEN ----------------->|
 |<--- BGP OPEN ------------------|
 |                                |
 |---- BGP KEEPALIVE ------------>|
 |<--- BGP KEEPALIVE -------------|
 |                                |
 |          BGP Established       |
 |                                |
 |<--- BGP UPDATE --------------->|
 |                                |
 |<--- BGP KEEPALIVE ------------>|
 |                                |
```

At a high level:

1. BGP begins in Idle.
2. TCP connection establishment begins.
3. The peers exchange OPEN messages.
4. Each peer validates the other's OPEN message.
5. The peers exchange KEEPALIVE messages.
6. Both peers reach Established.
7. Routing information is exchanged using UPDATE messages.

---

## BGP Connection Collision

Both BGP peers can initiate a TCP connection.

If they initiate connections simultaneously, two TCP connections may temporarily exist between the same peers.

This is called a **BGP connection collision**.

BGP detects the collision and closes one connection.

The BGP Router IDs are used to determine which connection is retained during connection establishment.

Normally, the connection initiated by the router with the **higher BGP Router ID** is retained.

For example:

```text
R1 Router ID: 1.1.1.1
R2 Router ID: 2.2.2.2
```

If both initiate connections simultaneously, the connection initiated by R2 is preferred.

The other connection is closed.

This allows the peers to maintain one BGP session rather than two competing sessions.

---

## BGP Neighbor Verification

The most important command for checking neighbor states is:

```text
R1# show bgp ipv4 unicast summary
```

Example output:

```text
BGP router identifier 1.1.1.1, local AS number 65001

Neighbor        V    AS MsgRcvd MsgSent TblVer InQ OutQ Up/Down State/PfxRcd
10.0.12.2       4 65002      20      21      5   0    0 00:15:20            3
```

The important fields are:

| Field | Meaning |
|---|---|
| Neighbor | Neighbor IP address |
| V | BGP version |
| AS | Remote ASN |
| MsgRcvd | BGP messages received |
| MsgSent | BGP messages sent |
| InQ | Messages waiting to be processed |
| OutQ | Messages waiting to be sent |
| Up/Down | Time since the session last changed state |
| State/PfxRcd | Current FSM state or received prefix count |

### Interpreting State/PfxRcd

If the session is not Established, the column displays a state such as:

```text
Idle
Connect
Active
OpenSent
OpenConfirm
```

If the session is Established, the column normally displays the number of prefixes received for that address family.

For example:

```text
State/PfxRcd
           3
```

This indicates the session is Established and the neighbor has supplied three prefixes counted by the summary.

A value of `0` can still indicate an Established session.

### Detailed Neighbor Information

Use:

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2
```

This command provides:

- Current BGP state
- Remote ASN and Router ID
- Configured and negotiated timers
- Neighbor capabilities
- Message statistics
- TCP connection information
- Session uptime
- Last reset information

For example:

```text
BGP neighbor is 10.0.12.2, remote AS 65002, external link
  BGP version 4, remote router ID 2.2.2.2
  BGP state = Established, up for 00:15:20
  Last read 00:00:15, last write 00:00:12
  Hold time is 180, keepalive interval is 60 seconds
```

The timer values and message statistics are particularly useful when troubleshooting session resets.

---

## Troubleshooting BGP Neighbor States

The current BGP state can help identify where session establishment is failing.

| State | What to Investigate |
|---|---|
| Idle | Neighbor configuration, administrative state, repeated resets |
| Connect | TCP connectivity, routing, ACLs, source address, TTL |
| Active | Failed TCP connection attempts |
| OpenSent | OPEN message negotiation, ASN mismatch, Router ID, capabilities |
| OpenConfirm | KEEPALIVE reception, hold timer, session errors |
| Established | Route advertisement, policies, address-family activation, next-hop reachability |

### Common Problems

**Neighbor remains Active**

Check connectivity and TCP port 179.

```text
R1# show ip route 10.0.12.2
R1# ping 10.0.12.2
R1# show tcp brief
```

**Neighbor repeatedly resets after TCP establishment**

Check OPEN negotiation errors and NOTIFICATION messages.

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2
```

**Neighbor is Established but no routes are learned**

Check advertised and received routes and address-family configuration.

```text
R1# show bgp ipv4 unicast summary
R1# show bgp ipv4 unicast neighbors 10.0.12.2 routes
R1# show bgp ipv4 unicast neighbors 10.0.12.2 advertised-routes
```

### Debugging BGP

Cisco IOS XE supports BGP debugging commands such as:

```text
R1# debug ip bgp events
R1# debug ip bgp updates
```

These can provide information about neighbor state changes and route advertisements.

Use debugging carefully on production routers because it can generate substantial output.

Disable debugging when finished:

```text
R1# undebug all
```

---

## Key Points

- BGP uses five message types: OPEN, UPDATE, NOTIFICATION, KEEPALIVE, and ROUTE-REFRESH.
- Every BGP message has a 19-byte common header.
- OPEN messages negotiate BGP session parameters and capabilities.
- UPDATE messages advertise and withdraw routes.
- KEEPALIVE messages maintain the session when routing updates are not being exchanged.
- The default Cisco BGP keepalive interval is 60 seconds, and the default hold time is 180 seconds.
- The negotiated hold time is the lower of the two advertised values.
- NOTIFICATION messages report errors and terminate BGP sessions.
- ROUTE-REFRESH requests that a neighbor resend routes without resetting the session.
- The six BGP states are Idle, Connect, Active, OpenSent, OpenConfirm, and Established.
- Active does not mean the BGP session is established.
- OpenSent waits for the peer's OPEN message.
- OpenConfirm waits for the peer's KEEPALIVE message.
- Established indicates successful session establishment, but does not guarantee route exchange.
- `show bgp ipv4 unicast summary` is the primary command for quickly verifying BGP neighbor states.
