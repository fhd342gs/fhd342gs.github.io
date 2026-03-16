---
title: "PG: Algernon"
date: 2023-01-19
draft: false
os: "Windows"
difficulty: "Easy"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/algernon/"]
tags:
  - offsec
  - proving-grounds
  - easy
  - windows
  - smartermail
  - rce
  - cve-2019-7214
categories:
  - walkthrough
---

{{< box-info name="Algernon" os="Windows" difficulty="Easy" ip="192.168.197.65" platform="OffSec Proving Grounds" >}}

---

## Recon

### Nmap

{{< collapse title="Full nmap output" >}}
```
21/tcp    open  ftp           Microsoft ftpd
80/tcp    open  http          Microsoft IIS httpd 10.0
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
5040/tcp  open  unknown
9998/tcp  open  http          Microsoft IIS httpd 10.0
17001/tcp open  remoting      MS .NET Remoting services
```
{{< /collapse >}}

---

## Enumeration

### Port 21 - FTP

Anonymous access is allowed. Downloaded the FTP contents:

```bash
wget -m ftp://anonymous:anonymous@192.168.197.65:21
```

Directory listing shows mail-related folders (`ImapRetrieval`, `Logs`, `PopRetrieval`, `Spool`) -- nothing interesting after grepping through.

### Port 80 - IIS

Default IIS starting page. Nothing here.

### Port 9998 - SmarterMail

![SmarterMail login page](/images/algernon/enum.png)

Grabbed the version via Burp:

![SmarterMail version in Burp](/images/algernon/enum-burp.png)

Searchsploit hit: `SmarterMail Build 6985 - Remote Code Execution`

{{< callout type="flag" >}}
**CVE-2019-7214** -- SmarterMail versions below build 6985 are vulnerable to RCE.
{{< /callout >}}

---

## Foothold

### SmarterMail RCE (CVE-2019-7214)

```bash
searchsploit smartermail
```

Found `SmarterMail Build 6985 - Remote Code Execution`. The exploit works on versions lower than build 6985. Modified the exploit with our IP/port and fired it off.

Lands us directly as **SYSTEM** -- no privesc needed.

---

## Privilege Escalation

Not required. The SmarterMail RCE exploit gives us a **SYSTEM** shell immediately.

---

## Proof

{{< terminal title="proof" >}}
local.txt: N/A
proof.txt: [redacted]
{{< /terminal >}}

![Root flag](/images/algernon/root-flag.png)

---

## Key Takeaways

{{< callout type="note" >}}
- Always enumerate non-standard ports -- port 9998 had the vulnerable SmarterMail instance
- Check service versions with Burp or browser dev tools when the web UI doesn't display them
- CVE-2019-7214 gives direct SYSTEM access on SmarterMail < Build 6985
{{< /callout >}}
