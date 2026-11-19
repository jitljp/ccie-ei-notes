# 1.5.d (i) Local-AS, Allowas-in, Remove-Private-AS

Cisco IOS XE supports several mechanisms for manipulating how **Autonomous System Numbers (ASNs)** are handled during BGP peering, route reception, and route advertisement.

Three important mechanisms are:

1. **Local-AS:** Allows a BGP router to use an alternative ASN when establishing a session with a particular neighbor.
2. **Allowas-in:** Allows a BGP router to accept routes containing its own ASN in the AS_PATH.
3. **Remove-Private-AS:** Removes or replaces private ASNs in routes advertised to EBGP neighbors.

Although all three mechanisms relate to AS numbers, they solve different problems.

| Feature | Primary Purpose | Direction |
|---|---|---|
| `local-as` | Use an alternative ASN for a particular BGP neighbor | Peering and AS_PATH processing |
| `allowas-in` | Accept routes containing the local ASN | Inbound |
| `remove-private-as` | Remove or replace private ASNs in advertised routes | Outbound |

---

## AS_PATH and Loop Prevention

The **AS_PATH** attribute records the autonomous systems that a BGP route has traversed.

By default, an EBGP router prepends its local ASN when advertising a route to another AS.

For example:

```text
       AS 65001           AS 65002           AS 65003

          R1 ---------------- R2 ---------------- R3
                  EBGP               EBGP

     192.0.2.0/24
```

R1 originates `192.0.2.0/24`.

The AS_PATH changes as the route travels between autonomous systems:

| Router | AS_PATH |
|---|---|
| R1 | Empty |
| R2 | `65001` |
| R3 | `65002 65001` |

BGP uses the AS_PATH for two important functions:

- **Loop prevention:** Reject routes containing the local ASN.
- **Best-path selection:** Normally prefer shorter AS_PATHs when higher-priority selection criteria are equal.

For example, if R1 receives the following path:

```text
65003 65002 65001
```

R1 normally rejects the route because its own ASN, `65001`, appears in the AS_PATH.

However, some network designs require exceptions to normal AS_PATH processing.

The `local-as`, `allowas-in`, and `remove-private-as` commands provide different ways of modifying this behavior.

See [AS_PATH](../../path-selection/attributes/as-path.md) for detailed AS_PATH operation.

---

## Local-AS

The `neighbor local-as` command allows a BGP router to use an **alternative ASN** for a particular BGP neighbor.

It is primarily used for **AS migrations**, where a router has moved to a new ASN but must continue establishing BGP sessions with neighbors configured to use its old ASN.

### Example

Consider the following topology:

```text
           R1                             R2
       AS 65001                        AS 65002
      (New ASN)

       Local-AS
        65010
          |                                |
          +--------------EBGP--------------+
     10.0.12.1                        10.0.12.2
```

R1 belongs to AS 65001, but R2 is still configured to establish EBGP with R1 using AS 65010.

R1 can use the following configuration:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# neighbor 10.0.12.2 local-as 65010
```

R2 continues using:

```text
R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.12.1 remote-as 65010
```

R1 can establish the EBGP session using AS 65010 without changing its globally configured BGP ASN of 65001.

### Local-AS Options

Cisco IOS XE supports several options for controlling local-AS behavior.

```text
R1(config-router)# neighbor 10.0.12.2 local-as 65010 no-prepend replace-as dual-as
```

| Option | Function |
|---|---|
| `local-as 65010` | Use AS 65010 as the alternative ASN for this neighbor |
| `no-prepend` | Prevent the alternative ASN from being prepended to routes received from the neighbor |
| `replace-as` | Advertise routes toward the neighbor using the alternative ASN instead of the global BGP ASN |
| `dual-as` | Allow the neighbor to establish the session using either the global or alternative ASN |

These options allow gradual transitions between autonomous systems while minimizing disruptions to existing EBGP sessions.

**Important:** `no-prepend` concerns how routes received from the neighbor are processed, while `replace-as` controls outbound AS_PATH behavior toward the neighbor.

The options do not change the ASN configured with `router bgp`.

See [Local-AS](local-as.md) for detailed examples of how each option affects BGP sessions and AS_PATH attributes.

---

## Allowas-in

By default, BGP rejects routes containing the receiving router's own ASN in the AS_PATH.

The `neighbor allowas-in` command allows a router to accept these routes, subject to an optional occurrence limit.

This is useful when the same ASN is intentionally used at multiple locations.

### Example

Consider a customer with two sites connected through a service provider.

Both customer sites use AS 65001.

```text
      Customer Site A          Service Provider        Customer Site B
         AS 65001                  AS 100                  AS 65001

            R1 --------------------- R2 --------------------- R3
                  EBGP                      EBGP

       192.0.2.0/24
