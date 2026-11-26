# Multi-ISP NAT and Failover

An enterprise router connected to multiple ISPs typically needs to translate inside source addresses according to the ISP through which traffic is forwarded. Each ISP may provide its own public IPv4 address or address block.

**Multi-ISP NAT** combines routing with policy-based NAT to select an appropriate inside global address for each forwarding path.

In an active/standby design, one ISP is preferred and the other is used when the preferred path fails. In an active/active design, both ISPs can carry traffic simultaneously.

For either design, two independent functions are required:

- **Routing:** Select an available forwarding path toward the destination.
- **NAT:** Select an inside global address appropriate for that path.

NAT does not install a backup default route, detect an upstream outage, or guarantee that existing sessions survive a change of ISP.

---

## Egress-Based PAT

Consider a router with two ISP-facing interfaces:

| Outgoing interface | ISP | Interface address | Required inside global address |
|---|---|---|---|
| E0/1 | ISP1 | `7.7.7.2` | `7.7.7.2` |
| E0/2 | ISP2 | `8.8.8.2` | `8.8.8.2` |

The same inside source network, `10.1.1.0/24`, must be translated differently depending on the outgoing interface.

The following configuration does not adequately distinguish between the paths:

```text
access-list 10 permit 10.1.1.0 0.0.0.255
!
ip nat inside source list 10 interface Ethernet0/1 overload
ip nat inside source list 10 interface Ethernet0/2 overload
Cannot change mapping's interface name, or the mapping already exists; remove mapping first to make change.
```

Notice that the second command is rejected.

Instead, configure a separate route map for each ISP:

```text
route-map NAT-ISP1 permit 10
 match ip address 10
 match interface Ethernet0/1
!
route-map NAT-ISP2 permit 10
 match ip address 10
 match interface Ethernet0/2
!
ip nat inside source route-map NAT-ISP1 interface Ethernet0/1 overload
ip nat inside source route-map NAT-ISP2 interface Ethernet0/2 overload
```

For a packet requiring a new translation, the selected egress interface determines which NAT policy matches.

This configuration is useful regardless of whether the forwarding decision is based on static routing, dynamic routing, ECMP, or PBR. The NAT rules do not need to be modified simply because the active default route changes.

See [Destination-Based NAT and Traffic Engineering](traffic-engineering.md) for the relationship between PBR and egress-based NAT.

---

## Lab Topology

Continue using the dual-ISP topology from the previous page.

```text
                             ISP1
                       E0/1       E0/0
                     7.7.7.1     192.0.2.1
                        /            \
                       /              \
             E0/1: 7.7.7.2       E0/1: 192.0.2.2
                       /              \
LH5 ---- LAN ---- Local_Router       Remote_Router ---- LAN ---- RH7
LH6               E0/0: 10.1.1.1      E0/0: 3.3.3.1            RH8
                       \              /
             E0/2: 8.8.8.2       E0/2: 192.0.2.6
                       \              /
                        \            /
                     8.8.8.1     192.0.2.5
                       E0/1       E0/0
                             ISP2
```

| Device | Interface | IPv4 address | NAT role |
|---|---|---|---|
| Local_Router | E0/0 | `10.1.1.1/24` | Inside |
| Local_Router | E0/1 | `7.7.7.2/24` | Outside |
| Local_Router | E0/2 | `8.8.8.2/24` | Outside |
| ISP1 | E0/1 | `7.7.7.1/24` | None |
| ISP1 | E0/0 | `192.0.2.1/30` | None |
| ISP2 | E0/1 | `8.8.8.1/24` | None |
| ISP2 | E0/0 | `192.0.2.5/30` | None |
| Remote_Router | E0/1 | `192.0.2.2/30` | None |
| Remote_Router | E0/2 | `192.0.2.6/30` | None |
| Remote_Router | E0/0 | `3.3.3.1/24` | None |

