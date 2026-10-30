# 4.2.a (ii) Storm Control

**Storm control** is a Layer 2 switch feature that protects the network from excessive broadcast, multicast, or unicast traffic arriving on a switch interface. A traffic storm can consume link bandwidth and switch forwarding resources, disrupt hosts, and spread across a VLAN through flooding.

Common causes include:

- A Layer 2 forwarding loop that repeatedly replicates flooded traffic.
- A faulty network interface, host, or application generating excessive frames.
- Excessive unknown-unicast flooding, for example when destination MAC addresses are not being learned or have aged out.
- A deliberate traffic-flooding attack.

Storm control monitors selected **ingress traffic classes** and takes action when their configured rates exceed a threshold. It can suppress the affected traffic or place the entire interface in an error-disabled state.

Storm control is **disabled by default** on typical Catalyst switches. Configuring a threshold enables storm control for that traffic class on the interface.

> Storm control mitigates the *effects* of a storm. It does not locate or repair a switching loop, replace STP, or provide per-host/per-flow security filtering.

---

## Traffic Types

The `storm-control` command identifies traffic by the destination MAC address and the forwarding category.

| Traffic type | Description | Examples |
|---|---|---|
| **Broadcast** | Frames sent to the Ethernet broadcast destination `FFFF.FFFF.FFFF`. | ARP requests, DHCP Discover messages |
| **Multicast** | Frames sent to multicast destination MAC addresses. | IPv4/IPv6 multicast, some routing-protocol packets |
| **Unicast** | Unicast traffic, including both known and unknown unicast on platforms such as Catalyst 9300. | Ordinary unicast traffic and unknown-destination flooding |
| **Unknown unicast** | Unicast frames whose destination MAC address is not in the switch's MAC address table for the VLAN. The switch would otherwise flood them. | Traffic to an aged-out or never-learned destination MAC |

On Catalyst 9300 switches, `unknown-unicast` is a separate command option in IOS XE 16.11.1 and later. Not all switch families or releases provide this keyword.

### Known Versus Unknown Unicast

**Known unicast:** The destination MAC address is present in the MAC address table, so the switch forwards the frame toward a specific port or port channel.

**Unknown unicast:** The destination MAC address has not been learned for the relevant VLAN, so the switch ordinarily floods the frame out eligible ports in that VLAN.

Both use a **unicast destination MAC address**. Whether a particular platform's `unicast` command applies to known traffic, unknown traffic, or separate underlying hardware policers is a platform-specific detail.

**Catalyst 9300 behavior:** Cisco documents `storm-control unicast` as covering **known and unknown unicast**, but the hardware evaluates the two categories separately. For example, with a 10% unicast threshold:

| Known unicast | Unknown unicast | Documented Catalyst 9300 result |
|---|---|---|
| 7% | 8% | Neither individual traffic class crosses 10%; no suppression despite 15% combined traffic. |
| 7% | 11% | Unknown-unicast traffic is suppressed. |
| 11% | 11% | Both traffic classes are suppressed. |

Do not assume that `storm-control unicast level 10` always imposes **one combined 10% limit** on all unicast traffic.

> Cisco warns against configuring **both** `storm-control unicast` and `storm-control unknown-unicast` on the same Catalyst 9300 interface because the commands can interfere with the unknown-unicast hardware suppression setting. Choose the category that matches the intended policy.

### Multicast and Control Traffic

Multicast storm control can suppress more than high-volume application multicast. On Catalyst switches, it can also suppress IP multicast routing and control packets, such as OSPF Hellos sent to `224.0.0.5` and `224.0.0.6`.

Cisco documents exceptions for certain Layer 2 control protocols, including **STP BPDUs and CDP**, which are not blocked by multicast storm control on the applicable Catalyst platforms. This protection does **not** extend to every protocol using a multicast destination MAC address.

A multicast threshold that is too low can therefore interrupt routing adjacencies or other legitimate multicast-dependent services.

---

## How Storm Control Works

Storm control operates on the **receiving interface**. It measures the rate of an enabled traffic class, compares that rate with a configured threshold, and suppresses traffic when the threshold is reached or exceeded.