```

R1 advertises `192.0.2.0/24` to R2.

R2 learns the route with:

```text
AS_PATH: 65001
```

When R2 advertises the route to R3, the AS_PATH becomes:

```text
AS_PATH: 100 65001
```

R3 normally rejects the route because it belongs to AS 65001, which already appears in the AS_PATH.

However, the route originated at another customer location rather than from R3 itself.

R3 can configure:

```text
R3(config)# router bgp 65001
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# neighbor 10.0.23.2 allowas-in 1
```

R3 can now accept routes containing its own ASN up to one time.

### Allowas-in Occurrence Limit

The optional number specifies how many occurrences of the local ASN are permitted in a received AS_PATH.

For example:

```text
R3(config-router-af)# neighbor 10.0.23.2 allowas-in 2
```

| Received AS_PATH | Result |
|---|---|
| `100 65001` | Accepted |
| `100 65001 200 65001` | Accepted |
| `100 65001 200 65001 300 65001` | Rejected |

When the number is omitted, Cisco IOS XE uses a default limit of **3 occurrences**.

**Important:** `allowas-in` does not remove or replace the local ASN in the received AS_PATH. It changes whether the router accepts a route that would normally fail AS_PATH loop detection.

Since this feature relaxes normal loop prevention, it must be used carefully.

See [Allowas-in](allowas-in.md).

---

## Remove-Private-AS

Private ASNs are frequently used in enterprise networks and customer-to-service-provider BGP connections.

The private ASN ranges are:

| Type | Range |
|---|---|
| 2-byte private ASNs | 64512-65534 |
| 4-byte private ASNs | 4200000000-4294967294 |

Private ASNs are not globally unique and should be removed before routes are advertised to the global Internet.

The `neighbor remove-private-as` command allows a router to remove private ASNs from outgoing EBGP updates.

### Example

Consider the following topology:

```text
          Customer                 ISP                 Internet
          AS 65001               AS 100               AS 200

             R1 ------------------- R2 ------------------- R3
                  EBGP                    EBGP

       192.0.2.0/24
