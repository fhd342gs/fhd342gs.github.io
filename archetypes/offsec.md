---
title: "PG: {{ replace .File.ContentBaseName "-" " " | title }}"
date: {{ .Date }}
draft: true
tags:
  - offsec
  - proving-grounds
  - # difficulty: easy / intermediate / hard
  - # os: linux / windows
  - # techniques
categories:
  - walkthrough
cover: ""
---

{{< box-info name="{{ replace .File.ContentBaseName "-" " " | title }}" os="Linux" difficulty="Easy" ip="192.168.x.x" platform="OffSec Proving Grounds" >}}

---

## Recon

### Nmap

```bash
nmap -sC -sV -oN nmap/initial 192.168.x.x
```

{{< collapse title="Full nmap output" >}}
```
# paste full output here
```
{{< /collapse >}}

### Enumeration



---

## Foothold



---

## Privilege Escalation



---

## Proof

{{< terminal title="proof" >}}
local.txt:
proof.txt:
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- takeaway 1
- takeaway 2
{{< /callout >}}
