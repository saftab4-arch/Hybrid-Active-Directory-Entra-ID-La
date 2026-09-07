# Hybrid Active Directory & Microsoft Entra ID Lab

## Overview

This project documents the deployment of a hybrid Microsoft identity environment combining traditional on-premises-style Active Directory Domain Services with Microsoft Entra ID.

The environment was hosted in Microsoft Azure but designed to represent a small organization's traditional Windows infrastructure connected to Microsoft cloud identity services.

The project covered the complete identity lifecycle:

- Windows Server deployment
- Active Directory Domain Services
- DNS
- Organizational Units
- Users and security groups
- Domain-joined Windows client
- Group Policy
- Custom UPN suffix
- Public DNS configuration
- Microsoft Entra ID custom domain
- Microsoft Entra Connect Sync
- Password Hash Synchronization
- Hybrid user authentication
- AD Recycle Bin
- Windows Server Backup
- Active Directory System State backup
- Directory Services Restore Mode (DSRM)
- Active Directory disaster recovery

The purpose of the lab was not simply to install Active Directory, but to understand how traditional Windows infrastructure connects to modern Microsoft cloud identity.

---

# Architecture

```text
                        INTERNET
                            |
                            |
                     Cloudflare DNS
                            |
                 basitcloudlab.com
                            |
                            |
                 Microsoft Entra ID
                            |
                  Verified Custom Domain
                            |
                            |
                 Microsoft Entra Connect
                  Password Hash Sync
                            |
                            |
                    ----------------
                    |              |
                    |    DC01      |
                    |              |
                    | AD DS        |
                    | DNS          |
                    | GPO          |
                    | Entra Connect|
                    |              |
                    ----------------
                           |
                    ad.basitcloudlab.com
                           |
              ---------------------------
              |                         |
              |                         |
       Windows Client                FS01
       Domain Joined            File Server Design
              |
              |
          Domain Users
```

The Azure environment provided the compute and networking infrastructure, while the Windows servers were configured similarly to machines running inside a traditional company network.

---

# Technology Used

## Microsoft Azure

- Azure Virtual Machines
- Azure Virtual Network
- Subnets
- Network Interfaces
- Private IP addressing
- Azure disks
- Remote Desktop

## Windows Infrastructure

- Windows Server
- Active Directory Domain Services
- DNS Server
- Group Policy Management
- Active Directory Administrative Center
- Windows Server Backup
- PowerShell

## Hybrid Identity

- Microsoft Entra ID
- Microsoft Entra Connect Sync
- Password Hash Synchronization
- Custom UPN suffix
- Microsoft Entra custom domain

## DNS

- Active Directory-integrated DNS
- Cloudflare public DNS
- Microsoft domain verification records

---

# Environment Design

The Active Directory DNS domain used for the lab was:

```text
ad.basitcloudlab.com
```

The publicly owned domain was:

```text
basitcloudlab.com
```

This distinction became important when Microsoft Entra ID was introduced.

The internal AD forest remained:

```text
ad.basitcloudlab.com
```

while users received a routable UPN:

```text
username@basitcloudlab.com
```

This allowed the same username format to be used for both local Active Directory and Microsoft cloud authentication.

Example:

```text
On-premises AD:

umer@basitcloudlab.com

        ↓ Entra Connect

Microsoft Entra ID:

umer@basitcloudlab.com
```

---

# Phase 1 — Deploy Windows Server

The first virtual machine was deployed as the future Domain Controller.

Server name:

```text
DC01
```

The server was assigned a stable private IP because DNS clients and domain members need a consistent address for the Domain Controller.

After connecting through Remote Desktop, the computer was renamed.

Example PowerShell:

```powershell
Rename-Computer -NewName "DC01" -Restart
```

After rebooting:

```powershell
hostname
```

was used to verify the server name.

---

# Phase 2 — Install Active Directory Domain Services

AD DS was installed through:

```text
Server Manager
→ Manage
→ Add Roles and Features
```

Selected:

```text
Active Directory Domain Services
```

DNS Server was also installed as part of the Domain Controller configuration.

