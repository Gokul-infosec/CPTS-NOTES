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

#### Pre-Engagement
Three parts: **scoping questionnaire → pre-engagement meeting → kick-off meeting**. An NDA comes before all of them.

**NDA types:**
- **Unilateral:** only one party must keep things confidential
- **Bilateral:** both parties do. Most common, and the one that protects pentesters.
- **Multilateral:** more than two parties (e.g. a cooperative network where every member must sign)

**Who can hire me:** only someone with real authority, usually C-level (CEO, CTO, CISO, CSO, CRO, CIO), VP / manager of internal audit, or VP / director of IT or security. In big companies it's usually IT, audit or security senior management. If a random employee hires me without authority, *I'm* the one in legal trouble. Confirm early who signs the contract and RoE, and who the primary, secondary, technical and escalation contacts are.

**Documents and when they're made:**

| Document | When |
|---|---|
| NDA | After initial contact |
| Scoping questionnaire | Before the pre-engagement meeting |
| Scoping document | During the pre-engagement meeting |
| Proposal / Contract / Scope of Work (SoW) | During the pre-engagement meeting |
| Rules of Engagement (RoE) | Before kick-off |
| Contractor's agreement (physical tests only) | Before kick-off |
| Reports | During and after testing |

All of these should be reviewed by a lawyer. If the client sends its own scope list (IPs, URLs, creds), it also goes into the RoE as an appendix.

**Scoping questionnaire.** The client picks the service: internal/external VA or pentest, wireless, application, web app, physical, social engineering, red team. Then I need sizing details:
- Live hosts, IP / CIDR ranges, domains / subdomains, SSIDs
- Number of web / mobile apps, and how many user roles if testing is authenticated
- Phishing: how many users, and does the client give the list or do I build it with OSINT?
- Physical: how many sites, and how spread out?
- Red team objective and what's off limits
- Separate AD assessment? Starting as an anonymous user or a domain user? NAC bypass needed?
- Black / grey / white box, and evasiveness: non-evasive, hybrid (start quiet, get louder to find the detection point), or fully evasive

This drives the people assigned, the timeline and the price (10 external IPs ≪ 30 internal /24s). The answers get summarized into the scoping document.

**Pre-engagement meeting → contract.** Topics that feed the SoW:
- NDA, goals (big first, then smaller milestones), scope (legal basis comes first), test type (I recommend, client decides)
- Methodology (e.g. OSSTMM, OWASP), location (remote VPN or on site), time windows per phase, in or out of business hours
- Third parties: the client must get written consent from cloud / ISP / hosting providers, and I should ask for a copy as proof
- Evasive testing yes/no, risks explained, critical systems to avoid, information-handling standards (HIPAA, PCI, HITRUST, FISMA/NIST)
- Contact list with escalation order, communication channels, reporting format and presentation, payment terms
- The key point: find the client's priorities and deliver what *they* ordered, not what I think is better.

**RoE contents (condensed):** who's involved and their contacts, purpose and goals, exact scope (IPs, domains, URLs, CIDRs), communication channels, dates and time of day, test type and location, methodology (OSSTMM, PTES, OWASP), objectives / flags, evidence and information handling (encryption), system backups, incident handling and when testing is interrupted, status meeting schedule, reporting, retest dates, liability disclaimers (damage, data loss), and signed permission to test.

**Kick-off meeting.** After all documents are signed. Attendees: client PoCs and technical staff, plus our team (practice lead, testers, sometimes a PM or sales).
- Usually **no DoS testing**.
- **Critical finding on an external test** (unauth RCE, SQLi, sensitive data exposure): pause testing, send a vulnerability notification, call the emergency contacts so the client can decide on an emergency fix.
- **Internal test is stopped only if:** a system goes unresponsive, I find illegal activity (e.g. illegal content on a share), or I find signs of a real attacker / prior breach.
- Warn the client about risks: lots of logs and alerts, accounts may lock from brute forcing, and they must call me immediately if the network is affected.
- Explain things so the *least* technical person in the room can follow.