For example:

```text
                 Excessive broadcasts
                          |
                          v
Host H1 -------- Gi1/0/1 SW1 -------- Other VLAN 10 ports
                          |
                   Storm control
                   on Gi1/0/1
```

If Gi1/0/1 has broadcast storm control configured, SW1 measures broadcasts **received on Gi1/0/1**, not the combined broadcasts received across VLAN 10.

- Normal-rate broadcasts can enter the switch and be flooded where permitted.
- Once the configured limit is reached, storm control suppresses the offending traffic according to the platform's implementation.
- Unconfigured traffic classes continue to be forwarded unless an interface-shutdown action occurs or another feature blocks them.

Storm-control limits generally apply to the **interface's ingress traffic**, including the traffic of multiple VLANs carried by a trunk, rather than being an independent limit for each VLAN.

### Measurement Interval and Hardware Enforcement

Cisco documents storm control as measuring traffic over 1-second intervals. The switch monitors the rate of the configured traffic type and takes action when the rate exceeds the configured threshold.

The measurement interval and enforcement behavior can vary by switch platform"

- **Interval-based suppression:** Some implementations block the affected traffic class for a subsequent measurement interval after detecting an excessive rate. Forwarding resumes when the relevant traffic measurement falls below the applicable threshold.
- **Hardware policing:** Other implementations use ASIC rate limiters to drop **excess packets**, rather than blocking every packet in the affected traffic class. A token bucket can accommodate short bursts before sustained excess traffic is discarded.

Consequently, `storm-control broadcast level 10` should **not** automatically be interpreted as a precise, universally implemented 10% traffic shaper. Depending on the platform, forwarded traffic can be bursty, and the actual suppression rate can differ from the configured value.

Cisco also notes that hardware granularity and frame-size differences can cause percentage thresholds to be approximations.

---

## Rising and Falling Thresholds

Storm Control configuration involves two distinct thresholds:

- **Rising threshold (upper):** The traffic level at which storm control activates.
- **Falling threshold (lower):** The level below which the switch permits normal forwarding again after suppression is triggered.

The lower threshold must be **less than or equal to** the upper threshold.

For example, with a **15% rising** and **10% falling** threshold:

| Measured broadcast rate | Suppression state |
|---|---|
| 8% | Normal forwarding |
| Increases to 16% | Rising threshold crossed, suppression activates |
| Decreases to 12% | Still suppressed, because the rate has not fallen below 10% |
| Decreases to 9% | Falling threshold crossed, normal forwarding resumes |

This is the threshold-state model. The exact number of frames forwarded while suppression is active depends on whether the hardware blocks the class over measurement intervals or polices only the excess frames.