The equivalent PowerShell installation can be performed with:

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

Verify:

```powershell
Get-WindowsFeature AD-Domain-Services
```

---

# Phase 3 — Create the Active Directory Forest

After AD DS installation:

```text
Server Manager
→ Notifications
→ Promote this server to a domain controller
```

Selected:

```text
Add a new forest
```

Root domain:

```text
ad.basitcloudlab.com
```

A Directory Services Restore Mode password was configured during promotion.

This password is separate from normal domain credentials and is required when recovering Active Directory while AD DS is offline.

After promotion, DC01 rebooted and became the Domain Controller for:

```text
ad.basitcloudlab.com
```

---

# Phase 4 — Verify Active Directory

PowerShell was used to inspect the new forest.

```powershell
Get-ADDomain
```

```powershell
Get-ADForest
```

Domain Controller discovery:

```powershell
Get-ADDomainController
```

DNS could also be tested using:

```powershell
nslookup ad.basitcloudlab.com
```

---

# Phase 5 — Organizational Unit Design

Rather than leaving every object in the default Users and Computers containers, an organizational structure was created.

Main company OU:

```text
Iqras-Cafe
```

Structure:

```text
Iqras-Cafe
│
├── Computers
│
├── Groups
│
├── Servers
│
├── Service-Accounts
│
└── Users
     │
     ├── Finance
     ├── HR
     ├── Management
     └── Staff
```

This provides logical separation for:

- users
- computers
- servers
- service accounts
- departments
- Group Policy targeting

GUI:

```text
Active Directory Users and Computers
→ Domain
→ New
→ Organizational Unit
```

PowerShell equivalent:

```powershell
New-ADOrganizationalUnit `
    -Name "Iqras-Cafe" `
    -Path "DC=ad,DC=basitcloudlab,DC=com"
```

Example nested OU:

```powershell
New-ADOrganizationalUnit `
    -Name "Finance" `
    -Path "OU=Users,OU=Iqras-Cafe,DC=ad,DC=basitcloudlab,DC=com"
```

---

# Phase 6 — Create Active Directory Users

Multiple users were created to represent employees.

Examples included:

```text
Umer
Hamdan
Ali
Usman
Test User
```

Users were placed into their appropriate departmental OUs.

Example PowerShell:

```powershell
$password = ConvertTo-SecureString "TemporaryPassword" -AsPlainText -Force

New-ADUser `
    -Name "Test User" `
    -SamAccountName "testuser" `
    -UserPrincipalName "TestUser@basitcloudlab.com" `
    -Path "OU=Finance,OU=Users,OU=Iqras-Cafe,DC=ad,DC=basitcloudlab,DC=com" `
    -AccountPassword $password `
    -Enabled $true
```

Verify:

```powershell
Get-ADUser testuser
```

List users:

```powershell
Get-ADUser -Filter *
```

---

# Phase 7 — Security Groups

Security groups were used to manage access rather than assigning permissions directly to individual users.

Example:

```text
Finance-Users
```

PowerShell example:

```powershell
New-ADGroup `
    -Name "Finance-Users" `
    -GroupScope Global `
    -GroupCategory Security `
    -Path "OU=Groups,OU=Iqras-Cafe,DC=ad,DC=basitcloudlab,DC=com"
```

Add a user:

```powershell
Add-ADGroupMember `
    -Identity "Finance-Users" `
    -Members "umer"
```

Verify:

```powershell
Get-ADGroupMember "Finance-Users"
```

This demonstrated an important enterprise pattern:

```text
User
 ↓
Security Group
 ↓
Permission / Policy
```

instead of:

```text
User
 ↓
Direct Permission
```

---

# Phase 8 — Windows Client Domain Join

A Windows client machine was deployed separately.

The client's DNS configuration was changed so that its primary DNS server pointed to DC01.

This is critical.

A domain client should query the Active Directory DNS server so it can locate services such as:

```text
LDAP
Kerberos
Domain Controllers
Global Catalog
```

The client was then joined to:

```text
ad.basitcloudlab.com
```

