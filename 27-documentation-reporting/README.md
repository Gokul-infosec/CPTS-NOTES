# Documentation & Reporting

> Status: ✅ Complete (reference; sections 7–8 are the hands-on lab to do myself)
> Date started: 2026-10-26
> Date finished: 2026-10-26

## 1. Goal
The report is the deliverable — the client pays for it, not for the pwn. Take thorough, organised notes and keep raw evidence as you test, so writing the report is assembly, not archaeology. Good documentation also covers you: when a client blames the test for an outage, timestamped logs + a confirmed scope file are what prove what you actually did.

## Why documentation matters (the horror stories)
- **VM died mid-engagement** → daily evidence sync to shared storage + a VM template meant zero lost work.
- **Client claimed scans downed critical servers** → timestamped scans + signed scope file showed those IPs were in-scope and testing was not reckless. (Lesson added: always ask for an explicit *exclude* list too.)
- **"Your scans slowed our network"** → scan logs proved best-practice testing; real cause was debug mode left on every network device. Logs forced them to investigate instead of blaming the tester.

Takeaway: network issues during a test get blamed on the tester by default. Be able to produce evidence for everything.

## Snapshot-in-time overview (goes in every report)
A pentest is a point-in-time snapshot. The report's overview states: type of work, who performed it, source IPs used, special considerations (remote/VPN or on-site), and the exact testing window. Add a disclaimer, e.g. *"All testing activities were performed between Jan 7–19, 2022. This report represents a snapshot in time; the state of assets outside this window cannot be attested to."*

## Notetaking structure (adapt per engagement)
Keep sections for: **Attack Path** · **Credentials** (compromised secrets, centralised) · **Findings** (a subfolder + narrative + evidence per finding) · **Vuln Scan Research** · **Service Enumeration Research** · **Web Application Research** · **AD Enumeration Research** · **OSINT** · **Administrative Info** (POCs, RoE objectives, running to-do) · **Scoping Info** (in-scope IPs/CIDRs/URLs, provided creds) · **Activity Log** (high-level, for event correlation) · **Payload Log** (what/where/hash of anything uploaded).

**Tools:** Obsidian (local, Markdown — great default), Outline (cloud/self-host), CherryTree, Notion, OneNote, etc. **Local vs cloud matters once real client data is involved** — check company policy/contract before using a cloud tool. Markdown exports move between tools easily.

