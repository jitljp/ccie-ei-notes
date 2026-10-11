# RPF Failures

An **RPF failure** occurs when a multicast packet arrives on an interface that does not match the router's expected Reverse Path Forwarding (RPF) interface.

When this happens, the router drops the packet rather than forwarding it to downstream receivers.

RPF failures can occur even when unicast routing works correctly, particularly when multicast traffic follows a different path from the one selected by the RPF lookup.

Common causes include:

- **Asymmetric routing:** Multicast traffic arrives through a different path from the selected RPF route.
- **Incorrect routing information:** The RPF lookup selects the wrong incoming interface.
- **Incorrect static mroutes:** An `ip mroute` configuration points toward the wrong upstream interface or neighbor.
- **Routing changes:** A route change alters the selected RPF interface before the multicast distribution tree converges.

## Example: RPF Failure

Consider the following topology:

```text
              Source
           10.10.10.10
                |
           Multicast Core
             /      \
            /        \
           R1        R2
           |          |
      E0/0 |          | E0/1
           \          /
            \        /
                R3
                |
             Receiver
            239.1.1.1
```

Suppose R3's selected RPF route toward `10.10.10.10` points through R1:

```text
RPF Lookup:    10.10.10.10
RPF Interface: Ethernet0/0
RPF Neighbor:  10.0.13.1 (R1)
```

However, multicast packets from the source are arriving through R2 on Ethernet0/1.

The RPF check fails:

```text
Packet source:       10.10.10.10
Incoming interface:  Ethernet0/1
Expected RPF intf:   Ethernet0/0

Result: RPF FAIL → DROP
```

Even if R3 has interested downstream receivers, these packets cannot be forwarded because they arrived on the wrong interface.

## Identifying RPF Failures

### `show ip mroute`

Start by examining the multicast routing entry:

```text
R3# show ip mroute 239.1.1.1

(10.10.10.10, 239.1.1.1), 00:05:00/00:02:59, flags: T
  Incoming interface: Ethernet0/0, RPF nbr 10.0.13.1
  Outgoing interface list:
    Ethernet0/2, Forward/Sparse, 00:05:00/00:02:59
```

Notice:

```text
Incoming interface: Ethernet0/0
```

This is the interface R3 expects multicast traffic to arrive on.

If traffic is actually arriving on Ethernet0/1, it fails the RPF check.

### `show ip mroute count`

Use the `count` keyword to examine multicast forwarding and drop counters:

```text
R3# show ip mroute 239.1.1.1 count
```

For example:

```text
IP Multicast Statistics
Forwarding Counts: Pkt Count/Pkts per second/Avg Pkt Size/Kilobits per second
Other counts: Total/RPF failed/Other drops(OIF-null, rate-limit etc)

Group: 239.1.1.1, Source count: 1,
       Packets forwarded: 0, Packets received: 471
  Source: 10.10.10.10/32,
    Forwarding: 0/0/0/0, Other: 471/471/0
```

The important field is:

```text
Other: 471/471/0
```

The three values represent:

| Field | Value | Meaning |
|---|---|---|
| Total | 471 | Total packets counted in this category |
| RPF failed | 471 | Packets that failed the RPF check |
| Other drops | 0 | Packets dropped for other reasons |

In this example, all 471 packets counted were dropped because of RPF failures.

**A continuously increasing RPF failure counter is strong evidence that multicast traffic is arriving on an unexpected interface.**

Compare the counters across successive checks to determine whether the failures are ongoing or occurred previously.

### `show ip rpf`

Next, examine the actual RPF selection:

```text
R3# show ip rpf 10.10.10.10
```

For example:

```text
RPF information for ? (10.10.10.10)
  RPF interface: Ethernet0/0
  RPF neighbor: ? (10.0.13.1)
  RPF route/mask: 10.10.10.0/255.255.255.0
  RPF type: unicast (ospf 1)
  RPF recursion count: 0
  Doing distance-preferred lookups across tables
```

This confirms that R3 expects traffic from `10.10.10.10` to arrive through Ethernet0/0.

Also examine the unicast route:

```text
R3# show ip route 10.10.10.10
```

If a static mroute or MBGP route is being used for RPF, the selected path may differ from the normal unicast route.

## Correcting RPF Failures

The appropriate solution depends on why the multicast traffic is arriving on the unexpected interface.

### Option 1: Correct the Routing or Multicast Distribution Tree

If the RPF interface is correct but multicast traffic is arriving through the wrong path, investigate the upstream multicast routers.

For example, verify that:

- Upstream PIM neighbors are correctly established.
- PIM Join/Prune messages are building the intended distribution tree.
- Routing changes have converged.
- Multicast packets are being forwarded by the expected upstream router.

Useful commands include:

```text
R3# show ip pim neighbor
R3# show ip route 10.10.10.10
R3# show ip rpf 10.10.10.10
```

The goal is to ensure that multicast traffic arrives through the interface selected by RPF.

### Option 2: Change the RPF Path Using a Static Mroute

If multicast traffic is intentionally being received through a different path from unicast traffic, configure a static mroute.

For example:

```text
R3(config)# ip mroute 10.10.10.10 255.255.255.255 10.0.23.2
```

Assuming `10.0.23.2` is R2's next-hop address, this changes the selected RPF path to:

```text
RPF Interface: Ethernet0/1
RPF Neighbor:  10.0.23.2 (R2)
```

Normal unicast forwarding remains unchanged.

The multicast distribution tree must also use the new upstream path.

Static mroutes are covered in detail in [Static Multicast Routes](static-mroutes.md).

## RPF Failures for Shared Trees

RPF failures are not limited to source-tree forwarding.

For `(*,G)` shared-tree state in PIM Sparse Mode, the RPF lookup is performed toward the **RP** rather than the multicast source.

For example:

```text
Source: 10.10.10.10
RP:     1.1.1.1
Group:  239.1.1.1
```

For shared-tree forwarding:

```text
RPF Lookup: 1.1.1.1
```

For source-tree forwarding:

```text
RPF Lookup: 10.10.10.10
```

If traffic is expected on the RP-rooted shared tree, troubleshooting should include the RPF path toward the RP.

Verify:

```text
R3# show ip pim rp mapping
R3# show ip rpf 1.1.1.1
R3# show ip mroute 239.1.1.1
```

An important distinction is that an `(S,G)` entry can exist while traffic is still being forwarded using the shared tree. Do not assume that the source-based RPF path is already being used simply because `(S,G)` state exists.

The relationship between source-tree and shared-tree RPF is covered in [RPF for Source and Shared Trees](tree-rpf.md).

## RPF Failures vs. Other Multicast Problems

Not every multicast forwarding problem is an RPF failure.

| Problem | Explanation |
|---|---|
| RPF failure | Packet arrives on the wrong incoming interface and is dropped |
| Missing RPF route | Router cannot determine the required upstream path |
| Missing PIM neighbor | Router may be unable to establish the required upstream distribution tree |
| Null OIL | No eligible downstream interfaces exist for forwarding |
| Missing receiver membership | Router may have no downstream receiver interest |

For example, a router can successfully pass the RPF check but still drop multicast traffic because its OIL is empty.

Conversely, if the router cannot determine an RPF path, it may be unable to build the required multicast routing state in the first place.

**These problems should not automatically be interpreted as packet-level RPF failures.**

On IOS XE platforms that support it, `show ip mfib` and `show ip mfib count` can provide additional forwarding-plane information, particularly when hardware forwarding behavior needs to be investigated.

## IPv6 RPF Failures

IPv6 uses the same fundamental RPF checking behavior.

If an IPv6 multicast packet arrives on an interface other than the expected RPF interface, the packet can be dropped.

Use:

```text
R3# show ipv6 rpf 2001:DB8:10::10
R3# show ipv6 mroute
R3# show ipv6 pim neighbor
```

IPv6 static multicast routes can also influence the RPF path:

```text
R3(config)# ipv6 route 2001:DB8:10::/64 Ethernet0/1 FE80::2 multicast
```

This makes Ethernet0/1 the intended RPF interface for the specified source prefix when that route is selected.

## RPF Troubleshooting Checklist

When troubleshooting suspected RPF failures:

1. **Identify the source and group** using `show ip mroute`.
2. **Check RPF failure counters** using `show ip mroute <group> count`.
3. **Identify the expected incoming interface** using `show ip rpf <source>` or, for shared-tree RPF, the RP address.
4. **Determine where packets actually arrive** and compare that interface with the selected RPF interface.
5. **Verify routing and PIM state**, including unicast routes, static mroutes, and upstream PIM neighbors.
6. **Correct the underlying path mismatch**, then verify that RPF failure counters stop increasing and forwarding resumes.

## Key Points

- An RPF failure occurs when multicast traffic arrives on an interface other than the expected RPF interface.
- RPF failures can occur even when normal unicast connectivity works.
- Use **`show ip mroute count`** to identify packets dropped because of RPF failures.
- Use **`show ip rpf`** to determine the selected RPF interface and upstream neighbor.
- Asymmetric routing, incorrect RPF routes, and routing changes are common causes.
- Static mroutes can correct intentional differences between unicast and multicast paths.
- Missing PIM neighbors and empty OILs can prevent multicast forwarding without necessarily causing packet-level RPF failures.
- IPv4 and IPv6 use the same fundamental RPF checking principles.