| Host | IPv4 address | Default gateway |
|---|---|---|
| LH5 | `10.1.1.5/24` | `10.1.1.1` |
| LH6 | `10.1.1.6/24` | `10.1.1.1` |
| RH7 | `3.3.3.7/24` | `3.3.3.1` |
| RH8 | `3.3.3.8/24` | `3.3.3.1` |

Configure the interface addresses and host default gateways. Remove the previous exercises' PBR policies, destination-specific static routes, and conflicting NAT rules. Stop active traffic and clear dynamic translations before removing in-use NAT rules.

### Outside Routing

```text
ISP1(config)# ip route 3.3.3.0 255.255.255.0 192.0.2.2
ISP2(config)# ip route 3.3.3.0 255.255.255.0 192.0.2.6
!
Remote_Router(config)# ip route 7.7.7.0 255.255.255.0 192.0.2.1
Remote_Router(config)# ip route 8.8.8.0 255.255.255.0 192.0.2.5
```

Remote_Router sends replies destined for `7.7.7.2` through ISP1 and replies destined for `8.8.8.2` through ISP2. No outside router needs a route to the inside local network for correctly translated traffic.

### NAT Configuration

The following NAT configuration remains in place throughout the exercises:

```text
Local_Router(config)# interface Ethernet0/0
Local_Router(config-if)# ip nat inside
Local_Router(config-if)# exit
!
Local_Router(config)# interface Ethernet0/1
Local_Router(config-if)# ip nat outside
Local_Router(config-if)# exit
!
Local_Router(config)# interface Ethernet0/2
Local_Router(config-if)# ip nat outside
Local_Router(config-if)# exit
!
Local_Router(config)# ip access-list standard INSIDE-HOSTS
Local_Router(config-std-nacl)# permit 10.1.1.0 0.0.0.255
Local_Router(config-std-nacl)# exit
!
Local_Router(config)# route-map NAT-ISP1 permit 10
Local_Router(config-route-map)# match ip address INSIDE-HOSTS
Local_Router(config-route-map)# match interface Ethernet0/1
Local_Router(config-route-map)# exit
!
Local_Router(config)# route-map NAT-ISP2 permit 10
Local_Router(config-route-map)# match ip address INSIDE-HOSTS
Local_Router(config-route-map)# match interface Ethernet0/2
Local_Router(config-route-map)# exit
!
Local_Router(config)# ip nat inside source route-map NAT-ISP1 interface Ethernet0/1 overload
Local_Router(config)# ip nat inside source route-map NAT-ISP2 interface Ethernet0/2 overload
```

---

## Lab 1: Failover with Floating Static Routes

### Requirements

- Prefer ISP1 for all remote destinations.
- Use ISP2 if the local interface toward ISP1 goes down.
- Translate new inside-to-outside flows using the address of the selected ISP-facing interface.

### Configuration

```text
Local_Router(config)# ip route 0.0.0.0 0.0.0.0 Ethernet0/1 7.7.7.1
Local_Router(config)# ip route 0.0.0.0 0.0.0.0 Ethernet0/2 8.8.8.1 10
```

The primary static route has an administrative distance (AD) of 1. The secondary route has an AD of 10, making it a **floating static route**.

Both routes are fully specified: they identify an outgoing interface and a next-hop address. This ties each route to its intended local link rather than allowing recursive resolution through an unintended path.

While the primary route is installed, the secondary route is not used for normal forwarding.

### Verification: Normal Operation

```text
LH5# ping 3.3.3.7

Local_Router# show ip nat translations
Pro  Inside global         Inside local          Outside local         Outside global
icmp 7.7.7.2:6             10.1.1.5:6            3.3.3.7:6             3.3.3.7:6
Total number of translations: 1
```

The inside global address for the new flow is `7.7.7.2`.

### Verification: Local Interface Failure

```text
Local_Router(config)# interface Ethernet0/1
Local_Router(config-if)# shutdown
```