## Logging (raw output is the fallback)
**Tmux logging** captures everything typed/returned in a pane to a file.
```bash
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm   # plugin manager
# ~/.tmux.conf:
#   set -g @plugin 'tmux-plugins/tpm'
#   set -g @plugin 'tmux-plugins/tmux-sensible'
#   set -g @plugin 'tmux-plugins/tmux-logging'
#   set -g history-limit 50000          # so retroactive logging doesn't lose scrollback
#   run '~/.tmux/plugins/tpm/tpm'
tmux source ~/.tmux.conf ; tmux new -s engagement
```
- `prefix + Shift+I` — install plugins
- `prefix + Shift+P` — start/stop logging (file only populates on stop/exit)
- `prefix + Alt+Shift+P` — retroactive log of the whole pane (limited by history-limit)
- `prefix + Alt+P` — screen-capture one pane cleanly (avoids grabbing the other pane's text)

Useful plugins: tmux-resurrect (restore sessions after reboot), tmux-sessionist, tmux-pain-control.

## Tracking what you changed (so it can be reverted)
- **Artifacts:** for every payload — when used, which host, file path, cleaned up or not, and a **file hash**. Provide this even if you deleted the webshell.
- **Account creation / system mods:** host/IP, timestamp, description, location, app/service touched, account name (+ password if you may have to surrender it). **Get written approval before making system changes** — settle the threshold on the kickoff call.

## Evidence: capture, store, redact
- **Prefer terminal text over screenshots** — easier to redact/highlight, neater, copy-pasteable for the client to reproduce. Strip formatting before pasting into Word (smart quotes break commands). Mark cuts with `<SNIP>`; never alter output.
- **Capture failed attempts too** — proves thoroughness, and gives you a "testing performed" narrative when findings are few.
- **Redaction:** black solid bars edited into the image — **never blur/pixelate** (reversible, e.g. Unredacter), and never CSS/HTML tricks on a blog. Redact creds + PII; for hashes keep first/last 3–4 chars. Annotate (arrows/boxes) and crop to the relevant part; include the URL/host bar.
- **What NOT to archive:** don't exfil real PII / legally-discoverable / criminal data — screenshot the *directory listing* of a sensitive share, not the file contents (GDPR and liability). "Do no harm."

## Engagement folder structure
```bash
mkdir -p ACME-IPT/{Admin,Deliverables,Evidence/{Findings,Scans/{Vuln,Service,Web,'AD Enumeration'},Notes,OSINT,Wireless,'Logging output','Misc Files'},Retest}
```
- **Admin** (SoW, kickoff notes, status reports) · **Deliverables** (report, spreadsheets, decks) · **Evidence** (Findings — one folder each; Scans — Vuln/Service/Web/AD exports; Notes; OSINT; Wireless; Logging output; Misc Files — payloads/webshells/scripts) · **Retest** (optional, mirror the structure).
- Obsidian can sit on top of this same folder tree — notes and evidence folders in one place. A script that makes the dirs + the ~10 numbered Notes `.md` files is a good engagement-start reflex.

## Assessment types (shapes the report)
- **Vulnerability Assessment** — automated scan, no exploitation; often validate results to flag false positives. Report themes it by severity/count and maps patterns to process gaps.
- **Penetration Test** — goes beyond scanning. Perspectives: **black box** (just a name/connection), **grey box** (in-scope IPs), **white box** (creds/source/config). Evasion: none → hybrid (start quiet, get noisier to test detection) → fully evasive (adversary simulation, long-term).
- **Internal vs External** — external = anonymous internet attacker on public assets; internal = from inside the network (anon or authenticated), aiming for foothold → horizontal/vertical privesc → lateral movement → domain compromise.
- **Inter-disciplinary** — purple team (red + IR/blue), cloud-focused, IoT (network/cloud/app/hardware), web app, hardware/physical. Standard report layout still fits most.

## Report types / deliverables
- **Draft → Final** — submit a draft, let the client comment (management responses, language tweaks), then issue final. Many auditors won't accept a draft for compliance.
- **Post-Remediation (retest)** — retest **only the original findings on the original hosts**, with a time limit. Don't re-scan the whole environment (scope creep). Note time elapsed + that it's a point-in-time recheck. If auditor-time pressure appears, suggest a documented remediation plan with a deadline rather than bending severities.
- **Attestation Report/Letter** — 1–2 pages for the client's vendors/customers: number of findings, approach, general comments — **no technical detail or secrets**.
- **Slide deck** — pitch to the audience's level (technical vs exec); anecdotes beat walls of graphs.
- **Spreadsheet of findings** — tabular findings (no exec summary/narrative) for sorting + ticketing import.
- **Vulnerability Notification** — stop-work alert for a critical (internet-facing unauth RCE, sensitive data exposure, default creds). Keep it lean: technical detail + reproducible evidence. Set the threshold at kickoff.

## Report components
**Prioritise** — disclose everything, but don't burn time validating 35 SSL/TLS variations; consolidate low/info issues into categories and focus effort on high-impact (RCE, sensitive data).

**Attack chain** — the narrative of how findings combined to go from foothold → domain compromise; shows *why* a medium becomes a high when chained. Format = summary bullets first, then step-by-step with command output + screenshots (reuse as per-finding evidence). The module's sample chain is a classic internal: Responder → crack NTLMv2 → BloodHound → Kerberoast → crack TGS → LSA secrets (CrackMapExec `--lsa`) → Pass-the-Ticket (Rubeus) → DCSync (mimikatz) → NTDS dump (secretsdump). *(Those actual attack commands live in [module 13 — Active Directory](../13-active-directory/); here the point is how to present them: preserve real command+output, `<SNIP>` noise, `<REDACTED>` creds/hashes, highlight the command run and the key output line.)*

**Executive Summary** (the most important page or two) — written for a **non-technical budget-holder**:
- *Do:* give specific numbers ("25 occurrences", not "several"); keep it to ~1.5–2 pages; describe *what you accessed* in business terms (HR docs, banking systems — not "Domain Admin"); describe the **process** that broke down (not "install 3 patches"); optionally set rough remediation effort (low/moderate/significant).
- *Don't:* name vendors (EDR fine, "CrowdStrike" no); use acronyms (SNMP, MitM); over-explain trivia; use obscure words; reference technical sections.
- **Translate** for the glossary: VPN/SSH → "a protocol for secure remote administration"; hash → "output of an algorithm used to validate file integrity"; password spraying → "trying one easy password against many accounts"; SQLi/XSS → "input accepted without sanitising characters meant to manipulate the app". Avoid absolutes ("testing seemed to go unnoticed", not "they can't detect anything").

**Summary of Recommendations** — short/medium/long-term, each tied to a finding; long-term maps to process fixes (patch program, SDLC, security templates). Stops the "which high is the most high?" question.

**Findings** — the technical write-ups (covered in sections 5–8, to be added).

**Appendices** — *static:* Scope, Methodology, Severity Ratings (map to CVSS or defend your own), Biographies (esp. PCI). *dynamic:* Exploitation Attempts/Payloads (so IR can tell you from a real attacker), Compromised Credentials, Configuration Changes (so they can revert), Additional Affected Scope, Information Gathering (external OSINT), **Domain Password Analysis** (DPAT stats from a cracked NTDS dump).

## How to write up a finding
Each finding needs, at minimum: **Description** (+ affected platform) · **Impact** if unresolved · **Affected systems** · **Recommendation** · **Reference links** · **Steps to reproduce + evidence**. Optional: CVE, OWASP/MITRE IDs, CVSS (fill it if your template has the field), ease/probability of exploitation. Tweak "stock" findings to the client's actual environment — "Default Credentials" on a DeskJet ≠ on the building HVAC or a key web app.

**Reproduction steps:**
- One step = one figure; don't cram multiple actions into one screenshot.
- If setup is needed (e.g. an MSF module), show the config in one figure and the result in a second.
- Write narrative *between* figures, not just captions.
- Prefer copy-pasteable text over screenshots (a Burp screenshot of a payload is useless if they can't copy it).
- Make evidence **defensible**: to prove cleartext creds, show the Wireshark capture of the human-readable auth — not just the login popup. For GUI/RDP/web, capture the URL bar or `ipconfig`/`ifconfig` to prove it's *their* host. Turn off your bookmarks bar and unprofessional extensions.

**Remediation recommendations** — be specific and actionable, give options:
- *Bad:* "Reconfigure the registry to harden against X." *Good:* list the full hive paths + value changes, and warn to test critical changes (like registry edits) on a small group first.
- *Bad:* "Buy [expensive tool]." *Good:* link the vendor's free workaround as interim, mention commercial tools exist but may be cost-prohibitive. Don't trap a client with no budget.

**References** — vendor-agnostic, thorough, no paywall, gets to the point, clean site (no ad/crypto-miner junk), unlikely to disappear (prefer reputable orgs over a personal blog). Writing your own blog source is a bonus.

**Findings DB** — maintain sanitised, reusable findings so you're not rewriting the same content (and risking inconsistency across a team). Tools: free — **WriteHat**, Ghostwriter, Dradis, VECTR; paid — PlexTrac, AttackForge, Rootshell Prism.

## Reporting tips & tricks
- **Write as you go** — fill templated parts (client, scope, contacts) during long scans; write the attack chain + findings with evidence *during* testing so nothing's recaptured at the end.
- **Templates** — one blank template per assessment type; never edit a previous client's report (you'll leave their name in — looks amateur).
- **MS Word** (Windows, not Mac — Mac Word breaks PDF margins/ToC links, no VB editor): use **font styles + table styles** (update once → changes everywhere, no direct formatting); built-in **captions** (auto-renumber); page numbers; ToC + List of Figures/Tables; a code/terminal style with **spell-check disabled** via language settings; custom dictionary for repeat typos ("pubic"→"public"). Hotkeys: `F4` repeat last action, `Ctrl+A`+`F9` update all fields/ToC, `Ctrl+Alt+S` split view, `Shift+F5` jump to last edit. Automate with `.dotm` macros (pop-up to inject client/dates/scope; strip bookmarked sections per assessment type).
- **Redaction/cleanup** — Greenshot with **solid shapes, never blur**; redact creds/hashes/secrets; remove unprofessional tool output (CME's `(Pwn3d!)` — change it in the config), check Hashcat output for crude cracked passwords (OK to swap these — the rare exceptions to "never alter output"); keep hostname/username professional (no `azzkicker@clientsmasher`); solid-black console theme (or light for printing).
- **Tell a story, show enough evidence (not 50 pages of console), stay organised, spell-check** (Grammarly/LanguageTool — but they may ship data to the cloud; get approval).

## Client communication
- **Start notification** (each engagement): tester name, type/scope, source IP(s), dates, primary/secondary contacts.
- **Daily stop notification**: signals testing ended for the day; good place for a high-level heads-up if there'll be 20+ highs. Start/stop times also let the client correlate alerts.
- **Open dialogue**: found a new subnet/subdomain? ask before adding to scope. Found external unauth RCE/SQLi? **stop and notify**. Got DA/EA? give a heads-up, keep testing, ask what else to focus on. Host went down? be upfront.

## QA & delivery
- The report is the highlight reel — "if you can't get it on paper so someone else understands, it may as well never have happened."
- **At least one QA reviewer who isn't the author** (ideally two passes: technical accuracy, then style). Solo? sleep on it and re-read. Keep a QA checklist in the template (remove before final). QA fixes only minor stuff; the author fixes missing evidence/findings/exec-summary.
- Turn on **Track Changes** to learn from QA edits.
- **Draft → review meeting (~a week later, findings one by one) → Final** (often identical bar the label; some auditors only accept "Final"). Archive testing data per retention policy until retest.

> Full workflow: set up logging + folder/vault structure **before** testing → take structured notes + capture defensible evidence as you go → write templated report parts during scans → build the attack chain → distil a non-technical **executive summary** → tie **recommendations** to findings → write each **finding** (description/impact/affected/recommendation/references/repro+evidence, specific & actionable) → attach appendices → self-QA → independent QA with Track Changes → issue **Draft** → review meeting → issue **Final** → archive. Write as you go, redact with solid shapes, tell a story.

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
