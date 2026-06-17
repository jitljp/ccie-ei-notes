# LDP-IGP Synchronization

In a normal MPLS core, the IGP and LDP both run on the same core links.

That creates an important timing problem:

```text
The IGP adjacency can come up before LDP is ready.
The IGP can select a link before labels are exchanged.
Traffic can be sent over a link that is not ready for MPLS forwarding.
```

**LDP-IGP Synchronization** solves this by making the IGP avoid a link until LDP is synchronized on that link.

---

## The Problem

Topology:

```text
            P3
           /  \
          /    \
 PE1 --- P1 --- P2 --- PE2
```

Loopbacks:

```text
PE1 Loopback0: 1.1.1.1/32
P1  Loopback0: 2.2.2.2/32
P2  Loopback0: 3.3.3.3/32
P3  Loopback0: 4.4.4.4/32
PE2 Loopback0: 5.5.5.5/32
```

PE1 needs to send traffic to PE2.

Basic MPLS logic:

```text
BGP route points to PE2 loopback.
IGP provides reachability to PE2 loopback.
LDP provides a transport label toward PE2 loopback.
```

Example:

```text
Destination: 203.0.113.0/24
BGP next hop: 5.5.5.5
IGP next hop: P1
LDP label:    label learned from P1 for 5.5.5.5/32
```

This works only if the IGP path and the LDP label path are usable.

---

## What Can Go Wrong

Suppose the P1-P2 link comes up.

```text
P1 --- P2
```

OSPF or IS-IS can form an adjacency quickly.

But LDP needs more work:

```text
1. LDP Hellos are exchanged.
2. The LDP TCP session is established.
3. LDP initialization completes.
4. Address messages are exchanged.
5. Label mappings are exchanged.
6. LIB and LFIB entries are updated.
```

If the IGP selects the P1-P2 link before LDP is ready, traffic may be forwarded toward P2 without a usable label.

In a BGP-free MPLS core, this can cause packet loss.

Example:

```text
PE1 has a BGP route to 203.0.113.0/24.
P1 and P2 do not have that BGP route.
Therefore, the packet must stay labeled through the core.
```

If the packet loses its transport label or is sent over a link with no usable label, a P router may have to do a normal IP lookup for the customer destination.

```text
P router receives unlabeled customer packet
  ↓
P router checks destination 203.0.113.1
  ↓
P router does not have that customer route
  ↓
Drop
```

This is the main problem LDP-IGP Synchronization is used to avoid.

---

## Two Common Failure Cases

LDP-IGP synchronization is useful in two main situations.

### Case 1: IGP Adjacency Comes Up Before LDP Is Ready

Example:

```text
P1-P2 link comes up.
OSPF adjacency forms.
OSPF installs the link as the best path.
LDP session is not ready yet.
Traffic is attracted to the link too early.
```

### Case 2: LDP Fails But the IGP Stays Up

Example:

```text
P1-P2 physical link remains up.
OSPF adjacency remains full.
LDP session fails.
```

Without LDP-IGP sync:

```text
IGP still thinks the link is usable.
Traffic continues using the link.
But MPLS labels are missing or unusable.
```

That can break MPLS forwarding even though IP routing looks healthy.

---

## Basic Idea

LDP-IGP Synchronization tells the IGP:

```text
Do not prefer this link for normal traffic until LDP is synchronized.
```

Depending on the state of the IGP adjacency, this can affect the IGP in two ways.

During initial bring-up, if the LDP peer is considered reachable, the IGP may wait before allowing the adjacency to fully come up.

```text
IGP adjacency forming
LDP peer reachable
LDP sync not achieved
  ↓
IGP waits for LDP sync
```

After the IGP adjacency already exists, if LDP sync is not achieved or is later lost, the IGP can keep the adjacency but advertise the link with a **maximum metric**.

```text
IGP adjacency up
LDP not synchronized
  ↓
Advertise max metric
  ↓
Other IGP paths are preferred
```

When LDP becomes synchronized:

```text
LDP session is established.
Labels are exchanged.
LDP tells the IGP that sync is achieved.
The IGP advertises the normal metric again.
```

---

## What "Synchronized" Means

For a link to be synchronized, LDP must be ready with the peer connected through that interface.

