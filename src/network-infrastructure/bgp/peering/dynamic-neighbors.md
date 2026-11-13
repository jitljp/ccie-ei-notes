# 1.5.a (iv) Dynamic Neighbors

**BGP Dynamic Neighbors** allows a router to automatically establish BGP sessions with remote peers whose IP addresses fall within a configured subnet range.

Normally, BGP requires an explicit `neighbor` statement for each peer.

With dynamic neighbors, a router instead uses:

- A **listen range** specifying permitted neighbor IP addresses.
- A **BGP peer group** specifying the configuration inherited by dynamically created neighbors.

When an eligible remote router initiates a TCP connection, the listening router automatically creates a BGP neighbor using the peer group's configuration.

Dynamic neighbors are especially useful when a router must establish BGP sessions with many devices sharing similar configurations.

---

## Dynamic Neighbor Operation

Consider a router that must establish EBGP sessions with several remote routers.

### Traditional Static Neighbors

```text
                    AS 65000

                       R1
                  10.0.0.1/24
                       |
          +------------+------------+
          |            |            |
          R2           R3           R4
       10.0.0.2     10.0.0.3     10.0.0.4
       AS 65101     AS 65101     AS 65101
```

With traditional BGP, R1 must explicitly configure all three neighbors:

```text
R1(config)# router bgp 65000
R1(config-router)# neighbor 10.0.0.2 remote-as 65101
R1(config-router)# neighbor 10.0.0.3 remote-as 65101
R1(config-router)# neighbor 10.0.0.4 remote-as 65101
```

Adding another router requires another neighbor statement.

### Dynamic Neighbors

With BGP Dynamic Neighbors, R1 can accept connections from addresses within `10.0.0.0/24`.

```text
R1(config)# router bgp 65000
R1(config-router)# neighbor DYNAMIC-PEERS peer-group
R1(config-router)# neighbor DYNAMIC-PEERS remote-as 65101
R1(config-router)# bgp listen range 10.0.0.0/24 peer-group DYNAMIC-PEERS
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor DYNAMIC-PEERS activate
```

R1 no longer needs individual neighbor statements for R2, R3, and R4.

When R2 initiates a BGP connection:

1. R2 sends a TCP SYN to R1's TCP port 179.
2. R1 examines the source IP address, `10.0.0.2`.
3. R1 determines that the source matches `10.0.0.0/24`.
4. R1 dynamically creates a neighbor using the `DYNAMIC-PEERS` peer group.
5. The routers exchange BGP OPEN messages and negotiate the session.
6. If the configuration is compatible, the session reaches Established.

The same process applies to R3 and R4.

**Important:** BGP dynamic neighbors are not discovered through multicast Hello messages or periodic network scanning.

The remote router must initiate a TCP connection to trigger dynamic neighbor creation.

---

## Configuring Dynamic Neighbors

The primary command is:

```text
R1(config-router)# bgp listen range <PREFIX/LENGTH> peer-group <NAME>
```

For example:

```text
R1(config-router)# bgp listen range 10.0.0.0/24 peer-group DYNAMIC-PEERS
```

This associates `10.0.0.0/24` with the `DYNAMIC-PEERS` peer group.

The command does not create 254 neighbors immediately.

Instead, neighbors are created as eligible remote routers initiate BGP connections.

### Complete Configuration Example

```text
                    AS 65000

                       R1
                  E0/0: 10.0.0.1/24
                       |
                 Shared Ethernet
                       |
              +--------+--------+
              |                 |
              R2                R3
         E0/0: 10.0.0.2    E0/0: 10.0.0.3
           AS 65101          AS 65101
```

Assume all three routers share the same Layer 2 network.

### R1: Dynamic Neighbor Listener

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip address 10.0.0.1 255.255.255.0
R1(config-if)# no shutdown

R1(config)# router bgp 65000
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# no bgp default ipv4-unicast
R1(config-router)# neighbor DYNAMIC-PEERS peer-group
R1(config-router)# neighbor DYNAMIC-PEERS remote-as 65101
R1(config-router)# bgp listen range 10.0.0.0/24 peer-group DYNAMIC-PEERS
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor DYNAMIC-PEERS activate
```

### R2: Static Neighbor

```text
R2(config)# interface Ethernet0/0
R2(config-if)# ip address 10.0.0.2 255.255.255.0
R2(config-if)# no shutdown

