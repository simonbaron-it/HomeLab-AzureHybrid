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
- Implement and validate Password Hash Synchronisation.
- Test user provisioning, attribute updates, group membership changes and account disablement.

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

The Microsoft Entra Cloud Sync provisioning agent runs on `MGMT01` and communicates with Microsoft Entra ID using outbound HTTPS connections. Selected users and groups are provisioned into Microsoft Entra ID, with password hash synchronisation enabling cloud authentication using their on-premises credentials.

<img src="../Images/Phase%202%20Hybrid%20Identity%20Architecture.png" width="1000"/>

> **Deployment Decision:** A dedicated Cloud Sync VM was originally considered, but available Hyper-V resources were limited. The provisioning agent was therefore deployed on `MGMT01`, which already provides management and monitoring services.
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

The alternative UPN suffix was added through Active Directory Domains and Trusts, allowing selected AD accounts to use the verified custom domain for cloud authentication. Existing domain controller names, AD DNS services and the internal domain namespace were retained.

> Entra ID custom domain registration
<img src="../Images/Custom%20Domain%20-%20Entra.png" width="1000"/>

> AD Domains and Trusts alternative UPN suffix
<img src="../Images/Custom%20Domain%20-%20AD.png" width="1000"/>

## Active Directory Preparation
Before enabling synchronisation, Active Directory health and replication were checked to confirm the existing environment was operating correctly.

The following checks were performed:
- Domain controller health using dcdiag.   
```powershell
dcdiag /q
```
- Active Directory replication using repadmin.   
```powershell
repadmin /replsummary
```
- DNS resolution and domain controller discovery.   
```powershell
Resolve-DnsName login.microsoftonline.com

Get-ADDomainController -Filter * |
    Select-Object HostName, IPv4Address
```
- Verification of available domain controller backups.   
```powershell
wbadmin get versions
```

### Pilot Organisational Unit
A dedicated OU named `CloudSync-Pilot` was created to separate test identities from existing users and administrative accounts.

The following pilot accounts were created:

|User|AD Username|Cloud UPN|
|---|---|---|
|Alex Morgan|`amorgan`|`amorgan@simonbaron.co.uk`|
|Jamie Taylor|`jtaylor`|`jtaylor@simonbaron.co.uk`|
|Sam Carter|`scarter`|`scarter@simonbaron.co.uk`|

Two global security groups were also created:

|Group|Initial Members|
|---|---|
|`CloudSync-IT`|`Alex Morgan`, `Jamie Taylor`|
|`CloudSync-Operations`|`Sam Carter`|

> The `CloudSync-Pilot` OU allows synchronisation and account lifecycle changes to be tested without modifying the existing infrastructure users or privileged security groups.

<img src="../Images/CloudSync%20AD%20OU.png" width="1000"/>

## Cloud Sync Agent Deployment
The Microsoft Entra Cloud Sync provisioning agent was installed on the existing `MGMT01` virtual machine.

Before installation, server performance, Active Directory connectivity and outbound Microsoft service connectivity were reviewed.

|Component|Configuration|
|---|---|
|Server|`MGMT01`|
|Operating System|`Windows Server 2025`|
|Existing Roles|`Windows Admin Center`, `Windows Event Forwarding`|
|Additional Component|`Microsoft Entra Cloud Sync Provisioning Agent`|
|Agent Version|`1.1.2507.0`|
|Connected AD Domain|`baron.example.com`|
|Service Account|`Group Managed Service Account` (gMSA)|
|Agent Status|`Active`|

The provisioning agent was registered with the existing Microsoft Entra tenant and configured to communicate with the on-premises Active Directory domain using a group Managed Service Account (gMSA).

Following installation, the agent's registration and service status were verified through the Microsoft Entra admin centre.

> The existing Windows Admin Center and Windows Event Forwarding functionality was also checked to ensure the additional service had not disrupted original `MGMT01` responsibilities.

<img src="../Images/Entra%20Connect%20Agent%20Active.png" width="1000"/>

### Synchronisation Configuration
A new **AD to Microsoft Entra ID sync** configuration was created using the registered provisioning agent.

|Setting|Configuration|
|---|---|
|Source Directory|`baron.example.com`|
|Target Directory|`Baron Hybrid Home Lab`|
|Provisioning Agent|`MGMT01`|
|Synchronisation Direction|`AD to Microsoft Entra ID`|
|Scope|`Selected Organisational Units`|
|Selected OU|`CloudSync-Pilot`|
|Password Hash Synchronisation|`Enabled`|
|Accidental Deletion Protection|`Enabled`|
|Accidental Deletion Threshold|`2 objects`|

