# 4.2.a (vi) Port Security

**Port Security** is a Layer 2 switch security feature that controls which **source MAC addresses** can send traffic into a switch interface. It can limit the number of allowed MAC addresses, restrict the interface to specified addresses, and take action when an unauthorized source MAC address appears.

Port Security is commonly used on access ports to prevent users from connecting additional devices or moving an authorized device to another secure switch port.

For example:

```text
                   SW1
              +-----------+
              |           |
              +-----------+
                 Gi1/0/1
                    |
             +-------------+
             |   PC1       |
             | MAC: AAAA   |
             +-------------+

Port Security on Gi1/0/1:
  Maximum secure MAC addresses: 1
  Learned secure MAC: AAAA
  Violation mode: shutdown
```

After SW1 learns `AAAA` as the secure MAC address on `Gi1/0/1`, a frame arriving on that interface with source MAC address `BBBB` causes a **Port Security violation**. With the default violation mode, the interface becomes **error-disabled**.

Port Security examines the **source MAC address in the Ethernet header**. It does not validate source IP addresses, ARP sender information, or DHCP bindings.

---

## Port Security Operation

When Port Security is enabled on an interface, the switch maintains a set of **secure MAC addresses** for that interface. The allowed addresses can be manually configured or learned automatically.

For an incoming Ethernet frame, the switch evaluates its source MAC address:

1. **Known secure MAC:** The source MAC address is already authorized on this interface and VLAN. The frame passes Port Security and is subject to other applicable forwarding and security checks.
2. **New MAC, capacity available:** The switch can learn the source MAC as a new secure address, subject to the configured learning mode.
3. **New MAC, maximum reached:** The source MAC is not authorized, so a **security violation** occurs.
4. **MAC secured on another port:** A source MAC address already secured on another interface in the same VLAN also causes a **security violation**.

### Maximum Number of Secure MAC Addresses

The maximum is configured per interface:

```text
SW1(config-if)# switchport port-security maximum 2
```

This allows up to **two secure MAC addresses** on the interface. It is not a limit of two frames, hosts authenticated by IP, or destinations.

For example, with a maximum of 2:

| Incoming source MAC | Secure addresses before frame | Result |
|---|---|---|
| `AAAA` | None | Learn `AAAA`; forward |
| `BBBB` | `AAAA` | Learn `BBBB`; forward |
| `AAAA` | `AAAA`, `BBBB` | Already authorized; forward |
| `CCCC` | `AAAA`, `BBBB` | **Violation:** maximum already reached |

A **full secure MAC address table is not itself a violation**. The violation occurs when an unauthorized source attempts to use the port after the limit is reached, or when a secure MAC address appears on another secure port in the same VLAN.

If you attempt to reduce the configured maximum below the number of secure addresses already configured or learned, the switch rejects the command.

### MAC Address Moves

Port Security also detects an attempt to use an address already secured on another switch interface in the same VLAN.

```text
   Host A (MAC AAAA)
          |
       Gi1/0/1        Gi1/0/2
          |              |
          +-----SW1------+ 

  AAAA is secured on Gi1/0/1.
  An incoming frame from AAAA on Gi1/0/2
  causes a violation if Gi1/0/2 is also secure.
```

This can happen when a user moves a device between secure ports, a duplicate or spoofed MAC appears, or an unexpected Layer 2 connection introduces the same source MAC on a different interface.

---

## Default Settings

Port Security is **disabled by default**. When it is enabled without other options, its defaults are:

| Setting | Default |
|---|---|
| Port Security | Disabled |
| Maximum secure MAC addresses | **1** per interface |
| MAC address learning | Dynamic secure learning |
| Sticky learning | Disabled |
| Violation mode | **shutdown** |
| Secure MAC aging | Disabled (`0` minutes) |
| Aging type | `absolute` (if aging is enabled) |
| Aging of statically configured secure MACs | Disabled |

