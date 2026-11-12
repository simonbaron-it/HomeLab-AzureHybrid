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
|Log Analytics Workspace||
|Data Collection Rule||
|Monitoring Agent|`Azure Monitor Agent`|
|Data Collection|Windows events, performance counters and agent heartbeats|
|Alert Notification|Email via Azure Monitor Action Group|

## Monitoring Architecture
Azure Monitor Agent collects selected operating-system events and performance data from the Azure Arc-enabled Windows servers and the Azure VM. A Data Collection Rule defines the collection scope and sends the data to a central Log Analytics workspace. KQL queries and Azure Monitor alert rules use the collected data for investigations and fault detection.

<i>Architecture Diagram</i>

> **Deployment Decision:** `MGMT01` and `FS01` are the primary monitoring targets because they remain available through the existing on-premises environment and Azure Arc connectivity. `AZ-LAB01` only reports telemetry while running with a functional outbound Azure Monitor connection. This avoids false availability alerts and unnecessary Azure VM compute charges.

## Log Analytics Workspace

### Workspace Deployment
A Log Analytics workspace was deployed in the existing Azure resource group to store and query monitoring data.

|Setting|Configuration|
|---|---|
|Workspace Name||
|Resource Group|`rg-hybridlab-swc-01`|
|Region|`Sweden Central`|
|Pricing Model||
|Data Retention||
|Resource Tags|Existing Phase 1 tags|
|Deployment Status||

<i>Screenshot</i>

### Data Retention and Cost Controls
A limited monitoring scope is used for this lab, including appropriate retention settings, ingested data volume and estimated costs.
> The £20 monthly budget configured in Phase 1, including alerts at £10, £16 and £20, remains in place. Log ingestion, retention beyond included allowances, and log-search alert rules may incur charges.

<i>Data retention table?</i>

## Data Collection Rules

### DCR Configuration
A Windows Data Collection Rule was created and associated with the Azure Arc-enabled `MGMT01` and `FS01` resources. `AZ-LAB01` was added separately once outbound connectivity was configured.

|Setting|Configuration|
|---|---|
|DCR Name||
|Region|`Sweden Central`|
|Platform Type|`Windows`|
|Primary Targets|`MGMT01`, `FS01`|
|Additional Test Target|`AZ-LAB01`|
|Destination||
|Data Sources|Windows Event Logs and Performance Counters|
|DCR Association Status||

### Windows Event Collection
Selected System and Application events were collected to support investigation of operational faults, without ingesting unnecessary informational or forwarded events.

|Event Log|Severity Levels|Purpose|
|---|---|---|
|`System`|`Critical`, `Error`, `Warning`|Operating system and service diagnostics|
|`Application`|`Critical`, `Error`, `Warning`|Application errors and controlled alert testing|
|`Security`|Not collected initially|Avoid unnecessary volume and sensitive event collection|
|`ForwardedEvents`|Not collected initially|Avoid duplicate ingestion of forwarded events on `MGMT01`|

The equivalent custom XPath filters are:
```
System!*[System[(Level=1 or Level=2 or Level=3)]]
Application!*[System[(Level=1 or Level=2 or Level=3)]]
```

### Performance Counter Collection
The DCR collects the following Windows performance counters at 60-second intervals

|Counter|Interval|Purpose|
|---|---|---|
|`\Processor(_Total)\% Processor Time`|60 seconds|Total CPU utilisation|
|`\Memory\Available MBytes`|60 seconds|Available system memory|
|`\LogicalDisk(*)\% Free Space`|60 seconds|Available disk capacity by volume|

```powershell
# Check that the selected performance counters exist locally.
Get-Counter -Counter @(
    '\Processor(_Total)\% Processor Time',
    '\Memory\Available MBytes',
    '\LogicalDisk(*)\% Free Space'
)
```

<i>Screenshot</i>

## Azure Monitor Agent Deployment
