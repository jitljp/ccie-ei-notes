# BGP Route Advertisement and Network Statements

BGP exchanges routing information between peers using **UPDATE messages**.

A BGP router can advertise routes that it originates locally or learns from other BGP peers.

There are several ways to introduce routes into BGP:

| Method | Purpose |
|---|---|
| `network` | Originate an existing route from the local IP routing table |
| `redistribute` | Import routes from another routing source into BGP |
| `aggregate-address` | Generate an aggregate from existing BGP routes |
| `neighbor default-originate` | Advertise a default route to a specific neighbor |

Once a route enters the BGP table, BGP determines whether it is eligible to be advertised to each neighbor.

---

## BGP Network Statements

The `network` command is one of the most common methods of originating routes into BGP.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# network 192.168.1.0 mask 255.255.255.0
```

This instructs BGP to originate `192.168.1.0/24`.

However, **the prefix must already exist in the local IP routing table as an exact match**.

The `network` command does not create a route in the IP routing table.

### Exact-Match Requirement

Suppose R1's routing table contains:

```text
C 192.168.1.0/24 is directly connected, Ethernet0/1
```

The following command successfully originates the prefix into BGP:

```text
R1(config-router-af)# network 192.168.1.0 mask 255.255.255.0
```

However, consider this command:

```text
R1(config-router-af)# network 192.168.0.0 mask 255.255.0.0
```

R1 has `192.168.1.0/24` in its routing table, but it does not have `192.168.0.0/16`.

Therefore, BGP does not originate the `/16` route.

```text
IP Routing Table:
192.168.1.0/24

BGP network statement:
192.168.0.0/16

Result:
No exact match
Route is not originated
```

The same applies in the opposite direction.

If the routing table contains `192.168.0.0/16`, a `network` statement for `192.168.1.0/24` does not originate the more-specific prefix unless that exact `/24` route also exists.

### Sources of the Matching Route

The matching route can come from several sources.

| Routing Source | Can Satisfy `network`? |
|---|---|
| Connected route | Yes |
| Static route | Yes |
| OSPF route | Yes |
| EIGRP route | Yes |
| IS-IS route | Yes |

The important requirement is that the exact prefix and prefix length exist in the local IP routing table.

**The `network` command does not require the route to be connected or learned through an IGP.**

---

## BGP Network vs. IGP Network Statements

The `network` command has a different purpose in BGP than in protocols such as OSPF and EIGRP.

### OSPF

```text
R1(config)# router ospf 1
R1(config-router)# network 10.0.12.0 0.0.0.255 area 0
```

The OSPF `network` command identifies interfaces on which OSPF should operate.

It enables OSPF on matching interfaces and associates them with an area.

### BGP

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# network 10.0.12.0 mask 255.255.255.0
```

The BGP `network` command originates the specified prefix into the BGP table if a matching route exists.

It does not enable BGP on the interface.

| Feature | OSPF `network` | BGP `network` |
|---|---|---|
| Activates the protocol on interfaces | Yes | No |
| Used for neighbor discovery | Indirectly, by enabling OSPF | No |
| Originates a matching prefix into the protocol | Connected networks of participating interfaces | Yes, based on an exact routing-table match |
| Requires an exact prefix in the IP routing table | No | Yes |

BGP neighbors are configured separately using the `neighbor` command.

---

## Network Statements and Subnet Masks

For IPv4, the BGP `network` command supports the following syntax:

```text
network <network-address> [mask <subnet-mask>]
```

For example:

```text
R1(config-router-af)# network 10.1.1.0 mask 255.255.255.0
```

This originates `10.1.1.0/24` if the route is present in the IP routing table.

### Omitting the Mask

If the `mask` keyword is omitted, the command uses the natural classful mask.

| Network Statement | Prefix |
|---|---|
| `network 10.0.0.0` | 10.0.0.0/8 |
| `network 172.16.0.0` | 172.16.0.0/16 |
| `network 192.168.1.0` | 192.168.1.0/24 |

For example:

```text
R1(config-router-af)# network 10.1.1.0
```

This does not originate `10.1.1.0/24`.

Without the mask, the classful network is `10.0.0.0/8`.

Use an explicit mask to avoid ambiguity:

```text
R1(config-router-af)# network 10.1.1.0 mask 255.255.255.0
```

---

## Basic BGP Route Origination

Consider the following topology:

```text
              AS 65001                   AS 65002

                R1 ----------------------- R2
              E0/0                       E0/0
           10.0.12.1/30               10.0.12.2/30

           Loopback0:
           192.168.1.1/24
```