For example:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# storm-control broadcast level 15.00 10.00
```

- Storm control starts suppressing broadcasts when the measured rate reaches approximately **15%** of the interface bandwidth.
- The interface's broadcast suppression state clears after the rate falls below **10%**.
- This separation provides **hysteresis**, reducing repeated transitions when the rate fluctuates near one threshold.

If the falling threshold is omitted, on platforms supporting this syntax it **defaults to the rising threshold**:

```text
SW1(config-if)# storm-control broadcast level 15.00
```

Equivalent threshold interpretation: **rising = 15%, falling = 15%**.

Some platforms or releases support only a single rising threshold. For example, Cisco documents that Catalyst C9550 storm-control configuration does not support falling thresholds.

### Special Values

For percentage thresholds:

| Value | Meaning |
|---|---|
| `0.00` | Suppresses the selected traffic type. |
| `100.00` | Places no effective percentage-based storm-control limit on the selected traffic type. |
| Any value between 0 and 100 | Applies an intermediate suppression threshold. |

---

## Threshold Units

On supported Catalyst platforms, a storm-control limit can be configured in three ways.

### 1. Percentage of Interface Bandwidth

```text
SW1(config-if)# storm-control broadcast level 1.00 0.50
```

This sets broadcast suppression at **1% rising** and **0.5% falling** of the port bandwidth.

If the interface operates at **1 Gbps**:

- Rising threshold: `1 Gbps x 0.01 = 10 Mbps`.
- Falling threshold: `1 Gbps x 0.005 = 5 Mbps`.

If the same percentage is used on a **10 Gbps** interface, the respective values are approximately **100 Mbps** and **50 Mbps**. Thus, percentage-based limits scale with interface bandwidth.

### 2. Bits per Second (bps)

```text
SW1(config-if)# storm-control broadcast level bps 10000000 5000000
```

This specifies:

- Rising: **10,000,000 bits per second (10 Mbps)**.
- Falling: **5,000,000 bits per second (5 Mbps)**.

Unlike a percentage threshold, the configured bps rate does not automatically become ten times larger when used on a ten-times-faster link.

Where supported, metric suffixes can shorten the command:

```text
SW1(config-if)# storm-control broadcast level bps 10m 5m
```

### 3. Packets per Second (pps)

```text
SW1(config-if)# storm-control broadcast level pps 2000 1000
```

This specifies:

- Rising: **2,000 packets per second**.
- Falling: **1,000 packets per second**.

The number of packets is considered regardless of packet length. This can be useful for a flood of small frames that consumes relatively little bandwidth but produces a high packet-processing rate.

Where supported, metric suffixes can also be used:

```text
SW1(config-if)# storm-control broadcast level pps 2k 1k
```

### Choosing a Unit

| Method | Advantage | Consideration |
|---|---|---|
| **Percentage** | Easy to express as a fraction of port capacity. | Absolute permitted traffic increases with link speed. |
| **bps** | Fixed throughput threshold regardless of port speed. | Frame rate still depends on packet size. |
| **pps** | Measures frame rate, useful for small-packet storms. | A given packet rate represents different bandwidth for different packet sizes. |

Supported modes, permitted combinations, maximum values, and threshold granularity vary by hardware and release. For example, Cisco documents that a Catalyst C9350 stack cannot mix bandwidth-based and pps-based storm-control measurement modes across the stack.

---

## Storm-Control Actions

The main IOS XE actions are configured per interface with:

```text
SW1(config-if)# storm-control action {shutdown | trap}
```

### Default Action: Suppress Traffic

With no explicit action configured, the normal behavior is to **suppress the offending traffic type** and not generate an SNMP storm-control trap.

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# storm-control broadcast level 1.00
```

If the broadcast threshold is exceeded:

- Broadcast traffic is suppressed according to the hardware's enforcement mechanism.
- Other enabled/disabled traffic classes are handled by their respective configurations.
- The interface is **not** error-disabled merely because the default storm-control suppression was triggered.

### `storm-control action trap`

```text
SW1(config-if)# storm-control action trap
```

When storm control detects a threshold violation, the switch generates an **SNMP trap** for monitoring.

- The interface remains operational.
- Traffic suppression still occurs.
- SNMP transport, destinations, and management-system configuration must also be in place for the notification to be received.

### `storm-control action shutdown`

```text
SW1(config-if)# storm-control action shutdown
```

When a configured traffic class crosses its threshold, the interface is placed in the **error-disabled** state.

This is more disruptive than ordinary suppression:

- **All** traffic through the interface stops, not only broadcasts or multicasts.
- The link remains unavailable until the error-disabled condition is cleared or an enabled recovery timer reactivates the port.
- This can disable a trunk or an access port even if the excess traffic was brief.

The shutdown and trap capabilities are independent; both can be configured. The shutdown action remains an interface-wide consequence even if only one storm-control traffic class exceeded its threshold.

### Recovering an Error-Disabled Interface

After correcting the underlying cause, manually reset the interface:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# shutdown
SW1(config-if)# no shutdown
```

Alternatively, IOS XE can automatically retry recovery from a storm-control error-disable event:

```text
SW1(config)# errdisable recovery cause storm-control
SW1(config)# errdisable recovery interval 300
```

This permits automatic recovery after the configured timer, typically **300 seconds in this example**. If the storm remains, the interface can become error-disabled again.

Automatic recovery is **not enabled by default** for this cause on Catalyst IOS XE platforms. It is a design decision: restarting the port repeatedly can reintroduce a storm.

---

## Configuration Examples

### Example 1: Access Port with Percentage Thresholds

Limit broadcasts to 1%, multicasts to 10%, and unknown unicasts to 5% on a host-facing interface. Generate a trap instead of shutting down the port.

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# storm-control broadcast level 1.00 0.50
SW1(config-if)# storm-control multicast level 10.00 5.00
SW1(config-if)# storm-control unknown-unicast level 5.00 2.00
SW1(config-if)# storm-control action trap
```