The primary route is removed, and the floating static route through ISP2 becomes active.

```text
LH6# ping 3.3.3.7

Local_Router# show ip nat translations
Pro  Inside global         Inside local          Outside local         Outside global
icmp 7.7.7.2:6             10.1.1.5:6            3.3.3.7:6             3.3.3.7:6
icmp 8.8.8.2:7             10.1.1.5:7            3.3.3.7:7             3.3.3.7:7
```

Using LH6 for the second test avoids reusing LH5's translation state. The new flow should use `8.8.8.2` as its inside global address.

| State | Active default next hop | Egress | Inside global for new flows |
|---|---|---|---|
| Normal | `7.7.7.1` | E0/1 | `7.7.7.2` |
| Primary interface down | `8.8.8.1` | E0/2 | `8.8.8.2` |
| Primary interface restored | `7.7.7.1` | E0/1 | `7.7.7.2` |

Restore E0/1 before continuing:

```text
Local_Router(config)# interface Ethernet0/1
Local_Router(config-if)# no shutdown
```

### Limitation: Upstream Failures

A floating static route alone does not detect every failure along the ISP path.

For example, shut down ISP1's interface toward Remote_Router:

```text
ISP1(config)# interface Ethernet0/0
ISP1(config-if)# shutdown
```

Local_Router's E0/1 remains up, and `7.7.7.1` remains directly connected. The primary default route therefore remains installed even though ISP1 cannot reach RH7 or RH8.

Traffic is forwarded into the failed path rather than switching to ISP2. NAT can still select the correct address for the selected path, but that does not make the path usable.

Restore ISP1's E0/0 before the next exercise.

---

## Lab 2: Failover with IP SLA and Object Tracking

**IP SLA** generates probes to measure reachability or performance. **Enhanced Object Tracking** can use the result of an IP SLA operation to control the installation of a static route.

The relationship between the features is:

```text
                 IP SLA probe result
                         |
                         v
                   Track object
                         |
                         v
           Install / withdraw primary route
                         |
                         v
               Select outgoing interface
                         |
                         v
               Select NAT policy for new flow
```

### Step 1: Choose a Probe Destination

An ICMP Echo operation targeting `7.7.7.1` verifies reachability to the directly connected ISP1 router. It does not verify reachability beyond that router.

To detect the upstream failure demonstrated in Lab 1, configure a loopback on Remote_Router:

```text
Remote_Router(config)# interface Loopback0
Remote_Router(config-if)# ip address 192.0.2.100 255.255.255.255
```

Configure a route to the probe target on ISP1:

```text
ISP1(config)# ip route 192.0.2.100 255.255.255.255 192.0.2.2
```

### Step 2: Pin the Probe to ISP1

The SLA probe must continue to test ISP1 even while ordinary traffic is using ISP2. Otherwise, a successful probe through ISP2 could incorrectly indicate that ISP1 is working.

```text
Local_Router(config)# ip route 192.0.2.100 255.255.255.255 Ethernet0/1 7.7.7.1
Local_Router(config)# ip route 192.0.2.100 255.255.255.255 Null0 254
```

The primary host route directs probe traffic through ISP1 independently of the active default route. Do not associate this route with the SLA track object: the probe needs to keep testing ISP1 while the tracked default is absent.

The floating Null0 host route prevents the probe from using ISP2's default route if E0/1 goes down and the primary host route is removed. Its `/32` prefix is more specific than the default route, regardless of its high AD.

**Specifying a source interface in an IP SLA operation does not by itself force the probe to exit that interface.** The host route determines the intended forwarding path.

### Step 3: Configure and Schedule IP SLA

```text
Local_Router(config)# ip sla 1
Local_Router(config-ip-sla)# icmp-echo 192.0.2.100 source-interface Ethernet0/1
Local_Router(config-ip-sla-echo)# timeout 1000
Local_Router(config-ip-sla-echo)# frequency 3
Local_Router(config-ip-sla-echo)# exit
Local_Router(config)# ip sla schedule 1 life forever start-time now
```

