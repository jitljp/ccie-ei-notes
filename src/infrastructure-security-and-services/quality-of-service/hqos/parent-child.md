# Hierarchical Policy Maps and Parent-Child Policies

**Hierarchical Quality of Service (HQoS)** uses nested policy maps to apply QoS mechanisms at multiple levels.

A **parent policy** contains one or more classes, and a **child policy** is attached to a parent class using the `service-policy` command.

The most common design uses:

- **Parent policy:** Shapes traffic to control the aggregate transmission rate.
- **Child policy:** Classifies traffic into queues and determines how the available bandwidth is distributed.

Unlike a flat QoS policy, HQoS allows traffic to be subject to both aggregate and per-class QoS treatment.

---

## Parent and Child Policy Maps

HQoS uses the same MQC components as ordinary QoS:

- `class-map`: Defines traffic classification criteria.
- `policy-map`: Defines QoS actions.
- `service-policy`: Attaches a policy to an interface or nests a policy inside another policy.

The difference is that a hierarchical configuration uses multiple policy maps.

### Parent Policy

The **parent policy** is the top-level policy attached directly to an interface.

In a typical WAN HQoS configuration, it performs aggregate shaping.

For example:

```text
R1(config)# policy-map PARENT
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average 20000000
```

This shapes traffic to 20 Mbps.

Because `class-default` is the only class in the parent policy, all traffic handled by the policy is subject to the shaper.

### Child Policy

The **child policy** is nested inside a class of the parent policy.

It typically performs classification, queuing, and scheduling.

For example, first configure the class maps:

```text
R1(config)# class-map match-all VOICE
R1(config-cmap)# match dscp ef
R1(config-cmap)# exit
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match dscp af31
```

Configure the child policy:

```text
R1(config)# policy-map CHILD
R1(config-pmap)# class VOICE
R1(config-pmap-c)# priority percent 10
R1(config-pmap-c)# exit
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth remaining percent 60
```

The child policy defines:

- VOICE: Strict priority scheduling.
- BUSINESS: A weighted share of excess bandwidth.
- `class-default`: Treatment for traffic not matching the named classes.

The child operates within the bandwidth available from the parent.

---

## Nesting a Child Policy

The command used to attach a child policy is:

```text
R1(config-pmap-c)# service-policy <policy-map>
```

This command is entered in **policy-map class configuration mode**.

For example:

```text
R1(config)# policy-map PARENT
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average 20000000
R1(config-pmap-c)# service-policy CHILD
```

This creates the following hierarchy:

```text
PARENT
|
+-- class-default
    |
    +-- shape average 20000000
    |
    +-- CHILD
        |
        +-- VOICE
        |   +-- priority percent 10
        |
        +-- BUSINESS
        |   +-- bandwidth remaining percent 60
        |
        +-- class-default
```

The parent policy is then applied to an interface:

```text
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output PARENT
```

### Two Uses of `service-policy`

The `service-policy` command has two different uses in HQoS.

| Configuration mode | Command | Purpose |
|---|---|---|
| Interface | `service-policy output PARENT` | Applies the top-level policy to the interface |
| Policy-map class | `service-policy CHILD` | Nests a child policy inside the parent class |

The distinction is important.

**The parent policy is attached directly to the interface. The child policy is attached to a class within the parent policy.**

The child policy does not require a separate interface-level `service-policy` command.

### Removing a Child Policy

To remove the child policy from the parent class:

```text
R1(config)# policy-map PARENT
R1(config-pmap)# class class-default
R1(config-pmap-c)# no service-policy CHILD
```

This removes the parent-child relationship.

It does not delete the CHILD policy-map definition.

---

## Classification at Multiple Levels

One of the most important HQoS concepts is **hierarchical classification**.

Traffic must match the classification criteria at each relevant level before receiving the corresponding QoS treatment.

For a packet to match a particular child class:

1. It must match the parent class containing the child policy.
2. It must match the appropriate class within that child policy.

