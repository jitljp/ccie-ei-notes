# PIM Hello Messages and Neighbors

**PIM Hello messages** are used to discover and maintain neighbor relationships between directly connected PIM routers.

When PIM is enabled on an interface, the router periodically sends Hello messages to discover other PIM-enabled routers on the same network segment.

Hello messages serve several purposes:

- **Neighbor discovery:** Identify other PIM routers on the local link.
- **Neighbor maintenance:** Detect when neighboring PIM routers become unreachable.
- **Designated Router (DR) election:** Exchange the information needed to elect a DR.
- **Capability advertisement:** Communicate supported PIM features and parameters.
- **Restart detection:** Identify when a neighboring PIM router has restarted.

PIM uses Hello messages in both Dense Mode and Sparse Mode, as well as other PIM operating modes.

## PIM Hello Message Operation

PIMv2 Hello messages use the following addresses and protocol information:

| Field | IPv4 | IPv6 |
|---|---|---|
| Destination IP | `224.0.0.13` | `FF02::D` |
| Destination group | All PIM Routers | All PIM Routers |
| IP Protocol / Next Header | 103 | 103 |
| TTL / Hop Limit | 1 | 1 |
| PIM Version | 2 | 2 |
| PIM Message Type | 0 (Hello) | 0 (Hello) |

For IPv4, the source address is normally the primary IPv4 address of the outgoing interface. IPv6 PIM Hellos use the interface's link-local IPv6 address.

Because Hello messages use a TTL or Hop Limit of 1, they are restricted to the local link and are not forwarded by routers.

> Not all PIM messages are link-local multicast messages. For example, PIM Register and Register-Stop messages are sent using unicast.

### Example: Neighbor Discovery

Consider two directly connected routers:

```text id="oq4ar2"
            10.0.12.0/24
       E0/0               E0/0
         |                  |
      R1 -------------------- R2
  10.0.12.1              10.0.12.2
```

Both routers have PIM enabled on Ethernet0/0.

R1 sends a Hello message:

```text id="wk8vz9"
Source IP:      10.0.12.1
Destination IP: 224.0.0.13
Protocol:       PIM (103)
Message Type:   Hello (0)
```

R2 receives the Hello and creates a PIM neighbor entry for R1.

Likewise, R2 sends its own Hello, allowing R1 to discover R2.

The resulting neighbor relationships are:

```text id="x534on"
R1:
PIM Neighbor: 10.0.12.2

R2:
PIM Neighbor: 10.0.12.1
```

Unlike OSPF or EIGRP, PIM does not establish a complex adjacency state machine or exchange routing databases.

**A PIM neighbor relationship is maintained by receiving valid Hello messages from the neighboring router.**

A router must have heard a Hello from a PIM neighbor before accepting certain control messages, such as PIM Join/Prune and Assert messages, from that neighbor.

## PIM Hello Message Format

PIMv2 Hello messages begin with the standard PIM header, followed by a variable number of **Type-Length-Value (TLV) options**.

```text id="v5yq2h"
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |  Ver  | Type  |   Reserved    |           Checksum            |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |          Option Type          |          Option Length        |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                         Option Value                          |
 |                             ...                               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |          Option Type          |          Option Length        |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                         Option Value                          |
 |                             ...                               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The standard PIM header is 4 bytes long.

Each Hello option contains:

- **Option Type (16 bits):** Identifies the Hello option.
- **Option Length (16 bits):** Length of the Option Value in bytes.
- **Option Value (variable):** Information carried by the option.

A Hello message can contain multiple TLVs.

### Important PIM Hello Options

| Type | Option | Purpose |
|---|---|---|
| 1 | Holdtime | Advertises how long the neighbor should be retained |
| 2 | LAN Prune Delay | Advertises propagation delay, override interval, and Join suppression capability |
| 19 | DR Priority | Advertises the router's priority for DR election |
| 20 | Generation ID (GenID) | Allows detection of PIM restarts |
| 21 | State-Refresh Capable | Advertises PIM Dense Mode State Refresh support |
| 22 | Bidirectional Capable | Advertises Bidirectional PIM support |
| 24 | Address List | Advertises additional interface addresses |

The most important options for basic neighbor operation are **Holdtime, DR Priority, and Generation ID**.

### Holdtime Option (Type 1)

The Holdtime option specifies how long a receiving router should retain the PIM neighbor without receiving another Hello.

The Holdtime is expressed in seconds.

Two special values are defined:

| Holdtime | Meaning |
|---|---|
| `0` | Immediately remove the PIM neighbor |
| `65535` (`0xFFFF`) | Never expire the neighbor based on the Holdtime timer |

A router may send a Hello with Holdtime 0 when PIM is being disabled on an interface or the interface's primary IP address is changing.

This allows neighboring routers to remove the adjacency immediately rather than waiting for the normal timer to expire.

### DR Priority Option (Type 19)

The DR Priority option advertises the router's priority for the **PIM Designated Router election**.

The default DR priority is **1**, and a higher value is preferred.

For example:

```text id="xd9ysk"
R1 DR Priority: 1
R2 DR Priority: 100
```

R2 is preferred as the DR, provided all routers on the link support priority-based DR election.

If any router does not advertise the DR Priority option, the election falls back to comparing primary IP addresses.

The complete election process is covered in [PIM Designated Router](designated-router.md).

### Generation ID Option (Type 20)

The **Generation ID (GenID)** is a randomly generated 32-bit value included in PIM Hello messages.

It is regenerated when PIM starts or restarts on an interface.

For example:

```text id="f1wurn"
R1 initially advertises:
Generation ID: 0xAABBCCDD

