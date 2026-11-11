
# Complete End-to-End MQC Configuration

This is a **CCIE-style configuration and verification lab** for an end-to-end Layer 3 QoS deployment using Cisco IOS XE's **Modular QoS CLI (MQC)**.

The lab combines classification, DSCP marking, policing, shaping, Low Latency Queuing (LLQ), Class-Based Weighted Fair Queuing (CBWFQ), Weighted Random Early Detection (WRED), and Hierarchical QoS (HQoS).

**Lab method:** Read the requirements for each task before looking at its solution. Configurations are **cumulative**, so each task builds on the previous tasks.

The lab uses three Cisco IOS XE routers and two Linux hosts. CSR1000v or C8000v routers are recommended because QoS feature support and scheduling statistics vary across virtual platforms.

---

## Scenario

An enterprise operates two sites connected through a routed WAN.

The company uses four traffic classes:

- **VOICE:** Delay-sensitive voice traffic requiring strict priority.
- **BUSINESS:** Important application traffic requiring preferential bandwidth allocation.
- **BULK:** Background transfers that should receive relatively low priority.
- **DEFAULT:** Ordinary best-effort traffic.

Two WAN connections have different contracted service rates:

- **WAN Link 1:** 20 Mbps.
- **WAN Link 2:** 50 Mbps.

Both connections use 1 Gbps physical interfaces.

The enterprise must implement a consistent QoS policy to provide appropriate application treatment across both links.

### Topology

```text
                     WAN Link 1         WAN Link 2
                    20 Mbps  CIR        50 Mbps CIR
HostA ---------- R1 ------------- R2 ------------- R3 -------------- HostB
            E0/0   E0/1        E0/0  E0/1       E0/0 E0/1
    10.10.10.0/24    10.0.12.0/30     10.0.23.0/30     10.30.30.0/24
```

All router-to-router links are Ethernet connections. Their contracted service rates will be enforced using QoS shaping policies.

### Device Addressing

| Device | Interface | IP address | Connected to |
|---|---|---|---|
| R1 | Ethernet0/0 | 10.10.10.1/24 | HostA |
| R1 | Ethernet0/1 | 10.0.12.1/30 | R2 |
| R2 | Ethernet0/0 | 10.0.12.2/30 | R1 |
| R2 | Ethernet0/1 | 10.0.23.2/30 | R3 |
| R3 | Ethernet0/0 | 10.0.23.3/30 | R2 |
| R3 | Ethernet0/1 | 10.30.30.1/24 | HostB |
| HostA | eth0 | 10.10.10.10/24 | R1 |
| HostB | eth0 | 10.30.30.10/24 | R3 |

### QoS Traffic Classes

| Class | DSCP | Decimal | Traffic |
|---|---|---|---|
| VOICE | EF | 46 | UDP destination port 5001 |
| BUSINESS | AF31 | 26 | TCP destination port 5201 |
| BULK | CS1 | 8 | TCP destination port 5202 |
| DEFAULT | CS0 | 0 | All other traffic |

The hosts will generate traffic using `iperf3`.

Traffic classification will initially occur at the network edges. Downstream routers will classify traffic based on DSCP.

---

## Task 1: Configure Basic IP Connectivity

### Requirements

1. Configure all router interface addresses according to the addressing table.
2. Configure static routes so HostA and HostB can communicate.
3. Configure the Linux hosts with their assigned IP addresses and default gateways.
4. Verify end-to-end connectivity before implementing QoS.
5. Do not configure any QoS policies yet.

### Solution

**R1**

```cisco
hostname R1
!
interface Ethernet0/0
 description TO-HOSTA
 ip address 10.10.10.1 255.255.255.0
 no shutdown
!
interface Ethernet0/1
 description TO-R2
 ip address 10.0.12.1 255.255.255.252
 no shutdown
!
ip route 10.30.30.0 255.255.255.0 10.0.12.2
```

**R2**

