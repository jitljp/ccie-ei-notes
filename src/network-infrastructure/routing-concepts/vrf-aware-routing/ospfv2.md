# VRF-Aware OSPFv2

OSPFv2 can run inside a VRF.

In IOS XE, OSPFv2 does this by using a **separate OSPF process per VRF**.

This is different from OSPFv3, EIGRP named mode, and BGP.

```text
OSPFv2:
router ospf 10 vrf CUSTOMER_A
router ospf 20 vrf CUSTOMER_B

OSPFv3:
router ospfv3 10
 address-family ipv4 unicast vrf CUSTOMER_A
 address-family ipv4 unicast vrf CUSTOMER_B

EIGRP named mode:
router eigrp JITL
 address-family ipv4 unicast vrf CUSTOMER_A autonomous-system 100
 address-family ipv4 unicast vrf CUSTOMER_B autonomous-system 100

BGP:
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
 address-family ipv4 vrf CUSTOMER_B
```

OSPFv2 does not use address families for VRF-aware IPv4 routing.

Each VRF gets its own OSPFv2 process.

## Main Idea

A VRF has its own routing table.

An OSPFv2 process associated with a VRF installs routes into that VRF routing table.

Example:

```text
router ospf 10 vrf CUSTOMER_A
```

This creates an OSPFv2 process for `CUSTOMER_A`.

Routes learned by this OSPF process are installed into the `CUSTOMER_A` routing table.

They are not installed into the global routing table.

They are not installed into `CUSTOMER_B`.

Verify with:

```text
show ip route vrf CUSTOMER_A ospf
```

## OSPFv2 Process per VRF

This is the key point.

For OSPFv2, use a different OSPF process for each VRF.

```text
router ospf 10 vrf CUSTOMER_A
router ospf 20 vrf CUSTOMER_B
```

These are separate OSPF instances.

Each one has its own:

```text
Router ID
Interfaces
Neighbors
LSDB
SPF calculation
Routes
```

The same network statement can exist under both processes.

```text
router ospf 10 vrf CUSTOMER_A
 network 10.10.10.0 0.0.0.255 area 0

router ospf 20 vrf CUSTOMER_B
 network 10.10.10.0 0.0.0.255 area 0
```

The first command matches interfaces in `CUSTOMER_A`.

The second command matches interfaces in `CUSTOMER_B`.

The prefixes can be identical because the VRFs are separate.

## Lab Topology

This page uses this topology.

```text
                    R1
        +------------------------------+
        |L0: 1.1.1.1/24  L1: 1.1.1.1/24|
        |   E0/0           E0/1        |
        | CUSTOMER_A     CUSTOMER_B    |
        | 10.10.10.1/24  10.10.10.1/24 |
        +----+----------------+--------+
             |                |
             |                |
       10.10.10.2/24    10.10.10.2/24
            CE1              CE2
             |                |
        192.168.1.0/24   192.168.1.0/24
```

R1 has two VRFs.

```text
CUSTOMER_A
CUSTOMER_B
```

R1 uses the same IP address on both customer-facing interfaces.

```text
E0/0 in CUSTOMER_A: 10.10.10.1/24
E0/1 in CUSTOMER_B: 10.10.10.1/24
```

R1 also uses the same loopback address in both VRFs.

```text
Loopback0 in CUSTOMER_A: 1.1.1.1/24
Loopback1 in CUSTOMER_B: 1.1.1.1/24
```

CE1 and CE2 use the same IP address on their R1-facing interfaces.

```text
CE1: 10.10.10.2/24
CE2: 10.10.10.2/24
```

Both customer sites use the same LAN prefix.

```text
CE1 LAN: 192.168.1.0/24
CE2 LAN: 192.168.1.0/24
```

This would not be possible in a single routing table on R1.

It works because `CUSTOMER_A` and `CUSTOMER_B` are separate routing contexts.

## R1 VRF Configuration

Create the VRFs.

```text
R1(config)# vrf definition CUSTOMER_A
R1(config-vrf)# address-family ipv4

R1(config)# vrf definition CUSTOMER_B
R1(config-vrf)# address-family ipv4
```

