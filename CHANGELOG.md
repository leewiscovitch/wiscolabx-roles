# Changelog

Newest first. The version is `galaxy.yml`'s; each lab installs the collection at a commit, so a change that
alters a role's variables or behaviour also says so here.

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
