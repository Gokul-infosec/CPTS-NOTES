# Information Gathering - Web Edition

> Status: ✅ Complete
> Date started: 2026-10-26
> Date finished: 2026-10-26

## 1. Goal
Map a target's web footprint: domains, subdomains, IPs, tech, hidden content. **Passive** (no touch: WHOIS, CT logs, search, archives — stealthy) vs **active** (probe DNS/vhosts — louder).

## WHOIS & DNS
```bash
whois example.com                    # registrar, dates, contacts, nameservers
dig example.com A                    # IPv4 (AAAA=IPv6, CNAME=alias, MX=mail, NS=nameservers, TXT, SOA)
dig any example.com @<ns>
dig -x <ip> @<ns>                     # PTR / reverse lookup (nslookup -query=PTR <ip>)
nslookup -query=MX example.com       # nslookup equivalents: -query=A/PTR/ANY/TXT/MX
```
Tip: `export TARGET="example.com"` once, then reuse `$TARGET` across all commands.

## Subdomain enumeration
- **Passive (stealth):** Certificate Transparency logs —
  ```bash
  curl -s "https://crt.sh/?q=%25.example.com&output=json" | jq -r '.[].name_value' | sed 's/\*\.//g' | sort -u
  ```
  also subfinder/amass (passive mode), search engines, VirusTotal/Censys.
  ```bash
  cat sources.txt | while read s; do theHarvester -d "$TARGET" -b $s -f "$s-$TARGET"; done
  # sources.txt: crtsh, hackertarget, otx, rapiddns, sublist3r, urlscan, virustotal, zoomeye...
  ```
- **Active (brute):** `dnsenum example.com -f subdomains.txt`, `gobuster dns -d example.com -w wordlist`.

## Active infrastructure fingerprinting
```bash
curl -I "http://$TARGET"              # server headers
whatweb -a3 -v https://$TARGET        # aggressive tech ID (CMS, frameworks, versions)
wafw00f -v https://$TARGET            # is there a WAF, and which one?
cat subdomains.txt | aquatone -out ./aquatone -screenshot-timeout 1000   # screenshot every subdomain
waybackurls -dates https://$TARGET > waybackurls.txt                     # historical URLs w/ dates
```
Aquatone's screenshot gallery is the fastest way to triage a big subdomain list — spot logins, dev panels, odd apps at a glance.

## Zone transfer (AXFR) — misconfig = whole DNS zone
```bash
dig axfr example.com @<nameserver>    # if allowed → all subdomains/records
```

## Virtual hosts (multiple sites on one IP)
```bash
gobuster vhost -u http://<ip> -w hostnames.txt    # Host-header brute
ffuf -w hosts.txt:FUZZ -u http://<ip>/ -H 'Host: FUZZ.example.com' -fs <baseline>
```

## Crawling & content discovery
```bash
curl http://t/robots.txt              # off-limits dirs = interesting dirs
whatweb http://t ; curl -IL http://t  # tech/banner
# Scrapy / katana for deep crawl → extract links, JS, files
```

## Passive OSINT
- **Google dorks:** `site:` `inurl:` `filetype:` `intitle:"index of"` `cache:` `-exclude` `"exact"` — e.g. `site:example.com filetype:pdf "confidential"`, `intitle:"index of" /backup`.
- **Wayback Machine** (web.archive.org): old pages, removed dirs, leaked backups, content history.
- Shodan/Censys for exposed services; theHarvester for emails.

## Fix (for the report)
WHOIS privacy, restrict zone transfers to secondary servers, don't expose dev/staging subdomains, scrub metadata, review what's indexed.

> Start passive (crt.sh + wayback + dorks = free, stealthy, surprisingly complete), then go active (dnsenum, vhost brute, AXFR). CT logs and forgotten subdomains (dev/staging/old apps) are where the soft targets hide.

## My notes / lab output
Filled in as I run these on lab targets.

## Full command reference (source notes)
Complete command set — nothing trimmed. `export TARGET="domain.tld"` first, then reuse `$TARGET`.

