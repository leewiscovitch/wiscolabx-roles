# ad_directory

The content of an existing Active Directory domain: organizational units, groups, users, and members added to
groups. Only what it is given is made; nothing else is changed or removed, so it can share a directory with what
Windows and people make. A second run changes nothing.

Runs on a **domain controller** (or a machine with the AD module for PowerShell that reaches one): on a machine
without the AD cmdlets every module fails with `Get-ADRootDSE is not recognized`, check mode included. The
caller picks the machine and runs the role once (one controller; what it makes replicates to the others).

## Variables

| Variable | Required | Default | What |
|---|---|---|---|
| `ad_directory_ous` | no | `[]` | the OUs, parents first: `{name, path}`, `path` the parent's distinguished name |
| `ad_directory_groups` | no | `[]` | security groups: `{name, path, scope, description}`; `scope` `global` (default), `domainlocal`, `universal` |
| `ad_directory_users` | no | `[]` | `{name, path, firstname, surname, groups, password, upn}`; `name` is the account and object name; `groups` added, never set; `password`, `upn` optional |
| `ad_directory_user_password` | when a user has no `password`, secret | `""` | every such user's starting password |
| `ad_directory_memberships` | no | `[]` | `{group, members}`: members added to a group (`Domain Admins` included), never set |
| `ad_directory_domain_admin_user` | no | `""` | a domain admin for the modules (`admin@example.test`); empty: the session's own credentials |
| `ad_directory_domain_admin_password` | no, secret | `""` | that user's password |

Example:

```yaml
ad_directory_ous:
  - {name: example, path: "DC=example,DC=test"}
  - {name: users, path: "OU=example,DC=example,DC=test"}
  - {name: groups, path: "OU=example,DC=example,DC=test"}
ad_directory_groups:
  - {name: example-admins, path: "OU=groups,OU=example,DC=example,DC=test", scope: global}
ad_directory_users:
  - {name: alice, path: "OU=users,OU=example,DC=example,DC=test", firstname: Alice, surname: Example,
     groups: [example-admins]}
ad_directory_user_password: "{{ vault_example_user_password }}"
ad_directory_memberships:
  - {group: Domain Admins, members: [example-admins]}
```

**The password is a starting one**: it is set only when the user is made (`update_password: on_create`). The
module's default (`always`) sets it on every run and so reports a change each time; changing it later is the
user's, or `Set-ADAccountPassword` by hand.

**A password the domain would refuse** is found before anything is made: the role reads the domain's policy
(`Get-ADDefaultDomainPasswordPolicy`: minimum length, complexity) and checks each user's password against it
(with complexity: three of upper case, lower case, digits and other characters, and not the account name in it).
The run stops naming the users and the variable; the value is never shown. A fine-grained policy is not read: if
one refuses the password, the users step fails without the reason (its error is hidden with the password).

**Check mode** says what it would make, but an OU or group made by an earlier step is not there yet in a check run:
the lookups of `groups` and `members` then warn (`Failed to find the AD object DNs ... Ignoring`) instead of
failing. In a real run they fail.

## What it does, and the same by hand

On a controller, as a domain admin (PowerShell; the AD module comes with the controller's tools):

1. The policy the passwords must meet:

   ```powershell
   Get-ADDefaultDomainPasswordPolicy | Select-Object MinPasswordLength, ComplexityEnabled
   ```

2. The OUs, parents first (made protected from accidental deletion, as `New-ADOrganizationalUnit` does by default):

   ```powershell
   New-ADOrganizationalUnit -Name example -Path 'DC=example,DC=test'
   New-ADOrganizationalUnit -Name users -Path 'OU=example,DC=example,DC=test'
   ```

3. The groups:

   ```powershell
   New-ADGroup -Name example-admins -SamAccountName example-admins -GroupScope Global -GroupCategory Security `
     -Path 'OU=groups,OU=example,DC=example,DC=test'
   ```

4. The users, enabled, then into their groups:

   ```powershell
   New-ADUser -Name alice -SamAccountName alice -GivenName Alice -Surname Example `
     -Path 'OU=users,OU=example,DC=example,DC=test' -Enabled $true `
     -AccountPassword (Read-Host -AsSecureString 'starting password')
   Add-ADGroupMember -Identity example-admins -Members alice
   ```

5. Members into other groups:

   ```powershell
   Add-ADGroupMember -Identity 'Domain Admins' -Members example-admins
   ```

## Checking it

```powershell
Get-ADObject -SearchBase 'OU=example,DC=example,DC=test' -Filter * | Select-Object ObjectClass, DistinguishedName
Get-ADGroupMember 'Domain Admins' | Select-Object SamAccountName
Get-ADUser alice -Properties MemberOf | Select-Object Enabled, MemberOf
```

Run on each controller (each answers from its own copy), it also shows that replication carried the objects.

## Taking something away

The role never removes. By hand (an OU made by the role is protected):

```powershell
Remove-ADUser alice -Confirm:$false
Set-ADOrganizationalUnit 'OU=users,OU=example,DC=example,DC=test' -ProtectedFromAccidentalDeletion $false
Remove-ADOrganizationalUnit 'OU=users,OU=example,DC=example,DC=test' -Recursive -Confirm:$false
```
