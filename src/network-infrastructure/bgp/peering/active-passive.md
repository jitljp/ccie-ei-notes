# 1.5.a (ii) Active and Passive

BGP uses **TCP port 179** to establish neighbor relationships.

When two BGP routers establish a session, one router must initiate the TCP connection, while the other accepts it.

These are called **active** and **passive** TCP connection establishment.

- **Active:** Initiates a TCP connection to the neighbor.
- **Passive:** Waits for the neighbor to initiate the TCP connection.

Cisco IOS XE allows the connection mode to be configured for individual BGP neighbors.

---

## Active and Passive TCP Connections

Consider two EBGP routers:

```text
       AS 65001                       AS 65002

          R1 ---------------------------- R2
        E0/0                            E0/0
     10.0.12.1/30                    10.0.12.2/30
```

Suppose R1 initiates the TCP connection.

```text
        R1 (Active)                   R2 (Passive)
             |                             |
             |-------- TCP SYN ----------->|
             |     Src Port: 49152         |
             |     Dst Port: 179           |
             |                             |
             |<------ TCP SYN-ACK ---------|
             |                             |
             |-------- TCP ACK ----------->|
             |                             |
             |     TCP Established         |
             |                             |
             |<------ BGP OPEN ----------->|
             |<---- BGP KEEPALIVE -------->|
             |                             |
             |     BGP Established         |
```

The active router uses:

- **Source TCP port:** An ephemeral port selected by the operating system.
- **Destination TCP port:** 179.

The passive router accepts the connection on TCP port 179.

Once TCP is established, both routers can exchange BGP messages in either direction.

**Active and passive describe TCP connection establishment, not the direction of BGP route advertisements.**

An active peer and a passive peer have the same routing capabilities after their BGP session reaches Established.

---

## Default BGP Connection Behavior

By default, BGP peers can generally initiate outgoing TCP connections and accept incoming TCP connections.

Therefore, it is normally unnecessary to configure active or passive connection modes explicitly.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002

R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
```

Either router may initiate the TCP connection.

The router that successfully initiates the TCP connection becomes the active side of that TCP connection, while the other is passive.

The **BGP router ID does not determine which router must initiate the connection**.

Router IDs are used to resolve simultaneous BGP connection attempts, as explained later.

---

## Configuring Active and Passive Modes

Cisco IOS XE supports the following command:

```text
R1(config-router)# neighbor <IP> transport connection-mode {active | passive}
```

| Mode | Function |
|---|---|
| `active` | Configures active TCP connection establishment toward the neighbor |
| `passive` | Waits for the remote peer to initiate the TCP connection |

### Active Mode

Configure R1 to initiate the TCP connection to R2:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# neighbor 10.0.12.2 transport connection-mode active
```

R1 actively attempts to establish a TCP connection with R2.

### Passive Mode

Configure R2 to wait for R1 to initiate the TCP connection:

```text
R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
R2(config-router)# neighbor 10.0.12.1 transport connection-mode passive
```

R2 does not initiate the TCP connection toward R1.

Instead, it accepts a connection initiated by R1.

### Restoring Default Behavior

Remove the configured connection mode:

```text
R1(config-router)# no neighbor 10.0.12.2 transport connection-mode
```

This restores the default TCP connection establishment behavior.

### Important Considerations

- **Both peers configured as passive cannot establish a session**, because neither initiates the TCP connection.
- At least one peer must initiate the TCP connection.
- **Both peers configured as active ALSO cannot establish a session**. One must be passive.
- Active and passive settings do not change the BGP ASN, NEXT_HOP, AS_PATH, or route advertisement rules.
- These settings apply to TCP session establishment, not to individual BGP address families or prefixes.

---

## BGP Connection Collisions

By default, both BGP peers may attempt to establish TCP connections simultaneously.

For example:

```text
               R1                         R2
                |                          |
                |------ TCP SYN ---------->|
                |<--------- TCP SYN -------|
                |                          |
                |    Two TCP connections   |
                |       may be formed      |
                |                          |
```

This is called a **BGP connection collision**.

BGP must select one connection and close the other.

### Connection Collision Resolution

RFC 4271 specifies that when a BGP connection collision occurs, the connection initiated by the router with the **higher BGP router ID** is retained.

For example:

```text
R1 BGP Router ID: 1.1.1.1
R2 BGP Router ID: 2.2.2.2
```

If both routers establish competing TCP connections:

1. R1 and R2 exchange BGP OPEN messages containing their router IDs.
2. Each router identifies the competing connection.
3. The router IDs are compared.
4. The connection initiated by R2 is retained because `2.2.2.2` is higher than `1.1.1.1`.
5. The competing connection initiated by R1 is closed.

