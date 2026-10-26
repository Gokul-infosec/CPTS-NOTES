# Attacking Web Applications with Ffuf

> Status: ✅ Complete
> Date started: 2026-10-26
> Date finished: 2026-10-26

## 1. Goal
Fuzz web apps with **ffuf** to find hidden directories, files, parameters, values, subdomains and vhosts. `FUZZ` is the placeholder the wordlist fills; `-fs`/`-fc`/`-fw` filter noise.

## Fuzzing recipes
```bash
# directories / files
ffuf -w wl.txt:FUZZ -u http://t/FUZZ
# extensions (what extensions exist on 'index')
ffuf -w ext.txt:FUZZ -u http://t/indexFUZZ
# pages under a known ext
ffuf -w wl.txt:FUZZ -u http://t/blog/FUZZ.php
# recursive
ffuf -w wl.txt:FUZZ -u http://t/FUZZ -recursion -recursion-depth 1 -e .php -v
# subdomains
ffuf -w subs.txt:FUZZ -u https://FUZZ.target.com/
# vhosts (filter the baseline size with -fs)
ffuf -w hosts.txt:FUZZ -u http://target/ -H 'Host: FUZZ.target.com' -fs <N>
# parameters — GET then POST
ffuf -w params.txt:FUZZ -u 'http://t/admin.php?FUZZ=key' -fs <N>
ffuf -w params.txt:FUZZ -u http://t/admin.php -X POST -d 'FUZZ=key' -H 'Content-Type: application/x-www-form-urlencoded' -fs <N>
# value fuzzing (e.g. brute an id)
ffuf -w ids.txt:FUZZ -u http://t/admin.php -X POST -d 'id=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -fs <N>
```

## Filtering & wordlists
- Filter: `-fs <size>` / `-fc <code>` / `-fw <words>` to hide the baseline response; `-mc all` to see everything first.
- Wordlists (SecLists): `Discovery/Web-Content/directory-list-2.3-*.txt`, `web-extensions.txt`, `DNS/subdomains-top1million-5000.txt`, `burp-parameter-names.txt`.
- Make a numeric list: `for i in $(seq 1 1000); do echo $i; done > ids.txt`.
- Add the vhost to `/etc/hosts` before vhost fuzzing resolves: `echo "<ip> target.htb" | sudo tee -a /etc/hosts`.

## Advanced flags & techniques
```bash
# ignore wordlist comments + colorize (use everywhere)
ffuf -ic -c -w wl.txt:FUZZ -u http://t/FUZZ
# auto-calibrate filtering (learns & filters the baseline for you — no manual -fs)
ffuf -ac -w wl.txt:FUZZ -u http://t/FUZZ          # -ach = per-host, -acc "<str>" = custom calib string
# recursion: greedy recurses on ALL matches, not just redirects
ffuf -w wl.txt:FUZZ -u http://t/FUZZ -recursion -recursion-strategy greedy
# multi-wordlist modes: clusterbomb (all combos, default), pitchfork (parallel), sniper
ffuf -w users.txt:W1 -w pass.txt:W2 -u http://t/login -X POST \
     -d 'username=W1&password=W2' -H 'Content-Type: application/x-www-form-urlencoded' -fc 200
# replay matched hits through Burp (only matches hit the proxy — clean history)
ffuf -w wl.txt:FUZZ -u http://t/FUZZ -replay-proxy http://127.0.0.1:8080
# send ALL traffic through a proxy
ffuf -w wl.txt:FUZZ -u http://t/FUZZ -x http://127.0.0.1:8080
# fuzz from a raw request saved out of Burp
ffuf -request req.txt -request-proto http -w api.txt:FUZZ -mc all -fs 0
# HTML report
ffuf -c -w wl.txt:FUZZ -u http://t/FUZZ -o report.html -of html   # -of: json/html/md/csv/all
```
Login username-enum via response regex (app leaks "already exists"):
```bash
ffuf -w names.txt:FUZZ -X POST -d 'username=FUZZ&email=x&password=x' \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -u http://t/signup -mr "already exists"
```
API endpoints: `-mc all -fs 0` (match everything, filter empty bodies) finds routes that return odd codes. Fuzz across found subdomains with a shell loop:
```bash
for sub in archive test faculty; do ffuf -ic -w ext.txt:FUZZ -u http://$sub.t.htb/indexFUZZ; done
```

> Workflow: run once with `-mc all` to learn the baseline size, then `-fs` that size to filter it out so only real hits show (or let `-ac` do it automatically). Hidden params (burp-parameter-names) often reveal unlinked, less-protected functionality — a top way in. `-replay-proxy` keeps Burp history clean by only sending the hits.

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