A child policy therefore processes only traffic belonging to its parent class.

### Example 1: Parent `class-default`

Consider:

```text
policy-map CHILD
 class VOICE
  priority percent 10
 class BUSINESS
  bandwidth remaining percent 60
!
policy-map PARENT
 class class-default
  shape average 20000000
  service-policy CHILD
```

Because the parent contains only `class-default`, all traffic entering the parent policy is eligible for classification by CHILD.

The child classifies traffic as follows:

| DSCP | Parent class | Child class |
|---|---|---|
| EF | `class-default` | VOICE |
| AF31 | `class-default` | BUSINESS |
| CS0 | `class-default` | `class-default` |
| AF41 | `class-default` | `class-default` |

All four packets are subject to the parent's 20 Mbps shaper.

However, their scheduling treatment depends on which child class they match.

### Example 2: Parent User-Defined Class

The parent does not have to use `class-default`.

A parent policy can classify specific traffic and apply a child policy only to that traffic.

Consider the following class maps:

```text
R1(config)# class-map match-any AF3-TRAFFIC
R1(config-cmap)# match dscp af31 af32 af33
R1(config-cmap)# exit
R1(config)# class-map match-all AF31-TRAFFIC
R1(config-cmap)# match dscp af31
R1(config-cmap)# exit
R1(config)# class-map match-all AF32-TRAFFIC
R1(config-cmap)# match dscp af32
```

Configure the child policy:

```text
R1(config)# policy-map AF3-CHILD
R1(config-pmap)# class AF31-TRAFFIC
R1(config-pmap-c)# bandwidth remaining percent 70
R1(config-pmap-c)# exit
R1(config-pmap)# class AF32-TRAFFIC
R1(config-pmap-c)# bandwidth remaining percent 20
```

Configure the parent policy:

```text
R1(config)# policy-map PARENT
R1(config-pmap)# class AF3-TRAFFIC
R1(config-pmap-c)# shape average 10000000
R1(config-pmap-c)# service-policy AF3-CHILD
```

The hierarchy is:

```text
PARENT
|
+-- AF3-TRAFFIC
|   |
|   +-- Shape 10 Mbps
|   |
|   +-- AF3-CHILD
|       |
|       +-- AF31-TRAFFIC
|       +-- AF32-TRAFFIC
|       +-- class-default
|
+-- class-default
```

Consider packets with different DSCP markings.

| Packet DSCP | Parent class | Child class |
|---|---|---|
| AF31 | AF3-TRAFFIC | AF31-TRAFFIC |
| AF32 | AF3-TRAFFIC | AF32-TRAFFIC |
| AF33 | AF3-TRAFFIC | `class-default` |
| EF | `class-default` | Does not enter AF3-CHILD |
| CS0 | `class-default` | Does not enter AF3-CHILD |

Notice that:

- AF33 matches the parent class but not either named child class, so it enters the child's `class-default`.
- EF and CS0 do not match AF3-TRAFFIC, so they do not enter AF3-CHILD.
- EF and CS0 are handled by the parent's own `class-default`, which has no explicit actions in this example.

**A child policy cannot classify traffic that did not match the parent class containing that child policy.**

This is sometimes described as a logical AND relationship between the classification criteria at successive hierarchy levels.

The example assumes an IOS XE platform supporting hierarchical queuing under user-defined parent classes.

---

## The Role of `class-default`

Both parent and child policies can contain a class named `class-default`.

However, they serve different purposes.

### Parent `class-default`

The parent's `class-default` matches traffic that did not match another class in the parent policy.

If no user-defined parent classes exist, it matches all traffic.

It is commonly used to apply one aggregate shaper to all outbound traffic.

### Child `class-default`

The child's `class-default` matches traffic that entered the child policy but did not match one of its named classes.

For example:

```text
PARENT
|
+-- class-default
    |
    +-- Shape 20 Mbps
    |
    +-- CHILD
        |
        +-- VOICE: EF
        +-- BUSINESS: AF31
        +-- class-default: Other traffic
```

