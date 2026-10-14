# Next-Hop-Based S-G Hashing

**Next-hop-based S-G hashing** is a multicast ECMP load-splitting algorithm that uses the **source, group, and next-hop IP addresses** to select an RPF path.

Like basic S-G hashing, it allows different multicast groups from the same source to use different equal-cost paths.

However, incorporating the next-hop address provides two important advantages:

- **Avoids polarization:** Hash results depend on the actual next-hop addresses, preventing correlated path selections across multiple routers.
- **Improves RPF stability:** When an ECMP path fails, multicast states using the remaining paths can retain their existing RPF selections.

Next-hop-based S-G hashing is generally preferable to S-hash and basic S-G hashing in networks with multiple equal-cost upstream paths.

## How Next-Hop-Based S-G Hashing Works

The basic S-G-hash algorithm uses the source and multicast group addresses to select an upstream path.

However, the hash calculation does not incorporate the actual IP addresses of the available next hops.

Next-hop-based S-G hashing adds these addresses to the selection process.

Consider the following topology:

```text id="r8fbzg"
            Src
     10.1.1.10 - .19
             |
             R1
          /  |  \
         /   |   \
        R2   R3   R4
         \   |   /
          \  |  /
             R5
             |
            Rec
```

R5 has three equal-cost paths toward the source network, `10.1.1.0/24`.

| RPF Neighbor | Next-Hop IP | R5 Interface |
|---|---|---|
| R2 | 10.2.5.2 | Ethernet0/0 |
| R3 | 10.3.5.3 | Ethernet0/1 |
| R4 | 10.4.5.4 | Ethernet0/2 |

Suppose the receiver requests the following multicast channel:

```text id="84oqol"
(S,G) = (10.1.1.10, 232.1.1.1)
```

The next-hop-based algorithm considers three types of input:

```text id="yob2mq"
Source: 10.1.1.10
Group:  232.1.1.1

Candidate next hops:
  10.2.5.2
  10.3.5.3
  10.4.5.4
```

The algorithm incorporates the individual next-hop addresses into its hash-based selection, producing a selected RPF neighbor.

Suppose R3 is selected.

R5 sends its `(S,G)` Join toward R3, and the multicast stream travels along:

```text id="bn18oc"
Src → R1 → R3 → R5 → Rec
```

Different `(S,G)` combinations may select R2 or R4 instead.

### RPF Address for Shared Trees

For `(S,G)` states, the RPF address is the multicast source.

For `(*,G)` states, the RPF address is the RP.

Therefore, shared-tree hashing uses the RP address, group address, and candidate next-hop addresses.

The same fundamental next-hop-based selection mechanism can be applied to source and shared trees.

## Avoiding Polarization

One of the main advantages of next-hop-based S-G hashing is that it avoids **polarization**.

As discussed in the previous page, polarization occurs when identical hash calculations on different routers produce correlated path selections, preventing effective use of all available paths.

### S-Hash and Basic S-G-Hash

Both S-hash and basic S-G-hash calculate their hash without considering the actual next-hop addresses.

For example, two routers with two equal-cost paths may independently calculate the same hash result for a particular multicast state.

```text id="2d5kyy"
Router A:
  Path 1 → Next Hop X
  Path 2 → Next Hop Y

Router B:
  Path 1 → Next Hop Z
  Path 2 → Next Hop W
```

With basic S-G hashing, both routers use the same source and group addresses.

Therefore, they calculate the same hash result and select the same relative path position.

In multi-stage ECMP topologies, these correlated decisions can cause certain links to carry most or all of the multicast traffic while others remain unused.

### Next-Hop-Based S-G-Hash

Next-hop-based S-G hashing avoids this problem by incorporating the **actual next-hop IP addresses** into the calculation.

Even when two routers process the same `(S,G)` channel, they generally have different next-hop addresses.

Their hash calculations therefore incorporate different information.

For example:

```text id="df1mzq"
Router A:
  Source = S
  Group  = G
  Next hops = X, Y

Router B:
  Source = S
  Group  = G
  Next hops = Z, W
```

