- [README](README.md)

# 1.0 Network Infrastructure

- [1.2 Routing Concepts](network-infrastructure/routing-concepts/overview.md)

- [1.2.d VRF-Lite](network-infrastructure/routing-concepts/vrf-lite/overview.md)
  - [VRF-Lite Inter-Router Connectivity](network-infrastructure/routing-concepts/vrf-lite/inter-router.md)
  - [EVN](network-infrastructure/routing-concepts/vrf-lite/evn.md)

  - [1.2.e VRF-Aware Routing with BGP, EIGRP, OSPF, and Static](network-infrastructure/routing-concepts/vrf-aware-routing/overview.md)
    - [VRF-Aware Static Routing](network-infrastructure/routing-concepts/vrf-aware-routing/static-routing.md)
    - [VRF-Aware OSPFv2](network-infrastructure/routing-concepts/vrf-aware-routing/ospfv2.md)
    - [VRF-Aware OSPFv3](network-infrastructure/routing-concepts/vrf-aware-routing/ospfv3.md)
    - [VRF-Aware EIGRP](network-infrastructure/routing-concepts/vrf-aware-routing/eigrp.md)
    - [VRF-Aware BGP](network-infrastructure/routing-concepts/vrf-aware-routing/bgp.md)
    - [VRF-Aware Services](network-infrastructure/routing-concepts/vrf-aware-routing/services.md)

  - [1.2.f Route Leaking Between VRFs Using Route Maps and VASI](network-infrastructure/routing-concepts/route-leaking/overview.md)
    - [Static VRF Route Leaking](network-infrastructure/routing-concepts/route-leaking/static-route-leaking.md)
    - [Route-Target Import/Export](network-infrastructure/routing-concepts/route-leaking/route-target-import-export.md)
    - [Import and Export Maps](network-infrastructure/routing-concepts/route-leaking/import-export-maps.md)
    - [Global-to-VRF Import and VRF-to-Global Export](network-infrastructure/routing-concepts/route-leaking/global-vrf-bgp-prefix-import.md)
    - [Route Replication](network-infrastructure/routing-concepts/route-leaking/route-replication.md)
    - [VASI](network-infrastructure/routing-concepts/route-leaking/vasi.md)