Thus, this simple configuration:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport port-security
```

allows **one dynamically learned secure source MAC address**. A violation places the interface into the **error-disabled** state.

The interface must be explicitly configured as an **access** or **trunk** port before enabling Port Security. An interface left in a dynamic switchport mode cannot be secured.

---

## Types of Secure MAC Addresses

Port Security supports three types of secure MAC addresses:

| Type | How it is established | Added to running configuration? | Preserved after a reload? |
|---|---|---|---|
| **Static** | Administrator configures a specific MAC | Yes | Yes, if configuration is saved |
| **Dynamic** | Switch learns a source MAC | No | No |
| **Sticky** | Switch learns or administrator specifies a sticky secure MAC | Yes | Yes, if configuration is saved |

All three types **count toward the interface's configured maximum**.

### 1. Static Secure MAC Addresses

A **static secure MAC address** is manually specified:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security mac-address 0011.2233.4455
```

The port authorizes source MAC `0011.2233.4455`. With the default maximum of 1, no additional MAC address can be learned.

To permit one additional MAC address dynamically:

```text
SW1(config-if)# switchport port-security maximum 2
```

Now one of the two available secure MAC slots is occupied by the configured static address, and the other can be learned dynamically.

To remove the configured static address:

```text
SW1(config-if)# no switchport port-security mac-address 0011.2233.4455
```

A static secure MAC is not the same as an ordinary static MAC address-table entry configured with `mac address-table static`. Port Security uses the `switchport port-security mac-address` configuration to define authorized **source** MAC addresses.

### 2. Dynamic Secure MAC Addresses

With Port Security enabled, the switch can automatically learn an incoming source MAC address:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport port-security
```

Suppose the first host sends a frame with source MAC `0011.2233.4455`. SW1 learns it as a **dynamic secure MAC address**.

- The address is stored in the secure MAC address table.
- It is **not added to the running configuration**.
- The address is **lost upon a reload**.
- It is also removed when the relevant link goes down, and it can be removed by configured secure address aging.

If the address disappears, the switch can learn another MAC address when a new frame arrives, provided capacity is available.

### 3. Sticky Secure MAC Addresses

**Sticky learning** combines automatic MAC learning with configuration persistence.

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security mac-address sticky
```

When `0011.2233.4455` sends a frame, the switch learns it and automatically adds a line to the **running configuration**:

```text
interface GigabitEthernet1/0/1
 switchport mode access
 switchport port-security
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 0011.2233.4455
```

The last line was **automatically generated** through sticky learning.

To retain the learned sticky addresses across a reload:

```text
SW1# copy running-config startup-config
```

Sticky addresses are not automatically written to **startup-config**. If the configuration has not been saved, they are lost after a reload and can be learned again.

#### Converting Existing Dynamic Addresses to Sticky

Enabling sticky learning also converts **already learned dynamic secure addresses** on that port into sticky secure addresses and adds them to the running configuration.

```text
SW1(config-if)# switchport port-security mac-address sticky
```

Static secure MAC addresses remain static; they are not converted to sticky addresses.

#### Manually Configuring a Sticky Address

Sticky addresses can also be specified manually, after enabling sticky learning:

```text
SW1(config-if)# switchport port-security mac-address sticky
SW1(config-if)# switchport port-security mac-address sticky 0011.2233.4455
```

#### Disabling Sticky Learning

```text
SW1(config-if)# no switchport port-security mac-address sticky
```

With Port Security still enabled, this **converts sticky secure addresses into dynamic secure addresses** and removes the learned sticky address configuration lines. The addresses are no longer preserved in the running configuration.

To remove an individual sticky address without disabling sticky learning:

```text
SW1(config-if)# no switchport port-security mac-address sticky 0011.2233.4455
```

For a PC replacement, check both the running configuration and the startup configuration. An old saved sticky address can reappear after a reload if it has not been removed from the saved configuration.

---

## Port Security Violation Modes

The **violation mode** determines the switch's response when a security violation occurs:

```text
SW1(config-if)# switchport port-security violation {protect | restrict | shutdown | shutdown vlan}
```