The two `class-default` classes are separate classification contexts.

**The child's `class-default` is not the same class as the parent's `class-default`.**

Each operates within its own policy map.

---

## Reusing Child Policies

Cisco IOS XE allows a child policy to be reused under multiple parent policies on supported platforms.

This is useful when different WAN connections require the same traffic scheduling behavior but have different available bandwidth.

For example:

```text
R1(config)# class-map match-all VOICE
R1(config-cmap)# match dscp ef
R1(config-cmap)# exit
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match dscp af31
```

Create a common child policy:

```text
R1(config)# policy-map CHILD-QOS
R1(config-pmap)# class VOICE
R1(config-pmap-c)# priority percent 10
R1(config-pmap-c)# exit
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth remaining percent 60
```

Create two parent policies:

```text
R1(config)# policy-map PARENT-100M
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average 100000000
R1(config-pmap-c)# service-policy CHILD-QOS
```

```text
R1(config)# policy-map PARENT-50M
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average 50000000
R1(config-pmap-c)# service-policy CHILD-QOS
```

Apply them to separate interfaces:

```text
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output PARENT-100M
R1(config-if)# exit
R1(config)# interface GigabitEthernet3
R1(config-if)# service-policy output PARENT-50M
```

### Result

Both parent policies reference the same CHILD-QOS definition.

However, each application creates a separate policy instance.

```text
GigabitEthernet2
|
+-- PARENT-100M
    |
    +-- Shape 100 Mbps
    |
    +-- CHILD-QOS
        +-- VOICE
        +-- BUSINESS
        +-- class-default


GigabitEthernet3
|
+-- PARENT-50M
    |
    +-- Shape 50 Mbps
    |
    +-- CHILD-QOS
        +-- VOICE
        +-- BUSINESS
        +-- class-default
```

The child policy's percentage-based rates are calculated relative to the applicable parent shaping rate.

For example:

| Interface | Parent shape rate | VOICE priority percentage | VOICE priority rate |
|---|---:|---:|---:|
| GigabitEthernet2 | 100 Mbps | 10% | 10 Mbps |
| GigabitEthernet3 | 50 Mbps | 10% | 5 Mbps |

The same child policy produces different configured priority rates because the parents have different shaping rates.

The priority rate is not necessarily an unconditional maximum transmission rate. Its enforcement depends on the platform's priority scheduling and policing implementation.

**Reusing a child policy does not cause separate interfaces to share one common shaper, queue, or policer state.**

Each instantiated hierarchy operates independently.

---

## Multiple Parent Classes

A parent policy can contain multiple user-defined classes.

On supported platforms, each parent class can have its own shaping rate and nested child policy.

For example, consider a provider edge router serving two customers.

```text
                   R1
                    |
                    | Shared physical interface
                    |
              +-----+-----+
              |           |
              v           v
          Customer A   Customer B
           50 Mbps      20 Mbps
```

A conceptual hierarchy is:

```text
PARENT
|
+-- CUSTOMER-A
|   |
|   +-- Shape 50 Mbps
|   |
|   +-- CHILD-QOS
|       +-- VOICE
|       +-- BUSINESS
|       +-- class-default
|
+-- CUSTOMER-B
|   |
|   +-- Shape 20 Mbps
|   |
|   +-- CHILD-QOS
|       +-- VOICE
|       +-- BUSINESS
|       +-- class-default
|
+-- class-default
```

The parent first identifies which customer owns the traffic.

Each customer then receives its own shaping and scheduling treatment.

The same CHILD-QOS definition can be reused, with its percentage-based rates interpreted relative to the applicable parent.

This is a common application of hierarchical scheduling in service-provider and multi-tenant environments.

**The sum of the parent shaping rates can exceed the physical interface bandwidth.**

A shaping rate is an upper limit, not a bandwidth reservation.

For example:

```text
Physical interface = 100 Mbps

Customer A shaper = 80 Mbps
Customer B shaper = 80 Mbps

Combined shaping limits = 160 Mbps
```