- [1.6 Multicast](network-infrastructure/multicast/overview.md)
  - [IPv4 Multicast Addressing](network-infrastructure/multicast/ipv4-multicast-addressing.md)
  - [Local Network Control Multicast](network-infrastructure/multicast/local-network-control.md)
  - [IPv4 Multicast MAC Addresses](network-infrastructure/multicast/multicast-mac-addresses.md)
  - [Multicast Distribution Trees](network-infrastructure/multicast/distribution-trees.md)
  - [Multicast Routing State](network-infrastructure/multicast/multicast-routing-state.md)

  - [1.6.a Layer 2 Multicast](network-infrastructure/multicast/layer2/overview.md)
    - [1.6.a (i) IGMPv2, IGMPv3](network-infrastructure/multicast/layer2/igmp/overview.md)
      - [IGMPv1](network-infrastructure/multicast/layer2/igmp/igmpv1.md)

      - [IGMPv2](network-infrastructure/multicast/layer2/igmp/igmpv2.md)
        - [IGMPv2 Message Types](network-infrastructure/multicast/layer2/igmp/igmpv2-message-types.md)
        - [IGMPv2 Queries](network-infrastructure/multicast/layer2/igmp/igmpv2-queries.md)
        - [IGMPv2 Membership Reports](network-infrastructure/multicast/layer2/igmp/igmpv2-reports.md)
        - [IGMPv2 Report Suppression](network-infrastructure/multicast/layer2/igmp/igmpv2-report-suppression.md)
        - [IGMPv2 Leave Process](network-infrastructure/multicast/layer2/igmp/igmpv2-leave.md)
        - [IGMPv1/IGMPv2 Compatibility](network-infrastructure/multicast/layer2/igmp/igmpv2-compatibility.md)

      - [IGMPv3](network-infrastructure/multicast/layer2/igmp/igmpv3.md)
        - [IGMPv3 Queries](network-infrastructure/multicast/layer2/igmp/igmpv3-queries.md)
        - [IGMPv3 Membership Reports](network-infrastructure/multicast/layer2/igmp/igmpv3-reports.md)
        - [IGMPv3 Source Filtering and INCLUDE/EXCLUDE Modes](network-infrastructure/multicast/layer2/igmp/igmpv3-source-filtering.md)
        - [IGMPv3 Group Record Types](network-infrastructure/multicast/layer2/igmp/igmpv3-record-types.md)
        - [IGMPv3 State Changes](network-infrastructure/multicast/layer2/igmp/igmpv3-state-changes.md)

    - [1.6.a (ii) IGMP Snooping, PIM Snooping](network-infrastructure/multicast/layer2/snooping/overview.md)
      - [IGMP Snooping](network-infrastructure/multicast/layer2/snooping/igmp-snooping.md)
      - [Multicast Listener Ports](network-infrastructure/multicast/layer2/snooping/listener-ports.md)
      - [Multicast Router Ports](network-infrastructure/multicast/layer2/snooping/router-ports.md)
      - [IGMP Snooping Report Suppression](network-infrastructure/multicast/layer2/snooping/report-suppression.md)
      - [IGMP Snooping Immediate Leave](network-infrastructure/multicast/layer2/snooping/immediate-leave.md)
      - [IGMP Snooping Explicit Tracking](network-infrastructure/multicast/layer2/snooping/explicit-tracking.md)
      - [Unknown Multicast Forwarding](network-infrastructure/multicast/layer2/snooping/unknown-multicast.md)
      - [Reserved Multicast Groups](network-infrastructure/multicast/layer2/snooping/reserved-groups.md)
      - [PIM Snooping](network-infrastructure/multicast/layer2/snooping/pim-snooping.md)

    - [1.6.a (iii) IGMP Querier](network-infrastructure/multicast/layer2/igmp-querier/overview.md)
      - [IGMP Querier Operation and Configuration](network-infrastructure/multicast/layer2/igmp-querier/querier-operation.md)
      - [IGMP Snooping Querier](network-infrastructure/multicast/layer2/igmp-querier/snooping-querier.md)

    - [1.6.a (iv) IGMP Filter](network-infrastructure/multicast/layer2/igmp-filter/overview.md)
      - [IGMP Profiles and Filtering](network-infrastructure/multicast/layer2/igmp-filter/igmp-profiles.md)
      - [IGMP Maximum Groups and Throttling](network-infrastructure/multicast/layer2/igmp-filter/maximum-groups.md)
      - [IGMP Filtering on Routed Interfaces](network-infrastructure/multicast/layer2/igmp-filter/routed-interfaces.md)

    - [1.6.a (v) MLD](network-infrastructure/multicast/layer2/mld/overview.md)
      - [IPv6 Multicast Addressing](network-infrastructure/multicast/layer2/mld/ipv6-multicast-addressing.md)
      - [MLDv1](network-infrastructure/multicast/layer2/mld/mldv1.md)
      - [MLDv2](network-infrastructure/multicast/layer2/mld/mldv2.md)
      - [MLDv2 Source Filtering](network-infrastructure/multicast/layer2/mld/source-filtering.md)
      - [MLD Querier Operation](network-infrastructure/multicast/layer2/mld/querier.md)
      - [MLD Filtering and Limits](network-infrastructure/multicast/layer2/mld/filtering.md)
      - [MLD Snooping](network-infrastructure/multicast/layer2/mld/mld-snooping.md)

  - [1.6.b Reverse Path Forwarding Check](network-infrastructure/multicast/rpf/overview.md)
    - [RPF Interface and Neighbor](network-infrastructure/multicast/rpf/rpf-interface-neighbor.md)
    - [RPF Lookup and Tie-Breaking](network-infrastructure/multicast/rpf/rpf-lookup.md)
    - [Static Multicast Routes](network-infrastructure/multicast/rpf/static-mroutes.md)
    - [RPF for Source and Shared Trees](network-infrastructure/multicast/rpf/tree-rpf.md)
    - [RPF Failures](network-infrastructure/multicast/rpf/rpf-failures.md)

  - [1.6.c PIM](network-infrastructure/multicast/pim/overview.md)
    - [PIM Hello Messages and Neighbors](network-infrastructure/multicast/pim/hello-neighbors.md)
    - [PIM Designated Router](network-infrastructure/multicast/pim/designated-router.md)
    - [PIM Assert](network-infrastructure/multicast/pim/assert.md)
    - [PIM Dense Mode](network-infrastructure/multicast/pim/dense-mode.md)
    - [Bidirectional PIM](network-infrastructure/multicast/pim/bidirectional-pim.md)

    - [1.6.c (i) Sparse Mode](network-infrastructure/multicast/pim/sparse-mode/overview.md)
      - [PIM-SM Shared Tree and (*,G) State](network-infrastructure/multicast/pim/sparse-mode/shared-tree.md)
      - [PIM-SM Source Tree and (S,G) State](network-infrastructure/multicast/pim/sparse-mode/source-tree.md)
      - [PIM Join and Prune Messages](network-infrastructure/multicast/pim/sparse-mode/join-prune.md)
      - [Receiver Join Process](network-infrastructure/multicast/pim/sparse-mode/receiver-join.md)
      - [Source Registration](network-infrastructure/multicast/pim/sparse-mode/source-registration.md)
      - [Shortest-Path Tree Switchover](network-infrastructure/multicast/pim/sparse-mode/spt-switchover.md)

    - [1.6.c (ii) Static RP, BSR, Auto-RP](network-infrastructure/multicast/pim/rp-discovery/overview.md)
      - [Static RP](network-infrastructure/multicast/pim/rp-discovery/static-rp.md)

      - [BSR](network-infrastructure/multicast/pim/rp-discovery/bsr/overview.md)
        - [Candidate BSR and BSR Election](network-infrastructure/multicast/pim/rp-discovery/bsr/candidate-bsr.md)
        - [Candidate RP and RP Advertisements](network-infrastructure/multicast/pim/rp-discovery/bsr/candidate-rp.md)
        - [Bootstrap Messages and RP-Set](network-infrastructure/multicast/pim/rp-discovery/bsr/bootstrap-messages.md)
        - [BSR Hashing](network-infrastructure/multicast/pim/rp-discovery/bsr/hashing.md)

      - [Auto-RP](network-infrastructure/multicast/pim/rp-discovery/auto-rp/overview.md)
        - [Candidate RP Announcements](network-infrastructure/multicast/pim/rp-discovery/auto-rp/rp-announcements.md)
        - [Mapping Agent and Discovery Messages](network-infrastructure/multicast/pim/rp-discovery/auto-rp/mapping-agent.md)
        - [Auto-RP Groups and Auto-RP Listener](network-infrastructure/multicast/pim/rp-discovery/auto-rp/groups.md)
	  - [Command Summary](network-infrastructure/multicast/pim/rp-discovery/commands.md)

    - [1.6.c (iii) Group-to-RP Mapping](network-infrastructure/multicast/pim/group-to-rp/overview.md)
      - [RP Mapping Database and Group Ranges](network-infrastructure/multicast/pim/group-to-rp/mapping-database.md)
      - [RP Mapping Selection and Precedence](network-infrastructure/multicast/pim/group-to-rp/selection-precedence.md)
      - [IPv6 Embedded RP](network-infrastructure/multicast/pim/group-to-rp/ipv6-embedded-rp.md)

    - [1.6.c (iv) Source-Specific Multicast](network-infrastructure/multicast/pim/ssm/overview.md)
      - [ASM vs SSM and SSM Address Ranges](network-infrastructure/multicast/pim/ssm/asm-vs-ssm.md)
      - [SSM Receiver Signaling and PIM Forwarding](network-infrastructure/multicast/pim/ssm/pim-forwarding.md)
      - [SSM Mapping](network-infrastructure/multicast/pim/ssm/ssm-mapping.md)
      - [Static and DNS-Based SSM Mapping](network-infrastructure/multicast/pim/ssm/static-ssm-mapping.md)

    - [1.6.c (v) Multicast Boundary, RP Announcement Filter](network-infrastructure/multicast/pim/filtering/overview.md)
      - [Multicast Boundaries and Administrative Scope](network-infrastructure/multicast/pim/filtering/multicast-boundary.md)
      - [Multicast Group-Range Filtering](network-infrastructure/multicast/pim/filtering/group-range.md)
      - [Auto-RP Announcement Filtering](network-infrastructure/multicast/pim/filtering/rp-announcement-filter.md)
      - [PIM RP Acceptance Filtering](network-infrastructure/multicast/pim/filtering/accept-rp.md)

    - [1.6.c (vi) PIMv6 Anycast RP](network-infrastructure/multicast/pim/pimv6-anycast-rp/overview.md)
      - [Anycast RP Addressing and Operation](network-infrastructure/multicast/pim/pimv6-anycast-rp/operation.md)
      - [PIM Register Replication and Source Discovery](network-infrastructure/multicast/pim/pimv6-anycast-rp/register-forwarding.md)
      - [Anycast RP Configuration and Failover](network-infrastructure/multicast/pim/pimv6-anycast-rp/configuration.md)

    - [1.6.c (vii) IPv4 Anycast RP Using MSDP](network-infrastructure/multicast/pim/msdp-anycast-rp/overview.md)
      - [Anycast RP Addressing and MSDP Operation](network-infrastructure/multicast/pim/msdp-anycast-rp/msdp-overview.md)
      - [MSDP Peering and Originator ID](network-infrastructure/multicast/pim/msdp-anycast-rp/peering.md)
      - [Source-Active Messages and SA Cache](network-infrastructure/multicast/pim/msdp-anycast-rp/sa-messages.md)
      - [MSDP Peer-RPF and Mesh Groups](network-infrastructure/multicast/pim/msdp-anycast-rp/peer-rpf.md)

    - [1.6.c (viii) Multicast Multipath](network-infrastructure/multicast/pim/multipath/overview.md)
      - [ECMP and Multicast Load Splitting](network-infrastructure/multicast/pim/multipath/ecmp-rpf.md)
      - [S-Hash and Basic S-G Hashing](network-infrastructure/multicast/pim/multipath/source-hash.md)
      - [Next-Hop-Based S-G Hashing](network-infrastructure/multicast/pim/multipath/next-hop-hash.md)