R2(config)# router bgp 65101
R2(config-router)# bgp router-id 2.2.2.2
R2(config-router)# no bgp default ipv4-unicast
R2(config-router)# neighbor 10.0.0.1 remote-as 65000
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.0.1 activate
```

### R3: Static Neighbor

```text
R3(config)# interface Ethernet0/0
R3(config-if)# ip address 10.0.0.3 255.255.255.0
R3(config-if)# no shutdown

R3(config)# router bgp 65101
R3(config-router)# bgp router-id 3.3.3.3
R3(config-router)# no bgp default ipv4-unicast
R3(config-router)# neighbor 10.0.0.1 remote-as 65000
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# neighbor 10.0.0.1 activate
```

R2 and R3 explicitly configure R1 as their neighbor.

R1 dynamically creates neighbors when they initiate TCP connections.

### Configuration Requirements

| Requirement | Explanation |
|---|---|
| Listen range | Defines eligible remote peer source addresses |
| Peer group | Provides inherited BGP configuration |
| Remote AS | Specifies acceptable ASN values |
| Address-family activation | Enables exchange of routes for the chosen address family |
| IP reachability | TCP must be able to communicate between the peers |
| Remote initiation | The remote router must initiate a connection to the listening router |

A listen range by itself does not establish a BGP session.

---

## Dynamic Neighbor Peer Groups

Dynamically created neighbors inherit configuration from their associated **BGP peer group**.

For example:

```text
R1(config)# router bgp 65000
R1(config-router)# neighbor DYNAMIC-PEERS peer-group
R1(config-router)# neighbor DYNAMIC-PEERS remote-as 65101
R1(config-router)# neighbor DYNAMIC-PEERS timers 10 30
R1(config-router)# bgp listen range 10.0.0.0/24 peer-group DYNAMIC-PEERS
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor DYNAMIC-PEERS activate
R1(config-router-af)# neighbor DYNAMIC-PEERS send-community both
```

All dynamically created neighbors inherit:

- Remote AS 65101.
- Keepalive timer of 10 seconds.
- Hold timer of 30 seconds.
- IPv4 unicast activation.
- Standard and extended community advertisement.

Additional neighbor policies can be configured on the peer group.

For example:

```text
R1(config-router-af)# neighbor DYNAMIC-PEERS prefix-list FROM-PEERS in
R1(config-router-af)# neighbor DYNAMIC-PEERS route-map TO-PEERS out
R1(config-router-af)# neighbor DYNAMIC-PEERS maximum-prefix 1000
```

These configurations apply to dynamically created peer-group members.

The prefix list and route map must be defined separately.

See [BGP Peer Groups](peer-groups-templates/peer-groups.md) for more information about peer-group inheritance.

---

## Dynamic Neighbors with Multiple Remote ASNs

By default, the listen range peer group expects a particular remote ASN.

For example:

```text
R1(config-router)# neighbor DYNAMIC-PEERS remote-as 65101
```

R1 expects incoming BGP OPEN messages to identify AS 65101.

However, Cisco IOS XE supports the **`alternate-as`** option for listen range peer groups.

Syntax:

```text
R1(config-router)# neighbor <PEER-GROUP> remote-as <ASN> alternate-as <ASN> [...]
```

Cisco documents support for up to **five alternate ASNs**.

### Example

```text
                    AS 65000

                       R1
                   10.0.0.1
                       |
          +------------+------------+
          |            |            |
          R2           R3           R4
       10.0.0.2     10.0.0.3     10.0.0.4
       AS 65101     AS 65102     AS 65103
```

Configure R1:

```text
R1(config)# router bgp 65000
R1(config-router)# neighbor DYNAMIC-PEERS peer-group
R1(config-router)# neighbor DYNAMIC-PEERS remote-as 65101 alternate-as 65102 65103
R1(config-router)# bgp listen range 10.0.0.0/24 peer-group DYNAMIC-PEERS
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor DYNAMIC-PEERS activate
```

The results are:

| Router | Remote ASN | Accepted? |
|---|---:|---|
| R2 | 65101 | Yes |
| R3 | 65102 | Yes |
| R4 | 65103 | Yes |
| Another router | 65104 | No |

All permitted ASNs share the same peer-group configuration.

**`alternate-as` allows different remote ASNs to use the same dynamic neighbor listen range.**

Cisco documents this option specifically for listen range peer groups, rather than ordinary individual neighbor definitions.

---

## Limiting the Number of Dynamic Neighbors

Without a configured limit, Cisco IOS XE normally supports up to **100 dynamically created neighbors by default**.

Use:

```text
R1(config-router)# bgp listen limit <NUMBER>
```

For example:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp listen limit 20
```