```cisco
hostname R2
!
interface Ethernet0/0
 description TO-R1
 ip address 10.0.12.2 255.255.255.252
 no shutdown
!
interface Ethernet0/1
 description TO-R3
 ip address 10.0.23.2 255.255.255.252
 no shutdown
!
ip route 10.10.10.0 255.255.255.0 10.0.12.1
ip route 10.30.30.0 255.255.255.0 10.0.23.3
```

**R3**

```cisco
hostname R3
!
interface Ethernet0/0
 description TO-R2
 ip address 10.0.23.3 255.255.255.252
 no shutdown
!
interface Ethernet0/1
 description TO-HOSTB
 ip address 10.30.30.1 255.255.255.0
 no shutdown
!
ip route 10.10.10.0 255.255.255.0 10.0.23.2
```

**HostA (Linux)**

```bash
sudo ip addr add 10.10.10.10/24 dev eth0
sudo ip link set eth0 up
sudo ip route replace default via 10.10.10.1
```

**HostB (Linux)**

```bash
sudo ip addr add 10.30.30.10/24 dev eth0
sudo ip link set eth0 up
sudo ip route replace default via 10.30.30.1
```

These commands assume that `eth0` is the host's connected interface and has no conflicting address configuration.

### Verify

**R1**

```cisco
R1# show ip interface brief
R1# show ip route
R1# ping 10.0.12.2
R1# ping 10.30.30.1
```

**HostA**

```bash
ping -c 5 10.30.30.10
```

**HostB**

```bash
ping -c 5 10.10.10.10
```

Both hosts should be able to reach each other.

**Do not proceed until end-to-end connectivity is working.**

---

## Task 2: Configure Classification and DSCP Marking

### Requirements

Configure R1 to classify traffic entering from HostA.

1. Match UDP packets destined for port 5001 as VOICE.
2. Match TCP packets destined for port 5201 as BUSINESS.
3. Match TCP packets destined for port 5202 as BULK.
4. Mark the traffic using the DSCP values specified in the QoS traffic class table.
5. Mark all other traffic as CS0.
6. Apply the policy inbound on R1's HostA-facing interface.

HostA is considered an **untrusted endpoint**, so R1 must assign the authoritative DSCP markings.

### Solution

**R1**

First, configure the classification ACLs.

```cisco
ip access-list extended VOICE-ACL
 permit udp host 10.10.10.10 host 10.30.30.10 eq 5001
!
ip access-list extended BUSINESS-ACL
 permit tcp host 10.10.10.10 host 10.30.30.10 eq 5201
!
ip access-list extended BULK-ACL
 permit tcp host 10.10.10.10 host 10.30.30.10 eq 5202
```

Configure the class maps.

```cisco
class-map match-all VOICE
 match access-group name VOICE-ACL
!
class-map match-all BUSINESS
 match access-group name BUSINESS-ACL
!
class-map match-all BULK
 match access-group name BULK-ACL
```

Configure the ingress policy.

```cisco
policy-map INGRESS-MARKING
 class VOICE
  set dscp ef
 class BUSINESS
  set dscp af31
 class BULK
  set dscp cs1
 class class-default
  set dscp default
```

Apply the policy.

```cisco
interface Ethernet0/0
 service-policy input INGRESS-MARKING
```

This policy performs classification and marking at the trust boundary.

All traffic that does not match VOICE, BUSINESS, or BULK is remarked to CS0.

### Verify

```cisco
R1# show class-map
R1# show policy-map INGRESS-MARKING
R1# show policy-map interface Ethernet0/0
```

The policy should be attached in the input direction.

To verify actual traffic matching, prepare HostB to receive traffic using `iperf3`.

**HostB**

```bash
iperf3 -s -D -p 5001
iperf3 -s -D -p 5201
iperf3 -s -D -p 5202
iperf3 -s -D -p 5203
```

Start a voice test from HostA.

**HostA**

```bash
iperf3 -c 10.30.30.10 -u -p 5001 -b 1M -t 15
```

Verify that the VOICE class's packet counter increases.

Next, test BUSINESS.

```bash
iperf3 -c 10.30.30.10 -p 5201 -t 15
```

Verify that the BUSINESS class counter increases.

Repeat for BULK and DEFAULT:

