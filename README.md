# Hybrid Azure Infrastructure Lab

`Azure Administration` · `Windows Server` · `Hybrid Active Directory/Entra ID` · `Azure Arc` · `Azure Monitor` · `Azure Files` · `Patching & Backup` · `PowerShell` · `Bicep` · `Infrastructure Troubleshooting`

> <b>Extending an existing Windows Server 2025 lab into Microsoft Azure</b>   
> A hands-on infrastructure engineering project focused on hybrid identity, Azure administration, server management, monitoring, storage, recovery, and automation.

## Project Overview
This project extends my completed [Windows Server 2025 Enterprise-Style Infrastructure Lab](https://github.com/simonbaron-it/HomeLab-EnterpriseEnvironment) into Microsoft Azure. The existing environment provides on-premises Active Directory, DNS, DHCP, routing, file services, centralised administration, monitoring, and PowerShell automation.

The project introduces Azure infrastructure and hybrid management while retaining the existing on-premises environment. It focuses on practical infrastructure engineering responsibilities, including deployment, administration, identity integration, monitoring, maintenance, automation and troubleshooting.

### Project objectives
- Deploy and securely administer Azure networking and a Windows Server VM.
- Synchronise selected on-premises Active Directory identities to Microsoft Entra ID.
- Manage existing Windows servers using Azure Arc.
- Collect operational logs, query telemetry, and configure alerts with Azure Monitor.
- Extend a test file share from the on-prem file server into Azure using Azure File Sync.
- Test patching, Azure VM backup, and recovery processes.
- Automate Azure deployment and administration using PowerShell and Bicep.
- Record incident investigations, root causes, remediation steps, and validation results.

## Implementation Roadmap

|Phase|Engineering Focus|Status|
|---|---|---|
|01 - Azure Foundation|Resource governance, networking, NSGs, VM deployment and secure administration|![Complete](https://img.shields.io/badge/-Complete-brightgreen)|
|02 - Hybrid Identity|Entra Cloud Sync, pilot users/groups, password sync and troubleshooting|![Complete](https://img.shields.io/badge/-Complete-brightgreen)|
|03 - Hybrid Management|Onboard `FS01` and `MGMT01` to Azure Arc|![Complete](https://img.shields.io/badge/-Complete-brightgreen)|
|04 - Monitoring|Log Analytics, Data Collection Rules, KQL and alert testing|![In Progress](https://img.shields.io/badge/-In%20Progress-yellow)|
|05 - Hybrid Storage|Azure Files, File Sync and synchronisation diagnostics|![Planned](https://img.shields.io/badge/-Planned-lightgrey)|
|06 - Operations|Update Manager, Azure VM backup and file-level recovery|![Planned](https://img.shields.io/badge/-Planned-lightgrey)|
|07 - Automation|Azure PowerShell, Bicep and hybrid health checks|![Planned](https://img.shields.io/badge/-Planned-lightgrey)|
|08 - Validation|Simulated incidents, runbooks, final testing and documentation|![Planned](https://img.shields.io/badge/-Planned-lightgrey)|

## Technical Documentation
The build will be documented as a series of practical engineering guides, listed below. Screenshots and command output will be included where they provide useful evidence.

|Area|Documentation|
|---|---|
|Azure Foundation|[Azure Foundation & Infrastructure Deployment](Documentation/01%20Azure%20Foundation.md)|
|Hybrid Identity|[AD & Entra ID - Entra Cloud Sync](Documentation/02%20Hybrid%20Identity.md)|
|Hybrid Management|[Azure Arc & Hybrid Server Management](Documentation/03%20Hybrid%20Management.md)|
|Monitoring|![In Progress](https://img.shields.io/badge/Status-In%20Progress-yellow)|
|File Services|![Planned](https://img.shields.io/badge/Status-Planned-lightgrey)|
|Patching and Recovery|![Planned](https://img.shields.io/badge/Status-Planned-lightgrey)|
|Automation|![Planned](https://img.shields.io/badge/Status-Planned-lightgrey)|
|Testing|![Planned](https://img.shields.io/badge/Status-Planned-lightgrey)|

## Technical Skills Being Developed
- <b>Azure Infrastructure Administration:</b> Deploy, configure and secure Azure Virtual Machines, Virtual Networks, Network Security Groups, RBAC and cloud resources.
- <b>Hybrid Identity & Server Management:</b>  Integrate on-premises Active Directory with Microsoft Entra ID using Cloud Sync, and manage Windows servers through Azure Arc.
- <b>Monitoring & Troubleshooting:</b>  Implement Azure Monitor, Log Analytics, KQL queries and alerting to monitor infrastructure, investigate failures and resolve operational issues.
- <b>Hybrid Storage, Backup & Recovery:</b>  Configure Azure Files and File Sync, manage server patching through Azure Update Manager, and validate backup and recovery procedures.
- <b>PowerShell & Infrastructure as Code:</b>  Automate infrastructure administration, health checks and reporting using PowerShell, and deploy Azure resources using Bicep.
- <b>Infrastructure Security & Governance:</b>  Apply least-privilege access, network security controls, resource governance and Azure cost management practices.
