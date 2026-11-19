# Local-AS

The BGP **Local-AS** feature allows a router to use an alternative Autonomous System Number (ASN) for a specific BGP neighbor without changing the ASN configured under `router bgp`.

It is primarily used during **AS migrations**, when a network changes its ASN but must maintain EBGP sessions with neighbors that are still configured to use the old ASN.

For example, a service provider might migrate from AS 65010 to AS 65001 while allowing customers to continue peering with AS 65010 until their configurations can be updated.

The `neighbor local-as` command allows the provider router to:

- Present an alternative ASN in BGP OPEN messages.
- Modify the AS_PATH attribute of routes received from a neighbor.
- Modify the AS_PATH attribute of routes advertised to a neighbor.
- Support EBGP sessions using either the old or new ASN during a migration.

On Cisco IOS XE, the command supports the following syntax:

```text
R1(config-router)# neighbor <IP> local-as <ASN> [no-prepend [replace-as [dual-as]]]
```

The optional keywords control different aspects of BGP behavior.

| Keyword | Function |
|---|---|
| `local-as <ASN>` | Configure an alternative ASN for the neighbor |
| `no-prepend` | Do not prepend the alternative ASN to received routes |
| `replace-as` | Advertise the alternative ASN instead of the real ASN toward the neighbor |
| `dual-as` | Allow the neighbor to establish a session using either the real or alternative ASN |

The most important distinction is that **`no-prepend` affects inbound AS_PATH processing, whereas `replace-as` affects outbound AS_PATH processing**.

---

## Local-AS Fundamentals

A BGP router normally identifies itself using the ASN configured under `router bgp`.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
```

R1 identifies itself as AS 65001 when establishing its EBGP session with R2.

R2 must therefore configure:

```text
R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
```

If R2 expects R1 to belong to AS 65010 instead, the BGP session will fail because the ASN in R1's BGP OPEN message does not match R2's expected remote ASN.

The `local-as` command solves this problem.

### Configuring an Alternative ASN

R1 can configure:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# neighbor 10.0.12.2 local-as 65010
```

R2 can continue using:

```text
R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.12.1 remote-as 65010
```

R1 now presents AS 65010 to R2.

The BGP session can establish even though R1's globally configured ASN is 65001.

Importantly:

- R1's real ASN remains 65001.
- R1's alternative ASN for R2 is 65010.
- R1 can use its real ASN when peering with other neighbors.
- The `local-as` configuration is specific to the neighbor or peer group.

**Local-AS does not create a second independent BGP routing process.**

The router continues operating within its globally configured BGP AS while using the alternative ASN for the specified peering relationship.

---

## Local-AS and BGP OPEN Messages

BGP peers exchange OPEN messages when establishing a BGP session.

The OPEN message includes the sender's ASN.

Normally, R1 sends:

```text
BGP OPEN
My Autonomous System: 65001
```

However, after configuring:

```text
R1(config-router)# neighbor 10.0.12.2 local-as 65010
```

R1 presents:

```text
BGP OPEN
My Autonomous System: 65010
```

R2 accepts the session because it expects AS 65010.

**Important:** The alternative ASN is used for the specified neighbor's BGP session. It does not replace the ASN configured under `router bgp`.

The `dual-as` option extends this behavior by allowing the session to use either ASN.

---

## Local-AS and AS_PATH Manipulation

In addition to changing the ASN used for the BGP session, `local-as` affects the AS_PATH attribute.

By default, the alternative ASN is included in both inbound and outbound AS_PATH processing.

The following topology will be used to demonstrate the behavior.

### Topology

```text
       R2                    R1                    R3
    AS 65002              AS 65001              AS 65003
                          (Real AS)

                        Local-AS
                         65010
       |                    |                     |
       +--------------------+---------------------+
          10.0.12.0/24           10.0.13.0/24
             EBGP                   EBGP

    192.0.2.0/24                           203.0.113.0/24
```

R1 has:

- Real ASN: 65001
- Alternative ASN toward R2: 65010
- EBGP neighbor R2: AS 65002
- EBGP neighbor R3: AS 65003

