# Lab 1 — Active Directory Domain Services

Windows Server 2025 · Azure · Identity & Access Management

Deploys a single-domain Active Directory forest (`lab.local`) on an Azure VM — domain controller promotion, OU structure, security groups, user accounts, and a Group Policy enforcing password/lockout rules.

## What This Builds

- 1x Azure VM (`Standard_B2s`, Windows Server 2025 Datacenter) promoted to a Domain Controller
- Forest/domain: `lab.local`
- 5 OUs: `IT`, `Finance`, `HR`, `Sales`, `Workstations`
- 4 security groups (Global scope): `IT_Admins`, `Finance_Users`, `HR_Users`, `Sales_Users`
- 4 test users, one per department, added to their matching group
- 1 GPO (`IT Security Policy`) linked to the `IT` OU — 12-char password minimum, complexity required, 15-min inactivity lock, USB storage blocked

## Prerequisites

- Azure subscription (free tier is sufficient)
- RDP client with clipboard redirection enabled

## Deployment Steps

### 1. Provision the VM

Azure Portal → Virtual Machines → Create:

| Setting | Value |
|---|---|
| Region | East US |
| Image | Windows Server 2025 Datacenter — Gen2 |
| Size | Standard_B2s |
| Inbound ports | RDP (3389) |

### 2. Install AD DS role

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
Install-WindowsFeature -Name GPMC
```

### 3. Promote to Domain Controller

```powershell
Import-Module ADDSDeployment
Install-ADDSForest -DomainName 'lab.local' -DomainNetBiosName 'LAB' -InstallDns `
  -SafeModeAdministratorPassword (ConvertTo-SecureString 'YourDSRM_Password!' -AsPlainText -Force) -Force
```

Verify after reboot:

```powershell
Get-ADDomainController
```

### 4. Create OUs

> ⚠️ Don't name an OU `Computers` — AD already has a built-in `CN=Computers` container at that path and will reject it. Use `Workstations` instead.

```powershell
New-ADOrganizationalUnit -Name 'IT'           -Path 'DC=lab,DC=local'
New-ADOrganizationalUnit -Name 'Finance'      -Path 'DC=lab,DC=local'
New-ADOrganizationalUnit -Name 'HR'           -Path 'DC=lab,DC=local'
New-ADOrganizationalUnit -Name 'Sales'        -Path 'DC=lab,DC=local'
New-ADOrganizationalUnit -Name 'Workstations' -Path 'DC=lab,DC=local'
```

Verify:

```powershell
Get-ADOrganizationalUnit -Filter * | Select-Object Name
```

### 5. Create security groups

```powershell
New-ADGroup -Name 'IT_Admins'     -GroupScope Global -GroupCategory Security -Path 'OU=IT,DC=lab,DC=local'
New-ADGroup -Name 'Finance_Users' -GroupScope Global -GroupCategory Security -Path 'OU=Finance,DC=lab,DC=local'
New-ADGroup -Name 'HR_Users'      -GroupScope Global -GroupCategory Security -Path 'OU=HR,DC=lab,DC=local'
New-ADGroup -Name 'Sales_Users'   -GroupScope Global -GroupCategory Security -Path 'OU=Sales,DC=lab,DC=local'
```

Verify before continuing — a missed group here causes confusing errors three steps later:

```powershell
Get-ADGroup -Filter * | Select-Object Name
```

### 6. Create users + assign group membership

> Run this as **one block**, not line by line — `$password` must be defined before `New-ADUser` executes, or PowerShell will prompt for input mid-script and fail.

