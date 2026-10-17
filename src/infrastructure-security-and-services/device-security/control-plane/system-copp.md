# Catalyst 9000 System CoPP

**Catalyst 9000 System Control Plane Policing (System CoPP)** protects the switch CPU by policing packets sent from the forwarding hardware to the control plane.

Unlike traditional router CoPP, which allows administrators to create custom ACLs, class maps, and policy maps, Catalyst 9000 switches use a predefined policy named **`system-cpp-policy`**.

The system automatically creates and applies this policy during initialization.

System CoPP provides:

- Predefined traffic classes for different types of CPU-bound traffic.
- Hardware CPU queues associated with these classes.
- Hardware policers that limit traffic admitted to the CPU.
- Configurable policing rates in **packets per second (pps)**.
- Hardware counters for monitoring accepted and dropped packets.

The primary administrator-controlled parameter is the **policing rate of each predefined class**.

---

## System CoPP Operation

Catalyst 9000 switches use dedicated forwarding ASICs to handle normal transit traffic.

Packets requiring CPU processing are redirected from the forwarding hardware toward the control plane. This is known as **punting**.

Examples include:

- Routing protocol packets, such as OSPF and BGP.
- Management traffic, such as SSH and SNMP.
- ARP and IPv6 Neighbor Discovery.
- STP, CDP, and LACP.
- IGMP and MLD control messages.
- Packets requiring exceptional processing.

System CoPP classifies these packets into predefined hardware CPU queues and polices them before they reach the CPU.

```text
                   Incoming Packet
                          |
                          v
                    Forwarding ASIC
                          |
              +-----------+-----------+
              |                       |
         Normal Forwarding       CPU-Bound Packet
              |                       |
              v                       v
        Egress Interface        Punt Classification
                                      |
                                      v
                                  CPU Queue
                                      |
                                      v
                                Hardware Policer
                                      |
                              +-------+-------+
                              |               |
                          Conform           Exceed
                              |               |
                              v               v
                         Forward to         Drop
                            CPU
```

**System CoPP operates primarily in the forwarding hardware**, allowing excessive CPU-bound traffic to be dropped before it consumes significant CPU processing resources.

Normal hardware-forwarded transit traffic is not subject to System CoPP.

---

## Predefined System CoPP Policy

Catalyst 9000 switches automatically create the following policy:

```text
policy-map system-cpp-policy
```

The policy is installed on the control-plane interface:

```text
control-plane
 service-policy input system-cpp-policy
```

The policy is normally present by default and does not need to be created manually.

Verify:

```text
SW1# show policy-map control-plane
```

The output begins with:

```text
Control Plane

 Service-policy input: system-cpp-policy
```

### System CoPP Restrictions

On modern Catalyst 9000 platforms, the following restrictions apply:

| Feature | Support |
|---|---|
| Predefined `system-cpp-policy` | Yes |
| Ingress CoPP | Yes |
| Egress CoPP | No |
| Custom control-plane policy maps | No |
| User-defined CoPP class maps | No |
| Modifying predefined class matching criteria | No |
| Changing predefined class policing rates | Yes |
| Policing rates in pps | Yes |
| Policing rates in bps | No |

Starting with IOS XE Fuji 16.8.1a, Catalyst 9300 switches no longer support creating user-defined CoPP class maps.

The predefined policy and classes cannot be deleted or redefined.

**Administrators modify the policing actions of existing system-defined classes rather than creating new traffic classifications.**

This is the main configuration difference between traditional router CoPP and Catalyst 9000 System CoPP.

---

## System CoPP Classes

System CoPP organizes CPU-bound traffic into predefined classes based on its intended processing function.

Each class is associated with a hardware policer and one or more CPU queues.

The following are important examples from Catalyst 9300 IOS XE.

| System CoPP Class | Traffic Examples |
|---|---|
| `system-cpp-police-routing-control` | OSPF, BGP, EIGRP, PIM, HSRP, VRRP, BFD |
| `system-cpp-police-l2-control` | CDP, LLDP, LACP, UDLD, DTP, VTP |
| `system-cpp-police-topology-control` | STP, REP |
| `system-cpp-police-forus` | Traffic destined for the switch, such as ICMP, SSH, SNMP |
| `system-cpp-police-data` | ICMP-generated traffic, broadcasts, ICMP redirects |
| `system-cpp-police-multicast` | Multicast data and multicast forwarding exceptions |
| `system-cpp-police-multicast-end-station` | IGMP and MLD listener control traffic |
| `system-cpp-police-protocol-snooping` | Protocol snooping, including ARP inspection |
| `system-cpp-police-dhcp-snooping` | DHCP snooping |
| `system-cpp-police-sw-forward` | Software forwarding and certain logging packets |
| `system-cpp-police-dot1x-auth` | IEEE 802.1X authentication |
| `system-cpp-police-punt-webauth` | Web authentication |
| `system-cpp-police-sys-data` | Various system exceptions and control-plane punt traffic |