The principal modes are **protect**, **restrict**, and **shutdown**. A fourth option, **shutdown vlan**, can disable an affected VLAN rather than the entire interface.

| Mode | Drop offending frames | Syslog notification | Violation counter increments | Interface error-disabled |
|---|---|---|---|---|
| **protect** | Yes | No | No | No |
| **restrict** | Yes | Yes | Yes | No |
| **shutdown** (default) | Yes | Yes | Yes | **Yes** |
| **shutdown vlan** | Yes, for affected VLAN | Yes | Yes | Only the affected VLAN is disabled |

With **restrict**, Cisco also documents an SNMP trap notification. Delivery requires SNMP notification configuration. With **protect**, violations are silently discarded.

### Protect

```text
SW1(config-if)# switchport port-security violation protect
```

When an unauthorized source MAC address triggers a violation:

- The switch **drops frames from the unauthorized source**.
- Previously authorized MAC addresses can continue communicating.
- The interface stays operational.
- **No security violation counter increment or syslog notification** is generated.

For example, with `maximum 1`, once MAC `AAAA` is learned, frames from `BBBB` are discarded without disabling the port.

### Restrict

```text
SW1(config-if)# switchport port-security violation restrict
```

The switch drops offending frames and reports the violation:

- Authorized MAC addresses continue to work.
- Unauthorized source frames are discarded.
- The **security violation counter increments**.
- A **syslog message** is generated, and an **SNMP notification** can be sent.
- The port remains up.

The key difference between `protect` and `restrict` is **visibility**: both drop unauthorized source traffic, but `restrict` reports violations.

### Shutdown

```text
SW1(config-if)# switchport port-security violation shutdown
```

This is the **default violation mode**. A violation places the **entire interface** into the **error-disabled** state.

```text
Host A (authorized) ---> Gi1/0/1 ---> SW1
Host B (unauthorized source) appears
                             |
                   Port Security violation
                             |
                    Gi1/0/1 error-disabled
```

- The entire interface stops forwarding, including traffic from previously authorized MAC addresses.
- The violation counter increments.
- The switch logs the violation.
- The interface does **not automatically recover** unless error-disable recovery is configured or the administrator intervenes.

For example, a violating interface can appear as:

```text
SW1# show interfaces status err-disabled
Port         Name               Status         Reason
Gi1/0/1                        err-disabled   psecure-violation
```

The normal solution is to correct the cause of the violation and then re-enable the interface.

### Shutdown VLAN

```text
SW1(config-if)# switchport port-security violation shutdown vlan
```

With this mode, a violation **disables the affected VLAN on the port** instead of error-disabling the entire physical interface.

This is particularly useful when a trunk carries multiple VLANs. For example, if a violation occurs in VLAN 10 on a secure trunk carrying VLANs 10 and 20, **VLAN 10 is disabled on that port**, while VLAN 20 can continue operating.

---

## Recovering From a Port Security Violation

### Manual Recovery