```powershell
$password = ConvertTo-SecureString 'Welcome@2026!' -AsPlainText -Force

New-ADUser -Name 'alice.chen' -GivenName 'Alice' -Surname 'Chen' `
  -SamAccountName 'alice.chen' -UserPrincipalName 'alice.chen@lab.local' `
  -Path 'OU=IT,DC=lab,DC=local' -AccountPassword $password -Enabled $true

New-ADUser -Name 'bob.patel' -GivenName 'Bob' -Surname 'Patel' `
  -SamAccountName 'bob.patel' -UserPrincipalName 'bob.patel@lab.local' `
  -Path 'OU=Finance,DC=lab,DC=local' -AccountPassword $password -Enabled $true

New-ADUser -Name 'carol.jones' -GivenName 'Carol' -Surname 'Jones' `
  -SamAccountName 'carol.jones' -UserPrincipalName 'carol.jones@lab.local' `
  -Path 'OU=HR,DC=lab,DC=local' -AccountPassword $password -Enabled $true

New-ADUser -Name 'david.smith' -GivenName 'David' -Surname 'Smith' `
  -SamAccountName 'david.smith' -UserPrincipalName 'david.smith@lab.local' `
  -Path 'OU=Sales,DC=lab,DC=local' -AccountPassword $password -Enabled $true

Add-ADGroupMember -Identity 'IT_Admins'     -Members 'alice.chen'
Add-ADGroupMember -Identity 'Finance_Users' -Members 'bob.patel'
Add-ADGroupMember -Identity 'HR_Users'      -Members 'carol.jones'
Add-ADGroupMember -Identity 'Sales_Users'   -Members 'david.smith'
```

Verify membership:

```powershell
Get-ADGroupMember -Identity 'IT_Admins'
Get-ADGroupMember -Identity 'Finance_Users'
Get-ADGroupMember -Identity 'HR_Users'
Get-ADGroupMember -Identity 'Sales_Users'
```

### 7. Group Policy

Group Policy Management → `IT` OU → Create a GPO → name it `IT Security Policy` → Edit, then configure:

| Setting | Value |
|---|---|
| Minimum password length | 12 |
| Password complexity | Enabled |
| Inactivity lockout | 900 seconds |
| Removable storage access | Deny all |

## Verification Checklist

```powershell
Get-ADDomainController                                   # Forest: lab.local
Get-ADOrganizationalUnit -Filter *                        # 5 OUs
Get-ADGroup -Filter *                                      # 4 groups
Get-ADUser -Filter {Enabled -eq $true}                     # 4 users
Get-ADGroupMember -Identity IT_Admins                       # alice.chen
Get-GPInheritance -Target 'OU=IT,DC=lab,DC=local'           # IT Security Policy linked
Get-ADDomain                                                 # DomainMode, Forest, PDCEmulator
```

## Common Help Desk Tasks (Practiced in This Lab)

```powershell
# Password reset
Set-ADAccountPassword -Identity 'bob.patel' -Reset `
  -NewPassword (ConvertTo-SecureString 'TempPass@2026!' -AsPlainText -Force)
Set-ADUser -Identity 'bob.patel' -ChangePasswordAtLogon $true

# Unlock account
Unlock-ADAccount -Identity 'carol.jones'

# Disable account (offboarding)
Disable-ADAccount -Identity 'david.smith'

# Stale account audit (90+ days inactive)
$cutoff = (Get-Date).AddDays(-90)
Get-ADUser -Filter {LastLogonDate -lt $cutoff -and Enabled -eq $true} -Properties LastLogonDate
```

## Errors Hit During This Deployment

| Error | Cause | Fix |
|---|---|---|
| `New-ADOrganizationalUnit` — name already in use | OU named `Computers` collides with AD's built-in `CN=Computers` container | Renamed OU to `Workstations` |
| `Add-ADGroupMember` — cannot find object with identity | Referenced group was never created (silent upstream failure) | Added `Get-ADGroup -Filter *` checkpoint before assigning membership |
| `New-ADGroup` — group already exists | Re-ran group creation after groups were already successfully created in a prior pass | AD operations aren't idempotent — verify with `Get-ADGroup` before re-running, skip creation if already present |

## Stack

`Windows Server 2025` `Active Directory Domain Services` `PowerShell` `Azure` `Group Policy`

---

Part of a multi-lab IT/cloud portfolio series. Next: **Lab 2 — Azure RBAC** (NTFS file shares + permissions on top of this domain).

`github.com/bryandenard0`
