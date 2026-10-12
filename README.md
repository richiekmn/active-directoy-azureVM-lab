Watch me explain this lab: [LoomLink](https://www.loom.com/share/d167aae4f93d43ec8af55d7a8dc2eb94)

# active-directory-azureVM-lab

Built a Windows Server Active Directory domain on Azure: domain controller, OUs, security groups, users, Group Policy, and help desk account lifecycle tasks, all driven from PowerShell.

# Lab 1: Active Directory on Azure

**Windows Server 2025 · Azure VM · Identity & Access Management**

I built a working Active Directory environment from scratch on an Azure VM: installed the AD DS role, promoted the server to a domain controller, designed the org structure, provisioned users and groups, enforced security settings with Group Policy, and practiced the account lifecycle tasks a help desk handles every day. Below is an overview of what I built, the key commands, and the problems I worked through.

My domain: **`lab.local`**.

| | |
|---|---|
| **Platform** | Azure VM, Windows Server 2025 Datacenter|
| **Time** | 3 to 5 hours across multiple sessions |

---

## What Active Directory does

Active Directory answers one question for an entire organisation: *who is allowed to do what?* It is the identity backbone that controls who can log into which machines, who can reach which file shares, and which policies apply to which people. When someone joins, IT creates their account and adds them to the right groups, and access to email, drives, printers, and apps follow from group membership. When they leave, disabling one account closes every door at once. These same concepts (users, groups, roles, policy) carry directly into Microsoft Entra ID (Azure Active Directory) for hybrid environments, making it foundational for IT support, sysadmin, and cloud related roles.

---

## What I built

### 1. Azure VM + remote access
Created a Windows Server 2025 Datacenter VM (Standard D2als v7, East US, Standard SSD, RDP open on 3389) and connected to it on the **Windows App** remote desktop client using the downloaded `.rdp` file. The browser console has very limited clipboard support, so enabling "clipboard" in settings under redirection is what makes pasting commands possible.

### 2. Installed the AD DS and GPMC roles
```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
Install-WindowsFeature -Name GPMC
```
Installing GPMC up front means **Group Policy Management** is already in the Server Manager Tools menu when it's needed later.

### 3. Promoted the server to a Domain Controller
Installing the AD DS role does not create a domain, **promotion** does. I created a new forest, and the rebooted the server when it finished:
```powershell
Import-Module ADDSDeployment
Install-ADDSForest -DomainName 'lab.local' -DomainNetBiosName 'LAB' -InstallDns:$true `
  -SafeModeAdministratorPassword (Read-Host -AsSecureString "DSRM password") -Force:$true
```
Confirmed with `Get-ADDomainController`: forest `lab.local`, DNS installed, server acting as the root domain controller.

### 4. Built the organisational structure
Created an OU for each department, then a global security group per department for role-based access:
```powershell
# Creates all 4 OUs at once
New-ADOrganizationalUnit -Name "IT"        -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Finance"   -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "HR"        -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Sales"     -Path "DC=lab,DC=local"

# Creates all 4 security groups
New-ADGroup -Name "IT_Admins"     -GroupScope Global -GroupCategory Security -Path "OU=IT,DC=lab,DC=local"
New-ADGroup -Name "Finance_Users" -GroupScope Global -GroupCategory Security -Path "OU=Finance,DC=lab,DC=local"
New-ADGroup -Name "HR_Users"      -GroupScope Global -GroupCategory Security -Path "OU=HR,DC=lab,DC=local"
New-ADGroup -Name "Sales_Users"   -GroupScope Global -GroupCategory Security -Path "OU=Sales,DC=lab,DC=local"
```
<img width="565" height="412" alt="ADLabOUs" src="https://github.com/user-attachments/assets/36fac78f-5156-4bc9-9710-196bba1fdca8" />


### 5. Provisioned users and group memberships
One user per department using the `firstname.lastname` convention, each added to their department group (run as a single block so `$password` is defined first):
```powershell
# Step 1 - defines the password variable
$password = ConvertTo-SecureString "Welcome@2026!" -AsPlainText -Force

# Step 2 - creates all 4 users
New-ADUser -Name "alice.chen" -GivenName "Alice" -Surname "Chen" `
  -SamAccountName "alice.chen" -UserPrincipalName "alice.chen@lab.local" `
  -Path "OU=IT,DC=lab,DC=local" -AccountPassword $password -Enabled $true

New-ADUser -Name "bob.patel" -GivenName "Bob" -Surname "Patel" `
  -SamAccountName "bob.patel" -UserPrincipalName "bob.patel@lab.local" `
  -Path "OU=Finance,DC=lab,DC=local" -AccountPassword $password -Enabled $true
 
New-ADUser -Name "carol.jones" -GivenName "Carol" -Surname "Jones" `
  -SamAccountName "carol.jones" -UserPrincipalName "carol.jones@lab.local" `
  -Path "OU=HR,DC=lab,DC=local" -AccountPassword $password -Enabled $true
 
New-ADUser -Name "david.smith" -GivenName "David" -Surname "Smith" `
  -SamAccountName "david.smith" -UserPrincipalName "david.smith@lab.local" `
  -Path "OU=Sales,DC=lab,DC=local" -AccountPassword $password -Enabled $true

# Step 3 - add each user to their department group
Add-ADGroupMember -Identity "IT_Admins"     -Members "alice.chen"
Add-ADGroupMember -Identity "Finance_Users" -Members "bob.patel"
Add-ADGroupMember -Identity "HR_Users"      -Members "carol.jones"
Add-ADGroupMember -Identity "Sales_Users"   -Members "david.smith"
```
> `Welcome@2026!` is a throwaway lab password. Never hardcode real credentials in scripts or commit them to a repo.

<img width="740" height="412" alt="ADLabProvisionUsersGPOs" src="https://github.com/user-attachments/assets/67dc60d8-c3f3-4720-8ea9-71a7fd30a632" />


### 6. Enforced security with Group Policy
Created an **IT Security Policy** GPO, linked it to the IT OU, and configured:

| Setting | Value | Why |
|---|---|---|
| Minimum password length | 12 | Stronger passwords |
| Password complexity | Enabled | Upper, lower, number, symbol |
| Interactive logon: Machine inactivity limit | 900 seconds (15 min) | Auto-lock idle screens |
| Removable storage: All classes, deny all access | Enabled | Blocks USB data exfiltration |

Verified the link with:
```powershell
Get-GPInheritance -Target "OU=IT,DC=lab,DC=local"
```
<img width="651" height="820" alt="ADLabLinkConfigureGPO" src="https://github.com/user-attachments/assets/dba28174-6469-404c-abe6-cdd5a3c877e4" />


### 7. Ran help desk account lifecycle tasks
The day one ticket work, all scriptable:
```powershell
# Reset a password and force change at next login
Set-ADAccountPassword -Identity "bob.patel" -Reset -NewPassword (ConvertTo-SecureString "NewPass@2026!" -AsPlainText -Force)
Set-ADUser -Identity "bob.patel" -ChangePasswordAtLogon $true

Unlock-ADAccount  -Identity "carol.jones"    # unlock a locked account
Disable-ADAccount -Identity "david.smith"    # offboard (disable, don't delete to preserve account history in case it is needed for audits)

# Audits and reporting
Search-ADAccount -AccountDisabled | Select-Object Name, SamAccountName # finds all currently disabled accounts
Get-ADPrincipalGroupMembership -Identity "alice.chen" | Select-Object Name # check group membership for a specific user

# Find accounts that have not logged in for 90 days (will spit out nothing as this is a brand new lab environment)
Search-ADAccount -AccountInactive -TimeSpan 90.00:00:00 -UsersOnly |
  Where-Object Enabled | Select-Object Name, LastLogonDate

```
Offboarding means **disable**, not delete: it preserves the account's history and group memberships for audit purposes, while deletion is permanent.

---

## Problems and fixes

| What broke | Cause | Fix |
|---|---|---|
| Can't paste commands into the VM | RDP clipboard sharing is off by default | On Windows App: Settings → Redirections → enable Clipboard |
| `New-ADUser` prompts for `Name:` | `$password` wasn't defined before the commands ran | Run the whole block at once, with the `$password` line first |
| Group Policy Management missing from Tools menu | GPMC not installed | `Install-WindowsFeature -Name GPMC`, then reopen Server Manager |
| Promotion fails with a DNS conflict | NIC pointing at the wrong DNS server | Set preferred DNS to `127.0.0.1` (or the VM's static IP) before promoting |
| Can't RDP after domain join | Logging in with the local account | Use `LAB\Administrator` (domain admin) |
| GPO not applying | Policy hasn't refreshed or isn't scoped to the target | `gpupdate /force`, then `gpresult /r` on the target machine |
| User can't log in after creation | Account disabled or password-change flag | Confirm `Enabled` is true and check `ChangePasswordAtLogon` |

---

## Key takeaways

1. **Install is not promote.** Installing the AD DS role doesn't create a domain. You promote the server, which creates the forest.
2. **Every path must match the real domain.** OU, group, and user paths are distinguished names, and one wrong `DC=` component breaks the command. `(Get-ADDomain).DistinguishedName` gives the exact string.
3. **Group Policy links to OUs, not groups.** The OU structure is what makes GPOs deliverable, so design it around how you want to apply policy.
4. **Access is group-based.** Grant access to groups, then put people in groups. It scales from 4 users to 4,000 and makes offboarding a single action.
5. **Idempotent scripts are safer.** A script that checks whether an object exists before creating it can be run twice without errors.

If I rebuilt this, I'd store the domain DN in a variable up front and reference it everywhere:
```powershell
$dn = (Get-ADDomain).DistinguishedName
New-ADOrganizationalUnit -Name "IT" -Path $dn
```

---

## Verification

| Check | Command | Expected result |
|---|---|---|
| Domain controller running | `Get-ADDomainController` | DC info with forest `lab.local` |
| OUs exist | `Get-ADOrganizationalUnit -Filter *` | All 4 OUs |
| Users exist and enabled | `Get-ADUser -Filter {Enabled -eq $true}` | The 4 test accounts |
| Group membership correct | `Get-ADGroupMember -Identity IT_Admins` | `alice.chen` |
| GPO linked | `Get-GPInheritance -Target 'OU=IT,DC=lab,DC=local'` | `IT Security Policy` listed as linked |

---

## Next steps

- Join a second VM to `lab.local`, move its computer account into the IT OU, run `gpupdate /force`, and confirm the screen lock policy applies when logging in as `alice.chen`

---

**Status: complete.** Azure VM → domain controller → OUs → groups → users → Group Policy → account lifecycle.
