# Hybrid Storage
> This documentation covers the integration of the existing on-premises Windows Server file services with Microsoft Azure using Azure Files and Azure File Sync.
>
> The implementation focuses on Azure file share deployment, File Sync agent configuration, bidirectional synchronisation, operational validation and cost management.

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
|Storage Account|`sthybridfileswc01`|
|Azure Files Share|`fs01-labdata`|
|Storage Sync Service|`stsync-hybridlab-swc-01`|
|Sync Group|`sg-fs01-test-01`|
|Cloud Endpoint|`fs01-labdata`|
|Server Endpoint|`FS01` — `E:\AzureSync-Test`|
|Cloud Tiering|`Disabled`|
|Existing Hybrid Management|Azure Arc-enabled Servers|

### Hybrid Storage Architecture
The design uses the Azure File Sync agent on `FS01` to synchronise a dedicated NTFS test folder with an Azure Files SMB share. The Storage Sync Service manages the synchronisation relationship through a sync group containing the cloud endpoint and the registered `FS01` server endpoint.

The existing Windows Server remains responsible for local file access and permissions. Azure Files provides the cloud-side copy of the synchronised test data. The File Sync agent communicates with Azure over outbound HTTPS through the existing on-premises routing and Hyper-V NAT configuration.

<img src="../Images/Phase%205%20Hybrid%20Storage%20-%20Architecture%20Diagram.png" width="1000"/>

> **Deployment Decision:** A dedicated folder on `FS01` was selected for the File Sync pilot rather than the existing `CompanyData` folder. This isolates synchronisation from departmental file shares and their AGDLP-based NTFS permissions.

## Azure Storage Configuration

### Storage Account Deployment
An Azure Storage Account is used to host the Azure Files SMB share for the synchronisation pilot. The deployment uses the existing Azure resource group and Sweden Central region.

|Setting|Configuration|
|---|---|
|Storage Account|`sthybridfileswc01`|
|Account Type|`StorageV2`|
|Resource Group|`rg-hybridlab-swc-01`|
|Region|`Sweden Central`|
|Primary Service|`Azure Files`|
|Performance|`Standard/HDD`|
|Billing Model|`Pay-as-you-go`|
|Redundancy|`Locally Redundant Storage (LRS)`|
|Secure Transfer|`Enabled`|
|Minimum TLS Version|`Version 1.2`|
|Storage Account Key Access|`Enabled`|
|Public Network Access|`Enabled`|
|Resource Tags|Existing Phase 1 tagging convention|

<img src="../Images/Storage%20Account.png" width="1000"/>

### Azure Files Share
An Azure Files SMB share is created within the storage account as the cloud endpoint for File Sync.

|Setting|Configuration|
|---|---|
|File Share|`fs01-labdata`|
|Protocol|`SMB`|
|Storage Account|`sthybridfileswc01`|
|Access Tier|`Hot`|
|File Share Soft Delete|`Enabled`|
|Initial Share Contents|Empty, before File Sync configuration|

<img src="../Images/Azure%20File%20Share.png" width="1000"/>

## Azure File Sync Configuration

### Storage Sync Service Deployment
A Storage Sync Service is deployed to manage registered Windows servers and the synchronisation relationships between local server endpoints and Azure file shares.

|Setting|Configuration|
|---|---|
|Storage Sync Service|`stsync-hybridlab-swc-01`|
|Resource Group|`rg-hybridlab-swc-01`|
|Region|`Sweden Central`|
|Resource Tags|Existing Phase 1 tagging convention|
|Registered Servers|`FS01`|

<img src="../Images/Storage%20Sync%20Service.png" width="1000"/>

### FS01 Preparation
Before installing the File Sync agent, the existing file server configuration and available storage were checked.

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

