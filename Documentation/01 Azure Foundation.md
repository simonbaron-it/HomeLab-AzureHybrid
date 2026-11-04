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

### Existing On-Premises Environment

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
The design places an Azure VM in a dedicated Azure workload subnet protected by a subnet-level NSG. Azure Bastion Developer provides browser-based administration. At this stage, there is no private connection to the on-premises lab, and Azure-provided DNS is used.

<img src="../Images/Azure%20Architecture%20Diagram.png" width="700"/>

> **Azure Region Decision:** The initial deployment plan targeted the `UK South` region, however the required VM size was unavailable to my Azure subscription in both `UK South` and `UK West`.
>
> Although suitable VM sizes were listed as available in `West Europe` and `North Europe`, Azure subscription restrictions prevented deployment in those regions.
>
> The `Sweden Central` region supported the required VM configuration and permitted deployment under the existing Azure subscription, therefore it was selected for the hybrid lab environment.

## Azure Environment and Configuration

### Tenant, Subscription and Cost Management

|Component|Configuration|
|---|---|
|Azure Region|`Sweden Central`|
|Tenant Display Name|`Baron Hybrid Home Lab`|
|Subscription|`Azure Plan` (30-day free account trial)|
|Administrative Access|Microsoft Entra account secured with MFA|

<img src="../Images/Azure%20Subscription.png" width="1000"/>

#### Budget Alerts
A £20 monthly budget was configured in Azure Cost Management, with email notification thresholds set at `50%: £10`, `80%: £16` and `100%: £20`. These alerts provide early warning of increasing Azure costs during deployment and testing, helping to identify unexpected expenditure.

> Budget alerts do not automatically stop resources or prevent further charges, so resource usage and associated costs must still be monitored.

<img src="../Images/Budget%20Alerts.png" width="1000"/>

### Resource Group
The resource group `rg-hybridlab-swc-01` was created in the `Sweden Central` region to contain the hybrid lab infrastructure.

To organise the deployment, I adopted a consistent resource naming convention and defined the following tags for application across Azure resources:    
`Project:HybridInfrastructureLab`, `Environment:Lab`, `Owner:PersonalLab`, `ManagedBy:Manual`, `CostCentre:Learning`

<img src="../Images/Resource%20Group.png" width="1000"/>

### Virtual Network and Subnet
A dedicated VNet was created for the Azure-hosted workload. The address space was kept separate from the on-premises home lab to support future private connectivity without needing to readdress the existing servers.

|Component|Configuration|
|---|---|
|Resource Group|`rg-hybridlab-swc-01`|
|Virtual Network|`vnet-hybridlab-swc-01`|
|Region|`Sweden Central`|
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
|NSG|`nsg-workloads-swc-01`|
|Scope|`snet-workloads-01`|
|Public Inbound RDP|Not allowed|
|Intra-VNet Traffic|Governed by applicable NSG rules|

<img src="../Images/Network%20Security%20Group.png" width="1000"/>

### Windows Server 2025 VM Deployment
The Virtual Machine `AZ-LAB01` was deployed using the `Windows Server 2025 Datacenter Gen 2` image and the `Standard_B2s_v2` VM size. I connected its network interface to the existing VNet and workload subnet, configured a managed OS disk, and reviewed the security and management settings during deployment.

|VM Component|Configuration|
|---|---|
|Name|`AZ-LAB01`|
|Region|`Sweden Central`|
|Operating System|`Windows Server 2025 Datacenter Gen 2`|
|VM Size|`Standard_B2s_v2` (2vCPUs, 8GB memory)|
|Private IP|`10.30.1.4`|
|Public IP|`None`|
|OS Disk|`Standard SSD LRS` (127GB)|
|Boot Security|`Trusted Launch`, `Secure Boot` & `vTPM` enabled|
|Administration|`Bastion Developer`|

<img src="../Images/Virtual%20Machine%202.png" width="1000"/>

### Azure Bastion Developer
Azure Bastion Developer was used to provide browser-based RDP administration of `AZ-LAB01` through the Azure portal, without requiring a public IP address on the virtual machine or exposing RDP directly to the internet.

The Developer tier was selected because it provides free connectivity for development and testing environments using shared infrastructure, without requiring a dedicated `AzureBastionSubnet` or public IP resource.

An RDP session was established through the Azure portal using the VM's administrator credentials, confirming that the server could be accessed through Bastion using its private IP address (`10.30.1.4`).

### Managed Data Disk
An additional managed data disk was attached to `AZ-LAB01` to provide storage separate from the operating system disk and demonstrate Azure VM disk management.

|Component|Configuration|
|---|---|
|Disk Name|`disk-az-lab01-data-01`|
|Disk Type|`Standard SSD LRS`|
|Disk Size|`4GB`|
|Drive Letter|`(E:)`|
|File System|`NTFS`|

Using Windows Disk Management, the new disk was initialised using GPT, formatted and assigned drive letter `(E:)`.

<img src="../Images/VM%20Data%20Disk%202.png" width="700"/>

> A test file was created on the managed disk using PowerShell
> ```powershell
> New-Item -Path 'E:\LabData' -ItemType Directory
>
> 'Azure managed disk test' |
>    Out-File 'E:\LabData\Storage-Test.txt'
> ```

### Lifecycle Management
The Azure VM `AZ-LAB01` is deallocated at the end of every lab session in order to stop VM compute charges and control expenditure.
> Managed disks and certain other resources can continue to incur costs even when the VM is deallocated.

<i>Deallocated VM screenshot</i>

#### Temporary Public IP and Windows Updates
To perform Windows Server updates, `AZ-LAB01` required outbound connectivity to Microsoft's update services. As the VM was deployed without a public IP or a dedicated outbound connectivity solution, a temporary public IP was associated with its network interface to provide internet access.

<img src="../Images/VM%20Temp%20Public%20IP.png" width="700"/>

Windows Update was used to check for and install available operating system updates. No inbound RDP rule was introduced, and administration continued through Azure Bastion Developer.

> Once the updates were completed, the public IP was disassociated and the resource removed, returning the VM to its original private-only configuration.

## Phase 1 Summary
`Phase 1` provided practical experience with Azure infrastructure deployment, resource governance, virtual networking, Windows Server administration and cost management.

- **Azure Infrastructure Deployment:** Provisioned and configured resource groups, virtual networks, Windows Server virtual machines and managed disks.
- **Azure Networking:** Configured subnets, IP addressing, network security groups and network connectivity.
- **Windows Server Administration:** Deployed and administered Windows Server 2025, including system configuration, updates and storage management.
- **Cloud Security:** Configured network security controls and secure remote administration using Azure Bastion Developer.
- **Cost Management:** Implemented budgets, cost alerts and resource tagging to control expenditure.

## Next Steps
`Phase 1` establishes the Azure infrastructure required for the next stage of the hybrid lab.

`Phase 2: Hybrid Identity` will focus on integrating selected on-premises Active Directory identities with Microsoft Entra ID using Microsoft Entra Cloud Sync.

This will include reviewing identity synchronisation requirements, configuring the necessary components and validating synchronisation of selected users and groups.