R1 restarts PIM.

R1 subsequently advertises:
Generation ID: 0x11223344
```

When R2 receives the new Generation ID, it recognizes that R1's PIM process has restarted.

R2 can then discard stale Hello information and update its neighbor state.

The GenID mechanism also helps trigger the refresh of relevant PIM control-plane state rather than waiting for normal periodic timers.

**GenID changes allow neighbors to detect a PIM restart even if the router's IP address remains unchanged.**

### LAN Prune Delay Option (Type 2)

The LAN Prune Delay option advertises:

- **Propagation Delay:** Expected time for control messages to propagate across the local link.
- **Override Interval:** Time window used when scheduling Join messages to override Prunes.
- **T-bit:** Indicates the router's capability to disable Join suppression (explicit tracking capability).

This information helps coordinate PIM Join/Prune behavior on multiaccess LANs.

The default propagation delay is 500 milliseconds, and the default override interval is 2500 milliseconds.

The negotiated LAN Prune Delay behavior is used when the relevant Hello option is supported by all PIM neighbors on the link.

Detailed Join suppression and Prune processing are covered in [PIM Join and Prune Messages](sparse-mode/join-prune.md).

## PIM Hello and Neighbor Timers

PIM Hello messages are sent periodically to maintain neighbor relationships.

The standard default timers are:

| Timer | Default |
|---|---|
| Hello Interval | 30 seconds |
| Hello Holdtime | 105 seconds |
| Triggered Hello Delay | Random delay of 0–5 seconds |

The default Holdtime is calculated as:

```text id="fu7y75"
Holdtime = Hello Interval × 3.5

         = 30 × 3.5
         = 105 seconds
```

The Holdtime is advertised in the Hello message, and the receiving router uses it to maintain its neighbor expiration timer.

### Example: Neighbor Expiration

Suppose R1 receives a Hello from R2 at time 0.

R2 advertises a Holdtime of 105 seconds.

```text id="vfio54"
Time 0:
R1 receives Hello from R2.
Neighbor expiration timer = 105 seconds

Time 30:
R1 receives another Hello.
Expiration timer resets to 105 seconds.

Time 60:
R1 receives another Hello.
Expiration timer resets to 105 seconds.

R2 stops sending Hellos.