# 2.0 Software-Defined Infrastructure


# 3.0 Transport Technologies and Solutions

- [3.1 Static Point-to-Point GRE Tunnels](transport-technologies-and-solutions/gre/overview.md)
  - [GRE Encapsulation and Packet Forwarding](transport-technologies-and-solutions/gre/encapsulation.md)
  - [Point-to-Point GRE Configuration](transport-technologies-and-solutions/gre/configuration.md)
  - [GRE Tunnels for IPv6 over IPv4](transport-technologies-and-solutions/gre/ipv6-over-ipv4.md)
  - [Routing over GRE Tunnels](transport-technologies-and-solutions/gre/routing.md)
  - [GRE Tunnel State and Keepalives](transport-technologies-and-solutions/gre/keepalives.md)
  - [GRE MTU, Fragmentation, and MSS](transport-technologies-and-solutions/gre/mtu-mss.md)
  - [GRE Tunnels and VRFs](transport-technologies-and-solutions/gre/vrf.md)

- [3.2 MPLS](transport-technologies-and-solutions/mpls/overview.md)

  - [3.2.a Operations](transport-technologies-and-solutions/mpls/operations/overview.md)

    - [3.2.a (i) Label Stack, LSR, LSP](transport-technologies-and-solutions/mpls/operations/labels.md)
      - [Basic MPLS Forwarding](transport-technologies-and-solutions/mpls/operations/basic-mpls-forwarding.md)
      - [PHP, Implicit Null, and Explicit Null](transport-technologies-and-solutions/mpls/operations/php-implicit-explicit-null.md)

    - [3.2.a (ii) LDP](transport-technologies-and-solutions/mpls/operations/ldp.md)
      - [LDP Session Protection](transport-technologies-and-solutions/mpls/operations/ldp-session-protection.md)
      - [LDP Authentication](transport-technologies-and-solutions/mpls/operations/ldp-authentication.md)
      - [Label Filtering](transport-technologies-and-solutions/mpls/operations/label-filtering.md)
      - [LDP-IGP Synchronization](transport-technologies-and-solutions/mpls/operations/ldp-igp-sync.md)

    - [3.2.a (iii) MPLS Ping, MPLS Traceroute](transport-technologies-and-solutions/mpls/operations/mpls-ping-traceroute.md)

  - [3.2.b L3VPN](transport-technologies-and-solutions/mpls/l3vpn/overview.md)
    - [VRFs](transport-technologies-and-solutions/mpls/l3vpn/vrfs.md)
    - [RD and RT](transport-technologies-and-solutions/mpls/l3vpn/rd-rt.md)
    - [VPN Labels](transport-technologies-and-solutions/mpls/l3vpn/vpn-labels.md)
    - [L3VPN Packet Forwarding](transport-technologies-and-solutions/mpls/l3vpn/packet-forwarding.md)
    - [MPLS MTU](transport-technologies-and-solutions/mpls/l3vpn/mpls-mtu.md)
    - [Route Import and Export](transport-technologies-and-solutions/mpls/l3vpn/route-import-export.md)
    - [Basic L3VPN Configuration](transport-technologies-and-solutions/mpls/l3vpn/basic-configuration.md)

    - [3.2.b (i) PE-CE Routing Using BGP](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/bgp/overview.md)
      - [Basic PE-CE BGP Configuration](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/bgp/bgp-basic-configuration.md)
      - [BGP AS-Override and Allowas-in](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/bgp/bgp-as-override-allowas.md)
      - [BGP PE-CE Multihoming](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/bgp/bgp-multihoming.md)
      - [BGP Site of Origin](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/bgp/bgp-site-of-origin.md)

    - [3.2.b (ii) Basic MP-BGP VPNv4/VPNv6](transport-technologies-and-solutions/mpls/l3vpn/mp-bgp-vpnv4.md)

    - [Additional PE-CE Routing](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/overview.md)
      - [PE-CE Route Redistribution](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/route-redistribution.md)
      - [PE-CE Static Routing](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/static-routing.md)

      - [PE-CE OSPF](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/ospf/overview.md)
        - [OSPF Superbackbone](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/ospf/ospf-superbackbone.md)
        - [OSPF Domain ID](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/ospf/ospf-domain-id.md)
        - [OSPF Sham Links](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/ospf/ospf-sham-links.md)
        - [OSPF Route Types in L3VPN](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/ospf/ospf-route-types.md)
        - [Down Bit](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/ospf/ospf-down-bit.md)
        - [Domain Tag](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/ospf/ospf-domain-tag.md)

      - [PE-CE EIGRP](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/eigrp/overview.md)
        - [EIGRP Route Types in L3VPN](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/eigrp/eigrp-route-types.md)
        - [EIGRP Metric Preservation](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/eigrp/eigrp-metric-preservation.md)
        - [EIGRP Site of Origin](transport-technologies-and-solutions/mpls/l3vpn/pe-ce/eigrp/eigrp-site-of-origin.md)

    - [Advanced MPLS L3VPN Topics](transport-technologies-and-solutions/mpls/l3vpn/advanced/overview.md)
      - [VPN Import Path Selection](transport-technologies-and-solutions/mpls/l3vpn/advanced/import-path.md)
      - [6PE](transport-technologies-and-solutions/mpls/l3vpn/advanced/6pe.md)
      - [6VPE](transport-technologies-and-solutions/mpls/l3vpn/advanced/6vpe.md)