The exact class list, traffic mapping, and default policing rates depend on the Catalyst model and IOS XE release.

### Class Maps vs. CPU Queues

A **System CoPP class is not necessarily equivalent to one CPU queue**.

For example, Cisco documents the following relationship:

```text
system-cpp-police-data
        |
        +---- ICMP GEN queue
        |
        +---- BROADCAST queue
        |
        +---- ICMP Redirect queue
        |
        v
   Shared Hardware Policer
```

All three CPU queues are associated with the same System CoPP class.

**Changing the policer rate of a class affects every CPU queue associated with that class.**

This matters because one traffic type can consume policing capacity needed by another traffic type sharing the same policer.

---

## Shared Hardware Policers

Multiple traffic types can share a single hardware policer.

Consider the `system-cpp-police-forus` class, which handles several types of traffic destined for the switch.

Depending on the platform and software release, the corresponding policer may protect traffic such as:

- SSH.
- SNMP.
- ICMP Echo Requests.
- ARP or address resolution traffic.
- Other locally destined IP traffic.

Suppose the shared policer is configured for 4,000 pps.

```text
         SSH -------+
                    |
         SNMP ------+
                    |
         ICMP ------+----> Shared Policer: 4000 pps
                    |              |
         ARP -------+       +------+------+
                            |             |
                         Conform        Exceed
                            |             |
                            v             v
                           CPU           Drop
```

If excessive ICMP or ARP traffic consumes the available policing allowance, legitimate SSH or SNMP packets sharing that policer may also be dropped.

**The policer rate is shared among its associated traffic queues, not independently available to every protocol.**

This is an important troubleshooting consideration.

> The exact CPU queue-to-policer mapping differs across Catalyst 9000 platforms and IOS XE releases. Always verify the mapping on the target device.

---

## Configuring System CoPP Policing Rates

The main configurable parameter in System CoPP is the **policing rate**, expressed in packets per second.

The command is:

```text
police rate <pps> pps
```

Unlike traditional byte-based policing, you do not configure CIR in bps or Bc in bytes.

### Example: Modify a Predefined Class

Suppose we want to configure the protocol-snooping class to allow 2,000 pps.

Enter the predefined policy:

```text
SW1(config)# policy-map system-cpp-policy
```

Select the existing class:

```text
SW1(config-pmap)# class system-cpp-police-protocol-snooping
```

Configure the policing rate:

```text
SW1(config-pmap-c)# police rate 2000 pps
SW1(config-pmap-c-police)#
```

The resulting configuration is:

```text
policy-map system-cpp-policy
 class system-cpp-police-protocol-snooping
  police rate 2000 pps
```

This modifies the policer associated with protocol-snooping traffic.

**The class-map itself is not changed.** Only the policing rate is modified.

### Verify the Policy Attachment

System CoPP is normally applied automatically.

If the policy attachment must be restored, use:

```text
SW1(config)# control-plane
SW1(config-cp)# service-policy input system-cpp-policy
```

On platforms requiring explicit reinstallation after certain changes, this attaches the policy to the forwarding engine.

There is no need to apply the policy to individual physical switchports.

### Policing Behavior

In the documented System CoPP implementation, policers normally use:

- **Conform:** Transmit.
- **Exceed:** Drop.

For example:

```text
police:
  rate 2000 pps, burst 488 packets
  conformed ...; actions:
    transmit
  exceeded ...; actions:
    drop
```

The configured rate controls how quickly the packet-based token bucket is replenished.

The burst allowance is expressed in **packets**, not bytes.

The displayed burst value is determined by the platform's policing implementation.

---

## Hardware Rate Granularity

The configured policing rate is not necessarily the exact rate implemented in hardware.

On Catalyst 9300 platforms documented by Cisco, policing rates are rounded to the nearest supported increment of **200 pps**.

Examples:

| Configured Rate | Effective Rate |
|---|---|
| 100 pps | 200 pps |
| 400 pps | 400 pps |
| 650 pps | 600 pps |
| 2,000 pps | 2,000 pps |

