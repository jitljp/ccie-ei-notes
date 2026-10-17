# CoPP Configuration

**Control Plane Policing (CoPP)** protects a device's CPU by applying QoS policies to traffic directed toward the control plane.

On supported Cisco IOS XE routers, CoPP is configured using the **Modular QoS CLI (MQC)** framework.

The basic configuration process is:

1. Create ACLs to identify CPU-bound traffic.
2. Create class maps that reference the ACLs.
3. Create a policy map defining policing actions for each class.
4. Apply the policy map to the **control plane**.
5. Verify traffic classification and policing statistics.

Unlike ordinary interface QoS policies, CoPP policies are applied to the logical `control-plane` interface rather than individual physical interfaces.

---

## Basic CoPP Configuration

Consider the following topology:

```text
                  R2
             10.0.12.2
                  |
                  | OSPF and BGP
                  |
             Ethernet0/0
              10.0.12.1
                  |
                  R1
             Loopback0
             10.255.0.1
                  |
                  | SSH
                  |
            Management PC
             10.10.10.10
```

R1 runs OSPF and BGP with R2 and accepts SSH connections to Loopback0.

We will create a CoPP policy that:

- Polices routing protocol traffic at 1 Mbps.
- Polices SSH traffic at 256 kbps.
- Polices ICMP Echo Requests at 128 kbps.
- Leaves unmatched CPU-bound traffic unrestricted.

The policing rates are illustrative, not recommended values for a production network.

### Step 1: Create Classification ACLs

First, create an ACL to classify routing protocol traffic.

```text
R1(config)# ip access-list extended COPP-ROUTING-ACL
R1(config-ext-nacl)# permit ospf any any
R1(config-ext-nacl)# permit tcp host 10.0.12.2 any eq 179
R1(config-ext-nacl)# permit tcp host 10.0.12.2 eq 179 any
```

The ACL matches:

- OSPF packets.
- BGP packets from R2 with destination TCP port 179.
- BGP packets from R2 with source TCP port 179.

Both TCP port directions are included because either BGP peer can initiate the TCP connection.

Next, classify SSH traffic destined for R1's Loopback0 address:

```text
R1(config)# ip access-list extended COPP-SSH-ACL
R1(config-ext-nacl)# permit tcp any host 10.255.0.1 eq 22
```

Finally, classify ICMP Echo Requests destined for R1:

```text
R1(config)# ip access-list extended COPP-ICMP-ACL
R1(config-ext-nacl)# permit icmp any any echo
```

This ACL matches ICMP Echo Requests reaching the control plane, including requests directed toward addresses other than Loopback0.

> ACLs referenced by CoPP class maps are used for **traffic classification**. An ACL `permit` identifies matching traffic, while an ACL `deny` excludes traffic from that ACL match criterion. ACL `deny` statements do not automatically drop packets.

### Step 2: Create Class Maps

Create a class map for each traffic category.

```text
R1(config)# class-map match-all COPP-ROUTING
R1(config-cmap)# match access-group name COPP-ROUTING-ACL

R1(config)# class-map match-all COPP-SSH
R1(config-cmap)# match access-group name COPP-SSH-ACL

R1(config)# class-map match-all COPP-ICMP
R1(config-cmap)# match access-group name COPP-ICMP-ACL
```

Each class map matches the packets permitted by its corresponding ACL.

Because each class map contains only one match criterion, `match-all` and `match-any` would produce the same result.

## Step 3: Create a Policy Map

Create a policy map and define the policing rate for each class.

```text
R1(config)# policy-map COPP-IN
R1(config-pmap)# class COPP-ROUTING
R1(config-pmap-c)# police cir 1000000 bc 31250 conform-action transmit exceed-action drop

R1(config-pmap)# class COPP-SSH
R1(config-pmap-c)# police cir 256000 bc 8000 conform-action transmit exceed-action drop

R1(config-pmap)# class COPP-ICMP
R1(config-pmap-c)# police cir 128000 bc 4000 conform-action transmit exceed-action drop
```

The policy applies the following policing parameters:

