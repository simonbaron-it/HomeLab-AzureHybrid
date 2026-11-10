# Hybrid Management
> This documentation covers the integration of the existing on-premises Windows Server infrastructure with Microsoft Azure using Azure Arc-enabled Servers.
>
> The implementation focuses on hybrid server onboarding, centralised inventory, resource governance, agent management, connectivity validation and troubleshooting.

## Scenario and Objectives
### Business Scenario
A simulated organisation operates an existing Windows Server 2025 infrastructure with Active Directory, file services and management tooling. Phase 2 established hybrid identity synchronisation between on-premises Active Directory and Microsoft Entra ID.

As the organisation adopts Azure, it requires centralised visibility of infrastructure that remains on-premises. The objective is to register selected Windows servers with Azure Arc so that they can be represented and managed as Azure resources without migrating their workloads or changing the existing network environment.

### Objectives
- Prepare the Azure subscription, resource providers and permissions for Azure Arc-enabled Servers.
- Onboard `MGMT01` and `FS01` using the Azure Connected Machine agent.
- Validate Azure Arc connectivity, server inventory and agent health.
- Apply resource tags and review role-based access control (RBAC).
- Explore Azure Arc extensions, agent configuration and Resource Graph queries.
- Test agent connectivity failure, troubleshooting and recovery.
- Verify that existing management, identity synchronisation and file services remain operational.

## Scope
Phase 3 introduces Azure Arc-enabled Servers for two existing on-premises Windows Server 2025 virtual machines: `MGMT01` and `FS01`.

Each server will communicate with Azure Arc over outbound HTTPS through the existing lab routing and Hyper-V NAT configuration. The servers retain their existing Active Directory membership, IP addresses and roles.
> No site-to-site VPN, Azure VM domain join or migration of on-premises workloads is required. Domain controllers `DC01` and `DC02` will not be onboarded during this phase.
>
> Azure Monitor Agent, Log Analytics, Data Collection Rules and alerting will be introduced in Phase 4: Monitoring, rather than deployed as part of this implementation.

## Phase 3 Architecture

|Component|Configuration|
|---|---|
|AD Domain|`baron.example.com`|
|Management Server|`MGMT01` - `10.10.10.30`|
|File Server|`FS01` - `10.10.10.20`|
|Server Network|`10.10.10.0/24`|
|Network Routing|`RTR01`, `Hyper-V NAT`|
|Microsoft Entra Tenant|`Baron Hybrid Home Lab`|
|Azure Region|`Sweden Central`|
|Resource Group|`rg-hybridlab-swc-01`|
|Existing Azure VM|`AZ-LAB01` — `Not an Arc onboarding target`|
|Hybrid Management Service|`Azure Arc-enabled Servers`|
|Onboarding Targets|`MGMT01`, `FS01`|
|Connectivity|`Outbound HTTPS` - `TCP 443`|

### Hybrid Management Architecture
The hybrid management design uses the Azure Connected Machine agent to provide a management-plane connection between on-premises servers and Azure Arc. Each onboarded machine is represented as a separate Azure resource, allowing its properties, connection status, tags, access permissions and supported extensions to be managed through Azure.

<i>Insert architecture diagram</i>

> **Deployment Decision:** The existing `MGMT01` server hosts Windows Admin Center, Windows Event Forwarding and the Microsoft Entra Cloud Sync provisioning agent deployed during Phase 2.
>
> Azure Arc will also be installed on `MGMT01` to reuse the existing Hyper-V resources, with server health and existing services checked before and after onboarding. `FS01` will be onboarded separately to demonstrate management of an operational file server.

## Azure Arc Preparation
### Resource Providers and Permissions
The Azure subscription and resource group established during Phase 1 will be reused. The required resource providers and permissions will be reviewed before generating the onboarding script.

|Requirement|Configuration|
|---|---|
|Subscription|Existing `Azure Plan` subscription|
|Resource Group|`rg-hybridlab-swc-01`|
|Region|`Sweden Central`|
|Microsoft.HybridCompute|<i>Record registration status</i>|
|Microsoft.GuestConfiguration|<i>Record registration status</i>|
|Microsoft.HybridConnectivity|<i>Record registration status</i>|
|Onboarding Permissions|Azure Connected Machine Onboarding|

> The **Azure Connected Machine Onboarding** role provides permissions to register servers with Azure Arc without granting broader extension management access. Local administrator permissions are also required to install the agent on each Windows Server.

<i>Resource Provider Registration screenshot</i>

