# Changelog

Newest first. The version is `galaxy.yml`'s; each lab installs the collection at a commit, so a change that
alters a role's variables or behaviour also says so here.

## 0.6.0 (2026-11-19)

- `ad_controller`: a machine that is no controller yet first joins the domain as a member (with its reboot) and
  waits until every controller has its account with the same `pwdLastSet` before the promotion. A promotion from a
  workgroup raced replication on a real rotation ("DCPromo exited with 53 ... Access is denied"). New variable
  `ad_controller_account_timeout` (600); `ad_controller_ready_timeout` also bounds the join's reboot. The login
  Ansible uses must work on a member too (a local one does: the join keeps local accounts)
- `ad_demote`: after a demotion that worked and a module that failed after it, the reboot no longer uses
  `win_reboot` (its CIM read of the boot time fails on the session's token and its retry needs a new login, which
  the machine refuses: `UNREACHABLE` on a real demotion). Instead `shutdown.exe /r` over the open session, a wait
  for a new login, and an uptime check (`tasks/reboot.yml`; tested on a member). New variable
  `ad_demote_reboot_delay` (5)

## 0.5.1 (2026-11-19)

- `ad_demote`: when the module fails after a demotion that worked ("No mapping between account names and security
  IDs was done", seen on a real demotion), and both `dcpromo.log` ("no longer a domain controller") and the
  machine's product type (`ServerNT`) say so, the role reboots the machine itself and passes once it is a member
  server, instead of failing unrebooted. No variable changed

## 0.5.0 (2026-11-19)

- New role `ad_demote`: a running controller to a member server of its domain, with the reboot; waits until the
  machine reports itself a member server; checks the local Administrator password before it starts; reads
  `dcpromo.log` when it fails. Not for the last controller nor a holder of an operation master role

## 0.4.0 (2026-11-18)

- New role `ad_directory`: OUs, groups, users (the password only when one is made) and members added to groups;
  checks each user's password against the domain's policy before it starts; never removes
- New role `ad_member`: joins the machine to a domain into a given OU, with the reboot; reads `NetSetup.LOG` when it
  fails

## 0.3.0 (2026-11-18)

- New role `ad_controller`: a further controller of an existing domain, with DNS and a global catalog, the
  reboot, waits until SYSVOL is replicated in and shared; checks the recovery password before it starts

## 0.2.0 (2026-11-18)

- New roles `ad_forest` (a new forest with DNS, the reboot, waits until the directory is ready; checks the recovery
  password before it starts), `ad_dns` (forwarders, scavenging, aging) and `ad_time` (the PDC emulator follows given
  NTP servers, other controllers the domain)

## 0.1.0 (2026-11-18)

- The empty collection: `galaxy.yml` (dependencies `microsoft.ad` 1.12.1, `ansible.windows` 3.9.2),
  `meta/runtime.yml`, the rules (`AGENTS.md`). No roles yet