| Class | CIR | Bc | Exceed Action |
|---|---|---|---|
| `COPP-ROUTING` | 1 Mbps | 31,250 bytes | Drop |
| `COPP-SSH` | 256 kbps | 8,000 bytes | Drop |
| `COPP-ICMP` | 128 kbps | 4,000 bytes | Drop |
| `class-default` | No policing | N/A | No policing action |

### Choosing Bc Values

In this example, each **Bc** value was calculated to allow **250 ms worth of traffic at the CIR**, which is the default Bc value for traditional IOS XE policing when Bc is not explicitly configured.

The formula is:

```
Bc (bytes) = (CIR × 0.25) / 8 = CIR / 32
```

Therefore:

| Class | Calculation | Bc |
|---|---|---|
| Routing | 1,000,000 / 32 | 31,250 bytes |
| SSH | 256,000 / 32 | 8,000 bytes |
| ICMP | 128,000 / 32 | 4,000 bytes |

These values are equivalent to the default Bc calculation, so **the `bc` parameters could be omitted** on IOS XE platforms using this default.

The values are illustrative, not universal recommendations. In production, Bc should be selected based on expected traffic bursts, particularly for routing protocols that may generate additional traffic during convergence.

> The 250 ms default is not universal across all IOS XE platforms and policing implementations.

### Step 4: Apply the Policy to the Control Plane

The policy map must be applied under the logical control-plane interface.

```text
R1(config)# control-plane
R1(config-cp)# service-policy input COPP-IN
```

The `input` keyword applies the policy to traffic received by the control plane.

**This is the command that activates the CoPP policy.**

The configuration is similar to applying a QoS service policy to a physical interface, but CoPP uses `control-plane` configuration mode.

```text
interface Ethernet0/0
 service-policy input QOS-IN
```

Compared with:

```text
control-plane
 service-policy input COPP-IN
```

The first applies an interface QoS policy.

The second applies a policy to CPU-bound traffic.

> You do not need to apply the CoPP policy to every physical interface. The aggregate control-plane policy covers applicable CPU-bound traffic arriving through multiple interfaces, subject to platform-specific forwarding paths.

---

## CoPP Input vs. Output Policies

On supported IOS XE platforms, the `service-policy` command supports two directions.

```text
R1(config-cp)# service-policy input <policy-map>
R1(config-cp)# service-policy output <policy-map>
```

| Direction | Traffic | Purpose |
|---|---|---|
| `input` | Forwarding plane → Control plane | Protects CPU resources from incoming traffic |
| `output` | Control plane → Forwarding plane | Controls traffic generated or transmitted by the control plane |

### Input CoPP

An input policy regulates traffic before it reaches the control-plane processing path.

```text
Incoming Packet
       |
       v
 Forwarding Plane
       |
       v
   Input CoPP
       |
       +---- Conform ----> CPU
       |
       +---- Exceed -----> Drop
```

This is the primary mechanism used to protect the CPU against excessive incoming traffic.

### Output CoPP

An output policy acts on traffic leaving the control plane.

```text
       CPU
        |
        v
  Output CoPP
        |
        +---- Conform ----> Forwarding Plane
        |
        +---- Exceed -----> Drop
```

Output policing can restrict control-plane-generated traffic, but it does not prevent the CPU from having already performed the work required to generate the packet.

Therefore, **input CoPP is the primary focus for control-plane CPU protection**.

Support for output CoPP depends on the platform.

---

## Policing vs. Dropping an Entire Class

A CoPP policy can either rate-limit traffic or discard all packets matching a particular class.

### Rate-Limit a Class

For example:

```text
R1(config)# policy-map COPP-IN
R1(config-pmap)# class COPP-SSH
R1(config-pmap-c)# police cir 256000 bc 8000 conform-action transmit exceed-action drop
```

This permits conforming traffic and drops traffic exceeding the token bucket allowance.

### Drop All Packets in a Class

To drop all packets matching a class, configure both conform and exceed actions to drop:

```text
R1(config)# policy-map COPP-IN
R1(config-pmap)# class COPP-UNWANTED
R1(config-pmap-c)# police cir 8000 bc 1500 conform-action drop exceed-action drop
```

The policer drops packets regardless of whether they conform to the rate.

A direct class-level `drop` action is also available on certain IOS XE platforms.

