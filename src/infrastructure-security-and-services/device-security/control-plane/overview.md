# 4.1.a Control Plane Policing and Protection

**Control Plane Policing (CoPP)** and **Control Plane Protection (CPPr)** are Cisco IOS XE security features that protect the device's **control plane** from excessive or unwanted traffic.

The control plane is responsible for processing routing protocols, management traffic, and other packets that require CPU processing.

If excessive traffic reaches the control plane, it can consume CPU resources and interfere with critical functions such as routing protocol operation and device management.

CoPP and CPPr help prevent this by **classifying, policing, and selectively dropping traffic before it reaches the CPU for processing**.

---

## Control Plane vs. Data Plane

Network devices use different functional planes to process traffic.

| Plane | Function | Examples |
|---|---|---|
| **Management Plane** | Provides administrative access and monitoring | SSH, SNMP, NETCONF |
| **Control Plane** | Maintains routing and other control information | OSPF, BGP, EIGRP |
| **Data Plane** | Forwards packets through the device | Normal IP transit traffic |

The management plane is logically distinct, but management traffic destined for the device also requires control-plane processing.

Most transit traffic is handled by the forwarding hardware without requiring CPU processing.

However, packets destined for the device itself, along with certain packets that cannot be handled by the normal forwarding path, must be processed by the CPU.

Packets redirected from the forwarding path to the CPU are commonly called **punted packets**.

CoPP and CPPr protect the processing resources used by this traffic.

---

## Control Plane Policing (CoPP)

**CoPP** applies Quality of Service (QoS) policies to traffic directed toward the control plane.

It uses the Cisco **Modular QoS CLI (MQC)** framework:

1. **Class maps** identify different categories of traffic.
2. **Policy maps** define actions for each traffic class.
3. **Service policies** apply the configured policies to the control plane.

For example, a CoPP policy could:

- Allow routing protocol traffic at a sufficient rate for normal operation.
- Limit the amount of SSH traffic reaching the CPU.
- Restrict excessive ICMP traffic.
- Drop traffic exceeding configured policing rates.

Unlike interface-level QoS policies, CoPP is applied to the device's logical **control-plane interface**.

This allows a policy to protect the control plane from traffic arriving through multiple physical interfaces.

> CoPP protects the control plane. It does not ordinarily police transit traffic that is forwarded entirely by the data plane.

---

## Control Plane Protection (CPPr)

**CPPr** extends CoPP by dividing control-plane traffic into three logical **control-plane subinterfaces**.

| Subinterface | Traffic |
|---|---|
| **Host** | IP traffic directly destined for the device, such as routing protocol and management traffic |
| **Transit** | Transit IP traffic that requires software switching by the CPU |
| **CEF-Exception** | Traffic requiring exceptional processing, such as certain packets with IP options, and non-IP control traffic such as ARP |

CPPr allows separate policing policies to be applied to these categories.

It also introduces additional protection mechanisms, including:

- **Port filtering:** Can drop packets destined for closed TCP/UDP ports.
- **Queue thresholding:** Limits the number of queued packets associated with particular protocols.

These additional mechanisms are supported on specific platforms and are primarily associated with the host subinterface.

CPPr therefore provides more granular control than traditional aggregate CoPP.

---

## CoPP vs. CPPr

| Feature | CoPP | CPPr |
|---|---|---|
| Protects CPU resources | Yes | Yes |
| Uses MQC policies | Yes | Yes |
| Aggregate control-plane policing | Yes | Yes |
| Separate host, transit, and CEF-exception policing | No | Yes |
| Port filtering | No | Supported on certain platforms |
| Per-protocol queue thresholding | No | Supported on certain platforms |

CoPP and CPPr are related mechanisms rather than mutually exclusive alternatives.

An aggregate CoPP policy can provide protection across control-plane traffic, while CPPr allows additional policies to be applied to individual control-plane traffic categories.

Actual implementation and supported commands depend on the Cisco IOS XE platform.

---

## Platform-Specific Control Plane Protection

Control-plane protection differs among Cisco IOS XE platforms.

For example, **Catalyst 9000 switches** use a predefined **System CoPP** policy, commonly named `system-cpp-policy`, to protect CPU-bound traffic.

System CoPP includes predefined traffic classes and policing rates that can be adjusted within platform-supported limits.

This differs from router platforms that support user-defined CoPP policies with their own class maps and policy maps.

Understanding these differences is important when configuring and troubleshooting control-plane protection on different devices.

---

## What This Section Covers

The following pages examine:

- Control-plane traffic types and packet punt paths.
- CoPP traffic classification using ACLs and class maps.
- Policing rates, burst parameters, and policing actions.
- CoPP configuration using MQC.
- CPPr subinterfaces and their traffic classifications.
- Port filtering and queue thresholding.
- Management Plane Protection (MPP).
- Catalyst 9000 System CoPP.
- Verification and troubleshooting of control-plane policies.

The primary objective is to understand **which traffic is subject to control-plane protection, how policies are applied, and how to verify that legitimate control-plane traffic remains operational while excessive or unwanted traffic is restricted**.

---

## References

- [Cisco IOS XE 17.x - Control Plane Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-ctrl-pln-plc-0.html)
- [Cisco - Control Plane Protection](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_plcshp/configuration/15-mt/qos-plcshp-15-mt-book/qos-plcshp-cpp.html)
- [Cisco IOS XE Software Hardening Guide](https://sec.cloudapps.cisco.com/security/center/resources/IOS_XE_hardening)
- [Cisco Catalyst 9300 IOS XE 17.18.x - Configuring Control Plane Policing](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-18/configuration_guide/sec/b_1718_sec_9300_cg/configuring_control_plane_policing.html)