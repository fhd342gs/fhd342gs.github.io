---
title: "PG: Exfiltrated"
date: 2024-02-10
draft: false
os: "Linux"
difficulty: "Easy"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/exfiltrated/"]
tags:
  - offsec
  - proving-grounds
  - easy
  - linux
  - subrion-cms
  - rce
  - cron
  - exiftool
  - cve-2021-22204
categories:
  - walkthrough
---

{{< box-info name="Exfiltrated" os="Linux" difficulty="Easy" ip="192.168.192.163" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
```
{{< /collapse >}}

Port 80 redirects to `exfiltrated.offsec` — add it to `/etc/hosts` before anything else resolves. The site is a [Subrion CMS](https://subrion.org/) install.

---

## Foothold

### Subrion CMS — Authenticated RCE

The admin panel at `/panel/` accepts the laziest credential pair in the book: `admin:admin`. Once inside, the dashboard reports **Subrion 4.2**, which is vulnerable to an authenticated file-upload RCE ([EDB-49876](https://www.exploit-db.com/exploits/49876)) — the uploader's extension blacklist misses the PHP-executable `.pht`.

```bash
python 49876.py -u http://exfiltrated.offsec/panel/ -l admin -p admin
```

Shell as `www-data`.

---

## Privilege Escalation

### Root cron + exiftool — CVE-2021-22204

A pair of files sits in `/opt` — a script and a `metadata` directory:

```bash
#!/bin/bash
# A BASH script to collect EXIF metadata
IMAGES='/var/www/html/subrion/uploads'
META='/opt/metadata'
FILE=`openssl rand -hex 5`
ls $IMAGES | grep "jpg" | while read filename; do
    exiftool "$IMAGES/$filename" >> "$META/$FILE"
done
```

It runs `exiftool` as **root** every minute (confirm with `pspy`) against any `.jpg` dropped into the Subrion uploads directory. That's a direct line to [CVE-2021-22204](https://github.com/convisolabs/CVE-2021-22204-exiftool) — arbitrary code execution via a malicious DjVu-embedded image.

The naive `exiftool "-comment<=payload"` injection won't trigger here — you need the real DjVu exploit. Generate a poisoned image carrying a reverse-shell payload, drop it in `/var/www/html/subrion/uploads`, and wait for the cron to swallow it.

Root.

---

## Proof

![Root proof — root and local flags, whoami, and hostname](/images/exfiltrated/proof.png)

---

## Key Takeaways

{{< callout type="note" >}}
- Always try trivial credentials (`admin:admin`) on CMS panels before pulling out heavier tooling — Subrion handed over the panel for free.
- `pspy` is the fastest way to catch root cron jobs that never appear in `/etc/crontab` — the `/opt` script here fired every 60 seconds.
- Any process feeding attacker-controllable files into a vulnerable `exiftool` is a CVE-2021-22204 privilege escalation. The `-comment` trick is a red herring; use the DjVu PoC.
{{< /callout >}}
