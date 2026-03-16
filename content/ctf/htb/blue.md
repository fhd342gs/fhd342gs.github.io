---
title: "HTB: Blue"
date: 2023-12-17
draft: false
os: "Windows"
difficulty: "Easy"
platform: "HTB"
aliases: ["/posts/htb/blue/"]
tags:
  - htb
  - easy
  - windows
  - ms17-010
  - eternalblue
categories:
  - walkthrough
---

{{< box-info name="Blue" os="Windows" difficulty="Easy" ip="10.129.44.168" platform="HTB" >}}

---

## Recon

Nothing special needed here -- straight to enumeration.

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT      STATE SERVICE
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
```
{{< /collapse >}}

**OS**: Windows 7 Professional 7601 Service Pack 1 x64

### Port 445 - SMB

Running NetExec against SMB with guest credentials:

```bash
netexec smb 10.129.44.168 -u 'guest' -p '' --shares
```

```
SMB  10.129.44.168  445  HARIS-PC  [*] Windows 7 Professional 7601 Service Pack 1 x64
     (name:HARIS-PC) (domain:haris-PC) (signing:False) (SMBv1:True)
[+] haris-PC\guest:
[*] Enumerated shares
Share           Permissions     Remark
-----           -----------     ------
ADMIN$                          Remote Admin
C$                              Default share
IPC$                            Remote IPC
Share           READ
Users           READ
```

SMBv1 enabled on Windows 7 SP1 -- that's a strong hint.

---

## Foothold

### MS17-010 (EternalBlue)

Checking for the vulnerability:

```bash
netexec smb 10.129.44.168 -u 'guest' -p '' -M MS17-010
```

```
MS17-010 [+] 10.129.44.168 is likely VULNERABLE to MS17-010!
         (Windows 7 Professional 7601 Service Pack 1)
```

Fire the Metasploit `ms17_010_eternalblue` module and get instant SYSTEM. No privesc needed.

---

## Proof

{{< terminal title="nt authority\system@HARIS-PC" >}}
C:\Users>type haris\Desktop\user.txt
[redacted]

C:\Users>type Administrator\Desktop\root.txt
[redacted]
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- Always check for SMBv1 on older Windows boxes -- MS17-010 is a free SYSTEM shell
- NetExec's MS17-010 module is a quick way to confirm the vulnerability before firing Metasploit
- This box took ~30 minutes, zero privesc required
{{< /callout >}}