Because the candidate next-hop addresses differ, the routers are not forced to select corresponding path positions simply because the source and group addresses are identical.

**Next-hop-based S-G hashing eliminates the polarization problem associated with S-hash and basic S-G-hash.**

However, it still does not guarantee equal bandwidth utilization. A small number of multicast streams or streams with significantly different traffic rates can still produce uneven traffic distribution.

## Improved RPF Stability During Failures

Another major advantage is **RPF path stability when the available ECMP paths change**.

With S-hash and basic S-G-hash, path selection depends on the number of available equal-cost paths.

If a path fails, the number of paths changes, potentially changing the hash selection for many multicast states.

This can cause unnecessary RPF changes even for streams that were using paths that remain available.

Next-hop-based S-G hashing avoids this behavior.

### Example: Three Equal-Cost Paths

Suppose R5 has three upstream paths:

```text id="bmpr7d"
R5 → R2
R5 → R3
R5 → R4
```

Assume the following multicast streams have been distributed across these paths:

| Multicast Channel | RPF Neighbor |
|---|---|
| (10.1.1.10, 232.1.1.1) | R2 |
| (10.1.1.11, 232.1.1.1) | R3 |
| (10.1.1.12, 232.1.1.1) | R4 |
| (10.1.1.13, 232.1.1.1) | R3 |
| (10.1.1.14, 232.1.1.1) | R2 |
| (10.1.1.15, 232.1.1.1) | R4 |

*These are illustrative selections, not calculated hash results.*

Now suppose the link between R3 and R5 fails.

The available ECMP paths change from:

```text id="ze3d7f"
Before:
  R2, R3, R4

After:
  R2, R4
```

### Behavior with S-Hash or Basic S-G-Hash

With S-hash or basic S-G-hash, the change from three available paths to two changes the path-selection calculation.

Consequently, even streams that previously used R2 or R4 may select a different RPF neighbor.

For example, a stream previously using R2 might move to R4 even though R2 remains available.

This causes unnecessary changes to PIM forwarding state.

### Behavior with Next-Hop-Based S-G-Hash

With next-hop-based hashing, **only the multicast states using the failed path need to select a different upstream path**.

For example:

| Multicast Channel | Before Failure | After Failure |
|---|---|---|
| (10.1.1.10, 232.1.1.1) | R2 | R2 |
| (10.1.1.11, 232.1.1.1) | R3 | R4 |
| (10.1.1.12, 232.1.1.1) | R4 | R4 |
| (10.1.1.13, 232.1.1.1) | R3 | R2 |
| (10.1.1.14, 232.1.1.1) | R2 | R2 |
| (10.1.1.15, 232.1.1.1) | R4 | R4 |

The important points are:

- States using R3 must reconverge to R2 or R4.
- States already using R2 continue using R2.
- States already using R4 continue using R4.

This minimizes unnecessary RPF neighbor changes and reduces the number of PIM forwarding states that must be rebuilt.

**Next-hop-based S-G hashing preserves the RPF selections of states using unaffected ECMP paths.**

### Path Restoration

Suppose the link between R3 and R5 recovers.

R5 again has three equal-cost paths:

```text id="601pe1"
R2, R3, R4
```

With the same hash inputs and next-hop addresses, multicast states that originally selected R3 can return to R3.

States using R2 and R4 remain on their existing paths.

This behavior provides more stable multicast forwarding during both path failures and restoration.

> **Note:** Improved RPF stability does not guarantee lossless multicast convergence. Streams using the failed path can experience packet loss while routing and PIM forwarding state reconverge.

## Predictability and Hashing

Like the other multicast multipath algorithms, next-hop-based S-G hashing is **deterministic**.

No randomization is used in calculating the hash.

Given the same hash inputs and eligible next-hop set, the algorithm produces the same RPF selection.

However, unlike S-hash and basic S-G-hash, the selected path cannot be reliably engineered using only the source or RP and multicast group addresses.

This is because the calculation also depends on the actual next-hop IP addresses.

Consequently:

- The same `(S,G)` channel can select different upstream paths on different routers.
- Changing a next-hop address can affect the selected RPF path.
- Source and group addresses alone are insufficient to predict the selection.

