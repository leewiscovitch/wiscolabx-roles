# wiscolabx-roles: instructions for AI agents and humans

Canonical instruction file for every agent working in this repo (Claude Code reads it via `CLAUDE.md`).

The Ansible collection `wiscolabx.roles`: generic roles for shared Windows services, Active Directory first
(`README.md`). Public. Every lab that uses it installs it from git at a pinned commit and passes its own values.

## Rules

- **No value of any one lab.** No address, host name, domain name, user name, password, node or URL of a real
  environment as a default, in an example or in a comment. Examples use `example.test`, `EXAMPLE`, `192.0.2.x`
- **A variable with no sensible general default is required**: the role's first task is an `assert` that stops
  with a message naming the variable
- **Role variables carry the role's name as prefix** (`ad_forest_...`). Nothing reads a lab's variables
  (`labx_*` or any other)
- **Nothing about how the machine is reached or made**: no connection variables, no address of the machine's own
  NIC, no host name set, no assumption about the transport (SSH, WinRM, ...) or about which account Ansible logs in
  as beyond "an administrator"
- **Secrets come in as variables and are never logged**: every task that carries one has `no_log: true`. No secret
  in the repo
- **Every role can run twice with `changed=0`**, and in check mode as far as the modules allow; where a check run
  can't know (before the forest exists), it says what it would do instead of failing
- **Each role's `README.md`** lists its variables (required ones marked) and the steps by hand it stands for, in the
  commit that adds the role
- **Argument names from the current docs** of the pinned versions (`ansible-doc microsoft.ad.<module>`), not from
  memory. The pins are in `galaxy.yml`; a lab pins the same versions in its own `requirements.yml`
- **`ansible-lint` before every commit** (production profile, `.ansible-lint`). There is no CI: the real test is a
  lab's playbook against real machines, which says so in that lab's repo
- A change to a role's variables or behaviour gets a line in `CHANGELOG.md` and, when a lab must change its side,
  a new `version` in `galaxy.yml`
- Explain as well as automate: each role's `README.md` and `docs/` say each step, why it is there, what to check,
  what goes wrong
- Small commits about one thing, with the reason in the message. No force-push, no history rewrite