```bash
iperf3 -c 10.30.30.10 -p 5202 -t 15
iperf3 -c 10.30.30.10 -p 5203 -t 15
```

TCP port 5203 does not match any named class, so it should match `class-default`.

**Expected result:** Each traffic type matches its intended class and receives the configured DSCP marking.

---

## Task 3: Configure Ingress Traffic Policing

### Requirements

The enterprise wants to prevent excessive bulk transfers from consuming network resources.

Modify R1's ingress policy:

1. Police BULK traffic to a CIR of 8 Mbps.
2. Transmit conforming packets.
3. Drop packets exceeding the policing profile.
4. Preserve the existing CS1 marking for forwarded BULK traffic.
5. Do not police VOICE or BUSINESS.

Do not create a second input service policy. Modify the existing policy.

### Solution

**R1**

```cisco
policy-map INGRESS-MARKING
 class BULK
  police cir 8000000 conform-action transmit exceed-action drop
```

The BULK class now has two QoS actions:

```cisco
policy-map INGRESS-MARKING
 class BULK
  set dscp cs1
  police cir 8000000 conform-action transmit exceed-action drop
```

The policer enforces a CIR of 8 Mbps.

Traffic is subject to the policer's token bucket behavior, so short bursts may be permitted according to the configured or default burst allowance.

### Verify

```cisco
R1# show policy-map interface Ethernet0/0
```

Generate a sustained BULK transfer exceeding the CIR.

**HostA**

```bash
iperf3 -c 10.30.30.10 -p 5202 -b 20M -t 60
```

The specified TCP bitrate is an offered-load target. Actual sending rates depend on TCP congestion control and host performance.

While the test runs, inspect the policy:

```cisco
R1# show policy-map interface Ethernet0/0
```

Check the BULK class for:

- Packets matching the class.
- Packets conforming to the policer.
- Packets exceeding the policer.
- Packet drops.

TCP may reduce its sending rate in response to packet loss, so the observed drop rate may vary.

**Expected result:** BULK traffic is limited by the 8 Mbps policer over time, subject to token bucket bursts and platform accounting. VOICE and BUSINESS are unaffected by this policer.

---

## Task 4: Configure Hierarchical QoS on R1

### Requirements

R1 connects to WAN Link 1.

The interface operates at 1 Gbps, but the contracted WAN service rate is 20 Mbps.

Configure a hierarchical output QoS policy:

1. Shape aggregate traffic to 20 Mbps.
2. Classify packets using their DSCP markings.
3. Allocate 10% of the parent shaping rate to VOICE using LLQ.
4. Configure the following excess-bandwidth sharing weights:
   - BUSINESS: 60%.
   - BULK: 10%.
   - DEFAULT: 30%.
5. Enable DSCP-based WRED for BUSINESS.
6. Apply the hierarchical policy outbound on Ethernet0/1.

### Solution

**R1**

Create the DSCP-based class maps.

```cisco
class-map match-all Q-VOICE
 match dscp ef
!
class-map match-all Q-BUSINESS
 match dscp af31
!
class-map match-all Q-BULK
 match dscp cs1
```

Create the child queuing policy.

```cisco
policy-map WAN-CHILD
 class Q-VOICE
  priority percent 10
 class Q-BUSINESS
  bandwidth remaining percent 60
  random-detect dscp-based
 class Q-BULK
  bandwidth remaining percent 10
 class class-default
  bandwidth remaining percent 30
```

Create the parent shaping policy.

```cisco
policy-map WAN-20M
 class class-default
  shape average 20000000
  service-policy WAN-CHILD
```

Apply the parent policy.

```cisco
interface Ethernet0/1
 service-policy output WAN-20M
```

### Understanding the Configuration

The policy creates the following hierarchy:

```text
              Egress Traffic
                    |
                    v
           +----------------+
           | Child Policy   |
           |                |
           | VOICE          | -> LLQ
           | BUSINESS       | -> CBWFQ / WRED
           | BULK           | -> CBWFQ
           | DEFAULT        | -> CBWFQ
           +----------------+
                    |
                    v
           +----------------+
           | Parent Policy  |
           |                |
           | Shape 20 Mbps  |
           +----------------+
                    |
                    v
               Ethernet0/1
```