R1 will advertise `192.168.1.0/24` to R2 through EBGP.

### R1 Configuration

Configure the interfaces:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip address 10.0.12.1 255.255.255.252
R1(config-if)# no shutdown

R1(config)# interface Loopback0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
```

R1 now has a connected route to `192.168.1.0/24`.

Verify:

```text
R1# show ip route 192.168.1.0
```

Example:

```text
Routing entry for 192.168.1.0/24
  Known via "connected", distance 0, metric 0
  directly connected, via Loopback0
```

Configure BGP:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 activate
R1(config-router-af)# network 192.168.1.0 mask 255.255.255.0
```

### R2 Configuration

```text
R2(config)# interface Ethernet0/0
R2(config-if)# ip address 10.0.12.2 255.255.255.252
R2(config-if)# no shutdown

R2(config)# router bgp 65002
R2(config-router)# bgp router-id 2.2.2.2
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.12.1 activate
```

### Verification on R1

Verify the BGP table:

```text
R1# show bgp ipv4 unicast
```

The locally originated prefix should appear with:

```text
*> 192.168.1.0/24  0.0.0.0  ...  32768 i
```

The relevant fields are:

| Field | Meaning |
|---|---|
| `*` | Valid BGP path |
| `>` | Best BGP path |
| `0.0.0.0` | Locally originated next hop |
| `32768` | Default Cisco weight for a locally originated route |
| `i` | IGP ORIGIN attribute |

**The ORIGIN code `i` does not mean the route was learned through an IGP.**

It indicates that the BGP ORIGIN attribute is IGP, which is the normal value for routes originated using the `network` command.

Verify the advertisement:

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 advertised-routes
```

### Verification on R2

Verify that R2 learned the prefix:

```text
R2# show bgp ipv4 unicast 192.168.1.0/24
```

The route should have:

```text
AS_PATH: 65001
NEXT_HOP: 10.0.12.1
ORIGIN: IGP
```

Verify installation in R2's routing table:

```text
R2# show ip route 192.168.1.0
```

If the BGP path is selected for installation, it appears as an EBGP route with an administrative distance of 20.

---

## Network Statements and Route Withdrawal

The `network` command depends on the existence of an exact matching route in the local IP routing table.

Suppose R1 originates:

```text
R1(config-router-af)# network 192.168.1.0 mask 255.255.255.0
```

The route initially exists because Loopback0 is operational.

```text
IP Routing Table
       |
       | 192.168.1.0/24 exists
       v
BGP originates 192.168.1.0/24
       |
       v
BGP advertises the prefix
```

Now suppose Loopback0 is shut down.

```text
R1(config)# interface Loopback0
R1(config-if)# shutdown
```

If the exact route disappears from the IP routing table, BGP can no longer originate it using the `network` statement.

```text
192.168.1.0/24 removed from RIB
              |
              v
BGP withdraws locally originated path
              |
              v
BGP sends withdrawal to peers
```

If another routing source continues supplying the exact prefix in the IP routing table, the `network` statement can continue originating it.

The dependency is on the **matching route**, not on a particular interface or routing protocol.

---

## Originating Routes Using Static Routes

A common method of originating BGP routes is to install a static route and reference it using a `network` statement.

For example, originate `10.10.0.0/16`:

```text
R1(config)# ip route 10.10.0.0 255.255.0.0 Null0

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# network 10.10.0.0 mask 255.255.0.0
```

The static route creates an exact match in the IP routing table.

BGP can then originate the prefix.

### Why Use Null0?

A static route to `Null0` provides a routing-table entry for the aggregate prefix.

More-specific routes can still forward traffic normally because of longest-prefix matching.

For example:

```text
10.10.0.0/16  -> Null0
10.10.1.0/24  -> Ethernet0/1
10.10.2.0/24  -> Ethernet0/2
```

Traffic destined for `10.10.1.0/24` follows the more-specific route.

Traffic matching `10.10.0.0/16` without a more-specific route is discarded.

This can prevent forwarding loops when advertising aggregate prefixes, but traffic to an unreachable part of the aggregate will be dropped.

---

## Redistributing Routes into BGP

Another way to introduce routes into BGP is through **redistribution**.

Unlike the `network` command, redistribution can import multiple routes from a routing source without an individual BGP network statement for each prefix.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# redistribute connected
```

This redistributes eligible connected routes into BGP.