**WHOIS & DNS**
| Command | Description |
|---|---|
| `whois $TARGET` | WHOIS lookup |
| `nslookup $TARGET` / `nslookup -query=A $TARGET` | A record |
| `dig $TARGET @<ns>` / `dig a $TARGET @<ns>` | A record (dig) |
| `nslookup -query=PTR <IP>` / `dig -x <IP> @<ns>` | PTR / reverse |
| `nslookup -query=ANY $TARGET` / `dig any $TARGET @<ns>` | ANY records |
| `nslookup -query=TXT $TARGET` / `dig txt $TARGET @<ns>` | TXT records |
| `nslookup -query=MX $TARGET` / `dig mx $TARGET @<ns>` | MX records |

**Passive subdomain enumeration**
| Resource/Command | Description |
|---|---|
| VirusTotal / Censys / Crt.sh | CT/scan web resources |
| `curl -s https://sonar.omnisint.io/subdomains/{domain} \| jq -r '.[]' \| sort -u` | All subdomains (omnisint) |
| `curl -s https://sonar.omnisint.io/tlds/{domain} \| jq -r '.[]' \| sort -u` | All TLDs for a domain |
| `curl -s https://sonar.omnisint.io/all/{domain} \| jq -r '.[]' \| sort -u` | All results across TLDs |
| `curl -s https://sonar.omnisint.io/reverse/{ip} \| jq -r '.[]' \| sort -u` | Reverse DNS on an IP |
| `curl -s https://sonar.omnisint.io/reverse/{ip}/{mask} \| jq -r '.[]' \| sort -u` | Reverse DNS on a CIDR |
| `curl -s "https://crt.sh/?q=${TARGET}&output=json" \| jq -r '.[] \| "\(.name_value)\n\(.common_name)"' \| sort -u` | Certificate Transparency |
| `cat sources.txt \| while read s; do theHarvester -d "${TARGET}" -b $s -f "${s}-${TARGET}"; done` | theHarvester over many sources |

sources.txt: `baidu, bufferoverun, crtsh, hackertarget, otx, projecdiscovery, rapiddns, sublist3r, threatcrowd, trello, urlscan, vhost, virustotal, zoomeye`

**Passive infrastructure ID**
| Resource/Command | Description |
|---|---|
| Netcraft (netcraft.com) / WayBackMachine (web.archive.org) | Infra history / archived pages |
| `waybackurls -dates https://$TARGET > waybackurls.txt` | Crawl archived URLs with dates |

**Active infrastructure ID**
| Command | Description |
|---|---|
| `curl -I "http://${TARGET}"` | Display HTTP response headers |
| `whatweb -a https://www.facebook.com -v` | Technology identification |
| Wappalyzer | Tech identification (browser) |
| `wafw00f -v https://$TARGET` | WAF fingerprinting |
| Aquatone | Screenshot tool for subdomains |
| `cat subdomain.list \| aquatone -out ./aquatone -screenshot-timeout 1000` | Screenshot all subdomains |

**Active subdomain enumeration**
| Command | Description |
|---|---|
| HackerTarget zone-transfer page / SecLists repo | Resources |
| `nslookup -type=any -query=AXFR $TARGET nameserver.target.domain` | Zone transfer (nslookup) |
| `gobuster dns -q -r "${NS}" -d "${TARGET}" -w "${WORDLIST}" -p ./patterns.txt -o "gobuster_${TARGET}.txt"` | Subdomain brute force |

**Virtual hosts**
| Command | Description |
|---|---|
| `curl -s http://192.168.10.10 -H "Host: randomtarget.com"` | Request a specific vhost |
| `cat ./vhosts.list \| while read vhost; do echo "\n****\nFUZZING: ${vhost}\n****"; curl -s -I http://<IP> -H "HOST: ${vhost}.target.domain" \| grep "Content-Length: "; done` | Brute-force vhosts (curl loop) |
| `ffuf -w ./vhosts -u http://<IP> -H "HOST: FUZZ.target.domain" -fs 612` | Brute-force vhosts (ffuf) |

**Crawling**
| Command | Description |
|---|---|
| ZAP (zaproxy.org) | Web proxy/crawler |
| `ffuf -recursion -recursion-depth 1 -u http://192.168.10.10/FUZZ -w /opt/useful/SecLists/Discovery/Web-Content/raft-small-directories-lowercase.txt` | Discover hidden files/folders |
| `ffuf -w ./folders.txt:FOLDERS,./wordlist.txt:WORDLIST,./extensions.txt:EXTENSIONS -u http://www.target.domain/FOLDERS/WORDLISTEXTENSIONS` | Mutated brute force |

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
