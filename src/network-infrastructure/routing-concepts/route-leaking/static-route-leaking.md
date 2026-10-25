# Static VRF Route Leaking

**Static VRF route leaking** uses explicitly configured static routes to provide selected connectivity across routing contexts.

This lesson demonstrates **bidirectional connectivity between `VRF_A` and `VRF_B`**, using the **global routing table for next-hop resolution**. It does not require BGP, route targets, or route replication.

The key distinction is that **two directly cross-referenced static routes between nondefault VRFs are not a supported general solution**. Instead, each VRF uses the `global` keyword to resolve a next hop through static routes in the global routing table.

## Main Idea

On R1, `VRF_A` and `VRF_B` have independent routing and forwarding tables.

A packet from R3 arrives on R1's `Ethernet0/1`, so R1 looks up its destination in `VRF_A`. However, the destination behind R4 is reachable through `VRF_B`, not through a directly connected interface in `VRF_A`.

The solution used here is:

1. Add a static route in `VRF_A` for R4's Loopback0, using the `global` keyword.
2. Add a static route in the global table to resolve that next hop out the `VRF_B` interface.
3. Configure the equivalent routes in the opposite direction.

Conceptually:

```text
R3 (VRF_A) ---> R1 VRF_A lookup
                        |
                        | Static route with global next-hop resolution
                        v
                 Global next-hop lookup
                        |
                        | Next-hop /32 points out the VRF_B interface
                        v
                       R4
```

The global table supplies **next-hop resolution**.

## Lab Topology

These lessons use a four-router route-leaking lab, but only R1, R3, and R4 are involved in this exercise.

```text
              R3
        Lo0: 3.3.3.3/32
               |
          E0/1 (VRF_A)
               |
              R1
               |
          E0/2 (VRF_B)
               |
              R4
        Lo0: 4.4.4.4/32
```

| Device | Interface | IP Address | R1 Routing Context |
| --- | --- | --- | --- |
| R1 | `Ethernet0/1` | `10.1.3.1/24` | `VRF_A` |
| R3 | `Ethernet0/0` | `10.1.3.3/24` | Global (on R3) |
| R1 | `Ethernet0/2` | `10.1.4.1/24` | `VRF_B` |
| R4 | `Ethernet0/0` | `10.1.4.4/24` | Global (on R4) |
| R3 | `Loopback0` | `3.3.3.3/32` | Global (on R3) |
| R4 | `Loopback0` | `4.4.4.4/32` | Global (on R4) |

R1's VRFs and interface membership are already configured in the base lab. R3 and R4 do not use VRFs themselves.

**Objective:** R3's `3.3.3.3/32` must communicate with R4's `4.4.4.4/32` in both directions.

## Initial Routing Tables

Before adding any static routes, R1 has the following connected networks:

```text
R1# show ip route vrf VRF_A connected
C    10.1.3.0/24 is directly connected, Ethernet0/1

R1# show ip route vrf VRF_B connected
C    10.1.4.0/24 is directly connected, Ethernet0/2
```

The corresponding local `/32` routes are omitted.

The **global routing table has neither `10.1.3.0/24` nor `10.1.4.0/24`** as connected networks, because R1's interfaces are assigned to different VRFs.

## Step 1: Add Ordinary Static Routes in Each VRF

First, configure routes from R1 to the loopback behind each VRF interface.

```text
R1(config)# ip route vrf VRF_A 3.3.3.3 255.255.255.255 10.1.3.3
R1(config)# ip route vrf VRF_B 4.4.4.4 255.255.255.255 10.1.4.4
```

These are ordinary VRF static routes:

```text
VRF_A:
  3.3.3.3/32 via 10.1.3.3

VRF_B:
  4.4.4.4/32 via 10.1.4.4
```

Verify:

```text
R1# show ip route vrf VRF_A 3.3.3.3
R1# show ip route vrf VRF_B 4.4.4.4
```

**These routes are not leaks.** Each prefix is still reachable only in its original routing context.