GUI:

```text
System
→ About
→ Advanced system settings
→ Computer Name
→ Change
→ Domain
```

Domain:

```text
ad.basitcloudlab.com
```

Domain administrator credentials were supplied and the computer was restarted.

PowerShell equivalent:

```powershell
Add-Computer `
    -DomainName "ad.basitcloudlab.com" `
    -Credential "BASITCLOUDLAB\Administrator" `
    -Restart
```

After rebooting, domain users could authenticate against DC01.

---

# Phase 9 — Verify Domain Authentication

A domain user was used to log into the client.

Useful commands:

```cmd
whoami
```

Example result:

```text
basitcloudlab\umer
```

Verify the Domain Controller:

```cmd
echo %LOGONSERVER%
```

Example:

```text
\\DC01
```

Additional verification:

```cmd
nltest /dsgetdc:ad.basitcloudlab.com
```

```cmd
nltest /sc_verify:ad.basitcloudlab.com
```

DNS:

```cmd
nslookup ad.basitcloudlab.com
```

---

# Phase 10 — Group Policy

Group Policy Management was opened on DC01:

```text
Server Manager
→ Tools
→ Group Policy Management
```

A GPO was created for Finance users:

```text
Finance User Security Policy
```

The policy was linked to:

```text
Iqras-Cafe
→ Users
→ Finance
```

A user restriction was configured to demonstrate centralized policy enforcement.

After updating Group Policy on the Windows client:

```cmd
gpupdate /force
```

Policy results were inspected:

```cmd
gpresult /r
```

The result showed:

```text
Applied Group Policy Objects
----------------------------
Finance User Security Policy
```

The logged-in user was also confirmed as a member of:

```text
Finance-Users
```

The configured restriction successfully generated:

```text
This operation has been cancelled due to restrictions
in effect on this computer.
Please contact your system administrator.
```

This proved:

```text
Domain Controller
      ↓
Group Policy
      ↓
Finance OU
      ↓
Umer
      ↓
Windows Client
      ↓
Restriction enforced
```

---

# Phase 11 — File Server Design

The architecture also included a planned file server:

```text
FS01
```

The purpose of FS01 was to demonstrate:

- Windows file shares
- NTFS permissions
- Security-group-based access
- Department shares
- centralized storage

Example planned architecture:

```text
Finance User
     |
     ↓
Finance-Users
     |
     ↓
\\FS01\Finance
     |
     ↓
Share Permissions
+
NTFS Permissions
```

Due to limited Azure vCPU quota available to the lab subscription, FS01 could not be kept running alongside all required lab systems.

Therefore, the file-server role and permission architecture were designed/configured as part of the lab, but the complete FS01 workload could not be demonstrated in operation.

This limitation is documented rather than claiming functionality that was not tested.

---

# Phase 12 — Why the AD Domain Could Not Be Used Directly in Entra

The Active Directory forest was:

```text
ad.basitcloudlab.com
```

However, the public domain owned for the project was:

```text
basitcloudlab.com
```

Microsoft Entra ID requires a domain to be verified before it can be used as a cloud identity suffix.

Therefore:

```text
ad.basitcloudlab.com
```

was the internal AD DNS namespace, while:

```text
basitcloudlab.com
```

was used as the user sign-in suffix.

---

# Phase 13 — Configure Alternate UPN Suffix

On DC01:

```text
Active Directory Domains and Trusts
→ Right-click Active Directory Domains and Trusts
→ Properties
```

Added:

```text
basitcloudlab.com
```

as an alternate UPN suffix.

Users could then be changed from:

```text
umer@ad.basitcloudlab.com
```

to:

```text
umer@basitcloudlab.com
```

without renaming the actual AD forest.

This is an important distinction:

```text
AD forest/domain:
ad.basitcloudlab.com

User login:
umer@basitcloudlab.com
```

The internal AD namespace remained unchanged.

---

# Phase 14 — Cloudflare DNS

The public domain:

```text
basitcloudlab.com
```

was managed through Cloudflare.

Microsoft supplied a DNS verification record for proving ownership of the domain.

The record was added in:

```text
Cloudflare
→ DNS
→ Records
```

Microsoft Entra then checked public DNS and verified ownership.

Important distinction:

```text
Active Directory DNS
        ↓