#### VOICE Priority Rate

```text
Parent shaping rate = 20 Mbps

Priority percent    = 10%

Configured priority rate = 2 Mbps
```

VOICE receives strict priority scheduling with a configured rate of 2 Mbps.

On supported IOS XE QFP implementations, `priority percent` uses conditional priority policing, meaning the rate is enforced during congestion.

#### Excess Bandwidth Sharing

The remaining bandwidth classes use the following weights:

| Class | Excess-bandwidth weight |
|---|---|
| BUSINESS | 60% |
| BULK | 10% |
| DEFAULT | 30% |

When the applicable remaining bandwidth is 18 Mbps and all three nonpriority classes are continuously backlogged, their nominal weighted shares are:

```text
BUSINESS = 18 Mbps × 60% = 10.8 Mbps

BULK     = 18 Mbps × 10% =  1.8 Mbps

DEFAULT  = 18 Mbps × 30% =  5.4 Mbps
```

This assumes VOICE consumes 2 Mbps and ignores scheduling overhead.

These values are **relative scheduling shares, not guaranteed minimums or maximum rates**.

If one class does not use its available share, other classes may use the unused capacity.

#### WRED

The BUSINESS class uses:

```cisco
random-detect dscp-based
```

WRED can probabilistically drop eligible packets as the average queue depth increases.

This helps TCP traffic react to congestion before the queue becomes completely full.

WRED is not configured on the strict priority VOICE class.

### Verify

```cisco
R1# show policy-map WAN-CHILD
R1# show policy-map WAN-20M
R1# show policy-map interface Ethernet0/1
```

Verify that:

- The parent shaping rate is 20 Mbps.
- The child policy is attached under the parent.
- VOICE has strict priority scheduling.
- BUSINESS, BULK, and DEFAULT have the intended bandwidth weights.
- WRED is enabled for BUSINESS.

The exact displayed counters and scheduling information depend on the platform.

---

## Task 5: Configure QoS on R2

### Requirements

R2 forwards traffic toward R3 over WAN Link 2.

WAN Link 2 has a contracted service rate of 50 Mbps.

Configure R2 to provide the same traffic class treatment as R1, but with a 50 Mbps aggregate shaping rate.

1. Classify traffic by DSCP.
2. Use the same child queuing policy as R1.
3. Shape traffic to 50 Mbps.
4. Apply the policy outbound on Ethernet0/1.
5. Preserve the DSCP markings received from R1.

Do not repeat application classification using transport-layer ports.

### Solution

**R2**

Configure the class maps.

```cisco
class-map match-all Q-VOICE
 match dscp ef
!
class-map match-all Q-BUSINESS
 match dscp af31
!
class-map match-all Q-BULK
 match dscp cs1
```

Configure the child policy.

```cisco
policy-map WAN-CHILD
 class Q-VOICE
  priority percent 10
 class Q-BUSINESS
  bandwidth remaining percent 60
  random-detect dscp-based
 class Q-BULK
  bandwidth remaining percent 10
 class class-default
  bandwidth remaining percent 30
```

Configure the parent policy.

```cisco
policy-map WAN-50M
 class class-default
  shape average 50000000
  service-policy WAN-CHILD
```

Apply the policy.

```cisco
interface Ethernet0/1
 service-policy output WAN-50M
```

### Verify

```cisco
R2# show policy-map interface Ethernet0/1
```

The parent shaping rate should be 50 Mbps.

The VOICE priority rate is now:

```text
50 Mbps × 10% = 5 Mbps
```

The child configuration is identical to R1, but percentage-based allocations are calculated using R2's applicable parent shaping rate.

### Important Observation

In this topology, R1 limits Site A traffic to 20 Mbps before sending it to R2.

Therefore, if R2 receives no other traffic, its 50 Mbps output shaper should not experience sustained congestion.

The policy is still configured to provide the intended forwarding treatment if additional traffic later enters R2 and competes for WAN Link 2.

**A configured queuing policy does not necessarily imply that the interface is congested.**

---

