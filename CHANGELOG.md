# Changelog

Newest first. The version is `galaxy.yml`'s; each lab installs the collection at a commit, so a change that
alters a role's variables or behaviour also says so here.

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
