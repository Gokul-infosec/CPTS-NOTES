# Enumeration (Nmap, services, footprinting) Cheat Sheet

Flat copy-paste reference. Explanations in `03-nmap/` + `04-footprinting/`; per-service depth in `service-enumeration.md` and `nse-scripts-by-service.md`.
Placeholders: `<TARGET_IP>`, `<LHOST>`, `<USER>`, `<PASS>`, `<DOMAIN>`, `<NS>`.

## Host discovery
```bash
nmap -sn <range>                        # ping sweep, no port scan
nmap -sn -PS <range>                    # TCP SYN ping
nmap -sn -PA <range>                    # TCP ACK ping (hosts behind ICMP-blocking FW)
nmap 10.10.10.0/24 -PR                  # ARP discovery (local segment)
nmap -sn <range> --dns-server <NS>      # resolve via a specific (internal) name server
fping -asgq 10.10.10.0/24               # fast ICMP sweep
arp-scan -l                             # local ARP sweep
```

## Port / service / version / OS
```bash
nmap -p- --min-rate 10000 -T4 <TARGET_IP>          # all 65535 TCP ports, fast
nmap -sC -sV -p<ports> -oA scans/host <TARGET_IP>  # default scripts + versions on open ports
nmap -sU --top-ports 100 <TARGET_IP>               # top UDP ports
nmap -Pn -sS -p- <TARGET_IP>                        # skip discovery, stealth SYN
nmap -O -A <TARGET_IP>                              # OS + version + default scripts + traceroute
nmap -p- --reason <TARGET_IP>                       # why each port got its state
```
Output: `-oN` normal · `-oX` XML · `-oG` grepable · `-oA <base>` all three. Default scan = top 1000 TCP only → always do a `-p-` pass, then `-sCV`.

## NSE
```bash
nmap --script=default,safe <TARGET_IP>
nmap --script "not intrusive" <TARGET_IP>
nmap --script vuln <TARGET_IP>                      # known-CVE checks
nmap -sV --script vulners <TARGET_IP>
nmap -p445 --script=smb-vuln* <TARGET_IP>
nmap -p80,443 --script=http-headers,http-title,http-methods,http-enum <TARGET_IP>
```
Scripts live in `/usr/share/nmap/scripts/`. `brute` / credential scripts are intrusive — scope + approval only.

## Firewall / IDS evasion (scope + approval only)
```bash
nmap -Pn -sA -p- <TARGET_IP>                        # ACK scan: map filtered vs unfiltered
nmap -f --mtu 24 <TARGET_IP>                        # fragment packets
nmap -D RND:5 <TARGET_IP>                            # 5 random decoys
nmap -S <spoofed_ip> -e tun0 <TARGET_IP>            # spoof source IP
nmap --source-port 53 <TARGET_IP>                    # look like DNS
nmap -T0 / -T1 + --scan-delay                        # slow = quieter to IDS
```

## DNS footprinting
```bash
dig axfr @<NS> <DOMAIN>                              # zone transfer
dig A/MX/TXT/NS <DOMAIN>
subfinder -d <DOMAIN> -v                             # passive subdomain enum
gobuster dns -d <DOMAIN> -w subdomains.txt           # brute subdomains
```

## Web recon
```bash
whatweb http://<TARGET_IP>
curl -I http://<TARGET_IP>                           # headers
# robots.txt, sitemap.xml, Wappalyzer (stack), EyeWitness (screenshots), crt.sh, Wayback, Shodan
```

> Flow: host discovery → full TCP `-p-` → `-sCV` on open ports → per-service deep enum (see `service-enumeration.md`) → map versions to CVEs. Save everything with `-oA` to grep/parse later.