## Task 6: Configure QoS in the Reverse Direction

### Requirements

So far, QoS has been configured only for traffic traveling from Site A toward Site B.

Configure equivalent QoS treatment for traffic traveling from Site B toward Site A.

1. Configure R3 to classify and mark traffic entering from HostB.
2. Use the same DSCP mapping as R1.
3. Police BULK traffic to 8 Mbps.
4. Configure R3 to shape traffic toward R2 to 50 Mbps.
5. Configure R2 to shape traffic toward R1 to 20 Mbps.
6. Use the same child queuing policy in both directions.

The classification ACLs on R3 must match traffic traveling from HostB to HostA.

### Solution

**R3: Classification and Marking**

```cisco
ip access-list extended VOICE-ACL
 permit udp host 10.30.30.10 host 10.10.10.10 eq 5001
!
ip access-list extended BUSINESS-ACL
 permit tcp host 10.30.30.10 host 10.10.10.10 eq 5201
!
ip access-list extended BULK-ACL
 permit tcp host 10.30.30.10 host 10.10.10.10 eq 5202
!
class-map match-all VOICE
 match access-group name VOICE-ACL
!
class-map match-all BUSINESS
 match access-group name BUSINESS-ACL
!
class-map match-all BULK
 match access-group name BULK-ACL
!
policy-map INGRESS-MARKING
 class VOICE
  set dscp ef
 class BUSINESS
  set dscp af31
 class BULK
  set dscp cs1
  police cir 8000000 conform-action transmit exceed-action drop
 class class-default
  set dscp default
!
interface Ethernet0/1
 service-policy input INGRESS-MARKING
```

**R3: Egress HQoS**

```cisco
class-map match-all Q-VOICE
 match dscp ef
!
class-map match-all Q-BUSINESS
 match dscp af31
!
class-map match-all Q-BULK
 match dscp cs1
!
policy-map WAN-CHILD
 class Q-VOICE
  priority percent 10
 class Q-BUSINESS
  bandwidth remaining percent 60
  random-detect dscp-based
 class Q-BULK
  bandwidth remaining percent 10
 class class-default
  bandwidth remaining percent 30
!
policy-map WAN-50M
 class class-default
  shape average 50000000
  service-policy WAN-CHILD
!
interface Ethernet0/0
 service-policy output WAN-50M
```

**R2: Reverse-Direction HQoS**

R2 already has the class maps and child policy from Task 5.

Create a 20 Mbps parent policy.

```cisco
policy-map WAN-20M
 class class-default
  shape average 20000000
  service-policy WAN-CHILD
!
interface Ethernet0/0
 service-policy output WAN-20M
```

### Verify

**R3**

```cisco
R3# show policy-map interface Ethernet0/1
R3# show policy-map interface Ethernet0/0
```

**R2**

```cisco
R2# show policy-map interface Ethernet0/0
R2# show policy-map interface Ethernet0/1
```

The final policy placement should be:

| Router | Interface | Direction | Policy |
|---|---|---|---|
| R1 | Ethernet0/0 | Input | INGRESS-MARKING |
| R1 | Ethernet0/1 | Output | WAN-20M |
| R2 | Ethernet0/0 | Output | WAN-20M |
| R2 | Ethernet0/1 | Output | WAN-50M |
| R3 | Ethernet0/0 | Output | WAN-50M |
| R3 | Ethernet0/1 | Input | INGRESS-MARKING |

Note that R2's two output policies operate in opposite traffic directions.

**The 20 Mbps and 50 Mbps service rates are enforced independently for each direction.**

---

## Task 7: Generate Traffic and Verify End-to-End QoS

### Requirements

Generate multiple traffic classes simultaneously to create congestion.

Verify that:

1. R1 classifies and marks each traffic class correctly.
2. R1's BULK policer enforces its configured rate.
3. R1 shapes aggregate outbound traffic to 20 Mbps.
4. VOICE receives strict priority scheduling.
5. BUSINESS, BULK, and DEFAULT receive differentiated scheduling treatment.
6. R2 identifies the traffic using its DSCP values.
7. The DSCP markings remain consistent across the network.
8. The reverse-direction policies operate as intended.

