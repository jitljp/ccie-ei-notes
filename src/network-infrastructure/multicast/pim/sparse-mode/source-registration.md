# Source Registration

In traditional PIM Sparse Mode (PIM-SM), **Source Registration** allows the Rendezvous Point (RP) to learn about active multicast sources.

When a source begins sending multicast traffic, its directly connected PIM Designated Router (DR) encapsulates the multicast packets in **PIM Register messages** and unicasts them to the RP.

The Register process serves two purposes:

- **Source discovery:** Inform the RP that a particular source is sending to a multicast group.
- **Initial data delivery:** Allow the RP to forward multicast traffic along the shared tree before a native multicast path from the source to the RP has been established.

Once the RP no longer needs encapsulated packets, it sends a **PIM Register-Stop message** to the source-facing DR.

Source registration is used for traditional PIM-SM Any-Source Multicast (ASM). It is not required for Source-Specific Multicast (SSM) or Bidirectional PIM.

## Source Registration Process

Consider the following topology:

```text
      Source (S)
     10.10.10.10
          |
          R1 (FHR/DR)
          |
          R2
          |
        R3 (RP)
        3.3.3.3
          |
          R4 (LHR)
          |
       Receiver
      239.1.1.1
```

Assume:

- R1 is the source-facing PIM DR.
- R3 is the RP for `239.1.1.1`.
- R4 has already established a `(*,G)` shared-tree Join toward R3.
- All relevant routers have PIM Sparse Mode enabled.

### Step 1: Source Begins Transmitting

The source begins sending packets:

```text
Source IP:      10.10.10.10
Destination IP: 239.1.1.1
```

R1 receives the multicast packets on its source-connected interface.

Because R1 is the PIM DR for that LAN, it is responsible for registering the source with the RP.

R1 creates `(S,G)` state for:

```text
(10.10.10.10, 239.1.1.1)
```

R1 determines the RP associated with the group:

```text
Group: 239.1.1.1
RP:    3.3.3.3
```

### Step 2: DR Sends PIM Register Messages

R1 encapsulates the original multicast packets in **PIM Register messages** and unicasts them to R3.

```text
R1 ------------------------> R3 (RP)
         PIM Register
    Encapsulated multicast
            packet
```

Unlike PIM Hello and Join/Prune messages, Register messages use **unicast**, allowing them to reach the RP through ordinary unicast routing.

The intermediate routers do not need multicast forwarding state for the encapsulated stream.

### Step 3: RP Processes the Register

When R3 receives a valid Register message, it identifies the source and group from the encapsulated packet.

```text
Source: 10.10.10.10
Group:  239.1.1.1
```

Because R3 already has downstream `(*,G)` interest, it can decapsulate the packet and forward the original multicast traffic along the shared tree.

```text
Source → R1
            \
             \ PIM Register (unicast)
              \
               R3 (RP)
                  |
                  | Native multicast
                  v
                 R4
                  |
               Receiver
```

This allows the receiver to begin receiving traffic before the RP has established a native multicast path toward the source.

### Step 4: RP Joins the Source Tree

The RP establishes an `(S,G)` source tree toward the source so that it can receive multicast packets natively (not encapsulated in unicast PIM Register messages).

R3 performs an RPF lookup toward `10.10.10.10` and sends an `(S,G)` Join upstream.

```text
R3 → R2: Join(10.10.10.10,239.1.1.1)

R2 → R1: Join(10.10.10.10,239.1.1.1)
```

This establishes the source-to-RP path:

```text
Source → R1 → R2 → R3 (RP)
```

Once native multicast forwarding is established, the RP no longer needs Register-encapsulated copies of the data.

### Step 5: RP Sends Register-Stop

When the RP has successfully switched to native source-tree forwarding, it sends a **Register-Stop** message to R1.

```text
R3 (RP) --------------------> R1 (DR)
            Register-Stop
       (10.10.10.10,239.1.1.1)
```

