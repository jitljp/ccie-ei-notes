# Remove-Private-AS

The BGP **Remove-Private-AS** feature allows a router to remove or replace private Autonomous System Numbers (ASNs) in AS_PATH attributes advertised to EBGP neighbors.

Private ASNs are commonly used within enterprise networks, data centers, and customer-to-service-provider BGP connections.

However, private ASNs are not globally unique and should not be advertised into the global Internet BGP routing table.

Cisco IOS XE supports private ASN removal using the `neighbor remove-private-as` command.

The feature can:

- Remove private ASNs from outgoing EBGP advertisements.
- Remove private ASNs from AS_PATHs containing both public and private ASNs.
- Replace private ASNs with the advertising router's ASN.
- Preserve AS_PATH length when using the `replace-as` option.
- Apply different private ASN removal policies to different EBGP neighbors or address families.

**Important:** Private ASN removal is an **outbound AS_PATH manipulation feature**. It does not remove private ASNs from routes received into the local BGP table.

---

## Private Autonomous System Numbers

An Autonomous System Number identifies an autonomous system participating in BGP.

ASNs can be either public or private.

**Public ASNs** are globally assigned and intended for use in interdomain routing.

**Private ASNs** are reserved for use within networks that do not require globally unique ASNs.

### Private ASN Ranges

