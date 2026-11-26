# Patching and Recovery
> This documentation covers the implementation and validation of Windows Server patch management, Azure VM backup and file-level recovery using Azure Update Manager and Azure Backup.
>
> The implementation focuses on update assessment and deployment, backup policy configuration, recovery-point verification, restoration of test data, operational validation and cost management.

## Scenario and Objectives

### Business Scenario
A simulated organisation operates an existing Windows Server 2025 infrastructure across an on-premises Hyper-V lab and Microsoft Azure. Previous phases introduced hybrid identity synchronisation, Azure Arc management, centralised monitoring and Azure File Sync.

As the organisation adopts Azure-hosted workloads, the infrastructure team must be able to assess and deploy operating system updates, protect virtual machines against data loss, and recover files following accidental deletion or modification. The objective is to establish and validate repeatable maintenance and recovery procedures for the Azure-hosted Windows Server without disrupting the existing on-premises services.

### Objectives
- Assess available Windows Server updates using Azure Update Manager.
- Deploy applicable updates to `AZ-LAB01` and verify server health following maintenance.
- Configure an Azure Recovery Services vault and an appropriate Azure VM backup policy.
- Create and verify backup recovery points before and after the maintenance exercise.
- Demonstrate file-level recovery of deleted and modified test files from an earlier recovery point.
- Validate recovered data against previously recorded SHA-256 file hashes.
- Review backup security, retention, outbound connectivity and ongoing Azure costs.

## Scope
Phase 6 focuses on the existing Azure VM `AZ-LAB01`, deployed in Phase 1 and monitored during Phase 4. It reuses the Azure resource group, VM, workload subnet, NSG, managed disks and budget established in earlier phases. A Recovery Services vault and Azure Backup policy are introduced to protect the VM, while Azure Update Manager is used for guest OS update assessment and installation.

A dedicated directory on the `AZ-LAB01` data disk `E:` is used for controlled recovery testing. The recovery exercise targets synthetic data and does not alter the existing on-premises file server or the Azure File Sync pilot directory.
> Azure VM Backup in this phase protects `AZ-LAB01` and its included managed disks. It does not back up the `FS01` server or the Azure Files share created in Phase 5. Azure File Sync provides synchronisation rather than an independent versioned backup.
> 
> `AZ-LAB01` does not retain a permanent public IP. Temporary outbound connectivity may be required for Windows Update and other VM-initiated service operations. No public inbound RDP access is introduced.

## Phase 6 Infrastructure

|Component|Configuration|
|---|---|
|Azure Region|`Sweden Central`|
|Resource Group|`rg-hybridlab-swc-01`|
|Azure VM|`AZ-LAB01` — `10.30.1.4`|
|Operating System|`Windows Server 2025 Datacenter Gen 2`|
|VM Size|`Standard_B2s_v2`|
|Boot Security|`Trusted Launch`, `Secure Boot`, `vTPM`|
|OS Disk|`Standard SSD LRS` — `127 GiB`|
|Data Disk|`disk-az-lab01-data-01` — `4 GiB`, `E:`|
|VM Administration|`Azure Bastion Developer`|
|Update Service|`Azure Update Manager`|
|Backup Service|`Azure Backup`|
|Recovery Services Vault||
|Backup Policy||
|Backup Storage Redundancy|`Locally Redundant Storage (LRS)`|
|Recovery Test Directory|`E:\Recovery-Test`|
|File Recovery Host||
|Existing Cost Budget|`£20` monthly; alerts at `50%`, `80%` and `100%`|

## Patching and Recovery Architecture
The implementation uses Azure Update Manager to assess and install applicable Windows updates on `AZ-LAB01`. Azure Backup protects the VM using a Recovery Services vault, creating recovery points from which selected files can be retrieved without restoring the full VM.

Test files are created on the VM's data disk before the first backup. After maintenance, a second recovery point captures the updated state. An earlier recovery point is then used to retrieve intentionally deleted or modified files and verify their integrity.

<i>Insert Architecture Diagram</i>

> Deployment Decision?

## Pre-Maintenance Preparation

### Server Health and Update Baseline
Before configuring the backup and patching services, `AZ-LAB01` was checked to establish a baseline of operating system health, disk availability and previously installed updates.

```powershell
# Operating system and uptime.
Get-ComputerInfo |
    Select-Object CsName, WindowsProductName, WindowsVersion, OsBuildNumber

Get-CimInstance Win32_OperatingSystem |
    Select-Object CSName, LastBootUpTime

# Available volumes and free capacity.
Get-Volume |
    Select-Object DriveLetter, FileSystem, HealthStatus, SizeRemaining

# Recently installed updates.
Get-HotFix |
    Sort-Object InstalledOn -Descending |
    Select-Object -First 15
```

|Validation|Result|
|---|---|
|VM Power State|[Running / other; date]|
|Windows Server Version/OS Build|[Record observed version/build]|
|OS and Data Volumes|[Health and free-space result]|
|Azure VM Agent|[Service status]|
|Last Installed Windows Update|[KB and installation date, if available]|
|Pending Reboot|[Observed state and how checked]|

### Recovery Test Data Preparation
A dedicated recovery directory was prepared on `AZ-LAB01`'s separate managed data disk. The test files were created before the initial backup so that their original versions could be recovered later.

```powershell
New-Item -Path 'E:\Recovery-Test' -ItemType Directory -Force

'Phase 6 - Azure Backup Recovery Test' |
    Set-Content 'E:\Recovery-Test\Recovery-Test.txt'

'Original configuration: Version 1' |
    Set-Content 'E:\Recovery-Test\Config-Test.txt'

'This file will be deleted after backup.' |
    Set-Content 'E:\Recovery-Test\Delete-Me.txt'

Get-FileHash 'E:\Recovery-Test\*.txt' -Algorithm SHA256 |
    Export-Csv 'C:\Users\Public\Recovery-Baseline.csv' -NoTypeInformation
```
> The baseline hash file was also saved outside the recovery-test directory to support later comparison with restored files.

<i>Screenshot of recovery test files</i>

## Azure Backup Configuration

### Recovery Services Vault Deployment
A Recovery Services vault was configured to hold the protected VM's backup data. Its region, replication setting and security configuration were reviewed before enabling backup protection.

|Setting|Configuration|
|---|---|
|Vault Name||
|Resource Group|`rg-hybridlab-swc-01`|
|Region|`Sweden Central`|
|Backup Storage Redundancy|`LRS`|
|Soft Delete||
|Immutability||
|Resource Tags||

<i>Recovery Services Vault Screenshot</i>

### Azure VM Backup Policy
A backup policy was configured for `AZ-LAB01`. The VM uses Trusted Launch, so the selected backup policy and configuration method were reviewed for compatibility.

|Setting|Configuration|
|---|---|
|Policy Name||
|Protected Workload|`Azure Virtual Machine`|
|Policy Subtype||
|Schedule|[Actual frequency, time and time zone]|
|Daily Retention||
|Instant Restore/Snapshot Retention||
|Long-term Retention|[Enabled/disabled, with details]|
|Target VM|`AZ-LAB01`|

<i>Backup Policy Screenshot</i>

### Baseline Backup and Recovery Point
Azure VM backup was enabled for `AZ-LAB01`. An on-demand backup was initiated before the patching exercise to preserve the original recovery-test files and the pre-maintenance server state.