Internal domain/service discovery

Cloudflare DNS
        ↓
Public Internet DNS/domain verification
```

These are two different DNS systems serving different purposes.

---

# Phase 15 — Add Custom Domain to Microsoft Entra ID

Inside the Microsoft Entra admin center:

```text
Microsoft Entra ID
→ Custom domain names
→ Add custom domain
```

Domain:

```text
basitcloudlab.com
```

After the Cloudflare verification record propagated, Microsoft Entra marked:

```text
basitcloudlab.com
```

as:

```text
Verified
```

This allowed synchronized users to use identities such as:

```text
umer@basitcloudlab.com
```

---

# Phase 16 — Microsoft Entra Connect

Microsoft Entra Connect Sync was installed to integrate on-premises Active Directory with Microsoft Entra ID.

The design became:

```text
On-Premises AD
ad.basitcloudlab.com
        |
        |
        | Microsoft Entra Connect
        |
        ↓
Microsoft Entra ID
basitcloudlab.com
```

---

# Phase 17 — Entra Connect Configuration

During Entra Connect configuration, the on-premises forest was connected.

Directory:

```text
ad.basitcloudlab.com
```

The configuration wizard detected UPN suffixes including:

```text
ad.basitcloudlab.com
basitcloudlab.com
```

The first showed as not added to Entra while:

```text
basitcloudlab.com
```

showed:

```text
Verified
```

The Microsoft Entra username attribute remained:

```text
userPrincipalName
```

This meant the on-premises UPN would become the cloud username.

---

# Phase 18 — Domain and OU Filtering

Entra Connect provides the ability to synchronize:

```text
All domains and OUs
```

or:

```text
Selected domains and OUs
```

For the lab, synchronization was configured broadly so the AD users could be observed in Microsoft Entra ID.

In a production environment, OU filtering can be used to limit which identities synchronize.

Example:

```text
AD
│
├── Service Accounts       X
├── Test Accounts          X
├── Legacy Users           X
└── Corporate Users        ✓
          |
          ↓
       Entra ID
```

---

# Phase 19 — User Identity Matching

Entra Connect was configured to uniquely identify users.

The default recommended source-anchor behavior was retained.

Microsoft Entra Connect used:

```text
mS-DS-ConsistencyGuid
```

as the source anchor.

The source anchor provides a stable relationship between:

```text
On-premises AD object
        ↕
Microsoft Entra object
```

even if certain user attributes later change.

---

# Phase 20 — Password Hash Synchronization

Password Hash Synchronization was enabled.

This does NOT mean Microsoft Entra simply receives the user's plaintext Active Directory password.

Conceptually:

```text
User password
      ↓
AD password hash
      ↓
Entra Connect processing
      ↓
derived hash
      ↓
Microsoft Entra ID
```

This allows users to authenticate to Microsoft cloud services using the same password experience as their on-premises account.

---

# Phase 21 — Entra Connect Scheduler

After configuration, the synchronization scheduler was inspected:

```powershell
Get-ADSyncScheduler
```

Important output:

```text
AllowedSyncCycleInterval             : 00:30:00
CurrentlyEffectiveSyncCycleInterval  : 00:30:00
SyncCycleEnabled                     : True
NextSyncCyclePolicyType              : Delta
```

This showed that the normal synchronization cycle was configured for approximately every 30 minutes.

A manual delta synchronization could be started with:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

Result:

```text
Success
```

A delta synchronization processes changes since the previous synchronization rather than performing a complete initial synchronization.

---

# Phase 22 — Verify Synced Users in Microsoft Entra ID

After synchronization, the users appeared in:

```text
Microsoft Entra Admin Center
→ Identity
→ Users
→ All users
```

Synchronized users showed:

```text
On-premises sync enabled: Yes
```

Their properties also exposed information such as:

```text
On-premises distinguished name
On-premises SAM account name
On-premises security identifier
On-premises user principal name
On-premises domain name
On-premises immutable ID
Last synchronization time
```

This proved that the users originated from Active Directory rather than being cloud-only identities.

---

# Phase 23 — Hybrid Authentication Test

Umer's synchronized account was tested against Microsoft Entra.

Cloud identity:

```text
umer@basitcloudlab.com
```

The same password used for the on-premises AD account successfully authenticated to Microsoft cloud services.

This demonstrated the complete authentication design:

```text
Umer
 |
 | On-premises
 ↓
