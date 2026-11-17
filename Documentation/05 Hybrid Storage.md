# Hybrid Storage
> This documentation covers the integration of the existing on-premises Windows Server file services with Microsoft Azure using Azure Files and Azure File Sync.
>
> The implementation focuses on Azure file share deployment, File Sync agent configuration, bidirectional synchronisation, operational validation, troubleshooting and cost management.

## Scenario and Objectives

### Business Scenario
A simulated organisation operates an existing Windows Server 2025 infrastructure with Active Directory and centralised file services hosted on `FS01`. Phases 3 and 4 introduced Azure Arc-enabled Servers and Azure Monitor, providing hybrid management and monitoring of the existing file server.

As the organisation expands its use of Microsoft Azure, the infrastructure team requires a method of synchronising selected on-premises files with cloud storage while retaining the existing Windows Server file services and access model. The objective is to implement and validate Azure File Sync using a dedicated test folder on `FS01`, without migrating or changing the established departmental shares.

### Objectives
- Deploy an Azure Storage Account and an Azure Files SMB share.
- Deploy a Storage Sync Service, sync group and cloud endpoint.
- Install the Azure File Sync agent on the existing `FS01` server and register it with Azure.
- Configure a server endpoint for a dedicated NTFS test folder, with cloud tiering disabled.
- Validate file creation, modification, folder structure and deletion synchronisation from on-premises to Azure.
- Validate cloud-to-server synchronisation, including manual cloud change detection where required.
- Review synchronisation health, Windows event logs and service connectivity.
- Perform a controlled File Sync failure-and-recovery test.
- Verify that existing SMB shares, NTFS permissions and Azure monitoring remain operational.
- Review storage security, data protection limitations and Azure costs.

## Scope
Phase 5 extends selected file services from the existing on-premises `FS01` server to Azure using Azure Files and Azure File Sync. A dedicated test directory `E:\AzureSync-Test` is used as the server endpoint, leaving the established `E:\CompanyData` directory and departmental SMB shares outside the synchronisation scope.

The implementation reuses the existing Azure subscription, resource group, resource tagging convention, and network connectivity. `FS01` is already connected to Azure Arc and monitored through Azure Monitor Agent from the previous phases.
> Azure File Sync uses outbound connections to Azure service endpoints; this phase does not require a site-to-site VPN or inbound SMB port forwarding. Direct cloud SMB access, Azure Files identity-based authentication and cloud tiering are outside the initial scope.
> 
> File synchronisation is not a replacement for backup. Deletions and changes can propagate between endpoints; protection and recovery will be assessed separately.

## Phase 5 Infrastructure

|Component|Configuration|
|---|---|
|AD Domain|`baron.example.com`|
|File Server|`FS01` — `10.10.10.20`|
|Dedicated Test Directory|`E:\AzureSync-Test`|
|Server Network|`10.10.10.0/24`|
|Azure Region|`Sweden Central`|
|Resource Group|`rg-hybridlab-swc-01`|
|Storage Account||
|Azure Files Share||
|Storage Sync Service||
|Sync Group||
|Cloud Endpoint||
|Server Endpoint|`FS01` — `E:\AzureSync-Test`|
|Cloud Tiering|`Disabled`|
|Existing Hybrid Management|Azure Arc-enabled Servers|
|Existing Monitoring Workspace|`law-hybridlab-swc-01`|

### Hybrid Storage Architecture
The design uses the Azure File Sync agent on `FS01` to synchronise a dedicated NTFS test folder with an Azure Files SMB share. The Storage Sync Service manages the synchronisation relationship through a sync group containing the cloud endpoint and the registered `FS01` server endpoint.

The existing Windows Server remains responsible for local file access and permissions. Azure Files provides the cloud-side copy of the synchronised test data. The File Sync agent communicates with Azure over outbound HTTPS through the existing on-premises routing and Hyper-V NAT configuration.

<i>Insert Architecture Diagram</i>

