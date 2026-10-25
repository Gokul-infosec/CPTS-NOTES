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

#### Academy Modules Layout
- Pentesting is like learning guitar: knowing the theory means nothing without hands-on reps. Analytical skill ("question everything") comes from practice, not one module.
- You can't attack what you don't understand. Deep knowledge of Linux, Windows, networking, web and AD makes the job easier.

**The 8 stages and where each one can lead (it's a loop, not a straight line):**

| Stage | What happens | Where I go next |
|---|---|---|
| Pre-Engagement | Scope, limits, contracts, agreements in writing | Information Gathering (only option) |
| Information Gathering | Find the targets, often starting from just a domain or IP range. Detailed notes. | Vulnerability Assessment |
| Vulnerability Assessment | Automated scans for known vulns, plus manual analysis connecting the dots | Exploitation (no access yet) · Post-Exploitation (on a box) · Lateral Movement (on a box) · back to Info Gathering (not enough info) |
| Exploitation | Attack based on what I found. Same app, different configs per company. | Info Gathering on the new host · Post-Exploitation · Lateral Movement (e.g. dual-homed host) · PoC (e.g. Domain Admin) |
| Post-Exploitation | Pillage the box first (mandatory), then escalate privileges | Exploitation of local services · Lateral Movement · PoC |
| Lateral Movement | Use access to reach other hosts or subnets. Admin not always needed. | Vulnerability Assessment · Pillaging again · PoC |
| Proof-of-Concept | Proof the vuln exists so admins can reproduce it before changing business-critical systems. Can be automated (e.g. Python). | Post-Engagement (only option) |
| Post-Engagement | Clean up everything I uploaded (e.g. no open bind shells left behind). Log every change, exploit, captured cred and uploaded file in the report appendix so the client can match alerts to my testing. Reconcile notes and deliver the report. | End |

**My takeaways:**
- Biggest beginner mistake: jumping straight to exploitation. Thorough enumeration first saves time on a time-boxed test.
- When stuck, go back a stage. Usually it's an information gathering gap.
- Fixes take time and money because of system dependencies, so a clear, reproducible PoC is what gets a finding fixed.
- Cleanup and documenting my own actions are part of the job, not optional extras.

#### Penetration Testing Overview
- **Pentest:** an authorized, targeted attack using real attacker techniques to measure the impact of vulns (or chains of vulns) on confidentiality, integrity and availability. Goal: find *all* vulns in scope.
- **Red team vs pentest:** a red team is scenario-based and only uses what it needs to hit one objective (e.g. read the CEO's inbox). A pentest tries to cover everything.
- **Vulnerability assessment vs pentest:** a VA is purely automated scanning (Nessus, Qualys, OpenVAS) against known issues and can't adapt to the target's config. A pentest is manual info gathering plus automated and manual testing, tailored to the target.
- **Risk management:** companies can accept, transfer (insurance, contracts), avoid or mitigate risk. Some inherent risk always remains, even with good controls.
- **My role:** report vulns, reproduction steps and remediation advice. I don't patch or change code; fixing is the client's job.
- **A pentest is a point-in-time snapshot, not monitoring.** The report should say so.
- **Authorization:** written approval is mandatory, and a client can only authorize testing of assets *they own*. Third-party hosting may need its own written approval (AWS allows some services without prior approval, but it varies by provider), so confirm asset ownership during scoping.
- **Sensitive data found during testing** (PII, salaries, card data) stays private. Recommend changing creds and encrypting the data.
- Employees usually aren't told about the test, though managers may choose to.

**Perspective:**

| | Where I test from | Notes |
|---|---|---|
| External | Internet, as an anonymous attacker (own host over VPN, or a VPS) | Client may want noisy, stealthy, or "hybrid" (gradually noisier to test detection). Goal: compromise external hosts, get sensitive data, or reach the internal network. |
| Internal | Inside the corporate network | After an external breach or from an assumed-breach start. Air-gapped systems may need me on site. |

**Type (how much info I get):**

| Type | Info given |
|---|---|
| Blackbox | Minimal: just IPs / domains. Slowest, lots of recon. |
| Greybox | Extended: specific URLs, hostnames, subnets |
| Whitebox | Everything: configs, admin creds, source code |
| Red teaming | Can add physical testing and social engineering. Combines with any type above. |
| Purple teaming | Works closely with the defenders. Combines with any type above. |

**What can be tested:** network, web apps, mobile, APIs, thick clients, IoT, cloud, source code, physical security, employees, hosts, servers, security policies, firewalls, IDS/IPS. Often mixed in one engagement.

**My takeaways (interview questions straight from this section):**
- Pentest vs vulnerability assessment vs red team: know the difference cold.
- Black / grey / white box and internal vs external.
- Always confirm the client actually owns what's in scope, especially cloud and third-party hosted assets.

#### Laws and Regulations
Every country has its own laws on unauthorized access, data protection, interception, health data and children's data. Breaking them can mean civil *and* criminal penalties, even during "research".

**Key laws by region (the ones worth remembering):**

| Region | Unauthorized access / hacking | Personal data | Other |
|---|---|---|---|
| USA | CFAA (criticized as broad and vague) | — | DMCA (bypassing copy protection), ECPA (intercepting comms), HIPAA (health data), COPPA (children under 13) |
| EU | NIS Directive (NIS2) | GDPR: fines up to 4% of global annual revenue or €20M, whichever is higher. Applies to *any* company processing EU citizens' data, wherever it's based. | E-Privacy Directive, Council of Europe Cybercrime Convention (first international cybercrime treaty) |
| UK | Computer Misuse Act 1990 | Data Protection Act 2018 | RIPA 2000, IPA 2016 (covert/investigatory powers), Human Rights Act 1998 |
| India | IT Act 2000 | Digital Personal Data Protection (DPDP) Act | Evidence and criminal law used in cybercrime cases |
| China | National Security Law, Anti-Terrorism Law | Cyber Security Law, cross-border data transfer rules | Critical infrastructure protection regulation |

**India: what applies to me (updated beyond the module):**
- **IT Act 2000** is the main one. Section 43 covers unauthorized access (civil penalty) and Section 66 makes it a criminal offence when done dishonestly or fraudulently.
- **DPDP Act** was listed as "proposed" in the module, but it was enacted in 2023.
- The **Indian Penal Code 1860** and **Indian Evidence Act 1872** named in the module were replaced in July 2024 by the **Bharatiya Nyaya Sanhita (BNS) 2023** and **Bharatiya Sakshya Adhiniyam 2023**.

**Precaution checklist for every engagement:**
- [ ] Written consent from the owner or an authorized representative
- [ ] Test only within that consent, respecting every limitation
- [ ] Take steps to avoid damaging systems or networks
- [ ] Don't access, use or disclose personal data without permission
- [ ] Don't intercept communications without consent of at least one party
- [ ] No testing of health-data (HIPAA) systems without proper authorization
- [ ] Unsure about local rules? Check, or ask a lawyer.

**My takeaways:**
- Written consent within scope is what separates a pentest from a crime, in every country.
- For Indian VAPT interviews, know IT Act Sections 43 / 66 and the DPDP Act.

#### Penetration Testing Process
- **Pentest process:** the successive steps I take to find a path to a predefined objective. It's *deterministic*: every step is based on results I discover or provoke, not chance.
- **Not a recipe.** Stages stay coarse and flexible because every client's infrastructure and expectations differ. I can build my own playbook per stage, but I always have to adapt.
- **Starting over isn't failure.** Going back to the start with new information is a new pass that builds on the old one.

**What this section adds to the stage table above:**
- **Pre-Engagement** sets up, in a meeting or call: NDA, goals, scope, time estimate, rules of engagement. Part of it is educating the client.
- **Post-Exploitation** also means keeping access if the box changes (persistence), plus pillaging for creds and the data the client most wants protected. It's done to show impact or to feed lateral movement.
- **Lateral Movement** example chain: foothold on a web server → privesc → password in the registry → same password is local admin on a DB server → pillage the DB → more creds → deeper access.
- **Proof-of-Concept** tells the story of how weaknesses *chain* together, so the client can see how each vuln fits in and prioritize fixes. Bad documentation = hard remediation.
- **Post-Engagement:**
  - Separate documentation for admins (technical) and management (severity and risk)
  - Clean up all traces on every host
  - Report walkthrough meeting, sometimes an executive or board presentation
  - Archive testing data per the contract and company policy, kept until a set period ends or until the **retest** (post-remediation assessment)

**My takeaways:**
- A finding alone is weak. A *chain* (initial access → privesc → lateral → crown jewels) shows real impact.
- Two audiences for every report: technical teams need repro steps, management needs risk.

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
