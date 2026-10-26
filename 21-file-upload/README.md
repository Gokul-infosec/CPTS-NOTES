# File Upload Attacks

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Abuse a file-upload feature to get code execution (upload a web shell) or other impact, by defeating the upload validation.

## The attack
Upload a server-side script (PHP/ASP/JSP) that runs commands, then browse to it → RCE. (I don't store web-shell payloads; the technique is "upload a file the server will execute.") The whole game is **bypassing the filter**.

## Bypassing validation
- **Client-side only** (JS checks the extension): intercept in Burp and change it back, or disable the JS.
- **Blacklist** (blocks `.php`): use an uncommon executable extension — `.phtml`, `.php5`, `.phar`, `.pHp` (case), `.asp`→`.aspx`/`.cshtml`.
- **Whitelist** (`.jpg` only): double extension `shell.jpg.php`, reverse `shell.php.jpg` (Apache misconfig), or char injection around the ext (`shell.php%00.jpg`, `%20`, `%0a`, trailing `.`/`/`).
- **Content-Type / MIME**: set `Content-Type: image/png` on the request even though the body is PHP.
- **Magic bytes**: prepend real file signature (e.g. `GIF89a;`) so content-sniffing passes, PHP after it still runs.

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
