# Azure Foundation
> This documentation establishes the Azure foundation for a hybrid infrastructure lab, covering virtual networking, Windows Server deployment, secure administration and cost management. The environment is designed to support future integration with existing on-premises infrastructure.

## Scenario and Objectives
### Business Scenario
A simulated organisation operates on-premises Active Directory, DNS, DHCP, file services and management tooling. It wants to introduce Azure while retaining its existing environment.

The objective is to establish a small, controlled Azure deployment that could later support hybrid identity, management, monitoring, storage and automation. Rather than moving production-style services into Azure immediately, the underlying infrastructure will be built and tested first.

### Objectives
- Prepare the Azure subscription, tenant, permissions and cost controls.
- Deploy a virtual network, workload subnet and network security group (NSG).
- Deploy and administer a Windows Server virtual machine without a permanent public IP.
- Test private administration, storage, VM lifecycle operations and troubleshooting.

## Scope
The Phase 1 build centres on an Azure resource group, a private Virtual Network and workload subnet, an associated NSG, and a Windows Server 2025 virtual machine. Administration is designed around private access rather than exposing RDP publicly.

> The existing on-premises environment remained separate throughout this phase. No site-to-site VPN, Azure-hosted domain controller, AD domain join, Microsoft Entra Cloud Sync, Azure Arc or centralised monitoring integration was implemented as part of the initial implementation.

## Existing On-Premises Environment

|System|Existing Function|Address/Network|
|---|---|---|
|`RTR01`|RRAS routing, NAT and DHCP relay|`10.10.10.1`, `10.10.20.1`, `172.16.0.2`|
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

<img src="../Images/Azure%20Architecture%20Diagram.png" width="700"/>

## Azure Environment and Configuration

### Tenant, Subscription and Cost Management

|Component|Configuration|
|---|---|
|Azure Region|`UK South`|
|Tenant Display Name|`Baron Hybrid Home Lab`|
|Subscription|`Azure Plan` (30-day free account trial)|
|Administrative Access|Microsoft Entra account secured with MFA|
|Budget Alerts|`50%: £10`, `80%: £16`, `100%: £20`|

#### Budget Alerts
<i>Explain budget alert setup</i>

<img src="../Images/Azure%20Subscription.png" width="1000"/>

### Resource Group
The resource group `rg-hybridlab-uks-01` was created in the `UK South` region to contain the Phase 1 infrastructure.

To organise the deployment, I adopted a consistent resource naming convention and defined the following tags for application across Azure resources:    
`Project:HybridInfrastructureLab`, `Environment:Lab`, `Owner:PersonalLab`, `ManagedBy:Manual`, `CostCentre:Learning`

<img src="../Images/Resource%20Group.png" width="1000"/>

### Virtual Network and Subnet
A dedicated VNet was created for the Azure-hosted workload. The address space was kept separate from the on-premises home lab to support future private connectivity without needing to readdress the existing servers.

|Component|Configuration|
|---|---|
|Resource Group|`rg-hybridlab-uks-01`|
|Virtual Network|`vnet-hybridlab-uks-01`|
|Region|`UK South`|
|Address Space|`10.30.0.0/16`|
|Workload Subnet|`snet-workloads-01`|
|Subnet Prefix|`10.30.1.0/24`|
|DNS|Azure-provided DNS during Phase 1|

<img src="../Images/Virtual%20Network.png" width="1000"/>

<img src="../Images/VNet%20Subnet.png" width="800"/>

> Azure-provided DNS was retained because the lab did not yet have a route to the on-premises domain controllers.
>
> I did not introduce a VPN Gateway, Azure Firewall or complex hub-and-spoke design. Those were unnecessary for the Phase 1 objectives and would have increased cost and scope.

### Network Security Group Configuration
A Network Security Group was deployed and associated with the `snet-workloads-01` subnet. It retained Azure's default security rules, permitting traffic within the Virtual Network while denying unmatched inbound traffic from the internet. No custom rule was introduced to expose RDP publicly.

|Control|Approach|
|---|---|
|NSG|`nsg-workloads-uks-01`|
|Scope|`snet-workloads-01`|
|Public Inbound RDP|Not allowed|
|Intra-VNet Traffic|Governed by applicable NSG rules|

<img src="../Images/Network%20Security%20Group.png" width="1000"/>

### Windows Server 2025 VM Deployment

### Azure Bastion Developer

### VM Storage and Lifecycle Management

## Validation and Testing

## Troubleshooting

## Lessons and Next Steps