## Step 2: Add Global Routes to the Next-Hop Addresses

We need the **global routing table** to be able to resolve the next-hop IP addresses on both VRF-facing Ethernet links.

Add host routes in the global table:

```text
R1(config)# ip route 10.1.3.3 255.255.255.255 Ethernet0/1
R1(config)# ip route 10.1.4.4 255.255.255.255 Ethernet0/2
```

These routes are configured globally, even though the outgoing interfaces belong to `VRF_A` and `VRF_B`.

| Global Static Route | Outgoing Interface | Interface VRF |
| --- | --- | --- |
| `10.1.3.3/32` | `Ethernet0/1` | `VRF_A` |
| `10.1.4.4/32` | `Ethernet0/2` | `VRF_B` |

An interface-only static route on Ethernet normally causes ARP resolution for the route's destination address. **That is intentional here**: the host routes point to the directly attached next-hop addresses (`10.1.3.3` and `10.1.4.4`), rather than to remote loopback prefixes.

> These are cross-context static routes. Their installation depends on support for `ip route static inter-vrf`, which is enabled by default.

To test the above point, I disabled the default `ip route static inter-vrf

```
R1(config)# do show run all | i inter-vrf
ip route static inter-vrf
R1(config)#no ip route static inter-vrf
*Oct 24 22:09:04.077: %IPRT-6-STATICROUTESACROSSVRF: Un-installing static route 10.1.3.3/32 from default routing table with outgoing interface Ethernet0/1 
```

Notice the error message. We need to re-enable the default setting for this lab to work:

```
R1(config)# ip route static inter-vrf
```

Verify:

```text
R1# show ip route static
...
S        10.1.3.3/32 is directly connected, Ethernet0/1
S        10.1.4.4/32 is directly connected, Ethernet0/2
```

The two `/32` routes should be visible in the **global** routing table.

## Step 3: Leak R4's Loopback0 into VRF_A

R3 needs to reach R4's `4.4.4.4/32`.

Configure this route on R1:

```text
R1(config)# ip route vrf VRF_A 4.4.4.4 255.255.255.255 10.1.4.4 global
```

This command has two important parts:

| Part | Meaning |
| --- | --- |
| `vrf VRF_A` | Installs the static route in `VRF_A` |
| `10.1.4.4 global` | Resolves the next hop through the global routing table |

The global table already has a route to `10.1.4.4/32` through `Ethernet0/2`.

```text
VRF_A: 4.4.4.4/32
        |
        | Next hop 10.1.4.4, resolved globally
        v
Global: 10.1.4.4/32 out Ethernet0/2
        |
        v
       R4
```

Without the `global` keyword, R1 would attempt to resolve `10.1.4.4` within `VRF_A`, where that next hop is not reachable.

Verify:

```text
R1# show ip route vrf VRF_A 4.4.4.4
R1# show ip cef vrf VRF_A 4.4.4.4 detail
```

## Step 4: Leak R3's Loopback0 into VRF_B

Configure the corresponding return route in `VRF_B`:

```text
R1(config)# ip route vrf VRF_B 3.3.3.3 255.255.255.255 10.1.3.3 global
```

Now `VRF_B` contains a route to R3's loopback, with its next hop resolved through the global routing table.

```text
VRF_B: 3.3.3.3/32
        |
        | Next hop 10.1.3.3, resolved globally
        v
Global: 10.1.3.3/32 out Ethernet0/1
        |
        v
       R3
