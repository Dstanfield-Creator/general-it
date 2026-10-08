# User Onboarding and Offboarding Checklist

> Tickable joiner and leaver checklists for a Windows Active Directory and Microsoft 365 environment, with the PowerShell behind each step.

**Status:** Active · **Updated:** 2026-10-08

## How to use this file

Copy the relevant checklist into the ticket for each joiner or leaver and tick items as they are completed. Every step should trace back to an approved request. Placeholders: `newuser` and `leaver` are the accounts, `manager01` is their manager, `tpl-finance` is a role template account, and group names use `LIC-` and `SEC-` prefixes. Replace them with your own conventions.

```powershell
Import-Module ActiveDirectory
Connect-MgGraph -Scopes "User.ReadWrite.All","Directory.ReadWrite.All","Group.ReadWrite.All"
Connect-ExchangeOnline
```

## Onboarding (joiner)

### Request and approval

- [ ] Ticket raised by the hiring manager with start date, role, department, manager and location
- [ ] Approval recorded in the ticket (manager, plus data-owner approval where the role needs it)
- [ ] Role mapped to an access template: groups, licensing group, shared mailboxes, Teams

### Directory account

- [ ] Create the AD account from the role template, in the correct OU
- [ ] Set a one-time password and require a change at first logon
- [ ] Populate title, department, manager and office (these drive the address book and dynamic groups)
- [ ] Add the user to role groups only; never grant direct permissions on resources

```powershell
$Template = Get-ADUser -Identity "tpl-finance" -Properties Department, Title, Office, Company
New-ADUser -Name "New User" -SamAccountName "newuser" -UserPrincipalName "newuser@example.com" `
  -GivenName "New" -Surname "User" -Instance $Template `
  -Path "OU=Users,OU=Finance,DC=example,DC=com" `
  -AccountPassword (Read-Host -AsSecureString "Initial password") `
  -ChangePasswordAtLogon $true -Enabled $true

# Copy group membership from the template, excluding Domain Users
Get-ADPrincipalGroupMembership -Identity "tpl-finance" |
  Where-Object { $_.Name -ne "Domain Users" } |
  ForEach-Object { Add-ADGroupMember -Identity $_ -Members "newuser" }

Set-ADUser -Identity "newuser" -Manager "manager01" -Replace @{ title = "Analyst"; department = "Finance" }
```

### Mailbox and licence

- [ ] Confirm the account has synced to Entra ID (`Get-MgUser -UserId newuser@example.com`)
- [ ] Set the usage location, then assign the licence through a licensing group
- [ ] Verify the mailbox exists and the primary SMTP address is correct

```powershell
Update-MgUser -UserId "newuser@example.com" -UsageLocation "AU"
New-MgGroupMember -GroupId (Get-MgGroup -Filter "displayName eq 'LIC-M365-E3'").Id `
  -DirectoryObjectId (Get-MgUser -UserId "newuser@example.com").Id