### Solution

#### Step 1: Generate Forward-Direction Traffic

Ensure that the `iperf3` servers from Task 2 are running on HostB.

On HostA, start four traffic streams simultaneously.

```bash
iperf3 -c 10.30.30.10 -u -p 5001 -b 1.5M -t 90 > /tmp/voice.log 2>&1 &
iperf3 -c 10.30.30.10 -p 5201 -b 10M -t 90 > /tmp/business.log 2>&1 &
iperf3 -c 10.30.30.10 -p 5202 -b 12M -t 90 > /tmp/bulk.log 2>&1 &
iperf3 -c 10.30.30.10 -p 5203 -b 5M -t 90 > /tmp/default.log 2>&1 &
wait
```

The intended offered load is:

| Class | Offered load |
|---|---:|
| VOICE | 1.5 Mbps |
| BUSINESS | 10 Mbps |
| BULK | 12 Mbps |
| DEFAULT | 5 Mbps |
| **Total** | **28.5 Mbps** |

This exceeds the 20 Mbps WAN Link 1 shaping rate.

The BULK policer also restricts traffic exceeding its 8 Mbps CIR.

Consequently, even after accounting for the nominal BULK policing rate, the aggregate offered load is approximately:

```text
VOICE       1.5 Mbps
BUSINESS   10.0 Mbps
BULK        8.0 Mbps
DEFAULT     5.0 Mbps
---------------------
TOTAL      24.5 Mbps
```

The actual sending and receiving rates will vary due to TCP behavior, policing, queuing, and virtual platform performance.

#### Step 2: Verify R1 Ingress

While the traffic is running:

```cisco
R1# show policy-map interface Ethernet0/0
```

Verify that the class counters increase.

The expected classes are:

```text
VOICE     -> UDP 5001 -> DSCP EF

BUSINESS  -> TCP 5201 -> DSCP AF31

BULK      -> TCP 5202 -> DSCP CS1

DEFAULT   -> TCP 5203 -> DSCP CS0
```

The BULK class should also display policing statistics.

If the offered BULK rate exceeds the policer's permitted rate and burst allowance, exceed/drop counters should increase.

#### Step 3: Verify R1 Egress

```cisco
R1# show policy-map interface Ethernet0/1
```

Inspect the parent policy.

Expected shaping rate:

```text
20,000,000 bps
```

Inspect the child policy.

Expected scheduling configuration:

```text
VOICE
  Priority: 10%

BUSINESS
  Bandwidth remaining: 60%
  WRED enabled

BULK
  Bandwidth remaining: 10%

DEFAULT
  Bandwidth remaining: 30%
```

The interface should transmit at approximately the configured shaping rate when sufficient traffic is offered and the platform can sustain the load.

The exact rate and queue statistics may differ because of encapsulation accounting, scheduler implementation, and virtual router performance.

#### Step 4: Verify R2 Egress

```cisco
R2# show policy-map interface Ethernet0/1
```

Verify that R2 classifies the forwarded packets into the same traffic classes.

This confirms that R2 can identify the traffic using the DSCP markings applied by R1.

R2 should not need to repeat the original ACL-based application classification.

Also note that its 50 Mbps shaper should not be persistently congested by traffic already limited to 20 Mbps at R1.

#### Step 5: Verify DSCP Markings at HostB

On HostB, capture the received traffic.

```bash
sudo tcpdump -ni eth0 -vv 'ip and (udp port 5001 or tcp port 5201 or tcp port 5202 or tcp port 5203)'
```

Inspect the IPv4 ToS values.

For packets whose ECN bits are zero:

| DSCP | Decimal | IPv4 ToS (hex) |
|---|---|---|
| EF | 46 | 0xb8 |
| AF31 | 26 | 0x68 |
| CS1 | 8 | 0x20 |
| CS0 | 0 | 0x00 |

The ToS byte also includes the two ECN bits, so the exact hex value may differ when ECN is used.

The upper six bits determine DSCP.

If the received packets have the intended markings, this verifies that the markings were preserved across R1, R2, and R3.

