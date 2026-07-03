---
title: "PG: Internal"
date: 2023-03-07
draft: false
os: "Windows"
difficulty: "Easy"
platform: "OffSec Proving Grounds"
aliases: ["/posts/offsec/internal/"]
tags:
  - offsec
  - proving-grounds
  - easy
  - windows
  - smb
  - ms09-050
  - metasploit
categories:
  - walkthrough
---

{{< box-info name="Internal" os="Windows" difficulty="Easy" ip="192.168.X.X" platform="OffSec Proving Grounds" >}}

---

## Enumeration

An old Windows host exposing SMB. Post-exploitation `sysinfo` confirms the target: **Windows Server 2008 (6.0 Build 6001, SP1), x86, workgroup member.** That 32-bit architecture matters — it kills one of the two rabbit holes below.

### Rabbit holes

This box is genuinely easy but *sneaky* — it dangles two distractions designed to burn your time:

- **MS17-010 (EternalBlue):** guest SMB access is allowed, but there are no accessible shares and the vuln check doesn't pan out.
- **BlueKeep (CVE-2019-0708):** looks plausible until you remember the host is **32-bit** — the reliable public RCE chains target x64. Dead end.

---

## Foothold

### MS09-050 — SMB2 Negotiate RCE

For an unauthenticated SMB target this old, MS09-050 (the SMBv2 `NEGOTIATE` command validation flaw) is the play. Metasploit has it:

```
search type:exploit platform:windows target:2008 smb
use exploit/windows/smb/ms09_050_smb2_negotiate_func_index
set RHOSTS <target>
run
```

The exploit returns a Meterpreter session running as **NT AUTHORITY\SYSTEM** directly — there's no separate privilege-escalation step on this box.

---

## Proof

![getuid returning NT AUTHORITY\SYSTEM, sysinfo, and the proof flag](/images/internal/proof.png)

---

## Key Takeaways

{{< callout type="note" >}}
- Fingerprint architecture early — the 32-bit detail here rules out BlueKeep before you waste an hour on it.
- Guest SMB access with no readable shares is a classic distraction. Don't tunnel-vision on MS17-010 just because port 445 is open.
- For legacy Windows Server 2008 SMB, MS09-050 lands a SYSTEM shell in one shot when the newer named vulns don't apply.
{{< /callout >}}