> CoPP class-map matching does not, by itself, authorize or block a connection. A class must have the appropriate policy action to cause packets to be discarded.

---

## Protecting Trusted Management Traffic

A common requirement is to give trusted management traffic different treatment from other traffic.

Suppose only `10.10.10.10` is an authorized management workstation.

We can create two SSH traffic classes:

1. Trusted SSH traffic.
2. Other SSH traffic.

### Classification ACLs

```text
R1(config)# ip access-list extended COPP-TRUSTED-SSH-ACL
R1(config-ext-nacl)# permit tcp host 10.10.10.10 host 10.255.0.1 eq 22
```

```text
R1(config)# ip access-list extended COPP-OTHER-SSH-ACL
R1(config-ext-nacl)# permit tcp any host 10.255.0.1 eq 22
```

The second ACL also matches traffic from the trusted workstation.

However, we will place the trusted class first in the policy map.

### Class Maps

```text
R1(config)# class-map match-all COPP-TRUSTED-SSH
R1(config-cmap)# match access-group name COPP-TRUSTED-SSH-ACL

R1(config)# class-map match-all COPP-OTHER-SSH
R1(config-cmap)# match access-group name COPP-OTHER-SSH-ACL
```

### Policy Map

```text
R1(config)# policy-map COPP-IN
R1(config-pmap)# class COPP-TRUSTED-SSH
R1(config-pmap-c)# police cir 256000 bc 8000 conform-action transmit exceed-action drop
R1(config-pmap-c)# exit
R1(config-pmap)# class COPP-OTHER-SSH
R1(config-pmap-c)# police cir 8000 bc 1500 conform-action drop exceed-action drop
```

The classes are evaluated in order:

```text
Incoming SSH Packet
        |
        v
 COPP-TRUSTED-SSH
        |
        +-- Match --> Rate-limit at 256 kbps
        |
        +-- No Match
               |
               v
        COPP-OTHER-SSH
               |
               +-- Match --> Drop
```

The trusted workstation matches the first class, so it is not evaluated against the second class.

Other SSH clients match the second class and are dropped.

**The order of policy-map classes is important when match criteria overlap.**

> CoPP should not be the only mechanism used to restrict SSH access. VTY access controls and AAA provide additional protection and authorization.

---

## IPv6 CoPP Configuration

On supported IOS XE router platforms, CoPP can also classify IPv6 traffic.

For example, configure a policy to limit ICMPv6 Echo Requests.

### Step 1: Create an IPv6 ACL

```text
R1(config)# ipv6 access-list COPP-ICMPV6-ACL
R1(config-ipv6-acl)# permit icmp any any echo-request
```

### Step 2: Create a Class Map

```text
R1(config)# class-map match-all COPP-ICMPV6
R1(config-cmap)# match access-group name COPP-ICMPV6-ACL
```

### Step 3: Add the Class to the Policy Map

```text
R1(config)# policy-map COPP-IN
R1(config-pmap)# class COPP-ICMPV6
R1(config-pmap-c)# police cir 128000 bc 4000 conform-action transmit exceed-action drop
```

If `COPP-IN` is already applied to the control plane, changes to the policy take effect subject to the platform's policy update behavior.

Otherwise, apply the policy:

```text
R1(config)# control-plane
R1(config-cp)# service-policy input COPP-IN
```

This limits the rate of ICMPv6 Echo Requests matching the class.

Other ICMPv6 messages are not matched by this ACL.

**Do not indiscriminately drop all ICMPv6 traffic**, because IPv6 relies on ICMPv6 for essential functions such as Neighbor Discovery and Path MTU Discovery.

> IPv6 CoPP support depends on the platform. Some IOS XE platforms support IPv6 ACL-based classification, while others have restrictions or use predefined hardware policing classes.

---

## Modifying an Existing CoPP Policy

After a policy has been applied, you may need to modify its policing rates.

For example, increase the SSH CIR from 256 kbps to 512 kbps.

```text
R1(config)# policy-map COPP-IN
R1(config-pmap)# class COPP-SSH
R1(config-pmap-c)# police cir 512000 bc 16000 conform-action transmit exceed-action drop
```

The policy remains attached to the control plane if the platform supports modifying the policy in place.

