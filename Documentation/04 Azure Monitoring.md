# Azure Monitoring
> This documentation covers the implementation of centralised infrastructure monitoring across the existing on-premises Windows Server environment and Microsoft Azure using Azure Monitor and Log Analytics.
> 
> The implementation focuses on Azure Monitor Agent, Data Collection Rules, Windows event and performance data, Kusto Query Language (KQL), operational alerting and monitoring troubleshooting.

## Scenario and Objectives

### Business Scenario
A simulated organisation operates Windows Server 2025 infrastructure across an on-premises Hyper-V lab and Microsoft Azure. Phase 3 introduced Azure Arc-enabled Servers, providing centralised visibility and management of the existing `MGMT01` and `FS01` servers.

As the organisation extends its use of Azure, the infrastructure team requires a method of collecting and analysing operational data across both environments. The objective is to implement centralised monitoring that supports server health checks, investigation of Windows errors, early fault detection and notification of infrastructure issues without replacing the existing Windows administration tools.

### Objectives
- Deploy a Log Analytics workspace for centralised monitoring data.
- Configure Data Collection Rules to collect selected Windows events and performance counters.
- Install and validate Azure Monitor Agent on the Azure Arc-enabled `MGMT01` and `FS01` servers, and test monitoring of the Azure VM `AZ-LAB01` when online.
- Use KQL to investigate server heartbeats, Windows events, CPU usage, available memory, disk space and monitoring data volume.
- Configure an Azure Monitor Action Group and validate email notification delivery.
- Create and test an application-event alert and a missing-heartbeat alert.
- Troubleshoot a controlled monitoring issue and verify that existing infrastructure services remain operational.
- Review monitoring security, data collection scope and Azure costs.

## Scope
Phase 4 introduces Azure Monitor and Log Analytics to the hybrid infrastructure established during Phases 1–3. The primary monitoring targets are `MGMT01` and `FS01`, which are already connected to Azure through Azure Arc. The Azure-hosted `AZ-LAB01` virtual machine will be included in a controlled monitoring test while powered on and provided with supported outbound connectivity.

The existing Windows Admin Center, Windows Event Forwarding, Microsoft Entra Cloud Sync and file services remain in place. The implementation does not replace these services or collect the entire Windows Security or ForwardedEvents logs.

> `AZ-LAB01` is normally deallocated outside lab sessions and does not retain a permanent public IP address. It will therefore not be included in continuous availability alerting. No site-to-site VPN or inbound management port forwarding is required for the on-premises monitoring targets.

## Phase 4 Architecture

|Component|Configuration|
|---|---|
|AD Domain|`baron.example.com`|
|Management Server|`MGMT01` — `10.10.10.30`|
|File Server|`FS01` — `10.10.10.20`|
|Azure VM|`AZ-LAB01` — `10.30.1.4`|
|Azure Region|`Sweden Central`|
|Resource Group|`rg-hybridlab-swc-01`|
|Existing Hybrid Management|`Azure Arc-enabled Servers`|
|Monitoring Service|`Azure Monitor`|
|Log Analytics Workspace|`law-hybridlab-swc-01`|
|Data Collection Rule|`dcr-hybrid-windows-swc-01`|
|Monitoring Agent|`Azure Monitor Agent`|
|Data Collection|`Windows events`, `performance counters`, `agent heartbeats`|
|Alert Notification|`Email via Azure Monitor Action Group`|

## Monitoring Architecture
Azure Monitor Agent collects selected operating-system events and performance data from the Azure Arc-enabled Windows servers and the Azure VM, then sends it to Azure Monitor for ingestion. A Data Collection Rule defines the collection scope and specifies the destination Log Analytics workspace. KQL queries and Azure Monitor alert rules use the collected data for investigations and fault detection.

<img src="../Images/Phase%204%20Azure%20Monitoring%20Architecture.png" width="1000"/>

> **Deployment Decision:** `MGMT01` and `FS01` are the primary monitoring targets because they remain available through the existing on-premises environment and Azure Arc connectivity. `AZ-LAB01` only reports telemetry while running with a functional outbound Azure Monitor connection. This avoids false availability alerts and unnecessary Azure VM compute charges.

## Log Analytics Workspace

### Workspace Deployment
A Log Analytics workspace was deployed in the existing Azure resource group to store and query monitoring data.

