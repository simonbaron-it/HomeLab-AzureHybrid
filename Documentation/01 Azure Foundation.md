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

## Architecture

> The design places an Azure VM in a dedicated Azure workload subnet protected by a subnet-level NSG. Azure Bastion Developer provides browser-based administration. At this stage, there is no private connection to the on-premises lab, and Azure-provided DNS is used.

<i>Insert architecture diagram</i>

## Environment and Configuration

## Implementation

## Validation

## Troubleshooting?