Both customers cannot simultaneously transmit at 80 Mbps over a 100 Mbps physical interface.

An additional scheduling level or other resource-sharing mechanism determines how competing traffic shares the physical interface.

---

## Three-Level Hierarchical Policies

Although two-level HQoS is the most common design, some Cisco IOS XE platforms support deeper hierarchies.

For example, Cisco ASR 1000 platforms support up to **three policy levels** in documented hierarchical QoS implementations.

These levels are commonly called:

1. **Parent:** Top-level policy attached to the interface.
2. **Child:** Policy nested within a parent class.
3. **Grandchild:** Policy nested within a child class.

### Example Structure

Consider a router that must:

- Shape the physical interface's aggregate traffic to 500 Mbps.
- Allocate separate shaping rates to customer VLANs.
- Apply application-level queuing within each VLAN.

```text
                       Physical Interface
                              |
                              v
                    +------------------+
                    |  PARENT POLICY   |
                    |                  |
                    | Shape: 500 Mbps  |
                    +------------------+
                              |
                              v
                    +------------------+
                    |   CHILD POLICY   |
                    |                  |
                    | Classify VLANs   |
                    +------------------+
                       |           |
                       v           v
                  VLAN 100      VLAN 200
                  100 Mbps       50 Mbps
                       |           |
                       v           v
                 +----------------------+
                 | GRANDCHILD POLICIES  |
                 |                      |
                 | VOICE / DATA / OTHER |
                 +----------------------+
```

The hierarchy can be configured conceptually as:

```text
policy-map GRANDCHILD
 class VOICE
  priority
  police cir percent 10
 class BUSINESS
  bandwidth remaining percent 30
!
policy-map CHILD
 class VLAN100
  shape average 100000000
  service-policy GRANDCHILD
 class VLAN200
  shape average 50000000
  service-policy GRANDCHILD
!
policy-map PARENT
 class class-default
  shape average 500000000
  service-policy CHILD
```

This example assumes that appropriate class maps exist and the platform supports the required VLAN classification and three-level scheduling.

The resulting rate constraints are:

| Level | Function | Rate |
|---|---|---|
| Parent | Aggregate physical-interface shaping | 500 Mbps |
| Child VLAN100 | VLAN100 shaping | 100 Mbps |
| Child VLAN200 | VLAN200 shaping | 50 Mbps |
| Grandchild | Per-application queuing and policing | Relative to the applicable parent constraints |

A packet belonging to VLAN100 and marked DSCP EF must match all three levels:

1. Parent `class-default`.
2. Child VLAN100.
3. Grandchild VOICE.

Its effective forwarding treatment is determined by the combination of actions at those levels.

### Hierarchy Depth Limitations

Hierarchy depth is platform-dependent.

For example:

- Certain ASR 1000 implementations support three-level policies.
- Other IOS XE platforms support only two levels.
- Some interface types support fewer hierarchical features than others.

The ability to configure an additional `service-policy` command does not guarantee that the target hardware supports the complete hierarchy.

---

## Hierarchical Policing

HQoS is not limited to shaping and queuing.

On supported Cisco IOS XE routers, hierarchical policies can also apply policing at multiple levels.

For example, a provider might want to enforce:

- An aggregate customer rate of 50 Mbps.
- A voice traffic rate of 5 Mbps.

The parent can enforce the aggregate policer, while the child enforces the per-class policer.

```text
policy-map CHILD-POLICE
 class VOICE
  police cir 5000000
!
policy-map PARENT-POLICE
 class class-default
  police cir 50000000
  service-policy CHILD-POLICE
```

Apply the parent inbound:

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input PARENT-POLICE
```

For this design:

- VOICE traffic is subject to the 5 Mbps child policer.
- The aggregate traffic is subject to the 50 Mbps parent policer.
- A packet must satisfy all applicable policing constraints to be forwarded.

### Hierarchical Policers: Order of Operation

On Cisco ASR 1000 routers, hierarchical policers are evaluated **from child to parent**.

For example:

```text
Child AF31 policer = 2 Mbps
Child AF41 policer = 2 Mbps