|Setting|Configuration|
|---|---|
|Workspace Name|`law-hybridlab-swc-01`|
|Resource Group|`rg-hybridlab-swc-01`|
|Region|`Sweden Central`|
|Pricing Model|`Pay-as-you-go`|
|Data Retention|`Default analytics retention`|
|Resource Tags|`Existing phase 1 tags`|
|Deployment Status|`Active`|

<img src="../Images/Log%20Analytics%20Workspace.png" width="1000"/>

### Data Retention and Cost Controls
A limited monitoring scope is used for this lab, including appropriate retention settings, ingested data volume and estimated costs.
> The £20 monthly budget configured in Phase 1, including alerts at £10, £16 and £20, remains in place. Log ingestion, retention beyond included allowances, and log-search alert rules may incur charges.

## Data Collection Rules

### DCR Configuration
A Windows Data Collection Rule was created and associated with the Azure Arc-enabled `MGMT01` and `FS01` resources. `AZ-LAB01` was added separately once outbound connectivity was configured.

|Setting|Configuration|
|---|---|
|DCR Name|`dcr-hybrid-windows-swc-01`|
|Region|`Sweden Central`|
|Platform Type|`Windows`|
|Primary Targets|`MGMT01`, `FS01`|
|Additional Test Target|`AZ-LAB01`|
|Destination|`law-hybridlab-swc-01`|
|Data Sources|`Windows Event Logs`, `Performance Counters`|
|DCR Association Status|`Passed initial checks`|

<img src="../Images/Data%20Collection%20Rule%20-%20Resources.png" width="1000"/>

### Windows Event Collection
Selected System and Application events were collected to support investigation of operational faults, without ingesting unnecessary informational or forwarded events.

|Event Log|Severity Levels|Purpose|
|---|---|---|
|`System`|`Critical`, `Error`, `Warning`|Operating system and service diagnostics|
|`Application`|`Critical`, `Error`, `Warning`|Application errors and controlled alert testing|
|`Security`|Not collected initially|Avoid unnecessary volume and sensitive event collection|
|`ForwardedEvents`|Not collected initially|Avoid duplicate ingestion of forwarded events on `MGMT01`|

### Performance Counter Collection
The DCR collects the following Windows performance counters at 60-second intervals.

|Counter|Interval|Purpose|
|---|---|---|
|`\Processor(_Total)\% Processor Time`|60 seconds|Total CPU utilisation|
|`\Memory\Available MBytes`|60 seconds|Available system memory|
|`\LogicalDisk(*)\% Free Space`|60 seconds|Percentage of free disk space by logical volume|

<img src="../Images/Data%20Collection%20Rule%20-%20Data%20Sources.png" width="1000"/>

## Azure Monitor Agent Deployment

### Arc-Enabled Server Monitoring
Azure Monitor Agent was deployed through the DCR resource associations. The extension `AzureMonitorWindowsAgent` and the associated DCR were verified for each Arc-enabled server.

|Server|Azure Arc Status|AMA Extension|DCR Association|Data Reporting|
|---|---|---|---|---|
|`MGMT01`|`Connected`|`Succeeded`|`Associated`|`Reporting`|
|`FS01`|`Connected`|`Succeeded`|`Associated`|`Reporting`|

> Azure Monitor Agent process activity and existing Windows service states were checked using PowerShell:

```powershell
# Verify Azure Monitor Agent activity.
Get-Process -Name MonAgentCore -ErrorAction SilentlyContinue |
    Select-Object ProcessName, Id, StartTime

# Verify existing Windows Event Forwarding and Cloud Sync services on `MGMT01`.
Get-Service Wecsvc, AADConnectProvisioningAgent, AzureADConnectAgentUpdater

# Verify the file server service and existing shares on `FS01`.
Get-Service LanmanServer
Get-SmbShare | Select-Object Name, Path, Description
```

<img src="../Images/AMA%20Extension.png" width="1000"/>

### Azure VM Monitoring
`AZ-LAB01` was associated with the Data Collection Rule and configured with the Azure Monitor Agent. Telemetry collection was validated while the VM was running and had outbound connectivity to the required Azure Monitor endpoints. Monitoring functionality was verified through the Azure VM extension status and Log Analytics workspace queries.

