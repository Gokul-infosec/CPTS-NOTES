# Information Gathering - Web Edition

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Map a target's web footprint: domains, subdomains, IPs, tech, hidden content. **Passive** (no touch: WHOIS, CT logs, search, archives — stealthy) vs **active** (probe DNS/vhosts — louder).

## WHOIS & DNS
```bash
whois example.com                    # registrar, dates, contacts, nameservers
dig example.com A                    # IPv4 (AAAA=IPv6, CNAME=alias, MX=mail, NS=nameservers, TXT, SOA)
dig any example.com @<ns>
```

## Subdomain enumeration
- **Passive (stealth):** Certificate Transparency logs —
  ```bash
  curl -s "https://crt.sh/?q=%25.example.com&output=json" | jq -r '.[].name_value' | sed 's/\*\.//g' | sort -u
  ```
  also subfinder/amass (passive mode), search engines.
- **Active (brute):** `dnsenum example.com -f subdomains.txt`, `gobuster dns -d example.com -w wordlist`.

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