Parent aggregate policer = 3 Mbps
```

The child policers individually allow up to 2 Mbps for AF31 and 2 Mbps for AF41.

However, the aggregate parent policer limits their combined traffic to 3 Mbps.

Even if packets conform to their child policers, they may still be dropped by the parent policer.

This behavior is distinct from the conceptual parent-to-child classification hierarchy.

**Classification hierarchy and policing evaluation order are not necessarily the same thing.**

Hierarchical policing behavior and supported actions vary by platform.

---

## HQoS Configuration Restrictions

Not every QoS action can be configured at every hierarchy level.

For a common shape-on-parent, queue-on-child design:

| HQoS level | Typical supported actions |
|---|---|
| Parent | Shaping, aggregate bandwidth control |
| Child | Priority queuing, bandwidth allocation, queue limits, WRED |
| Grandchild, if supported | Additional classification and supported QoS actions |

Important restrictions include:

### 1. Queuing Is Generally an Egress Operation

Commands such as:

```text
priority
bandwidth
bandwidth remaining
shape average
```

Control transmission scheduling.

They are generally used in outbound QoS policies.

### 2. Parent Shaping Is Commonly Required for Child Queuing

Some IOS XE platforms require a parent shaper when applying a hierarchical queuing policy.

For example, on documented platforms, a child CBWFQ policy attached to a logical interface may require shaping in the parent.

### 3. Unsupported Action Combinations

Certain combinations of shaping, queuing, marking, and policing may not be supported at a particular hierarchy level.

The IOS XE parser may accept an individual command but reject the complete hierarchy when the policy is attached.

### 4. Interface Support Varies

HQoS support depends on the platform, interface type, and software version.

Potential attachment points include:

- Physical interfaces.
- Ethernet subinterfaces.
- VLAN interfaces.
- Tunnel interfaces.
- Other supported logical interfaces.

Not all platforms support HQoS on every interface type.

### 5. Policy Validation

Cisco IOS XE may validate QoS configurations at two stages:

1. When creating or modifying the policy map.
2. When attaching the policy to an interface.

A policy can be syntactically valid but unsupported on a particular interface.

Always verify that the policy is successfully installed.

---

## HQoS Verification

The primary command for verifying HQoS is:

```text
R1# show policy-map interface GigabitEthernet2
```

This displays the applied policy hierarchy and associated statistics.

### Verify Policy Definitions

```text
R1# show policy-map PARENT
R1# show policy-map CHILD
```

Check that:

- The parent contains the intended shaping or policing actions.
- The parent references the correct child.
- The child contains the expected classes and QoS actions.

### Verify Policy Attachment

```text
R1# show running-config interface GigabitEthernet2
```

The interface should contain:

```text
service-policy output PARENT
```

The child policy does not appear as a separate interface-level service policy.

### Verify the Active Hierarchy

```text
R1# show policy-map interface GigabitEthernet2 output
```

The output should show the nesting relationship.

A simplified representation is:

```text
GigabitEthernet2

  Service-policy output: PARENT

    Class-map: class-default
      shape (average) cir 20000000

      Service-policy: CHILD

        Class-map: VOICE
          priority percent 10

        Class-map: BUSINESS
          bandwidth remaining percent 60

        Class-map: class-default