For example:

```text
SW1(config)# policy-map system-cpp-policy
SW1(config-pmap)# class system-cpp-police-protocol-snooping
SW1(config-pmap-c)# police rate 650 pps
```

The hardware may implement a rate of 600 pps.

**The supported increment is hardware-dependent**, so verify the effective rate rather than assuming that every configured value is enforced exactly.

---

## Disabling a System CoPP Policer

Cisco documents the following command for removing a configured rate from a System CoPP class:

```text
no police rate <pps> pps
```

For example:

```text
SW1(config)# policy-map system-cpp-policy
SW1(config-pmap)# class system-cpp-police-protocol-snooping
SW1(config-pmap-c)# no police rate 2000 pps
```

**This is not the recommended way to restore a class's default policing rate.**

Cisco documents this operation as disabling the corresponding CPU queue or its policing protection, depending on the implementation.

Potential consequences include:

- Loss of CPU protection against excessive traffic.
- Disruption of legitimate control-plane traffic.
- Changes affecting multiple queues associated with the same class.

Cisco strongly recommends against disabling predefined policers.

On the documented Catalyst 9300 implementation, the system also restores default policer rates after a reload to protect system startup.

> For normal administration, modify policer rates rather than disabling them. Use `cpp system-default` when the objective is to restore default rates.

---

## Restoring Default Policing Rates

To restore the system-defined policing rates, use the following global configuration command:

```text
SW1(config)# cpp system-default
```

Example response:

```text
Defaulting CPP : Policer rate for all classes will be set to their defaults
```

**This restores the default rates for all System CoPP classes**, not just one class.

It overwrites customized policing rates.

Cisco also recommends this command following certain IOS XE software upgrades to apply the current release's default System CoPP values.

Verify the result:

```text
SW1# show policy-map control-plane
```

And:

```text
SW1# show platform hardware fed switch active qos queue stats internal cpu policer
```

Because hardware rates may be rounded, the displayed default and effective rates can differ.

---

## IOS XE Verification

System CoPP has both MQC policy verification commands and hardware-specific verification commands.

### Display the Applied Policy

```text
SW1# show policy-map control-plane
```

This shows:

- The active control-plane service policy.
- Predefined traffic classes.
- Policing rates.
- Packet and byte counters.
- Conform and exceed actions.

For example:

```text
Control Plane

 Service-policy input: system-cpp-policy

  Class-map: system-cpp-police-protocol-snooping (match-any)
    0 packets, 0 bytes
    5 minute offered rate 0000 bps, drop rate 0000 bps
    Match: none
    police:
      rate 2000 pps, burst 488 packets
      conformed 0 bytes; actions:
        transmit
      exceeded 0 bytes; actions:
        drop
```

The `Match: none` entry can appear because System CoPP classification is predefined and implemented by the platform rather than using ordinary user-configured MQC match statements.

### Display the System CoPP Policy

```text
SW1# show policy-map system-cpp-policy
```

This displays the predefined classes and policer configurations.

### Display Hardware CPU Queue and Policer Statistics

One of the most important Catalyst 9000 verification commands is:

```text
SW1# show platform hardware fed switch active qos queue stats internal cpu policer
```

The output contains information about hardware CPU queues and their policers.

An abbreviated example:

```text
CPU Queue Statistics
============================================================

QId PlcIdx Queue Name        Enabled Default Rate Set Rate Drop(Frames)
----------------------------------------------------------------------
0   11     DOT1X Auth        Yes       1000        1000       0
1    1     L2 Control        Yes       2000        2000       0
2   14     Forus traffic     Yes       4000        4000       0
4    2     Routing Control   Yes       5400        5400       0
16  12     Proto Snooping    Yes       2000        2000       0
```

The fields are:

| Field | Meaning |
|---|---|
| `QId` | Hardware CPU queue identifier |
| `PlcIdx` | Hardware policer index |
| `Queue Name` | Traffic category |
| `Enabled` | Whether the queue is enabled |
| `Default Rate` | System-defined default policing rate |
| `Set Rate` | Effective configured policing rate |
| `Drop(Bytes)` | Bytes dropped by the policer |
| `Drop(Frames)` | Packets dropped by the policer |

This output is particularly useful because it exposes the **hardware-enforced policing rates**, rather than merely the configured policy.

### Identifying Shared Policers

Consider:

```text
QId  PlcIdx  Queue Name
-----------------------------
3       0   ICMP GEN
6       0   ICMP Redirect
12      0   BROADCAST
```

