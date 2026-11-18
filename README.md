# wiscolabx-roles

The Ansible collection `wiscolabx.roles`: generic roles for shared Windows services, Active Directory first. They
hold no value of any one environment: every name, address and secret comes in as a variable, and the roles say
nothing about how the machines are made or reached.

Written for a home lab on Proxmox and the same lab rebuilt in AWS, which build one domain with the same roles.

## Install

From git at a pinned commit, in your `requirements.yml`:

```yaml
collections:
  - name: microsoft.ad
    version: 1.12.1
  - name: ansible.windows
    version: 3.9.2
  - name: https://github.com/leewiscovitch/wiscolabx-roles.git
    type: git
    version: <commit>
```

```
ansible-galaxy collection install -r requirements.yml
```

## Roles

Each with its own `README.md` (variables, and the steps by hand it stands for). Written so far: `ad_forest`,
`ad_controller`, `ad_dns`, `ad_time`; the others are planned:

| Role | Runs on | Does |
|---|---|---|
| `ad_forest` | the first controller | a new forest, its DNS, the reboot |
| `ad_controller` | each further controller | promotion into the existing domain, replication |
| `ad_dns` | controllers | DNS forwarders, scavenging |
| `ad_time` | controllers | the PDC emulator follows given NTP servers; the others follow the domain |
| `ad_directory` | one controller | OUs, groups, users and memberships from a variable |
| `ad_member` | members | joins the domain into a given OU |

## Requirements

ansible-core 2.22 or later (`meta/runtime.yml`), `microsoft.ad` 1.12.1 and `ansible.windows` 3.9.2 (`galaxy.yml`).
Windows Server 2025 targets.

## Working on it

Rules for people and agents: `AGENTS.md`. Changes: `CHANGELOG.md`.