| Command | Purpose |
|---|---|
| `ip sla 1` | Create IP SLA operation 1 |
| `icmp-echo ... source-interface ...` | Send ICMP Echo probes using the specified interface's address as the source |
| `timeout 1000` | Wait up to 1,000 milliseconds for a response |
| `frequency 3` | Start a probe every three seconds |
| `ip sla schedule ...` | Activate the operation immediately and run it indefinitely |

The operation uses `7.7.7.2` as its source address. Remote_Router already has a return route to that address through ISP1.

These probes originate on Local_Router; they are not traffic arriving from the inside LAN and do not require its PAT rules.

The probe verifies connectivity to the remote router's loopback. It does not guarantee reachability to every Internet destination or even to the remote LAN if that LAN fails independently.

### Step 4: Configure Object Tracking

```text
Local_Router(config)# track 1 ip sla 1 reachability
Local_Router(config-track)# exit
```

`reachability` tracks whether the SLA operation can reach its target. An OK or OverThreshold return code is considered reachable; a failed reachability result, such as a timeout, is not.

By contrast, tracking IP SLA `state` treats an OverThreshold result as down. Reachability is appropriate here because the objective is connectivity failover, rather than failover based on a performance threshold.

### Step 5: Track the Primary Default Route

Remove the untracked primary route from Lab 1 and replace it with a tracked route. The floating backup route remains unchanged.

```text
Local_Router(config)# no ip route 0.0.0.0 0.0.0.0 Ethernet0/1 7.7.7.1
Local_Router(config)# ip route 0.0.0.0 0.0.0.0 Ethernet0/1 7.7.7.1 track 1
```

The resulting default route configuration is:

```text
ip route 0.0.0.0 0.0.0.0 Ethernet0/1 7.7.7.1 track 1
ip route 0.0.0.0 0.0.0.0 Ethernet0/2 8.8.8.1 10
```

| Track 1 state | Primary route | Backup route |
|---|---|---|
| Up | Eligible for installation and preferred | Floating |
| Down | Ineligible for installation | Installed if its next hop and interface are usable |

The backup route is not itself SLA-tracked. If ISP2 experiences an upstream outage, this configuration cannot detect it.

### Step 6: Verify the Primary Path

```text
Local_Router# show ip sla configuration 1
Local_Router# show ip sla statistics 1
Local_Router# show track 1
Local_Router# show ip route 192.0.2.100
Local_Router# show ip route 0.0.0.0
```

Verify that the SLA operation returns OK, track 1 is up, and the primary default route is installed. Then generate a new flow from LH5 to RH7 and verify that PAT uses `7.7.7.2`.

### Step 7: Simulate an Upstream Failure

```text
ISP1(config)# interface Ethernet0/0
ISP1(config-if)# shutdown
```

Local_Router's E0/1 remains up. After the SLA operation detects the failure, track 1 goes down, the primary default route is withdrawn, and the floating route through ISP2 is installed.

```text
Local_Router# show ip interface brief
Local_Router# show ip sla statistics 1
Local_Router# show track 1
Local_Router# show ip route 0.0.0.0
Local_Router# show ip route 192.0.2.100
```

The host route toward `192.0.2.100` still points through ISP1 even though the default route now points through ISP2.

Generate a fresh flow from LH6 to RH7 (ensure no matching translation from an earlier exercise remains):

```text
LH6# ping 3.3.3.7
Local_Router# show ip nat translations
```

The new translation should use `8.8.8.2`. The NAT configuration has not changed; the routing decision has changed.

### Step 8: Verify Failback

```text
ISP1(config)# interface Ethernet0/0
ISP1(config-if)# no shutdown
```