```text
       R1 (RID 1.1.1.1)          R2 (RID 2.2.2.2)
                |                        |
                |---- Connection A ----->|
                |<---- Connection B -----|
                |                        |
                |   Collision detected   |
                |                        |
                |   Connection A closed  |
                |   Connection B kept    |
                |                        |
                |     BGP Established    |
```

**Important:** The higher router ID determines which connection is retained during a collision. It does not mean the higher router ID always initiates the first connection.

A connection collision does not necessarily prevent the BGP session from reaching Established.

---

## Active/Passive vs. BGP FSM States

Do not confuse **active/passive TCP connection modes** with the **BGP Finite State Machine (FSM)**.

BGP uses six FSM states:

1. Idle
2. Connect
3. Active
4. OpenSent
5. OpenConfirm
6. Established

The FSM **Active state** indicates that BGP is attempting to establish a TCP connection after connection establishment has not succeeded.

It does not mean the neighbor is configured with `transport connection-mode active`.

Similarly, a neighbor configured with `transport connection-mode passive` can reach the **Established** FSM state.

For example:

```text
R2(config-router)# neighbor 10.0.12.1 transport connection-mode passive
```

After R1 initiates a successful connection, R2 can reach:

```text
BGP state = Established
```

**TCP active/passive modes and BGP FSM states are separate concepts.**

See [BGP Messages and Neighbor States](../messages-states.md) for the complete FSM operation.

---

## Verification

### BGP Neighbor Status

```text
R1# show ip bgp summary
R1# show ip bgp neighbors 10.0.12.2
```

Verify that the BGP neighbor has reached Established.

The detailed neighbor output also displays TCP connection information, including local and foreign addresses and ports.

### TCP Connection Details

Use:

```text
R1# show tcp brief
```

For an established session initiated by R1, the output may resemble:

```text
Local Address           Foreign Address         (state)
10.0.12.1.49152         10.0.12.2.179           ESTAB
```

This indicates that R1 initiated the connection:

- R1 uses ephemeral source port `49152`.
- R2 uses TCP port `179`.

On R2, the same connection may appear as:

```text
Local Address           Foreign Address         (state)
10.0.12.2.179           10.0.12.1.49152         ESTAB
```

R2 is the passive side of the established TCP connection.

The TCP port numbers are an effective way to determine which router initiated the **currently established connection**.

### Detailed TCP Information

The `show tcp brief` output includes the TCP Control Block (TCB) identifier.

Use that identifier to examine the connection:

```text
R1# show tcp tcb <TCB>
```

Look for information such as:

```text
Local host: 10.0.12.1, Local port: 49152
Foreign host: 10.0.12.2, Foreign port: 179
Status Flags: active open
```

The `active open` flag indicates that the local router initiated this TCP connection.

### BGP Debugging

To examine BGP session establishment events:

```text
R1# debug ip bgp events
```

Disable debugging afterward:

```text
R1# undebug all
```

Use debugging carefully on production routers.

---

## Troubleshooting Active and Passive Peering

| Symptom | Possible cause |
|---|---|
| Both peers remain unestablished | Both configured as passive |
| Passive peer never receives a connection | Active peer cannot reach the passive peer or TCP traffic is filtered |
| TCP establishes but BGP does not | BGP parameter mismatch or protocol error |
| Session repeatedly resets | TCP connectivity, BGP timers, authentication, or other session issues |
| Duplicate connection attempts | Both peers initiating TCP connections |

When troubleshooting, verify:

1. At least one peer can initiate a TCP connection.
2. TCP port 179 is permitted by firewalls and ACLs.
3. Both routers have IP reachability to the correct peering addresses.
4. The TCP source IP address matches the neighbor address expected by the remote router.
5. The remote ASNs and any authentication settings are correct.

Useful commands:

```text
R1# show ip bgp summary
R1# show ip bgp neighbors 10.0.12.2
R1# show tcp brief
R1# show running-config | section router bgp
R1# debug ip bgp events
```

---

## Key Points

- BGP uses **TCP port 179**.
- The **active** TCP peer initiates the connection.
- The **passive** TCP peer waits for an incoming connection.
- The active peer normally uses an ephemeral source TCP port and destination port 179.
- IOS XE supports `neighbor <IP> transport connection-mode {active | passive}`.
- By default, BGP normally allows peers to initiate or accept TCP connections.
- **Two passive peers cannot establish a BGP session.**
- During a simultaneous connection collision, the connection initiated by the router with the **higher BGP router ID** is retained.
- The BGP FSM **Active state** is unrelated to the configured active TCP connection mode.
- Once Established, active and passive peers exchange BGP routes normally in both directions.

---

## Cisco Documentation

- [Cisco IOS BGP Command Reference: neighbor transport](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [Cisco IOS XE 17.x: BGP 4](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-bgp4.html)
- [RFC 4271: BGP-4, Section 6.8, Connection Collision Detection](https://www.rfc-editor.org/rfc/rfc4271.html#section-6.8)