```

R1 originates `192.0.2.0/24`.

R2 receives:

```text
AS_PATH: 65001
```

Without private ASN removal, R3 receives:

```text
AS_PATH: 100 65001
```

R2 can remove the private ASN from its outbound advertisements to R3:

```text
R2(config)# router bgp 100
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.23.3 remove-private-as
```

R3 receives:

```text
AS_PATH: 100
```

The private ASN `65001` is removed, but R2 still prepends its own public ASN of 100.

### Remove-Private-AS Options

Cisco IOS XE supports several options:

```text
R2(config-router-af)# neighbor 10.0.23.3 remove-private-as
R2(config-router-af)# neighbor 10.0.23.3 remove-private-as all
R2(config-router-af)# neighbor 10.0.23.3 remove-private-as all replace-as
```

| Command | Function |
|---|---|
| `remove-private-as` | Remove private ASNs from outgoing EBGP updates |
| `remove-private-as all` | Explicitly remove all private ASNs, including those in mixed public/private AS_PATHs |
| `remove-private-as all replace-as` | Replace private ASNs with the advertising router's ASN |

Modern IOS XE supports removing private ASNs from mixed public/private AS_PATHs. The restriction against mixed AS_PATHs applied to older software releases.

### Removing vs. Replacing Private ASNs

Suppose R2 belongs to AS 100 and has learned a route with:

```text
AS_PATH: 65001 200 65002
```

Without private ASN removal, R2 advertises:

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

The `replace-as` option preserves the number of ASN occurrences by replacing each removed private ASN with the advertising router's ASN.

**Important:** Private ASN removal affects the AS_PATH in outgoing EBGP updates. It does not change the AS_PATH stored in the local BGP table.

Also, Cisco documents this feature for EBGP advertisement from routers using a public ASN, with supported `local-as` configurations providing an alternative in some designs.

See [Remove-Private-AS](remove-private-as.md).

---

## Comparing the Three Mechanisms

The primary differences are summarized below.

| Characteristic | Local-AS | Allowas-in | Remove-Private-AS |
|---|---|---|---|
| Primary purpose | AS migration | Loop-prevention exception | Private ASN removal |
| Changes BGP session ASN? | Yes, for the specified neighbor | No | No |
| Can affect received AS_PATH? | Yes | No | No |
| Can affect advertised AS_PATH? | Yes | No | Yes |
| Allows routes containing the local ASN? | Not its primary function | Yes | No |
| Configured per neighbor? | Yes | Yes | Yes |
| Common use case | Migrating to a new ASN | Repeated customer ASNs | Internet-facing BGP |

### Local-AS vs. Remove-Private-AS Replace-AS

Both mechanisms support a keyword named `replace-as`, but their purposes are different.

**Local-AS `replace-as`:**

```text
neighbor 10.0.12.2 local-as 65010 no-prepend replace-as
```

Uses the alternative ASN in place of the router's global BGP ASN for the neighbor's outgoing AS_PATH processing.

**Remove-Private-AS `replace-as`:**

```text
neighbor 10.0.12.2 remove-private-as all replace-as
```

Replaces private ASNs found in the outgoing AS_PATH with the advertising router's ASN.

These commands are not interchangeable.

### Allowas-in vs. AS-Override

Another related mechanism is **BGP AS-Override**, commonly used in MPLS Layer 3 VPN environments.

Both features can solve problems where multiple customer sites use the same ASN.

However, they operate differently:

| Feature | Behavior |
|---|---|
| `allowas-in` | Receiving router accepts an AS_PATH containing its own ASN |
| `as-override` | Advertising router replaces occurrences of the neighbor's ASN in the outgoing AS_PATH with its own ASN |

For example, in an MPLS L3VPN design:

- `allowas-in` is typically configured on the CE router.
- `as-override` is typically configured on the PE router.

See [BGP AS-Override and Allowas-in](../../../../transport-technologies-and-solutions/mpls/l3vpn/pe-ce/bgp/bgp-as-override-allowas.md) for their use in MPLS L3VPNs.

---

## Verification and Troubleshooting

The following commands are useful when troubleshooting these features.

### Verify BGP Neighbors

```text
R1# show ip bgp summary
R1# show ip bgp neighbors 10.0.12.2
```

Use the detailed neighbor output to verify session parameters, including alternative local-AS configuration where supported.

### Verify AS_PATH Attributes

```text
R1# show ip bgp
R1# show ip bgp 192.0.2.0
```

Inspect the AS_PATH of received routes.

For `allowas-in`, verify that routes containing the local ASN are accepted.

### Verify Outbound Advertisements

```text
R1# show ip bgp neighbors 10.0.12.2 advertised-routes
```

This command displays advertised routes, although the exact final outbound AS_PATH transformations may not be reflected in every IOS XE implementation.

**Verify the received AS_PATH on the neighboring router** when testing `local-as` or `remove-private-as`.

### Verify Configuration

```text
R1# show running-config | section router bgp
```

Check the relevant neighbor configuration.

### Reprocess BGP Routes

After changing route-processing policies, a soft clear may be useful:

```text
R1# clear ip bgp 10.0.12.2 soft in
R1# clear ip bgp 10.0.12.2 soft out
```

However, changes to the ASN used in a BGP OPEN message require the session to be reestablished. A soft clear cannot renegotiate the ASN of an existing session.

---

## Key Points

- **Local-AS** allows a BGP router to use an alternative ASN for a particular neighbor.
- `local-as` is commonly used during AS migrations.
- The `no-prepend`, `replace-as`, and `dual-as` keywords control local-AS behavior.
- `local-as no-prepend` affects received AS_PATH processing.
- `local-as replace-as` affects outgoing AS_PATH processing toward the neighbor.
- `local-as dual-as` allows the neighbor to establish the session using either ASN.
- **Allowas-in** allows a router to accept routes containing its own ASN.
- The optional `allowas-in` number limits how many times the local ASN can appear.
- Without an explicit occurrence count, `allowas-in` permits up to 3 occurrences.
- **Remove-Private-AS** removes private ASNs from outgoing EBGP advertisements.
- `remove-private-as all` supports mixed public/private AS_PATHs.
- `remove-private-as all replace-as` replaces removed private ASNs with the local ASN.
- `local-as replace-as` and `remove-private-as replace-as` perform different functions.
- `allowas-in` and `as-override` are alternative approaches to repeated customer ASN problems, but operate on different routers and in different directions.

---

## Topics Covered

This section contains three detailed pages:

1. **[Local-AS](local-as.md):** Alternative ASNs, AS migrations, `no-prepend`, `replace-as`, `dual-as`, and AS_PATH behavior.
2. **[Allowas-in](allowas-in.md):** AS_PATH loop prevention, occurrence limits, repeated ASN designs, and configuration.
3. **[Remove-Private-AS](remove-private-as.md):** Private ASN removal, mixed AS_PATHs, `all`, `replace-as`, and verification.

---

## References

- [Cisco: BGP Support for Dual AS Configuration for Network AS Migrations](https://www.cisco.com/c/en/us/td/docs/ios/12_2s/feature/guide/fsbgpdas.html)
- [Cisco: BGP Command Reference, neighbor local-as](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [Cisco: Configure Allowas-in Feature in BGP](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/112236-allowas-in-bgp-config-example.html)
- [Cisco: Removing Private AS Numbers from the AS Path in BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-remove-as-0.html)
- [RFC 6996: Autonomous System (AS) Reservation for Private Use](https://www.rfc-editor.org/rfc/rfc6996.html)
