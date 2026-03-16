---
title: "HTB: {{ replace .File.ContentBaseName "-" " " | title }}"
date: {{ .Date }}
draft: true
os: ""
difficulty: ""
platform: "HTB"
tags:
  - htb
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
