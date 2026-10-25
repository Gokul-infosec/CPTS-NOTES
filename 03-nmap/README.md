# Network Enumeration with Nmap

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Use Nmap to discover live hosts, open ports, services, versions and the OS, and to run NSE scripts for deeper enumeration and known-CVE checks.

## 2. Nmap flag reference

### Host discovery
| Flag | Example | Purpose |
|---|---|---|
| `-sL` | `nmap 192.168.1.1-3 -sL` | List targets only, no scan |
| `-sn` | `nmap 192.168.1.0/24 -sn` | Ping sweep, no port scan |
| `-Pn` | `nmap 192.168.1.1-5 -Pn` | Skip host discovery, port scan only |
| `-PS` | `-PS22-25,80` | TCP SYN discovery on listed ports |
| `-PA` | `-PA22-25,80` | TCP ACK discovery |
| `-PU` | `-PU53` | UDP discovery |
| `-PR` | `nmap 192.168.1.0/24 -PR` | ARP discovery (local network) |
| `-n` | `nmap 192.168.1.1 -n` | No DNS resolution |

### Target specification
| Form | Meaning |
|---|---|
| `192.168.1.1` | Single IP |
| `192.168.1.1 192.168.2.1` | Specific IPs |
| `192.168.1.1-254` | Range |
| `192.168.1.0/24` | CIDR |
| `-iL targets.txt` | Targets from file |
| `-iR 100` | 100 random hosts |
| `--exclude 192.168.1.1` | Exclude hosts |

### Scan techniques
| Flag | Scan |
|---|---|
| `-sS` | TCP SYN (default, needs root) |
| `-sT` | TCP connect |
| `-sU` | UDP |
| `-sA` | TCP ACK (firewall mapping) |
| `-sW` | TCP Window |
| `-sM` | TCP Maimon |

### Port specification
| Flag | Meaning |
|---|---|
| `-p 21` | Single port |
| `-p 21-100` | Range |
| `-p U:53,T:21-25,80` | Mixed TCP/UDP |
| `-p-` | All 65535 ports |
| `-p http,https` | By service name |
| `-F` | Fast (top 100) |
| `--top-ports 2000` | Top N ports |

### Timing & performance
| Flag | Meaning |
|---|---|
| `-T0`–`-T5` | Paranoid → Insane (T3 default; T0/T1 for IDS evasion, T4 typical for labs) |
| `--host-timeout <t>` | Max time per host (e.g. 30m) |
| `--min-rtt-timeout` / `--max-rtt-timeout` | Probe timeout bounds |
| `--scan-delay` / `--max-scan-delay` | Delay between probes |
| `--max-retries <n>` | Probe retransmissions per port |
| `--min-rate` / `--max-rate` | Packets per second bounds |

### Service / version & OS detection
| Flag | Meaning |
|---|---|
| `-sV` | Service version detection |
| `--version-intensity <0-9>` / `--version-light` / `--version-all` | Detection depth |
| `-O` | OS detection |
| `--osscan-guess` | Guess aggressively |
| `-A` | OS + version + default scripts + traceroute |

### NSE scripts
| Flag | Meaning |
|---|---|
| `-sC` / `--script default` | Default script set |
| `--script=<name>` | Specific script |
| `--script=http*` | Wildcard |
| `--script=http,banner` | Multiple |
| `--script "not intrusive"` | Exclude intrusive |
| `--script-args <args>` | Script arguments |
| | Scripts live in `/usr/share/nmap/scripts/` |

### Firewall / IDS evasion
`-f` fragment · `--mtu <val>` · `-D` decoys · `-S` spoof source IP · `-g` source port · `--proxies` · `--data-length <bytes>`

### Output
| Flag | Meaning |
|---|---|
| `-oN <file>` | Normal |
| `-oX <file>` | XML |
| `-oG <file>` | Grepable |
| `-oA <prefix>` | All three formats |
| `--append-output` | Append |
| `-oN -` | To screen |

## 3. Tips
- Default scan = top 1000 TCP only; always do a `-p-` pass, then `-sV -sC` on the open ports.
- `filtered` usually = firewall; `closed` still responds. Use `--reason` to see why a port got its state.
- Save everything with `-oA` so you can grep/parse later (`grep open`, `xsltproc`, `nmaptocsv`).
- `--script vuln` or `-sV --script vulners` for known-CVE checks against detected versions.

## 3b. Practical scan recipes

```text
nmap -sn <ip>                                   # host discovery only, no port scan
nmap -sn -PS <ip>                               # SYN ping sweep
nmap -sn -PA <ip>                               # ACK ping (hosts behind ICMP-blocking FW)
nmap -T4 -sS -p- <ip>                           # full SYN scan, aggressive timing
nmap -sC -sV --script=<name> -p- -T4 <ip>       # scripts + version on all ports
nmap -Pn -sA -p- <ip>                           # ACK scan: map filtered vs unfiltered (firewall)
nmap -Pn -sS -sV -p- --data-length 200 -D <gw>,<gw> <ip>   # decoy scan + packet padding
nmap -Pn -n -sS -p- -T4 <ip>                    # stealth SYN, all ports, no DNS
```

### Extra flags worth knowing
- **Host/network discovery:** `-PE` ICMP echo ping · `--disable-arp-ping`
- **Packet handling:** `--packet-trace` (show every packet) · `--reason` (why a port got its state)
- **Stealth/spoofing:** `-D RND:5` (5 random decoys) · `-S <IP>` (source IP) · `-e tun0` (send via interface) · `--source-port 53` (fixed source port, e.g. to look like DNS)
- **Timing:** `--host-timeout 5s` · `--scan-delay 5s` · `--initial-rtt-timeout 50ms` · `--max-rtt-timeout 100ms`
- **Output:** `-oN` normal · `-oX` XML · `-oG` grepable · `-oA <base>` all three

> Note: faster scans (`-T4/-T5`, high `--min-rate`) finish quicker but are louder to an IDS; slow scans (`-T0/-T1`, `--scan-delay`) are for evasion. Decoy/spoof/source-port tricks are evasion techniques — scope + approval only.

## 4. Commands I ran (lab)
Filled in once I run these on Pwnbox, with real IPs as `<TARGET_IP>`.

```bash
# [ATTACK BOX - Pwnbox]

```

## 5. Open questions / revisit
- 

## 6. Cheat sheet lines
Verified commands copied into `cheatsheets/enumeration.md`.