### Redistribution vs. Network Statements

| Characteristic | `network` | `redistribute` |
|---|---|---|
| Route source | Exact match in local RIB | Specified routing source |
| Prefix selection | Explicit individual prefixes | Routes selected from routing source |
| Default ORIGIN attribute | IGP (`i`) | Usually Incomplete (`?`) |
| Route map support | Yes | Yes |
| Typical use | Originate selected prefixes | Import routes from another protocol |

For example, a connected route can be introduced using either:

```text
R1(config-router-af)# network 192.168.1.0 mask 255.255.255.0
```

or:

```text
R1(config-router-af)# redistribute connected
```

However, `redistribute connected` may introduce many connected networks, including transit networks that should not be advertised.

It is generally preferable to control redistributed routes using route maps.

### Selective Redistribution

Suppose only `192.168.1.0/24` should be redistributed.

Create a prefix list:

```text
R1(config)# ip prefix-list BGP-CONNECTED permit 192.168.1.0/24
```

Create a route map:

```text
R1(config)# route-map BGP-CONNECTED permit 10
R1(config-route-map)# match ip address prefix-list BGP-CONNECTED
```

Apply it to redistribution:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# redistribute connected route-map BGP-CONNECTED
```

Only matching connected routes are redistributed.

---

## BGP Network Statements with Route Maps

The `network` command can also reference a route map.

For example:

```text
R1(config)# route-map ORIGINATE permit 10
R1(config-route-map)# set metric 100

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# network 192.168.1.0 mask 255.255.255.0 route-map ORIGINATE
```

This originates `192.168.1.0/24` and applies the route map to the locally originated BGP path.

In this example, the route map sets the **MED** attribute to 100.

A route map attached to `network` can also be used to control other attributes of locally originated paths.

The exact-match routing-table requirement still applies.

---

## BGP Route Aggregation

BGP can generate aggregate routes using the `aggregate-address` command.

Unlike `network`, aggregation depends on more-specific routes already present in the **BGP table**.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# aggregate-address 10.0.0.0 255.255.252.0
```

This creates an aggregate for `10.0.0.0/22` when at least one contributing more-specific route exists in the BGP table.

For example:

```text
BGP Table:

10.0.0.0/24
10.0.1.0/24
10.0.2.0/24
10.0.3.0/24
      |
      v
10.0.0.0/22
```

By default, BGP can advertise both the aggregate and its component routes.

To suppress the more-specific advertisements:

```text
R1(config-router-af)# aggregate-address 10.0.0.0 255.255.252.0 summary-only
```

The important distinction is:

| Command | Requirement |
|---|---|
| `network` | Exact matching prefix in the IP routing table |
| `aggregate-address` | At least one contributing more-specific prefix in the BGP table |

Unlike the `network` command, `aggregate-address` does not require an exact matching route in the IP routing table.

Aggregation and AS_SET behavior are covered in detail in section 1.5.e.

---

## Advertising a Default Route

BGP provides multiple methods of advertising a default route.

### Method 1: Network Statement

If a default route exists in the IP routing table, it can be originated using:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# network 0.0.0.0 mask 0.0.0.0
```

This requires an exact matching `0.0.0.0/0` route in the local routing table.

For example:

```text
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.12.2
```

The default route is originated into BGP and can be advertised to eligible neighbors, subject to normal BGP advertisement rules and policies.

### Method 2: Neighbor Default-Originate

A router can advertise a default route to a specific BGP neighbor using:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 default-originate
```

**This command does not require a default route in the local routing table.**

The default route is generated specifically for the configured neighbor.

This differs from the `network` command.

| Command | Requires Local Default Route? | Scope |
|---|---|---|
| `network 0.0.0.0 mask 0.0.0.0` | Yes | Normal BGP origination and advertisement |
| `neighbor ... default-originate` | No | Specific neighbor |

The `default-originate` command also supports a route map for conditional default-route advertisement:

```text
R1(config-router-af)# neighbor 10.0.12.2 default-originate route-map CHECK
```

This allows the advertisement to depend on conditions defined in the route map.

### Method 3: Redistribution

A default route can also be redistributed into BGP.