Active Directory
umer@basitcloudlab.com
 |
 | Password Hash Sync
 ↓
Microsoft Entra ID
umer@basitcloudlab.com
 |
 ↓
Microsoft cloud authentication
```

It is important to note that synchronization does not automatically grant administrative privileges.

A synchronized normal user is still a normal user unless an Entra directory role is explicitly assigned.

---

# Phase 24 — Test New User Synchronization

A new user was created on-premises:

```text
Test User
```

The user did not initially exist in Microsoft Entra.

A delta sync was manually initiated:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

After synchronization, Test User appeared in Microsoft Entra with:

```text
On-premises sync: Yes
```

This proved:

```text
Create user in AD
        ↓
Entra Connect detects change
        ↓
Synchronization
        ↓
User created in Entra ID
```

---

# Phase 25 — Test User Deletion Synchronization

Test User was then deleted from on-premises Active Directory.

Another delta synchronization was initiated:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

The synchronized user disappeared from the active Entra user list.

This demonstrated that Entra Connect was synchronizing lifecycle changes and not merely copying users once.

---

# Phase 26 — Active Directory Recycle Bin

The Active Directory Recycle Bin was enabled through:

```text
Active Directory Administrative Center
→ ad (local)
→ Tasks
→ Enable Recycle Bin
```

Important:

Enabling AD Recycle Bin is effectively a one-way operation for the forest.

After enabling it, deleted Active Directory objects could be viewed under:

```text
Deleted Objects
```

Test User appeared there after deletion.

The user was selected and:

```text
Restore
```

was used.

The user returned to the original:

```text
Finance OU
```

This demonstrated object-level Active Directory recovery.

---

# Phase 27 — Windows Server Backup

The final portion of the project focused on Domain Controller disaster recovery.

Windows Server Backup was checked:

```powershell
Get-WindowsFeature Windows-Server-Backup
```

Initially it was:

```text
Available
```

It was installed:

```powershell
Install-WindowsFeature Windows-Server-Backup
```

Verification:

```powershell
Get-WindowsFeature Windows-Server-Backup
```

Result:

```text
[X] Windows Server Backup
```

---

# Phase 28 — Dedicated Backup Disk

A separate Azure managed disk was created and attached to DC01.

Disk:

```text
dc01-backup
```

Size:

```text
30 GiB
```

Inside Windows:

```text
Computer Management
→ Storage
→ Disk Management
```

The disk was:

1. Brought online
2. Initialized
3. New Simple Volume created
4. Formatted NTFS
5. Assigned drive letter E:
6. Labeled AD-Backup

PowerShell verification:

```powershell
Get-Volume
```

Result included:

```text
E    AD-Backup    NTFS    Fixed    Healthy
```

---

# Phase 29 — Active Directory System State Backup

The Domain Controller System State was backed up to:

```text
E:
```

Command:

```powershell
wbadmin start systemstatebackup -backuptarget:E: -quiet
```

Windows backed up the system state required for Domain Controller recovery.

The backup completed successfully:

```text
The backup operation successfully completed.
The backup of the system state successfully completed.
```

The backup included system components required for recovering the Domain Controller, including Active Directory-related state, SYSVOL, registry, and system configuration.

---

# Phase 30 — Verify the Backup

A backup should never be assumed valid simply because the backup command completed.

Available recovery versions were inspected:

```powershell
wbadmin get versions
```

The lab returned:

```text
Backup time: 9/7/2026 10:05 AM

