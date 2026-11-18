# ad_controller

A further controller of an existing Active Directory domain on this machine, with its DNS and a global catalog. A
machine that is a controller already is left alone, so a second run changes nothing.

**The machine must find the domain first**: its DNS client asks a controller of the domain (or a DNS server that
forwards the domain's zone to one). Setting the NIC is the caller's.

## Variables

| Variable | Required | Default | What |
|---|---|---|---|
| `ad_controller_domain` | yes | | the DNS name of the domain to join as a controller (`example.test`) |
| `ad_controller_domain_admin_user` | yes | | a member of `Domain Admins`, as `user@domain` (`admin@example.test`) |
| `ad_controller_domain_admin_password` | yes, secret | | that user's password |
| `ad_controller_safe_mode_password` | yes, secret | | this controller's Directory Services Restore Mode password |
| `ad_controller_ready_timeout` | no | `1800` | seconds to wait for the reboot and for the directory to be ready |

The argument spec (`meta/argument_specs.yml`) stops a run that lacks one. The recovery password is checked before
anything is done (at least 8 characters, from three of upper case, lower case, digits and other characters), as in
`ad_forest`.

**The account Ansible logs in as.** A further controller keeps no local accounts: after the reboot the same name
is the domain's account, with a new profile and none of the old one's files. An SSH key login therefore only comes
back if the key is somewhere the domain account reads too (with OpenSSH for Windows: `administrators_authorized_keys`
and the `Match Group administrators` block in `sshd_config`), and the domain account is an administrator there.
Otherwise the promotion works, but the module waits for the login until `ad_controller_ready_timeout` runs out and
fails (`Failed to reboot after module returned reboot_required`). Preparing that is the caller's.

## What it does, and the same by hand

1. The features, with their tools (about 70 s, no reboot):

   ```powershell
   Install-WindowsFeature AD-Domain-Services, DNS -IncludeManagementTools
   ```

2. The promotion, then the reboot (about 90 s with the reboot):

   ```powershell
   Install-ADDSDomainController -DomainName example.test -InstallDns `
     -Credential (Get-Credential admin@example.test) `
     -SafeModeAdministratorPassword (Read-Host -AsSecureString 'DSRM password')
   ```

   A writable controller and a global catalog in the default site; the directory is copied from a partner it
   finds. It points the NIC's DNS client at a partner, then `127.0.0.1`. If it fails, the reason is at the end of
   `C:\Windows\debug\dcpromo.log`; the role reads that out (the module's own error is hidden with the passwords).

3. Waits until the directory is ready, with local reads only:

   ```powershell
   Get-Service NTDS, ADWS                       # both Running
   (Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters).SysvolReady   # 1
   Get-SmbShare SYSVOL, NETLOGON                # both there: only after SYSVOL's first replication in
   ```

## Reading replication

Run as a domain admin (a remote session without passed-on credentials only sees its own controller and reports
error 110 for the others):

```powershell
repadmin /replsummary            # fails/total per controller, as source and as destination: 0 fails
repadmin /showrepl               # each partner, each naming context, the last success
Get-ADDomainController -Filter * | Select-Object Name, IsGlobalCatalog
```

A test object made on one controller shows on the other within about a minute (between controllers in one site
replication is notified, about 15 s, not scheduled).

**Check mode** on a machine that is no controller: the features and the promotion are predicted as changed, the
waits are skipped and a message says what would be done.