```text
The LDP peer is discovered.
The LDP TCP session is up.
The peer's transport address is reachable.
Label exchange is complete enough for the link to be used.
```

The output of `show mpls ldp igp sync` should show this:

```text
Sync status: sync achieved; peer reachable.
```

If LDP is not ready, you may see:

```text
Sync status: sync not achieved; peer reachable.
```

That means:

```text
The peer appears reachable.
LDP-IGP sync expects LDP to come up.
Until then, the IGP should avoid the link.
```

---

## Behavior Without LDP-IGP Sync

Topology:

```text
            P3
           /  \
          /    \
 PE1 --- P1 --- P2 --- PE2
```

Assume all links have equal cost.

Without LDP-IGP sync, the process after the P1-P2 link comes up is roughly:

1. P1-P2 link comes up.
2. OSPF/IS-IS forms an adjacency.
3. The IGP advertises the normal link metric.
4. Other routers may select the P1-P2 link as the best path.
5. LDP is still not ready.
6. MPLS traffic may be sent over the link before labels are usable.
7. Packet loss can occur.

This is especially problematic if the P1-P2 link is the preferred path to a PE loopback.

---

## Behavior With LDP-IGP Sync

With LDP-IGP sync enabled, the behavior depends on whether the IGP adjacency is still forming or already established.

During initial link bring-up:

1. P1-P2 link comes up.
2. If the LDP peer is reachable but LDP sync is not achieved, the IGP may wait before fully bringing up the adjacency.
3. LDP establishes the session and exchanges labels.
4. LDP notifies the IGP that sync is achieved.
5. The IGP adjacency can fully come up and the normal link metric can be advertised.
6. Traffic can now use the link normally.

If the IGP adjacency is already established:

1. LDP sync is not achieved or is lost.
2. The IGP keeps the adjacency up.
3. The IGP advertises the link with max metric.
4. Other available paths are preferred.
5. LDP synchronizes again.
6. The IGP advertises the normal link metric again.

In simple form:

```text
Without LDP-IGP sync:
IGP uses link first → LDP catches up later

With LDP-IGP sync:
LDP becomes ready first → IGP uses link after
```

---

## Max Metric Behavior

LDP-IGP sync does not disable the interface.

It can delay IGP adjacency formation during initial bring-up if the LDP peer is considered reachable.

After the IGP adjacency is already up, it does not tear the adjacency down just because LDP sync is lost.

Instead, the IGP advertises the link with a maximum metric while the link is not synchronized.

```text
Interface up
IGP adjacency up
LDP sync not achieved
  ↓
Advertise max metric
```

This allows the link to remain in the topology, but makes it undesirable.

If an alternate path exists, the alternate path should be preferred.

Example:

```text
P1 can reach P2 through P3.

Direct P1-P2 link:
- IGP adjacency up
- LDP sync not achieved
- advertised with max metric

Alternate path:
P1 → P3 → P2

Result:
Traffic uses the alternate path.
```

If no alternate path exists, the max-metric link may still be selected because it is the only path.

---

## Peer Reachability Logic

When LDP-IGP sync is enabled on an interface, LDP checks whether a peer connected through that interface is reachable.

It does this by looking up the peer's **transport address** in the routing table.

Usually, the transport address is the peer's LDP router ID.

Example:

```text
Peer LDP ID:        3.3.3.3:0
Peer transport:     3.3.3.3
Local routing table: route to 3.3.3.3 exists
```

If a route to the peer's transport address exists, LDP assumes synchronization is required for that interface.

```text
Peer transport address reachable
  ↓
Wait for LDP sync
```

This is why loopback reachability matters. Always advertise the loopbacks in the IGP!

---

## Inaccurate Peer Reachability

Be careful with default routes, summary routes, and static routes.

LDP-IGP sync may see a route to the peer's transport address and assume the peer is reachable.

But that route might not actually lead to the peer.

Example:

```text
Peer transport address: 3.3.3.3

Routing table:
O*IA 0.0.0.0/0 via 10.1.13.3
```

A default route technically matches `3.3.3.3`.

So LDP may believe the peer is reachable.

But if that default route does not actually provide working reachability to `3.3.3.3`, the LDP session will not come up.

Result:

```text
LDP waits for sync.
The IGP advertises max metric.
The link will remain undesirable until the holddown expires.
```

