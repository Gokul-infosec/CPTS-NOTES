# My Pentest Methodology

My own workflow, built up module by module. Updated whenever a module changes how I approach a target.

## 1. Pre-engagement
- Scope, rules of engagement, what's out of bounds
- Contract/NDA, authorization letter, emergency contacts, test window & allowed source IPs
- Goals: compromise level wanted (foothold / DA / specific data), evasive or not, prod vs staging
- Set up Sysreptor project + a findings log from the start — log as you go, not at the end

## 2. Information gathering
Do all four, every time:
- OSINT: company, employees, leaked secrets (code repos, forums), tech stack from job posts
- Infrastructure: DNS, name / mail / web servers, cloud assets → host + IP list, checked against scope; spot WAFs / firewalls
- Services: name, version, what it leaks, why it exists
- Hosts: OS, ports, services, role, what it talks to
- After a foothold: pillage the host (files, creds, configs, users, local services)

Web recon tooling:
- Passive: `whois`, DNS (`dig`, `nslookup`), crt.sh / cert transparency, Wayback, Google dorks, Shodan
- DNS: zone transfer `dig axfr @ns domain`; subdomain brute `gobuster dns` / passive `subfinder`
- Active: `curl -I` headers, robots.txt, Wappalyzer (stack), vhost fuzzing, EyeWitness screenshots

## 3. Enumeration
Port/service scan first, then enumerate each service in depth.
- Host discovery: `nmap -sn <range>` (ping sweep); skip with `-Pn` if ICMP blocked
- Port scan: `nmap -p- --min-rate 10000 <ip>` → then `nmap -sCV -p<ports> <ip>` (version + default scripts)
- UDP where it matters: `nmap -sU --top-ports 100`
- Output everything: `-oA scans/<host>` (reuse later, feed to tools)
- Per-service deep enum (see Footprinting + Common Services):
  - FTP (21): `ftp`/`nc`, anon login, `ncftp`, nmap ftp-* scripts
  - SMB (139/445): `smbclient -L`, `smbmap -H`, `enum4linux-ng`, `crackmapexec smb`, RID brute, null session
  - NFS (2049): `showmount -e`, mount, check squashing
  - DNS (53): `dig axfr`, subdomain enum
  - SMTP (25): `VRFY`/`EXPN`/`RCPT`, `smtp-user-enum`, open-relay test with `swaks`
  - SNMP (161): `snmpwalk -v2c -c public`, `onesixtyone`, braa; grab community strings
  - MSSQL/MySQL (1433/3306): `mssqlclient.py`, `mysql -h`, default creds, version
  - RDP/WinRM (3389/5985): `rdp-sec-check`, `evil-winrm`
- Web: directory/vhost brute with `ffuf` / `gobuster`, parameter fuzzing, tech fingerprint, review source & JS
- Always note version strings → map to CVEs in the next phase

## 4. Vulnerability identification
- Map each enumerated service+version to known CVEs (searchsploit, vendor advisories, CVE/NVD)
- Authenticated scan where in scope: Nessus / OpenVAS (credentialed = fewer false positives); export and triage
- Score and prioritise: CVSS for base severity, then real exploitability in this environment (DREAD/context)
- Keep the three buckets separate: **confirmed** (verified reachable/exploitable) vs **hypothesis** (plausible, unverified) vs **noise** (false positive)
- Web: test for the OWASP-class bugs against discovered params — SQLi, XSS, LFI/RFI, file upload, command injection, IDOR, SSRF, XXE, auth/brute
- Default creds & weak config always worth a pass before building a custom exploit

