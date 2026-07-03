---
title: "PG: ClamAV"
date: 2022-10-13
draft: false
os: "Linux"
difficulty: "Easy"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/clamav/"]
tags:
  - offsec
  - proving-grounds
  - easy
  - linux
  - clamav
  - sendmail
  - snmp
  - smtp
categories:
  - walkthrough
---

{{< box-info name="ClamAV" os="Linux" difficulty="Easy" ip="192.168.57.42" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Full nmap output (TCP)" >}}
```
PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 3.8.1p1 Debian 8.sarge.6 (protocol 2.0)
25/tcp    open  smtp        Sendmail 8.13.4/8.13.4/Debian-3sarge3
80/tcp    open  http        Apache httpd 1.3.33 ((Debian GNU/Linux))
139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
199/tcp   open  smux        Linux SNMP multiplexer
445/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
60000/tcp open  ssh         OpenSSH 3.8.1p1 Debian 8.sarge.6 (protocol 2.0)
```
{{< /collapse >}}

{{< collapse title="Full nmap output (UDP)" >}}
```
PORT     STATE         SERVICE      VERSION
137/udp  open          netbios-ns   Samba nmbd netbios-ns (workgroup: WORKGROUP)
161/udp  open          snmp         SNMPv1 server (public)
```
{{< /collapse >}}

{{< callout type="warning" >}}
Don't forget to enumerate UDP. This box has **SNMP v1** on UDP 161 which reveals critical information.
{{< /callout >}}

Ports 199 (smux -- the SNMP multiplexer, tied to the SNMP service on UDP 161) and 60000 (a second OpenSSH listener) are open but weren't part of the path.

### Port 25 - SMTP (Sendmail)

Sendmail 8.13.4 is ancient. Searchsploit reveals multiple exploits, and importantly -- one that involves **ClamAV** (hint from the box name):

```
Sendmail with clamav-milter < 0.91.2 - Remote Command Execution
```

### Port 80 - Apache

The web page contains binary-encoded text. Decoding it gives `ifyoudontpwnmeuran00b` -- looks like a password. The page title is `Ph33r`. This turned out to be a rabbit hole.

### Port 161 (UDP) - SNMP

SNMP enumeration confirmed ClamAV is running on the target:

```bash
snmp-check 192.168.57.42
```

```
3780  runnable  clamd              /usr/local/sbin/clamd
3784  runnable  clamav-milter      /usr/local/sbin/clamav-milter --black-hole-mode -l -o -q /var/run/clamav/clamav-milter.ctl
```

---

## Foothold

### Sendmail + ClamAV Milter RCE

The exploit: [Sendmail with clamav-milter < 0.91.2 - Remote Command Execution (EDB-4761)](https://www.exploit-db.com/exploits/4761)

![Exploit execution](/images/clamav/exploit.png)

The exploit sends a crafted email through Sendmail that triggers ClamAV's milter to execute arbitrary commands. Lands us directly as **root**.

---

## Privilege Escalation

Not required -- the exploit gives immediate root access.

---

## Proof

{{< terminal title="proof" >}}
local.txt: [redacted]
proof.txt: [redacted]
{{< /terminal >}}

![Root proof](/images/clamav/loot.png)

---

## Key Takeaways

{{< callout type="note" >}}
- Always enumerate **UDP ports** -- SNMP gave us process listing that confirmed ClamAV
- Pay attention to the box name -- it's often a hint toward the attack vector
- The binary text on the web page was a rabbit hole; don't tunnel-vision on the first interesting thing you find
- `searchsploit` results for one service can reference another (Sendmail results included clamav-milter)
{{< /callout >}}