For IPv4, BGP uses `default-information originate` together with the appropriate redistribution command to allow the default route to be redistributed.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# default-information originate
R1(config-router-af)# redistribute static
```

This enables redistribution of an eligible static default route, assuming it exists in the routing table.

A route map can be used with redistribution to restrict which static routes are imported.

---

## BGP Route Advertisement Rules

A BGP route being present in the BGP table does not automatically mean it will be advertised to every neighbor.

BGP must consider:

1. Whether the route is valid and selected for advertisement.
2. Whether the route is permitted by BGP advertisement rules.
3. Whether any outbound filtering or routing policy suppresses the route.
4. Whether the neighbor is activated for the relevant address family.
5. Whether the BGP session is Established.

### Best-Path Advertisement

By default, BGP normally advertises only its **best path** for each prefix.

For example:

```text
BGP Table:

Prefix: 192.168.1.0/24

Path 1: via R2  <- Best
Path 2: via R3
Path 3: via R4
```

BGP normally advertises Path 1.

The other paths remain in the BGP table but are not advertised.

Features such as BGP ADD-PATH can modify this behavior by allowing multiple paths for the same prefix to be advertised.

### IBGP Advertisement Rules

BGP also applies rules based on how a route was learned and the type of neighbor receiving the advertisement.

| Route Learned From | Advertised to EBGP? | Advertised to IBGP? |
|---|---|---|
| Locally originated | Yes | Yes |
| EBGP neighbor | Yes | Yes |
| IBGP neighbor | Yes | No, by default |

All advertisements remain subject to best-path selection, routing policies, and other BGP restrictions.

Route reflectors provide an exception to the normal IBGP advertisement restriction.

### NEXT_HOP Handling

When advertising a route through EBGP, the router normally changes the NEXT_HOP attribute to identify itself.

When advertising a route through IBGP, the router normally preserves the existing NEXT_HOP attribute.

For example:

```text
R1 (AS 65001) ---EBGP--- R2 (AS 65002) ---IBGP--- R3
```

If R1 advertises a locally originated prefix to R2:

```text
R2 receives:
NEXT_HOP = R1
```

When R2 advertises the route to R3 through IBGP:

```text
R3 receives:
NEXT_HOP = R1
```

R3 must have a valid route to R1's next-hop address.

Otherwise, the BGP path cannot be selected as a valid best path.

The `next-hop-self` command can change this behavior.

---

## Outbound Route Filtering

BGP supports policies that control which routes are advertised to specific neighbors.

For example, R1 originates:

```text
192.168.1.0/24
192.168.2.0/24
```

But it should advertise only `192.168.1.0/24` to R2.

Create a prefix list:

```text
R1(config)# ip prefix-list ADVERTISE permit 192.168.1.0/24
```

Apply it outbound:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 prefix-list ADVERTISE out
```

R2 receives only the permitted prefix.

```text
BGP Table on R1:
192.168.1.0/24
192.168.2.0/24

           |
           | Outbound prefix list
           v

Advertised to R2:
192.168.1.0/24
```

The denied route remains in R1's BGP table.

The prefix list only controls advertisement toward the specified neighbor.

---

## BGP Advertisements and RIB Failures

An important distinction is that **the best BGP path does not have to be installed in the IP routing table to be advertised**.

Suppose R1 learns `192.168.1.0/24` through both OSPF and IBGP.

```text
OSPF:
Administrative distance = 110

IBGP:
Administrative distance = 200
```

The OSPF route is installed in the IP routing table because of its lower administrative distance.

However, the IBGP path may still be the best path in the BGP table.

The BGP route is marked as a **RIB failure**.

By default, Cisco IOS XE can continue advertising that best BGP path to eligible BGP neighbors, even though it was not installed in the IP routing table.

### Suppressing Inactive Routes

The following command enables suppression of inactive BGP route advertisements:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp suppress-inactive
```

For RIB failures caused by administrative distance, suppression depends on whether the BGP next hop and the installed route's forwarding next hop resolve to the same forwarding adjacency.

Verify RIB failures with:

```text
R1# show ip bgp rib-failure
```

This feature is useful when the network should avoid advertising BGP paths that are not being used for local forwarding.

---

## BGP UPDATE Generation

BGP sends UPDATE messages when routing information changes.

Examples include:

- A new prefix is originated into BGP.
- The best path for a prefix changes.
- A previously advertised prefix is withdrawn.
- An outbound routing policy changes the advertised information.
- A neighbor requests route refresh.

BGP does not periodically resend its complete routing table.

After initial session establishment, route advertisements are primarily event-driven.

### Minimum Route Advertisement Interval

BGP can limit how frequently successive UPDATEs are advertised to a neighbor.

Cisco IOS XE supports:

```text
R1(config-router-af)# neighbor 10.0.12.2 advertisement-interval 10
```

This configures a minimum advertisement interval of 10 seconds for the neighbor.

It can reduce excessive UPDATE traffic during rapid routing changes.

The advertisement interval is not a periodic update timer. It does not cause the router to advertise routes every 10 seconds.

---

## IPv6 Route Origination

BGP can also originate IPv6 routes using the IPv6 unicast address family.

For example:

```text
R1(config)# interface Loopback0
R1(config-if)# ipv6 address 2001:DB8:1::1/64