R2 originates `192.0.2.0/24`.

R3 originates `203.0.113.0/24`.

### R1 Configuration

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# neighbor 10.0.12.2 local-as 65010
R1(config-router)# neighbor 10.0.13.3 remote-as 65003
```

R1 uses the alternative ASN only toward R2.

### R2 Configuration

```text
R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.12.1 remote-as 65010
R2(config-router)# network 192.0.2.0 mask 255.255.255.0
```

### R3 Configuration

```text
R3(config)# router bgp 65003
R3(config-router)# neighbor 10.0.13.1 remote-as 65001
R3(config-router)# network 203.0.113.0 mask 255.255.255.0
```

The `network` statements assume the corresponding prefixes exist in the local routing tables.

IPv4 unicast neighbors are activated by default on IOS XE unless this behavior has been disabled.

### Inbound AS_PATH Processing

R2 advertises `192.0.2.0/24` to R1.

R2 sends:

```text
AS_PATH: 65002
```

Because R1 uses `local-as 65010` without `no-prepend`, R1 adds the alternative ASN to the received AS_PATH.

R1's BGP table contains:

```text
AS_PATH: 65010 65002
```

Notice that R1's real ASN, 65001, is not added to the received AS_PATH at this stage.

The alternative ASN is added during inbound processing.

### Outbound AS_PATH Processing

R1 advertises the route learned from R2 to R3.

Because R3 is an ordinary EBGP neighbor using R1's real ASN, R1 prepends AS 65001.

R3 receives:

```text
AS_PATH: 65001 65010 65002
```

Now consider the opposite direction.

R3 advertises `203.0.113.0/24` to R1.

R1 receives:

```text
AS_PATH: 65003
```

When R1 advertises the route to R2, it prepends both its alternative ASN and its real ASN.

R2 receives:

```text
AS_PATH: 65010 65001 65003
```

The leftmost ASN is 65010, matching the ASN R2 expects from its EBGP neighbor.

### Summary of Default Local-AS Behavior

| Direction | AS_PATH |
|---|---|
| R2 sends to R1 | `65002` |
| R1 stores route from R2 | `65010 65002` |
| R1 advertises route to R3 | `65001 65010 65002` |
| R3 sends to R1 | `65003` |
| R1 advertises route to R2 | `65010 65001 65003` |

The default behavior introduces additional ASN occurrences into the AS_PATH.

This can affect best-path selection, route filtering, and loop prevention.

---

## Local-AS with no-prepend

The `no-prepend` keyword prevents the alternative ASN from being added to AS_PATHs of routes received from the specified neighbor.

Configure:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 local-as 65010 no-prepend
```

### Inbound Behavior

Without `no-prepend`, R1 receives the route from R2 and adds AS 65010:

```text
AS_PATH: 65010 65002
```

With `no-prepend`, R1 preserves the received AS_PATH:

```text
AS_PATH: 65002
```

The alternative ASN is no longer inserted during inbound processing.

### Advertisement to Another EBGP Neighbor

When R1 advertises the route to R3, it performs normal EBGP prepending using its real ASN.

R3 receives:

```text
AS_PATH: 65001 65002
```

Compare:

| Configuration | AS_PATH Received by R3 |
|---|---|
| `local-as 65010` | `65001 65010 65002` |
| `local-as 65010 no-prepend` | `65001 65002` |

Therefore, `no-prepend` prevents the old ASN from propagating into the wider network through routes received on the local-AS session.

### Outbound Behavior Toward the Local-AS Neighbor

The `no-prepend` keyword does **not** remove the real ASN from advertisements sent toward R2.

For example, R1 learns:

```text
AS_PATH: 65003
```

From R3.

R1 advertises the route to R2 with:

```text
AS_PATH: 65010 65001 65003
```

This is unchanged from the default local-AS outbound behavior.

**Key distinction:** `no-prepend` prevents the alternative ASN from being added during inbound processing. It does not hide the real ASN in outbound updates sent toward the local-AS neighbor.

To hide the real ASN from that neighbor, use `replace-as`.

---