Time 165:
105 seconds since the last Hello.
R1 removes R2 from its PIM neighbor table.
```

Notice that the Holdtime is refreshed whenever another valid Hello is received.

### Triggered Hello Messages

PIM routers do not always wait for the next periodic Hello interval.

When PIM is first enabled, the initial Hello is sent after a randomized delay of up to 5 seconds.

Similarly, receiving a Hello from a previously unknown neighbor or detecting a new GenID can trigger a response Hello.

This helps new or restarted routers discover their neighbors without waiting for the full periodic Hello interval.

## Configuring PIM Neighbors

PIM neighbor discovery is enabled automatically when PIM is configured on an interface.

No separate command is required to enable Hello messages.

### IPv4 Configuration

Using the earlier two-router topology:

```text id="zizwkc"
R1(config)# ip multicast-routing
R1(config)# interface Ethernet0/0
R1(config-if)# ip address 10.0.12.1 255.255.255.0
R1(config-if)# ip pim sparse-mode
```

```text id="odau6n"
R2(config)# ip multicast-routing
R2(config)# interface Ethernet0/0
R2(config-if)# ip address 10.0.12.2 255.255.255.0
R2(config-if)# ip pim sparse-mode
```

Once both interfaces are operational, R1 and R2 exchange PIM Hello messages and discover each other.

The basic neighbor relationship does not require an RP to be configured. An RP is required later for traditional PIM-SM ASM operation.

To disable PIM on an interface:

```text id="mrxizj"
R1(config-if)# no ip pim sparse-mode
```

This stops PIM operation, including Hello transmission, on that interface.

### Changing the IPv4 Hello Interval

On IOS XE, configure the PIM Hello interval using:

```text id="1uuefq"
R1(config-if)# ip pim query-interval 10
```

This changes the Hello interval to 10 seconds.

The normal advertised Holdtime is correspondingly calculated as:

```text id="wlt4j1"
10 × 3.5 = 35 seconds
```

The command supports the optional `msec` keyword on platforms that support millisecond intervals:

```text id="1rndbd"
R1(config-if)# ip pim query-interval 500 msec
```

This sets the Hello interval to 500 milliseconds.

To restore the default interval:

```text id="s25zep"
R1(config-if)# no ip pim query-interval
```

> **Note:** The name `query-interval` is historical. PIMv1 neighbor discovery messages were called Queries, while PIMv2 uses Hello messages. Do not confuse this command with `ip igmp query-interval`, which controls IGMP Queries.

PIM neighbors do not need identical Hello intervals to form a neighbor relationship. Each router advertises its own Holdtime, which the receiver uses for that neighbor.

However, a shorter Holdtime causes neighbors to expire more quickly if Hellos are lost.

## PIM Passive Mode

On interfaces connected only to multicast receivers or sources, PIM neighbor discovery may be unnecessary.

IOS XE supports **PIM passive mode**, which allows the interface to participate in multicast routing without establishing PIM neighbor relationships.

Configure:

```text id="06lb2b"
R1(config)# interface Ethernet0/1
R1(config-if)# ip pim sparse-mode
R1(config-if)# ip pim passive
```

In passive mode:

- The router does not send PIM Hello messages.
- The router does not accept PIM control messages from neighboring routers on the interface.
- The router assumes it is the only PIM router on the link.
- IGMP operation is unaffected.

This is useful on a receiver-facing LAN with only one multicast router.

**Do not configure PIM passive mode on a network containing multiple PIM routers.** Each router would assume it is the only router and could independently act as DR, causing duplicate traffic or other forwarding problems.

Remove passive mode with:

```text id="qzax7b"
R1(config-if)# no ip pim passive
```

## PIM Neighbor Filtering

IOS XE also supports filtering PIM neighbors using an ACL.

This can be useful when only specific routers should be allowed to participate in PIM on a multiaccess network.

For example, suppose R1 should accept PIM control messages from R2 (`10.0.12.2`) but not from other routers.

Configure a standard ACL:

```text id="ovrfml"
R1(config)# ip access-list standard PIM-NEIGHBORS
R1(config-std-nacl)# permit host 10.0.12.2
R1(config-std-nacl)# deny any
```

Apply it to the PIM interface:

```text id="3n17fb"
R1(config)# interface Ethernet0/0
R1(config-if)# ip pim neighbor-filter PIM-NEIGHBORS
```

The ACL identifies permitted PIM neighbors.

PIM control traffic from disallowed neighbors is filtered, preventing those neighbors from participating normally in PIM on the interface.

Remove the filter using:

```text id="z4hqns"
R1(config-if)# no ip pim neighbor-filter PIM-NEIGHBORS
```

Unlike passive mode, neighbor filtering can selectively restrict PIM neighbors on a multi-router LAN rather than disabling PIM neighbor communication entirely.

## IPv6 PIM Neighbors

IPv6 PIM uses the same Hello-based neighbor discovery process as IPv4.

However, IPv6 PIM Hello messages use:

```text id="uv99p6"
Source:      Interface link-local IPv6 address
Destination: FF02::D
Hop Limit:   1
Next Header: 103
```

IPv6 PIM neighbors are identified by their link-local addresses.

Unlike IPv4 PIM, IPv6 PIM is enabled automatically when `ipv6 multicast-routing` is configured.

For example:

```
R1(config)# ipv6 multicast-routing
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 address 2001:DB8:12::1/64
```

Some IOS XE platforms automatically enable PIM on IPv6 multicast-routing interfaces, so the exact requirements depend on the platform.

### IPv6 Hello Interval

Configure the IPv6 PIM Hello interval using:

```text id="r3u5on"
R1(config-if)# ipv6 pim hello-interval 10
```

> IPv4 uses `query-interval`, and IPv6 uses `hello-interval`.

### IPv6 Routable Address Hello Option

IPv6 PIM also supports a **Routable Address Hello option**, which advertises additional addresses assigned to the interface.

This is useful because the PIM neighbor is normally identified by a link-local IPv6 address, while other multicast operations may require a routable address associated with that router.

The additional addresses can be displayed using:

```text id="kuqn32"
R1# show ipv6 pim neighbor detail
```

## Verification

### `show ip pim neighbor`

The primary command for verifying IPv4 PIM neighbors is:

```text id="hkid3x"
R1# show ip pim neighbor
```

Example output:

```text id="nocjs0"
PIM Neighbor Table
Mode: B - Bidir Capable, DR - Designated Router,
      N - Default DR Priority, P - Proxy Capable,
      S - State Refresh Capable, G - GenID Capable