All three queues use policer index 0.

Therefore, they share the same hardware policer.

**Matching `PlcIdx` values indicate that queues share policing resources.**

The output also includes aggregate statistics for each hardware policer and, on platforms supporting them, additional second-level policing information.

---

## CPU Queue and Punt Verification

The following commands help identify which traffic is being punted to the CPU and how the forwarding engine processes it.

### Display Punt Causes

```text
SW1# show platform software fed switch active punt cause summary
```

This displays why packets are being redirected toward the CPU.

For example:

```text
Cause  Cause Info                     Rcvd       Dropped
-------------------------------------------------------
7      ARP request or response        142962     0
11     For-us data                    490817     0
24     Glean adjacency                2          0
55     For-us control                 415222     0
96     Layer2 control protocols       358614     0
```

The important distinction is:

- **Punt cause:** Why was the packet sent toward the CPU?
- **CPU queue:** Where was the punted packet classified?
- **Policer:** What rate limit was applied?

These are related, but they are not interchangeable.

### Display CPU Queue Statistics

```text
SW1# show platform software fed switch active punt cpuq all
```

This displays statistics for all CPU punt queues.

To inspect a specific queue:

```text
SW1# show platform software fed switch active punt cpuq 16
```

Queue 16 corresponds to protocol snooping in the documented Catalyst 9300 mapping.

Example fields include:

```text
CPU Q Id                    : 16
CPU Q Name                  : CPU_Q_PROTO_SNOOPING
Packets received from ASIC  : 55661
Send to IOSd total attempts : 55661
Send to IOSd failed count   : 0
RX dropped count            : 0
```

These counters help determine whether packets passed hardware policing but encountered problems later in the software punt path.

**Hardware policer drops and software CPU queue drops are different events.**

A packet discarded by a hardware policer does not proceed through the normal software punt queue processing path.

### Display CPU Queue Rates

```text
SW1# show platform software fed switch active punt cpuq rates
```

Displays packet rates associated with CPU punt queues on supported platforms.

This helps identify which traffic categories are generating unusually high CPU-bound packet rates.

### Identify Ingress Interfaces

```text
SW1# show platform software fed switch active punt rates interfaces
```

Displays rates of CPU-bound traffic arriving through individual interfaces.

This is especially useful when excessive punt traffic appears to originate from a particular physical port.

---

## Troubleshooting System CoPP Drops

A common troubleshooting scenario is a protocol or management function failing even though the CPU utilization is low.

For example:

- BGP neighbors intermittently reset.
- SSH connections time out.
- ARP resolution becomes unreliable.
- STP or LACP packets are lost.
- DHCP snooping or multicast control traffic experiences drops.

**Low CPU utilization does not prove that System CoPP is operating without drops.**

System CoPP may be successfully protecting the CPU by discarding excessive traffic before it reaches CPU processing.

### Troubleshooting Procedure

1. Verify the active policy:

   ```text
   SW1# show policy-map control-plane
   ```

2. Check hardware policer drops:

   ```text
   SW1# show platform hardware fed switch active qos queue stats internal cpu policer
   ```

3. Identify the CPU queue and associated policer index.

4. Determine which other queues share the policer.

5. Inspect punt causes:

   ```text
   SW1# show platform software fed switch active punt cause summary
   ```

6. Inspect CPU queue statistics:

   ```text
   SW1# show platform software fed switch active punt cpuq all
   ```

7. Identify the interfaces generating CPU-bound traffic:

   ```text
   SW1# show platform software fed switch active punt rates interfaces
   ```

8. If legitimate traffic is being dropped, investigate the traffic source and determine whether the existing policer rate is appropriate.

Do not immediately raise policing rates without investigating the cause of the excess traffic.

### Historical vs. Active Drops

Many System CoPP hardware counters accumulate over time.

A nonzero drop counter does not necessarily indicate a current problem.

Run the command several times:

```text
SW1# show platform hardware fed switch active qos queue stats internal cpu policer
```

If the relevant drop counter continues increasing while the problem occurs, System CoPP is actively discarding packets.

If the counter remains unchanged, the drops may be historical.

---

## Hierarchical Hardware Policing

Some Catalyst 9000 platforms implement an additional level of shared hardware policing.

This provides another layer of protection for groups of CPU queues.

For example, a platform may have:

- A first-level policer for a particular CPU traffic class.
- A second-level policer shared across several first-level policers.