```

Verify:

```text
R1# show ip route vrf VRF_B 3.3.3.3
R1# show ip cef vrf VRF_B 3.3.3.3 detail
```

At this point, **both VRFs contain the selected destination prefix from the other VRF**. The global table is used to resolve the opposite interface's next-hop address.

## Step 5: Configure Return Routes on R3 and R4

R3 and R4 must send the other router's Loopback0 traffic toward R1.

On R3:

```text
R3(config)# ip route 4.4.4.4 255.255.255.255 10.1.3.1
```

On R4:

```text
R4(config)# ip route 3.3.3.3 255.255.255.255 10.1.4.1
```

Both commands are normal global static routes on the end routers.

Alternatively, you could just configure default routes on R3 and R4. The point is, they need valid routes to reach each other.

## Step 6: Verify Bidirectional Connectivity

R1 should now have the following static routes:

| Routing Table | Destination | Next Hop / Outgoing Interface | Type |
| --- | --- | --- | --- |
| Global | `10.1.3.3/32` | `Ethernet0/1` | Next-hop resolution |
| Global | `10.1.4.4/32` | `Ethernet0/2` | Next-hop resolution |
| `VRF_A` | `3.3.3.3/32` | `10.1.3.3` | Ordinary VRF static route |
| `VRF_A` | `4.4.4.4/32` | `10.1.4.4 global` | Static leak |
| `VRF_B` | `4.4.4.4/32` | `10.1.4.4` | Ordinary VRF static route |
| `VRF_B` | `3.3.3.3/32` | `10.1.3.3 global` | Static leak |

Check the individual routing tables:

```text
R1# show ip route static
...
S        10.1.3.3/32 is directly connected, Ethernet0/1
S        10.1.4.4/32 is directly connected, Ethernet0/2

R1# show ip route vrf VRF_A static
...
S        3.3.3.3 [1/0] via 10.1.3.3
S        4.4.4.4 [1/0] via 10.1.4.4

R1# show ip route vrf VRF_B static
...
S        3.3.3.3 [1/0] via 10.1.3.3
S        4.4.4.4 [1/0] via 10.1.4.4
```

Check the forwarding entries:

```text
R1# show ip cef vrf VRF_A 4.4.4.4 detail 
4.4.4.4/32, epoch 0
  recursive via 10.1.4.4
    attached to Ethernet0/2

R1# show ip cef vrf VRF_B 3.3.3.3 detail 
3.3.3.3/32, epoch 0
  recursive via 10.1.3.3
    attached to Ethernet0/1

R1# show ip cef 10.1.3.3 detail
10.1.3.3/32, epoch 0, flags [attached]
  Interest List:
     - exported attached prefix tracking
  Adj source: IP adj out of Ethernet0/1, addr 10.1.3.3 73ECBEBE0DE8
    Dependent covered prefix type adjfib, cover 0.0.0.0/0
  1 RR source [no flags]
  attached to Ethernet0/1

R1# show ip cef 10.1.4.4 detail
10.1.4.4/32, epoch 0, flags [attached]
  Interest List:
     - exported attached prefix tracking
  Adj source: IP adj out of Ethernet0/2, addr 10.1.4.4 73ECC34D9780
    Dependent covered prefix type adjfib, cover 0.0.0.0/0
  1 RR source [no flags]
  attached to Ethernet0/2
```

Test from the end routers, using **Loopback0 as the source**:

```text
R3# ping 4.4.4.4 source l0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 4.4.4.4, timeout is 2 seconds:
Packet sent with a source address of 3.3.3.3 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```

```
R4# ping 3.3.3.3 source l0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 3.3.3.3, timeout is 2 seconds:
Packet sent with a source address of 4.4.4.4 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/4 ms
```

When the forwarding path is supported, both tests should succeed.

### Forwarding from R3 to R4

```text
R3 (3.3.3.3)
       |
       v
R1 Ethernet0/1 (VRF_A)
       |
       v
VRF_A: 4.4.4.4/32 via 10.1.4.4 global
       |
       v
Global next-hop lookup: 10.1.4.4/32 out E0/2
       |
       v