R1(config)# router bgp 65001
R1(config-router)# neighbor 2001:DB8:12::2 remote-as 65002
R1(config-router)# address-family ipv6 unicast
R1(config-router-af)# neighbor 2001:DB8:12::2 activate
R1(config-router-af)# network 2001:DB8:1::/64
```

This originates `2001:DB8:1::/64` if an exact matching route exists in the IPv6 routing table.

Verify the route:

```text
R1# show ipv6 route 2001:DB8:1::/64
R1# show bgp ipv6 unicast 2001:DB8:1::/64
```

The same general principles apply:

- The specified prefix must exist in the corresponding IP routing table.
- BGP originates the prefix into the appropriate address family.
- The route is subject to BGP path selection and advertisement policies.
- Neighbor relationships must support and activate the relevant address family.

---

## Verification Commands

The following commands are useful when troubleshooting BGP route origination and advertisement.

| Command | Purpose |
|---|---|
| `show ip route <prefix>` | Verify that the prefix exists in the IP routing table |
| `show bgp ipv4 unicast` | Display all IPv4 unicast BGP paths |
| `show bgp ipv4 unicast <prefix>` | Display detailed BGP information for a prefix |
| `show bgp ipv4 unicast summary` | Verify neighbor state and received prefix counts |
| `show bgp ipv4 unicast neighbors <IP> advertised-routes` | Display routes advertised to a neighbor |
| `show bgp ipv4 unicast neighbors <IP> routes` | Display accepted routes from a neighbor |
| `show ip bgp rib-failure` | Display BGP paths that were not installed in the IP routing table |
| `show running-config | section router bgp` | Verify BGP configuration |

### Troubleshooting a Missing Advertisement

If a route is not being advertised, use the following process.

**Step 1: Check the IP routing table**

```text
R1# show ip route 192.168.1.0
```

If using a `network` statement, verify that the exact route exists.

**Step 2: Check the BGP table**

```text
R1# show bgp ipv4 unicast 192.168.1.0/24
```

If the prefix is missing, inspect the `network` statement, redistribution, or aggregation configuration.

**Step 3: Check the best path**

Verify that BGP has a valid path selected for advertisement.

**Step 4: Check the neighbor session**

```text
R1# show bgp ipv4 unicast summary
```

Verify that the neighbor is Established.

**Step 5: Check advertised routes**

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 advertised-routes
```

If the prefix is not advertised, inspect:

- Outbound prefix lists, filter lists, and route maps
- IBGP advertisement restrictions
- Route suppression and aggregation
- Community-based advertisement restrictions
- Address-family activation
- Other BGP advertisement eligibility rules

**Step 6: Check the receiving router**

```text
R2# show bgp ipv4 unicast 192.168.1.0/24
R2# show ip route 192.168.1.0
```

A route may be advertised but rejected by inbound policy, considered invalid because of next-hop reachability, or not installed in the routing table.

---

## Key Points

- BGP uses UPDATE messages to advertise and withdraw routes.
- The `network` command originates an existing route into BGP.
- A BGP `network` statement requires an exact prefix and prefix-length match in the IP routing table.
- The matching route can be connected, static, or learned through an IGP.
- BGP `network` statements do not enable BGP on interfaces.
- Omitting the IPv4 `mask` keyword uses the natural classful mask.
- Routes originated using `network` normally have an ORIGIN attribute of IGP (`i`).
- `redistribute` imports eligible routes from another routing source into BGP.
- `aggregate-address` requires at least one contributing more-specific route in the BGP table.
- `network 0.0.0.0 mask 0.0.0.0` requires a default route in the local IP routing table.
- `neighbor default-originate` does not require a local default route.
- BGP normally advertises only its best path for each prefix.
- IBGP-learned routes are not advertised to other IBGP peers by default.
- Outbound policies can control which prefixes are advertised to each neighbor.
- BGP can advertise a best path even when that path was not installed in the IP routing table.
- BGP advertisements are primarily event-driven, not periodic complete-table updates.