After correcting the problem, manually recover an error-disabled interface:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# shutdown
SW1(config-if)# no shutdown
```

This resets the error-disabled condition. If an unauthorized MAC address is still generating traffic, the interface can immediately become error-disabled again.

### Automatic Error-Disable Recovery

To automatically retry an interface disabled by Port Security:

```text
SW1(config)# errdisable recovery cause psecure-violation
```

The default recovery interval is **300 seconds (5 minutes)**. To set it to 120 seconds:

```text
SW1(config)# errdisable recovery interval 120
```

Verify the recovery settings:

```text
SW1# show errdisable recovery
```

**Automatic recovery is disabled by default.** Specifying an interval alone does not enable recovery for a particular cause; use `errdisable recovery cause psecure-violation`.

---

## Port Security Aging

**Port Security aging** automatically removes secure MAC address entries so that a different device can eventually use a port without manually clearing an old secure MAC address.

Aging is **disabled by default**. The aging time is configured in **minutes**:

```text
SW1(config-if)# switchport port-security aging time 10
```

This configures an aging period of **10 minutes**.

Two aging modes are supported: **absolute** and **inactivity**.

### Absolute Aging

```text
SW1(config-if)# switchport port-security aging type absolute
SW1(config-if)# switchport port-security aging time 10
```

With **absolute aging**, an eligible secure MAC address is removed when the configured time expires **regardless of whether it has been sending traffic**.

Example:

```text
00:00  MAC AAAA is learned.
00:05  MAC AAAA continues sending frames.
00:10  MAC AAAA ages out anyway.
```

If the device continues transmitting after the address ages out, the switch may immediately learn the address again if capacity is available.

### Inactivity Aging

```text
SW1(config-if)# switchport port-security aging type inactivity
SW1(config-if)# switchport port-security aging time 10
```

With **inactivity aging**, the secure MAC is removed only when it has **not sent traffic for the configured time**.

Example:

```text
00:00  MAC AAAA is learned.
00:05  MAC AAAA sends a frame; inactivity timer refreshes.
00:15  MAC AAAA ages out if no more traffic arrives.
```

Inactivity aging is usually more suitable than absolute aging when a secure address should remain authorized while its device is active.

### Aging of Static Secure Addresses

By default, static secure MAC addresses are **not subject to Port Security aging**. To enable aging for statically configured secure MAC addresses:

```text
SW1(config-if)# switchport port-security aging static
```

This works together with the configured `aging time` and `aging type`.

### Sticky MAC Address Aging

**Sticky secure MAC addresses do not age out using Port Security aging.** To replace a sticky address, remove the corresponding configuration or disable sticky learning as appropriate.

### Port Security Aging Versus Normal MAC Table Aging

Do not confuse the two mechanisms:

| Mechanism | Command | Default | Purpose |
|---|---|---|---|
| **Ordinary dynamic MAC table aging** | `mac address-table aging-time` | **300 seconds** | Remove inactive ordinary dynamic forwarding entries |
| **Port Security aging** | `switchport port-security aging time` | **0 minutes (disabled)** | Remove eligible *secure* MAC addresses from a secure port |

Changing the ordinary MAC address table aging timer does **not** replace the Port Security aging configuration.

---

## Configuring Port Security on an Access Port

The following configuration permits **two dynamically learned MAC addresses** on access port `Gi1/0/1`, generates notifications for violations, and ages learned secure MAC addresses after **30 minutes of inactivity**:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 2
SW1(config-if)# switchport port-security violation restrict
SW1(config-if)# switchport port-security aging time 30
SW1(config-if)# switchport port-security aging type inactivity
```

Operation:

1. The first source MAC address is learned and allowed.
2. A second source MAC address is learned and allowed.
3. A third source MAC address generates a violation and its frames are dropped.
4. The interface stays operational because `restrict` is configured.
5. An eligible secure address is removed after 30 minutes without sending traffic, making capacity available for another device.

### Secure a Port to One Known Device

When a known computer must be the only permitted device:

```text
SW1(config)# interface GigabitEthernet1/0/2
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 1
SW1(config-if)# switchport port-security mac-address 0011.2233.4455
SW1(config-if)# switchport port-security violation shutdown
```

Only the specified source MAC address is allowed. There is no dynamic learning capacity remaining.

### Secure a Port Using Sticky Learning

```text
SW1(config)# interface GigabitEthernet1/0/3
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 1
SW1(config-if)# switchport port-security mac-address sticky
SW1(config-if)# switchport port-security violation restrict
!
SW1# copy running-config startup-config
```

The first learned source MAC becomes sticky. To preserve it across reloads, save the running configuration **after the address has been learned**.

---

## Port Security with a Voice VLAN

An access switchport connecting a Cisco IP phone with a PC connected behind the phone can learn **two distinct source MAC addresses**:

```text
                SW1
             Gi1/0/10
                 |
            +----------+
            | IP Phone |  Voice VLAN 20
            +----------+
                 |
                PC        Data VLAN 10
```

- The **phone MAC address** is associated with the **voice VLAN**.
- The **PC MAC address** is associated with the **access VLAN**.

