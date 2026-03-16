---
title: "UA_MFA_bypass"
date: 2026-03-16
draft: false
tags:
  - azure
  - entra-id
  - mfa
  - conditional-access
  - bypass
  - user-agent
  - ropc
categories:
  - tools-and-techniques
description: "Test whether Conditional Access policies can be bypassed via User-Agent spoofing through ROPC authentication flow."
---

## The technique

Orgs love enforcing MFA via Conditional Access policies. Makes the compliance people happy, box gets checked, everyone goes home. Except then someone realizes their meeting room displays can't handle modern auth prompts. And the office printers need to talk to SharePoint. And legacy Outlook 2013 clients are still on half the floor.

So they add exclusions. Lots of exclusions:

- Legacy mail clients (Outlook 2010, 2013)
- Meeting room devices (Teams Rooms, Surface Hub)
- IoT devices, printers, healthcare monitors
- Gaming consoles (yes, a PlayStation can authenticate to Entra ID)

Here's the thing — when you hit the ROPC (Resource Owner Password Credential) flow, you authenticate with just username + password. No browser, no redirect, no MFA prompt. And the only thing telling Azure what "device" is connecting is the **User-Agent header**.

Spoof a Teams Room User-Agent. Hit ROPC. Skip MFA entirely.

Spoiler: meeting room devices don't do MFA. Neither does a PlayStation. Ask me how I know.

---

## What the script does

PowerShell script that automates testing every interesting combination:

- **Multiple Azure resources** — ARM, Microsoft Graph, Key Vault, Storage, Azure DevOps, and more
- **35+ User-Agent profiles** across categories:
  - Gaming consoles (Xbox, PlayStation, Nintendo Switch)
  - Meeting room devices (Teams Rooms, Surface Hub, Cisco/Poly endpoints)
  - Legacy Outlook (2010, 2013, 2016 pre-modern-auth)
  - Printers and embedded systems
  - Healthcare devices
- **Reports which combinations succeed** — resource + User-Agent pairs that return tokens
- **Decodes returned tokens** — shows you exactly what access you got

---

## Usage

Test Graph API access with a Teams Room user-agent:

```powershell
.\UA_MFA_bypass.ps1 -Resource Graph -UserAgent TeamsRoom -Verbose
```

Try Key Vault with legacy Outlook:

```powershell
.\UA_MFA_bypass.ps1 -Resource KeyVault -UserAgent Outlook2013
```

Shotgun approach — test all user-agents against a resource:

```powershell
.\UA_MFA_bypass.ps1 -Resource Graph -UserAgent All
```

The script will prompt for credentials (or you can pass them as parameters if you're scripting it).

---

## After a successful bypass

When the script finds a working combination, the token gets stored:

```powershell
# Token is in $MFApwn
$MFApwn.access_token     # JWT — use directly with API calls
$MFApwn.refresh_token    # For token renewal without re-auth

# Example: hit Graph API with the stolen token
$headers = @{ Authorization = "Bearer $($MFApwn.access_token)" }
Invoke-RestMethod -Uri "https://graph.microsoft.com/v1.0/me" -Headers $headers
```

At this point you have a valid token for whatever resource you targeted, and MFA never happened.

---

## Why this matters for defenders

If you're on the blue side, here's the fix list:

1. **Audit your CA policy exclusions** — right now, today. You probably have more than you think.
2. **Block legacy authentication entirely** — if nothing in your environment actually needs it, kill it. Microsoft has been begging you to do this since 2020.
3. **Use device compliance instead of (or in addition to) MFA** — "Require device to be marked as compliant" is harder to spoof than a User-Agent string.
4. **Monitor for ROPC flows** — in Entra sign-in logs, filter for `Authentication Protocol: ROPC`. If you see it and didn't expect it, someone's testing you. Or worse.
5. **Named locations aren't enough** — combining CA exclusions with "trusted locations" doesn't fix this if the attacker is on your VPN or inside the network.

{{< callout type="info" >}}
There's also a Python version (`UA_MFA_bypass.py`) in the repo. It's a work in progress — functional but rough around the edges. Use the PowerShell version for now.
{{< /callout >}}

---

## Source

Code is on GitHub: [fhd342gs/AZURE_security_scripts](https://github.com/fhd342gs/AZURE_security_scripts)

{{< callout type="warning" >}}
For authorized security testing only. This is a penetration testing tool, not a toy. Get written permission before you run it against anything you don't own.
{{< /callout >}}