Neighbor        Interface    Uptime/Expires    Ver   DR
Address                                            Prio/Mode
10.0.12.2       Ethernet0/0  00:05:24/00:01:32  v2    1 / S P G
```

Important fields include:

| Field | Description |
|---|---|
| Neighbor Address | IP address of the PIM neighbor |
| Interface | Local interface through which the neighbor was discovered |
| Uptime | Time since the neighbor relationship was established |
| Expires | Time remaining before the neighbor expires |
| Ver | PIM version |
| DR Prio | Neighbor's advertised DR priority |
| Mode | Advertised PIM capabilities and operational flags |

The `Expires` timer is reset when another valid Hello message is received.

Common capability flags include:

| Flag | Meaning |
|---|---|
| `DR` | Neighbor is the Designated Router |
| `B` | Bidirectional PIM capable |
| `S` | State Refresh capable |
| `G` | Generation ID capable |
| `P` | PIM proxy capable |
| `N` | Default DR priority indication |

The exact flags displayed depend on the platform, software version, and advertised capabilities.

### `show ip pim interface detail`

Use:

```text id="10ofga"
R1# show ip pim interface Ethernet0/0 detail
```

This displays operational information such as:

```text id="t2wkbw"
Ethernet0/0 is up, line protocol is up
  Internet address is 10.0.12.1/24
  PIM: enabled
    PIM version: 2, mode: sparse
    PIM DR: 10.0.12.2
    PIM neighbor count: 1
    PIM Hello/Query interval: 30 seconds
```

This is useful for verifying that PIM is enabled on the correct interface and checking the Hello interval and neighbor count.

### IPv6 Verification

Use:

```text id="qiz31f"
R1# show ipv6 pim neighbor
R1# show ipv6 pim neighbor detail
R1# show ipv6 pim interface
R1# show ipv6 pim traffic
```

The `show ipv6 pim traffic` command displays transmitted and received PIM message counters, including Hello messages.

## Key Points

- PIMv2 uses **Hello messages (Type 0)** to discover and maintain neighbors.
- IPv4 Hellos are sent to `224.0.0.13`; IPv6 Hellos are sent to `FF02::D`.
- PIM uses **IP protocol number 103**, with a TTL/Hop Limit of 1 for Hello messages.
- The default Hello interval is **30 seconds**, and the default Holdtime is **105 seconds**.
- Hello messages carry TLVs including Holdtime, DR Priority, Generation ID, and LAN Prune Delay.
- A Holdtime of `0` removes a neighbor immediately; `65535` prevents timer-based expiration.
- A changed **Generation ID** indicates that PIM has restarted on the neighboring interface.
- IPv4 Hello intervals are configured using `ip pim query-interval`; IPv6 uses `ipv6 pim hello-interval`.
- **PIM passive mode** disables PIM neighbor communication on an interface without disabling IGMP.
- **PIM neighbor filtering** can restrict which routers participate in PIM.
- Use `show ip pim neighbor` and `show ip pim interface detail` to verify IPv4 neighbor operation.
- PIM neighbor establishment does not guarantee successful multicast forwarding.