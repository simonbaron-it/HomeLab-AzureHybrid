# Azure Foundation
> This documentation establishes the Azure foundation for my hybrid infrastructure lab. It brings together Azure networking, a Windows Server virtual machine, secure administration, cost management and operational troubleshooting. The environment will support later phases of the project.

## Scenario and Objectives
### Business Scenario
A simulated organisation operates on-premise Active Directory, DNS, DHCP, file services and management tooling. It wants to introduce Azure while retaining its existing environment.

The objective is to establish a small, controlled Azure deployment that could later support hybrid identity, management, monitoring, storage and automation. Rather than moving production-style services into Azure immediately, the underlying infrastructure will be built and tested first.

### Objectives
- Prepare the Azure subscription, tenant, permissions and cost controls.
- Deploy a virtual network, workload subnet and network security group (NSG).
- Deploy and administer a Windows Server virtual machine without a permanent public IP.
- Test private administration, storage, VM lifecycle operations and troubleshooting.

## Scope
The Phase 1 build centres on an Azure resource group, a private Virtual Network and workload subnet, an associated NSG, and a Windows Server 2025 virtual machine. Administration is designed around private access rather than exposing RDP publicly.

> The existing on-premises environment remained separate throughout this phase. No site-to-site VPN, Azure-hosted domain controller, AD domain join, Microsoft Entra Cloud Sync, Azure Arc or centralised monitoring integration was implemented as part of the initial implementation.

## Existing On-Premise Environment

|System|Existing Function|Address/Network|
|---|---|---|
|`RTR01`|RRAS routing, NAT and DHCP relay|`10.10.10.1`/`10.10.20.1`/`172.16.0.2`|
|`DC01`|Active Directory, DNS and DHCP|`10.10.10.10`|
|`DC02`|Secondary domain controller, DNS and DHCP|`10.10.10.11`|
|`FS01`|Windows file services|`10.10.10.20`|
|`MGMT01`|Management and monitoring|`10.10.10.30`|
|`CLIENT01`|Domain-joined Windows 11 client|DHCP|
|AD domain|Existing on-premises domain|`baron.example.com`|
|Server subnet|On-premises infrastructure|`10.10.10.0/24`|
|Client subnet|On-premises clients|`10.10.20.0/24`|
|Lab WAN network|Hyper-V NAT-side network|`172.16.0.0/24`|

## Azure Architecture

> The design places an Azure VM in a dedicated Azure workload subnet protected by a subnet-level NSG. Azure Bastion Developer provides browser-based administration. At this stage, there is no private connection to the on-premises lab, and Azure-provided DNS is used.

<i>Insert architecture diagram</i>

## Azure Environment and Configuration

## Implementation

## Validation

## Troubleshooting?
