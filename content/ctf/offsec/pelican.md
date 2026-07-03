---
title: "PG: Pelican"
date: 2024-02-15
draft: false
os: "Linux"
difficulty: "Medium"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/pelican/"]
tags:
  - offsec
  - proving-grounds
  - medium
  - linux
  - zookeeper
  - exhibitor
  - memory-dump
  - gcore
  - sudo
categories:
  - walkthrough
---

{{< box-info name="Pelican" os="Linux" difficulty="Medium" ip="192.168.53.98" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Full nmap output" >}}
```
PORT      STATE SERVICE
22/tcp    open  ssh
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
631/tcp   open  ipp
2181/tcp  open  eforward
2222/tcp  open  EtherNetIP-1
8080/tcp  open  http-proxy
8081/tcp  open  blackice-icecap
44091/tcp open  unknown
```
{{< /collapse >}}

### Port 8081 / 8080 - Exhibitor for ZooKeeper

Nmap reveals nginx on 8081 redirecting to Exhibitor's web UI:

```
8081/tcp  open  http        nginx 1.14.2
|_http-title: Did not follow redirect to http://192.168.53.98:8080/exhibitor/v1/ui/index.html
```

![Exhibitor UI](/images/pelican/exhibitor.png)

---

## Foothold

### Exhibitor RCE

Exhibitor's web UI allows editing the ZooKeeper config. This is a known RCE vector -- you can inject shell commands into the config fields.

![RCE via Exhibitor config](/images/pelican/rce.png)

Got a shell as a low-privilege user.

---

## Privilege Escalation

### Sudo + gcore Memory Dump

Checking `sudo -l` reveals we can run `gcore` without a password. `gcore` creates memory dumps of running processes.

Running `linpeas.sh` pointed toward credential extraction from process memory ([HackTricks reference](https://book.hacktricks.wiki/linux-hardening/privilege-escalation#process-memory)).

Found a password-related cron job process. Dumped its memory:

```bash
sudo gcore 493
```

![gcore dump process](/images/pelican/gcore.png)

Searched the dump for credentials:

```bash
strings core.493 | grep -i password -A5 -B5
```

![Password found in memory dump](/images/pelican/password-dump.png)

Found root credentials: `root:ClogKingpinInning731`

```bash
su root
# ClogKingpinInning731
```

---

## Proof

{{< terminal title="proof" >}}
local.txt: [redacted]
proof.txt: [redacted]
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- Exhibitor for ZooKeeper has a well-known RCE via config editing -- if the UI is exposed without auth, it's game over
- `sudo gcore` is a powerful privesc vector -- you can dump any process's memory and extract credentials with `strings`
- Always check `sudo -l` and look for uncommon binaries -- `gcore` isn't a typical GTFOBins entry but is still exploitable
- Cron jobs that handle passwords are prime targets for memory dumping
{{< /callout >}}