## Local-AS with no-prepend replace-as

The `replace-as` keyword changes **outbound AS_PATH processing**.

It prevents the real BGP ASN from being prepended when advertising routes to the local-AS neighbor.

Only the alternative ASN is prepended for that peering relationship.

On Cisco IOS XE, `replace-as` is configured after `no-prepend`.

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 local-as 65010 no-prepend replace-as
```

### Outbound Behavior

R1 learns `203.0.113.0/24` from R3.

Its AS_PATH is:

```text
65003
```

Without `replace-as`, R1 advertises the route toward R2 with:

```text
65010 65001 65003
```

With `replace-as`, R1 advertises:

```text
65010 65003
```

The real ASN, 65001, is no longer prepended.

### Comparison

| Configuration | AS_PATH Advertised to R2 |
|---|---|
| `local-as 65010` | `65010 65001 65003` |
| `local-as 65010 no-prepend` | `65010 65001 65003` |
| `local-as 65010 no-prepend replace-as` | `65010 65003` |

The `replace-as` keyword makes the router appear to belong only to AS 65010 in the outgoing AS_PATH toward R2.

### Inbound Behavior

Because `replace-as` is configured together with `no-prepend`, received routes do not have the alternative ASN inserted.

For example:

R2 advertises:

```text
AS_PATH: 65002
```

R1 stores:

```text
AS_PATH: 65002
```

When R1 advertises the route to R3, it prepends its real ASN:

```text
AS_PATH: 65001 65002
```

Thus, with `no-prepend replace-as`:

- R2 sees only R1's alternative ASN in the AS_PATH added toward R2.
- R3 sees R1's real ASN when routes are advertised toward R3.
- Routes received from R2 are not artificially prepended with AS 65010.

**Important:** `replace-as` controls the ASN R1 prepends toward the configured neighbor. It does not remove arbitrary occurrences of AS 65001 that may already exist in a route's AS_PATH.

---

## Local-AS with no-prepend replace-as dual-as

The `dual-as` keyword allows a BGP neighbor to establish a session using either:

- The router's real ASN, configured under `router bgp`.
- The alternative ASN, configured with `neighbor local-as`.

It is especially useful during staged AS migrations.

### Example

R1 belongs to AS 65001 but previously belonged to AS 65010.

R2 is still configured to peer with AS 65010.

```text
         R1                              R2
      AS 65001                        AS 65002
      (New ASN)

      Local-AS
       65010
         |                                |
         +--------------EBGP--------------+
```

R1 configures:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# neighbor 10.0.12.2 local-as 65010 no-prepend replace-as dual-as
```

R2 initially uses:

```text
R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.12.1 remote-as 65010
```

The EBGP session can establish using AS 65010.

### Migrating the Neighbor

At a later time, R2 can be updated:

```text
R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
```

Because R1 has `dual-as` configured, R1 can also establish the session using its real ASN of 65001.

Without `dual-as`, R1 would normally continue presenting only the alternative ASN to R2, preventing the session from establishing after R2 changes its expected remote ASN.

### What dual-as Does Not Do

The `dual-as` keyword does not:

- Create two simultaneous BGP sessions with the same neighbor.
- Change the globally configured BGP ASN.
- Automatically update the neighbor's configuration.
- Eliminate the need for BGP session renegotiation when the neighbor changes its expected ASN.

Instead, it provides flexibility during the transition.

A peer can be migrated from the old ASN to the new ASN without requiring another configuration change on R1.

**Important:** The session must still be reestablished when R2's `remote-as` configuration changes. `dual-as` avoids an additional coordinated configuration change on R1, rather than eliminating every session interruption.

---

## Comparing Local-AS Options

The following table summarizes the principal configurations.

Assume:

- R1 real ASN: 65001
- R1 alternative ASN: 65010
- R2 ASN: 65002
- R3 ASN: 65003

R1 learns `192.0.2.0/24` from R2 and `203.0.113.0/24` from R3.

### AS_PATH Comparison

