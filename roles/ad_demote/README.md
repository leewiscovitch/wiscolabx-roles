# ad_demote

Demotes this controller to a member server of its domain: Active Directory leaves the machine, the domain's copy
of it is removed from the other controllers' view by the demotion itself (its controller object, its DNS records,
its replication links), and the machine reboots as an ordinary member. A machine that is no controller is left
alone, so a second run changes nothing. The AD DS feature stays installed.

**Not for**:

- **a domain's last controller**: the demotion stops (it would end the domain; the cmdlet wants
  `-LastDomainControllerInDomain` for that, which this role never passes)
- **a holder of an operation master (FSMO) role**: the module refuses before it starts, and the role says so. Move
  the roles to another controller first (`Move-ADDirectoryServerOperationMasterRole`, as a member of `Domain Admins`
  for the domain's three and `Enterprise Admins` and `Schema Admins` for the forest's two). Seizing is for a
  controller that is gone, never for one that runs
- **a controller that can't reach another**: the demotion replicates its last changes out and tells a partner it is
  leaving; without a partner it fails. A controller that is dead is removed from another controller instead
  (metadata cleanup), not with this role

## Variables

| Variable | Required | Default | What |
|---|---|---|---|
| `ad_demote_domain_admin_user` | yes | | a member of `Domain Admins`, as `user@domain` (`admin@example.test`) |
| `ad_demote_domain_admin_password` | yes, secret | | that user's password |
| `ad_demote_local_admin_password` | yes, secret | | the password the local `Administrator` gets back once the machine is no controller |
| `ad_demote_reboot_timeout` | no | `900` | seconds to wait for the reboot and for the machine to report itself a member server |
| `ad_demote_reboot_delay` | no | `5` | seconds between `shutdown.exe /r` and the reboot, when the role reboots the machine itself (below) |

The argument spec (`meta/argument_specs.yml`) stops a run that lacks one. The local `Administrator` password is
checked before anything is done (at least 8 characters, from three of upper case, lower case, digits and other
characters), as `ad_controller`'s recovery password; the check says yes or no, never the value.

**The account Ansible logs in as.** A controller has no local accounts; a member server has them again, with the
local `Administrator` (and only it) enabled, holding the password given here. The domain's accounts still log in,
and `Domain Admins` is in the local `Administrators` group of a member. So a login as a domain admin keeps working
after the reboot; a login as a local account other than `Administrator` does not (that account does not exist
yet). Preparing that is the caller's: otherwise the demotion works, but the module waits for the login until
`ad_demote_reboot_timeout` runs out and fails (`Failed to reboot after module returned reboot_required`).

## What it does, and the same by hand

1. Before anything, on the controller: it holds no operation master role, and another controller answers:

   ```powershell
   netdom query fsmo                              # this controller is named on no line
   Get-ADDomainController -Filter * | Select-Object Name   # another one is listed
   repadmin /replsummary                          # 0 fails: a demotion replicates out first
   ```

2. The demotion, then the reboot (about two minutes with the reboot):

   ```powershell
   Uninstall-ADDSDomainController `
     -Credential (Get-Credential admin@example.test) `
     -LocalAdministratorPassword (Read-Host -AsSecureString 'local Administrator password')
   ```

   It replicates its last changes to a partner, removes its NTDS settings object and its DNS records (the
   controller's SRV records, its name server entries), turns its computer object back into a member's, and asks
   for the reboot. The DNS server service and the AD DS binaries stay. If it fails, the reason is at the end of
   `C:\Windows\debug\dcpromo.log`; the role reads that out (the module's own error is hidden with the passwords).

   **A demotion that worked and a module that failed after it.** The module can fail at its very end, after the
   demotion, with "No mapping between account names and security IDs was done" (the session's token is a domain
   account's, which the machine no longer maps), and then it does not reboot. The role reads the log and the
   machine's product type; when the log says "This machine is no longer a domain controller" and the product type
   is `ServerNT` (a controller's is `LanmanNT`), it reboots the machine over the session still open and goes on as
   after a clean run (`tasks/reboot.yml`).

   **Not with `win_reboot`** (tried in 0.5.1, `UNREACHABLE` on a real demotion): it first reads the boot time through
   CIM (`Win32_OperatingSystem`), which fails on that session's token; its retry resets the connection, and the new
   login is refused (no domain vouches for the account any more, and the machine has no local one but
   `Administrator`). So the role reads the uptime with the kernel's `GetTickCount64` (no CIM), runs
   `shutdown.exe /r /t 5` over the open session, drops the session, waits until a new login works (only after the
   reboot), and checks the uptime went down. If the read before fails too, the login that works is the evidence.
   If `shutdown.exe` refuses, it says to reboot from outside. By hand, the same check and the reboot:

   ```powershell
   Get-Content C:\Windows\debug\dcpromo.log -Tail 15        # "This machine is no longer a domain controller"
   (Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\ProductOptions).ProductType   # ServerNT
   shutdown.exe /r /t 5        # Restart-Computer goes through CIM, which this session's token fails
   ```

   A new login before that reboot fails (no domain answers on the machine, and it has no local account yet but
   `Administrator`); if the session is gone too, the role says so, and the machine is rebooted from outside (its
   console or hypervisor). Run the role again after: it finds a member server and changes nothing.

3. Waits until the machine is a member server, with a local read:

   ```powershell
   (Get-CimInstance Win32_ComputerSystem).DomainRole    # 3: member server (4 and 5 are controllers)
   ```

## What to check on another controller

Run as a domain admin on a controller that stays:

```powershell
Get-ADDomainController -Filter * | Select-Object Name       # the demoted one is gone
repadmin /replsummary                                       # no partner left that fails
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.example.test # no record names the demoted one
Get-ADComputer <name> | Select-Object DistinguishedName     # in CN=Computers (or where it is moved), no longer OU=Domain Controllers
```

The demoted machine's computer object stays in the domain as a member's. Removing it, and the machine, is the
caller's.

## What goes wrong

- **It holds a role**: the module stops before it starts, naming the roles (`This domain controller has operation
  master role(s) (...) assigned`: the run shows that line, it holds no password); the role adds the end of
  `dcpromo.log`, which is then an older operation's, and says to check `netdom query fsmo`. Move the roles, run
  again
- **No partner answers, or replication fails**: `dcpromo.log` says it could not replicate or reach a controller.
  Fix replication first (`repadmin /replsummary` on each controller)
- **The login after the reboot fails**: the demotion is done, the wait runs out. Log in another way, check
  `DomainRole` by hand (3 is done); the account Ansible uses must be one a member server knows (above)
- **The DNS client still points at itself**: a demoted machine whose NIC lists `127.0.0.1` first asks a DNS server
  that no longer holds the domain's zone. Setting the NIC is the caller's, as before the promotion

**Check mode** on a controller: predicted as changed (no role held), the wait is skipped and a message says what
would be done. On a controller that holds a role, check mode fails the same way a real run does. On a machine that is
no controller: nothing changes.
