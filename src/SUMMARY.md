- [README](README.md)

# 1.0 Network Infrastructure

- [1.2 Routing Concepts](routing-concepts/overview.md)

  - [1.2.d VRF-Lite](vrf-lite/1-overview.md)

  - [1.2.e VRF-Aware Routing with BGP, EIGRP, OSPF, and Static](vrf-lite/vrf-aware-routing/1-overview.md)
    - [VRF-Aware Static Routing](vrf-lite/vrf-aware-routing/2-static-routing.md)
    - [VRF-Aware OSPFv2](vrf-lite/vrf-aware-routing/3-ospfv2.md)
    - [VRF-Aware OSPFv3](vrf-lite/vrf-aware-routing/4-ospfv3.md)
    - [VRF-Aware EIGRP](vrf-lite/vrf-aware-routing/5-eigrp.md)
    - [VRF-Aware BGP](vrf-lite/vrf-aware-routing/6-bgp.md)
    - [VRF-Aware Services](vrf-lite/vrf-aware-routing/7-services.md)

  - [1.2.f Route Leaking Between VRFs Using Route Maps and VASI](vrf-lite/route-leaking/1-overview.md)
    - [Static VRF Route Leaking](vrf-lite/route-leaking/2-static-route-leaking.md)
    - [Global-to-VRF Static Leaking](vrf-lite/route-leaking/3-global-vrf-static-leaking.md)
    - [Route-Target Import/Export](vrf-lite/route-leaking/4-route-target-import-export.md)
    - [Import and Export Maps](vrf-lite/route-leaking/5-import-export-maps.md)
    - [Global-to-VRF BGP Prefix Import](vrf-lite/route-leaking/6-global-vrf-bgp-prefix-import.md)
    - [Route Replication](vrf-lite/route-leaking/7-route-replication.md)
    - [VASI](vrf-lite/route-leaking/8-vasi.md)