[RFC 6996](https://www.rfc-editor.org/rfc/rfc6996.html) defines two private ASN ranges.

| ASN Type | Private Range |
|---|---|
| 2-byte private ASNs | 64512-65534 |
| 4-byte private ASNs | 4200000000-4294967294 |

Both ranges are supported by modern Cisco IOS XE software.

### Important Reserved ASNs

Not every ASN near the private ranges is actually private.

| ASN | Purpose |
|---|---|
| 0 | Reserved |
| 23456 | AS_TRANS, used for 4-byte ASN compatibility |
| 65535 | Reserved |
| 4294967295 | Reserved |

**65535 is not a private ASN.** Although older Cisco documentation sometimes lists the private range as 64512-65535, RFC 6996 defines the actual 2-byte private range as 64512-65534.

See [Private AS Numbers](../../peering/private-asn.md) for more information.

---

## Why Private ASNs Must Be Removed

Private ASNs are not globally unique.

Two unrelated organizations can independently use the same private ASN.

For example:

```text
     Enterprise A                                Enterprise B
       AS 65001                                    AS 65001
          |                                           |
          R1                                          R4
          |                                           |
       AS 100                 Internet              AS 200
          R2 ----------------------------------------- R3
```

Suppose Enterprise A advertises a route that contains AS 65001.

If the route travels across the Internet without private ASN removal, Enterprise B might receive:

```text
AS_PATH: 200 100 65001
```

Because Enterprise B also uses AS 65001, its BGP router normally rejects the route due to AS_PATH loop prevention.

Even though the two enterprises are unrelated, BGP cannot distinguish their reuse of the same private ASN.

Additionally, Internet service providers commonly filter routes containing private ASNs.

For these reasons, private ASNs should be removed before routes are advertised to the global Internet.

### Private ASNs Are Valid Within Private Networks

Private ASN removal is not required for every EBGP relationship.

For example, an enterprise can establish EBGP sessions using private ASNs:

```text
        AS 65001                  AS 65002

           R1 ---------------------- R2
                      EBGP
```

Both routers operate normally.

Private ASNs participate in:

- EBGP session establishment.
- Normal AS_PATH prepending.
- AS_PATH loop prevention.
- BGP best-path selection.
- AS-path filtering.

The problem arises when private ASNs are propagated into networks where global ASN uniqueness is required.

---

## Remove-Private-AS Command

Cisco IOS XE supports the following command:

```text
R1(config-router-af)# neighbor <IP> remove-private-as [all [replace-as]]
```

The command can also be configured in BGP router configuration mode.

Three common forms are:

```text
R1(config-router-af)# neighbor 10.0.12.2 remove-private-as
R1(config-router-af)# neighbor 10.0.12.2 remove-private-as all
R1(config-router-af)# neighbor 10.0.12.2 remove-private-as all replace-as
```

| Command | Function |
|---|---|
| `remove-private-as` | Remove private ASNs from outgoing EBGP advertisements |
| `remove-private-as all` | Explicitly remove all private ASNs from outgoing AS_PATHs |
| `remove-private-as all replace-as` | Replace private ASNs with the advertising router's ASN |

The feature is disabled by default.

Without `remove-private-as`, BGP normally preserves received private ASNs when advertising routes to another EBGP neighbor.

### Direction of Operation

The command is applied to the **neighbor receiving the modified advertisements**.

Consider:

```text
        Customer                 ISP                  Internet
        AS 65001               AS 100                 AS 200

           R1 ------------------- R2 ------------------- R3
```

R1 advertises routes containing AS 65001 to R2.

R2 should remove the private ASN when advertising those routes to R3.

Therefore, the command is configured on R2 toward R3:

```text
R2(config)# router bgp 100
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.23.3 remove-private-as
```

It is not configured inbound from R1.

**Remove-Private-AS affects outgoing EBGP updates, not incoming updates.**

---

## Basic Remove-Private-AS Example

Consider the following network.

### Topology

```text
          Customer                 ISP                 External AS
          AS 65001               AS 100                 AS 200

             R1 ------------------- R2 ------------------- R3
          E0/0  E0/0           E0/1  E0/0
       10.0.12.1 10.0.12.2   10.0.23.2 10.0.23.3

       Lo0: 192.0.2.1/24
```

R1 originates `192.0.2.0/24`.

R2 receives the route from R1 and advertises it to R3.

The objective is to remove AS 65001 from advertisements sent to R3.

### R1 Configuration

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip address 10.0.12.1 255.255.255.252
R1(config-if)# no shutdown

R1(config)# interface Loopback0
R1(config-if)# ip address 192.0.2.1 255.255.255.0

R1(config)# router bgp 65001
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# neighbor 10.0.12.2 remote-as 100
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# network 192.0.2.0 mask 255.255.255.0
```

### R2 Configuration

```text
R2(config)# interface Ethernet0/0
R2(config-if)# ip address 10.0.12.2 255.255.255.252
R2(config-if)# no shutdown

R2(config)# interface Ethernet0/1
R2(config-if)# ip address 10.0.23.2 255.255.255.252
R2(config-if)# no shutdown

R2(config)# router bgp 100
R2(config-router)# bgp router-id 2.2.2.2
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
R2(config-router)# neighbor 10.0.23.3 remote-as 200
```

### R3 Configuration

```text
R3(config)# interface Ethernet0/0
R3(config-if)# ip address 10.0.23.3 255.255.255.252
R3(config-if)# no shutdown

R3(config)# router bgp 200
R3(config-router)# bgp router-id 3.3.3.3
R3(config-router)# neighbor 10.0.23.2 remote-as 100
```

### Behavior Without Remove-Private-AS

R1 originates `192.0.2.0/24`.

Its local AS_PATH is empty because the route was locally originated.

R1 advertises the route to R2.

R2 receives:

```text
AS_PATH: 65001
```

R2 advertises the route to R3.

Because R2 belongs to AS 100, it prepends AS 100.

R3 receives:

```text
AS_PATH: 100 65001
```

The private ASN 65001 has been propagated into AS 200.

### Enabling Remove-Private-AS

Configure R2:

```text
R2(config)# router bgp 100
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.23.3 remove-private-as
```

R2 removes AS 65001 from the outgoing AS_PATH.

R3 now receives:

```text
AS_PATH: 100
```

### AS_PATH Comparison

| Location | Before Removal | After Removal |
|---|---|---|
| R1 local BGP table | Empty | Empty |
| R2 BGP table | `65001` | `65001` |
| R3 BGP table | `100 65001` | `100` |

**Important:** R2's local BGP table still contains AS 65001.

The private ASN is removed only when R2 prepares the outgoing update toward R3.

This distinction is especially important when verifying the configuration.

---

## Removing Multiple Private ASNs

A route may traverse multiple private autonomous systems before reaching a public AS.

For example:

```text
       AS 65001         AS 65002         AS 65003         AS 100         AS 200

          R1 -------------- R2 -------------- R3 ------------ R4 ------------ R5
```

R1 originates a route.

R4 receives:

```text
AS_PATH: 65003 65002 65001
```

Without private ASN removal, R5 receives:

```text
AS_PATH: 100 65003 65002 65001
```

Configure R4:

```text
R4(config)# router bgp 100
R4(config-router)# address-family ipv4 unicast
R4(config-router-af)# neighbor 10.0.45.5 remove-private-as
```

R4 removes the private ASNs from its outgoing advertisement.

R5 receives:

```text
AS_PATH: 100
```

All three private ASNs have been removed.

### Empty AS_PATH After Removal?

Suppose R4's original received AS_PATH contains only private ASNs.

```text
65003 65002 65001
```

Removing all three produces an empty path before normal EBGP AS prepending.

However, R4 adds its own public ASN when advertising the route to R5.

Therefore, R5 receives:

```text
100
```

The final AS_PATH is not empty.

**Cisco explicitly notes that private ASN removal does not result in an empty advertised AS_PATH in this scenario because the advertising EBGP router prepends its own ASN.**

---

## Removing Private ASNs from Mixed AS_PATHs

An AS_PATH may contain both public and private ASNs.

For example:

```text
65001 200 65002
```

Here:

- `65001` is private.
- `200` is public.
- `65002` is private.

This is called a **mixed AS_PATH**.

### Historical Behavior

Older Cisco IOS implementations did not remove private ASNs from mixed AS_PATHs.

For example, suppose R1 in AS 100 receives:

```text
65001 200 65002
```

On an older implementation, configuring:

```text
R1(config-router)# neighbor 10.0.12.2 remove-private-as
```

Might leave the private ASNs unchanged because the path contains public AS 200.

The neighbor would receive:

```text
100 65001 200 65002
```

This was a restriction of older software releases.

### Modern Cisco IOS XE Behavior

Beginning with Cisco IOS XE Release 3.1S, Cisco enhanced the feature to remove private ASNs from mixed AS_PATHs.

For example, R1 belongs to AS 100 and receives:

```text
65001 200 65002
```

Configure:

```text
R1(config)# router bgp 100
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 remove-private-as
```

The receiving neighbor gets:

```text
100 200
```

Both private ASNs have been removed, while public AS 200 is preserved.

### Using the all Keyword

Cisco IOS XE also supports:

```text
R1(config-router-af)# neighbor 10.0.12.2 remove-private-as all
```

The `all` keyword explicitly requests removal of all private ASNs from the outgoing AS_PATH.

For the same route:

```text
Original received AS_PATH:
65001 200 65002

Advertised AS_PATH:
100 200
```

**Important:** On modern Cisco IOS XE, the basic `remove-private-as` command already supports removing private ASNs from mixed public/private AS_PATHs.

Therefore, do not assume that `all` is required merely because the AS_PATH contains both public and private ASNs.

The historical mixed-AS_PATH restriction does not apply to modern IOS XE.

---

## Replacing Private ASNs

Removing private ASNs shortens the AS_PATH.

Because AS_PATH length is an important BGP best-path selection criterion, changing its length may influence how external routers select routes.

Cisco IOS XE supports the `replace-as` keyword to replace private ASNs with the advertising router's ASN instead of removing them completely.

### Configuration

```text
R1(config)# router bgp 100
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 remove-private-as all replace-as
```

**The `replace-as` keyword requires `all`.**

The following syntax is not supported:

```text
neighbor 10.0.12.2 remove-private-as replace-as
```

The correct syntax is:

```text
neighbor 10.0.12.2 remove-private-as all replace-as
```

### Example 1: All Private ASNs

Suppose R1 belongs to AS 100.

R1 receives:

```text
AS_PATH: 65001 65002 65003
```

Without private ASN removal, the neighbor receives:

```text
100 65001 65002 65003
```

With:

```text
R1(config-router-af)# neighbor 10.0.12.2 remove-private-as all
```

The neighbor receives:

```text
100
```

With:

```text
R1(config-router-af)# neighbor 10.0.12.2 remove-private-as all replace-as
```

The neighbor receives:

```text
100 100 100 100
```

The leftmost `100` is R1's normal EBGP prepend.

The other three occurrences replace the three private ASNs.

### Example 2: Mixed Public and Private ASNs

Suppose R1 belongs to AS 100.

R1 receives:

```text
AS_PATH: 65001 200 65002
```

Without removal:

```text
100 65001 200 65002
```

With `remove-private-as all`:

```text
100 200
```

With `remove-private-as all replace-as`:

```text
100 100 200 100
```

The public ASN `200` remains unchanged.

Both private ASNs are replaced with R1's ASN, `100`.

### Example 3: Repeated Private ASNs

Consider:

```text
AS_PATH: 65001 65001 65002
```

R1 belongs to AS 100.

With `remove-private-as all`:

```text
100
```

With `remove-private-as all replace-as`:

```text
100 100 100 100
```

Each occurrence of a private ASN is replaced individually.

**The replacement operation does not collapse repeated ASNs into a single occurrence.**

---

## Preserving AS_PATH Length

The `replace-as` option is useful because it preserves the number of AS_SEQUENCE ASN occurrences.

Consider R1 in AS 100 receiving:

```text
65001 200 65002 65003
```

The received AS_PATH length is 4.

Without private ASN removal, R1 advertises:

```text
100 65001 200 65002 65003
```

Length: 5

With `remove-private-as all`:

```text
100 200
```

Length: 2

With `remove-private-as all replace-as`:

```text
100 100 200 100 100
```

Length: 5

### Comparison

| Configuration | Advertised AS_PATH | Length |
|---|---|---:|
| No removal | `100 65001 200 65002 65003` | 5 |
| `remove-private-as` | `100 200` | 2 |
| `remove-private-as all` | `100 200` | 2 |
| `remove-private-as all replace-as` | `100 100 200 100 100` | 5 |

The `replace-as` option preserves the path length that would have been advertised without private ASN removal.

This matters because BGP normally prefers the shortest AS_PATH when earlier best-path selection criteria are equal.

### Why Preserve the Length?

Suppose an external router has two paths to the same prefix.

```text
Path A: 100 200
Path B: 300 400 500
```

Path A has length 2.

Path B has length 3.

If earlier criteria are equal, the external router prefers Path A.

However, suppose Path A originally traversed several private ASes:

```text
100 65001 65002 200
```

After removal, the AS_PATH becomes:

```text
100 200
```

The path now appears shorter.

Using `replace-as` retains the original length:

```text
100 100 100 200
```

This can influence path selection by avoiding an artificial reduction in AS_PATH length.

**Important:** Preserving AS_PATH length does not guarantee identical BGP path selection.

Other attributes and policies may still influence the selected path.

Also, replacing ASNs changes the AS_PATH's contents, which may affect AS-path filtering and loop-prevention behavior on downstream routers.

---

## Remove-Private-AS Configuration Summary

The three command forms can be compared using a single example.

Assume:

- R1 belongs to public AS 100.
- R1 receives a route with AS_PATH `65001 200 65002`.
- R1 advertises the route to an EBGP neighbor.

| Command | AS_PATH Advertised |
|---|---|
| No removal | `100 65001 200 65002` |
| `remove-private-as` | `100 200` |
| `remove-private-as all` | `100 200` |
| `remove-private-as all replace-as` | `100 100 200 100` |

### Key Distinctions

**`remove-private-as`:**

- Removes private ASNs.
- Reduces AS_PATH length.
- Supports mixed AS_PATHs on modern IOS XE.

**`remove-private-as all`:**

- Explicitly removes all private ASNs.
- Reduces AS_PATH length.
- Is also required when using `replace-as`.

**`remove-private-as all replace-as`:**

- Replaces private ASNs with the advertising router's ASN.
- Preserves AS_PATH length.
- Does not preserve the original private ASN values.

---

## Remove-Private-AS Restrictions

Cisco IOS XE documents several restrictions.

### EBGP Neighbors Only

The feature applies to **EBGP neighbors**, not ordinary IBGP neighbors.

For example:

```text
         AS 100                     AS 200

           R1 ------------------------ R2
                       EBGP
```

R1 can configure:

```text
R1(config-router-af)# neighbor 10.0.12.2 remove-private-as
```

However, the feature is not intended for removing private ASNs in ordinary IBGP advertisements between routers belonging to the same AS.


### Public ASN Requirement

Cisco documents that `remove-private-as` normally requires the advertising router to use a **public ASN**.

For example:

```text
R1(config)# router bgp 100
```

R1 can use `remove-private-as` to remove private ASNs from routes advertised to EBGP neighbors.

However, consider:

```text
R1(config)# router bgp 65001
```

R1 itself uses a private ASN.

The `remove-private-as` command cannot simply remove R1's own ASN from normal EBGP advertisements because the router automatically prepends its ASN when advertising routes to an EBGP neighbor.

#### Workaround: Local-AS with a Public ASN

Cisco documents a workaround using `neighbor local-as` with the `no-prepend replace-as` options.

Consider the following topology:

```text
       R1                            R2
    AS 65001                       AS 200
   (Private)                      (Public)
        |                             |
        +------------EBGP-------------+

       R1 advertises 10.1.1.0/24
```

R1 uses private AS 65001 internally but wants to advertise routes to R2 using public AS 100.

Configure R1:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 200
R1(config-router)# neighbor 10.0.12.2 local-as 100 no-prepend replace-as
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 remove-private-as
R1(config-router-af)# network 10.1.1.0 mask 255.255.255.0
```

Configure R2 to expect AS 100:

```text
R2(config)# router bgp 200
R2(config-router)# neighbor 10.0.12.1 remote-as 100
```

Assuming R1 has a matching route to `10.1.1.0/24`, it advertises the prefix to R2.

**Without the workaround**, R2 would receive:

```text
AS_PATH: 65001
```

**With the workaround**, R2 receives:

```text
AS_PATH: 100
```

The commands perform different functions:

| Command | Function |
|---|---|
| `local-as 100` | Use public AS 100 for the EBGP session with R2 |
| `no-prepend` | Do not add the alternative ASN to routes received from R2 |
| `replace-as` | Prepend public AS 100 instead of private AS 65001 in advertisements toward R2 |
| `remove-private-as` | Remove other private ASNs already present in outgoing AS_PATHs |

#### Why Is remove-private-as Still Needed?

Suppose R1 learns a route from another EBGP neighbor in private AS 65002.

R1's BGP table contains:

```text
AS_PATH: 65002
```

With only `local-as 100 no-prepend replace-as`, R2 receives:

```text
AS_PATH: 100 65002
```

R1's own private ASN, 65001, is hidden, but private AS 65002 remains.

With `remove-private-as` also configured, R2 receives:

```text
AS_PATH: 100
```

Therefore, the two features have complementary purposes:

- **`local-as ... replace-as`** prevents the router's own private ASN from being prepended to outgoing advertisements.
- **`remove-private-as`** removes private ASNs already present in the route's AS_PATH.

**Important:** The router still operates under its actual ASN of 65001. The public ASN is used only for the specified neighbor relationship. In a production Internet-facing deployment, the public ASN must be authorized for use by the organization.

See [Local-AS](local-as.md) for detailed explanations of `no-prepend`, `replace-as`, and `dual-as`.

### Per-Neighbor Operation

Private ASN removal is configured separately for each neighbor.

For example:

```text
R1(config)# router bgp 100
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 remove-private-as
```

This affects updates sent to `10.0.12.2`.

It does not automatically affect updates sent to other EBGP neighbors.

### Per-Address-Family Operation

Modern Cisco IOS XE supports configuring the feature separately for different address families.

For example:

```text
R1(config)# router bgp 100
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 remove-private-as all
```

This applies the feature to the IPv4 unicast address family.

Other address families can use different policies.

The feature can also be inherited through supported peer-group or peer-template configurations.

### AS_PATH Loop Prevention

Private ASNs normally participate in AS_PATH loop prevention just like public ASNs.

Removing or replacing private ASNs changes the information available to downstream routers.

For example, a router belonging to AS 65001 normally rejects:

```text
AS_PATH: 100 65001
```

However, if AS 65001 has been removed, it might receive:

```text
AS_PATH: 100
```

The local ASN no longer appears.

Therefore, private ASN removal can affect AS_PATH loop detection.

**Private ASN removal should be applied at appropriate routing boundaries, not indiscriminately throughout a private BGP network.**

---

## Remove-Private-AS vs. Other AS_PATH Manipulations

Several Cisco BGP features modify AS_PATH behavior.

Although they may produce similar-looking AS_PATHs, their purposes are different.

| Feature | Purpose |
|---|---|
| `neighbor remove-private-as` | Remove private ASNs from outgoing updates |
| `neighbor remove-private-as all replace-as` | Replace private ASNs with the advertising ASN |
| `neighbor local-as` | Use an alternative ASN for a particular neighbor |
| `neighbor local-as ... replace-as` | Use the alternative ASN instead of the real ASN in outgoing updates |
| `neighbor as-override` | Replace occurrences of a neighbor's ASN with the advertising router's ASN |
| `set as-path prepend` | Add ASN occurrences to an AS_PATH |

### Remove-Private-AS vs. Local-AS Replace-AS

Consider:

```text
neighbor 10.0.12.2 remove-private-as all replace-as
```

This replaces **private ASNs already in the outgoing AS_PATH** with the router's ASN.

Compare:

```text
neighbor 10.0.12.2 local-as 65010 no-prepend replace-as
```

This controls which ASN the router presents toward a particular neighbor during an AS migration.

The `replace-as` keyword has different meanings in these commands.

### Remove-Private-AS vs. AS-Override

`remove-private-as` removes or replaces private ASNs based on whether the ASN belongs to a private range.

`as-override` replaces occurrences of the remote neighbor's ASN in outgoing AS_PATHs.

The remote neighbor's ASN does not need to be private for AS-Override to operate.

AS-Override is commonly used when multiple customer sites share an ASN in an MPLS L3VPN environment.

See [BGP AS-Override and Allowas-in](../../../../transport-technologies-and-solutions/mpls/l3vpn/pe-ce/bgp/bgp-as-override-allowas.md).

---

## Private ASN Removal and Confederations

BGP confederations use special AS_PATH segment types to identify member autonomous systems.

These include:

- `AS_CONFED_SEQUENCE`
- `AS_CONFED_SET`

Older Cisco software had additional restrictions on private ASN removal when AS_PATHs contained confederation segments.

For example, older implementations could remove private ASNs only when they appeared after the confederation portion of the AS_PATH.

Cisco IOS XE Release 3.1S introduced enhanced behavior that supports removing private ASNs appearing before confederation segments.

**Important:** Confederation AS_PATH handling is more complicated than ordinary AS_SEQUENCE processing.

Confederation segments are normally removed when routes are advertised outside the confederation.

Private ASN removal should not be confused with this normal confederation behavior.

See [AS_PATH](../../path-selection/attributes/as-path.md) for information about AS_PATH segment types.

---

## Verification and Troubleshooting

Private ASN removal affects outbound BGP updates.

Therefore, verifying the feature requires inspecting both the advertising router and the receiving neighbor.

### Verify the Local BGP Table

On R2:

```text
R2# show ip bgp
R2# show ip bgp 192.0.2.0
```

Suppose R2 has received:

```text
AS_PATH: 65001
```

Even after configuring:

```text
R2(config-router-af)# neighbor 10.0.23.3 remove-private-as
```

R2's local BGP table still contains:

```text
AS_PATH: 65001
```

This is expected.

The command does not modify the stored received AS_PATH.

### Verify the Receiving Neighbor

On R3:

```text
R3# show ip bgp 192.0.2.0
```

After private ASN removal, R3 should see:

```text
AS_PATH: 100
```

This is the most reliable way to confirm that private ASN removal is working.

### Verify Advertised Routes

On R2:

```text
R2# show ip bgp neighbors 10.0.23.3 advertised-routes
```

This command helps identify prefixes advertised to R3.

However, depending on the IOS XE implementation, the command may display the locally selected AS_PATH rather than every final outbound AS_PATH transformation.

**Do not rely only on `advertised-routes` to verify private ASN removal. Inspect the route on the receiving BGP neighbor.**

### Verify Neighbor Configuration

```text
R2# show running-config | section router bgp
```

Look for:

```text
neighbor 10.0.23.3 remove-private-as
```

Or:

```text
neighbor 10.0.23.3 remove-private-as all replace-as
```

### Verify Detailed Neighbor Information

```text
R2# show ip bgp neighbors 10.0.23.3
```

On supported IOS XE versions, detailed neighbor output provides information about private ASN removal or replacement.

### Verify Update Groups

```text
R2# show ip bgp update-group
```

BGP update groups can reflect different outbound policies applied to different neighbors.

Cisco documents this command as a method of verifying whether private ASN removal or replacement is enabled.

### Reapply Outbound Policies

After changing private ASN removal configuration, reapply outbound BGP processing if necessary:

```text
R2# clear ip bgp 10.0.23.3 soft out
```

This normally allows the updated advertisements to be sent without resetting the BGP session.

---

## Common Troubleshooting Scenarios

### Private ASN Still Appears on the Receiving Router

**Problem:** R3 receives:

```text
100 65001
```

Even though R2 should remove AS 65001.

Check:

1. Whether `remove-private-as` is configured toward R3.
2. Whether R2 is using an eligible public ASN.
3. Whether the configuration is applied to the correct address family.
4. Whether the outbound policy has been reapplied.
5. Whether R3 is receiving the route from the expected neighbor.

### Private ASN Removal Works for One Neighbor but Not Another

**Problem:** R2 removes private ASNs toward R3 but not R4.

**Cause:** The feature is configured per neighbor.

For example:

```text
R2(config-router-af)# neighbor 10.0.23.3 remove-private-as
```

This does not automatically enable removal toward R4.

Configure the appropriate policy for the other neighbor if required.

### AS_PATH Becomes Much Shorter

**Problem:** A route originally has:

```text
65001 65002 65003
```

After private ASN removal by AS 100, the neighbor receives:

```text
100
```

**Cause:** Private ASN removal eliminates all three private ASNs.

If preserving AS_PATH length is required, configure:

```text
R2(config-router-af)# neighbor 10.0.23.3 remove-private-as all replace-as
```

The neighbor receives:

```text
100 100 100 100
```

### Mixed AS_PATH Is Not Modified

**Problem:** A router does not remove private ASNs from a path containing both public and private ASNs.

**Possible cause:** An older IOS implementation with the historical mixed-AS_PATH restriction.

Modern Cisco IOS XE supports removing private ASNs from mixed paths.

Check the software version and the configured outbound policy before assuming that the `all` keyword is required.

---

## Command Summary

| Command | Purpose |
|---|---|
| `neighbor <IP> remove-private-as` | Remove private ASNs from outbound EBGP updates |
| `neighbor <IP> remove-private-as all` | Explicitly remove all private ASNs |
| `neighbor <IP> remove-private-as all replace-as` | Replace private ASNs with the local ASN |
| `no neighbor <IP> remove-private-as` | Disable private ASN removal |
| `show ip bgp` | Display BGP routes and AS_PATH attributes |
| `show ip bgp <PREFIX>` | Inspect the AS_PATH of a specific route |
| `show ip bgp neighbors <IP> advertised-routes` | Display advertised prefixes |
| `show ip bgp neighbors <IP>` | Display detailed neighbor information |
| `show ip bgp update-group` | Display BGP update-group information |
| `clear ip bgp <IP> soft out` | Reapply outbound BGP processing |

---

## Key Points

- Private ASNs are not globally unique.
- The private 2-byte ASN range is **64512-65534**.
- The private 4-byte ASN range is **4200000000-4294967294**.
- **65535 is reserved**, not private.
- Private ASNs should not be propagated into the global Internet BGP routing table.
- Private ASN removal is disabled by default.
- `neighbor remove-private-as` removes private ASNs from **outbound EBGP updates**.
- The feature applies to EBGP neighbors, not ordinary IBGP neighbors.
- Cisco documents the feature for routers using a public ASN, with a Local-AS workaround for some designs.
- The command does **not** modify the local BGP table.
- Private ASN removal can shorten the advertised AS_PATH.
- On modern IOS XE, `remove-private-as` supports mixed public/private AS_PATHs.
- The historical restriction against mixed AS_PATHs does not apply to modern IOS XE.
- `remove-private-as all` explicitly removes all private ASNs.
- `remove-private-as all replace-as` replaces private ASNs with the local ASN.
- The `replace-as` option requires the `all` keyword.
- Each removed private ASN is replaced individually.
- `replace-as` preserves AS_PATH length relative to the unmodified advertisement.
- Private ASN removal can affect remote BGP best-path selection.
- Private ASN removal can also affect AS_PATH loop prevention and filtering.
- The feature can be configured per neighbor and per address family.
- **Verify the final AS_PATH on the receiving BGP neighbor**, not only on the advertising router.

---

## References

- [Cisco IOS XE 17.x: Removing Private AS Numbers from the AS Path in BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-remove-as-0.html)
- [Cisco: BGP Command Reference, neighbor remove-private-as](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [RFC 6996: Autonomous System (AS) Reservation for Private Use](https://www.rfc-editor.org/rfc/rfc6996.html)
- [RFC 6793: BGP Support for Four-Octet ASN Space](https://www.rfc-editor.org/rfc/rfc6793.html)
- [IANA: Special-Purpose Autonomous System Numbers](https://www.iana.org/assignments/iana-as-numbers-special-registry/)