The default maximum of 1 is not sufficient for this arrangement.

### Configuration

```text
SW1(config)# interface GigabitEthernet1/0/10
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# switchport voice vlan 20
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 2
SW1(config-if)# switchport port-security maximum 1 vlan access
SW1(config-if)# switchport port-security maximum 1 vlan voice
SW1(config-if)# switchport port-security violation restrict
```

This configuration allows:

- **1 secure MAC** in the access VLAN.
- **1 secure MAC** in the voice VLAN.
- **2 secure MACs in total** on the interface.

The `vlan access` and `vlan voice` keywords select the configured access VLAN and voice VLAN, respectively. They do not require you to repeat their VLAN numbers.

For a port with multiple hosts behind a phone, increase the appropriate limits to accommodate the expected MAC addresses.

A fixed phone MAC address can be secured to the voice VLAN explicitly:

```text
SW1(config-if)# switchport port-security mac-address 00aa.bbcc.ddee vlan voice
```

Or sticky learning can assign the phone's MAC address to the voice VLAN:

```text
SW1(config-if)# switchport port-security mac-address sticky
SW1(config-if)# switchport port-security mac-address sticky 00aa.bbcc.ddee vlan voice
```

---

## Port Security on Trunk Ports

Port Security can also be enabled on a **statically configured trunk port**. Secure MAC addresses are associated with their VLANs, and the switch can enforce an **overall interface maximum** and **per-VLAN maximums**.

Example requirements:

- Secure trunk `Gi1/0/24` carries VLANs 10 and 20.
- Allow up to 2 secure MAC addresses in VLAN 10.
- Allow up to 3 secure MAC addresses in VLAN 20.
- Allow a total of up to 5 secure MAC addresses.
- Disable only the violating VLAN if a violation occurs.

```text
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,20
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 5
SW1(config-if)# switchport port-security maximum 2 vlan 10
SW1(config-if)# switchport port-security maximum 3 vlan 20
SW1(config-if)# switchport port-security violation shutdown vlan
```

A frame with an unrecognized source MAC in VLAN 10 causes a violation when VLAN 10 already has 2 secure MACs, even if fewer than 5 have been learned on the interface as a whole.

An explicitly configured secure MAC address on a trunk can include its VLAN:

```text
SW1(config-if)# switchport port-security mac-address 0011.2233.4455 vlan 10
```

Likewise, a manually configured sticky MAC can include its VLAN:

```text
SW1(config-if)# switchport port-security mac-address sticky
SW1(config-if)# switchport port-security mac-address sticky 0011.2233.4466 vlan 20
```

When no VLAN is specified for an explicitly configured secure MAC on a trunk, it is associated with the **native VLAN**.

`protect` is not a good choice with per-VLAN limits on trunks: when one VLAN reaches its maximum, it can prevent additional MAC learning even if the interface has capacity available elsewhere. Prefer an appropriate reported violation mode.

---

## Verification Commands

### Overall Port Security Summary

```text
SW1# show port-security
```

This summarizes secure interfaces, including:

- **MaxSecureAddr:** Configured MAC limit.
- **CurrentAddr:** Number of currently secured MAC addresses.
- **SecurityViolation:** Number of recorded violations.
- **Security Action:** Configured violation mode.

### Detailed Interface Status

```text
SW1# show port-security interface GigabitEthernet1/0/1
```

Representative output:

```text
Port Security              : Enabled
Port Status                : Secure-up
Violation Mode             : Restrict
Aging Time                 : 30 mins
Aging Type                 : Inactivity
SecureStatic Address Aging : Disabled
Maximum MAC Addresses      : 2
Total MAC Addresses        : 2
Configured MAC Addresses   : 0
Sticky MAC Addresses       : 0
Last Source Address:Vlan   : 00aa.bbcc.ddee:10
Security Violation Count   : 4
```

Interpretation:

- Port Security is operational and set to `restrict`.
- The maximum of 2 secure MAC addresses has been reached.
- Both addresses were learned dynamically because the configured and sticky counts are 0.
- A total of 4 violations has been recorded.
- The interface remains up in `restrict` mode.

The **Last Source Address:Vlan** field can help identify the source involved in recent traffic or a violation. It is not a complete list of authorized secure MAC addresses.

### Show Secure MAC Addresses

```text
SW1# show port-security address
```

Representative output:

```text
Secure Mac Address Table
------------------------------------------------------------------------
Vlan    Mac Address       Type                 Ports       Remaining Age
----    -----------       ----                 -----       -------------
  10    0011.2233.4455    SecureDynamic        Gi1/0/1     26 mins
  10    0011.2233.4466    SecureDynamic        Gi1/0/1     28 mins
------------------------------------------------------------------------
```

Common secure address types in output include:

- `SecureDynamic`: Dynamically learned secure address.
- `SecureConfigured`: Manually configured static secure address.
- `SecureSticky`: Sticky secure address.

To narrow the display to an interface:

```text
SW1# show port-security interface GigabitEthernet1/0/1 address
```

### Show Per-VLAN Limits

```text
SW1# show port-security interface GigabitEthernet1/0/24 vlan
```

This displays secure MAC address counts and configured maxima for individual VLANs on the interface.

### Verify Interface State and Configuration

```text
SW1# show interfaces status
SW1# show interfaces status err-disabled
SW1# show interfaces GigabitEthernet1/0/1 switchport
SW1# show running-config interface GigabitEthernet1/0/1
SW1# show mac address-table interface GigabitEthernet1/0/1
SW1# show errdisable recovery
SW1# show logging
```

Use `show port-security address` to verify **secure authorization**, not just `show mac address-table`. Ordinary MAC address learning and Port Security's authorized-address state are distinct concepts.

---

## Troubleshooting Port Security

### 1. Port Becomes Error-Disabled After a Host Is Connected

**Likely cause:** The maximum secure MAC count has been reached, or the source MAC address is already secured on another port.

Check:

```text
SW1# show interfaces status err-disabled
SW1# show port-security interface GigabitEthernet1/0/1
SW1# show port-security address
SW1# show logging
```