### Example 2: Limit Broadcasts by Packet Rate

```text
SW1(config)# interface GigabitEthernet1/0/2
SW1(config-if)# storm-control broadcast level pps 2000 1000
```

### Example 3: Multicast Threshold with Shutdown Action

```text
SW1(config)# interface GigabitEthernet1/0/3
SW1(config-if)# storm-control multicast level 20.00 10.00
SW1(config-if)# storm-control action shutdown
```

A multicast storm exceeding the rising threshold error-disables **Gi1/0/3**, interrupting all traffic on that port until it is recovered.

### Example 4: Trunk Port

```text
SW1(config)# interface TenGigabitEthernet1/1/1
SW1(config-if)# switchport mode trunk
SW1(config-if)# storm-control broadcast level 5.00 2.00
```

The configured threshold applies to monitored **broadcast traffic received on the trunk interface**. It is not a separate 5% allowance for each allowed VLAN.

On an inter-switch trunk, aggressive thresholds or the `shutdown` action can cause a widespread outage. Normal broadcast and multicast load may be substantially higher than on a host-facing port.

### Example 5: EtherChannel

On Catalyst switches that support storm control on EtherChannels, configure the logical port channel:

```text
SW1(config)# interface Port-channel1
SW1(config-if)# storm-control broadcast level 5.00 2.00
```

The settings **propagate to the member physical interfaces**. Do not assume that the configured percentage represents a single shared threshold across the aggregate EtherChannel bandwidth. Evaluate how the particular switch enforces the threshold on member interfaces, especially if member speeds differ or traffic is unevenly distributed by the EtherChannel hash.

---

## Key Points

1. Storm control is **ingress, interface-based** traffic suppression, not a per-VLAN or per-host ACL.
2. Broadcast, multicast, and applicable unicast are configured as **separate traffic classes**.
3. Thresholds may use **percentage, bps, or pps**, depending on switch capabilities.
4. When configured, the **rising threshold** starts suppression; the **falling threshold** controls restoration. If omitted, the lower threshold usually equals the upper threshold on platforms that support two levels.
5. With no explicit action, storm control normally **suppresses the offending traffic** without error-disabling the interface or generating an SNMP trap.
6. `storm-control action trap` provides notification; `storm-control action shutdown` **error-disables the entire interface**.
7. Multicast storm control can disrupt **OSPF and other legitimate multicast traffic**, even when STP BPDUs and CDP are exempt.
8. On Catalyst 9300, the `unicast` option can affect both known and unknown unicast, and Cisco warns against combining it with `unknown-unicast` on the same interface.
9. Storm control on a Catalyst **port channel propagates to member interfaces**; verify whether enforcement is per member rather than assuming a single aggregate policer.
10. The exact behavior of traffic suppression, rate granularity, supported syntax, and available thresholds depends on the **ASIC, switch model, and software release**.

---

## Cisco Documentation

- [Cisco Catalyst 9300: Port-Based Traffic Control (IOS XE 26.x)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/26-x/configuration_guide/sec/b_26x_sec_9300_cg/configuring_port_based_traffic_control.html)
- [Cisco Catalyst 9300: Security Command Reference, `storm-control` (IOS XE 17.3)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-3/command_reference/b_173_9300_cr/security_commands.html)
- [Cisco IOS XE: Flow Control and Storm Control Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr2-fwd/flowcontrol-stormcontrol/flow-control-and-storm-control-configuration-guide/m-storm-control.html)
- [Cisco Catalyst 9300: Interface and Hardware Command Reference, Error-Disable Recovery](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-15/command_reference/b_1715_9300_cr/interface_and_hardware_commands.html)
