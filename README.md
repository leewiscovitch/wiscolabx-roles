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

Each with its own `README.md` (variables, and the steps by hand it stands for):

| Role | Runs on | Does |
|---|---|---|
| `ad_forest` | the first controller | a new forest, its DNS, the reboot |
| `ad_controller` | each further controller | promotion into the existing domain, replication |
| `ad_dns` | controllers | DNS forwarders, scavenging |
| `ad_time` | controllers | the PDC emulator follows given NTP servers; the others follow the domain |
| `ad_directory` | one controller | OUs, groups, users and memberships from a variable |
| `ad_member` | members | joins the domain into a given OU |

## Using it

The roles do one thing each and know nothing of each other; the playbook that uses them decides the order and
whether a machine is a new forest or joins. One way, the order this collection was written for:

1. **Before the roles** (yours to do: the roles set no address, host name or connection): the machines reach each
   other, the first controller has a fixed address, a further controller uses an existing controller for DNS
2. `ad_forest` on the first controller, **only when no controller of that domain answers**. Decide that from
   what the machines report (`(Get-CimInstance Win32_ComputerSystem).DomainRole`: 4 or 5 is a controller), never
   from a host's place in a list: a first controller rebuilt while another holds the domain must join it, and a
   second forest of the same name splits the domain in two
3. `ad_controller` on every other machine that is no controller yet (one at a time), with a domain admin's
   credentials
4. `ad_dns` and `ad_time` on every controller. `ad_time` reads the machine's role at each run, so after the PDC
   emulator moves (a seize or a transfer), run it again on every controller
5. `ad_directory` once, on one controller (what it makes replicates)
6. `ad_member` on the members

**Not in the roles**, because they differ per environment or need a person's yes: moving or seizing the FSMO
roles, removing a dead controller's objects (metadata cleanup), demotion, the machines themselves. The lab that
uses them keeps those as a runbook.

**The same roles elsewhere.** The home lab reaches its Windows machines over OpenSSH; the AWS version of the same
lab runs the same roles over AWS Systems Manager Session Manager (the `amazon.aws.aws_ssm` connection), with its
own names, addresses and secrets. Nothing in a role assumes either: the connection and every value are the
inventory's.

## Requirements

ansible-core 2.22 or later (`meta/runtime.yml`), `microsoft.ad` 1.12.1 and `ansible.windows` 3.9.2 (`galaxy.yml`).
Windows Server 2025 targets.

## Working on it

Rules for people and agents: `AGENTS.md`. Changes: `CHANGELOG.md`.