Version identifier:
09/07/2026-10:05

Can recover:
Volume(s)
File(s)
Application(s)
System State
```

This confirmed Windows Server Backup recognized a System State recovery point.

---

# Phase 31 — Create a Post-Backup Marker

To prove that recovery actually returned Active Directory to the backup point, an additional user was created AFTER the backup.

```powershell
New-ADUser `
    -Name "After Backup User" `
    -SamAccountName "afterbackup" `
    -UserPrincipalName "afterbackup@basitcloudlab.com" `
    -Path "OU=Finance,OU=Users,OU=Iqras-Cafe,DC=ad,DC=basitcloudlab,DC=com" `
    -AccountPassword (ConvertTo-SecureString "TemporaryPassword" -AsPlainText -Force) `
    -Enabled $true
```

Verification:

```powershell
Get-ADUser afterbackup
```

Because this object did not exist when the backup was taken, it provided a simple way to identify the difference between the backed-up directory state and the later directory state.

---

# Phase 32 — Directory Services Restore Mode

The boot configuration was first inspected:

```powershell
bcdedit /enum
```

The active Windows boot loader was:

```text
{current}
```

The Domain Controller was configured to boot into Directory Services Restore Mode:

```powershell
bcdedit /set "{current}" safeboot dsrepair
```

PowerShell requires the `{current}` identifier to be quoted.

The server was restarted:

```powershell
Restart-Computer
```

---

# Phase 33 — DSRM Authentication

Normal domain authentication should not be relied upon during DSRM recovery because Active Directory Domain Services is offline.

The DSRM Administrator account was used:

```text
.\Administrator
```

with the Directory Services Restore Mode password configured for the Domain Controller.

This illustrates why the DSRM credential is important.

```text
Normal operation:

User
 ↓
AD DS
 ↓
Domain authentication


DSRM:

AD DS OFFLINE

.\Administrator
 ↓
Local DSRM authentication
```

---

# Phase 34 — Verify Backup From DSRM

Before beginning recovery, the backup disk was checked again from DSRM:

```powershell
wbadmin get versions -backuptarget:E:
```

Result:

```text
Backup time: 9/7/2026 10:05 AM
Backup target: Disk labeled E:

Version identifier:
09/07/2026-10:05

Can recover:
Volume(s), File(s), Application(s), System State
```

This was an important verification.

It proved that the recovery environment could actually access the backup before attempting restoration.

---

# Phase 35 — System State Recovery

The System State recovery was started using the verified backup version:

```powershell
wbadmin start systemstaterecovery `
    -version:09/07/2026-10:05 `
    -backuptarget:E:
```

The recovery confirmation prompt was accepted.

Conceptually:

```text
E:\ System State Backup
        |
        |
        ↓
DC01 running in DSRM
        |
        ├── Active Directory state
        ├── SYSVOL
        ├── Registry
        └── System configuration
```

This simulated the recovery procedure that could be required following serious Active Directory corruption or failure.

---

# Phase 36 — Return DC01 to Normal Boot

Because the server was explicitly configured for DSRM, the Safe Boot setting must be removed after recovery.

```powershell
bcdedit /deletevalue "{current}" safeboot
```

Then:

```powershell
Restart-Computer
```

DC01 can then return to normal Windows Server / Active Directory operation.

---

# Post-Recovery Verification Checklist

After recovery, the following should be validated.

## Domain Controller

```powershell
Get-ADDomainController
```

## AD DS

```powershell
Get-Service NTDS
```

## DNS

```powershell
Get-Service DNS
```

```cmd
nslookup ad.basitcloudlab.com
```

## Active Directory users

```powershell
Get-ADUser -Filter *
```

## Organizational Units

```powershell
Get-ADOrganizationalUnit -Filter *
```

## Security groups

```powershell
Get-ADGroup -Filter *
```

## Group Policy

On the client:

```cmd
gpupdate /force
```

```cmd
gpresult /r
```

## Domain Controller discovery

```cmd
nltest /dsgetdc:ad.basitcloudlab.com
```

## Secure channel

```cmd
nltest /sc_verify:ad.basitcloudlab.com
```

## Entra Connect

```powershell
Get-ADSyncScheduler
```

Then:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

Finally verify synchronized users in the Microsoft Entra admin center.

---

# Active Directory Recovery vs Recycle Bin

This project demonstrated two very different recovery mechanisms.

## AD Recycle Bin

Used for:

```text
Accidentally deleted user
Accidentally deleted group
Accidentally deleted OU
```

Example:

```text
Delete Test User
      ↓