R1 stops encapsulating multicast data packets in Registers.

However, **R1 continues forwarding the original multicast packets natively** toward downstream routers that have joined the `(S,G)` tree.

The final source-to-RP path is:

```text
Source → R1 → R2 → R3 (RP)
                       |
                       v
                      R4
                       |
                    Receiver
```

At this point:

- R1 forwards multicast traffic natively.
- R3 receives multicast traffic through its source-facing RPF interface.
- R3 forwards traffic along the shared tree toward R4.
- R1 suppresses further data Registers for the affected `(S,G)`.

> **Important:** Register-Stop does not mean that the multicast source should stop transmitting. It means that the DR should stop **encapsulating the source's packets in PIM Register messages**.

## PIM Register Message

PIM Register messages have the following characteristics:

| Field | Value |
|---|---|
| PIM Version | 2 |
| PIM Message Type | **1 (Register)** |
| Destination IP | RP address |
| IP Protocol / Next Header | 103 |
| Transport | Unicast |
| Payload | Original multicast packet |

A Register contains an outer unicast IP header, a PIM Register header, and the encapsulated multicast packet.

For example:

```text
Outer IPv4 Header
  Source:      R1 (DR)
  Destination: 3.3.3.3 (RP)
  Protocol:    PIM (103)
       |
       v
PIM Register Header
  Version: 2
  Type:    1
       |
       v
Inner IPv4 Header
  Source:      10.10.10.10
  Destination: 239.1.1.1
       |
       v
Multicast Payload
```

The **outer IP addresses** identify the DR and RP.

The **inner IP addresses** identify the original multicast source and group.

The RP uses the inner addresses to identify the `(S,G)` being registered.

### Register Message Flags

The PIM Register header contains two important flags:

| Flag | Name | Purpose |
|---|---|---|
| B | Border | Indicates that the Register was generated by a PIM multicast border router |
| N | Null-Register | Indicates a Register containing a dummy packet rather than multicast data |

The **N-bit** is particularly important for Register suppression and periodic probing.

## PIM Register-Stop Message

A **PIM Register-Stop** is sent by the RP to the DR to indicate that Register encapsulation is no longer needed for the specified source and group.

| Field | Value |
|---|---|
| PIM Version | 2 |
| PIM Message Type | **2 (Register-Stop)** |
| Source IP | RP address |
| Destination IP | DR address |
| Transport | Unicast |
| Payload | Multicast group and source addresses |

Register-Stop messages are typically sent in either of two situations:

1. **No downstream receivers:** The RP does not need the multicast traffic.
2. **Native forwarding established:** The RP already receives the traffic through the source tree.

### Example: Source Active, No Receivers

Consider:

```text
Source → R1 (DR) → R2 → R3 (RP)

No interested receivers
```

The source begins sending to `239.1.1.1`.

R1 sends a PIM Register to R3.

However, R3 has no interested receivers for the group.

Therefore, it has no reason to forward the multicast traffic along a shared tree.

R3 sends Register-Stop to R1:

```text
R1 → R3: Register(S,G)

R3 → R1: Register-Stop(S,G)
```

R1 stops Register encapsulation.

The source can continue transmitting, but the traffic is not forwarded throughout the PIM-SM network because there is no downstream interest.

**Source registration does not require any receivers to be present.**

This is an important difference from the receiver-driven shared-tree Join process.

## Register Suppression and Null-Register Messages

Register-Stop does not permanently disable source registration.

The DR maintains a **Register-Stop Timer** for the affected `(S,G)` state.

After receiving Register-Stop, the DR temporarily suppresses data Registers.

Before the timer expires, the DR sends a **Null-Register** to check whether Register suppression is still necessary.

### Null-Register Operation

A Null-Register is a PIM Register with the N-bit set.

Rather than encapsulating an actual multicast data packet, it contains a dummy IP header identifying the source and group.

