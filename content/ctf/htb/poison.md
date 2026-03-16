---
title: "HTB: Poison"
date: 2023-12-21
draft: false
os: "FreeBSD"
difficulty: "Medium"
platform: "HTB"
aliases: ["/posts/htb/poison/"]
tags:
  - htb
  - medium
  - freebsd
  - lfi
  - vnc
  - ssh-tunneling
  - base64
categories:
  - walkthrough
---

{{< box-info name="Poison" os="FreeBSD" difficulty="Medium" ip="10.129.1.254" platform="HTB" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2 (FreeBSD 20161230; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((FreeBSD) PHP/5.6.32)
```
{{< /collapse >}}

**OS**: FreeBSD

### Port 80 - Web App

A minimal PHP test site with a form that accepts a filename and includes it via `browse.php?file=`:

```
Sites to be tested: ini.php, info.php, listfiles.php, phpinfo.php
```

Directory fuzzing finds: `index.php`, `info.php`, `phpinfo.php`.

Supplying `listfiles.php` to the form reveals an interesting file:

```
[8] => pwdbackup.txt
```

---

## Foothold

### LFI

The `browse.php?file=` parameter is a straight LFI:

```bash
curl -sS 'http://10.129.1.254/browse.php?file=../../../../../../../etc/passwd'
```

```
root:*:0:0:Charlie &:/root:/bin/csh
charix:*:1001:1001:charix:/home/charix:/bin/csh
```

### Encoded Password

Reading `pwdbackup.txt` through the LFI reveals a base64 blob with a note: "This password is secure, it's encoded atleast 13 times.. what could go wrong really.."

Decode it 13 times (a simple loop in any language will do) and get the password: `Charix!2#4%6&8(0`.

Combined with the username from `/etc/passwd`: `charix:Charix!2#4%6&8(0`. SSH in as `charix`.

---

## Privilege Escalation

### VNC via SSH Tunnel

In the home directory there's a `secret.zip` file. Extract it using `charix`'s password -- it contains a binary file called `secret`.

Running `sockstat` (FreeBSD's netstat equivalent) reveals a VNC server running as root on `localhost:5901`.

Set up an SSH tunnel:

```bash
ssh -N -L 127.0.0.1:5901:127.0.0.1:5901 charix@10.129.1.254
```

Connect to the VNC server using the `secret` file as the password file:

```bash
vncviewer 127.0.0.1:5901 -passwd secret
```

The `secret` file isn't a password -- it's a VNC authentication key file. The VNC session opens as root.

---

## Proof

{{< terminal title="root@poison" >}}
user.txt: [redacted]
root.txt: [redacted]
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- LFI is always worth testing when a filename parameter exists -- `browse.php?file=` is textbook
- "Security through obscurity" (13x base64 encoding) is no security at all
- Always check for local services with `sockstat`/`netstat` -- VNC running as root on localhost is a common privesc path
- VNC `secret` files act as authentication keys, not plaintext passwords
{{< /callout >}}