Deleted Objects
      ↓
Restore
      ↓
User returns
```

## System State Recovery

Used for significantly larger directory/server recovery scenarios.

```text
Serious AD corruption/failure
        ↓
DSRM
        ↓
System State Backup
        ↓
Recover directory/system state
```

Understanding when to use each recovery mechanism is as important as knowing how to perform the recovery.

---

# Important PowerShell / CMD Command Reference

## Active Directory

```powershell
Get-ADDomain
Get-ADForest
Get-ADDomainController
Get-ADUser -Filter *
Get-ADGroup -Filter *
Get-ADOrganizationalUnit -Filter *
```

## Domain Client

```cmd
whoami
echo %LOGONSERVER%
nltest /dsgetdc:ad.basitcloudlab.com
nltest /sc_verify:ad.basitcloudlab.com
nslookup ad.basitcloudlab.com
```

## Group Policy

```cmd
gpupdate /force
gpresult /r
```

## Entra Connect

```powershell
Get-ADSyncScheduler
```

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

## Windows Server Backup

```powershell
Get-WindowsFeature Windows-Server-Backup
```

```powershell
Install-WindowsFeature Windows-Server-Backup
```

```powershell
Get-Volume
```

```powershell
wbadmin start systemstatebackup -backuptarget:E: -quiet
```

```powershell
wbadmin get versions
```

```powershell
wbadmin get versions -backuptarget:E:
```

## DSRM

```powershell
bcdedit /enum
```

```powershell
bcdedit /set "{current}" safeboot dsrepair
```

```powershell
Restart-Computer
```

## System State Recovery

```powershell
wbadmin start systemstaterecovery `
    -version:09/07/2026-10:05 `
    -backuptarget:E:
```

## Return to Normal Boot

```powershell
bcdedit /deletevalue "{current}" safeboot
```

```powershell
Restart-Computer
```

---

# Troubleshooting Lessons

## Domain Client Cannot Find Domain

Check DNS first.

```cmd
ipconfig /all
```

The client's DNS server should point to the Domain Controller rather than an arbitrary public DNS resolver.

Then:

```cmd
nslookup ad.basitcloudlab.com
```

---

## GPO Does Not Apply

Force an update:

```cmd
gpupdate /force
```

Then inspect:

```cmd
gpresult /r
```

Check:

- user OU
- computer OU
- GPO link
- security filtering
- group membership
- DNS/domain connectivity

---

## Entra User Does Not Appear

Check scheduler:

```powershell
Get-ADSyncScheduler
```

Trigger delta:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

Then inspect the Entra user list.

Also verify the user's UPN uses the verified domain:

```text
@basitcloudlab.com
```

---

## PowerShell BCDEdit Error

This failed in PowerShell:

```powershell
bcdedit /set {current} safeboot dsrepair
```

because PowerShell interpreted the braces.

Correct:

```powershell
bcdedit /set "{current}" safeboot dsrepair
```

---

## Domain Credentials Do Not Work in DSRM

DSRM does not use normal Active Directory authentication.

Use:

```text
.\Administrator
```

with the DSRM password.

---

# Key Concepts Learned

This project helped connect several technologies that are often learned separately.

### DNS is fundamental to Active Directory

Clients do not simply "know" where the Domain Controller is.

DNS allows them to discover AD services.

```text
Client
 ↓
DNS
 ↓
DC discovery
 ↓
Kerberos / LDAP
 ↓
Authentication
```

### OUs organize and target policy

```text
Domain
 ↓
Company OU
 ↓
Department OU
 ↓
Users / Computers
 ↓