For example:

```text
Outer IP:
  Source: R1
  Destination: 3.3.3.3

PIM Register:
  Type: 1
  N-bit: 1

Dummy Inner IP:
  Source: 10.10.10.10
  Destination: 239.1.1.1
```

This allows the RP to evaluate the registration state without receiving the original multicast payload.

### Null-Register Response

There are two possible outcomes.

**Case 1: Register suppression is still necessary**

For example, the RP is already receiving native multicast traffic or still has no downstream receivers.

```text
DR → RP: Null-Register(S,G)

RP → DR: Register-Stop(S,G)
```

The DR restarts its suppression timer and continues avoiding data Register encapsulation.

**Case 2: Register suppression is no longer necessary**

For example, the RP needs Register-encapsulated traffic again because its native source-tree path is no longer active.

```text
DR → RP: Null-Register(S,G)

RP: No Register-Stop response

DR: Register-Stop Timer expires

DR → RP: Data Registers resume
```

The DR resumes encapsulating multicast data after suppression expires.

**A Null-Register is a probe, not a periodic multicast data packet.**

## Register Timers

RFC 7761 defines two important default timer values:

| Timer | Default | Purpose |
|---|---|---|
| Register Suppression Time | 60 seconds | Base interval for suppressing data Registers after Register-Stop |
| Register Probe Time | 5 seconds | Interval reserved for probing the RP before suppression expires |

The actual suppression interval is randomized.

After receiving Register-Stop, the DR sets its Register-Stop Timer to:

```text
Random(0.5 × Suppression Time,
       1.5 × Suppression Time)
- Register Probe Time
```

With the defaults:

```text
Random(30, 90) - 5

= 25–85 seconds
```

When this timer expires, the DR sends a Null-Register and starts a 5-second probe timer.

If a new Register-Stop arrives, suppression is renewed.

If no Register-Stop arrives before the probe timer expires, the DR resumes data Register encapsulation.

### Example Timeline

Assume the randomized timer happens to be 55 seconds.

```text
Time 0:
  DR receives Register-Stop.
  Data Register encapsulation stops.

Time 55:
  DR sends Null-Register.
  Probe timer starts (5 seconds).

Time 56:
  RP replies with Register-Stop.
  DR renews suppression.

Later:
  Another Null-Register is sent before
  the new suppression period expires.
```

The randomized suppression period helps prevent synchronization between multiple DRs.

> **Note:** The 60-second Register Suppression Time is not a fixed interval at which Null-Registers are sent. The actual timing varies because the suppression timer is randomized.

## Register State at the DR

RFC 7761 defines four Register states, maintained per `(S,G)` at the source-facing DR.

| State | Meaning |
|---|---|
| NoInfo | No active registration information |
| Join | Register encapsulation is enabled |
| Prune | Register encapsulation is suppressed after Register-Stop |
| Join-Pending | A Null-Register has been sent; the DR is waiting to determine whether encapsulation should resume |

These states are separate from the normal upstream and downstream PIM Join/Prune state machines.

For example, a router can have active native `(S,G)` forwarding while its Register state is **Prune**.

This is normal after the RP has joined the source tree and sent Register-Stop.

## IOS XE Configuration

Source registration operates automatically when PIM Sparse Mode is enabled and a valid RP mapping exists.

No separate command is required to enable normal Register or Register-Stop messages.

### Basic Configuration

On the source-facing DR:

```text
R1(config)# ip multicast-routing

R1(config)# interface Ethernet0/0
R1(config-if)# ip pim sparse-mode

R1(config)# ip pim rp-address 3.3.3.3
```

The RP must also have multicast routing enabled and the relevant PIM interfaces configured.

All routers must use consistent group-to-RP mappings.

### Configure the Register Source Address

By default, the DR selects an appropriate local address as the source of its Register messages.

IOS XE allows this to be explicitly configured:

```text
R1(config)# interface Loopback0
R1(config-if)# ip address 1.1.1.1 255.255.255.255

R1(config)# ip pim register-source Loopback0
```

The outer Register packets now use `1.1.1.1` as their source address.

This is useful when a stable, routable address is preferred.

The RP must have a unicast route back to this address because Register-Stop messages are sent to the Register's outer source address.

### Limit the Register Rate

On IOS XE platforms supporting it, the following command limits how many Register messages a DR sends per second for each `(S,G)` entry:

```text
R1(config)# ip pim register-rate-limit 10
```

This limits Register transmission to **10 messages per second per `(S,G)`**.

By default, no explicit maximum rate is configured.

This can protect the DR and RP from excessive Register processing, particularly when high-rate multicast sources become active.

However, limiting Registers can cause some initial multicast packets to be dropped before native forwarding is established.

### Filter Register Messages at the RP

The RP can restrict which multicast sources are allowed to register.

For example, permit only source `10.10.10.10` to register for group `239.1.1.1`:

```text
R3(config)# ip access-list extended REGISTER-FILTER
R3(config-ext-nacl)# permit ip host 10.10.10.10 host 239.1.1.1
R3(config-ext-nacl)# deny ip any any

R3(config)# ip pim accept-register list REGISTER-FILTER
```

The extended ACL matches the **inner multicast source and group addresses**, not the outer DR and RP addresses.

If a Register is rejected by this filter, the RP sends Register-Stop to the originating DR.

Remove the filtering policy:

> **Note:** Register filtering controls registration at the RP. It is not equivalent to applying a data-plane multicast boundary or packet-filtering ACL throughout the network.

## PIM Register Tunnels

IOS XE automatically creates logical **PIM Register tunnels** when a group-to-RP mapping is learned. These tunnels are used for Register encapsulation and decapsulation.

There are two types:

| Tunnel Type | Purpose |
|---|---|
| **PIM Encap** | Encapsulates multicast packets in unicast PIM Register messages and sends them to the RP |
| **PIM Decap** | Decapsulates received PIM Register messages at the RP |

In a basic network with one RP:

- **Non-RP routers:** Typically have one **PIM Encap** tunnel pointing toward the RP.
- **RP:** Typically has two tunnels: one **PIM Encap** tunnel and one **PIM Decap** tunnel.

The RP has an Encap tunnel because it can also act as a source-facing DR for directly connected multicast sources. The Decap tunnel is used to receive and decapsulate Registers from other DRs.

Verify using:

```text
R1# show ip pim tunnel

Tunnel0
  Type  : PIM Encap
  RP    : 3.3.3.3
  Source: 1.1.1.1
```

On the RP:

```text
R3# show ip pim tunnel

Tunnel0
  Type  : PIM Encap
  RP    : 3.3.3.3*
  Source: 3.3.3.3

Tunnel1
  Type  : PIM Decap
  RP    : 3.3.3.3*
  Source: -
```

The `*` indicates that the router itself is the RP.

> **Note:** These tunnels are automatically created and do not appear in the running configuration. They are internal PIM interfaces, not manually configured GRE tunnels. Additional RP mappings can result in additional tunnels.

You can view the automatically generated config with `show derived-config`:

```
R1# show derived-config
...
!
interface Tunnel0
 description Pim Register Tunnel (Encap) for RP 99.99.99.99
 ip unnumbered GigabitEthernet1
 tunnel source GigabitEthernet1
 tunnel destination 99.99.99.99
!
interface Tunnel1
 description Pim Register Tunnel (Decap) for RP 99.99.99.99
 ip unnumbered Loopback0
 tunnel source Loopback0
 tunnel destination 99.99.99.99
!  
...
```

## Verification

### Verify the RP Mapping

On the source-facing DR:

```text
R1# show ip pim rp mapping
```

Confirm that the source's multicast group maps to the expected RP.