### Server Health and Connectivity
Before installing the Connected Machine agent, both servers will be checked for network connectivity, external DNS resolution and normal operation of existing Windows services.

```powershell
Get-NetIPConfiguration

Get-NetRoute -DestinationPrefix '0.0.0.0/0'

Resolve-DnsName login.microsoftonline.com

Test-NetConnection management.azure.com -Port 443
```

|Validation|MGMT01|FS01|
|---|---|---|
|Domain and network configuration|||
|External DNS resolution|||
|Outbound HTTPS connectivity|||
|Existing services functioning|||

> These checks provide an initial connectivity baseline. Following agent installation, azcmagent check will validate the Azure Arc-specific service endpoints. No inbound port forwarding is required for standard Arc onboarding.

## Azure Arc Agent Deployment

### Onboarding Configuration
Azure Arc onboarding will use the Azure portal's Onboard existing machines workflow. A Windows PowerShell installation script will be generated using the following configuration:

|Setting|Configuration|
|---|---|
|Azure Service|`Azure Arc > Machines`|
|Onboarding Method|Portal-generated PowerShell script|
|Resource Group|`rg-hybridlab-swc-01`|
|Region|`Sweden Central`|
|Operating System|`Windows`|
|Connectivity Method|Public endpoint — outbound HTTPS|
|Onboarding Targets|`MGMT01`, `FS01`|

<i>Azure Arc onboarding wizard screenshot</i>

### MGMT01 Onboarding
`MGMT01` was onboarded first to validate the agent and its connectivity without changing the file server.

Following execution of the portal-generated script in an elevated Windows PowerShell session, the agent and its services were inspected using:

```powershell
$ArcAgent = "$env:ProgramFiles\AzureConnectedMachineAgent\azcmagent.exe"

Get-Service himds, GCArcService, ExtensionService
& $ArcAgent show
& $ArcAgent version
& $ArcAgent check
```

|Component|Configuration|
|---|---|
|Server|`MGMT01`|
|Operating System|`Windows Server 2025`|
|Existing Roles|`Windows Admin Center`, `Windows Event Forwarding`, `Entra Cloud Sync`|
|Connected Machine Agent Version||
|Azure Arc Connection Status||
|Endpoint Connectivity Check||
|Existing Service Health||

<i>Screenshot - MGMT01 registered and connected in Azure Arc</i>

<i>Screenshot - MGMT01 agent and service health verification</i>

### FS01 Onboarding
`FS01` was then onboarded using the same subscription, resource group and region. The original SMB shares and server services were checked before and after installation.

```powershell
# Pre- and post-installation file-service checks
Get-Service LanmanServer
Get-SmbShare | Select-Object Name, Path, Description

# Azure Arc agent checks after installation
$ArcAgent = "$env:ProgramFiles\AzureConnectedMachineAgent\azcmagent.exe"
& $ArcAgent show
& $ArcAgent check
```

|Component|Configuration|
|---|---|
|Server|`FS01`|
|Operating System|`Windows Server 2025`|
|Existing Role|`Windows File Services`|
|Connected Machine Agent Version||
|Azure Arc Connection Status||
|Endpoint Connectivity Check||
|SMB Share Access||

<i>Screenshot - FS01 registered and connected in Azure Arc</i>

## Hybrid Inventory and Governance

### Arc Machine Inventory
The Azure Arc Machines page was used to confirm that both servers appear as independent Azure resources with the expected properties and connection status.

|Machine|Role|Arc Status|Agent Version|
|---|---|---|---|
|`MGMT01`|Management/Cloud Sync|||
|`FS01`|File Services|||

<i>Screenshot — Azure Arc machine inventory showing both servers</i>

### Resource Tagging
The Phase 1 tagging convention will be retained, with additional tags to identify on-premises hosting and server roles.

|Tag|MGMT01|FS01|
|---|---|---|
|`Project`|`HybridInfrastructureLab`|`HybridInfrastructureLab`|
|`Environment`|`Lab`|`Lab`|
|`Owner`|`PersonalLab`|`PersonalLab`|
|`ManagedBy`|`Manual`|`Manual`|
|`CostCentre`|`Learning`|`Learning`|
|`Hosting`|`OnPremises`|`OnPremises`|
|`ServerRole`|`Management`|`FileServices`|

<i>Screenshot — Azure Arc resource tags?</i>

### Role-Based Access Control
Azure RBAC assignments were reviewed to demonstrate how management access to Arc-enabled resources can be restricted. A non-privileged pilot identity from Phase 2 was granted the Reader role at the appropriate scope for a controlled test.