# Verify Azure Arc connection
& 'C:\Program Files\AzureConnectedMachineAgent\azcmagent.exe' show
```

|Validation|Result|
|---|---|
|`LanmanServer` Status|`Running`|
|Existing SMB Shares|`Verified`|
|E: NTFS Volume Free Space|`20964868096 bytes`|
|External DNS / HTTPS Connectivity|`Successful`|
|Azure Arc Connection|`Connected`|

A dedicated NTFS folder is prepared on `FS01` with synthetic test data, separate from `E:\CompanyData`.

<img src="../Images/Azure%20Sync%20test%20Directory%20FS01.png" width="1000"/>

### Agent Installation and Server Registration
The Azure File Sync agent is installed on the existing `FS01` Windows Server 2025 VM.

|Component|Configuration|
|---|---|
|Server|`FS01`|
|Operating System|Windows Server 2025|
|Installation Method|`.msi`|
|File Sync Agent Version|`23.2`|
|Storage Sync Service|`stsync-hybridlab-swc-01`|

> The File Sync server registration process associates `FS01` with the existing Azure subscription, resource group and Storage Sync Service. The Azure portal's Registered servers view is used to verify the connection status and agent version.

<img src="../Images/Azure%20File%20Sync%20-%20Registered%20Servers.png" width="1000"/>

### Sync Group and Cloud Endpoint
A sync group is created to link the Azure Files share to the registered on-premises server.

|Setting|Configuration|
|---|---|
|Sync Group|`sg-fs01-test-01`|
|Storage Sync Service|`stsync-hybridlab-swc-01`|
|Storage Account|`sthybridfileswc01`|
|Cloud Endpoint|`fs01-labdata`|
|Provisioning Status|`Provisioned`|

> The cloud endpoint identifies the Azure file share. Only the intended test share is associated with this sync group.

<img src="../Images/Azure%20File%20Sync%20Endpoints.png" width="1000"/>

### FS01 Server Endpoint
A server endpoint is added to the sync group, targeting the dedicated on-premises test folder.

|Setting|Configuration|
|---|---|
|Registered Server|`FS01`|
|Local Path|`E:\AzureSync-Test`|
|Cloud Tiering|`Disabled`|
|Initial Synchronisation Mode|`Merge`|
|Endpoint Health|`Healthy`|
|Persistent Sync Errors|`0`|

> The server endpoint does not target `E:\CompanyData`. With cloud tiering disabled, synchronised test files remain available locally on `FS01` rather than being intentionally tiered into cloud-backed placeholders.

<img src="../Images/Azure%20File%20Sync%20-%20Server%20Endpoint.png" width="1000"/>

## Validation and Testing

### File Synchronisation Validation
The following tests verify that changes made on FS01 are reflected in Azure Files and that changes made directly in the Azure file share are eventually downloaded to the server.

#### Initial Upload
The `FS01-Origin.txt`, `IT` and `Documents` data prepared on `FS01` successfully replicated to the Azure File Share.

<img src="../Images/File%20Sync%20Validation.png" width="1000"/>

#### Folder/File Creation
A `New-File.txt` file and `New-Department` folder created on `FS01` successfully replicated to the Azure File Share.

<img src="../Images/File%20Sync%20Validation%20-%20New%20Folder.png" width="1000"/>

#### File Modification
`New-File.txt` was renamed to `Renamed-File.txt` on `FS01` and the change was verified in the Azure File Share.

<img src="../Images/File%20Sync%20Validation%20-%20Renamed%20File.png" width="1000"/>

#### Cloud-to-Server Replication
An `Azure-Origin.txt` file was uploaded directly to the Azure File Share and successfully replicated to `FS01` by manually triggering change detection.

<img src="../Images/File%20Sync%20Validation%20-%20New%20Azure%20File.png" width="1000"/>

#### File Deletion
A `Delete-Me.txt` file was deleted from `FS01` and the deletion was successfully replicated in the Azure File Share.

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
The Phase 1 £20 monthly budget and alert thresholds at £10, £16 and £20 remained in place.

|Cost Area|Decision/Observation|
|---|---|
|Storage Billing Model|`Pay-As-You-Go`|
|Redundancy|`Locally Redundant Storage (LRS)`|
|Storage Sync Service/Registered Servers|`FS01`|
|Budget Alerts|Existing £20 monthly budget|

## Phase 5 Summary
Phase 5 extended the existing on-premises file server infrastructure into Microsoft Azure using Azure Files and Azure File Sync.

- Azure Storage: Deployed an Azure Storage Account and Azure Files share in Sweden Central to provide cloud-based storage for synchronised test data.
- Azure File Sync: Installed and registered the Azure File Sync agent on `FS01`, configuring a Storage Sync Service, sync group and server endpoint targeting `E:\AzureSync-Test`, with cloud tiering disabled.
- Synchronisation Validation: Validated bidirectional file synchronisation, including initial uploads, file creation, modification, deletion propagation and cloud-to-server change detection.
- Existing Services: Preserved the existing departmental SMB shares, NTFS permissions and file services by restricting synchronisation to a dedicated test directory.
- Security and Cost: Maintained outbound HTTPS connectivity without exposing inbound SMB ports, reviewed Azure storage and synchronisation costs, and retained the existing £20 monthly budget and cost alerts.

## Next Steps
**Phase 6: Patching, Backup and Recovery** will focus on operational maintenance of the Azure Windows Server environment and on validating backup and recovery procedures.

This will include Azure Update Manager, Azure VM Backup and controlled recovery of test data. The distinction between Azure File Sync and backup will inform the recovery design and documentation.