This configures a maximum of 20 dynamic neighbors.

The documented configurable range is **1 to 5000**, subject to platform resources and feature support.

The limit applies to dynamically created peers, not ordinary statically configured BGP neighbors.

When the maximum number has been reached, additional dynamic neighbors cannot be created until capacity becomes available.

### Example

R1 has:

```text
R1(config-router)# bgp listen limit 2
R1(config-router)# bgp listen range 10.0.0.0/24 peer-group DYNAMIC-PEERS
```

Suppose R2 and R3 have already established dynamic sessions.

When R4 attempts to establish another session, R1 cannot create a third dynamic neighbor because the configured limit has been reached.

**The listen limit controls the number of dynamic neighbors, not the number of addresses in the listen subnet.**

Cisco recommends using appropriately restricted listen ranges and neighbor limits to avoid admitting unexpected peers.

---

## Multiple Listen Ranges

A BGP router can configure multiple listen ranges.

For example:

```text
R1(config)# router bgp 65000
R1(config-router)# neighbor PEERS-A peer-group
R1(config-router)# neighbor PEERS-A remote-as 65101
R1(config-router)# neighbor PEERS-B peer-group
R1(config-router)# neighbor PEERS-B remote-as 65102
R1(config-router)# bgp listen range 10.1.0.0/24 peer-group PEERS-A
R1(config-router)# bgp listen range 10.2.0.0/24 peer-group PEERS-B
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor PEERS-A activate
R1(config-router-af)# neighbor PEERS-B activate
```

The resulting behavior is:

| Source IP | Peer Group | Expected Remote AS |
|---|---|---:|
| 10.1.0.10 | PEERS-A | 65101 |
| 10.1.0.20 | PEERS-A | 65101 |
| 10.2.0.10 | PEERS-B | 65102 |
| 10.2.0.20 | PEERS-B | 65102 |

This allows R1 to support different groups of BGP neighbors with different session parameters and routing policies.

For predictable behavior, use nonoverlapping ranges when different groups require different policies.

---

## Dynamic Neighbors and Multihop Peering

Dynamic neighbors are not limited to directly connected peers.

For multihop EBGP sessions, the listen range must include the **source IP address used by the remote BGP peer**, and normal multihop requirements still apply.

For example:

```text
       AS 65000                                      AS 65101

          R1 ---------------- R2 --------------------- R3
      Lo0: 1.1.1.1                               Lo0: 3.3.3.3

        <------------ Multihop EBGP ----------------->
```

Assume routing already provides bidirectional reachability between the loopbacks.

### R1: Dynamic Listener

```text
R1(config)# router bgp 65000
R1(config-router)# neighbor DYNAMIC-PEERS peer-group
R1(config-router)# neighbor DYNAMIC-PEERS remote-as 65101
R1(config-router)# neighbor DYNAMIC-PEERS ebgp-multihop 3
R1(config-router)# bgp listen range 3.3.3.0/24 peer-group DYNAMIC-PEERS
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor DYNAMIC-PEERS activate
```

### R3: Static Neighbor

```text
R3(config)# router bgp 65101
R3(config-router)# neighbor 1.1.1.1 remote-as 65000
R3(config-router)# neighbor 1.1.1.1 update-source Loopback0
R3(config-router)# neighbor 1.1.1.1 ebgp-multihop 3
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# neighbor 1.1.1.1 activate
```

R3 initiates the TCP connection using source IP `3.3.3.3`.

R1 accepts the connection because `3.3.3.3` is within `3.3.3.0/24`.

The TTL and routing must permit TCP communication across the transit router.

**The listen range matches the remote TCP source address, not the BGP router ID or an advertised NLRI prefix.**

---

## IPv6 Dynamic Neighbors

Cisco IOS XE supports IPv6 dynamic neighbors.

IPv6 dynamic peering support was introduced in IOS XE Denali 16.3.

The configuration uses an IPv6 subnet with `bgp listen range`.

### Example

```text
                 AS 65000                 AS 65101

                    R1 --------------------- R2
                   E0/0                     E0/0
              2001:DB8:12::1/64       2001:DB8:12::2/64
```

### R1: Dynamic Listener

