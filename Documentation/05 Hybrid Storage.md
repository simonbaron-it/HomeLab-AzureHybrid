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