|Setting|Configuration|
|---|---|
|VM|`AZ-LAB01`|
|Existing Private IP|`10.30.1.4`|
|Monitoring Extension|`AzureMonitorWindowsAgent`|
|Outbound Connectivity During Test|Temporary Public IP|
|VM Managed Identity Requirement|System-assigned managed identity|
|Telemetry Validation|AMA heartbeat and monitoring data received in Log Analytics|
|Post-Test VM State|Deallocated after lab session|

> No permanent inbound RDP access was introduced. Temporary Public IP address was deleted after validation testing.

## Log Analytics and KQL

### Monitoring Data Validation
KQL queries were run against `law-hybridlab-swc-01` to verify that monitored machines report heartbeat, Windows event and performance data.

#### Heartbeat Monitoring
<img src="../Images/KQL%20Query%201.png" width="1000"/>

#### Windows Event Analysis
<img src="../Images/KQL%20Query%202.png" width="1000"/>

#### CPU, memory and disk Monitoring
<img src="../Images/KQL%20Query%203.png" width="1000"/>

## Azure Monitor Alerting

### Action Group Configuration
An Azure Monitor Action Group was configured to send email notifications when a monitoring rule detects a defined condition.

|Setting|Configuration|
|---|---|
|Action Group|`ag-hybridlab-email-01`|
|Resource Group|`rg-hybridlab-swc-01`|
|Notification Method|Email|
|Recipient|Lab administrator email address|

<img src="../Images/Action%20Group.png" width="1000"/>

### Application Event Alert

A controlled Application error was generated on `FS01` using an existing registered event source. The event was collected by AMA and used to validate an Azure Monitor log-search alert without interrupting production-style Windows services.

<img src="../Images/FS01%20-%20Alert%20Rule.png" width="1000"/>

#### Alert email delivery validation

<img src="../Images/Alert%20Email.png" width="1000"/>

## Validation and Testing

## Security and Operational Considerations

### Monitoring Security
The monitoring solution is intended to provide operational visibility without changing the existing on-premises Windows security model.

Key considerations:
- Maintained outbound-only connectivity from on-premises servers to Azure services; no inbound port forwarding was required.
- Applied least-privilege Azure RBAC to monitoring resources, workspace queries, DCR management and extension deployment.
- Restricted Windows event collection to operationally useful data rather than broadly ingesting security or forwarded logs.
- Avoided unnecessary duplicate event collection from `MGMT01`, which already provides Windows Event Forwarding.
- Monitored the additional workload on `MGMT01`, which also hosts Windows Admin Center and the privileged Microsoft Entra Cloud Sync agent.

### Cost Management
The Phase 1 £20 monthly budget and alert thresholds were retained. Monitoring costs were reviewed using Log Analytics usage information and Azure Cost Management.
- Used limited System/Application event severities and selected performance counters.
- Avoided broad Security and ForwardedEvents collection.
- Checked the cost of log-search alert rules and any notifications.
- Deallocated `AZ-LAB01` outside planned monitoring tests.
- Removed temporary test alerts when no longer required.

## Phase 4 Summary
Phase 4 introduced centralised monitoring across the hybrid infrastructure using Azure Monitor and Log Analytics, providing operational visibility into both on-premises and Azure-hosted Windows Server systems.
- **Centralised Monitoring:** Deployed a Log Analytics workspace in `Sweden Central` to consolidate monitoring data from the Azure Arc-enabled `MGMT01` and `FS01` servers and the Azure VM `AZ-LAB01`.
- **Data Collection:** Configured Data Collection Rules and Azure Monitor Agent to collect selected Windows event logs, performance counters and agent heartbeats, with telemetry reporting validated.
- **KQL and Diagnostics:** Used Kusto Query Language to analyse server heartbeats, Windows event logs, CPU utilisation, available memory and disk capacity; confirming successful telemetry collection and centralised visibility across monitored systems.
- **Alerting:** Configured Azure Monitor alert rules for application errors, with an Action Group providing email notifications. Validated alert triggering and notification delivery following a controlled test scenario.
- **Security and Cost:** Restricted monitoring to selected infrastructure resources and event sources, retained outbound-only connectivity and reviewed monitoring expenditure against the existing £20 monthly budget.

## Next Steps
**Phase 5:** Hybrid Storage will focus on extending selected `FS01` file services into Azure using Azure Files and Azure File Sync.

The monitoring capabilities introduced in Phase 4 will provide a basis for investigating server health and operational events while hybrid file synchronisation is configured and validated.