Assign the physical interfaces to the VRFs.

```text
R1(config)# interface Ethernet0/0
R1(config-if)# vrf forwarding CUSTOMER_A
R1(config-if)# ip address 10.10.10.1 255.255.255.0
R1(config-if)# no shutdown

R1(config)# interface Ethernet0/1
R1(config-if)# vrf forwarding CUSTOMER_B
R1(config-if)# ip address 10.10.10.1 255.255.255.0
R1(config-if)# no shutdown
```

Assign the loopbacks to the VRFs.

```text
R1(config)# interface Loopback0
R1(config-if)# vrf forwarding CUSTOMER_A
R1(config-if)# ip address 1.1.1.1 255.255.255.0

R1(config)# interface Loopback1
R1(config-if)# vrf forwarding CUSTOMER_B
R1(config-if)# ip address 1.1.1.1 255.255.255.0
```

The same IP address can be used in both VRFs.

Verify the VRF interfaces.

```text
R1#show vrf

Name                             Default RD            Protocols   Interfaces
CUSTOMER_A                       <not set>             ipv4        Et0/0
                                                                   Lo0
CUSTOMER_B                       <not set>             ipv4        Et0/1
                                                                   Lo1
```

## CE Configuration

The CEs are not using VRFs.

```text
CE1(config)# interface Ethernet0/0
CE1(config-if)# ip address 10.10.10.2 255.255.255.0
CE1(config-if)# no shutdown

CE1(config)# interface Loopback0
CE1(config-if)# ip address 192.168.1.1 255.255.255.0
```

```text
CE2(config)# interface Ethernet0/0
CE2(config-if)# ip address 10.10.10.2 255.255.255.0
CE2(config-if)# no shutdown

CE2(config)# interface Loopback0
CE2(config-if)# ip address 192.168.1.1 255.255.255.0
```

## R1 OSPFv2 Configuration

Configure one OSPFv2 process for `CUSTOMER_A`.

```text
R1(config)#router ospf 10 vrf CUSTOMER_A
R1(config-router)# router-id 1.1.1.1
R1(config-router)# capability vrf-lite
R1(config-router)# network 10.10.10.0 0.0.0.255 area 0
R1(config-router)# network 1.1.1.0 0.0.0.255 area 0
```

Configure a separate OSPFv2 process for `CUSTOMER_B`.

```text
R1(config)#router ospf 20 vrf CUSTOMER_B
R1(config-router)#router-id 1.1.1.1
R1(config-router)#capability vrf-lite
R1(config-router)# network 10.10.10.0 0.0.0.255 area 0
R1(config-router)# network 1.1.1.0 0.0.0.255 area 0
```

The two processes use the same router ID in this lab.

That is acceptable here because they are separate OSPF domains in separate VRFs.

In production, using unique router IDs is often clearer for operations and troubleshooting.

The important point is the process separation.

```text
OSPF process 10 -> CUSTOMER_A
OSPF process 20 -> CUSTOMER_B
```

This is not one OSPF process with multiple address families.

It is two separate OSPFv2 processes.

## Why capability vrf-lite Is Used

In a VRF-lite design, the router is using VRFs without an MPLS L3VPN core.

When OSPF runs inside a VRF, IOS XE treats the process as if it is connected to the MPLS VPN superbackbone.

That behavior is useful for MPLS L3VPN PE-CE OSPF.

It can be undesirable in plain VRF-lite.

Use `capability vrf-lite` under the OSPF process to disable the MPLS VPN PE behavior for that process.

Example:

```text
router ospf 10 vrf CUSTOMER_A
 capability vrf-lite
```

Use it under each VRF OSPFv2 process.

```text
router ospf 10 vrf CUSTOMER_A
 capability vrf-lite

router ospf 20 vrf CUSTOMER_B
 capability vrf-lite
```

Without:

```
R1(config-router)#do show ip ospf 10 | i MPLS
 Connected to MPLS VPN Superbackbone, VRF CUSTOMER_A
 ```

 With:

```
R1(config-router)#do sh ip ospf 10 | i MPLS
R1(config-router)#
```

In our simple setup, this command isn't necessary. 

