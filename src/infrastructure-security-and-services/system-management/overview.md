# 4.3 System Management

System management encompasses the protocols and mechanisms used to access, configure, monitor, and troubleshoot network devices.

Although routing and switching protocols enable a network to forward traffic, system management enables administrators to maintain and operate the network.

System management functions include:

- Accessing devices through local and remote management sessions.
- Transferring configuration files and software images.
- Programmatically retrieving and modifying device configurations.
- Monitoring device status and performance.
- Receiving notifications about network events.
- Recording operational events and configuration changes.
- Troubleshooting device behavior.

## Management Plane

The **management plane** consists of the functions used to manage a network device.

It is distinct from the control plane and data plane:

| Plane | Function | Examples |
|---|---|---|
| Management plane | Device administration and monitoring | SSH, SNMP, NETCONF |
| Control plane | Building and maintaining forwarding information | OSPF, BGP, STP |
| Data plane | Forwarding user traffic | IP routing, Ethernet switching |

Management traffic can be carried over the same network infrastructure as user traffic or through a dedicated management network.

### In-Band and Out-of-Band Management

There are two general approaches to network device management:

- **In-band management:** Management traffic traverses the production network.
- **Out-of-band (OOB) management:** Management traffic uses a separate management path or network.

```text
                In-Band Management

 Administrator ---- Production Network ---- R1
                              |
                           User Traffic


              Out-of-Band Management

 Administrator ---- Management Network ---- R1 (Mgmt)
                                             |
 User Traffic ----- Production Network ---- R1 (Data)
```

In-band management does not require a separate physical management network, but connectivity to a device can be lost if the production network experiences a failure.

Out-of-band management provides an independent path to managed devices, allowing administrators to access them even when production network connectivity is unavailable.

Dedicated management interfaces, management VRFs, and console servers are commonly used as part of management network designs. A management VRF provides logical isolation but does not, by itself, guarantee an independent management path.

## 4.3.a Device Management

Cisco IOS XE supports several methods of accessing and managing network devices.

### Console and VTY

The console provides direct access to a device's CLI without requiring IP connectivity.

Virtual terminal (VTY) lines provide remote CLI access using protocols such as Telnet and SSH.

IOS XE supports various line configuration options:

- Authentication and authorization.
- Permitted transport protocols.
- Access restrictions.
- Session and inactivity timeouts.
- EXEC session behavior.
- Rotary groups for selecting groups of lines through specific TCP ports.

### SSH and SCP

**Secure Shell (SSH)** provides encrypted remote CLI access and is the preferred alternative to Telnet.

**Secure Copy Protocol (SCP)** uses SSH to securely transfer files between devices.

SSH and SCP can be used for:

- Remote device administration.
- Configuration backup and restoration.
- Software image transfers.
- Secure file management.

### RESTCONF and NETCONF

RESTCONF and NETCONF provide programmatic access to network devices using structured, model-driven data.

Both protocols use YANG data models to define the structure of configuration and operational data.

- **NETCONF:** Uses remote procedure calls (RPCs), typically transported over SSH, with XML-encoded messages.
- **RESTCONF:** Provides an HTTP-based interface to YANG-modeled data, supporting XML and JSON encoding.

Unlike traditional CLI-based management, these protocols allow applications to retrieve and modify structured device data without parsing human-readable CLI output.

## 4.3.b SNMP

The **Simple Network Management Protocol (SNMP)** allows network management stations (NMSs) to monitor and manage network devices.

An SNMP manager communicates with SNMP agents running on managed devices.

SNMP provides two primary mechanisms for monitoring devices:

- **Polling:** The SNMP manager requests information from an agent.
- **Notifications:** An SNMP agent sends unsolicited notifications to a manager when specified events occur.

SNMP can be used to monitor:

- Interface status and traffic counters.
- CPU and memory utilization.
- Environmental conditions.
- Device uptime.
- Protocol and neighbor status.

The CCIE EI blueprint covers two versions of SNMP:

- **SNMPv2c:** Uses community strings for access control and does not provide encryption.
- **SNMPv3:** Supports authentication, encryption, and granular access control.

SNMP notifications can be sent as **traps** or **informs**. Informs require acknowledgment from the receiver, whereas traps do not.

## 4.3.c Logging

Logging provides visibility into device operation by recording events that occur on a network device.

IOS XE supports several logging destinations and mechanisms:

- Console logging.
- Monitor logging.
- Buffered logging.
- Persistent logging.
- Remote syslog servers.
- Debug output.
- Configuration change logging.

### Local Logging and Syslog

System messages report events such as interface state changes, routing protocol adjacency changes, authentication failures, and hardware problems.

Each syslog message is associated with a severity level ranging from 0 (Emergency) to 7 (Debugging).

Administrators can independently configure severity thresholds for different logging destinations.

### Debugging

Debug commands provide detailed information about internal device operations, including protocol processing and packet handling.

Conditional debugging restricts debug output based on supported conditions, such as a specific interface or peer.

Debugging should be used carefully because excessive debug output can consume CPU resources and affect device performance.

### Configuration Change Notification and Logging

IOS XE supports recording configuration changes using the configuration archive logging feature.

Configuration change logging can record information such as:

- The configuration commands entered.
- The user responsible for the changes.
- The time at which changes occurred.

Configuration change notifications can also be sent to syslog, providing visibility into administrative activity.

### Timestamps

Timestamps associate system messages and debug output with the time at which events occurred.

Accurate timestamps are important for troubleshooting, event correlation, and auditing, particularly when analyzing logs from multiple devices.

IOS XE supports several timestamp formats and options, including:

- System uptime.
- Date and time.
- Millisecond precision.
- Local time.
- Time zone information.

Network Time Protocol (NTP) is commonly used to synchronize clocks across network devices.

## System Management Protocol Summary

| Protocol | Default Transport and Port | Purpose |
|---|---|---|
| Telnet | TCP 23 | Unencrypted remote CLI access |
| SSH | TCP 22 | Encrypted remote CLI access |
| SCP | TCP 22 | Secure file transfer over SSH |
| NETCONF over SSH | TCP 830 | Model-driven device management |
| RESTCONF over HTTPS | TCP 443 | HTTP-based model-driven device management |
| SNMP | UDP 161 | Device monitoring and management |
| SNMP notifications | UDP 162 | Traps and informs |
| Syslog | UDP 514 | Remote system message logging |

These are the standard or commonly used default ports. Some protocols support alternative transports and configurable port numbers.

## Key Concepts

- The management plane provides device administration, monitoring, and troubleshooting functions.
- In-band management uses the production network, whereas out-of-band management uses a separate management path.
- Console access does not require IP connectivity.
- VTY lines support remote CLI access through protocols such as Telnet and SSH.
- Rotary groups allow incoming connections to be directed to specific groups of lines.
- SSH provides encrypted remote access, and SCP provides secure file transfer over SSH.
- NETCONF and RESTCONF provide model-driven, programmatic device management.
- SNMP supports polling and asynchronous notifications.
- SNMPv3 provides security mechanisms unavailable in SNMPv2c.
- Syslog provides event logging, while debug commands provide more detailed operational information.
- Configuration change logging records administrative configuration activity.
- Accurate, synchronized timestamps facilitate troubleshooting and event correlation.