#### Step 6: Test Reverse-Direction Traffic

Start equivalent `iperf3` servers on HostA.

```bash
iperf3 -s -D -p 5001
iperf3 -s -D -p 5201
iperf3 -s -D -p 5202
iperf3 -s -D -p 5203
```

Generate traffic from HostB toward HostA.

**HostB**

```bash
iperf3 -c 10.10.10.10 -u -p 5001 -b 1.5M -t 90 > /tmp/voice-reverse.log 2>&1 &
iperf3 -c 10.10.10.10 -p 5201 -b 10M -t 90 > /tmp/business-reverse.log 2>&1 &
iperf3 -c 10.10.10.10 -p 5202 -b 12M -t 90 > /tmp/bulk-reverse.log 2>&1 &
iperf3 -c 10.10.10.10 -p 5203 -b 5M -t 90 > /tmp/default-reverse.log 2>&1 &
wait
```

Verify the reverse path.

```cisco
R3# show policy-map interface Ethernet0/1
R3# show policy-map interface Ethernet0/0
R2# show policy-map interface Ethernet0/0
```

Expected behavior:

1. R3 classifies and marks the traffic.
2. R3 polices BULK traffic.
3. R3 shapes traffic toward R2 to 50 Mbps.
4. R2 classifies the traffic using DSCP.
5. R2 shapes traffic toward R1 to 20 Mbps.
6. The 20 Mbps R2 egress policy becomes the principal controlled bottleneck for the reverse traffic.

---

## Task 8: Troubleshooting Exercises

After completing the configuration, use the following troubleshooting scenarios to practice identifying QoS problems.

Try to determine the cause before reading the solution.

### Scenario 1: BUSINESS Traffic Matches `class-default`

**Symptom:** TCP traffic destined for port 5201 increments R1's `class-default` counter rather than the BUSINESS class.

**Task:** Identify possible classification problems.

<details>
<summary>Solution</summary>

Check:

```cisco
R1# show access-lists BUSINESS-ACL
R1# show class-map BUSINESS
R1# show policy-map interface Ethernet0/0
```

Possible causes include:

- Incorrect source or destination IP address.
- Incorrect TCP destination port.
- ACL not referenced by the class map.
- Class map not referenced by the policy.
- Service policy attached to the wrong interface or direction.

Also verify that the test traffic is being generated as expected.

</details>

### Scenario 2: R1 Transmits Above the Contracted Rate

**Symptom:** R1 appears to transmit more than 20 Mbps toward R2.

**Task:** Verify the hierarchical policy and its attachment.

<details>
<summary>Solution</summary>

Check:

```cisco
R1# show policy-map interface Ethernet0/1
R1# show running-config interface Ethernet0/1
```

Verify that:

- `WAN-20M` is attached in the output direction.
- The parent contains `shape average 20000000`.
- The child policy is attached beneath the parent.
- The output rate is being measured over an appropriate interval.

Short bursts can exceed the configured average shaping rate.

Also consider whether the observed counters include encapsulation overhead and whether the virtual platform is enforcing the policy correctly.

</details>

### Scenario 3: VOICE Experiences Packet Loss

**Symptom:** Voice traffic experiences drops during congestion.

**Task:** Determine whether the problem is related to classification, priority policing, or another forwarding issue.

<details>
<summary>Solution</summary>

Check:

```cisco
R1# show policy-map interface Ethernet0/0
R1# show policy-map interface Ethernet0/1
```

Possible causes include:

- Voice packets do not match the VOICE class.
- Voice packets are not marked EF.
- The egress policy does not match EF.
- Voice traffic exceeds the configured priority rate during congestion.
- Packets are lost elsewhere in the network.

The configured priority rate is:

```text
20 Mbps × 10% = 2 Mbps
```

If voice traffic exceeds 2 Mbps during sustained congestion, the priority policer may drop excess packets.

Increasing the priority allocation may be appropriate if the actual voice traffic requirement justifies it.

However, excessive priority allocation can reduce the bandwidth available to other traffic classes.

</details>

### Scenario 4: R2 Has No Queue Drops

