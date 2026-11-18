# ad_member

Joins this machine to an existing Active Directory domain under its own computer name, its computer object into
a given OU, and reboots it. A machine that is a member of the domain already is left alone (and not moved to
another OU), so a second run changes nothing. The host name is not changed: naming the machine is the caller's.

**The machine must find the domain first**: its DNS client asks a controller of the domain, or a DNS server that
forwards the domain's zone to one (`nltest /dsgetdc:example.test` answers with a controller). Setting that is the
caller's.

## Variables

| Variable | Required | Default | What |
|---|---|---|---|
| `ad_member_domain` | yes | | the DNS name of the domain (`example.test`) |
| `ad_member_domain_admin_user` | yes | | a user allowed to join machines, as `user@domain` (`admin@example.test`) |
| `ad_member_domain_admin_password` | yes, secret | | that user's password |
| `ad_member_ou` | no | `""` | the OU for the computer object (`OU=servers,DC=example,DC=test`); empty: the domain's default, `CN=Computers` |
| `ad_member_reboot_timeout` | no | `900` | seconds to wait for the machine after the reboot |

The argument spec (`meta/argument_specs.yml`) stops a run that lacks one.

**The account Ansible logs in as.** The join keeps the machine's local accounts and their profiles; a local
administrator Ansible logs in as (by key or password) logs in after the reboot as before. The domain's `Domain
Admins` become members of the machine's local `Administrators`.

## What it does, and the same by hand

1. Is it a member already?

   ```powershell
   (Get-CimInstance Win32_ComputerSystem) | Select-Object PartOfDomain, Domain
   ```

2. The join and the reboot (about a minute):

   ```powershell
   Add-Computer -DomainName example.test -OUPath 'OU=servers,DC=example,DC=test' `
     -Credential (Get-Credential admin@example.test) -Restart
   ```

   If it fails, the reason is at the end of `C:\Windows\debug\NetSetup.LOG`; the role reads that out (the
   module's own error is hidden with the password). Common: the domain not found (DNS), the OU not there, a
   computer object of the same name that belongs to another machine.

## Checking it

```powershell
(Get-CimInstance Win32_ComputerSystem).Domain       # example.test
nltest /sc_verify:example.test                       # the secure channel to a controller: ERROR_SUCCESS
```

On a controller: `Get-ADComputer <name> | Select-Object DistinguishedName` shows the OU.

## Leaving the domain

```powershell
Remove-Computer -UnjoinDomainCredential (Get-Credential admin@example.test) -WorkgroupName WORKGROUP -Restart
```

The computer object stays in the directory, disabled: remove it there (`Remove-ADComputer <name>`) before the
same name joins again from another machine.