> **Deployment Decision:** A dedicated folder on `FS01` was selected for the File Sync pilot rather than the existing `CompanyData` folder. This isolates synchronisation and troubleshooting tests from departmental file shares and their AGDLP-based NTFS permissions.

## Azure Storage Configuration

### Storage Account Deployment
An Azure Storage Account is used to host the Azure Files SMB share for the synchronisation pilot. The deployment uses the existing Azure resource group and Sweden Central region.

|Setting|Configuration|
|---|---|
|Storage Account||
|Resource Group|`rg-hybridlab-swc-01`|
|Region|`Sweden Central`|
|Primary Service|`Azure Files`|
|Performance|`Standard/HDD`|
|Billing Model|`Pay-as-you-go`|
|Redundancy||
|Secure Transfer||
|Minimum TLS Version||
|Storage Account Key Access||
|Public Network Access||
|Resource Tags|Existing Phase 1 tagging convention|

<i>Storage Account Screenshot</i>

### Azure Files Share
An Azure Files SMB share is created within the storage account as the cloud endpoint for File Sync.

|Setting|Configuration|
|---|---|
|File Share||
|Protocol|`SMB`|
|Storage Account||
|Access Tier||
|Quota/Provisioned Capacity||
|File Share Soft Delete||
|Initial Share Contents|Empty, before File Sync configuration|

<i>Azure File Share Screenshot</i>

## Azure File Sync Configuration

### Storage Sync Service Deployment
A Storage Sync Service is deployed to manage registered Windows servers and the synchronisation relationships between local server endpoints and Azure file shares.

|Setting|Configuration|
|---|---|
|Storage Sync Service||
|Resource Group|`rg-hybridlab-swc-01`|
|Region|`Sweden Central`|
|Resource Tags|Existing Phase 1 tagging convention|
|Registered Servers|`FS01`|

<i>Storage Sync Service Screenshot</i>

### FS01 Preparation
Before installing the File Sync agent, the existing file server configuration and available storage are checked. A dedicated NTFS folder is prepared with synthetic test data, separate from `E:\CompanyData`.

```powershell
# Verify file server services and shares.
Get-Service LanmanServer
Get-SmbShare | Select-Object Name, Path, Description

# Verify the E: data volume.
Get-Volume -DriveLetter E |
    Select-Object DriveLetter, FileSystem, Size, SizeRemaining

# Verify outbound HTTPS connectivity.
Resolve-DnsName management.azure.com
Test-NetConnection management.azure.com -Port 443

# Create an isolated test dataset, if it does not exist.
New-Item -Path 'E:\AzureSync-Test\IT' -ItemType Directory -Force
New-Item -Path 'E:\AzureSync-Test\Documents' -ItemType Directory -Force

'Initial file created on FS01' |
    Set-Content 'E:\AzureSync-Test\FS01-Origin.txt'
'IT department test data' |
    Set-Content 'E:\AzureSync-Test\IT\IT-Test.txt'
'Document synchronisation test' |
    Set-Content 'E:\AzureSync-Test\Documents\Document-Test.txt'

# Record the test directory and NTFS permissions.
Get-ChildItem 'E:\AzureSync-Test' -Recurse
icacls 'E:\AzureSync-Test'
```

|Validation|Result|
|---|---|
|`LanmanServer` Status||
|Existing SMB Shares||
|E: NTFS Volume and Free Space||
|External DNS / HTTPS Connectivity||
|Test Directory and Files||
|NTFS Permissions Baseline||
|Backup / Recovery Point Available||

<i>File Sync Test Folder Screenshot</i>

### Agent Installation and Server Registration
The Azure File Sync agent is installed on the existing `FS01` Windows Server 2025 VM.

|Component|Configuration|
|---|---|
|Server|`FS01`|
|Operating System|Windows Server 2025|
|Installation Method||
|File Sync Agent Version||
|Agent Service||
|Storage Sync Service||
|Registration Status||