The SLA operation can again reach the remote loopback. Track 1 goes up and the primary default route is reinstalled, replacing the floating route.

Generate a new flow and verify that it uses `7.7.7.2`.

| Stage | E0/1 | SLA result | Track 1 | Default next hop | Inside global for new flows |
|---|---|---|---|---|---|
| Normal | Up | OK | Up | `7.7.7.1` | `7.7.7.2` |
| ISP1 upstream failure | Up | Failure | Down | `8.8.8.1` | `8.8.8.2` |
| ISP1 recovery | Up | OK | Up | `7.7.7.1` | `7.7.7.2` |

### Optional: Delay Track State Changes

```text
Local_Router(config)# track 1 ip sla 1 reachability
Local_Router(config-track)# delay down 5 up 15
```

A down transition is delayed by five seconds, and an up transition is delayed by 15 seconds. The delay helps prevent frequent route changes when reachability fluctuates. Delaying failback is particularly useful when a recovered path is unstable.

The total failover time depends on when the next probe is sent, its timeout, the tracking delay, and forwarding convergence. It is not simply equal to the SLA frequency.

---

## Existing NAT Translations During Failover

The previous exercises verify **new flows**. This is different from preserving an existing connection across an ISP change.

Suppose LH5 establishes a TCP session to RH7 through ISP1:

| Endpoint | Address and port |
|---|---|
| Inside local | `10.1.1.5:51000` |
| Inside global | `7.7.7.2:51000` |
| Outside endpoint | `3.3.3.7:23` |

The port numbers are illustrative. The remote TCP endpoint identifies the connection using `7.7.7.2:51000`, not `10.1.1.5:51000`.

If the default route changes to ISP2, there are two separate problems to consider:

- **Existing translation state:** An applicable translation can retain the ISP1 inside global address. A routing change does not guarantee that every affected NAT entry is removed or recreated. Interface-down events and upstream failures can have different effects on the translation table.
- **Transport-layer state:** Even if PAT selects `8.8.8.2`, the remote host sees a different TCP endpoint. The original TCP connection cannot continue normally using the new source IP address.

If the old translation is retained, packets might exit through ISP2 with an ISP1 source address. ISP2 may reject them due to source address validation, or replies may follow the route toward the failed ISP1 path.

If the old translation is removed and replaced with one using ISP2's address, the original TCP connection still needs to be reestablished.

**Successful routing failover and correct NAT for new flows do not imply stateful session preservation.**

The same considerations apply during failback. A session established via ISP2 may be affected when routing returns to ISP1.

### Lab 3: Observe Existing Session Behavior

Use the tracked configuration from Lab 2. Enable a TCP listener on RH7, such as an SSH or Telnet service, and perform the following test:

1. With ISP1 active, establish a TCP session from LH5 to RH7.
2. Record the translation with `show ip nat translations verbose`.
3. Shut down ISP1's upstream interface, leaving Local_Router's E0/1 up.
4. Wait until track 1 is down and the backup default route is installed.
5. Attempt to use the existing session. Inspect the translation table and capture packets on both ISP-facing links.
6. Establish a separate TCP connection from LH6 to RH7 and verify its inside global address.
7. Restore ISP1's upstream interface and repeat the observations during failback.

A successful new connection through ISP2 verifies that failover works, even if the original connection cannot continue. The exact lifetime of old translations must be observed rather than inferred from the active default route.

Repeat the test with a local interface shutdown instead of an upstream failure to compare translation state in both cases.

### Clearing Stale Translations

To remove dynamic NAT state in the lab:

```text
Local_Router# clear ip nat translation *
```

When new traffic arrives, the NAT router creates translations using the applicable rules and current forwarding decisions.

Clearing translations can disrupt sessions using either ISP. It does not restore the original TCP session after a change of inside global address.

