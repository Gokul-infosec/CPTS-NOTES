# Initial Enumeration Checklist

Reusable checklist for what to enumerate on a target. Core CPTS-relevant items are network + service + web + DNS; cloud / mobile / IoT / SCADA are here for completeness but are usually out of a standard network/web VAPT scope unless stated.

## Basic discovery
- [ ] Target IP(s) and hostname
- [ ] Host discovery (alive?) — ping sweep, ARP scan
- [ ] Traceroute / network mapping

## Port scanning
- [ ] TCP connect / SYN scans
- [ ] UDP scan (top ports at least)
- [ ] Full port scan (`-p-`)
- [ ] Version detection + default scripts

## Service enumeration
- [ ] Banner grabbing
- [ ] SMB, SNMP, SMTP, SSH, HTTP(S)
- [ ] Databases: MySQL, PostgreSQL, MSSQL, Oracle
- [ ] LDAP, RPC, RDP, VNC
- [ ] Network shares, NFS, TFTP
- [ ] (if present) printers, SCADA, hardware

## OS detection
- [ ] Nmap OS detection
- [ ] TTL analysis
- [ ] SMB OS discovery

## Vulnerability scanning
- [ ] Nmap NSE (`vuln`, `vulners`)
- [ ] Nikto (web)
- [ ] Nessus / OpenVAS (if available/authorized)
- [ ] Manual checks against identified versions

## Web enumeration
- [ ] Web server + tech detection (WhatWeb, Wappalyzer)
- [ ] Directory / content discovery
- [ ] CMS enum (WordPress, Joomla, Drupal)
- [ ] Source review: comments, JS, API keys
- [ ] robots.txt, sitemap.xml
- [ ] Headers / methods (Burp), CORS
- [ ] Input validation: SQLi, command injection, LFI/RFI
- [ ] Default credentials

## DNS enumeration
- [ ] Lookups, reverse DNS
- [ ] Zone transfer
- [ ] Subdomain enum / brute

## WHOIS & certificates
- [ ] WHOIS
- [ ] SSL/TLS cert inspection (names, SANs)

## External intelligence (OSINT)
- [ ] Shodan / Censys
- [ ] Code repositories (leaked secrets)

## Infrastructure
- [ ] Firewall / IDS-IPS / VPN / load balancer detection

## Containers & virtualization
- [ ] Docker / Kubernetes exposure

## Beyond standard network/web scope (only if in scope)
- [ ] Cloud storage: S3 / Azure Blob / GCS
- [ ] Mobile/IoT: APK/IPA, mobile API, Bluetooth, firmware

## Always
- [ ] Run automated recon only where allowed
- [ ] Keep notes: errors, hidden params, odd responses