> The File Sync server registration process associates FS01 with the existing Azure subscription, resource group and Storage Sync Service. The Azure portal's Registered servers view is used to verify the connection status and agent version.

<i>Sync Agent/Registered Servers Screenshots</i>

### Sync Group and Cloud Endpoint
A sync group is created to link the Azure Files share to the registered on-premises server.

|Setting|Configuration|
|---|---|
|Sync Group||
|Storage Sync Service||
|Storage Account||
|Cloud Endpoint||
|Provisioning Status||

> The cloud endpoint identifies the Azure file share. Only the intended test share is associated with this sync group.

<i>Screenshots?</i>

### FS01 Server Endpoint
A server endpoint is added to the sync group, targeting the dedicated on-premises test folder.

|Setting|Configuration|
|---|---|
|Registered Server|`FS01`|
|Local Path|`E:\AzureSync-Test`|
|Cloud Tiering|`Disabled`|
|Initial Synchronisation Mode||
|Endpoint Health||
|Persistent Sync Errors||

> The server endpoint does not target `E:\CompanyData`. With cloud tiering disabled, synchronised test files remain available locally on FS01 rather than being intentionally tiered into cloud-backed placeholders.

<i>Screenshots?</i>

## Validation and Testing

### File Synchronisation Validation
The following tests are designed to verify that changes made on FS01 are reflected in Azure Files and that changes made directly in the Azure file share are eventually downloaded to the server.

|Test|Action|Result|
|---|---|---|
|Initial Upload|Synchronise the prepared `FS01-Origin.txt`, `IT` and `Documents` data||
|Folder/File Creation|Create a new folder and file on FS01||
|File Modification|Append text to `New-File.txt` and inspect the cloud copy||
|Cloud-to-Server|Upload `Azure-Origin.txt` to Azure Files and detect/download the change||
|Deletion|Create, sync and delete `Delete-Me.txt`; verify propagation||
|NTFS Permissions|Compare relevant test-folder ACLs before and after synchronisation||
|Existing File Services|Verify `LanmanServer`, original SMB shares and `E:\CompanyData`||

<i>Validation Screenshots</i>

## Security and Operational Considerations

### Hybrid Storage Security
The File Sync implementation is intended to extend selected file storage into Azure without replacing the existing on-premises file access model.

Key considerations:
- Keep the synchronisation scope limited to the dedicated `E:\AzureSync-Test` directory.
- Retain the existing AGDLP-based NTFS and SMB access controls on `E:\CompanyData`.
- Avoid exposing inbound SMB or RDP ports from the home network to the internet.
- Use secure outbound HTTPS connections for File Sync service communication.
- Review storage account networking, secure transfer, encryption and management permissions.
- Keep cloud tiering disabled during the initial lab and verify local file availability.
- Treat file synchronisation as separate from versioned backup and recovery.

### Cost Management
The Phase 1 £20 monthly budget and alert thresholds at £10, £16 and £20 remain in place. Azure Files and associated services can generate charges even while the Azure VM `AZ-LAB01` is deallocated.

|Cost Area|Decision/Observation|
|---|---|
|Storage Billing Model||
|Redundancy and Capacity||
|File Transactions/Data Transfer||
|Storage Sync Service/Registered Servers||
|Snapshots/Retention||
|Actual Phase 5 Cost||
|Budget Alerts|Existing £20 monthly budget|

## Phase 5 Summary
- Azure Storage:
- Azure File Sync:
- Synchronisation Validation:
- Monitoring and Troubleshooting:
- Existing Services:
- Security and Cost:

## Next Steps
**Phase 6: Patching, Backup and Recovery** will focus on operational maintenance of the Azure Windows Server environment and on validating backup and recovery procedures.

This will include Azure Update Manager, Azure VM Backup and controlled recovery of test data. The distinction between Azure File Sync and backup will inform the recovery design and documentation.