```text
R1(config)# ipv6 unicast-routing
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 address 2001:DB8:12::1/64
R1(config-if)# no shutdown

R1(config)# router bgp 65000
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# neighbor IPV6-PEERS peer-group
R1(config-router)# neighbor IPV6-PEERS remote-as 65101
R1(config-router)# bgp listen range 2001:DB8:12::/64 peer-group IPV6-PEERS
R1(config-router)# address-family ipv6 unicast
R1(config-router-af)# neighbor IPV6-PEERS activate
```

### R2: Static Neighbor

```text
R2(config)# ipv6 unicast-routing
R2(config)# interface Ethernet0/0
R2(config-if)# ipv6 address 2001:DB8:12::2/64
R2(config-if)# no shutdown

R2(config)# router bgp 65101
R2(config-router)# bgp router-id 2.2.2.2
R2(config-router)# neighbor 2001:DB8:12::1 remote-as 65000
R2(config-router)# address-family ipv6 unicast
R2(config-router-af)# neighbor 2001:DB8:12::1 activate
```

R1 dynamically creates the IPv6 neighbor when R2 initiates its connection.

Verify with:

```text
R1# show bgp ipv6 unicast summary
R1# show bgp ipv6 unicast neighbors 2001:DB8:12::2
```

The transport address family and the BGP NLRI address family are distinct concepts. IPv6-addressed BGP sessions can also carry other supported address families when appropriately configured.

---

## Dynamic Neighbors in VRFs

Cisco IOS XE also supports dynamic neighbors within VRFs.

The listen range can be configured at the BGP process level, with VRF-specific peer-group configuration and address-family activation.

For example, assume `CUST-A` already exists and contains the relevant interfaces:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp listen range 10.10.10.0/24 peer-group CUST-A-PEERS
R1(config-router)# address-family ipv4 unicast vrf CUST-A
R1(config-router-af)# neighbor CUST-A-PEERS peer-group
R1(config-router-af)# neighbor CUST-A-PEERS remote-as 65101
R1(config-router-af)# neighbor CUST-A-PEERS activate
```

A remote BGP neighbor initiating a connection from an eligible address within VRF `CUST-A` can be dynamically created using the VRF's peer-group configuration.

Verify using the appropriate VRF-aware BGP commands:

```text
R1# show bgp ipv4 unicast vrf CUST-A summary
R1# show bgp ipv4 unicast vrf CUST-A neighbors
```

The global routing table and each VRF have separate BGP routing contexts. Configure and verify the appropriate address family and VRF.

---

## Blocking Individual Dynamic Neighbors

Starting with **Cisco IOS XE 17.2.1**, Cisco supports excluding specific IP addresses from dynamic neighbor creation.

Use:

```text
R1(config-router)# bgp listen block <IP>
```

For example:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp listen range 10.0.0.0/24 peer-group DYNAMIC-PEERS
R1(config-router)# bgp listen block 10.0.0.50
```

R1 can dynamically accept eligible peers in `10.0.0.0/24`, except `10.0.0.50`.

### Important Behavior

- A blocked address cannot create a new dynamic BGP session.
- An existing dynamic session for that address is terminated when blocked.
- The block does **not** affect a statically configured BGP neighbor.

This permits static and dynamic BGP peers to coexist within the same listen range.

For example:

```text
R1(config)# router bgp 65000
R1(config-router)# neighbor DYNAMIC-PEERS peer-group
R1(config-router)# neighbor DYNAMIC-PEERS remote-as 65101
R1(config-router)# bgp listen range 10.0.0.0/24 peer-group DYNAMIC-PEERS
R1(config-router)# bgp listen block 10.0.0.50
R1(config-router)# neighbor 10.0.0.50 remote-as 65102
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor DYNAMIC-PEERS activate
R1(config-router-af)# neighbor 10.0.0.50 activate
```

Here:

- Most eligible peers are dynamically created under AS 65101.
- `10.0.0.50` is a static neighbor in AS 65102.

Remove the block:

```text
R1(config-router)# no bgp listen block 10.0.0.50
```

The command can also be used in supported VRF address-family contexts.

---

## Persistent Dynamic Neighbors

By default, dynamically created neighbor entries can be removed after their sessions terminate.

**Persistent Dynamic Neighbors** allows IOS XE to retain dynamically created neighbor information after a session goes down.

This feature was introduced in **IOS XE 17.13.1a**.

It is useful when operational features need neighbor state to be retained, including some maximum-prefix restart and Nonstop Forwarding scenarios.

### Per-Range Persistence

