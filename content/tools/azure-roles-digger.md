---
title: "azure-roles-digger.sh"
date: 2026-03-16
draft: false
tags:
  - azure
  - entra-id
  - rbac
  - pim
  - powershell
  - bash
  - cloud-security
categories:
  - tools-and-techniques
description: "Comprehensive Azure/Entra ID role and permission discovery — finds everything including inherited and PIM-eligible roles that basic queries miss."
---

## The problem

Azure permission auditing is a special kind of hell.

You run `az role assignment list` and get back... direct assignments. That's it. No inherited roles from group nesting. No Entra ID directory roles. No PIM-eligible roles sitting there waiting to be activated. Nothing about what that service principal can *actually* do when you trace the full chain.

So you're left clicking through 15 portal blades, running separate Graph queries, cross-referencing group memberships by hand, piecing it all together in a spreadsheet like it's 2005. And after 30 minutes you're still not sure you found everything.

Built this script because doing that monkey job one more time was going to give me an aneurysm.

---

## What it does

Feed it any Entra Object ID — user, service principal, group, doesn't matter — and it finds **all** effective permissions:

| Category | What it finds |
|----------|---------------|
| Direct Azure RBAC | Role assignments directly on the identity |
| Inherited Azure RBAC | Roles inherited via group memberships (transitive) |
| Entra ID Directory Roles | Global Admin, User Admin, etc. — direct + through groups |
| PIM Eligible Roles | Roles that can be activated via Privileged Identity Management |

The key word is **transitive**. User is in Group A, which is in Group B, which has Contributor on a subscription? The script finds that. `az role assignment list` does not.

---

## Features

The stuff that actually matters:

- **Auto-detects identity type** — pass any Object ID, it figures out if it's a user, SP, or group
- **Transitive group resolution** — walks the full group nesting chain via Microsoft Graph
- **Full permission extraction** — actions, notActions, dataActions, notDataActions for every role
- **Table output** — human-readable tables with role name, scope, assignment type, and source
- **JSON output** — `--json` flag for piping into jq or feeding into reports
- **Selective discovery** — `--skip-pim`, `--skip-entra`, `--skip-rbac` to speed things up
- **Parallel subscription queries** — doesn't crawl subscriptions one by one like a caveman
- **Timeout protection** — won't hang forever if Graph decides to take a nap

---

## Usage

Basic full audit — give it an Object ID and let it rip:

```bash
./azure-roles-digger.sh a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

Quick RBAC-only check when you don't need the Entra/PIM overhead:

```bash
./azure-roles-digger.sh --skip-entra --skip-pim $OID
```

Export to JSON for reporting or downstream processing:

```bash
./azure-roles-digger.sh --json --quiet $OID > audit.json
```

Check what a service principal can actually do (this one's fun — SPs accumulate permissions like lint):

```bash
./azure-roles-digger.sh $SP_OID
```

---

## Example output

### Table mode (default)

This is what you get out of the box. One command, full picture:

{{< collapse title="Full table output example" >}}
```
============================================================
  AZURE ROLE DIGGER — Comprehensive Permission Discovery
============================================================

[*] Target Object ID: a1b2c3d4-e5f6-7890-abcd-ef1234567890
[*] Identity Type:    User
[*] Display Name:     john.doe@contoso.com

------------------------------------------------------------
  DIRECT AZURE RBAC ASSIGNMENTS
------------------------------------------------------------
Role Name         Scope                                    Type
---------         -----                                    ----
Reader            /subscriptions/sub-1111                  Direct
Contributor       /subscriptions/sub-2222/rg/prod-rg       Direct

------------------------------------------------------------
  INHERITED AZURE RBAC (via group membership)
------------------------------------------------------------
Role Name         Scope                                    Source Group
---------         -----                                    ------------
Owner             /subscriptions/sub-3333                  SG-Azure-Admins
Key Vault Admin   /subscriptions/sub-2222/rg/kv-rg         SG-KV-Team
                                                           └─ via SG-DevOps (nested)

------------------------------------------------------------
  ENTRA ID DIRECTORY ROLES
------------------------------------------------------------
Role Name                    Assignment    Source
---------                    ----------    ------
Global Reader                Direct        —
User Administrator           Inherited     SG-IT-Admins
Privileged Role Admin        Inherited     SG-Azure-Admins
                                           └─ via SG-Security (nested)

------------------------------------------------------------
  PIM ELIGIBLE ROLES
------------------------------------------------------------
Role Name                    Status        Scope
---------                    ------        -----
Global Administrator         Eligible      Directory
Application Administrator    Eligible      Directory

============================================================
  SUMMARY
============================================================
[+] Direct RBAC:      2 assignments
[+] Inherited RBAC:   2 assignments (via 3 groups)
[+] Entra Roles:      3 roles (1 direct, 2 inherited)
[+] PIM Eligible:     2 roles
[!] Total Groups:     5 (including nested)
============================================================
```
{{< /collapse >}}

### JSON mode

```json
{
  "identity": {
    "objectId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "type": "User",
    "displayName": "john.doe@contoso.com"
  },
  "rbac": {
    "direct": [...],
    "inherited": [...]
  },
  "entraRoles": {
    "direct": [...],
    "inherited": [...]
  },
  "pim": {
    "eligible": [...]
  },
  "groups": {
    "total": 5,
    "chain": [...]
  }
}
```

Pipe it through `jq` and extract whatever you need. Feed it into a SIEM. Stick it in a report. Whatever.

---

## Requirements

| Requirement | Details |
|-------------|---------|
| Shell | Bash 3.2+ (macOS default works) |
| Azure CLI | `az` — authenticated with `az login` |
| jq | For JSON parsing |
| Permissions | See below |

### Required permissions on your authenticated identity

| Permission | Why |
|------------|-----|
| Reader | On target subscriptions (for RBAC enumeration) |
| Directory.Read.All | Graph API — reading group memberships and directory roles |
| RoleManagement.Read.Directory | Graph API — reading Entra role assignments |
| Privileged Role Reader | For PIM eligible role discovery (optional — use `--skip-pim` without it) |

{{< callout type="info" >}}
If you don't have all permissions, the script won't crash — it'll skip what it can't access and tell you what it missed. Use the `--skip-*` flags to suppress warnings for stuff you intentionally can't query.
{{< /callout >}}

---

## Source

Code is on GitHub: [fhd342gs/AZURE_security_scripts](https://github.com/fhd342gs/AZURE_security_scripts)

{{< callout type="warning" >}}
For authorized security testing and auditing only. Don't be stupid.
{{< /callout >}}