```text
CPU Queue A ----> Level-1 Policer A ---+
                                      |
CPU Queue B ----> Level-1 Policer B ---+---> Level-2 Policer
                                      |            |
CPU Queue C ----> Level-1 Policer C ---+            v
                                                  CPU
```

In this architecture, a packet may need to satisfy both applicable policers before reaching the CPU.

Cisco documents second-level categories associated with routing and other system traffic, including:

- `system-cpp-police-ios-routing`
- `system-cpp-police-ios-feature`

The exact hierarchy depends on the platform.

The hardware CPU policer statistics command can display:

```text
Second Level Policer Statistics
```

And:

```text
Policer Index Mapping and Settings
```

These sections help identify whether drops or congestion are occurring at an additional shared policing level.

For CCIE troubleshooting, remember that **a packet can pass its individual queue's policer but still be limited by another policing stage** on platforms with hierarchical System CoPP.

---

## System CoPP and Software Upgrades

System CoPP policies are automatically created and managed by IOS XE.

However, software upgrades may introduce:

- New traffic classes.
- Different CPU queue assignments.
- Updated default policing rates.
- Changes in available hardware policing resources.

Cisco documents using:

```text
SW1(config)# cpp system-default
```

After certain software upgrades to restore the release's default policing rates.

This command affects all System CoPP classes.

Always review customized rates before resetting the policy.

### Saving Configuration

Changes to System CoPP policing rates should be saved:

```text
SW1# copy running-config startup-config
```

Default System CoPP configuration may not appear completely in `show running-config`.

Use:

```text
SW1# show policy-map system-cpp-policy
```

Or:

```text
SW1# show policy-map control-plane
```

To inspect system-defined classes and their active policers.

---

## Key Points

- Catalyst 9000 switches use a predefined System CoPP policy named **`system-cpp-policy`**.
- The policy is normally created and applied automatically during system initialization.
- System CoPP protects the CPU by **policing punted traffic in hardware**.
- Only **ingress CoPP** is supported in the documented Catalyst 9300 implementation.
- Modern Catalyst 9000 System CoPP does not support creating custom CoPP classes or replacing the system-defined policy.
- Administrators can modify policing rates for existing system-defined classes.
- Configure rates with **`police rate <pps> pps`**.
- System CoPP rates are configured in **packets per second**, not bits per second.
- **Multiple CPU queues can share one hardware policer.**
- Increasing or decreasing a class's policer rate affects all associated CPU queues.
- Heavy traffic from one protocol can cause other protocols sharing the same policer to experience drops.
- Hardware may round configured rates to supported increments.
- On documented Catalyst 9300 implementations, rates are rounded to the nearest **200 pps**.
- Do not use **`no police rate`** as a normal method of restoring default policer rates.
- Use **`cpp system-default`** to restore the default rates for all System CoPP classes.
- Use **`show policy-map control-plane`** to inspect the applied policy and policers.
- Use **`show platform hardware fed switch active qos queue stats internal cpu policer`** to inspect hardware rates, CPU queues, shared policers, and drops.
- Use **`show platform software fed switch active punt cause summary`** to identify why packets are punted.
- Use **`show platform software fed switch active punt cpuq all`** to inspect software punt queue statistics.
- **Punt causes, CPU queues, and policers are distinct concepts.**
- Some Catalyst 9000 platforms implement additional second-level hardware policers.
- System CoPP drops can occur even when CPU utilization is low.
- Hardware policer counters may be cumulative, so check whether they are actively increasing.
- System-defined classes, default rates, and hardware policing behavior vary by Catalyst model and IOS XE release.

---

## References

- [Cisco Catalyst 9300 IOS XE 17.18.x - Configuring Control Plane Policing](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-18/configuration_guide/sec/b_1718_sec_9300_cg/configuring_control_plane_policing.html)
- [Cisco - Troubleshoot Control Plane Operations on Catalyst 9000 Switches](https://www.cisco.com/c/en/us/support/docs/switches/catalyst-9300-switch/221841-troubleshoot-control-plane-operations-on.html)
- [Cisco - Troubleshoot CPU Usage in Catalyst Switches on Cisco IOS XE](https://www.cisco.com/c/en/us/support/docs/ios-nx-os-software/ios-xe-16/213549-troubleshoot-high-cpu-usage-in-catalyst.html)
- [Cisco - Operate and Troubleshoot DHCP Snooping on Catalyst 9000 Switches](https://www.cisco.com/c/en/us/support/docs/ip/dynamic-host-configuration-protocol-dhcp-dhcpv6/217055-operate-and-troubleshoot-dhcp-snooping.html)