| Configuration on R1 toward R2 | R1's AS_PATH for Route from R2 | AS_PATH Advertised from R1 to R2 for Route from R3 |
|---|---|---|
| `local-as 65010` | `65010 65002` | `65010 65001 65003` |
| `local-as 65010 no-prepend` | `65002` | `65010 65001 65003` |
| `local-as 65010 no-prepend replace-as` | `65002` | `65010 65003` |
| `local-as 65010 no-prepend replace-as dual-as` | `65002` | `65010 65003` when peering using the alternative ASN |

The `dual-as` keyword additionally permits peering with the real ASN. When a session is established using the real ASN, AS migration translation is unnecessary, and the resulting AS_PATH should be verified on the receiving neighbor.

### Functional Comparison

| Keyword | Inbound AS_PATH | Outbound AS_PATH Toward Neighbor | BGP Session |
|---|---|---|---|
| `local-as` | Add alternative ASN | Prepend alternative and real ASNs | Alternative ASN |
| `no-prepend` | Do not add alternative ASN | No change from default local-AS behavior | Alternative ASN |
| `replace-as` | No additional change | Prepend alternative ASN instead of real ASN | Alternative ASN |
| `dual-as` | Depends on negotiated session and other options | Depends on negotiated session and other options | Real or alternative ASN |

**Remember:**

- `no-prepend` = inbound AS_PATH behavior.
- `replace-as` = outbound AS_PATH behavior.
- `dual-as` = BGP session ASN negotiation.

---

## AS Migration Example

Consider a service provider migrating from AS 65010 to AS 65001.

The provider has multiple customer connections and wants to avoid reconfiguring all customers simultaneously.

### Before Migration

```text
     Customer A             Provider              Customer B
      AS 65100              AS 65010               AS 65200

         R2 ------------------- R1 ------------------- R3
```

R1 runs:

```text
R1(config)# router bgp 65010
R1(config-router)# neighbor 10.0.12.2 remote-as 65100
R1(config-router)# neighbor 10.0.13.3 remote-as 65200
```

The customers expect their provider to use AS 65010.

### Stage 1: Migrate the Provider to AS 65001

The provider migrates R1's BGP process to AS 65001.

R1 is configured to retain AS 65010 as an alternative ASN for both customers.

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65100
R1(config-router)# neighbor 10.0.12.2 local-as 65010 no-prepend replace-as dual-as
R1(config-router)# neighbor 10.0.13.3 remote-as 65200
R1(config-router)# neighbor 10.0.13.3 local-as 65010 no-prepend replace-as dual-as
```

The customers can retain their existing BGP configurations.

Customer A:

```text
R2(config)# router bgp 65100
R2(config-router)# neighbor 10.0.12.1 remote-as 65010
```

Customer B:

```text
R3(config)# router bgp 65200
R3(config-router)# neighbor 10.0.13.1 remote-as 65010
```

R1 presents AS 65010 to both customers even though its real ASN is now 65001.

The `no-prepend replace-as` options prevent the extra AS_PATH modifications normally introduced by basic `local-as`.

This does not mean the provider's internal ASN migration itself is disruption-free. Moving the BGP process to the new ASN must be coordinated with its internal peers and routing design.

### Stage 2: Migrate Customer A

Customer A can later change its configuration:

```text
R2(config)# router bgp 65100
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
```

Because R1 supports `dual-as`, it can establish the new session using AS 65001.

Customer B continues using AS 65010.

### Stage 3: Migrate Customer B

Customer B is updated:

```text
R3(config)# router bgp 65200
R3(config-router)# neighbor 10.0.13.1 remote-as 65001
```

Both customers now use the provider's real ASN.

### Stage 4: Remove Local-AS

After all neighbors have migrated, the alternative ASN configuration can be removed:

```text
R1(config)# router bgp 65001
R1(config-router)# no neighbor 10.0.12.2 local-as
R1(config-router)# no neighbor 10.0.13.3 local-as
```

The provider now uses AS 65001 for its BGP sessions without requiring the local-AS feature.

**Cisco recommends treating Local-AS as a migration mechanism rather than retaining it indefinitely**, because modifying AS_PATH behavior can increase the risk of routing loops.

---

## Local-AS Restrictions and Considerations

Several restrictions are important.

### Alternative ASN Restrictions

For ordinary EBGP local-AS migration, the alternative ASN must be different from:

- The real ASN configured under `router bgp`.
- The ASN of the remote EBGP neighbor.

For example, the following is a valid combination:

```text
router bgp 65001
 neighbor 10.0.12.2 remote-as 65002
 neighbor 10.0.12.2 local-as 65010