---

## The Chicken-and-Egg Problem

There is a subtle startup issue.

LDP often uses loopbacks as transport addresses.

Example:

```text
P1 LDP router ID: 2.2.2.2
P2 LDP router ID: 3.3.3.3
```

For the LDP TCP session to form, P1 and P2 must have IP reachability to each other's loopbacks.

That reachability usually comes from the IGP.

But LDP-IGP sync tells the IGP to wait for LDP.

So the relationship looks circular:

```text
IGP is needed so LDP can reach the peer transport address.
LDP is needed before the IGP should use the link normally.
```

This is handled by the sync logic and the holddown timer.

- If the peer transport address is not reachable yet,
the IGP can establish the adjacency without synchronization so the route can be learned.

- If the peer transport address is reachable, LDP expects the session to come up and synchronization to be achieved.

---

## Initial Adjacency vs Established Adjacency

LDP-IGP sync can affect the IGP differently depending on timing.

```text
Initial adjacency formation:
The IGP may wait for LDP sync before fully bringing the adjacency up.

Established adjacency:
If LDP sync is lost, the IGP usually keeps the adjacency but advertises max metric.
```

So both of these statements can be true:

- LDP-IGP sync can delay IGP adjacency formation.
- LDP-IGP sync can also keep an established adjacency up but make the link unattractive with max metric.

---

## Configuration Overview

There are several main configuration commands:

| Feature | Command | Mode |
| :--- | :--- | :--- |
| Enable sync for an IGP process | `mpls ldp sync` | OSPF / IS-IS router config |
| Re-enable sync on an interface after disabling it | `mpls ldp igp sync` | Interface config |
| Disable sync on selected interfaces | `no mpls ldp igp sync` | Interface config |
| Limit how long IGP waits for LDP | `mpls ldp igp sync holddown <milliseconds>` | Global config |
| Delay IGP notification after LDP sync | `mpls ldp igp sync delay <seconds>` | Interface config |

The two timers use different units:

```text
Holddown timer = milliseconds
Delay timer    = seconds
```

I'll cover these timers and configurations more in the following sections.

---

## Basic OSPF Configuration

Configuration:

```text
router ospf 1
 mpls ldp sync
```

This enables LDP-IGP synchronization on interfaces that belong to OSPF process 1.

These two commands are different:

| Command | Purpose |
| :--- | :--- |
| `mpls ldp autoconfig` | Enables LDP on OSPF-enabled interfaces |
| `mpls ldp sync` | Synchronizes OSPF metric advertisement with LDP readiness |

---

## Basic IS-IS Configuration

The command to enable LDP-IGP sync is the same for IS-IS.

Configuration:

```text
router isis
 mpls ldp sync
```

---

## Disabling Sync on Specific Interfaces

When you configure `mpls ldp sync` under the IGP process, it applies to the interfaces in that IGP process.

You can disable it on specific interfaces after enabling it globally:

```text
interface Ethernet0/3
 no mpls ldp igp sync
```

---

## Holddown Timer

By default, if the LDP peer is considered reachable but LDP synchronization is not achieved,
the IGP can wait indefinitely for LDP before establishing an adjacency.

The **holddown timer** limits how long the IGP waits for LDP synchronization.

In `show mpls ldp igp sync` output, the default appears as:

```text
IGP holddown time: infinite.
```

Syntax:

```text
P1(config)# mpls ldp igp sync holddown <milliseconds>
```

Example:

```text
mpls ldp igp sync holddown 10000
```

After the holddown timer expires, the IGP is allowed to proceed even if LDP synchronization has not been achieved.

This does **not** mean LDP synchronization is successful.
It only means the IGP no longer waits indefinitely for LDP before bringing up the adjacency.

Important distinction:

| Behavior                 | Purpose                                                                               |
| :----------------------- | :------------------------------------------------------------------------------------ |
| Holddown timer           | Controls how long the IGP  adjacency waits for LDP sync                               |
| Max-metric advertisement | Used when the IGP adjacency exists but LDP synchronization is not achieved or is lost |

In simple form:

```text
Initial bring-up:

LDP peer reachable
LDP sync not achieved
  ↓
IGP waits for LDP
  ↓
Holddown timer limits the wait
```