- [1.6 Multicast](multicast/overview.md)
  - [IPv4 Multicast Addressing](multicast/ipv4-multicast-addressing.md)
  - [Local Network Control Multicast](multicast/local-network-control.md)
  - [IPv4 Multicast MAC Addresses](multicast/multicast-mac-addresses.md)
  - [Multicast Distribution Trees](multicast/distribution-trees.md)
  - [Multicast Routing State](multicast/multicast-routing-state.md)

  - [1.6.a Layer 2 Multicast](multicast/layer2/overview.md)

    - [1.6.a (i) IGMPv2, IGMPv3](multicast/layer2/igmp/overview.md)
      - [IGMPv1](multicast/layer2/igmp/igmpv1.md)

      - [IGMPv2](multicast/layer2/igmp/igmpv2.md)
        - [IGMPv2 Message Types](multicast/layer2/igmp/igmpv2-message-types.md)
        - [IGMPv2 Queries](multicast/layer2/igmp/igmpv2-queries.md)
        - [IGMPv2 Querier Election](multicast/layer2/igmp/igmpv2-querier-election.md)
        - [IGMPv2 Membership Reports](multicast/layer2/igmp/igmpv2-reports.md)
        - [IGMPv2 Report Suppression](multicast/layer2/igmp/igmpv2-report-suppression.md)
        - [IGMPv2 Leave Process](multicast/layer2/igmp/igmpv2-leave.md)

      - [IGMPv3](multicast/layer2/igmp/igmpv3.md)
        - [IGMPv3 Queries](multicast/layer2/igmp/igmpv3-queries.md)
        - [IGMPv3 Membership Reports](multicast/layer2/igmp/igmpv3-reports.md)
        - [IGMPv3 Source Filtering](multicast/layer2/igmp/igmpv3-source-filtering.md)
        - [INCLUDE and EXCLUDE Modes](multicast/layer2/igmp/igmpv3-include-exclude.md)
        - [IGMPv3 Group Record Types](multicast/layer2/igmp/igmpv3-record-types.md)
        - [IGMPv3 State Changes](multicast/layer2/igmp/igmpv3-state-changes.md)

    - [1.6.a (ii) IGMP Snooping, PIM Snooping](multicast/layer2/snooping/overview.md)
      - [IGMP Snooping](multicast/layer2/snooping/igmp-snooping.md)
      - [Multicast Listener Ports](multicast/layer2/snooping/listener-ports.md)
      - [Multicast Router Ports](multicast/layer2/snooping/router-ports.md)
      - [IGMP Snooping Report Suppression](multicast/layer2/snooping/report-suppression.md)
      - [IGMP Snooping Immediate Leave](multicast/layer2/snooping/immediate-leave.md)
      - [IGMP Snooping Explicit Tracking](multicast/layer2/snooping/explicit-tracking.md)
      - [Unknown Multicast Forwarding](multicast/layer2/snooping/unknown-multicast.md)
      - [Reserved Multicast Groups](multicast/layer2/snooping/reserved-groups.md)
      - [PIM Snooping](multicast/layer2/snooping/pim-snooping.md)

    - [1.6.a (iii) IGMP Querier](multicast/layer2/igmp-querier/overview.md)
      - [Why an IGMP Querier Is Required](multicast/layer2/igmp-querier/why-querier.md)
      - [IGMP Snooping Querier](multicast/layer2/igmp-querier/snooping-querier.md)
      - [Querier Election and Operation](multicast/layer2/igmp-querier/election-operation.md)
      - [IGMP Querier Configuration](multicast/layer2/igmp-querier/configuration.md)

    - [1.6.a (iv) IGMP Filter](multicast/layer2/igmp-filter/overview.md)
      - [IGMP Profiles](multicast/layer2/igmp-filter/igmp-profiles.md)
      - [Permit and Deny Group Membership](multicast/layer2/igmp-filter/group-filtering.md)
      - [IGMP Maximum Groups](multicast/layer2/igmp-filter/maximum-groups.md)

    - [1.6.a (v) MLD](multicast/layer2/mld/overview.md)
      - [IPv6 Multicast Addressing](multicast/layer2/mld/ipv6-multicast-addressing.md)
      - [MLDv1](multicast/layer2/mld/mldv1.md)
      - [MLDv2](multicast/layer2/mld/mldv2.md)
      - [MLD Queries and Reports](multicast/layer2/mld/queries-reports.md)
      - [MLD Source Filtering](multicast/layer2/mld/source-filtering.md)
      - [MLD Snooping](multicast/layer2/mld/mld-snooping.md)

  - [1.6.b Reverse Path Forwarding Check](multicast/rpf/overview.md)
    - [RPF Interface](multicast/rpf/rpf-interface.md)
    - [RPF Neighbor](multicast/rpf/rpf-neighbor.md)
    - [RPF Lookup](multicast/rpf/rpf-lookup.md)
    - [RPF for (S,G) State](multicast/rpf/source-tree-rpf.md)
    - [RPF for (*,G) State](multicast/rpf/shared-tree-rpf.md)
    - [RPF Tie-Breaking](multicast/rpf/tie-breaking.md)
    - [RPF Failures](multicast/rpf/rpf-failures.md)

  - [1.6.c PIM](multicast/pim/overview.md)
    - [PIM Hello Messages](multicast/pim/hello-messages.md)
    - [PIM Neighbors](multicast/pim/neighbors.md)
    - [PIM Designated Router](multicast/pim/designated-router.md)
    - [PIM Dense Mode](multicast/pim/dense-mode.md)
    - [Bidirectional PIM](multicast/pim/bidirectional-pim.md)

    - [1.6.c (i) Sparse Mode](multicast/pim/sparse-mode/overview.md)
      - [PIM-SM Shared Tree](multicast/pim/sparse-mode/shared-tree.md)
      - [(*,G) State](multicast/pim/sparse-mode/star-g-state.md)
      - [PIM-SM Source Tree](multicast/pim/sparse-mode/source-tree.md)
      - [(S,G) State](multicast/pim/sparse-mode/s-g-state.md)
      - [PIM Join and Prune Messages](multicast/pim/sparse-mode/join-prune.md)
      - [Receiver Join Process](multicast/pim/sparse-mode/receiver-join.md)
      - [Source Registration](multicast/pim/sparse-mode/source-registration.md)
      - [PIM Register Messages](multicast/pim/sparse-mode/register.md)
      - [PIM Register-Stop Messages](multicast/pim/sparse-mode/register-stop.md)
      - [Shortest-Path Tree Switchover](multicast/pim/sparse-mode/spt-switchover.md)
      - [SPT Threshold](multicast/pim/sparse-mode/spt-threshold.md)
      - [PIM Assert](multicast/pim/sparse-mode/assert.md)

    - [1.6.c (ii) Static RP, BSR, Auto-RP](multicast/pim/rp-discovery/overview.md)
      - [Static RP](multicast/pim/rp-discovery/static-rp.md)

      - [BSR](multicast/pim/rp-discovery/bsr/overview.md)
        - [Candidate BSR](multicast/pim/rp-discovery/bsr/candidate-bsr.md)
        - [BSR Election](multicast/pim/rp-discovery/bsr/election.md)
        - [Candidate RP](multicast/pim/rp-discovery/bsr/candidate-rp.md)
        - [Candidate RP Advertisements](multicast/pim/rp-discovery/bsr/candidate-rp-advertisements.md)
        - [Bootstrap Messages](multicast/pim/rp-discovery/bsr/bootstrap-messages.md)
        - [RP-Set](multicast/pim/rp-discovery/bsr/rp-set.md)
        - [BSR Hashing](multicast/pim/rp-discovery/bsr/hashing.md)

      - [Auto-RP](multicast/pim/rp-discovery/auto-rp/overview.md)
        - [Candidate RP Announcements](multicast/pim/rp-discovery/auto-rp/rp-announcements.md)
        - [Mapping Agent](multicast/pim/rp-discovery/auto-rp/mapping-agent.md)
        - [Auto-RP Discovery Messages](multicast/pim/rp-discovery/auto-rp/discovery-messages.md)
        - [Auto-RP Groups](multicast/pim/rp-discovery/auto-rp/groups.md)
        - [Auto-RP Listener](multicast/pim/rp-discovery/auto-rp/listener.md)

    - [1.6.c (iii) Group-to-RP Mapping](multicast/pim/group-to-rp/overview.md)
      - [RP Mapping Database](multicast/pim/group-to-rp/mapping-database.md)
      - [Group Ranges](multicast/pim/group-to-rp/group-ranges.md)
      - [Overlapping RP Mappings](multicast/pim/group-to-rp/overlapping-mappings.md)
      - [RP Mapping Selection and Precedence](multicast/pim/group-to-rp/selection-precedence.md)

    - [1.6.c (iv) Source-Specific Multicast](multicast/pim/ssm/overview.md)
      - [SSM Address Range](multicast/pim/ssm/address-range.md)
      - [ASM vs SSM](multicast/pim/ssm/asm-vs-ssm.md)
      - [IGMPv3 Receiver Signaling](multicast/pim/ssm/igmpv3-receiver-signaling.md)
      - [PIM SSM Forwarding](multicast/pim/ssm/pim-forwarding.md)
      - [SSM Mapping](multicast/pim/ssm/ssm-mapping.md)
      - [Static SSM Mapping](multicast/pim/ssm/static-ssm-mapping.md)
      - [DNS-Based SSM Mapping](multicast/pim/ssm/dns-ssm-mapping.md)

    - [1.6.c (v) Multicast Boundary, RP Announcement Filter](multicast/pim/filtering/overview.md)
      - [Multicast Boundaries](multicast/pim/filtering/multicast-boundary.md)
      - [Administratively Scoped Multicast](multicast/pim/filtering/administrative-scope.md)
      - [Boundary ACLs](multicast/pim/filtering/boundary-acls.md)
      - [RP Announcement Filtering](multicast/pim/filtering/rp-announcement-filter.md)
      - [Auto-RP Announcement Filtering](multicast/pim/filtering/auto-rp-filtering.md)

    - [1.6.c (vi) PIMv6 Anycast RP](multicast/pim/pimv6-anycast-rp/overview.md)
      - [Anycast RP Addressing](multicast/pim/pimv6-anycast-rp/addressing.md)
      - [PIMv6 Anycast RP Operation](multicast/pim/pimv6-anycast-rp/operation.md)
      - [Register Forwarding](multicast/pim/pimv6-anycast-rp/register-forwarding.md)
      - [Anycast RP State Synchronization](multicast/pim/pimv6-anycast-rp/state-synchronization.md)
      - [Anycast RP Failure](multicast/pim/pimv6-anycast-rp/failure.md)
      - [PIMv6 Anycast RP Configuration](multicast/pim/pimv6-anycast-rp/configuration.md)

    - [1.6.c (vii) IPv4 Anycast RP Using MSDP](multicast/pim/msdp-anycast-rp/overview.md)
      - [Anycast RP Design](multicast/pim/msdp-anycast-rp/design.md)
      - [MSDP Overview](multicast/pim/msdp-anycast-rp/msdp-overview.md)
      - [MSDP Peering](multicast/pim/msdp-anycast-rp/peering.md)
      - [Source-Active Messages](multicast/pim/msdp-anycast-rp/sa-messages.md)
      - [SA Cache](multicast/pim/msdp-anycast-rp/sa-cache.md)
      - [MSDP Peer-RPF](multicast/pim/msdp-anycast-rp/peer-rpf.md)
      - [Anycast RP Source Synchronization](multicast/pim/msdp-anycast-rp/source-synchronization.md)
      - [Anycast RP Failure](multicast/pim/msdp-anycast-rp/failure.md)
      - [Anycast RP with MSDP Configuration](multicast/pim/msdp-anycast-rp/configuration.md)

    - [1.6.c (viii) Multicast Multipath](multicast/pim/multipath/overview.md)
      - [ECMP and RPF Selection](multicast/pim/multipath/ecmp-rpf.md)
      - [Multicast Load Splitting](multicast/pim/multipath/load-splitting.md)
      - [Source-Based Hashing](multicast/pim/multipath/source-hash.md)
      - [Source-Group Hashing](multicast/pim/multipath/source-group-hash.md)
      - [Next-Hop-Based Source-Group Hashing](multicast/pim/multipath/next-hop-hash.md)