But it's a good practice to enable it, as it can be necessary in more complex topologies.

## CE OSPFv2 Configuration

CE1 uses a normal OSPF process.

```text
CE1(config)#router ospf 1
CE1(config-router)# router-id 192.168.1.1
CE1(config-router)# network 10.10.10.0 0.0.0.255 area 0
CE1(config-router)# network 192.168.1.0 0.0.0.255 area 0
```

CE2 also uses a normal OSPF process.

```text
CE2(config)# router ospf 1
CE2(config-router)# router-id 192.168.1.1
CE2(config-router)# network 10.10.10.0 0.0.0.255 area 0
CE2(config-router)# network 192.168.1.0 0.0.0.255 area 0
```

CE1 and CE2 can use the same OSPF process ID and router ID because they are in separate customer networks.

They do not form an adjacency with each other.

CE1 forms an adjacency with R1 in `CUSTOMER_A`.

CE2 forms an adjacency with R1 in `CUSTOMER_B`.

## OSPF Neighbors

R1 should form one OSPF neighbor in each VRF.

```text
R1# show ip ospf 10 neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
192.168.1.1       1   FULL/BDR        00:00:35    10.10.10.2      Ethernet0/0

R1# show ip ospf 20 neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
192.168.1.1       1   FULL/BDR        00:00:37    10.10.10.2      Ethernet0/1
```

The neighbor ID and neighbor address look the same, but they are different neighbors in different VRFs.

## OSPF Routes in Each VRF

R1 learns CE1's LAN in `CUSTOMER_A`.

```text
R1# show ip route vrf CUSTOMER_A ospf

O     192.168.1.0/24 [110/11] via 10.10.10.2, 00:01:08, Ethernet0/0
```

R1 learns CE2's LAN in `CUSTOMER_B`.

```text
R1# show ip route vrf CUSTOMER_B ospf

O        192.168.1.1 [110/11] via 10.10.10.2, 00:02:05, Ethernet0/1
```

The routes look the same, but they are in different VRF tables.

```text
CUSTOMER_A:
192.168.1.0/24 via 10.10.10.2, Ethernet0/0

CUSTOMER_B:
192.168.1.0/24 via 10.10.10.2, Ethernet0/1
```

`CUSTOMER_A` does not learn `CUSTOMER_B` routes.

`CUSTOMER_B` does not learn `CUSTOMER_A` routes.

OSPFv2 is VRF-aware, but it does not leak routes between VRFs.

## CE Routes to R1 Loopbacks

CE1 learns R1's `CUSTOMER_A` loopback.

```text
CE1# show ip route ospf

O        1.1.1.1 [110/11] via 10.10.10.1, 00:02:07, Ethernet0/0
```

CE2 learns R1's `CUSTOMER_B` loopback.

```text
CE2#show ip route ospf

O        1.1.1.1 [110/11] via 10.10.10.1, 00:03:09, Ethernet0/0
```

The routes look identical on the CE routers.

But they point to different VRF instances on R1.

```text
CE1 -> 1.1.1.1 reaches R1 Loopback0 in CUSTOMER_A
CE2 -> 1.1.1.1 reaches R1 Loopback1 in CUSTOMER_B
```

## Interface-Level OSPF Configuration

Instead of using `network` commands, you can enable OSPF directly on the interfaces.

Example for `CUSTOMER_A`:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip ospf 10 area 0

R1(config)# interface Loopback0
R1(config-if)# ip ospf 10 area 0
```

Example for `CUSTOMER_B`:

```text
R1(config)# interface Ethernet0/1
R1(config-if)# ip ospf 20 area 0

R1(config)# interface Loopback1
R1(config-if)# ip ospf 20 area 0
```

You do not need to specify a VRF in the command, because the interface is already associated with a VRF.

## Network Statement Behavior

The OSPFv2 `network` command matches interfaces in the routing context of the OSPF process.

Example:

```text
router ospf 10 vrf CUSTOMER_A
 network 10.10.10.0 0.0.0.255 area 0
