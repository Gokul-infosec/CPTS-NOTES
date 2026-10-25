# Attacking Common Services

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Attack the services found in almost every network (FTP, SMB, SQL, RDP, DNS, SMTP). (Service *enumeration* reference lives in `cheatsheets/service-enumeration.md`; this module is the *attacking* side.)

## Methodology

### SPPD attack model (Source → Process → Privileges → Destination)
A way to reason about any service/app attack as a linear flow:
- **Source** — where input enters (user input, config, API, libraries, code)
- **Process** — how the service handles it (parsing, variables, logging)
- **Privileges** — what access that processing runs with (user/group/policy)
- **Destination** — the outcome (local action, or network/transfer out)

**Log4Shell (CVE-2021-44228) mapped:** Source = crafted `User-Agent` with a JNDI lookup → Process = Log4j evaluates the lookup → Privileges = runs with the logging process's (often high) rights → Destination = fetches a malicious Java class from the attacker = RCE.

### Service misconfigurations (the recurring wins)
- Default creds (`admin:admin`), weak/blank passwords, anonymous auth
- Over-permissive RBAC/ACLs
- Unnecessary enabled features / open ports
- Verbose errors leaking info

**Prevention:** lock down infra, disable unused features/ports, strong passwords, regular scans/audits, a repeatable hardening baseline, patch management, network segmentation, security headers, automated config verification.

### Finding sensitive information (detective mindset)
Everything is loot: usernames, emails, passwords, DNS records, IPs, source code, config files, PII.
- **Where it hides:** file shares (FTP/SMB), email (SMTP), databases (SQL)
- **Process:** understand the service → know what's worth finding for *this* target → use every scrap of info you already have
- **Tools:** nmap, metasploit, sqlmap, `grep`/`find`, custom scripts

> Through-line of the module: every piece of info is valuable, misconfigs leak info and sometimes give RCE, and automation speeds the hunt. SPPD is the lens for *why* an attack works, not just *that* it does.

## Per-service attacks
(FTP, SMB, SQL, RDP, DNS, SMTP — filled as I send those sections.)

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