- [3.3 DMVPN](transport-technologies-and-solutions/dmvpn/overview.md)
  - [DMVPN Architecture and Components](transport-technologies-and-solutions/dmvpn/architecture.md)
  - [NHRP Fundamentals](transport-technologies-and-solutions/dmvpn/nhrp-fundamentals.md)
  - [DMVPN Phase 1](transport-technologies-and-solutions/dmvpn/phase1.md)
  - [DMVPN Phase 2](transport-technologies-and-solutions/dmvpn/phase2.md)

  - [3.3.a Troubleshoot DMVPN Phase 3 with Dual Hub](transport-technologies-and-solutions/dmvpn/phase3/overview.md)
    - [Phase 3 Operation and Packet Forwarding](transport-technologies-and-solutions/dmvpn/phase3/operation.md)

    - [3.3.a (i) NHRP](transport-technologies-and-solutions/dmvpn/phase3/nhrp/overview.md)
      - [NHRP Redirects and Shortcuts](transport-technologies-and-solutions/dmvpn/phase3/nhrp/redirect-shortcut.md)

    - [Routing over DMVPN Phase 3](transport-technologies-and-solutions/dmvpn/phase3/routing.md)
    - [Dual-Hub DMVPN Topologies](transport-technologies-and-solutions/dmvpn/phase3/dual-hub/topologies.md)
    - [Dual-Hub Phase 3 Configuration](transport-technologies-and-solutions/dmvpn/phase3/dual-hub/configuration.md)
    - [Dual-Hub Failover and Convergence](transport-technologies-and-solutions/dmvpn/phase3/dual-hub/failover.md)

    - [3.3.a (ii) IPsec/IKEv2 Using Preshared Key](transport-technologies-and-solutions/dmvpn/phase3/ipsec/overview.md)
      - [IKEv2 Negotiation and Preshared Key Authentication](transport-technologies-and-solutions/dmvpn/phase3/ipsec/ikev2.md)
      - [IPsec ESP and Tunnel Protection](transport-technologies-and-solutions/dmvpn/phase3/ipsec/esp.md)
      - [IKEv2/IPsec Configuration for DMVPN](transport-technologies-and-solutions/dmvpn/phase3/ipsec/configuration.md)

    - [Complete DMVPN Phase 3 Configuration](transport-technologies-and-solutions/dmvpn/phase3/complete_config.md)