If the IGP adjacency is already up and LDP synchronization is not achieved or is later lost,
the IGP will advertise the link with maximum metric.

```text
Adjacency already up:

LDP sync not achieved / lost
  ↓
IGP adjacency remains up
  ↓
Link is advertised with max metric
```

---

### Why Holddown Exists

Holddown is useful when the router thinks the peer transport address is reachable, but LDP cannot actually establish the session.

Possible causes:

```text
MPLS not enabled on the neighbor interface.
LDP authentication mismatch.
Transport address not actually reachable.
LDP peer blocked by filtering or control-plane policy.
Wrong LDP router ID.
```

Without a holddown timer:

```text
The IGP may wait indefinitely for LDP synchronization.
If LDP is broken, the IGP may not proceed as expected.
```

With a holddown timer:

```text
The IGP waits for a limited time.
If LDP still does not synchronize, the IGP can proceed instead of waiting forever.
```

This is a tradeoff.

```text
Long/infinite holddown:
Better MPLS protection, but a broken LDP session can delay
IGP convergence for a long time.

Short holddown:
Faster fallback, but traffic may use the link before MPLS is ready.
```

The holddown timer does not mean LDP synchronization succeeded.

It only means the IGP stopped waiting for LDP.

After the IGP proceeds, still verify the actual sync state and advertised metric.

Configuring a holddown time can be useful when the router believes the neighbor's LDP transport address is reachable,
but LDP cannot establish the session.

For example, this can happen if the routing table has a default route or summary route that matches the neighbor's transport address,
but that route does not actually provide working reachability.

- The router finds a route to the neighbor's transport address.
- LDP assumes synchronization is required on the interface.
- The IGP waits for LDP synchronization before proceeding.
- But the LDP session never establishes, because the transport address is not actually reachable or LDP is otherwise broken.
- After the holddown timer expires, the IGP is allowed to proceed instead of waiting forever.
- If the new IGP adjacency provides real reachability to the transport address, the LDP session can then establish.

Without the holdown timer, the IGP would wait indefinitely.

---

## Delay Timer

The delay timer is different from the holddown timer.

The delay timer is configured per interface. It is 0 seconds by default.

Syntax:

```text
interface Ethernet0/1
 mpls ldp igp sync delay <seconds>
```

Example:

```text
interface Ethernet0/1
 mpls ldp igp sync delay 10
```

This means:

```text
After LDP synchronization is achieved,
wait 10 seconds before notifying the IGP that synchronization is valid.
```

In simple form:

```text
LDP sync achieved
  ↓
Delay timer starts
  ↓
Delay timer expires
  ↓
LDP checks that synchronization is still valid
  ↓
LDP notifies the IGP
```

The result depends on the IGP state.

If the IGP adjacency is already established but the link is being advertised with max metric,
the delay timer can delay restoration of the normal metric.

```text
Adjacency already up
Link advertised with max metric
LDP sync achieved
  ↓
Delay timer runs
  ↓
Normal metric is restored after the IGP is notified
```

If the IGP is waiting during initial bring-up,
the delay timer can delay the point at which LDP notifies the IGP that it is safe to proceed with the adjacency.

```text
Initial bring-up
IGP waiting for LDP sync
LDP sync achieved
  ↓
Delay timer runs
  ↓
IGP is notified after the timer expires
```

This can be useful to dampen instability.

If LDP briefly flaps and comes back, the delay timer prevents LDP from immediately telling the IGP to use the link again. LDP waits, rechecks that synchronization is still valid, and only then notifies the IGP.

---

### Holddown vs Delay

These timers are easy to confuse:

| Timer | Command | Unit | Purpose |
| :--- | :--- | :--- | :--- |
| Holddown | `mpls ldp igp sync holddown <milliseconds>` | milliseconds | Limits how long the IGP waits for LDP sync |
| Delay | `mpls ldp igp sync delay <seconds>` | seconds | Delays IGP notification after LDP sync is achieved |

Example:

```text
mpls ldp igp sync holddown 30000
!
interface Ethernet0/1
 mpls ldp igp sync delay 10
```

Meaning:

```text
If sync is not achieved, the IGP will wait up to 30 seconds.
After sync is achieved, LDP will wait 10 seconds before notifying the IGP.
```

---

## Full Configuration Example

Topology:

```text
            P3
           /  \
          /    \
 PE1 --- P1 --- P2 --- PE2
```

Requirement:

```text
Automatically enable MPLS and LDP on all OSPF links.
Enable LDP-IGP synchronization.
Use a 30-second holddown.
Delay metric restoration by 10 seconds on the P1-P2 link.
Do not enable MPLS/LDP and LDP synchronization on P1's management interface E0/3.
```

P1:

```text
mpls ldp router-id Loopback0 force
mpls ldp igp sync holddown 30000
!
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
 ip ospf 1 area 0
!
interface Ethernet0/0
 description to PE1
 ip address 10.1.2.2 255.255.255.0
 ip ospf 1 area 0
!
interface Ethernet0/1
 description to P2
 ip address 10.2.3.2 255.255.255.0
 ip ospf 1 area 0
 mpls ldp igp sync delay 10
!
interface Ethernet0/2
 description to P3
 ip address 10.2.4.2 255.255.255.0
 ip ospf 1 area 0
!
interface Ethernet0/3
 description management
 ip address 192.168.2.2 255.255.255.0
 ip ospf 1 area 0
 no mpls ldp igp sync
 no mpls ldp igp autoconfig
!
router ospf 1
 mpls ldp autoconfig
 mpls ldp sync
```

Notes:

```text
mpls ldp autoconfig enables LDP on OSPF-enabled interfaces.
mpls ldp sync enables LDP-IGP synchronization for OSPF interfaces.
no mpls ldp igp sync excludes Ethernet0/3 from synchronization.
```

---

## `show mpls ldp igp sync`

Example output when sync is working:

```text
P1# show mpls ldp igp sync
    Ethernet0/0:
        LDP configured; LDP-IGP Synchronization enabled.
        Sync status: sync achieved; peer reachable.
        Sync delay time: 0 seconds (0 seconds left)
        IGP holddown time: infinite.
        Peer LDP Ident: 1.1.1.1:0
        IGP enabled: OSPF 1
    Ethernet0/1:
        LDP configured; LDP-IGP Synchronization enabled.
        Sync status: sync achieved; peer reachable.
        Sync delay time: 0 seconds (0 seconds left)
        IGP holddown time: infinite.
        Peer LDP Ident: 3.3.3.3:0
        IGP enabled: OSPF 1
    Ethernet0/2:
        LDP configured; LDP-IGP Synchronization enabled.
        Sync status: sync achieved; peer reachable.
        Sync delay time: 0 seconds (0 seconds left)
        IGP holddown time: infinite.
        Peer LDP Ident: 4.4.4.4:0
        IGP enabled: OSPF 1
```

Important fields:

| Field | Meaning |
| :--- | :--- |
| `LDP configured` | LDP is enabled on the interface |
| `LDP-IGP Synchronization enabled` | Sync is enabled on the interface |
| `sync achieved` | LDP and IGP are synchronized |
| `peer reachable` | LDP believes the peer transport address is reachable |
| `Sync delay time` | How long LDP waits to notify the IGP after sync |
| `IGP holddown time` | How long the IGP waits for sync |
| `Peer LDP Ident` | The LDP peer associated with the interface |

---

## `show ip ospf mpls ldp interface`

Example:

```text
P1# show ip ospf mpls ldp interface Ethernet0/1

Ethernet0/1
  Process ID 1, Area 0
  LDP is configured through LDP autoconfig
  LDP-IGP Synchronization : Required
  Holddown timer is not configured
  Interface is up 
```

This command is useful because it shows the OSPF view of LDP synchronization.

---

## Relationship to LDP Autoconfig

LDP autoconfig and LDP-IGP sync are related but separate.

### LDP Autoconfig

```text
router ospf 1
 mpls ldp autoconfig
```

This means:

```text
Enable MPLS/LDP on OSPF-enabled interfaces.
```

It's just a quick shortcut to enable MPLS/LDP on interfaces.

### LDP-IGP Sync

```text
router ospf 1
 mpls ldp sync
```

This means:

```text
Enable the LDP-IGP sync feature for OSPF-enabled interfaces.
```

---

## Relationship to L3VPNs

LDP-IGP sync becomes very important before L3VPNs.

In MPLS L3VPN, packets usually have two labels:

```text
Outer label = transport label
Inner label = VPN label
```

