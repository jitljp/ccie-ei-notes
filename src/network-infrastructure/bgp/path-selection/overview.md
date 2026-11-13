# 1.5.b Path Selection

**BGP path selection** is the process of determining which path to a destination prefix is preferred when multiple BGP paths are available.

Unlike OSPF and EIGRP, BGP does not primarily select paths based on a cumulative routing metric. Instead, it uses a **policy-based decision process** that evaluates BGP path attributes in a defined order.

These attributes allow administrators to influence which paths are preferred for reaching particular destinations.

---

## BGP Path Selection Overview

A BGP router may learn multiple paths to the same destination prefix from different neighbors.

Consider the following topology:

```text
             AS 65001
                R1
                 |
                 | EBGP
                 |
                 R2 AS 65002
                 |
                 | EBGP
                 |
                R3
             AS 65003
```

Both R1 and R3 advertise `192.0.2.0/24` to R2.

R2 receives two paths:

| Path | AS_PATH | LOCAL_PREF |
|---|---|---|
| Via R1 | 65001 65010 65020 | 200 |
| Via R3 | 65003 65020 | 100 |

Although the path through R3 has a shorter AS_PATH, **R2 prefers the path through R1** because it has a higher LOCAL_PREF.

This is because **LOCAL_PREF is evaluated before AS_PATH length** in the BGP best path selection algorithm.

BGP does not combine the attributes into a single metric. Instead, it evaluates the selection criteria in order until one path is preferred.

---

## BGP Path Selection Process

The overall process can be divided into three stages:

1. **Path eligibility:** Determine which paths are valid candidates for selection.
2. **Best path selection:** Compare eligible paths using the BGP best path selection algorithm.
3. **RIB installation:** Attempt to install the selected path into the IP routing table.

```text
       Multiple BGP Paths
                |
                v
       +------------------+
       | Path Eligibility |
       +------------------+
                |
                v
       +------------------+
       | Best Path        |
       | Selection        |
       +------------------+
                |
                v
       +------------------+
       | Selected BGP     |
       | Best Path        |
       +------------------+
                |
                v
       +------------------+
       | RIB Installation |
       +------------------+
                |
                v
       IP Routing Table
       (if installed)
```

### Path Eligibility

Before comparing paths, BGP determines whether they are eligible for best path selection.

For example:

- A path with an **unreachable NEXT_HOP** is not eligible.
- A path rejected by an inbound routing policy is not eligible.
- A path rejected because of AS_PATH loop prevention is not eligible.

Paths rejected by policy or AS loop prevention are normally excluded before best path selection.

### Best Path Selection

BGP compares eligible paths using an ordered set of criteria.

Some of the most important criteria include:

| Criterion | Preferred Value |
|---|---|
| Weight | Highest |
| LOCAL_PREF | Highest |
| Local origination | Locally originated |
| AS_PATH length | Shortest |
| ORIGIN | IGP, then EGP, then Incomplete |
| MED | Lowest, when compared |
| EBGP vs. IBGP | EBGP |
| IGP metric to NEXT_HOP | Lowest |

Additional tie-breakers are used if necessary.

The order is significant. **Once a criterion determines the preferred path, subsequent criteria do not override that decision.**

The complete algorithm, including later tie-breakers and configurable behavior, is covered in [BGP Best Path Selection Steps](best-path/selection-steps.md).

### RIB Installation

After selecting its best path, BGP attempts to install it into the appropriate routing table.

However, **the BGP best path is not necessarily installed in the global RIB**.

For example, if OSPF and BGP both provide a route to the same prefix, the router compares their administrative distances.

A route learned through IBGP has a default administrative distance of 200, whereas an OSPF route has an administrative distance of 110.

Therefore, an OSPF route may be installed instead of the selected IBGP best path.

BGP can still retain the best path in its own table.

See [BGP Path Selection and RIB Installation](best-path/rib-installation.md).

---

## BGP Best Path vs. Multipath

By default, BGP selects **one best path** for each destination prefix.

However, BGP also supports **multipath**, allowing multiple eligible paths to be installed in the routing table for load sharing.

Multipath does not eliminate the best path selection algorithm.

BGP still selects a single best path, while additional paths may qualify for installation.

By default, BGP advertises the selected best path to eligible neighbors, subject to routing policies and advertisement rules.

See [BGP Load Balancing](load-balancing.md).

---

## Verification

The following commands are useful when examining BGP path selection:

```text
R1# show bgp ipv4 unicast
R1# show bgp ipv4 unicast 192.0.2.0/24
R1# show ip route 192.0.2.0
R1# show ip cef 192.0.2.1
```

In the BGP table, common status codes include:

| Code | Meaning |
|---|---|
| `*` | Valid BGP path |
| `>` | Selected best path |
| `r` | RIB failure |
| `m` | Multipath candidate |

The `show bgp ipv4 unicast <prefix>` command provides detailed information about the available paths, including their attributes and the reason a particular path was selected.

---

## Topics Covered

This section covers three major aspects of BGP path selection:

1. [BGP Path Attributes](attributes/overview.md): Attribute categories, Weight, LOCAL_PREF, AS_PATH, ORIGIN, MED, and NEXT_HOP.
2. [BGP Best Path Selection Algorithm](best-path/overview.md): Selection order, configuration options, and RIB installation.
3. [BGP Load Balancing](load-balancing.md): EBGP and IBGP multipath, multipath requirements, and load-sharing configuration.

---

## References

- [Cisco: Select BGP Best-path Algorithm](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html)