# 4.0 Infrastructure Security and Services

- [4.1 Device Security on Cisco IOS XE](infrastructure-security-and-services/device-security/overview.md)

  - [4.1.a Control Plane Policing and Protection](infrastructure-security-and-services/device-security/control-plane/overview.md)
    - [Control-Plane Traffic and Packet Punt Paths](infrastructure-security-and-services/device-security/control-plane/traffic-paths.md)
    - [CoPP Traffic Classification](infrastructure-security-and-services/device-security/control-plane/classification.md)
    - [CoPP Policing Rates and Actions](infrastructure-security-and-services/device-security/control-plane/policing.md)
    - [CoPP Configuration](infrastructure-security-and-services/device-security/control-plane/copp-configuration.md)
    - [Control Plane Protection and Subinterfaces](infrastructure-security-and-services/device-security/control-plane/cppr.md)
    - [CPPr Port Filtering and Queue Thresholding](infrastructure-security-and-services/device-security/control-plane/port-filtering.md)
    - [Management Plane Protection](infrastructure-security-and-services/device-security/control-plane/mpp.md)
    - [Catalyst 9000 System CoPP](infrastructure-security-and-services/device-security/control-plane/system-copp.md)

  - [4.1.b AAA](infrastructure-security-and-services/device-security/aaa/overview.md)
    - [AAA Method Lists and Processing](infrastructure-security-and-services/device-security/aaa/method-lists.md)
    - [Local Users and Privilege Levels](infrastructure-security-and-services/device-security/aaa/local-users.md)
    - [TACACS+](infrastructure-security-and-services/device-security/aaa/tacacs.md)
    - [RADIUS](infrastructure-security-and-services/device-security/aaa/radius.md)
    - [AAA Server Groups and VRF-Aware Configuration](infrastructure-security-and-services/device-security/aaa/server-groups.md)
    - [Login and Enable Authentication](infrastructure-security-and-services/device-security/aaa/authentication.md)
    - [EXEC Authorization](infrastructure-security-and-services/device-security/aaa/exec-authorization.md)
    - [Command Authorization](infrastructure-security-and-services/device-security/aaa/command-authorization.md)
    - [AAA Accounting](infrastructure-security-and-services/device-security/aaa/accounting.md)
    - [AAA Server Failover and Local Fallback](infrastructure-security-and-services/device-security/aaa/failover.md)
    - [Complete AAA Configuration](infrastructure-security-and-services/device-security/aaa/configuration.md)
	