Correct the MAC address configuration or remove the unauthorized device, then recover the port:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# shutdown
SW1(config-if)# no shutdown
```

### 2. Unauthorized Host Cannot Communicate, but Port Remains Up

**Likely cause:** Violation mode is `protect` or `restrict`.

- With `protect`, the unauthorized frames are silently dropped and the violation counter does not increment.
- With `restrict`, the unauthorized frames are dropped and the violation counter increments.

Check the current secure address list and the configured maximum before assuming there is a general connectivity problem.

### 3. Port Security Appears Not to Learn New MAC Addresses

Check whether:

- The maximum number of secure addresses is already reached.
- A static or sticky address is already occupying the available slot.
- The address is secured on another interface in the same VLAN.
- The interface is operationally down or error-disabled.

To clear dynamically learned secure MAC addresses on an interface:

```text
SW1# clear port-security dynamic interface GigabitEthernet1/0/1
```

Do not expect this command to remove static or saved sticky configuration lines.

### 4. A New PC Cannot Replace an Old PC

**Likely cause:** The previous PC's MAC address is still a static or sticky secure address.

For a manually configured static address:

```text
SW1(config-if)# no switchport port-security mac-address 0011.2233.4455
```

For a sticky address:

```text
SW1(config-if)# no switchport port-security mac-address sticky 0011.2233.4455
```

Check that the old address is gone, and save the revised configuration so an old sticky entry is not restored on reload.

### 5. Phone Works but the PC Behind It Does Not

**Likely cause:** The port's maximum secure MAC count is still 1, or the access VLAN has insufficient secure MAC capacity.

Verify the phone's voice VLAN, the PC's access VLAN, and both overall and per-VLAN limits:

```text
SW1# show port-security interface GigabitEthernet1/0/10
SW1# show port-security interface GigabitEthernet1/0/10 vlan
SW1# show port-security address
```

For one phone and one PC, allow at least two secure MAC addresses in total, with suitable per-VLAN allocation.

### 6. A Port Becomes Error-Disabled Again Immediately After Recovery

**Likely cause:** The violating source is still present, or the original authorized MAC address remains secured while a different device is transmitting.

A `shutdown` / `no shutdown` sequence clears the error-disabled state. It does **not** automatically fix the secure MAC address list or the cause of the violation.

---

## Port Security and Other Layer 2 Security Features

Port Security protects against **unauthorized source MAC addresses**, not every kind of address spoofing.

| Feature | What it validates or controls |
|---|---|
| **Port Security** | Source MAC address of incoming Ethernet frames; number of secure MACs on a port |
| [**IP Source Guard**](ip-source-guard.md) | Source IP address of incoming IP packets, optionally source MAC as well |
| [**Dynamic ARP Inspection**](dynamic-arp-inspection.md) | ARP sender IP-to-MAC information and optional ARP field checks |
| [**DHCP Snooping**](dhcp-snooping/overview.md) | DHCP messages, trust boundaries, and DHCP-assigned IP/MAC/port bindings |
| [**Storm Control**](storm-control.md) | Excessive broadcast, multicast, or unicast traffic rates |
| [**VACL and PACL**](vacl-pacl.md) | Traffic permitted or denied by VLAN or port ACL policies |

These features are **independent** and can complement each other.

A host that successfully impersonates an already authorized source MAC address is not automatically identified by Port Security. Port Security also does not prevent an authorized MAC address from attempting an IP spoofing or ARP spoofing attack. IP Source Guard and Dynamic ARP Inspection address those separate problems.

---

## Restrictions and Design Considerations

- Port Security is configured on **Layer 2 switchports** in **static access** or **static trunk** mode, not on routed interfaces or ports negotiating dynamic access/trunk mode.
- It is not supported on **EtherChannel interfaces** or **private VLAN ports** on Catalyst 9300 switches.
- A secure port can be a **SPAN source**, but not a **SPAN destination**.
- Be careful when combining Port Security with **802.1X**, since both control client access and have operational interactions.
- Account for phones, PCs behind phones, virtual machines, and other devices that can introduce multiple source MAC addresses on one port.
- Avoid overly restrictive MAC limits on links to downstream switches, since they may carry frames sourced by many different devices.

---

## Key Points

- **Port Security validates incoming Ethernet source MAC addresses**, not IP or ARP bindings.
- Port Security is **disabled by default**. Once enabled, the default maximum is **1 secure MAC address**, with **shutdown** as the default violation mode.
- Secure MAC addresses may be **static**, **dynamic**, or **sticky**.
- **Dynamic** addresses are learned automatically but do not appear in the running configuration.
- **Sticky** learning automatically writes learned secure MAC addresses into the running configuration; save the configuration to retain them after a reload.
- `protect` drops unauthorized frames **without notifications or counter increments**.
- `restrict` drops unauthorized frames **and reports violations**, while leaving the interface up.
- `shutdown` places the **entire interface** in the error-disabled state; `shutdown vlan` disables only the violating VLAN on a port.
- Port Security aging supports **absolute** and **inactivity** modes. It is disabled by default, and **sticky addresses are not aged**.
- Use an overall secure MAC maximum and, when needed, **per-VLAN maxima** for voice VLAN and trunk configurations.
- The primary verification commands are `show port-security`, `show port-security interface`, and `show port-security address`.

---

## References

- [Cisco: Catalyst 9300 Security Configuration Guide, Port Security](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-17/configuration_guide/sec/b_1717_sec_9300_cg/port_security.html)
- [Cisco: IOS Security Command Reference, switchport port-security Commands](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/security/s1/sec-s1-cr-book/sec-cr-s6.html)
- [Cisco: Recover Errdisable Port State on Cisco IOS Platforms](https://www.cisco.com/c/en/us/support/docs/lan-switching/spanning-tree-protocol/69980-errdisable-recovery.html)