```

The alternative ASN is different from both the local and remote ASNs.

Cisco IOS XE also provides a separate **iBGP Local-AS** feature, which supports a special configuration where `remote-as` and `local-as` specify the same ASN. This is used for iBGP migration and route-reflector scenarios rather than ordinary EBGP local-AS operation.

### Confederations

The standard EBGP Local-AS migration feature is intended for true EBGP sessions.

It is not supported for peering between different member ASNs within the same BGP confederation.

A router belonging to a confederation can still use Local-AS when peering with an external neighbor, subject to the platform's configuration requirements.

### Peer Groups and Templates

Local-AS can be configured for individual neighbors or through supported peer groups and peer templates.

However, when Local-AS is inherited from a peer group, Cisco documents restrictions on customizing the configuration independently for individual members.

### Four-Byte ASNs

Modern Cisco IOS XE supports 4-byte ASNs with `local-as`.

For example:

```text
R1(config)# router bgp 100000
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# neighbor 10.0.12.2 local-as 200000 no-prepend replace-as
```

The alternative ASN does not need to be a 2-byte ASN.

Normal 4-byte ASN capability negotiation and compatibility rules still apply.

See [4-Byte AS Numbers](../../peering/four-byte-asn.md).

---

## Local-AS and Loop Prevention

BGP uses the AS_PATH attribute to prevent inter-AS routing loops.

By default, a BGP router rejects received paths containing its local ASN.

When Local-AS is configured, the router also needs to account for its alternative ASN during AS_PATH loop detection.

For example, R1 has:

```text
Real ASN:        65001
Alternative ASN: 65010
```

If R1 receives a route containing either its real ASN or its configured alternative ASN, that route may be rejected by AS_PATH loop prevention.

The Local-AS feature does not eliminate normal loop prevention.

However, the `no-prepend` and `replace-as` options change the AS_PATH information propagated between routers.

Improperly configured ASN replacement can therefore conceal information that would otherwise help detect routing loops.

This is particularly important when:

- Multiple routers still use the old ASN.
- A network is partially migrated.
- Routes are exchanged between the old and new ASes.
- Multiple customers reuse ASNs.
- Local-AS is combined with `allowas-in` or other AS_PATH manipulation features.

**Important:** Do not use `allowas-in` or disable other BGP loop-prevention mechanisms simply to make an incorrectly configured Local-AS migration work.

The ASN migration design must ensure that routing loops cannot form.

---

## Verification and Troubleshooting

Several commands are useful for verifying Local-AS operation.

### Verify Neighbor Session Details

```text
R1# show ip bgp neighbors 10.0.12.2
```

The detailed neighbor output can identify:

- Local and remote ASNs.
- The configured alternative local ASN.
- The `no-prepend` option.
- The `replace-as` option.
- The `dual-as` option.
- BGP session state.

The exact output varies by IOS XE release.

### Verify the BGP Summary

```text
R1# show ip bgp summary
```

Check whether the neighbor has reached the Established state.

If the session fails after changing Local-AS, verify the ASN expected by the remote neighbor.

### Verify Inbound AS_PATH

```text
R1# show ip bgp 192.0.2.0
```

Check the AS_PATH of the route learned from R2.

For example, with:

```text
neighbor 10.0.12.2 local-as 65010
```

R1 can store:

```text
65010 65002
```

With:

```text
neighbor 10.0.12.2 local-as 65010 no-prepend
```

R1 stores:

```text
65002
```

This is an effective way to verify `no-prepend`.

### Verify Outbound AS_PATH

On R1:

```text
R1# show ip bgp neighbors 10.0.12.2 advertised-routes
```

This identifies routes advertised toward R2.

However, the local router's advertised-routes output may not display every final outbound AS_PATH modification.

To verify `replace-as`, inspect the route on the receiving router:

```text
R2# show ip bgp 203.0.113.0
```

Without `replace-as`, the received AS_PATH can be:

```text
65010 65001 65003
```

With `replace-as`:

```text
65010 65003
```

### Verify the Running Configuration

```text
R1# show running-config | section router bgp
```

Confirm that Local-AS is applied to the intended neighbor.

### BGP Session Resets

Changes to Local-AS configuration can cause the BGP session to reset.

This is because Local-AS affects the ASN used during BGP session establishment.

For example, modifying:

```text
neighbor 10.0.12.2 local-as 65010
```

To:

```text
neighbor 10.0.12.2 local-as 65010 no-prepend replace-as
```

can result in a neighbor reset due to a Local-AS configuration change.

A soft BGP clear cannot renegotiate the ASN used in an established BGP session.

Plan Local-AS changes accordingly.

---

## Common Troubleshooting Scenarios

### BGP Session Fails with Wrong AS

**Problem:** R2 expects AS 65010, but R1 presents AS 65001.

**Solution:** Configure the alternative ASN on R1:

```text
R1(config-router)# neighbor 10.0.12.2 local-as 65010
```

Alternatively, migrate R2 to expect the real ASN.

### AS_PATH Contains an Unexpected ASN

**Problem:** R1 learns a route from R2 with an unexpected AS 65010 added to its AS_PATH.

**Cause:** The default inbound Local-AS behavior.

**Solution:**

```text
R1(config-router)# neighbor 10.0.12.2 local-as 65010 no-prepend
```

### Neighbor Still Sees the Real ASN

**Problem:** R2 receives:

```text
65010 65001 65003
```

But should receive:

```text
65010 65003
```

**Solution:**

```text
R1(config-router)# neighbor 10.0.12.2 local-as 65010 no-prepend replace-as
```

### Neighbor Cannot Migrate to the Real ASN

**Problem:** R2 changes its expected remote ASN from 65010 to 65001, but the BGP session fails.

**Possible cause:** R1 is still configured to present only AS 65010.

**Solution:**

```text
R1(config-router)# neighbor 10.0.12.2 local-as 65010 no-prepend replace-as dual-as
```

This allows R1 to support both the old and new peering ASNs during migration.

---

## Key Points

- `neighbor local-as` allows a router to use an alternative ASN for a particular BGP neighbor.
- It does not change the ASN configured under `router bgp`.
- Local-AS is primarily used for **AS migrations**.
- By default, the alternative ASN is inserted into the AS_PATH of routes received from the local-AS neighbor.
- By default, both the alternative and real ASNs are prepended in outbound updates to the local-AS neighbor.
- `no-prepend` prevents the alternative ASN from being inserted into received AS_PATHs.
- `no-prepend` does not remove the real ASN from outbound updates.
- `replace-as` prevents the real ASN from being prepended to advertisements toward the local-AS neighbor.
- On IOS XE, `replace-as` is configured together with `no-prepend`.
- `dual-as` allows the BGP session to use either the real or alternative ASN.
- `dual-as` does not create two simultaneous BGP sessions with the same neighbor.
- Local-AS changes can reset the BGP session.
- Local-AS is neighbor-specific and can coexist with ordinary EBGP sessions using the real ASN.
- Local-AS supports 4-byte ASNs on modern IOS XE.
- ASN manipulation can affect loop prevention and must be planned carefully.

---

## References

- [Cisco: Configure the BGP Local-AS Feature](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13761-39.html)
- [Cisco IOS XE: BGP Support for Dual AS Configuration for Network AS Migrations](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-dual-as.html)
- [Cisco: BGP Command Reference, neighbor local-as](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [Cisco IOS XE: BGP Support for iBGP Local-AS](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-local-as-policy.html)
- [RFC 7705: Autonomous System Migration Mechanisms and Their Effects on the BGP AS_PATH Attribute](https://www.rfc-editor.org/rfc/rfc7705.html)