# 2.0 Software-Defined Infrastructure


# 3.0 Transport Technologies and Solutions

- [3.2 MPLS](mpls/1-overview.md)

  - [3.2.a Operations](mpls/operations.md)

    - [3.2.a (i) Label Stack, LSR, LSP](mpls/2-labels.md)
      - [Basic MPLS Forwarding](mpls/3-basic-mpls-forwarding.md)
      - [PHP, Implicit Null, and Explicit Null](mpls/4-php-implicit-explicit-null.md)

    - [3.2.a (ii) LDP](mpls/5-ldp.md)
      - [LDP Session Protection](mpls/6-ldp-session-protection.md)
      - [LDP Authentication](mpls/7-ldp-authentication.md)
      - [Label Filtering](mpls/8-label-filtering.md)
      - [LDP-IGP Synchronization](mpls/9-ldp-igp-sync.md)

    - [3.2.a (iii) MPLS Ping, MPLS Traceroute](mpls/10-mpls-ping-traceroute.md)

  - [3.2.b L3VPN](mpls-l3vpn/1-overview.md)
    - [VRFs](mpls-l3vpn/2-vrfs.md)
    - [RD and RT](mpls-l3vpn/3-rd-rt.md)
    - [VPN Labels](mpls-l3vpn/5-vpn-labels.md)
    - [L3VPN Packet Forwarding](mpls-l3vpn/6-packet-forwarding.md)
    - [MPLS MTU](mpls-l3vpn/7-mpls-mtu.md)
    - [Route Import and Export](mpls-l3vpn/8-route-import-export.md)
    - [Basic L3VPN Configuration](mpls-l3vpn/9-basic-configuration.md)

    - [3.2.b (i) PE-CE Routing Using BGP](mpls-l3vpn-pe-ce/bgp/1-overview.md)
      - [Basic PE-CE BGP Configuration](mpls-l3vpn-pe-ce/bgp/2-bgp-basic-configuration.md)
      - [BGP AS-Override and Allowas-in](mpls-l3vpn-pe-ce/bgp/3-bgp-as-override-allowas.md)
      - [BGP PE-CE Multihoming](mpls-l3vpn-pe-ce/bgp/4-bgp-multihoming.md)
      - [BGP Site of Origin](mpls-l3vpn-pe-ce/bgp/5-bgp-site-of-origin.md)

    - [3.2.b (ii) Basic MP-BGP VPNv4/VPNv6](mpls-l3vpn/4-mp-bgp-vpnv4.md)

    - [Additional PE-CE Routing](mpls-l3vpn-pe-ce/1-overview.md)
      - [PE-CE Route Redistribution](mpls-l3vpn-pe-ce/2-route-redistribution.md)
      - [PE-CE Static Routing](mpls-l3vpn-pe-ce/3-static-routing.md)

      - [PE-CE OSPF](mpls-l3vpn-pe-ce/ospf/1-overview.md)
        - [OSPF Superbackbone](mpls-l3vpn-pe-ce/ospf/2-ospf-superbackbone.md)
        - [OSPF Domain ID](mpls-l3vpn-pe-ce/ospf/3-ospf-domain-id.md)
        - [OSPF Sham Links](mpls-l3vpn-pe-ce/ospf/4-ospf-sham-links.md)
        - [OSPF Route Types in L3VPN](mpls-l3vpn-pe-ce/ospf/5-ospf-route-types.md)
        - [Down Bit](mpls-l3vpn-pe-ce/ospf/6-ospf-down-bit.md)
        - [Domain Tag](mpls-l3vpn-pe-ce/ospf/7-ospf-domain-tag.md)

      - [PE-CE EIGRP](mpls-l3vpn-pe-ce/eigrp/1-overview.md)
        - [EIGRP Route Types in L3VPN](mpls-l3vpn-pe-ce/eigrp/2-eigrp-route-types.md)
        - [EIGRP Metric Preservation](mpls-l3vpn-pe-ce/eigrp/3-eigrp-metric-preservation.md)
        - [EIGRP Site of Origin](mpls-l3vpn-pe-ce/eigrp/4-eigrp-site-of-origin.md)

    - [Advanced MPLS L3VPN Topics](mpls-l3vpn-advanced/1-overview.md)
      - [VPN Import Path Selection](mpls-l3vpn-advanced/2-import-path.md)
      - [6PE](mpls-l3vpn-advanced/3-6pe.md)
      - [6VPE](mpls-l3vpn-advanced/4-6vpe.md)


# 4.0 Infrastructure Security and Services


# 5.0 Infrastructure Automation and Programmability