Get-Mailbox -Identity "newuser@example.com" | Select-Object PrimarySmtpAddress, RecipientTypeDetails
```

### Identity protection

- [ ] Send MFA enrolment instructions, or issue a Temporary Access Pass for first sign-in
- [ ] Confirm the user is in scope for the baseline Conditional Access policies

### Device

- [ ] Assign a device in the asset register (serial, model, user, date)
- [ ] Enrol the device (Autopilot or manual Intune join); confirm compliance and disk encryption with an escrowed recovery key

### Collaboration

- [ ] Add to shared mailboxes required by the role (Full Access, plus Send As or Send on Behalf)
- [ ] Add to Teams, SharePoint sites and distribution lists via role groups, not directly

```powershell
Add-MailboxPermission -Identity "finance-inbox@example.com" -User "newuser@example.com" -AccessRights FullAccess -AutoMapping $true
Add-RecipientPermission -Identity "finance-inbox@example.com" -Trustee "newuser@example.com" -AccessRights SendAs -Confirm:$false
```

### Close out

- [ ] Send the welcome note with first-logon steps and the support contact
- [ ] Record an access review date in the ticket (for example 90 days after start), then close with the account name, groups and licence

## Offboarding (leaver)

Order matters. Cut off sign-in and sessions first; everything else can follow. In a hybrid setup, disabling the AD account reaches the cloud only at the next sync cycle, so block cloud sign-in directly as well.

### Immediate (at the agreed time)

- [ ] Disable the AD account and block sign-in in Entra ID
- [ ] Revoke all sessions and refresh tokens
- [ ] Reset the password to a random value (stops cached-credential use before sync)
- [ ] Remove registered MFA methods and any registered devices
- [ ] Note the time and the person who authorised the cut-off in the ticket

```powershell
Disable-ADAccount -Identity "leaver"
Set-ADAccountPassword -Identity "leaver" -Reset -NewPassword (ConvertTo-SecureString ([guid]::NewGuid().ToString()) -AsPlainText -Force)
Set-ADUser -Identity "leaver" -Replace @{ description = "Leaver 2026-10-08 ticket INC-1234" }

Update-MgUser -UserId "leaver@example.com" -AccountEnabled:$false
Revoke-MgUserSignInSession -UserId "leaver@example.com"
```

### Same day

- [ ] Export current group membership to the ticket, then remove the user from all groups except the leaver group
- [ ] Move the account to the Leavers OU
- [ ] Convert the mailbox to shared, or set forwarding to the manager with an end date recorded in the ticket
- [ ] Grant the manager delegated access and set an out-of-office reply naming the new contact
- [ ] Transfer OneDrive ownership to the manager; confirm the retention period (default 30 days after the account is deleted)
- [ ] Remove the user from third-party SaaS (password manager, CRM, source control, cloud consoles) and rotate shared secrets the user knew
- [ ] Retrieve company devices and update the asset register; issue an Intune wipe or retire for anything not returned in the agreed period

```powershell
$Groups = Get-ADPrincipalGroupMembership -Identity "leaver" | Where-Object { $_.Name -ne "Domain Users" }
$Groups | Select-Object Name | Export-Csv "C:\Offboarding\leaver-groups.csv" -NoTypeInformation
$Groups | ForEach-Object { Remove-ADGroupMember -Identity $_ -Members "leaver" -Confirm:$false }
Add-ADGroupMember -Identity "SEC-Leavers" -Members "leaver"
Move-ADObject -Identity (Get-ADUser "leaver").DistinguishedName -TargetPath "OU=Leavers,DC=example,DC=com"

# Shared mailbox (no licence needed under 50 GB) with manager access
Set-Mailbox -Identity "leaver@example.com" -Type Shared
Add-MailboxPermission -Identity "leaver@example.com" -User "manager01@example.com" -AccessRights FullAccess -AutoMapping $false
Set-MailboxAutoReplyConfiguration -Identity "leaver@example.com" -AutoReplyState Enabled `
  -InternalMessage "This mailbox is no longer monitored. Contact manager01@example.com." `
  -ExternalMessage "This mailbox is no longer monitored. Contact manager01@example.com."

# Alternative: forward, and diarise the removal date
Set-Mailbox -Identity "leaver@example.com" -ForwardingAddress "manager01@example.com" -DeliverToMailboxAndForward $true
```

### After the retention period

- [ ] Remove the licence once the mailbox shows as Shared and OneDrive has been transferred (`Remove-MgGroupMemberByRef` on the licensing group)
- [ ] Remove forwarding at the recorded end date
- [ ] Delete the account after the policy retention period (commonly 90 days), after confirming there is no legal hold

### Close out

- [ ] Attach the group export and a summary of actions to the ticket
- [ ] Confirm with the manager that mailbox and file access work, then close the ticket with dated follow-ups for the deferred steps

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
