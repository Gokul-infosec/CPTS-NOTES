# Penetration Testing Process

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
The big picture of a pentest: the phases, how every later module fits into them, and the legal/ethical rules that apply to all of it. Theory module, worth re-reading as the path goes on.

### Section notes

#### Introduction to the Penetration Tester Path
- The path covers three assessment types: external pentests, internal pentests (network and Active Directory), and web app assessments.
- It's built around one simulated client company. It ends with a reporting/soft-skills module and a full mock pentest of a large network.
- Modules are meant to be done in order; topics like lateral movement and pillaging don't get their own module but come back again and again.
- The goal is my own repeatable methodology that works on any target size or industry, not just knowing tools. Understand *why* a flaw exists so I can explain why a tool fails and give real remediation advice.

**Legal and ethical rules (the part that matters on a real job):**
- Never scan or interact with an organization's systems without a signed Scope of Work: detailed scope, contract, and rules of engagement, signed by both sides.
- Passive OSINT with public sources is fine. Active probing without consent is illegal.
- Scope is exact: URLs, IPs, CIDR ranges, SSIDs, physical sites, or email/phone lists for social engineering. Stay inside it.
- Found an interesting host that's not in scope? Ask the client, and get the answer in writing, not on a call.
- Do no harm: before running a tool or PoC, ask whether it could crash something. If unsure, get written approval from my manager and the client first.
- Before joining a pentest company, check it's legitimate and only tests with signed paperwork. Taking part in illegal testing makes me liable even with good intentions.
- Document everything and over-communicate.
- Legal practice outside HTB: bug bounty programs (each has its own scope and rules, and many ban automated scanning).

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