EEM can automate clearing translations in response to a tracked-object transition. However, indiscriminately clearing the entire translation table on every transition also interrupts unaffected connections. Prefer targeted clearing when necessary and supported, and account for both failover and failback. It is not a prerequisite for the two-route-map design.

See [NAT Translation Tables and Verification](../verification.md) for translation clearing and timeout commands.

---

## Optional: Track Both ISP Paths

The backup route in Lab 2 is installed whenever the primary route is unavailable, provided the backup route is otherwise valid. To monitor both paths, use a second probe target and track object.

Configure a second remote loopback and its route through ISP2:

```text
Remote_Router(config)# interface Loopback1
Remote_Router(config-if)# ip address 192.0.2.101 255.255.255.255
!
ISP2(config)# ip route 192.0.2.101 255.255.255.255 192.0.2.6
```

On Local_Router:

```text
ip route 192.0.2.101 255.255.255.255 Ethernet0/2 8.8.8.1
ip route 192.0.2.101 255.255.255.255 Null0 254
!
ip sla 2
 icmp-echo 192.0.2.101 source-interface Ethernet0/2
 timeout 1000
 frequency 3
!
ip sla schedule 2 life forever start-time now
!
track 2 ip sla 2 reachability
!
no ip route 0.0.0.0 0.0.0.0 Ethernet0/2 8.8.8.1 10
ip route 0.0.0.0 0.0.0.0 Ethernet0/2 8.8.8.1 10 track 2
```

The two default routes are now independently tracked.

| Track 1 | Track 2 | Installed default route |
|---|---|---|
| Up | Up | ISP1 |
| Up | Down | ISP1 |
| Down | Up | ISP2 |
| Down | Down | Neither |

This assumes no other source of default routes. The existing NAT rules remain unchanged.

---

## Active/Active Multi-ISP NAT

The same NAT policies also support an active/active design.

To convert the dual-tracked configuration to equal-cost multipath (ECMP), replace the backup default route with one using the default AD of 1:

```text
Local_Router(config)# no ip route 0.0.0.0 0.0.0.0 Ethernet0/2 8.8.8.1 10 track 2
Local_Router(config)# ip route 0.0.0.0 0.0.0.0 Ethernet0/2 8.8.8.1 track 2
```

When both tracked routes are eligible and installed as equal-cost paths, forwarding can use both ISPs. New translations are selected according to the egress path.

```text
Local_Router# show ip route 0.0.0.0
Local_Router# show ip cef 3.3.3.7
Local_Router# show ip cef exact-route 10.1.1.5 3.3.3.7
Local_Router# show ip cef exact-route 10.1.1.6 3.3.3.8
```

CEF load sharing normally uses a hash to select a path. Do not expect every successive ping or every second connection to use a different ISP. The distribution depends on the forwarding implementation and hash inputs.

Test multiple source/destination combinations and inspect both the actual forwarding path and NAT translations. If one tracked path fails, only the other default path remains available. Existing connections may be affected when the forwarding path changes.

If a particular application must always use a specific ISP while that ISP is available, use a suitable routing or PBR policy rather than relying on the ECMP hash.

---

## Multi-ISP NAT with Address Pools

Interface PAT is convenient when each ISP provides a single usable address. The same egress matching can instead select an ISP-specific address pool.

For example:

```text
ip nat pool ISP1-POOL 7.7.7.50 7.7.7.60 prefix-length 24
ip nat pool ISP2-POOL 8.8.8.50 8.8.8.60 prefix-length 24
```

After stopping traffic and clearing the old dynamic translations, replace the interface PAT rules:

```text
no ip nat inside source route-map NAT-ISP1 interface Ethernet0/1 overload
no ip nat inside source route-map NAT-ISP2 interface Ethernet0/2 overload
!
ip nat inside source route-map NAT-ISP1 pool ISP1-POOL overload
ip nat inside source route-map NAT-ISP2 pool ISP2-POOL overload
```