R4 (4.4.4.4)
```

Traffic in the reverse direction uses the equivalent `VRF_B` static route and the global route to `10.1.3.3/32`.

## Why Not Use Direct VRF-to-VRF Static Routes?

It might seem natural to configure the following routes instead:

```text
! Do not rely on these as the inter-VRF forwarding solution.
ip route vrf VRF_A 4.4.4.4 255.255.255.255 Ethernet0/2 10.1.4.4
ip route vrf VRF_B 3.3.3.3 255.255.255.255 Ethernet0/1 10.1.3.3
```

These commands point a static route in one nondefault VRF **directly** at the other nondefault VRF's interface.

Cisco documents that reciprocal direct static routes between different VRFs are **not a supported general forwarding solution**. A route might appear in the configuration or routing table without passing traffic correctly.

The method used in this lesson is different:

- The destination static routes reside in `VRF_A` and `VRF_B`.
- Both use `global` for next-hop resolution.
- Separate host routes in the **global routing table** resolve those next hops out the corresponding VRF interfaces.

For more scalable or policy-rich route leaking, consider route-target import/export, route replication, or VASI, covered in later chapters.

## Troubleshooting

### Leaked Static Route Is Missing

Check that the global next-hop route exists:

```text
R1# show ip route 10.1.4.4
R1# show ip route 10.1.3.3
```

Then check the VRF routes:

```text
R1# show ip route vrf VRF_A 4.4.4.4
R1# show ip route vrf VRF_B 3.3.3.3
```

If the route does not appear, check whether the expected outgoing interface is up and whether inter-VRF static routes have been restricted.

### Route Exists but Ping Fails

Check the following, in order:

1. R3 routes `4.4.4.4/32` toward R1 at `10.1.3.1`.
2. `VRF_A` has `4.4.4.4/32` with next-hop `10.1.4.4 global`.
3. The global table has `10.1.4.4/32` via `Ethernet0/2`.
4. R1 can resolve ARP for `10.1.4.4` on `Ethernet0/2`.
5. R4 routes `3.3.3.3/32` toward R1 at `10.1.4.1`.
6. `VRF_B` has `3.3.3.3/32` with next-hop `10.1.3.3 global`.
7. The global table has `10.1.3.3/32` via `Ethernet0/1`.
8. R1 can resolve ARP for `10.1.3.3` on `Ethernet0/1`.
9. CEF supports forwarding across the required routing contexts on this image.

Useful commands:

```text
R1# show ip arp vrf VRF_A
R1# show ip arp vrf VRF_B
R1# show ip cef vrf VRF_A 4.4.4.4 detail
R1# show ip cef vrf VRF_B 3.3.3.3 detail
```

### One-Way Connectivity

Remember that both directions must be configured. A successful ping from R3 requires not only the `VRF_A` route toward R4, but also the reverse route from R4 through `VRF_B` back to R3.

## Key Points

- This lab establishes selected **`VRF_A` ↔ `VRF_B` connectivity** using static routes and the global routing table for next-hop resolution.
- The **`global`** keyword on `ip route vrf` installs a route in a VRF but resolves its next hop in the **global routing table**.
- Since the next-hop addresses are on VRF interfaces, the global table needs explicit `/32` routes pointing out those interfaces.
- On Ethernet, the interface-only global host routes are suitable here because they point to the **directly attached next-hop addresses**, not remote destination prefixes.
- Direct reciprocal static routes pointing from one nondefault VRF interface to another are **not a supported solution**.
- **`no ip route static inter-vrf`** prevents cross-context interface static routes where supported, including the global host routes needed for this lab.
- Return routing on **R3 and R4** is required.
- Check the **CEF forwarding path and end-to-end pings**, not just the RIB entries.

## References

- [Cisco: Route Leaking in MPLS/VPN Networks](https://www.cisco.com/c/en/us/support/docs/multiprotocol-label-switching-mpls/multiprotocol-label-switching-vpns-mpls-vpns/47807-routeleaking.html)
- [Cisco: IOS MPLS Command Reference, `ip route static inter-vrf` and `ip route vrf`](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/mpls/command/mp-cr-book/mp-e1.html)
- [NetworkLessons: VRF Lite Route Leaking (Static Routes)](https://networklessons.com/mpls/vrf-lite-route-leaking)