```

This is a structural illustration, not literal IOS XE output. Actual output includes platform-specific statistics and formatting.

### Important Statistics

| Statistic | Purpose |
|---|---|
| Parent shaping rate | Verify the aggregate transmission limit |
| Parent offered rate | Determine how much traffic reaches the parent |
| Parent transmitted rate | Check traffic forwarded under the shaping policy |
| Child class match counters | Verify classification |
| Child queue depth | Identify queue buildup |
| Child drops | Identify congestion-related packet loss |
| Priority policing counters | Identify packets exceeding priority policing limits |

For hardware verification on supported QFP-based routers:

```text
R1# show platform hardware qfp active feature qos interface GigabitEthernet2
```

The exact syntax and available information depend on the platform.

---

## Troubleshooting Parent-Child Policies

### Child Policy Does Not Match Traffic

Possible causes:

- The packet does not match the parent class.
- The packet does not match the intended child class.
- An earlier class in the policy matches the packet.
- Traffic markings differ from those expected.
- The policy is applied to the wrong interface or direction.

Remember that **matching the child class alone is insufficient if the packet does not match the parent class containing that child**.

### Shaping Is Working, but Traffic Is Not Prioritized

Check:

- Whether the child policy is correctly nested.
- Whether the child classes match the expected traffic.
- Whether the child has queuing actions configured.
- Whether the platform supports the configured scheduling mechanism.

A parent shaper by itself does not guarantee differentiated scheduling between applications.

### Child Bandwidth Rates Are Unexpected

Check the parent shaping rate.

For example:

```text
Parent shape = 50 Mbps
Child priority percent = 20
```

The configured priority rate is:

```text
50 Mbps × 20% = 10 Mbps
```

It is not 20% of the physical interface speed when the relevant bandwidth reference is the 50 Mbps parent shaper.

### Policy Cannot Be Attached

Possible causes:

- Unsupported hierarchy depth.
- Unsupported QoS action combination.
- Unsupported interface type.
- Bandwidth requirements exceeding supported limits.
- Hardware queue or scheduler resource limitations.

Examine the error message returned when applying the policy.

---

## Key Commands

| Objective | Command |
|---|---|
| Create a parent policy | `policy-map PARENT` |
| Create a child policy | `policy-map CHILD` |
| Select parent class | `class class-default` |
| Shape the parent to 20 Mbps | `shape average 20000000` |
| Nest a child policy | `service-policy CHILD` |
| Remove a nested child policy | `no service-policy CHILD` |
| Apply parent outbound | `service-policy output PARENT` |
| Apply parent inbound, when supported | `service-policy input PARENT` |
| Display policy definitions | `show policy-map` |
| Display a specific policy | `show policy-map PARENT` |
| Verify active hierarchy | `show policy-map interface GigabitEthernet2` |
| Verify interface attachment | `show running-config interface GigabitEthernet2` |

---

## Key Points

- HQoS organizes MQC policies into parent-child relationships.
- The parent policy is attached directly to the interface.
- The child policy is attached to a parent class using `service-policy`.
- `service-policy` in interface configuration mode attaches a policy to an interface.
- `service-policy` in policy-map class configuration mode nests a child policy.
- The parent commonly uses `class-default` to shape all outbound traffic.
- A child policy processes only traffic matching its parent class.
- Parent and child `class-default` classes operate in separate classification contexts.
- Parent policies can contain user-defined classes, not just `class-default`.
- The same child policy can be reused under multiple parents.
- Each application of a reused child policy has its own operational state.
- Child percentage-based rates typically use the applicable parent shaping rate as their reference.
- Some platforms support three-level HQoS with parent, child, and grandchild policies.
- Hierarchical policing can enforce both aggregate and per-class rate limits.
- On ASR 1000 platforms, hierarchical policers are evaluated from child to parent.
- Hierarchy depth, supported QoS actions, and interface attachment points are platform-dependent.
- `show policy-map interface` is the primary command for verifying HQoS operation.

---

## References

- [Cisco IOS XE 17.x: Applying QoS Features Using the MQC](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-mqc-0.html)
- [Cisco IOS XE 17.x: QoS Hierarchical Scheduling](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-hierarchical_scheduling.html)
- [Cisco IOS XE 17.x: Regulating Packet Flow on a Per-Class Basis Using Class-Based Traffic Shaping](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-regpkt-cls-0.html)
- [Cisco IOS XE 17.x: QoS Packet Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-policing.html)