### Verify Source-Specific State

On the FHR:

```text
R1# show ip mroute 239.1.1.1
```

Look for an `(S,G)` entry associated with the directly connected multicast source.

For example, a simplified entry might show:

```text
(10.10.10.10,239.1.1.1)
  Incoming interface: Ethernet0/0
  RPF nbr: 0.0.0.0
```

The RPF neighbor is `0.0.0.0` because the source is directly connected.

On the RP:

```text
R3# show ip mroute 239.1.1.1
```

When receivers exist and the RP has joined the source tree, verify that the `(S,G)` entry has the expected incoming interface toward the source.

### Examine Register Tunnels

On supported IOS XE platforms:

```text
R1# show ip pim tunnel
```

PIM Register encapsulation uses a logical Register tunnel toward the RP.

This is an internal PIM mechanism, not a GRE tunnel that must be manually configured.

### Inspect Register Messages

Use a packet capture or PIM debugging to observe:

```text
DR → RP: Register (Type 1)

RP → DR: Register-Stop (Type 2)

DR → RP: Null-Register (Type 1, N-bit set)
```

For troubleshooting:

```text
R1# debug ip pim
```

Disable debugging afterward:

```text
R1# undebug all
```

The detailed output and available debug options vary by IOS XE platform.

## Common Source Registration Problems

| Problem | Possible Cause |
|---|---|
| DR never sends Registers | No valid RP mapping, incorrect DR election, or source not recognized as directly connected |
| Registers do not reach RP | Unicast connectivity or filtering problem |
| Register-Stop never reaches DR | RP lacks a return route to the Register's outer source address |
| RP receives Registers but traffic is not delivered | No receiver interest, incorrect shared-tree state, or multicast filtering |
| RP never establishes native `(S,G)` forwarding | RPF problem or missing upstream PIM state |
| Repeated data Registers | Register-Stop is missing, rejected, or suppression repeatedly expires |
| Source registration rejected | `ip pim accept-register` filtering |

A particularly important troubleshooting distinction is that **successful Register delivery does not guarantee multicast forwarding to receivers**.

The RP must also have valid downstream multicast interest and forwarding state.

## IPv6 and Other PIM Modes

IPv6 PIM-SM ASM uses the same general source-registration process, with IPv6 Register encapsulation and Register-Stop messages.

However, the register mechanism is not used in every PIM mode.

| Mode | Source Registration |
|---|---|
| PIM-SM ASM | Required for normal discovery of directly connected sources |
| PIM-SSM | Not used |
| Bidirectional PIM | Not used |
| PIM-DM | Not used |

SSM already identifies the source through receiver signaling, so receivers can build `(S,G)` trees directly.

Bidirectional PIM allows native multicast traffic to travel toward the RP without Register encapsulation.

## Key Points

- **Source registration allows the RP to discover active multicast sources** in PIM-SM ASM.
- The source-facing **PIM DR** encapsulates multicast packets in **PIM Register messages (Type 1)**.
- Register messages are **unicast to the RP** and contain the original multicast packet.
- The RP can decapsulate Register packets and forward the traffic down an existing shared tree.
- The RP normally establishes a native `(S,G)` path toward the source when downstream interest exists.
- **PIM Register-Stop (Type 2)** instructs the DR to stop Register encapsulation, not to stop forwarding native multicast traffic.
- The RP can send Register-Stop when there are **no receivers** or when it already receives native multicast traffic.
- **Null-Registers** use the N-bit to probe whether Register suppression should continue.
- The default **Register Suppression Time is 60 seconds**, with a **Register Probe Time of 5 seconds**.
- Register suppression is randomized, so Null-Register transmissions do not occur at a fixed 60-second interval.
- Use `ip pim register-source` to select the DR's outer Register source address.
- Use `ip pim register-rate-limit` to limit registration traffic and `ip pim accept-register` to filter registrations at the RP.