```

This activates OSPF on interfaces in `CUSTOMER_A` with matching IP addresses.

It does not activate OSPF on `Ethernet0/1` in `CUSTOMER_B`, even though `Ethernet0/1` also uses `10.10.10.1/24`.

That interface belongs to a different VRF and a different OSPF process.

For `CUSTOMER_B`, configure its own OSPF process.

```text
router ospf 20 vrf CUSTOMER_B
 network 10.10.10.0 0.0.0.255 area 0
```

The same `network` command can be used.

The process and VRF context make it separate.

## OSPF Router IDs

Each OSPF process has its own router ID.

In this lab, both R1 OSPF processes use the same router ID.

```text
router ospf 10 vrf CUSTOMER_A
 router-id 1.1.1.1

router ospf 20 vrf CUSTOMER_B
 router-id 1.1.1.1
```

This works because the processes are in separate VRFs.

They do not share an LSDB.

They do not form adjacencies with each other.

However, unique router IDs are usually easier to troubleshoot.

Example:

```text
router ospf 10 vrf CUSTOMER_A
 router-id 1.1.1.10

router ospf 20 vrf CUSTOMER_B
 router-id 1.1.1.20
```

The rule is:

```text
Router IDs must be unique inside an OSPF domain.
Separate VRFs are separate OSPF domains.
```

## Redistribution Inside a VRF

Redistribution happens inside the selected VRF.

Example:

```text
R1(config)# ip route vrf CUSTOMER_A 172.16.1.0 255.255.255.0 Null0

R1(config)# router ospf 10 vrf CUSTOMER_A
R1(config-router)# redistribute static subnets
```

This redistributes static routes from `CUSTOMER_A` into OSPF process 10.

It does not redistribute static routes from the global routing table.

It does not redistribute static routes from `CUSTOMER_B`.

OSPFv2 redistribution is process-specific and VRF-specific.

## OSPFv2 Does Not Leak Routes Between VRFs

OSPFv2 running in a VRF learns routes for that VRF.

It does not automatically share them with another VRF.

Example:

```text
OSPF process 10 in CUSTOMER_A learns 192.168.1.0/24.
OSPF process 20 in CUSTOMER_B also learns 192.168.1.0/24.
```

These are separate routes in separate routing tables.

```text
show ip route vrf CUSTOMER_A 192.168.1.0
show ip route vrf CUSTOMER_B 192.168.1.0
```

There is no automatic exchange between the two processes.

To share routes between VRFs, configure route leaking.

Route leaking is covered in a separate section.

## Troubleshooting

### OSPF Was Configured Globally

This is a global OSPF process.

```text
router ospf 10
 network 10.10.10.0 0.0.0.255 area 0
```

It does not run inside `CUSTOMER_A`.

For a VRF, specify the VRF in the OSPF process.

```text
router ospf 10 vrf CUSTOMER_A
 network 10.10.10.0 0.0.0.255 area 0
```

> Once an OSPF process has been configured in a VRF (`router ospf 10 vrf CUSTOMER_A`),
> you don't have to specify the VRF to configure the process again.
>
> You can just use `router ospf 10`.

### capability vrf-lite Is Missing

When using OSPFv2 with VRF-lite, include `capability vrf-lite` under the OSPF process.

```text
router ospf 10 vrf CUSTOMER_A
 capability vrf-lite
```

Without it, IOS XE can show OSPF behavior related to the MPLS VPN superbackbone, which is usually not desirable
when not actually using MPLS L3VPN.

## Key Points

- OSPFv2 can run inside a VRF.
- IOS XE OSPFv2 uses a separate OSPF process per VRF.
- OSPFv2 does not use address families for VRF-aware IPv4 routing.
- Use `router ospf <process-id> vrf <vrf-name>`.
- Each OSPFv2 VRF process has its own neighbors, LSDB, SPF calculation, and routes.
- The same prefixes can exist in multiple OSPFv2 VRF processes.
- The same `network` command can be used under different VRF OSPF processes.
- Use `capability vrf-lite` under OSPFv2 VRF processes in VRF-lite designs.
- OSPF routes are installed into the associated VRF routing table.
- OSPFv2 does not leak routes between VRFs by itself.
- Use VRF-aware verification commands when testing.