**Symptom:** R2's 50 Mbps egress policy reports no queue drops, even when the Site A hosts generate more than 20 Mbps.

**Task:** Determine whether this indicates a configuration problem.

<details>
<summary>Solution</summary>

Not necessarily.

R1 shapes outbound traffic to 20 Mbps before forwarding it toward R2.

R2 receives approximately 20 Mbps from R1 but can transmit up to 50 Mbps toward R3.

Unless R2 receives additional traffic, its 50 Mbps egress interface should not experience sustained congestion.

**A correctly configured QoS policy may have no queue drops because there is no congestion.**

</details>

### Scenario 5: WRED Has No Random Drops

**Symptom:** BUSINESS packets match the correct class, but WRED reports no random drops.

**Task:** Determine whether WRED is functioning incorrectly.

<details>
<summary>Solution</summary>

WRED does not necessarily drop packets simply because it is enabled.

Random drops depend on the average queue depth and configured thresholds.

If the average queue depth remains below the minimum threshold, WRED does not perform random drops.

TCP congestion control may also reduce offered traffic before the queue reaches a threshold that triggers random drops.

Verify:

```cisco
R1# show policy-map interface Ethernet0/1
```

Check the BUSINESS class's queue and WRED statistics.

If supported, generate sufficient eligible traffic to create sustained queue pressure.

</details>

---

## Final Verification Checklist

Before completing the lab, verify the following:

- [ ] HostA and HostB can communicate in both directions.
- [ ] R1 classifies and marks traffic correctly.
- [ ] R3 classifies and marks reverse-direction traffic correctly.
- [ ] BULK traffic is policed to 8 Mbps at the ingress trust boundaries.
- [ ] R1 shapes outbound traffic to 20 Mbps.
- [ ] R2 shapes traffic toward R3 to 50 Mbps.
- [ ] R2 shapes traffic toward R1 to 20 Mbps.
- [ ] R3 shapes traffic toward R2 to 50 Mbps.
- [ ] VOICE uses LLQ with a 10% priority allocation.
- [ ] BUSINESS, BULK, and DEFAULT use the intended bandwidth weights.
- [ ] WRED is enabled for BUSINESS.
- [ ] DSCP markings are preserved across the routed network.
- [ ] Traffic matches the expected classes at each QoS-enabled hop.
- [ ] Congestion and packet drops occur only where expected under the test conditions.

---

## Lab Summary

This lab demonstrates how multiple MQC features work together to implement end-to-end Layer 3 QoS.

| QoS mechanism | Configuration location |
|---|---|
| Classification using ACLs | R1 and R3 ingress |
| DSCP marking | R1 and R3 ingress |
| Policing | R1 and R3 ingress |
| DSCP-based classification | WAN egress policies |
| Hierarchical shaping | R1, R2, and R3 WAN egress |
| LLQ | Child queuing policy |
| CBWFQ | Child queuing policy |
| WRED | BUSINESS class |
| End-to-end verification | All three routers and both hosts |

The lab illustrates several important principles:

1. **Classify and mark near the source.** Downstream routers can use DSCP for classification rather than repeating detailed ACL-based classification.

2. **Police where traffic contracts must be enforced.** Policing is appropriate for immediate rate enforcement at network boundaries.

3. **Shape before lower-bandwidth connections.** The parent shaper controls aggregate traffic so that a faster physical interface does not overwhelm a lower-rate service.

4. **Use LLQ and CBWFQ to manage congestion.** Different traffic classes receive scheduling treatment appropriate to their requirements.

5. **Apply QoS independently in both directions.** Forward and reverse traffic require their own policy placement.

6. **Verify under congestion.** Class matching alone does not prove that traffic receives the intended queuing treatment.

**The purpose of end-to-end QoS is not to eliminate congestion, but to control how network resources are allocated when congestion occurs.**

---

## References

- [Cisco IOS XE 17.x: Applying QoS Features Using the MQC](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-mqc-0.html)
- [Cisco IOS XE 17.x: QoS Scheduling](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-scheduling.html)
- [Cisco IOS XE 17.x: QoS Packet Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-policing.html)