Some hardware platforms may reject certain changes while a policy is attached, requiring the policy to be removed and reapplied.

Verify the result:

```text
R1# show policy-map control-plane
```

### Removing a CoPP Policy

Remove the service policy from the control plane:

```text
R1(config)# control-plane
R1(config-cp)# no service-policy input COPP-IN
```

This detaches the policy without deleting its class maps, ACLs, or policy-map configuration.

To delete the policy map itself:

```text
R1(config)# no policy-map COPP-IN
```

The policy must not be in use by another service-policy attachment when it is deleted.

> Removing a CoPP policy can expose the control plane to excessive traffic. In production, replace or modify the policy carefully rather than removing protection unnecessarily.

---

## Safe CoPP Deployment

An incorrect CoPP policy can disrupt essential control-plane protocols or prevent remote administration.

For example, a policy with an excessively low OSPF policing rate might cause OSPF packets to be dropped during a topology change.

A policy that drops SSH traffic might disconnect remote administrative sessions.

A safer deployment process is:

1. Identify legitimate control-plane traffic.
2. Create traffic classes with sufficiently specific ACLs.
3. Configure policing rates based on expected traffic.
4. Initially configure `exceed-action transmit` when supported.
5. Apply the policy and examine class counters.
6. Verify routing protocol and management operations.
7. Change the exceed action to `drop` after validating the rates.
8. Monitor drops during normal operation and convergence events.

### Initial Monitoring Configuration

For example:

```text
R1(config)# policy-map COPP-IN
R1(config-pmap)# class COPP-ROUTING
R1(config-pmap-c)# police cir 1000000 bc 31250 conform-action transmit exceed-action transmit
```

Both conforming and exceeding traffic is transmitted.

This allows the class to be evaluated without enforcing the committed rate through packet drops.

After determining that the rate and burst allowance are sufficient, change the action:

```text
R1(config-pmap-c)# police cir 1000000 bc 31250 conform-action transmit exceed-action drop
```

Now packets exceeding the policer's allowance are discarded.

> Keep a separate administrative recovery method available when applying restrictive CoPP policies. Console or out-of-band access can be particularly valuable if a policy accidentally blocks remote management.

---

## IOS XE Verification

The most important command for CoPP verification is:

```text
R1# show policy-map control-plane
```

This displays the policy attached to the control plane and its traffic statistics.

### Verify the Applied Policy

```text
R1# show running-config | section control-plane
```

Expected configuration:

```text
control-plane
 service-policy input COPP-IN
```

Alternatively:

```text
R1# show policy-map control-plane input
```

This displays the input control-plane policy and its statistics on supported IOS XE platforms.

### Verify the Policy Map

```text
R1# show policy-map COPP-IN
```

Displays the configured classes and actions.

### Verify Class Maps

```text
R1# show class-map
```

Displays the configured classification criteria.

### Verify Classification ACLs

```text
R1# show ip access-lists COPP-ROUTING-ACL
R1# show ip access-lists COPP-SSH-ACL
R1# show ip access-lists COPP-ICMP-ACL
```

For IPv6:

```text
R1# show ipv6 access-list COPP-ICMPV6-ACL
```

ACL counters may not reflect all hardware-classified CoPP traffic, so policy-map statistics are generally more useful for checking CoPP matches and drops.

---

## Interpreting CoPP Statistics

Consider the following illustrative output:

```text
R1# show policy-map control-plane

Control Plane

 Service-policy input: COPP-IN

  Class-map: COPP-SSH (match-all)
    1000 packets, 800000 bytes
    5 minute offered rate 200000 bps, drop rate 20000 bps
    Match: access-group name COPP-SSH-ACL
    police:
      cir 256000 bps, bc 8000 bytes
      conformed 900 packets, 720000 bytes; actions:
        transmit
      exceeded 100 packets, 80000 bytes; actions:
        drop

  Class-map: class-default (match-any)
    Match: any
```

The output shows:

| Field | Meaning |
|---|---|
| `Service-policy input` | Policy applied to incoming control-plane traffic |
| `Class-map` | Traffic class being evaluated |
| `offered rate` | Observed traffic rate entering the class |
| `drop rate` | Observed packet drop rate |
| `cir` | Configured committed rate |
| `bc` | Configured committed burst allowance |
| `conformed` | Packets within the policer's allowance |
| `exceeded` | Packets exceeding the allowance |