- [4.2 Network Security](infrastructure-security-and-services/network-security/overview.md)

  - [4.2.a Switch Security Features](infrastructure-security-and-services/network-security/switch-security/overview.md)
    - [4.2.a (i) VACL and PACL](infrastructure-security-and-services/network-security/switch-security/vacl-pacl.md)
    - [4.2.a (ii) Storm Control](infrastructure-security-and-services/network-security/switch-security/storm-control.md)
    - [4.2.a (iii) DHCP Snooping and DHCP Option 82](infrastructure-security-and-services/network-security/switch-security/dhcp-snooping/overview.md)
      - [DHCP Option 82](infrastructure-security-and-services/network-security/switch-security/dhcp-snooping/option82.md)
    - [4.2.a (iv) IP Source Guard](infrastructure-security-and-services/network-security/switch-security/ip-source-guard.md)
    - [4.2.a (v) Dynamic ARP Inspection](infrastructure-security-and-services/network-security/switch-security/dynamic-arp-inspection.md)
    - [4.2.a (vi) Port Security](infrastructure-security-and-services/network-security/switch-security/port-security.md)

    - [Private VLANs](infrastructure-security-and-services/network-security/switch-security/private-vlans/overview.md)
      - [Primary and Secondary VLANs](infrastructure-security-and-services/network-security/switch-security/private-vlans/vlan-types.md)
      - [PVLAN Port Types and Forwarding Rules](infrastructure-security-and-services/network-security/switch-security/private-vlans/port-types.md)
      - [Private VLAN Configuration](infrastructure-security-and-services/network-security/switch-security/private-vlans/configuration.md)
      - [Private VLAN Trunks](infrastructure-security-and-services/network-security/switch-security/private-vlans/trunks.md)
      - [Layer 3 Connectivity with Private VLANs](infrastructure-security-and-services/network-security/switch-security/private-vlans/layer3.md)

  - [4.2.b Router Security Features](infrastructure-security-and-services/network-security/router-security/overview.md)
    - [4.2.b (i) IPv6 Traffic Filters](infrastructure-security-and-services/network-security/router-security/ipv6-traffic-filters.md)
    - [4.2.b (ii) IPv4 Access Control Lists](infrastructure-security-and-services/network-security/router-security/ipv4-acls.md)
    - [4.2.b (iii) Unicast Reverse Path Forwarding](infrastructure-security-and-services/network-security/router-security/urpf.md)


# 5.0 Infrastructure Automation and Programmability