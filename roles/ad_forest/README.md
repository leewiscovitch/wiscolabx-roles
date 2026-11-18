# ad_forest

A new Active Directory forest on this machine, with its DNS: the first controller of a new domain. A machine that is
a controller already is left alone, so a second run changes nothing.

**It makes a forest whenever the machine is no controller.** Whether that is right is the caller's decision: a
machine rebuilt while another controller still holds a domain of the same name must join that domain
(`microsoft.ad.domain_controller`), never get this role, or the domain is split in two.

## Variables

| Variable | Required | Default | What |
|---|---|---|---|
| `ad_forest_domain` | yes | | the forest root domain's DNS name (`example.test`) |
| `ad_forest_netbios` | yes | | the domain's NetBIOS name, at most 15 characters (`EXAMPLE`) |
| `ad_forest_safe_mode_password` | yes, secret | | the Directory Services Restore Mode password |
| `ad_forest_ready_timeout` | no | `900` | seconds to wait for the reboot and for the directory to be ready |

The argument spec (`meta/argument_specs.yml`) stops a run that lacks one. The recovery password is checked before
anything is done (at least 8 characters, from three of upper case, lower case, digits and other characters): the
module accepts any value in check mode, and Windows refuses a weak one only deep inside the promotion.

**The built-in `Administrator`** (the local account whose SID ends in `-500`) becomes the domain's `Administrator`
with the password it has, and the promotion enables it. Windows refuses a blank one (`DCPromo exited with 94`, after
most of a quarter of an hour). Giving it a password, and what happens to it afterwards, is the caller's.

## What it does, and the same by hand

1. The features, with their tools (about 70 s, no reboot):

   ```powershell
   Install-WindowsFeature AD-Domain-Services, DNS -IncludeManagementTools
   ```

2. The forest, then the reboot (about 90 s with the reboot):

   ```powershell
   Install-ADDSForest -DomainName example.test -DomainNetbiosName EXAMPLE -InstallDns `
     -SafeModeAdministratorPassword (Read-Host -AsSecureString 'DSRM password')
   ```

   It also points the NIC's DNS client at `127.0.0.1` alone. If it fails, the reason is at the end of
   `C:\Windows\debug\dcpromo.log`; the role reads that out (the module's own error is hidden with the password).

3. Waits until the directory is ready, with local reads only:

   ```powershell
   Get-Service NTDS, ADWS                       # both Running (the AD cmdlets and modules need ADWS)
   (Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters).SysvolReady   # 1
   Get-SmbShare SYSVOL, NETLOGON                # both there
   nltest /dsgetdc:example.test                 # names this machine
   ```

**Check mode** on a machine that is no controller: the features and the forest are predicted as changed, the waits
are skipped and a message says what would be made.