In this example, 100 packets were dropped because they exceeded the policer's allowance.

A small number of drops may be acceptable for noncritical traffic.

However, persistent drops in routing protocol classes may indicate that the policing rate or burst allowance is too low.

### Troubleshooting Classification

If a class shows no matched packets:

- Verify that the classification ACL matches the expected traffic.
- Confirm that the traffic is actually reaching the control plane.
- Check whether an earlier policy-map class is matching the traffic.
- Confirm that the policy is applied in the correct direction.
- Check platform-specific restrictions and hardware counters.

### Troubleshooting Unexpected Drops

If legitimate traffic is being dropped:

- Check the class's CIR and Bc values.
- Check whether traffic bursts exceed the available token allowance.
- Verify that critical protocols are classified correctly.
- Consider whether the policing rate is sufficient during routing convergence.
- Check `class-default` for unexpected traffic matches or aggressive policing.

---

## Platform Considerations

CoPP capabilities differ between IOS XE platforms.

### IOS XE Routers

Supported router platforms allow user-defined class maps and policy maps to be attached to the control plane.

However, supported match criteria, policing options, counter availability, and hardware resources vary.

Certain platforms also provide predefined punt policers or additional control-plane protection mechanisms.

### Catalyst 9000 Switches

Modern Catalyst 9000 switches use predefined **System CoPP** policies.

For example, Catalyst 9300 switches use `system-cpp-policy`.

Unlike router platforms supporting arbitrary user-defined CoPP classes, Catalyst 9000 System CoPP generally requires administrators to work with predefined classes and supported policing parameters.

System CoPP configuration is covered separately in the [Catalyst 9000 System CoPP](system-copp.md) chapter.

### Out-of-Band Management Interfaces

Some IOS XE platforms have dedicated management interfaces connected directly to the route processor.

Traffic arriving through these interfaces may bypass the forwarding-plane CoPP processing path.

For example, Cisco documents this behavior for certain ASR 1000 control-plane implementations.

Therefore, **CoPP coverage of dedicated management interfaces must be verified for the specific platform**.

---

## Key Points

- CoPP is configured using **MQC class maps and policy maps**.
- ACLs classify traffic, while policy maps define policing actions.
- Configure aggregate CoPP using `control-plane` in global configuration mode.
- Apply an input policy with **`service-policy input <policy-map>`**.
- Input CoPP protects the CPU from excessive incoming traffic.
- Output CoPP controls traffic leaving the control plane, where supported.
- A single aggregate CoPP policy can protect traffic arriving through multiple physical interfaces.
- **ACL `deny` does not automatically drop traffic** when the ACL is used for class-map matching.
- Policy-map classes are evaluated in order, so overlapping match criteria must be considered.
- To drop an entire class using a two-color policer, configure both conform and exceed actions to drop.
- **`class-default`** matches traffic that does not match earlier classes.
- Aggressively policing `class-default` can disrupt legitimate control-plane traffic.
- For safer initial deployment, use **`exceed-action transmit`** to observe traffic before enforcing drops.
- Remove an applied CoPP policy with **`no service-policy input <policy-map>`** under `control-plane`.
- Use **`show policy-map control-plane`** to verify the applied policy, match counters, and policing drops.
- CoPP support and available configuration options depend on the IOS XE platform.

---

## References

- [Cisco IOS XE 17.x - Control Plane Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-ctrl-pln-plc-0.html)
- [Cisco IOS XE 17.x - Applying QoS Features Using the MQC](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-apply.html)
- [Cisco ASR 1000 - Control Plane Policing](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_plcshp/configuration/xe-3s-asr1000/qos-plcshp-xe-3s-asr-1000-book/qos-plcshp-ctrl-pln-plc.html)
- [Cisco - Control Plane Policing Implementation Best Practices](https://www.cisco.com/c/en/us/about/security-center/copp-best-practices.html)
- [Cisco Catalyst 9300 IOS XE 17.18.x - Configuring Control Plane Policing](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-18/configuration_guide/sec/b_1718_sec_9300_cg/configuring_control_plane_policing.html)