For non-overloaded dynamic NAT, omit `overload`. Each ISP must provide return reachability to its respective pool addresses. In this topology, those addresses are on-link from the corresponding ISP router's perspective, so ARP or explicit routes to Local_Router are required. If testing without relying on NAT ARP behavior, install `/32` routes on the appropriate ISP for the pool addresses being tested.

The NAT policies still follow the forwarding path; using pools does not change the failover mechanism.

---

## Verification and Troubleshooting

```text
Local_Router# show ip interface brief
Local_Router# show ip route 0.0.0.0
Local_Router# show ip route 192.0.2.100
Local_Router# show ip cef 3.3.3.7
Local_Router# show ip sla configuration 1
Local_Router# show ip sla statistics 1
Local_Router# show track
Local_Router# show ip policy
Local_Router# show route-map
Local_Router# show ip nat translations
Local_Router# show ip nat translations verbose
Local_Router# show ip nat statistics
```

| Symptom | What to check |
|---|---|
| Local ISP interface is down but primary route remains | Incorrect or additional default route configuration; recursive next-hop resolution |
| Primary ISP has an upstream outage but traffic does not fail over | Missing tracking, probe target too close, untracked primary route |
| Track stays up during primary failure | Probe escaping via ISP2, unsuitable probe target, alternate reachability to the target |
| Track stays down after recovery | Probe host route, return routing, ICMP filtering, SLA schedule, recovery delay |
| Backup route does not appear | Backup interface, next-hop reachability, track state, competing routes |
| Packets use the wrong ISP despite default-route failover | More-specific routes, PBR, additional routing policies |
| New flows use the wrong inside global address | NAT route map's `match interface`, conflicting rules, existing translation state |
| New flows are translated correctly but receive no replies | Upstream routing, source address validation, ARP for pool addresses |
| Existing sessions fail but new sessions work | Old NAT state or changed TCP/UDP endpoint |
| ECMP uses only one ISP in a small test | CEF hash inputs, installed paths, test traffic diversity |

Test from an inside host, not just from Local_Router. Router-originated IP SLA probes and pings can verify reachability without proving that forwarded inside traffic is translated correctly.

Use packet captures on the outside links to verify both the actual egress and the translated source address. A CEF lookup alone does not prove the outcome of PBR or NAT processing.

---

## Key Points

- A multi-ISP PAT design uses separate route maps to match the selected outgoing interfaces.
- The interface specified in a NAT statement supplies the inside global address; `match interface` classifies the forwarding path.
- Floating static routes provide backup routing when a preferred route becomes invalid.
- IP SLA and object tracking can withdraw a primary static route when the upstream path fails, even if the local interface remains up.
- Pin an SLA probe to the ISP it monitors independently of the tracked default route.
- An untracked floating Null0 host route can prevent the probe from following the backup default route when the primary interface goes down.
- NAT follows the selected path for new translations without requiring NAT configuration changes during failover.
- Existing NAT state may outlive a routing change. Changing the inside global address does not preserve an existing TCP connection.
- Equal-cost routes can use both ISPs with the same egress-based NAT policies.
- Both ISP paths can be independently tracked for active/standby or active/active operation.

---

## References

- [Cisco: Configure ISP Failover with Default Routes Using IP SLA Tracking](https://www.cisco.com/c/en/us/support/docs/ip/ip-routing/200785-ISP-Failover-with-default-routes-using-I.html)
- [Cisco IOS XE 17.x: Configuring IP SLAs ICMP Echo Operations](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_sla_icmp_echo.html)
- [Cisco IOS XE 17.x: Configuring Enhanced Object Tracking](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iap-eot-xe.html)
- [Cisco: IOS NAT Load-Balancing with Optimized Edge Routing for Two Internet Connections](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/99427-ios-nat-2isp.html)
- [Cisco: Troubleshoot Network Address Translation with Frequently Asked Questions](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/218012-troubleshoot-network-address-translation.html)