## 5. Exploitation
Pick the lowest-risk path to a shell; prefer reliable over flashy.
- Web → RCE:
  - SQLi: manual (UNION, error, blind boolean/time) → `sqlmap` to automate/dump; stacked queries → `xp_cmdshell` (MSSQL) / `INTO OUTFILE` webshell (MySQL)
  - File upload: bypass validation (client-side, blacklist alt-ext, whitelist double/reverse ext + char injection, Content-Type, magic bytes) → web/reverse shell
  - File inclusion: LFI → log poisoning / PHP wrappers / `data://` → RCE; RFI if remote allowed
  - Command injection: inject with `;` `|` `&&` `$()` ``` `` ```; bypass filters (encoding, `${IFS}`, case, wildcards)
  - XSS: steal session/cookie, CSRF, phish; chain to admin
- Service exploit: searchsploit PoC (read it before running), or Metasploit module (`msfconsole`, set RHOSTS/LHOST, check first)
- Password attacks for access: `hydra`/`crowbar` spray, hash cracking `hashcat`/`john`, keep a creds list
- Payloads/shells: `msfvenom` (staged vs stageless, match arch/OS), catch with `nc -lvnp` or multi/handler; upgrade to full TTY (`python3 -c pty`, `stty raw -echo`)
- File transfers to/from target: HTTP (`python3 -m http.server` + `wget`/`curl`), SMB (`impacket-smbserver`), base64 paste, `certutil`/PowerShell on Windows, living-off-the-land
- Validate every exploit (don't trust scanner), note exact command + output for the report

## 6. Post-exploitation & privilege escalation
Enumerate as the current user before escalating.
- Linux (see Linux Privesc): `whoami`/`id`, sudo `-l`, SUID/SGID (`find / -perm -4000`), cron, capabilities, writable paths, kernel version, `.bash_history`, configs & creds, GTFOBins for any allowed binary; tools: LinPEAS, pspy
- Windows (see Windows Privesc): `whoami /priv` (SeImpersonate→Potato, SeBackup, etc.), unquoted service paths, weak service perms, AlwaysInstallElevated, scheduled tasks, stored creds (`cmdkey`, registry, SAM), UAC bypass; tools: WinPEAS, Seatbelt, PowerUp
- Pillage every host: creds, config files, SSH keys, DB connection strings, password managers, tokens
- Harvest for reuse: dump hashes, Kerberos tickets, browser/app creds → feed back into lateral movement
- Establish stable access / persistence only if in scope and authorised

## 7. Lateral movement & pivoting
- Reuse creds/hashes: Pass-the-Hash, Pass-the-Ticket, `psexec.py`/`wmiexec.py`/`smbexec.py`/`evil-winrm`, CrackMapExec spray across subnet
- Pivoting (reach internal segments from the foothold):
  - SSH tunnels: local `-L`, remote `-R`, dynamic `-D` + proxychains
  - `ligolo-ng` (preferred — clean TUN interface), or chisel, sshuttle
  - Metasploit `autoroute` + `socks_proxy`
- Port-forward single services when a full tunnel isn't needed
- Active Directory (see AD module):
  - Enum: BloodHound/SharpHound, `ldapsearch`, `rpcclient`, Kerbrute user enum
  - Attacks: Kerberoasting, AS-REP roasting, password spray, ACL abuse, delegation (unconstrained/constrained/RBCD), DCSync, Golden/Silver tickets
  - Tools: impacket suite, Rubeus, Mimikatz, NetExec
- Re-run phases 3–6 from each new host/segment reached; keep the network/host map updated

## 8. Reporting
Deliverable is the product — write continuously, not at the end.
- Capture per finding as you go: title, affected asset, severity (CVSS vector), description, step-by-step PoC, screenshots, impact, remediation, references
- Keep raw evidence: scan outputs, request/response, command logs, timestamps
- Structure: executive summary (business-level, risk posture) → findings (ranked by severity) → technical detail/PoC → remediation roadmap → appendices (scope, methodology, tool list)
- Tie each finding to a concrete, actionable fix; re-test / note retest status
- Tooling: Sysreptor for the report; templates/ in this repo for finding boilerplate
- Deliver securely; offer a debrief; track remediation for the retest