> Synchronisation was restricted to the pilot OU rather than the entire Active Directory environment.
>
> Default attribute mappings were reviewed, including `userPrincipalName`, `displayName`, `givenName`, `sn`, `department` and `title`.

<img src="../Images/Entra%20Connect%20Agent%20Properties.png" width="1000"/>

<img src="../Images/Entra%20Connect%20Agent%20Scope.png" width="1000"/>

### Initial Identity Synchronisation
The pilot users and security groups were checked in the Entra admin centre to verify that the expected objects had been created.

**Initial synchronisation results:**
- Users successfully synchronised: **`3`** 
- Groups successfully synchronised: **`2`**
- Synchronisation errors: **`0`**
- Cloud UPN suffix: **`simonbaron.co.uk`**
- Initial provisioning status: **`Successful`**

<img src="../Images/Entra%20Cloud%20Sync%20Users.png" width="1000"/>

<img src="../Images/Entra%20Cloud%20Sync%20Groups.png" width="1000"/>

## Identity Lifecycle Validation and Testing
A series of tests was performed to confirm that user lifecycle changes originating in on-premises Active Directory were correctly reflected in Microsoft Entra ID.

|Test|Action|Outcome|
|---|---|---|
|Cloud Authentication|Signed in to a Microsoft Entra-integrated service using the synchronised work or school account `amorgan@simonbaron.co.uk`|Successful sign-in using synchronised Entra work or school credentials|
|New User Provisioning|New user Casey Reed was created in the `CloudSync-Pilot` OU in the on-premises AD.|New AD user successfully provisioned into Microsoft Entra ID|
|Attribute Synchronisation|The Job Title and Department attributes of `jtaylor@simonbaron.co.uk` were updated in Active Directory.|Updated Job Title and Department values reflected in Entra ID|
|Password Hash Synchronisation|The password for `amorgan@simonbaron.co.uk` was changed in AD.|Successful cloud sign-in using the updated AD password|
|Group Membership Updates|`creed@simonbaron.co.uk` was added to the `CloudSync-Operations` security group|Updated security group membership reflected in Entra ID|
|Account Disablement|The account `scarter@simonbaron.co.uk` was disabled in AD|Synchronised account disabled for new Entra sign-ins|

## Security and Operational Considerations
### Identity Security
The implementation was designed to limit the impact of synchronisation on the existing infrastructure.

Key considerations included:
- Restricting initial synchronisation to a dedicated pilot OU.
- Excluding privileged AD accounts from the pilot.
- Retaining a cloud-only Entra administrative account.
- Using a group Managed Service Account for the provisioning agent.
- Maintaining secure administrative access to `MGMT01`.
- Reviewing accidental deletion protection.
- Keeping on-premises `AD` as the source of authority for synchronised objects.

### Cost Management
Microsoft Entra ID Free and Microsoft Entra Cloud Sync were used to implement the basic hybrid identity synchronisation.

The provisioning agent was installed on existing on-premises infrastructure, avoiding the need to deploy another Hyper-V VM or additional Azure compute resources.

## Phase 2 Summary
Phase 2 established hybrid identity synchronisation between the existing on-premises Active Directory environment and Microsoft Entra ID.

- **Hybrid Identity:** Verified the `simonbaron.co.uk` domain and configured an alternative Active Directory UPN suffix for cloud sign-in.
- **Cloud Sync Deployment:** Installed and registered the Microsoft Entra provisioning agent on MGMT01, using a group Managed Service Account.
- **Identity Synchronisation:** Successfully synchronised three pilot users and two security groups from the `CloudSync-Pilot` OU.
- **Identity Lifecycle:** Validated user provisioning, attribute changes, password hash synchronisation, group membership updates and account disablement.
- **Security:** Restricted synchronisation scope, excluded privileged AD identities and enabled accidental-deletion protection.

## Next Steps
**Phase 3: Hybrid Management** will focus on integrating the existing on-premises Windows Server environment with Azure Arc.

The next stage will include onboarding selected servers, including `FS01` and `MGMT01`, to Azure Arc to provide centralised inventory, management and visibility through Microsoft Azure.

The Cloud Sync configuration established in Phase 2 will remain available for further hybrid identity testing and ongoing operational monitoring.
