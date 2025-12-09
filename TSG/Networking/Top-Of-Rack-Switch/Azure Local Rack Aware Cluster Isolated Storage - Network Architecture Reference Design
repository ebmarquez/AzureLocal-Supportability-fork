---
name: Azure Local RAC Network Architecture
title: Azure Local Rack Aware Cluster - Network Architecture Reference Design
aliases:
  - RAC Isolated Storage Design
  - Azure Local Network Reference Architecture
topic: Network Architecture
summary: Reference network architecture for Azure Local Rack Aware Cluster with
  isolated storage design
tags:
  - azure-local
  - rack-aware-cluster
  - network-architecture
  - cisco-nexus
  - rdma
  - reference-architecture
  - rac
status: published
project: Azure Local Reference Architecture
type:
  - Documentation
  - Architecture
  - Design
  - Reference
author: Eric Marquez
---

# Azure Local Rack Aware Cluster Isolated Storage - Network Architecture Reference Design

---
![](attachment/3064000a3ac6b5e78a897b83420793e7.png)

## Table of Contents

- [Azure Local Rack Aware Cluster - Network Architecture Reference Design](#azure-local-rack-aware-cluster---network-architecture-reference-design)
	- [Table of Contents](#table-of-contents)
	- [Executive Summary](#executive-summary)
	- [Network Requirements](#network-requirements)
		- [Devices Used in This Configuration](#devices-used-in-this-configuration)
	- [Network Architecture Design](#network-architecture-design)
		- [High-Level Topology](#high-level-topology)
	- [Routing Protocol Design](#routing-protocol-design)
		- [BGP Architecture Overview](#bgp-architecture-overview)
		- [BGP Configuration](#bgp-configuration)
			- [Spine1 Configuration](#spine1-configuration)
			- [Spine2 Configuration](#spine2-configuration)
		- [eBGP Northbound Datacenter](#ebgp-northbound-datacenter)
	- [Network Configuration Details](#network-configuration-details)
		- [Native VLAN Strategy](#native-vlan-strategy)
		- [Spine Management and Compute Configurations](#spine-management-and-compute-configurations)
			- [Spine1](#spine1)
			- [Spine2](#spine2)
	- [HSRP Peer Link Spine1 \& Spine2](#hsrp-peer-link-spine1--spine2)
		- [Leaf Network Configuration](#leaf-network-configuration)
			- [Leaf1 Configuration](#leaf1-configuration)
			- [Leaf2 Configuration](#leaf2-configuration)
			- [Leaf3 Configuration](#leaf3-configuration)
			- [Leaf4 Configuration](#leaf4-configuration)
	- [Storage Network Configuration](#storage-network-configuration)
		- [Architecture Design](#architecture-design)
		- [QOS Policy](#qos-policy)
		- [Storage1 Configuration](#storage1-configuration)
		- [Storage 2 Configuration](#storage-2-configuration)
		- [Storage3 Configuration](#storage3-configuration)
		- [Storage4 Configuration](#storage4-configuration)
	- [Physical Topology \& Cabling](#physical-topology--cabling)
		- [Equipment Summary](#equipment-summary)
		- [Spine Connectivity (Wire Map)](#spine-connectivity-wire-map)
			- [Spine1 to Leaf Connections](#spine1-to-leaf-connections)
			- [Spine2 to Leaf Connections](#spine2-to-leaf-connections)
			- [Spine-to-Spine Connectivity (vPC Peer-Link, vPC Keepalive, iBGP)](#spine-to-spine-connectivity-vpc-peer-link-vpc-keepalive-ibgp)
		- [Leaf Connectivity (Wire Map)](#leaf-connectivity-wire-map)
			- [Room1 Leaf to Node](#room1-leaf-to-node)
			- [Room1 Leaf to Spine](#room1-leaf-to-spine)
			- [Room2 Leaf to Node](#room2-leaf-to-node)
			- [Room2 Leaf to Spine](#room2-leaf-to-spine)
		- [Storage Connectivity (Wire Map)](#storage-connectivity-wire-map)
			- [Room1 Node Storage](#room1-node-storage)
			- [Room1 Storage Room to Room](#room1-storage-room-to-room)
			- [Room2 Node Storage](#room2-node-storage)
			- [Room2 Storage Room to Room](#room2-storage-room-to-room)

---

## Summary

This document describes the reference network architecture for deploying Microsoft Azure Local as a Rack Aware Cluster (RAC) with isolated storage design. The architecture implements a two-tier spine-leaf topology with dedicated isolated storage switches to support RDMA-based Storage Spaces Direct across two physical rooms.

This comprehensive design guide enables network engineers to understand the complete architecture and reproduce the environment with confidence.

For more information on Azure Local Rack Aware Cluster, refer to the [Azure Local Rack Aware Cluster Overview](https://docs.microsoft.com/en-us/azure-stack/hci/concepts-azure-local-rac).

---

## Network Requirements

Azure Local deployments use network intents to logically separate traffic types for cluster operations, workload traffic, and storage replication. Best practices recommend three distinct network intents.

**Management Network Intent** - Provides connectivity for cluster management operations, Windows administrative traffic, and Azure Arc integration. Requires reliable Layer 3 routing to external management systems and cloud services.

**Compute Network Intent** - Supports VM workload traffic, tenant networks, and external connectivity for hosted applications. Requires flexible VLAN configuration to support multiple tenant environments and external network integration.

**Storage Network Intent** - Dedicated isolated Layer 2 domain for RDMA-based Storage Spaces Direct traffic. Must remain completely isolated from other traffic types with no routing to ensure optimal RDMA performance. High-bandwidth cross-room connectivity enables synchronous replication.

> [!Note]
> While separating management and compute traffic into distinct intents provides cleaner traffic isolation and simplified troubleshooting, these two intents can be combined on the same network if needed. However, storage traffic **must** remain isolated on a dedicated RDMA network to ensure optimal performance and reliability.

### Devices Used in This Configuration

| Device Role    | Quantity | Model Example                    | Port Speed   | Minimum Ports | Purpose                                   |
| -------------- | -------- | -------------------------------- | ------------ | ------------- | ----------------------------------------- |
| Spine Switch   | 2        | Cisco Nexus 9332PQ or 93180YC-FX | 25G/40G/100G | 52 ports      | Distribution layer, L3 routing, HSRP      |
| Leaf Switch    | 4        | Cisco Nexus 93180YC-FX           | 10G/25G/100G | 48 ports      | Access layer, server connectivity         |
| Storage Switch | 4        | Cisco Nexus 93180YC-FX           | 10G/25G/100G | 50 ports      | Isolated storage fabric, RDMA (SMB1/SMB2) |

---

## Network Architecture Design

### High-Level Topology

This two-tier spine-leaf design physically separates traffic domains while maintaining operational simplicity for mission-critical workloads spanning two physical rooms.

**Architecture Overview:**

The design employs a two-tier spine-leaf architecture with an isolated storage fabric:

**Spine Layer** - Two vPC-paired switches provide Layer 3 routing, HSRP gateway services, and BGP capabilities for northbound datacenter connectivity. This layer enables cross-room communication for management and compute traffic while eliminating spanning-tree through vPC technology. Spine-leaf connectivity uses aggregated 25G links delivering sufficient bandwidth for TCP/IP-based traffic.

**Leaf Layer** - Four access switches (two per room) connect Azure Local nodes using dual-homed 25G links via Microsoft SET teaming. Management traffic runs untagged (native VLAN), while compute traffic uses tagged VLANs. Ports 1-4 are reserved for node connectivity and ports 47-48 are used for Spine uplink connectivity.

**Storage Layer** - Four isolated storage switches (Storage1-4) form a dedicated RDMA fabric completely separate from the spine-leaf infrastructure. Each node connects via four discrete RDMA NICs (not bonded) across two SMB traffic paths for multipath redundancy. Storage1 and Storage3 carry SMB1 traffic (VLAN 711), while Storage2 and Storage4 carry SMB2 traffic (VLAN 712). Dedicated 100G cross-room port-channels ensure high-bandwidth, low-latency synchronous replication without impacting other traffic.

**Design Rationale:** This architecture delivers complete storage traffic isolation, eliminates spanning-tree complexity through vPC, provides dedicated high-bandwidth cross-room replication, and enables modular scaling. The design balances performance, operational simplicity for enterprise deployments. Detailed VLAN assignments, IP addressing, MTU requirements, and bandwidth specifications appear in the [Network Configuration Details](#network-configuration-details) section.

```
               ┌─────────────────────────────────────┐
               │      Upstream Network/Internet      │
               └───────────────────┬─────────────────┘
                                   │
                    ┌──────eBGP────┴────eBGP─────┐
                    │                            │
            ┌───────▼────────┐          ┌────────▼────────┐
            │   Spine1       │◄──iBGP──►|   Spine2        │
            │   (Primary)    │◄────────►│   (Secondary)   │
            │  HSRP Primary  │  vPC     │  HSRP Secondary │
            │  Gateway IP    │ PeerLink │  Gateway IP     │
            └───────┬────────┘          └────────┬────────┘
                    │ 25G                    25G │
         ┌──────────┼────────────────────────────┼────────┐
    PO11 │     PO12 │                       PO13 │   PO14 │
    ┌────▼────┐ ┌───▼─────┐              ┌───────▼─┐ ┌────▼────┐
    │ Leaf1   │ │ Leaf2   │              │ Leaf3   │ │ Leaf4   │
    │ (Room1) │ │ (Room1) │              │ (Room2) │ │ (Room2) │
    └────┬────┘ └──┬──────┘              └──┬──────┘ └────┬────┘
         │         │                        │             │
    ┌────▼─────────▼─────┐              ┌───▼─────────────▼────┐
    │    Node1-4         │              │    Node5-8           │
    │    (Room1)         │              │    (Room2)           │
    │                    │              │                      │
    │   Mgmt (25G):      │              │   Mgmt (25G):        │
    │   Compute (25G):   │              │   Compute (25G):     │
    │     via Leaf       │              │     via Leaf         │
    └────┬────────┬──────┘              └────┬──────────┬──────┘
         │ 25G    │ 25G                  25G │      25G │
    ┌────▼────┐ ┌─▼───────┐              ┌───▼──────┐ ┌─▼───────┐
    │Storage1 │ │Storage2 │              │Storage3  │ │Storage4 │
    │ (SMB1)  │ │ (SMB2)  │              │ (SMB1)   │ │ (SMB2)  │
    │ (Room1) │ │ (Room1) │              │ (Room2)  │ │ (Room2) │
    └────▲────┘ └────▲────┘              └────▲─────┘ └────▲────┘
         │ 100G      │ 100G                   │ 100G   100G│
         │           │                        │            │
         └───────────┼────────────────────────┘            │
                     └─────────────────────────────────────┘

         Cross-Room Storage Links (PO49: Eth1/51-52, 2x100G each):
         Storage1 (SMB1) ↔ Storage3 (SMB1): PO49 (2×100G)
         Storage2 (SMB2) ↔ Storage4 (SMB2): PO49 (2×100G)
```

---

## Routing Protocol Design

### BGP Architecture Overview

In Azure Local BGP is the primary protocol used to support the environment. BGP was selected due to it's ease of use, flexibility and dynamic capabilities.  As new services become available, BGP is used to integrate  Azure Local and the physical network fabric.  In this BGP configuration, Spine1 and Spine2 are configured as iBGP neighbors and other devices are considered eBGP neighbors.  The iBGP connection uses a port-channel to support dual 25G interfaces between the devices used to form a BGP session and provide VPC heartbeat traffic.  The Spine also peer with a single or multiple core devices which provide a default route for networks not defined on the device route table.  using BGP best practices, a prefix-list is used with the BGP neighbor configurations to control the routes it learns.  

> [!note] 
> Spine devices are not required to be iBGP neighbors.

### BGP Configuration

Both spine switches operate in the same autonomous system (AS 64512) and peer via iBGP for internal route synchronization. The iBGP peering uses a dedicated point-to-point link (10.10.0.8/30) separate from the vPC peer-link. Each spine switch also establishes an independent eBGP session with the upstream datacenter core router (AS 65000) to receive a default route for northbound connectivity. This dual-homed design ensures full redundancy - if either spine loses its uplink connection, the other spine can still provide external connectivity via iBGP route synchronization. Prefix filtering is applied on each eBGP neighbor to accept only the default route (0.0.0.0/0) from the uplink device, preventing unnecessary external routes from entering the Azure Local cluster.

**IP Addressing:**
- **iBGP Peering Network**: 10.10.0.8/30
	- Spine1: 10.10.0.9
	- Spine2: 10.10.0.10
- **Uplink Network (Spine1)**: 10.10.0.0/30
	- Uplink Device: 10.10.0.1
	- Spine1: 10.10.0.2
- **Uplink Network (Spine2)**: 10.10.0.4/30
	- Uplink Device: 10.10.0.5
	- Spine2: 10.10.0.6

**Configuration Elements:**
- **ASN**: 64512 (private ASN for Azure Local cluster)
- **Uplink ASN**: 65000 (datacenter core)
- **Router IDs**: Based on loopback0 addresses (192.168.50.1 and 192.168.50.2)
- **Advertised Networks**: Management (192.168.100.0/24), Compute (192.168.200.0/24), loopback addresses, and peering subnets
- **Prefix Filtering**: Applied to eBGP neighbor on Spine2 to accept only default route from uplink device

#### Spine1 Configuration

```cisco
feature bgp

! Prefix list to accept only default route from upstream datacenter
ip prefix-list DEFAULT-ONLY seq 10 permit 0.0.0.0/0

! Loopback interface for BGP router ID
interface loopback0
  ip address 192.168.50.1/32

! iBGP peering interface
interface Ethernet1/25
  description Spine1-to-iBGP-Peer-Spine2:VPC-Heartbeat
  no switchport
  ip address 10.10.0.9/30
  no shutdown

! eBGP uplink interface
interface Ethernet1/47
  description eBGP-to-Datacenter-Core
  no switchport
  ip address 10.10.0.2/30
  no shutdown

! BGP configuration
router bgp 64512
  router-id 192.168.50.1
  address-family ipv4 unicast
    network 10.10.0.0/30
    network 10.10.0.8/30
    network 192.168.50.1/32
    network 192.168.100.0/24
    network 192.168.200.0/24
  ! iBGP neighbor (Spine2)
  neighbor 10.10.0.10
    remote-as 64512
    description Spine2-to-iBGP-Peer-Spine1
    address-family ipv4 unicast
  ! eBGP neighbor (Datacenter Core Uplink)
  neighbor 10.10.0.1
    remote-as 65000
    description Spine1-to-Datacenter-Core-Uplink
    address-family ipv4 unicast
      prefix-list DEFAULT-ONLY in
```

#### Spine2 Configuration

```cisco
feature bgp

! Prefix list to accept only default route from upstream datacenter
ip prefix-list DEFAULT-ONLY seq 10 permit 0.0.0.0/0

! Loopback interface for BGP router ID
interface loopback0
  ip address 192.168.50.2/32

! iBGP peering interface
interface Ethernet1/25
  description Spine2-to-iBGP-Peer-Spine1:VPC-Heartbeat
  no switchport
  ip address 10.10.0.10/30
  no shutdown

! eBGP uplink interface
interface Ethernet1/47
  description eBGP-to-Datacenter-Core
  no switchport
  ip address 10.10.0.6/30
  no shutdown

! BGP configuration
router bgp 64512
  router-id 192.168.50.2
  address-family ipv4 unicast
    network 10.10.0.4/30
    network 10.10.0.8/30
    network 192.168.50.2/32
    network 192.168.100.0/24
    network 192.168.200.0/24
  ! iBGP neighbor (Spine1)
  neighbor 10.10.0.9
    remote-as 64512
    description Spine2-to-iBGP-Peer-Spine1
    address-family ipv4 unicast
  ! eBGP neighbor (Datacenter Core Uplink)
  neighbor 10.10.0.5
    remote-as 65000
    description Spine2-to-Datacenter-Core-Uplink
    address-family ipv4 unicast
      prefix-list DEFAULT-ONLY in
```

### eBGP Northbound Datacenter

Both spine switches establish eBGP peering sessions with the upstream datacenter core router. The Azure Local cluster (AS 64512) advertises its internal networks (management, compute, and infrastructure subnets) to the datacenter core (AS 65000) via these eBGP sessions. Following BGP best practices, a prefix-list should be applied on the datacenter core's eBGP neighbor configuration to accept only explicitly permitted routes from the Azure Local cluster, protecting the datacenter network from receiving unwanted or invalid route advertisements.

**BGP Prefix-List Strategy**:
The `DEFAULT-ONLY` prefix-list on eBGP peering ensures:
- **Inbound**: Only accept default route (0.0.0.0/0) from datacenter core
- **Purpose**: Prevents Azure Local cluster from receiving full datacenter routing table
- **Benefits**: Simplifies routing, reduces memory, improves convergence
- **Default route usage**: All non-local traffic follows default route to datacenter core

**Outbound Advertisement**:
Spine switches advertise specific prefixes to datacenter:
- Management network: 192.168.100.0/24
- Compute network: 192.168.200.0/24  
- Loopback network: 10.10.0.8/30 (iBGP peering subnet)

---

## Network Configuration Details

### Native VLAN Strategy

**VLAN 99 Security Practice**:
- Used as native VLAN on spine-to-leaf trunk links
- Intentionally unused/unrouted VLAN for security
- Prevents VLAN hopping attacks via native VLAN exploitation
- Never used for production traffic

**VLAN 300 on Node Ports**:
- Management network (VLAN 300) native on leaf-to-node ports
- Allows PXE boot and out-of-band management untagged
- Normal production practice for server access ports

### Spine Management and Compute Configurations

#### Spine1

```cisco
hostname spine1
!
feature interface-vlan
feature hsrp
feature vpc
feature lldp

vlan 1,200,300
vlan 20
  name compute
vlan 300
  name management

! Multiple Spanning Tree Configuration
spanning-tree mode mst
spanning-tree mst configuration
  name AZURE-LOCAL-RAC
  revision 1
  instance 1 vlan 200,300
spanning-tree mst 0-1 priority 4096
spanning-tree loopguard default

vpc domain 1
  peer-switch
  role priority 1
  peer-keepalive destination 10.10.0.10 source 10.10.0.9 vrf default
  delay restore 150
  peer-gateway
  auto-recovery

interface Vlan200
  description Compute-Intent
  no shutdown
  mtu 9216
  no ip redirects
  ip address 192.168.200.2/24
  no ipv6 redirects
  hsrp version 2
  hsrp 20 
    priority 150 forwarding-threshold lower 1 upper 150
    ip 192.168.200.1

interface Vlan300
  description Management-Intent
  no shutdown
  mtu 9216
  no ip redirects
  ip address 192.168.100.2/24
  no ipv6 redirects
  hsrp version 2
  hsrp 30 
    priority 140 forwarding-threshold lower 1 upper 140
    ip 192.168.100.1

interface port-channel11
  description Link:PO:11-to-Leaf1
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  vpc 11
!
interface port-channel12
  description Link:PO:12-to-Leaf2
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  vpc 12
!
interface port-channel13
  description Link:PO:13-to-Leaf3
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  vpc 13
!
interface port-channel14
  description Link:PO:14-to-Leaf4
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  vpc 14
!
interface Ethernet1/1-2
  switchport
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 11
!
interface Ethernet1/3-4
  switchport
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 12
!
interface Ethernet1/5-6
  switchport
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 13
!
interface Ethernet1/7-8
  switchport
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 14
```

#### Spine2

```cisco
hostname spine2
!
feature interface-vlan
feature hsrp
feature vpc
feature lldp

vlan 1,200,300
vlan 20
  name compute
vlan 300
  name management

! Multiple Spanning Tree Configuration
spanning-tree mode mst
spanning-tree mst configuration
  name AZURE-LOCAL-RAC
  revision 1
  instance 1 vlan 200,300
spanning-tree mst 0-1 priority 8192
spanning-tree loopguard default

vpc domain 1
  peer-switch
  role priority 1
  peer-keepalive destination 10.10.0.9 source 10.10.0.10 vrf default
  delay restore 150
  peer-gateway
  auto-recovery

interface Vlan200
  description Compute-Intent
  no shutdown
  mtu 9216
  no ip redirects
  ip address 192.168.200.3/24
  no ipv6 redirects
  hsrp version 2
  hsrp 20 
    priority 140 forwarding-threshold lower 1 upper 140
    ip 192.168.200.1

interface Vlan300
  description Management-Intent
  no shutdown
  mtu 9216
  no ip redirects
  ip address 192.168.100.3/24
  no ipv6 redirects
  hsrp version 2
  hsrp 30 
    priority 150 forwarding-threshold lower 1 upper 150
    ip 192.168.100.1

interface port-channel11
  description Link:PO:11-to-Leaf1
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  vpc 11
!
interface port-channel12
  description Link:PO:12-to-Leaf2
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  vpc 12
!
interface port-channel13
  description Link:PO:13-to-Leaf3
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  vpc 13
!
interface port-channel14
  description Link:PO:14-to-Leaf4
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  vpc 14
!
interface Ethernet1/1-2
  switchport
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 11
!
interface Ethernet1/3-4
  switchport
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 12
!
interface Ethernet1/5-6
  switchport
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 13
!
interface Ethernet1/7-8
  switchport
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 14
```

## HSRP Peer Link Spine1 & Spine2

```cisco
interface port-channel101
  description HSRP_VPC:PEERLINK:PO101
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  logging event port link-status
  vpc peer-link
!
interface Ethernet1/49-52
  description HSRP_VPC:PEERLINK:PO101
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  channel-group 101
```

### Leaf Network Configuration

> [!Note]
> **Leaf Switch Design Philosophy:**
> - Leaf switches are **access layer devices** and do NOT configure vPC
> - Each leaf has **two separate port-channels** (one to each spine)
> - **vPC is only on the spine side** - spines present as a single logical switch to leafs
> - Both uplinks remain active via LACP with no spanning-tree blocking
> - MST configuration ensures consistent spanning-tree topology across the fabric

#### Leaf1 Configuration

```cisco
hostname Leaf1
!
feature lacp
feature lldp

! VLANs for management and compute traffic
vlan 1,200,300
vlan 200
  name compute
vlan 300
  name management

! =====================================
! Multiple Spanning Tree Configuration
! =====================================

spanning-tree mode mst
spanning-tree mst configuration
  name AZURE-LOCAL-RAC
  revision 1
  instance 1 vlan 200,300
spanning-tree mst 0-1 priority 61440
spanning-tree loopguard default
spanning-tree portfast bpduguard default

! =====================================
! Port-Channels to Spine Switches
! =====================================

! Port-channel to Spine1
interface port-channel47
  description Uplink-to-Spine1:PO11
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  spanning-tree guard root
  mtu 9216

! Port-channel to Spine2
interface port-channel45
  description Uplink-to-Spine2:PO11
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  spanning-tree guard root
  mtu 9216

! =====================================
! Physical Interfaces to Spine Switches
! =====================================

! Connections to Spine1
interface Ethernet1/47
  description Uplink-to-Spine1:Eth1/1
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 47 mode active

interface Ethernet1/48
  description Uplink-to-Spine1:Eth1/2
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 47 mode active

! Connections to Spine2
interface Ethernet1/45
  description Uplink-to-Spine2:Eth1/1
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 45 mode active

interface Ethernet1/46
  description Uplink-to-Spine2:Eth1/2
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 45 mode active

! =====================================
! Connections to Azure Local Nodes
! =====================================

interface Ethernet1/1
  description AzLocal-Node1:SET-Member1
  switchport mode trunk
  switchport trunk native vlan 300
  switchport trunk allowed vlan 200,300
  spanning-tree port type edge trunk
  mtu 9216

interface Ethernet1/2
  description AzLocal-Node2:SET-Member1
  switchport mode trunk
  switchport trunk native vlan 300
  switchport trunk allowed vlan 200,300
  spanning-tree port type edge trunk
  mtu 9216

interface Ethernet1/3
  description AzLocal-Node3:SET-Member1
  switchport mode trunk
  switchport trunk native vlan 300
  switchport trunk allowed vlan 200,300
  spanning-tree port type edge trunk
  mtu 9216

interface Ethernet1/4
  description AzLocal-Node4:SET-Member1
  switchport mode trunk
  switchport trunk native vlan 300
  switchport trunk allowed vlan 200,300
  spanning-tree port type edge trunk
  mtu 9216

! =====================================
! Unused Ports (Shutdown for Security)
! =====================================

interface Ethernet1/5-44
  description UNUSED
  shutdown

interface Ethernet1/49-52
  description UNUSED
  shutdown
```

> [!Note]
> **MST Configuration Explained:**
> - **MST Instance 0**: Default instance (VLAN 1 and any unmapped VLANs)
> - **MST Instance 1**: Maps management (VLAN 300) and compute (VLAN 200) for efficient convergence
> - **MST Region Name**: "AZURE-LOCAL-RAC" - must match across all switches in the MST region
> - **Priority 61440**: High value ensures leaf switches never become root (spines are root)
> - **vPC on Spine**: Eliminates MST blocking on uplinks; both paths remain active
> - **Edge Ports**: Node-facing ports use `spanning-tree port type edge trunk` with BPDU guard enabled by default

#### Leaf2 Configuration

```cisco
hostname Leaf2
!
feature lacp
feature lldp

vlan 1,200,300
vlan 200
  name compute
vlan 300
  name management

spanning-tree mode mst
spanning-tree mst configuration
  name AZURE-LOCAL-RAC
  revision 1
  instance 1 vlan 200,300
spanning-tree mst 0-1 priority 61440
spanning-tree loopguard default
spanning-tree portfast bpduguard default

interface port-channel47
  description Uplink-to-Spine1:PO12
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  spanning-tree guard root
  mtu 9216

interface port-channel45
  description Uplink-to-Spine2:PO12
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  spanning-tree guard root
  mtu 9216

interface Ethernet1/47
  description Uplink-to-Spine1:Eth1/3
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 47 mode active

interface Ethernet1/48
  description Uplink-to-Spine1:Eth1/4
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 47 mode active

interface Ethernet1/45
  description Uplink-to-Spine2:Eth1/3
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 45 mode active

interface Ethernet1/46
  description Uplink-to-Spine2:Eth1/4
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 45 mode active

interface Ethernet1/1
  description AzLocal-Node1:SET-Member2
  switchport mode trunk
  switchport trunk native vlan 300
  switchport trunk allowed vlan 200,300
  spanning-tree port type edge trunk
  mtu 9216

interface Ethernet1/2
  description AzLocal-Node2:SET-Member2
  switchport mode trunk
  switchport trunk native vlan 300
  switchport trunk allowed vlan 200,300
  spanning-tree port type edge trunk
  mtu 9216

interface Ethernet1/3
  description AzLocal-Node3:SET-Member2
  switchport mode trunk
  switchport trunk native vlan 300
  switchport trunk allowed vlan 200,300
  spanning-tree port type edge trunk
  mtu 9216

interface Ethernet1/4
  description AzLocal-Node4:SET-Member2
  switchport mode trunk
  switchport trunk native vlan 300
  switchport trunk allowed vlan 200,300
  spanning-tree port type edge trunk
  mtu 9216

interface Ethernet1/5-44
  description UNUSED
  shutdown

interface Ethernet1/49-52
  description UNUSED
  shutdown
```

#### Leaf3 Configuration

```cisco
hostname Leaf3
!
feature lacp
feature lldp

vlan 1,200,300
vlan 200
  name compute
vlan 300
  name management

spanning-tree mode mst
spanning-tree mst configuration
  name AZURE-LOCAL-RAC
  revision 1
  instance 1 vlan 200,300
spanning-tree mst 0-1 priority 61440
spanning-tree loopguard default
spanning-tree portfast bpduguard default

interface port-channel47
  description Uplink-to-Spine1:PO13
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  spanning-tree guard root
  mtu 9216

interface port-channel45
  description Uplink-to-Spine2:PO13
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  spanning-tree guard root
  mtu 9216

interface Ethernet1/47
  description Uplink-to-Spine1:Eth1/5
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 47 mode active

interface Ethernet1/48
  description Uplink-to-Spine1:Eth1/6
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 47 mode active

interface Ethernet1/45
  description Uplink-to-Spine2:Eth1/5
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 45 mode active

interface Ethernet1/46
  description Uplink-to-Spine2:Eth1/6
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 45 mode active

interface Ethernet1/1
  description AzLocal-Node5:SET-Member1
  switchport mode trunk
  switchport trunk native vlan 300
  switchport trunk allowed vlan 200,300
  spanning-tree port type edge trunk
  mtu 9216

interface Ethernet1/2
  description AzLocal-Node6:SET-Member1
  switchport mode trunk
  switchport trunk native vlan 300
  switchport trunk allowed vlan 200,300
  spanning-tree port type edge trunk
  mtu 9216

interface Ethernet1/3
  description AzLocal-Node7:SET-Member1
  switchport mode trunk
  switchport trunk native vlan 300
  switchport trunk allowed vlan 200,300
  spanning-tree port type edge trunk
  mtu 9216

interface Ethernet1/4
  description AzLocal-Node8:SET-Member1
  switchport mode trunk
  switchport trunk native vlan 300
  switchport trunk allowed vlan 200,300
  spanning-tree port type edge trunk
  mtu 9216

interface Ethernet1/5-44
  description UNUSED
  shutdown

interface Ethernet1/49-52
  description UNUSED
  shutdown
```

#### Leaf4 Configuration

```cisco
hostname Leaf4
!
feature lacp
feature lldp

vlan 1,200,300
vlan 200
  name compute
vlan 300
  name management

spanning-tree mode mst
spanning-tree mst configuration
  name AZURE-LOCAL-RAC
  revision 1
  instance 1 vlan 200,300
spanning-tree mst 0-1 priority 61440
spanning-tree loopguard default
spanning-tree portfast bpduguard default

interface port-channel47
  description Uplink-to-Spine1:PO14
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  spanning-tree guard root
  mtu 9216

interface port-channel45
  description Uplink-to-Spine2:PO14
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  spanning-tree guard root
  mtu 9216

interface Ethernet1/47
  description Uplink-to-Spine1:Eth1/7
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 47 mode active

interface Ethernet1/48
  description Uplink-to-Spine1:Eth1/8
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 47 mode active

interface Ethernet1/45
  description Uplink-to-Spine2:Eth1/7
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 45 mode active

interface Ethernet1/46
  description Uplink-to-Spine2:Eth1/8
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 200,300
  spanning-tree port type network
  mtu 9216
  channel-group 45 mode active

interface Ethernet1/1
  description AzLocal-Node5:SET-Member2
  switchport mode trunk
  switchport trunk native vlan 300
  switchport trunk allowed vlan 200,300
  spanning-tree port type edge trunk
  mtu 9216

interface Ethernet1/2
  description AzLocal-Node6:SET-Member2
  switchport mode trunk
  switchport trunk native vlan 300
  switchport trunk allowed vlan 200,300
  spanning-tree port type edge trunk
  mtu 9216

interface Ethernet1/3
  description AzLocal-Node7:SET-Member2
  switchport mode trunk
  switchport trunk native vlan 300
  switchport trunk allowed vlan 200,300
  spanning-tree port type edge trunk
  mtu 9216

interface Ethernet1/4
  description AzLocal-Node8:SET-Member2
  switchport mode trunk
  switchport trunk native vlan 300
  switchport trunk allowed vlan 200,300
  spanning-tree port type edge trunk
  mtu 9216

interface Ethernet1/5-44
  description UNUSED
  shutdown

interface Ethernet1/49-52
  description UNUSED
  shutdown
```

## Storage Network Configuration

The isolated storage fabric consists of four dedicated switches operating as two independent pairs. Each pair handles one of the two SMB multipath networks required for Storage Spaces Direct RDMA traffic:

- **Storage1 and Storage3**: SMB1 network (VLAN 711)
- **Storage2 and Storage4**: SMB2 network (VLAN 712)

### Architecture Design

**Node Interface Configuration:**
Each Azure Local node connects to the storage fabric via two discrete RDMA-capable network interfaces. These interfaces are **NOT bonded** - instead, each interface connects to a separate storage switch within the same room to provide multipath redundancy:

**Room 1 Nodes (Node1-4):**
- **Port 1** → Storage1 (SMB1 network, VLAN 711)
- **Port 2** → Storage2 (SMB2 network, VLAN 712)

**Room 2 Nodes (Node5-8):**
- **Port 1** → Storage3 (SMB1 network, VLAN 711)
- **Port 2** → Storage4 (SMB2 network, VLAN 712)

**Network ATC Configuration:**
Azure Local uses Network ATC (Azure Traffic Controller) to automatically configure the host-side storage interfaces with appropriate QoS policies and VLAN tags. Each interface is assigned to only one SMB network (SMB1 or SMB2) to maintain traffic separation.

**Switch Configuration Requirements:**
Each storage switch interface connecting to Azure Local nodes must be configured with matching QoS policies to support RDMA traffic. This design supports both RoCEv2 (RDMA over Converged Ethernet v2) and iWARP transport protocols, both of which utilize the QoS policy configuration. The QoS configuration is critical - **without matching QoS settings on both the host and switch, RDMA traffic can fail**.

**Cross-Room Connectivity:**
Storage switches have **no uplink connections** to the spine-leaf fabric - they are completely isolated. The only inter-switch connections are the dedicated 100G port-channels between paired switches across rooms:

- **Storage1 (Room 1) ↔ Storage3 (Room 2)**: 2x100G port-channel (PO49) for SMB1 cross-room replication
- **Storage2 (Room 1) ↔ Storage4 (Room 2)**: 2x100G port-channel (PO49) for SMB2 cross-room replication

Each cross-room link carries only one VLAN (SMB1 or SMB2), ensuring dedicated high-bandwidth, low-latency paths for synchronous storage replication between rooms.

> [!info] **Additional Resources**
> For detailed information on QoS policies and room-to-room link configuration, refer to:
> 
> - [Reference TOR QoS Policy Configuration](https://github.com/Azure/AzureLocal-Supportability/blob/main/TSG/Networking/Top-Of-Rack-Switch/Reference-TOR-QOS-Policy-Configuration.md)
> - [Azure Local Rack Aware Cluster Room-to-Room Connectivity](https://learn.microsoft.com/en-us/azure/azure-local/concepts/rack-aware-cluster-room-to-room-connectivity?view=azloc-2511)

### QOS Policy

**RDMA Traffic Requirements**:
- **CoS 3**: Industry standard for RDMA/RoCEv2 traffic
- **PFC (Priority Flow Control)**: Prevents packet drops by signaling congestion
- **ECN (Explicit Congestion Notification)**: Marks packets at 300KB threshold
- **50% bandwidth allocation**: Ensures RDMA gets priority during congestion

**Cluster Traffic**:
- **CoS 7**: Reserved for cluster heartbeat and control plane
- **1% bandwidth guarantee**: Small but guaranteed allocation for critical control traffic

```CiscoNXOS

policy-map type network-qos QOS_NETWORK
  class type network-qos c-8q-nq3
    mtu 9216
    pause pfc-cos 3
  class type network-qos c-8q-nq-default
    mtu 9216
  class type network-qos c-8q-nq7
    mtu 9216
!
class-map type qos match-all RDMA
  match cos 3
class-map type qos match-all CLUSTER
  match cos 7
!
policy-map type qos AZLocal_SERVICES
  class RDMA
    set qos-group 3
  class CLUSTER
    set qos-group 7
!
policy-map type queuing QOS_EGRESS_PORT
  class type queuing c-out-8q-q3
    bandwidth remaining percent 50
    random-detect minimum-threshold 300 kbytes maximum-threshold 300 kbytes drop-probability 100 weight 0 ecn
  class type queuing c-out-8q-q-default
    bandwidth remaining percent 48
  class type queuing c-out-8q-q7
    bandwidth percent 1
  class type queuing c-out-8q-q1
    bandwidth remaining percent 0
  class type queuing c-out-8q-q2
    bandwidth remaining percent 0
  class type queuing c-out-8q-q4
    bandwidth remaining percent 0
  class type queuing c-out-8q-q5
    bandwidth remaining percent 0
  class type queuing c-out-8q-q6
    bandwidth remaining percent 0
!
system qos
  service-policy type queuing output QOS_EGRESS_PORT
  service-policy type network-qos QOS_NETWORK
  
  
spanning-tree mode mst
spanning-tree mst configuration
  name STORAGE-FABRIC
  revision 1
  instance 1 vlan 711,712
spanning-tree mst 0-1 priority 32768
spanning-tree loopguard default
spanning-tree portfast bpduguard default
```

### Storage1 Configuration

```CiscoNXOS
hostname Storage1
!
feature lacp
!
vlan 1,711
vlan 711
  name SMB1

interface Ethernet1/1
  description AzLocal-Node:Storage-SMB1:Port1
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 711
  priority-flow-control mode on send-tlv
  spanning-tree port type edge trunk
  mtu 9216
  no logging event port link-status
  service-policy type qos input AZLocal_SERVICES
!
interface Ethernet1/2
  description AzLocal-Node:Storage-SMB1:Port1
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 711
  priority-flow-control mode on send-tlv
  spanning-tree port type edge trunk
  mtu 9216
  no logging event port link-status
  service-policy type qos input AZLocal_SERVICES
!
interface Ethernet1/3
  description AzLocal-Node:Storage-SMB1:Port1
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 711
  priority-flow-control mode on send-tlv
  spanning-tree port type edge trunk
  mtu 9216
  no logging event port link-status
  service-policy type qos input AZLocal_SERVICES
!
interface Ethernet1/4
  description AzLocal-Node:Storage-SMB1:Port1
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 711
  priority-flow-control mode on send-tlv
  spanning-tree port type edge trunk
  mtu 9216
  no logging event port link-status
  service-policy type qos input AZLocal_SERVICES 
!
interface port-channel49
  description Room-to-Room-Storage3:PO49
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 711
  spanning-tree port type network
  spanning-tree guard root
  mtu 9216
!  
interface Ethernet1/49
  description Room-to-Room-Storage3:Eth1/49
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 711
  priority-flow-control mode on send-tlv
  spanning-tree port type network
  mtu 9216
  channel-group 49 mode active
  service-policy type qos input AZLocal_SERVICES
!
interface Ethernet1/50
  description Room-to-Room-Storage3:Eth1/50
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 711
  priority-flow-control mode on send-tlv
  spanning-tree port type network
  mtu 9216
  channel-group 49 mode active
  service-policy type qos input AZLocal_SERVICES
```

### Storage 2 Configuration

```CiscoNXOS
hostname Storage2
!
feature lacp
!
vlan 1,712
vlan 712
  name SMB2

interface Ethernet1/1
  description AzLocal-Node:Storage-SMB1:Port2
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 712
  priority-flow-control mode on send-tlv
  spanning-tree port type edge trunk
  mtu 9216
  no logging event port link-status
  service-policy type qos input AZLocal_SERVICES
!
interface Ethernet1/2
  description AzLocal-Node:Storage-SMB1:Port2
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 712
  priority-flow-control mode on send-tlv
  spanning-tree port type edge trunk
  mtu 9216
  no logging event port link-status
  service-policy type qos input AZLocal_SERVICES
!
interface Ethernet1/3
  description AzLocal-Node:Storage-SMB1:Port2
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 712
  priority-flow-control mode on send-tlv
  spanning-tree port type edge trunk
  mtu 9216
  no logging event port link-status
  service-policy type qos input AZLocal_SERVICES
!
interface Ethernet1/4
  description AzLocal-Node:Storage-SMB1:Port2
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 712
  priority-flow-control mode on send-tlv
  spanning-tree port type edge trunk
  mtu 9216
  no logging event port link-status
  service-policy type qos input AZLocal_SERVICES 
!
interface port-channel49
  description Room-to-Room-Storage4:PO49
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 712
  spanning-tree port type network
  spanning-tree guard root
  mtu 9216
!  
interface Ethernet1/49
  description Room-to-Room-Storage4:Eth1/49
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 712
  priority-flow-control mode on send-tlv
  spanning-tree port type network
  mtu 9216
  channel-group 49 mode active
  service-policy type qos input AZLocal_SERVICES
!
interface Ethernet1/50
  description Room-to-Room-Storage4:Eth1/50
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 712
  priority-flow-control mode on send-tlv
  spanning-tree port type network
  mtu 9216
  channel-group 49 mode active
  service-policy type qos input AZLocal_SERVICES
```

### Storage3 Configuration

```CiscoNXOS
hostname Storage3
!
feature lacp
!
vlan 1,711
vlan 711
  name SMB1

interface Ethernet1/1
  description AzLocal-Node:Storage-SMB1:Port1
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 711
  priority-flow-control mode on send-tlv
  spanning-tree port type edge trunk
  mtu 9216
  no logging event port link-status
  service-policy type qos input AZLocal_SERVICES
!
interface Ethernet1/2
  description AzLocal-Node:Storage-SMB1:Port1
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 711
  priority-flow-control mode on send-tlv
  spanning-tree port type edge trunk
  mtu 9216
  no logging event port link-status
  service-policy type qos input AZLocal_SERVICES
!
interface Ethernet1/3
  description AzLocal-Node:Storage-SMB1:Port1
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 711
  priority-flow-control mode on send-tlv
  spanning-tree port type edge trunk
  mtu 9216
  no logging event port link-status
  service-policy type qos input AZLocal_SERVICES
!
interface Ethernet1/4
  description AzLocal-Node:Storage-SMB1:Port1
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 711
  priority-flow-control mode on send-tlv
  spanning-tree port type edge trunk
  mtu 9216
  no logging event port link-status
  service-policy type qos input AZLocal_SERVICES 
!
interface port-channel49
  description Room-to-Room-Storage1:PO49
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 711
  spanning-tree port type network
  spanning-tree guard root
  mtu 9216
!  
interface Ethernet1/49
  description Room-to-Room-Storage1:Eth1/49
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 711
  priority-flow-control mode on send-tlv
  spanning-tree port type network
  mtu 9216
  channel-group 49 mode active
  service-policy type qos input AZLocal_SERVICES
!
interface Ethernet1/50
  description Room-to-Room-Storage1:Eth1/50
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 711
  priority-flow-control mode on send-tlv
  spanning-tree port type network
  mtu 9216
  channel-group 49 mode active
  service-policy type qos input AZLocal_SERVICES
```

### Storage4 Configuration

```CiscoNXOS
hostname Storage4
!
feature lacp
!
vlan 1,712
vlan 712
  name SMB2

interface Ethernet1/1
  description AzLocal-Node:Storage-SMB1:Port2
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 712
  priority-flow-control mode on send-tlv
  spanning-tree port type edge trunk
  mtu 9216
  no logging event port link-status
  service-policy type qos input AZLocal_SERVICES
!
interface Ethernet1/2
  description AzLocal-Node:Storage-SMB1:Port2
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 712
  priority-flow-control mode on send-tlv
  spanning-tree port type edge trunk
  mtu 9216
  no logging event port link-status
  service-policy type qos input AZLocal_SERVICES
!
interface Ethernet1/3
  description AzLocal-Node:Storage-SMB1:Port2
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 712
  priority-flow-control mode on send-tlv
  spanning-tree port type edge trunk
  mtu 9216
  no logging event port link-status
  service-policy type qos input AZLocal_SERVICES
!
interface Ethernet1/4
  description AzLocal-Node:Storage-SMB1:Port2
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 712
  priority-flow-control mode on send-tlv
  spanning-tree port type edge trunk
  mtu 9216
  no logging event port link-status
  service-policy type qos input AZLocal_SERVICES 
!
interface port-channel49
  description Room-to-Room-Storage2:PO49
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 712
  spanning-tree port type network
  spanning-tree guard root
  mtu 9216
!  
interface Ethernet1/49
  description Room-to-Room-Storage2:Eth1/49
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 712
  priority-flow-control mode on send-tlv
  spanning-tree port type network
  mtu 9216
  channel-group 49 mode active
  service-policy type qos input AZLocal_SERVICES
!
interface Ethernet1/50
  description Room-to-Room-Storage2:Eth1/50
  switchport mode trunk
  switchport trunk native vlan 99
  switchport trunk allowed vlan 712
  priority-flow-control mode on send-tlv
  spanning-tree port type network
  mtu 9216
  channel-group 49 mode active
  service-policy type qos input AZLocal_SERVICES
```

## Physical Topology & Cabling

### Equipment Summary

| Device Role       | Quantity | Model Example                    | Port Speed   | Purpose                                        |
| ----------------- | -------- | -------------------------------- | ------------ | ---------------------------------------------- |
| Spine Switch      | 2        | Cisco Nexus 9332PQ or 93180YC-FX | 25G/40G/100G | Distribution layer, L3 routing, HSRP           |
| Leaf Switch       | 4        | Cisco Nexus 93180YC-FX           | 10G/25G/100G | Access layer, server connectivity              |
| Storage Switch    | 4        | Cisco Nexus 93180YC-FX           | 10G/25G/100G | Isolated storage fabric, RDMA (SMB1/SMB2)      |
| Azure Local Nodes | 2-8      | Customer-supplied                | 25G          | Compute, management, and storage cluster nodes |

> [!note] 
> - All switches run Cisco NX-OS version 10.x or later
> - Spine switches require higher port density for scale
> - Leaf and Storage switches are typically same model for operational simplicity
> - Node count: 2 nodes per room minimum, 4 nodes per room maximum (8 total)
> - **Spine-to-Leaf links**: 25G (management and compute traffic via TCP/IP)
> - **Node-to-Storage links**: 25G per RDMA NIC (4 NICs per node for storage redundancy)
> - **Cross-room storage links**: 100G (dedicated high-bandwidth replication)

### Spine Connectivity (Wire Map)

#### Spine1 to Leaf Connections

| Source Device | Source Interface | Dest Device | Dest Interface | Speed | Purpose             |
| ------------- | ---------------- | ----------- | -------------- | ----- | ------------------- |
| Spine1        | Eth1/1           | Leaf1       | Eth1/47        | 25G   | vPC to Leaf1 (PO11) |
| Spine1        | Eth1/2           | Leaf1       | Eth1/48        | 25G   | vPC to Leaf1 (PO11) |
| Spine1        | Eth1/3           | Leaf2       | Eth1/47        | 25G   | vPC to Leaf2 (PO12) |
| Spine1        | Eth1/4           | Leaf2       | Eth1/48        | 25G   | vPC to Leaf2 (PO12) |
| Spine1        | Eth1/5           | Leaf3       | Eth1/47        | 25G   | vPC to Leaf3 (PO13) |
| Spine1        | Eth1/6           | Leaf3       | Eth1/48        | 25G   | vPC to Leaf3 (PO13) |
| Spine1        | Eth1/7           | Leaf4       | Eth1/47        | 25G   | vPC to Leaf4 (PO14) |
| Spine1        | Eth1/8           | Leaf4       | Eth1/48        | 25G   | vPC to Leaf4 (PO14) |

#### Spine2 to Leaf Connections

| Source Device | Source Interface | Dest Device | Dest Interface | Speed | Purpose             |
| ------------- | ---------------- | ----------- | -------------- | ----- | ------------------- |
| Spine2        | Eth1/1           | Leaf1       | Eth1/45        | 25G   | vPC to Leaf1 (PO11) |
| Spine2        | Eth1/2           | Leaf1       | Eth1/46        | 25G   | vPC to Leaf1 (PO11) |
| Spine2        | Eth1/3           | Leaf2       | Eth1/45        | 25G   | vPC to Leaf2 (PO12) |
| Spine2        | Eth1/4           | Leaf2       | Eth1/46        | 25G   | vPC to Leaf2 (PO12) |
| Spine2        | Eth1/5           | Leaf3       | Eth1/45        | 25G   | vPC to Leaf3 (PO13) |
| Spine2        | Eth1/6           | Leaf3       | Eth1/46        | 25G   | vPC to Leaf3 (PO13) |
| Spine2        | Eth1/7           | Leaf4       | Eth1/45        | 25G   | vPC to Leaf4 (PO14) |
| Spine2        | Eth1/8           | Leaf4       | Eth1/46        | 25G   | vPC to Leaf4 (PO14) |

**Design Note**: Spine-to-leaf links use 25G connections (2 per spine, per leaf = 50G total uplink bandwidth via vPC). This bandwidth is appropriate for compute and management traffic which uses TCP/IP and can tolerate minor congestion. Storage traffic uses dedicated 25G RDMA fabric per node with isolated switches. **Leaf ports 1-4 are reserved for Hyper-V hosts (Azure Local nodes).**

#### Spine-to-Spine Connectivity (vPC Peer-Link, vPC Keepalive, iBGP)

| Source Device | Source Interface | Dest Device | Dest Interface | Speed | Purpose                    |
| ------------- | ---------------- | ----------- | -------------- | ----- | -------------------------- |
| Spine1        | Eth1/41          | Spine2      | Eth1/41        | 25G   | vPC Keepalive, iBGP (PO50) |
| Spine1        | Eth1/42          | Spine2      | Eth1/42        | 25G   | vPC Keepalive, iBGP (PO50) |
| Spine1        | Eth1/49          | Spine2      | Eth1/49        | 100G  | vPC Peer-Link (PO101)      |
| Spine1        | Eth1/50          | Spine2      | Eth1/50        | 100G  | vPC Peer-Link (PO101)      |
| Spine1        | Eth1/51          | Spine2      | Eth1/51        | 100G  | vPC Peer-Link (PO101)      |
| Spine1        | Eth1/52          | Spine2      | Eth1/52        | 100G  | vPC Peer-Link (PO101)      |

### Leaf Connectivity (Wire Map)

#### Room1 Leaf to Node

| Source Device | Source Interface | Dest Device   | Dest Interface | Speed | Purpose             |
| ------------- | ---------------- | ------------- | -------------- | ----- | ------------------- |
| Leaf1         | Eth1/1           | AzLocal Node1 | Eth1           | 25G   | AzLocal Node1 (SET) |
| Leaf1         | Eth1/2           | AzLocal Node2 | Eth1           | 25G   | AzLocal Node2 (SET) |
| Leaf1         | Eth1/3           | AzLocal Node3 | Eth1           | 25G   | AzLocal Node3 (SET) |
| Leaf1         | Eth1/4           | AzLocal Node4 | Eth1           | 25G   | AzLocal Node4 (SET) |
|               |                  |               |                |       |                     |
| Leaf2         | Eth1/1           | AzLocal Node1 | Eth2           | 25G   | AzLocal Node1 (SET) |
| Leaf2         | Eth1/2           | AzLocal Node2 | Eth2           | 25G   | AzLocal Node2 (SET) |
| Leaf2         | Eth1/3           | AzLocal Node3 | Eth2           | 25G   | AzLocal Node3 (SET) |
| Leaf2         | Eth1/4           | AzLocal Node4 | Eth2           | 25G   | AzLocal Node4 (SET) |

#### Room1 Leaf to Spine

| Source Device | Source Interface | Dest Device | Dest Interface | Speed | Purpose        |
| ------------- | ---------------- | ----------- | -------------- | ----- | -------------- |
| Leaf1         | Eth1/45          | Spine2      | Eth1/1         | 25G   | PO45 to Spine2 |
| Leaf1         | Eth1/46          | Spine2      | Eth1/2         | 25G   | PO45 to Spine2 |
| Leaf1         | Eth1/47          | Spine1      | Eth1/1         | 25G   | PO47 to Spine1 |
| Leaf1         | Eth1/48          | Spine1      | Eth1/2         | 25G   | PO47 to Spine1 |
|               |                  |             |                |       |                |
| Leaf2         | Eth1/45          | Spine2      | Eth1/3         | 25G   | PO45 to Spine2 |
| Leaf2         | Eth1/46          | Spine2      | Eth1/4         | 25G   | PO45 to Spine2 |
| Leaf2         | Eth1/47          | Spine1      | Eth1/3         | 25G   | PO47 to Spine1 |
| Leaf2         | Eth1/48          | Spine1      | Eth1/4         | 25G   | PO47 to Spine1 |

#### Room2 Leaf to Node

| Source Device | Source Interface | Dest Device   | Dest Interface | Speed | Purpose             |
| ------------- | ---------------- | ------------- | -------------- | ----- | ------------------- |
| Leaf3         | Eth1/1           | AzLocal Node5 | Eth1           | 25G   | AzLocal Node5 (SET) |
| Leaf3         | Eth1/2           | AzLocal Node6 | Eth1           | 25G   | AzLocal Node6 (SET) |
| Leaf3         | Eth1/3           | AzLocal Node7 | Eth1           | 25G   | AzLocal Node7 (SET) |
| Leaf3         | Eth1/4           | AzLocal Node8 | Eth1           | 25G   | AzLocal Node8 (SET) |
|               |                  |               |                |       |                     |
| Leaf4         | Eth1/1           | AzLocal Node5 | Eth2           | 25G   | AzLocal Node5 (SET) |
| Leaf4         | Eth1/2           | AzLocal Node6 | Eth2           | 25G   | AzLocal Node6 (SET) |
| Leaf4         | Eth1/3           | AzLocal Node7 | Eth2           | 25G   | AzLocal Node7 (SET) |
| Leaf4         | Eth1/4           | AzLocal Node8 | Eth2           | 25G   | AzLocal Node8 (SET) |

#### Room2 Leaf to Spine

| Source Device | Source Interface | Dest Device | Dest Interface | Speed | Purpose        |
| ------------- | ---------------- | ----------- | -------------- | ----- | -------------- |
| Leaf3         | Eth1/45          | Spine2      | Eth1/5         | 25G   | PO45 to Spine2 |
| Leaf3         | Eth1/46          | Spine2      | Eth1/6         | 25G   | PO45 to Spine2 |
| Leaf3         | Eth1/47          | Spine1      | Eth1/5         | 25G   | PO47 to Spine1 |
| Leaf3         | Eth1/48          | Spine1      | Eth1/6         | 25G   | PO47 to Spine1 |
|               |                  |             |                |       |                |
| Leaf4         | Eth1/45          | Spine2      | Eth1/7         | 25G   | PO45 to Spine2 |
| Leaf4         | Eth1/46          | Spine2      | Eth1/8         | 25G   | PO45 to Spine2 |
| Leaf4         | Eth1/47          | Spine1      | Eth1/7         | 25G   | PO47 to Spine1 |
| Leaf4         | Eth1/48          | Spine1      | Eth1/8         | 25G   | PO47 to Spine1 |

### Storage Connectivity (Wire Map)

#### Room1 Node Storage

| Source Device | Source Interface | Dest Device       | Dest Interface | Speed | Purpose              |
| ------------- | ---------------- | ----------------- | -------------- | ----- | -------------------- |
| Storage1      | Eth1/1           | Azure Local Node1 | Port1          | 25G   | SMB1 Cluster Storage |
| Storage1      | Eth1/2           | Azure Local Node2 | Port1          | 25G   | SMB1 Cluster Storage |
| Storage1      | Eth1/3           | Azure Local Node3 | Port1          | 25G   | SMB1 Cluster Storage |
| Storage1      | Eth1/4           | Azure Local Node4 | Port1          | 25G   | SMB1 Cluster Storage |
|               |                  |                   |                |       |                      |
| Storage2      | Eth1/1           | Azure Local Node1 | Port2          | 25G   | SMB2 Cluster Storage |
| Storage2      | Eth1/2           | Azure Local Node2 | Port2          | 25G   | SMB2 Cluster Storage |
| Storage2      | Eth1/3           | Azure Local Node3 | Port2          | 25G   | SMB2 Cluster Storage |
| Storage2      | Eth1/4           | Azure Local Node4 | Port2          | 25G   | SMB2 Cluster Storage |

#### Room1 Storage Room to Room

| Source Device | Source Interface | Dest Device | Dest Interface | Speed | Purpose              |
| ------------- | ---------------- | ----------- | -------------- | ----- | -------------------- |
| Storage1      | Eth1/49          | Storage3    | Eth1/49        | 100G  | SMB1 Cluster Storage |
| Storage1      | Eth1/50          | Storage3    | Eth1/50        | 100G  | SMB1 Cluster Storage |
|               |                  |             |                |       |                      |
| Storage2      | Eth1/49          | Storage4    | Eth1/49        | 100G  | SMB2 Cluster Storage |
| Storage2      | Eth1/50          | Storage4    | Eth1/50        | 100G  | SMB2 Cluster Storage |

#### Room2 Node Storage

| Source Device | Source Interface | Dest Device       | Dest Interface | Speed | Purpose              |
| ------------- | ---------------- | ----------------- | -------------- | ----- | -------------------- |
| Storage3      | Eth1/1           | Azure Local Node5 | Port1          | 25G   | SMB1 Cluster Storage |
| Storage3      | Eth1/2           | Azure Local Node6 | Port1          | 25G   | SMB1 Cluster Storage |
| Storage3      | Eth1/3           | Azure Local Node7 | Port1          | 25G   | SMB1 Cluster Storage |
| Storage3      | Eth1/4           | Azure Local Node8 | Port1          | 25G   | SMB1 Cluster Storage |
|               |                  |                   |                |       |                      |
| Storage4      | Eth1/1           | Azure Local Node5 | Port2          | 25G   | SMB2 Cluster Storage |
| Storage4      | Eth1/2           | Azure Local Node6 | Port2          | 25G   | SMB2 Cluster Storage |
| Storage4      | Eth1/3           | Azure Local Node7 | Port2          | 25G   | SMB2 Cluster Storage |
| Storage4      | Eth1/4           | Azure Local Node8 | Port2          | 25G   | SMB2 Cluster Storage |

#### Room2 Storage Room to Room

| Source Device | Source Interface | Dest Device | Dest Interface | Speed | Purpose              |
| ------------- | ---------------- | ----------- | -------------- | ----- | -------------------- |
| Storage3      | Eth1/49          | Storage1    | Eth1/49        | 100G  | SMB1 Cluster Storage |
| Storage3      | Eth1/50          | Storage1    | Eth1/50        | 100G  | SMB1 Cluster Storage |
|               |                  |             |                |       |                      |
| Storage4      | Eth1/49          | Storage2    | Eth1/49        | 100G  | SMB2 Cluster Storage |
| Storage4      | Eth1/50          | Storage2    | Eth1/50        | 100G  | SMB2 Cluster Storage |