LDP provides the outer transport label.

MP-BGP provides the inner VPN label.

Example:

```text
PE1 learns a VPNv4 route from PE2.
BGP next hop: 5.5.5.5
VPN label:    300
LDP label:    transport label to 5.5.5.5
```

If the IGP sends traffic over a link where LDP is not ready, the outer transport label may be missing or unusable.

The P router does not understand the customer route.

It only knows how to forward based on the outer label.

```text
L3VPN forwarding depends on stable transport labels.
LDP-IGP sync helps protect those transport labels during convergence.
```

---

## Example Lab Tasks

### Task 1: Enable LDP-IGP Sync for OSPF

Requirement:

```text
All OSPF core links should wait for LDP before being preferred.
```

Configuration:

```text
router ospf 1
 mpls ldp sync
```

If LDP should also be enabled automatically:

```text
router ospf 1
 mpls ldp autoconfig
 mpls ldp sync
```

---

### Task 2: Enable LDP-IGP Sync for IS-IS

Requirement:

```text
Enable LDP synchronization on IS-IS core links.
```

Configuration:

```text
router isis
 mpls ldp sync
```

If LDP should also be enabled automatically:

```text
router isis
 mpls ldp autoconfig
 mpls ldp sync
```

---

### Task 3: Exclude an Interface from Sync

Requirement:

```text
OSPF runs on Ethernet0/3, but Ethernet0/3 should not use LDP-IGP sync.
```

Configuration:

```text
interface Ethernet0/3
 no mpls ldp igp sync
```

---

### Task 4: Use a 20-Second Holddown

Requirement:

```text
Do not let the IGP wait more than 20 seconds for LDP sync.
```

Configuration:

```text
mpls ldp igp sync holddown 20000
```

Remember:

```text
The holddown value is in milliseconds.
```

---

### Task 5: Reduce the impact of unstable links

Requirement:

```text
After LDP sync is achieved on Ethernet0/1,
wait 15 seconds before the IGP uses the normal metric.
```

Configuration:

```text
interface Ethernet0/1
 mpls ldp igp sync delay 15
```

---

## Common Misunderstandings

### Misunderstanding 1: LDP-IGP sync prevents the IGP adjacency

Partly true, depending on the state.

During initial bring-up, LDP-IGP sync can delay the IGP adjacency if the LDP peer is considered reachable.

After the adjacency is already established,
loss of LDP sync causes the IGP to advertise max metric on the link rather than simply tearing the adjacency down.

```text
Initial bring-up:
LDP sync can delay the IGP adjacency.

Established adjacency:
LDP sync loss usually causes max-metric advertisement.
```

Furthermore, if a holddown time is configured, the IGP adjacency can establish even without LDP sync.

---

### Misunderstanding 2: Holddown and delay timers

They are easy to confuse:

```text
Holddown = how long the IGP waits for LDP synchronization.
Delay    = how long LDP waits after synchronization before notifying the IGP.
```

---

## Key Points

* LDP-IGP synchronization ensures an LDP peering is established on a link before the IGP establishes an adjacency or prefers the link.
* It is configured under OSPF or IS-IS.
  * Use `mpls ldp sync` under the IGP process.
  * Use `no mpls ldp igp sync` to disable it on a specific interface.
* LDP-IGP sync is separate from LDP autoconfig.
  * `mpls ldp autoconfig` enables MPLS/LDP on IGP interfaces.
  * `mpls ldp sync` synchronizes the IGP with LDP readiness.
* During initial bring-up, the IGP waits for LDP sync before fully bringing up the adjacency.
  * This happens if the peer's LDP transport address is considered reachable.
  * The holddown timer limits how long the IGP waits.
* After the IGP adjacency is already up, loss of LDP sync usually does not tear the adjacency down.
  * Instead, the IGP advertises the link with maximum metric.
  * This makes the link unattractive if another path exists.
* The holddown timer controls how long the IGP waits for LDP sync.
  * Use `mpls ldp igp sync holddown <milliseconds>`.
  * The value is in milliseconds.
  * The default is infinite.
* The delay timer controls how long LDP waits after sync is achieved before notifying the IGP.
  * Use `mpls ldp igp sync delay <seconds>` under the interface.
  * The value is in seconds.
  * The default is 0 seconds.
