# Hybrid Identity
> This documentation covers the integration of the existing on-premises Active Directory environment with Microsoft Entra ID using Microsoft Entra Cloud Sync.
>
> The implementation focuses on hybrid identity synchronisation, user and group management, authentication, security and operational troubleshooting.

## Scenario and Objectives
### Business Scenario
A simulated organisation operates an existing Windows Server 2025 infrastructure with Active Directory providing centralised identity and authentication services. As the organisation adopts Microsoft Azure, it requires a method of extending existing on-premises identities into Microsoft Entra ID without replacing its Active Directory environment.

The objective is to establish hybrid identity synchronisation, allowing selected on-premises users and security groups to be managed through Active Directory while being available for cloud authentication and access to Microsoft Entra-integrated services.

### Objectives
- Integrate the existing Active Directory environment with Microsoft Entra ID.
- Configure a verified custom domain for consistent on-premises and cloud user sign-in names.
- Deploy Microsoft Entra Cloud Sync using an existing Windows Server VM.
- Synchronise selected Active Directory users and security groups.
- Implement and validate Password Hash Synchronisation (PHS).
- Test user provisioning, attribute updates, group membership changes and account disablement.
- Simulate, troubleshoot and recover from a synchronisation failure.

## Scope
Phase 2 introduces hybrid identity synchronisation between the existing `baron.example.com` Active Directory domain and the Microsoft Entra tenant established during Phase 1.

Microsoft Entra Cloud Sync is deployed on the existing `MGMT01` server to avoid additional Hyper-V resource requirements. A dedicated pilot OU is used to restrict synchronisation to selected test accounts and security groups.

> The existing Active Directory DNS namespace remains unchanged. The purchased custom domain is configured as an alternative UPN suffix for cloud sign-in, rather than replacing the existing AD domain.

## Phase 2 Infrastructure

|Component|Configuration|
|---|---|
|AD Domain|`baron.example.com`|
|Primary Domain Controller|`DC01` - `10.10.10.10`|
|Secondary Domain Controller|`DC02` - `10.10.10.11`|
|Cloud Sync Host|`MGMT01` - `10.10.10.30`|
|Server Network|`10.10.10.0/24`|
|Microsoft Entra Tenant|`Baron Hybrid Home Lab`|
|Custom Domain|`simonbaron.co.uk`|
|Synchronisation Method|`Microsoft Entra Cloud Sync`|
|Synchronisation Scope|`CloudSync-Pilot` OU|

## Hybrid Identity Architecture
The hybrid identity implementation uses the existing Active Directory domain controllers as the authoritative source for synchronised identities.

The Microsoft Entra Cloud Sync provisioning agent runs on `MGMT01` and communicates with Microsoft Entra ID using outbound HTTPS connections. Selected users, groups and password hashes are synchronised to the cloud, while on-premises Active Directory remains responsible for managing the source identities.

<i> Hybrid identity architecture diagram</i>

> **Deployment Decision:** A dedicated Cloud Sync VM was originally considered, but available Hyper-V resources were limited. The provisioning agent was therefore deployed on MGMT01, which already provides management and monitoring services.
>
> This avoids introducing another virtual machine, although it requires additional resource monitoring and security considerations because the Cloud Sync agent is a privileged identity component.

## Hybrid Identity Configuration
### Custom Domain and UPN Configuration
A custom domain was purchased and added to the Microsoft Entra tenant to provide a consistent sign-in namespace for hybrid identities.

The domain's ownership was verified through Microsoft Entra ID before being configured as an alternative UPN suffix in Active Directory.

|Component|Configuration|
|---|---|
|Existing AD DNS Domain|`baron.example.com`|
|Verified Entra Domain|`simonbaron.co.uk`|
|Alternative AD UPN Suffix|`simonbaron.co.uk`|
|Example Pilot UPN|`amorgan@simonbaron.co.uk`|
|Domain Verification Method|`DNS TXT Record`|

The alternative UPN suffix was added through Active Directory Domains and Trusts, allowing selected AD accounts to use the verified custom domain for cloud authentication.

> Existing domain controller names, AD DNS services and the internal domain namespace were retained.

<i>Insert screenshot: Verified custom domain in Microsoft Entra ID</i>

<i>Insert screenshot: Alternative UPN suffix in Active Directory Domains and Trusts</i>

## Active Directory Preparation
Before enabling synchronisation, Active Directory health and replication were checked to confirm the existing environment was operating correctly.

The following checks were performed:
- Domain controller health using dcdiag.   
<i>Insert PowerShell or screenshot</i>
- Active Directory replication using repadmin.   
<i>Insert PowerShell or screenshot</i>
- DNS resolution and domain controller discovery.   
<i>Insert PowerShell or screenshot</i>
- Verification of available domain controller backups.   
<i>Insert PowerShell or screenshot</i>

### Pilot Organisational Unit
A dedicated OU named `CloudSync-Pilot` was created to separate test identities from existing users and administrative accounts.

The following pilot accounts were created:

|User|AD Username|Cloud UPN|
|---|---|---|
|Alex Morgan|amorgan|amorgan@simonbaron.co.uk|
|Jamie Taylor|jtaylor|jtaylor@simonbaron.co.uk|
|Sam Carter|scarter|scarter@simonbaron.co.uk|

Two Global Security Groups were also created:

|Group|Initial Members|
|---|---|
|`CloudSync-IT`|`Alex Morgan`, `Jamie Taylor`|
|`CloudSync-Operations`|`Sam Carter`|

> The `CloudSync-Pilot` OU allows synchronisation and account lifecycle changes to be tested without modifying the existing infrastructure users or privileged security groups.

<i>screenshot: Pilot OU containing users and groups</i>

## Cloud Sync Agent Deployment
The Microsoft Entra Cloud Sync provisioning agent was installed on the existing `MGMT01` virtual machine.

Before installation, server performance, Active Directory connectivity and outbound Microsoft service connectivity were reviewed.

|Component|Configuration|
|---|---|
|Server|`MGMT01`|
|Operating System|`Windows Server 2025`|
|Existing Roles|`Windows Admin Center`, `Windows Event Forwarding`|
|Additional Component|`Microsoft Entra Cloud Sync Provisioning Agent`|
|Agent Version|`-`|
|Connected AD Domain|`baron.example.com`|
|Service Account|`-`|
|Agent Status|`-`|

The provisioning agent was registered with the existing Microsoft Entra tenant and configured to communicate with the on-premises Active Directory domain using a group Managed Service Account (gMSA).

Following installation, the agent's registration and service status were verified through the Microsoft Entra admin centre.

The existing Windows Admin Center and Windows Event Forwarding functionality was also checked to ensure the additional service had not disrupted original `MGMT01` responsibilities.

<i>screenshot: Registered Cloud Sync agent in Entra</i>

### Synchronisation Configuration
A new AD to Microsoft Entra ID sync configuration was created using the registered provisioning agent.