GPO
```

### Groups should control access

```text
User
 ↓
Security Group
 ↓
Resource Permission
```

### Internal AD domain and user UPN do not have to match

```text
AD DNS domain:
ad.basitcloudlab.com

User UPN:
umer@basitcloudlab.com
```

### Entra Connect bridges identity systems

```text
Active Directory
      ↓
Entra Connect
      ↓
Microsoft Entra ID
```

### Password Hash Synchronization provides a common password experience

A user can authenticate to both environments using the synchronized identity/password experience without Microsoft Entra needing the plaintext AD password.

### Backups must be verified

Running a backup command is not sufficient.

The recovery point was explicitly checked using:

```powershell
wbadmin get versions
```

and checked again from DSRM before recovery.

### Recovery is part of infrastructure engineering

Building Active Directory is only half of the problem.

An administrator should also understand:

```text
How do I restore a deleted user?

How do I recover from directory corruption?

What happens if AD DS cannot start?

How do I authenticate when AD itself is offline?

Where is my recovery point?

Can the recovery environment actually see it?
```

---

# End-to-End Identity Flow

The completed environment can be summarized as:

```text
                     basitcloudlab.com
                            |
                      Cloudflare DNS
                            |
                  Microsoft verification
                            |
                            ↓
                    Microsoft Entra ID
                            ↑
                            |
                Password Hash Synchronization
                            |
                   Microsoft Entra Connect
                            |
                            ↑
                    Active Directory
                 ad.basitcloudlab.com
                            |
               +------------+------------+
               |            |            |
             DNS           GPO         Users
               |            |            |
               +------------+------------+
                            |
                      Windows Client
                            |
                      Domain User
                            |
                    umer@basitcloudlab.com
```

---

# Disaster Recovery Flow

```text
                  NORMAL OPERATION
                         |
                         ↓
                       DC01
                         |
        +----------------+----------------+
        |                |                |
       AD DS            DNS             GPO
        |
        ↓
System State Backup
        |
        ↓
     E:\AD-Backup
        |
        |
     FAILURE
        |
        ↓
Boot DC01 into DSRM
        |
        ↓
Verify backup
        |
        ↓
System State Recovery
        |
        ↓
Remove DSRM boot option
        |
        ↓
Normal reboot
        |
        ↓
Verify AD / DNS / GPO / Client / Entra
```

---

# Project Limitations

This was a personal hands-on lab rather than a production deployment.

The environment intentionally remained small to control Azure cost and resource usage.

A dedicated FS01 file server was part of the infrastructure design; however, limited Azure vCPU quota prevented the complete multi-server environment from running simultaneously. The file-server design and intended group-based permission model were therefore documented without claiming a fully validated FS01 production workflow.

The lab also used a single Domain Controller.

A real production Active Directory environment would normally include additional considerations such as:

- Multiple Domain Controllers
- AD replication
- Multiple Availability Zones/sites
- redundant DNS
- dedicated Entra Connect architecture
- privileged administrative accounts
- MFA
- Conditional Access
- monitoring
- centralized logging
- formal backup retention
- tested recovery procedures
- least-privilege administration
- network segmentation
- hardened administrative access

---

# Final Result

The project successfully connected traditional Windows infrastructure concepts with modern Microsoft cloud identity.

The lab covered:

```text
Azure Infrastructure
        ↓
Windows Server
        ↓
Active Directory Domain Services
        ↓
DNS
        ↓
OUs / Users / Groups
        ↓
Domain-Joined Client
        ↓
Group Policy
        ↓
Custom UPN
        ↓
Cloudflare Public DNS
        ↓
Microsoft Entra Custom Domain
        ↓
Microsoft Entra Connect
        ↓
Password Hash Synchronization
        ↓
Hybrid Authentication
        ↓
AD Recycle Bin
        ↓
System State Backup
        ↓
DSRM
        ↓
Active Directory Disaster Recovery
```

This project provided hands-on practice with both the **day-to-day administration** and the **recovery lifecycle** of a small hybrid Microsoft identity environment.
