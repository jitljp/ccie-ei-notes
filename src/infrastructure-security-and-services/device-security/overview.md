# Device Security on Cisco IOS XE

Network devices must be protected against unauthorized access and attacks that could disrupt their operation.

Cisco IOS XE provides several security mechanisms to protect the device itself, including **Control Plane Policing and Protection (CoPP/CPPr)** and **Authentication, Authorization, and Accounting (AAA)**.

These technologies serve different purposes:

| Technology | Purpose |
|---|---|
| **CoPP/CPPr** | Protects the device's control plane from excessive or unwanted traffic |
| **AAA** | Controls who can access the device, what actions they can perform, and how their activities are recorded |

---

## Control Plane Policing and Protection

The **control plane** processes traffic that requires the device's CPU, including routing protocol messages, management traffic, and packets that cannot be handled entirely by the forwarding hardware.

Excessive control-plane traffic can overwhelm the CPU, potentially disrupting routing protocols, management access, and other critical functions.

Cisco IOS XE provides two related mechanisms to protect the control plane:

- **Control Plane Policing (CoPP):** Uses QoS policies to classify and police traffic destined for the control plane.
- **Control Plane Protection (CPPr):** Extends CoPP by separating control-plane traffic into different categories, allowing more granular protection.

This section covers control-plane traffic classification, policing, CoPP and CPPr configuration, platform-specific features, and verification and troubleshooting.

---

## Authentication, Authorization, and Accounting

**AAA** provides a framework for controlling administrative access to network devices.

AAA consists of three functions:

| Function | Purpose |
|---|---|
| **Authentication** | Verifies the identity of a user attempting to access the device |
| **Authorization** | Determines which services and commands an authenticated user is permitted to use |
| **Accounting** | Records user activities, such as login sessions and executed commands |

Cisco IOS XE supports local AAA services and centralized authentication servers using **TACACS+** and **RADIUS**.

This section covers AAA method lists, local and centralized authentication, EXEC and command authorization, accounting, server groups, fallback mechanisms, and troubleshooting.

---

## How CoPP and AAA Work Together

CoPP and AAA protect different aspects of device operation.

For example, when an administrator connects to a router using SSH:

1. **CoPP** can limit the rate of SSH traffic reaching the device's CPU.
2. **AAA authentication** verifies the administrator's identity.
3. **AAA authorization** determines which commands the administrator is permitted to execute.
4. **AAA accounting** can record the administrator's session and activities.

Together, these mechanisms help protect the device's availability and restrict administrative access to authorized users.