# Documentation & Reporting

> Status: 🟨 In progress (sections 1–2 of 8 filled)
> Date started: 2026-10-26
> Date finished: 

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

> Workflow so far: set up Tmux logging + the folder/vault structure *before* testing → take structured notes as you go (attack path, creds, per-finding folders) → capture terminal evidence (text > screenshots), redact with solid bars → log every payload/system change with hashes and timestamps → keep raw scan output as the fallback. Report-writing itself (types of reports, exec summary, findings write-ups) is sections 3–8 — to be added.

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