**Contractor's agreement (physical tests).** Different laws apply to physical intrusion, and staff usually don't know about the test. If I get caught and the police are called, this document is my "get out of jail free card". It covers: purpose, goal, testers and contacts, physical addresses, building names, floors, room IDs, physical components, timeline, notarization and permission to test.

**My takeaways:**
- The *right person* has to authorize the test, not just anyone at the company.
- Know the pause-and-notify rules (external critical finding vs internal stop conditions). That's a classic scenario question.
- For physical tests, always carry the signed contractor's agreement.

#### Information Gathering
The cornerstone of the whole test and the stage I return to most. Every exploit I run is only as good as the info behind it. Four categories, **all mandatory on every test:**

| Category | What I'm after | Notes |
|---|---|---|
| OSINT | Public info on the company and its people: events, dependencies, connections | Passwords, hashes, keys and tokens leak in misconfigured GitHub repos and code snippets posted on forums like StackOverflow. Found a live secret? Report it immediately per the RoE's incident handling section, and have the client's admin review it before I continue. |
| Infrastructure enumeration | Map of the company's presence on the internet / intranet: name servers, mail, web, cloud instances | OSINT + first active scans (DNS is key). Build a host / IP list and **check it against scope**. Also identify defenses (firewalls, WAFs) to know what will trigger alerts. Internally, this map feeds password spraying (one password vs many usernames). |
| Service enumeration | Each service: what it is, version, what info it leaks, *why it exists* | Old versions = known vulns. Admins often leave them unpatched because they're scared of breaking something that works. |
| Host enumeration | Per in-scope host: OS, services, versions, ports, its role, what it talks to | Internal services are often misconfigured because admins assume "not internet-facing = safe" (e.g. anonymous FTP, end-of-life OS). After exploitation, host enum moves *inside* the box: sensitive files, local services, scripts, apps. |

**Pillaging:** collecting sensitive info on an already exploited host (employee names, customer data, creds, configs). It isn't its own stage. It's part of information gathering and privesc, done locally after access. It shows impact and fuels privesc / lateral movement. There's no dedicated module; it's spread across Nmap, Getting Started, Password Attacks, AD, Linux/Windows privesc, Common Services, Common Applications and Enterprise Networks. The path has 150+ targets and 9 mini pentests to practice it.

**My takeaways:**
- Every target: do all four categories (OSINT, infrastructure, service, host). Skipping one is how things get missed.
- For every service, ask *why is this here?* The purpose points to the attack.
- Internal = where the lazy configs are.

#### Sections 9 to 14
To be added in my own words.

#### Practice
- Theory is useless without hands-on reps. Communication counts just as much: clear emails, presenting and defending my work, solid reports.
- Practice the client-facing parts too: role-play kickoff calls and report walkthroughs with a friend, and be ready to defend every recommendation on the spot.
- A thin report kills credibility. Every claim needs clear evidence, solid repro steps, clear remediation, and a well-written executive summary.

**Practice plan (starting example):** 2 modules → 3 retired machines → 5 active machines → 1 Pro Lab / Endgame

| Stage | How to do it |
|---|---|
| Modules | Read, do the exercises, finish. Then **redo the exercises from scratch taking notes**, and write technical *and* non-technical documentation (sample finding + exec summary entry). |
| Retired machines (2 easy + 1 medium) | User and root on my own → write tech and non-tech docs → compare with a write-up → list what I missed → watch IppSec → update my notes. |
| Active machines (2 easy + 2 medium + 1 hard) | Blackbox: no write-ups. User and root → tech and non-tech docs → get them proofread by a technical *and* a non-technical person. |
| Pro Lab / Endgame | Multi-host enterprise networks: hosts interacting, user-traffic attacks, full attack chain write-up from foothold to network compromise. |

**My takeaways:**
- Writing client-ready docs after every module/box is what makes the knowledge stick.
- Go through modules in order. If already partway through, revisit them with the 8-stage process in mind.

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
- Write my own notes for sections 9 to 14 (vulnerability assessment through lateral movement / PoC / post-engagement detail)

## 7. Cheat sheet lines
Copied into `cheatsheets/`:
- 
