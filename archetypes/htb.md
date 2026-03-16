---
title: "HTB: {{ replace .File.ContentBaseName "-" " " | title }}"
date: {{ .Date }}
draft: true
tags:
  - htb
  - # difficulty: easy / medium / hard / insane
  - # os: linux / windows
  - # techniques used, e.g. sqli, privesc-suid, port-forwarding
categories:
  - walkthrough
cover: ""
---

{{< box-info name="{{ replace .File.ContentBaseName "-" " " | title }}" os="Linux" difficulty="Easy" ip="10.10.10.x" platform="HTB" >}}

---

## Recon

### Nmap

```bash
nmap -sC -sV -oN nmap/initial 10.10.10.x
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

## User



---

## Privilege Escalation



---

## Key Takeaways

{{< callout type="note" >}}
- takeaway 1
- takeaway 2
{{< /callout >}}