```text
R1(config)# router bgp 65000
R1(config-router)# bgp listen range 10.0.0.0/24 peer-group DYNAMIC-PEERS persistent 60
```

This retains dynamic neighbor state for **60 minutes** after the session leaves Established, subject to the feature's deletion behavior.

The configurable value is 1 to 65535 minutes.

### Indefinite Persistence

```text
R1(config-router)# bgp listen range 10.0.0.0/24 peer-group DYNAMIC-PEERS persistent
```

Omitting the timer makes the neighbors persistent indefinitely.

### Global Persistence

The feature can also be enabled globally:

```text
R1(config-router)# bgp listen persistent 60
```

**Persistence retains the dynamic neighbor's state. It does not keep a failed TCP/BGP session Established or guarantee route retention.**

Indefinite persistence should be used carefully because retained neighbors consume resources.

When using persistence with a maximum-prefix restart timer, the persistence timer must exceed the maximum-prefix restart timer unless indefinite persistence is configured.

---

## Dynamic Neighbors and Security

Dynamic peering introduces a security consideration: a router may accept BGP connections from addresses that have not been individually configured.

A source address matching the listen range does not by itself authenticate the remote router.

Recommended precautions include:

1. Use the narrowest practical listen ranges.
2. Configure an appropriate `bgp listen limit`.
3. Restrict TCP port 179 to authorized peers with infrastructure ACLs.
4. Enforce acceptable remote ASNs.
5. Apply inbound prefix filtering and maximum-prefix limits.
6. Use supported TCP authentication mechanisms where required.

For example, a broad listen range:

```text
R1(config-router)# bgp listen range 10.0.0.0/8 peer-group DYNAMIC-PEERS
```

could match a large number of potential source addresses.

A more restricted configuration is:

```text
R1(config-router)# bgp listen range 10.0.0.0/24 peer-group DYNAMIC-PEERS
R1(config-router)# bgp listen limit 10
```

The listen limit reduces resource exposure, but it is not a substitute for filtering or authentication.

### Prefix Filtering

Restrict which prefixes can be accepted from dynamic neighbors.

For example:

```text
R1(config)# ip prefix-list ALLOWED-PREFIXES permit 192.0.2.0/24
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor DYNAMIC-PEERS prefix-list ALLOWED-PREFIXES in
R1(config-router-af)# neighbor DYNAMIC-PEERS maximum-prefix 100
```

This permits `192.0.2.0/24` and applies a maximum-prefix limit of 100.

In a real deployment, tailor the prefix policy to the authorized remote networks.

---

## Dynamic Neighbors vs. Dynamic Update Groups

Do not confuse **BGP Dynamic Neighbors** with **BGP Dynamic Update Groups**.

| Feature | Purpose |
|---|---|
| Dynamic Neighbors | Automatically creates BGP neighbor sessions based on incoming connections from a configured address range |
| Dynamic Update Groups | Automatically groups established neighbors with compatible outbound UPDATE characteristics |

Dynamic neighbors simplify session configuration.

Dynamic update groups optimize BGP UPDATE generation and replication.

**Dynamic update groups do not dynamically discover or create BGP neighbors.**

See [BGP Peer Groups](peer-groups-templates/peer-groups.md) for additional details.

---

## Verification

### Display Dynamic Neighbors

```text
R1# show ip bgp summary
```

Example output:

```text
Neighbor       V     AS      MsgRcvd MsgSent   Up/Down  State/PfxRcd
*10.0.0.2      4     65101        10      12  00:05:00             5
*10.0.0.3      4     65101        10      12  00:05:00             5

* Dynamically created based on a listen range command
Dynamically created neighbors: 2/(100 max), Subnet ranges: 1

BGP peergroup DYNAMIC-PEERS listen range group members:
  10.0.0.0/24
```

The asterisk (`*`) identifies a dynamically created BGP neighbor.

The output also displays:

- Number of dynamically created neighbors.
- Maximum configured neighbor limit.
- Number of listen subnet ranges.
- Peer-group-to-subnet associations.

### Display Peer-Group Information

```text
R1# show ip bgp peer-group
R1# show ip bgp peer-group DYNAMIC-PEERS
```

These commands help verify peer-group membership and the configured listen ranges.

### Display Individual Dynamic Neighbor Details

```text
R1# show ip bgp neighbors 10.0.0.2
```

Look for information similar to:

```text
BGP neighbor is *10.0.0.2, remote AS 65101, external link
  Member of peer-group DYNAMIC-PEERS for session parameters
  Belongs to the subnet range group: 10.0.0.0/24
  BGP state = Established
```