The next-hop-based algorithm uses the same hashing algorithm as IPv6 multicast ECMP, which is based on the hash function used for PIM BSR RP selection.

The exact hash calculation is not normally necessary for configuring or troubleshooting multicast multipath.

## IOS XE Configuration

Configure next-hop-based S-G hashing using the following global configuration command:

```text id="0kdlpe"
R5(config)# ip multicast multipath s-g-hash next-hop-based
```

This enables multicast ECMP load splitting using the source/RPF address, group address, and next-hop addresses.

The command is configured on the **downstream router with multiple equal-cost upstream paths**, which is R5 in this topology.

The available routing information must already contain multiple equal-cost paths toward the source or RP.

### VRF Configuration

Next-hop-based S-G hashing can also be configured for a specific VRF.

For example:

```text id="2oa2gz"
R5(config)# ip multicast vrf A multipath s-g-hash next-hop-based
```

This enables the algorithm for IPv4 multicast traffic associated with VRF `A`.

### Disabling Multicast Multipath

To disable multicast ECMP load splitting:

```text id="vyyp63"
R5(config)# no ip multicast multipath
```

The router returns to its normal IPv4 multicast ECMP RPF selection behavior, preferring the highest-addressed PIM neighbor among equal-cost next hops.

For VRF `A`:

```text id="vn4d8y"
R5(config)# no ip multicast vrf A multipath
```

### Configuration Consistency

Cisco recommends configuring the same multicast multipath algorithm consistently across routers in a redundant topology.

Inconsistent RPF selection can cause PIM forwarding problems, particularly when multiple downstream routers share a LAN and select different upstream routers.

In such cases, the PIM Assert process can override the intended load-splitting decisions.

Also, as with the other multicast multipath algorithms, **PIM neighbor status is not considered when selecting the hashed RPF path**.

The available equal-cost routing next hops determine the candidates, even if one of those next hops is not an established PIM neighbor.

## IPv6 and Bidir-PIM Considerations


### IPv6 Multicast

IPv6 multicast uses the same next-hop-based hashing algorithm as IPv4 next-hop-based S-G hashing.

This provides the same benefits:

- **Load splitting:** Distributes multicast traffic across equal-cost RPF paths.
- **Avoidance of polarization:** Incorporates next-hop addresses into the hash calculation.
- **RPF stability:** Minimizes unnecessary RPF changes when ECMP paths fail or recover.

Unlike IPv4, **IPv6 multicast multipath is enabled by default** on Cisco IOS XE.

It can be explicitly enabled or disabled using the following global configuration commands:

```text
! Enable (default)
R5(config)# ipv6 multicast multipath

! Disable
R5(config)# no ipv6 multicast multipath
```

When disabled, the router selects the **RPF neighbor with the highest IPv6 address** among the equal-cost paths instead of distributing multicast traffic across them.

IPv6 multicast multipath also supports VRF-specific configuration:

```text
R5(config)# ipv6 multicast vrf A multipath
```

Verify the selected RPF path using:

```text
R5# show ipv6 rpf 2001:DB8:1::10
```

Unlike IPv4, IPv6 does not provide separate configuration options for S-hash, basic S-G-hash, and next-hop-based S-G-hash.

### Bidirectional PIM

The next-hop-based S-G-hash algorithm **ignores Bidir-PIM groups**.

Therefore, this algorithm does not provide ECMP load splitting for bidirectional PIM forwarding states.

## Verification

### Verify the Configuration

Check the configured multicast multipath algorithm:

```text id="to6usa"
R5# show running-config | include ip multicast
```

The configuration should include:

```text id="t9ptzz"
ip multicast multipath s-g-hash next-hop-based
```

### Verify Equal-Cost Routes

Verify that R5 has three equal-cost paths toward the source network:

```text id="yys0ko"
R5# show ip route 10.1.1.10
```

The routing table should contain the next hops through R2, R3, and R4.

If only one path is available, there is no opportunity for multicast ECMP load splitting.

### Verify RPF Selection

Use `show ip rpf` with both the source and group addresses:

```text id="c6vk8d"
R5# show ip rpf 10.1.1.10 232.1.1.1
R5# show ip rpf 10.1.1.10 232.1.1.2
R5# show ip rpf 10.1.1.10 232.1.1.3
```

Different groups from the same source may select different RPF interfaces.

Additional sources can also be tested:

```text id="e3j3lw"
R5# show ip rpf 10.1.1.11 232.1.1.1
R5# show ip rpf 10.1.1.12 232.1.1.1
R5# show ip rpf 10.1.1.13 232.1.1.1
```

To verify installed multicast forwarding states:

```text id="uxbw4a"
R5# show ip mroute
```

Compare the incoming interfaces and RPF neighbors for the different `(S,G)` entries.

### Verify Failover Behavior

With multiple multicast states established:

1. Record the current RPF neighbor for each `(S,G)` state.
2. Shut down R5's interface toward R3.
3. Verify that R5 now has only two equal-cost paths toward the source network.
4. Check the RPF selections again.
5. Restore the interface and verify that the original RPF selections return.

For example, shut down the link to R3:

```text id="8pj0b5"
R5(config)# interface Ethernet0/1
R5(config-if)# shutdown
```

After routing reconverges, check the RPF selections:

```text id="58dg8i"
R5# show ip rpf 10.1.1.10 232.1.1.1
R5# show ip rpf 10.1.1.11 232.1.1.1
R5# show ip rpf 10.1.1.12 232.1.1.1
```

States that previously selected R3 should move to R2 or R4.

States that previously selected R2 or R4 should retain those neighbors.

Restore the link:

```text id="1chksz"
R5(config)# interface Ethernet0/1
R5(config-if)# no shutdown
```

After R3 becomes an available equal-cost next hop again, the original hash selections should be restored.

## Comparison of Multicast Multipath Algorithms

| Feature | S-Hash | Basic S-G-Hash | Next-Hop-Based S-G-Hash |
|---|---|---|---|
| Uses source/RPF address | Yes | Yes | Yes |
| Uses group address | No | Yes | Yes |
| Uses next-hop address | No | No | Yes |
| Different groups from the same source can select different paths | No | Yes | Yes |
| Subject to polarization | Yes | Yes | No |
| Preserves unaffected RPF selections after an ECMP path failure | Not guaranteed | Not guaranteed | Yes |
| Supports Bidir-PIM load splitting | No | No | No |

Next-hop-based S-G hashing provides the advantages of basic S-G hashing while addressing polarization and reducing unnecessary RPF changes during path failures.


## Key Points

- Next-hop-based S-G hashing uses the **source/RPF address, group address, and next-hop IP addresses**.
- Configure it with `ip multicast multipath s-g-hash next-hop-based`.
- Different groups from the same source can select different RPF paths.
- Incorporating next-hop addresses **eliminates polarization** associated with S-hash and basic S-G-hash.
- When an ECMP path fails, **only states using the failed path need to change their RPF selection**.
- States using surviving paths retain their existing RPF neighbors.
- When a failed path returns, the original RPF selections can be restored.
- The algorithm is deterministic, but source and group addresses alone cannot reliably predict the selected next hop.
- **IPv6 multicast uses the same next-hop-based hashing algorithm**, with multicast multipath enabled by default.
- IPv6 multicast multipath can be disabled with `no ipv6 multicast multipath`. When disabled, the highest-addressed RPF neighbor is selected.
- Next-hop-based S-G hashing ignores Bidir-PIM groups.
- Use `show ip rpf <source> <group>` for IPv4 and `show ipv6 rpf <source>` for IPv6 to verify RPF selection.

## References

- [Cisco IOS XE: IP Multicast Load Splitting across Equal-Cost Paths](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/26-x/configuration_guide/ip_mcast_rtng/b_26x_ip_mcast_rtng_9300_cg/ip_multicast_optimization__ip_multicast_load_splitting_across_equal_cost_paths.html)
- [Cisco IOS XE: ip multicast multipath Command Reference](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst4500/XE3-10-0E/command/reference/xe-310-cmd/int_sess.html)