|Validation|Result|
|---|---|
|View Arc machine resources|Permitted for Reader|
|Modify Arc machine resources|Denied for Reader|
|Access local Windows or SMB services|Not granted through Azure Reader|

> Azure RBAC on an Arc resource does not provide local Windows administrator rights, interactive logon rights or access to `FS01` shares.
>
> <i>Screenshot — RBAC assignment or access test?</i>

### Extensions and Agent Management
The Azure Arc Extensions page was reviewed to identify supported management capabilities. Azure Monitor Agent deployment will be deferred until Phase 4 so that its Log Analytics and Data Collection Rule configuration can be documented together.

The local Connected Machine agent can be inspected using:
```powershell
& $ArcAgent show
& $ArcAgent version
& $ArcAgent config list
```

<i>Arc machine extensions and agent management options screenshot</i>

### Azure Resource Graph Inventory
Azure Resource Graph will be used to query Arc machine metadata and confirm that the registered servers can be identified using Kusto Query Language (KQL).

```kusto
Resources
| where type =~ 'microsoft.hybridcompute/machines'
| project
    name,
    resourceGroup,
    location,
    Status = tostring(properties.status),
    OperatingSystem = tostring(properties.osName),
    ServerRole = tostring(tags.ServerRole)
| order by name asc
```

<i>Resource Graph query and results.</i>

## Agent Connectivity Validation and Testing

|Test|Action|Result|
|---|---|---|
|MGMT01 Onboarding|Register with Azure Arc|Connected resource visible|
|FS01 Onboarding|Register with Azure Arc|Connected resource visible|
|Agent Health|Run `azcmagent show` and `azcmagent check`|Healthy status and required endpoint access|
|MGMT01 Services|Check WAC, WEF and Cloud Sync|Existing functions retained|
|FS01 Services|Check SMB shares and client access|Existing functions retained|
|Resource Governance|Review tags and RBAC|Correct metadata and permissions|
|Resource Graph|Query Arc-enabled machines|Both machines listed|
|Agent Failure|Stop `himds` on MGMT01|Azure eventually reports disconnection|
|Agent Recovery|Start `himds` on MGMT01|Azure returns to Connected|

### Agent Connectivity Failure and Recovery??

## Security and Operational Considerations

### Hybrid Management Security
The Azure Arc implementation was designed to provide cloud management visibility while maintaining the existing on-premises security model.

Key considerations include:
- Maintaining outbound-only Azure Arc connectivity without exposing inbound management ports.
- Applying least-privilege Azure RBAC permissions.
- Preserving existing Windows, Active Directory and SMB access controls.
- Reviewing the additional agent workload on `MGMT01`, which also hosts the privileged Cloud Sync provisioning agent.
- Keeping onboarding scripts, credentials and diagnostic archives out of the public repository.
- Reviewing agent health and software maintenance requirements.
- Deferring optional extensions and paid management features until their costs and permissions have been evaluated.

> Although Azure RBAC does not automatically grant local Windows access, roles that permit extension deployment can enable privileged operations on Arc-enabled servers. Management permissions should therefore be restricted to appropriately authorised administrators.

### Cost Management
The core Azure Arc-enabled Servers control plane does not introduce an additional per-server onboarding charge. Optional services such as Azure Monitor, security features and some patch-management capabilities can incur charges.

The £20 monthly budget established during Phase 1, with email notifications at 50%, 80% and 100%, will remain in place. The phase will reuse existing on-premises servers and should not require an additional Azure VM or VPN Gateway.

## Phase 3 Summary
- **Hybrid Server Management:** Summarise onboarding of MGMT01 and FS01 and their final connection statuses.   
- **Azure Arc Deployment:** Record agent versions, installation results and configuration decisions.   
- **Resource Governance:** Summarise machine inventory, resource tagging and RBAC validation.   
- **Operational Validation:** Record verification of existing MGMT01 and FS01 services.   
- **Troubleshooting:** Summarise the agent outage simulation, investigation and recovery.   
- **Security and Cost:** Document the final controls, observed charges and deferred features.   

## Next Steps
Phase 4: Monitoring will extend the hybrid management configuration by implementing Azure Monitor, a Log Analytics workspace, Azure Monitor Agent, Data Collection Rules, Kusto Query Language (KQL) queries and operational alerts.

The Azure Arc-enabled `MGMT01` and `FS01` servers will provide the on-premises endpoints for monitoring, alongside the existing Azure virtual machine `AZ-LAB01`.