### Display Configuration

```text
R1# show running-config | section router bgp
```

The running configuration contains the peer group and listen range configuration.

Dynamically created peers do not require individual neighbor statements in the running configuration.

### Debug Dynamic Neighbor Creation

```text
R1# debug ip bgp range
R1# debug ip bgp range detail
```

These commands can help identify:

- Incoming connections that match listen ranges.
- Dynamic neighbor creation.
- Blocked dynamic neighbors.
- Listen-range processing problems.

Additional BGP session debugging:

```text
R1# debug ip bgp events
```

Disable debugging afterward:

```text
R1# undebug all
```

---

## Troubleshooting Dynamic Neighbors

| Symptom | Possible Cause |
|---|---|
| No dynamic neighbor appears | Remote router did not initiate TCP |
| TCP connection rejected | Source IP outside listen range |
| Session rejected during OPEN exchange | Remote ASN not accepted |
| Additional peers fail to appear | `bgp listen limit` reached |
| Neighbor appears but stays Idle/Active | TCP, routing, or session-establishment problem |
| Session Established, no prefixes | Address family, policies, or route advertisement issue |
| Specific source cannot establish | Address excluded with `bgp listen block` |
| Neighbor disappears after failure | Normal dynamic peer deletion without persistence |

### Troubleshooting Checklist

1. Verify the listener's `bgp listen range`.
2. Confirm that a valid peer group exists and has a configured `remote-as`.
3. Check that the required address family is activated.
4. Confirm that the remote router explicitly configures the listener as a BGP neighbor.
5. Verify the source address used by the remote TCP connection.
6. Check bidirectional IP reachability and TCP port 179.
7. Verify the ASN against `remote-as` and `alternate-as`.
8. Check the dynamic neighbor limit and any blocked addresses.
9. Examine `show ip bgp summary`, `show ip bgp peer-group`, and `debug ip bgp range`.

**CCIE distinction:** An Established dynamic neighbor proves the BGP session is operating. It does not prove that routes are being advertised, accepted, or installed.

---

## Command Summary

| Command | Purpose |
|---|---|
| `bgp listen range PREFIX/LENGTH peer-group NAME` | Configure a dynamic neighbor listen range |
| `neighbor NAME peer-group` | Create the listen-range peer group |
| `neighbor NAME remote-as ASN` | Configure the expected remote ASN |
| `neighbor NAME remote-as ASN alternate-as ASN` | Permit additional remote ASNs |
| `bgp listen limit NUMBER` | Limit the number of dynamic neighbors |
| `bgp listen block IP` | Exclude an address from dynamic peering |
| `bgp listen persistent MINUTES` | Retain dynamic neighbor state globally |
| `bgp listen range PREFIX/LENGTH peer-group NAME persistent MINUTES` | Configure per-range persistence |
| `show ip bgp summary` | Display dynamic neighbors and listen-range counts |
| `show ip bgp peer-group NAME` | Display peer-group and listen-range membership |
| `show ip bgp neighbors IP` | Display detailed dynamic neighbor status |
| `debug ip bgp range` | Debug dynamic neighbor processing |

---

## Key Points

- **BGP Dynamic Neighbors** allows a router to accept BGP sessions from remote peers within configured IP address ranges.
- Dynamic neighbors are created when an eligible remote router initiates a **TCP connection to port 179**.
- The listening router does not need individual `neighbor IP` statements.
- Dynamic neighbors **inherit configuration from their associated peer group**.
- The primary command is `bgp listen range PREFIX/LENGTH peer-group NAME`.
- `alternate-as` permits additional remote ASNs for a listen range peer group.
- The default maximum is normally **100 dynamic neighbors**.
- `bgp listen limit` changes the maximum number of dynamic neighbors.
- IPv4 and IPv6 dynamic BGP peering are supported, including supported VRF contexts.
- `bgp listen block` excludes individual addresses from dynamic peering without affecting statically configured neighbors.
- `bgp listen persistent` retains dynamic neighbor state after sessions terminate.
- Dynamic neighbors and dynamic update groups are separate features.

---

## Cisco Documentation

- [Cisco IOS XE: BGP Dynamic Neighbors](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-bgp-dynamic-neighbors.html)
- [Cisco IOS BGP Command Reference: bgp listen](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-a1.html)
- [Cisco IOS XE: Configuring BGP Neighbor Session Options](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-neighbor-0.html)
