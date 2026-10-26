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

## Full command reference (source notes)
Complete ffuf command set + flag reference — nothing trimmed.

**Quick reference**
| Command | Description |
|---|---|
| `ffuf -h` | Help |
| `ffuf -ic -c -w wordlist.txt:FUZZ -u http://SERVER_IP:PORT/FUZZ` | Directory fuzzing |
| `ffuf -ic -c -w wordlist.txt:FUZZ -u http://SERVER_IP:PORT/indexFUZZ` | Extension fuzzing |
| `ffuf -ic -c -w wordlist.txt:FUZZ -u http://SERVER_IP:PORT/blog/FUZZ.php` | Page fuzzing |
| `ffuf -ic -c -w wordlist.txt:FUZZ -u http://SERVER_IP:PORT/FUZZ -recursion -recursion-depth 1 -e .php -v` | Recursive fuzzing |
| `ffuf -ic -c -w wordlist.txt:FUZZ -u https://FUZZ.hackthebox.eu/` | Sub-domain fuzzing |
| `ffuf -ic -c -w wordlist.txt:FUZZ -u http://academy.htb:PORT/ -H 'Host: FUZZ.academy.htb' -fs xxx` | VHost fuzzing |
| `ffuf -ic -c -w wordlist.txt:FUZZ -u http://admin.academy.htb:PORT/admin/admin.php?FUZZ=key -fs xxx` | Parameter fuzzing (GET) |
| `ffuf -ic -c -w wordlist.txt:FUZZ -u http://admin.academy.htb:PORT/admin/admin.php -X POST -d 'FUZZ=key' -H 'Content-Type: application/x-www-form-urlencoded' -fs xxx` | Parameter fuzzing (POST) |
| `ffuf -c -w ids.txt:FUZZ -u .../admin.php -X POST -d 'id=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -fs xxx` | Value fuzzing |

**Wordlists**
| Path | Use |
|---|---|
| `/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt` | Directory/page |
| `/usr/share/seclists/Discovery/Web-Content/web-extensions.txt` | Extensions |
| `/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt` | Subdomains |
| `/usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt` | Parameters |

**Misc**
| Command | Description |
|---|---|
| `sudo sh -c 'echo "SERVER_IP  academy.htb" >> /etc/hosts'` | Add a vhost to /etc/hosts |
| `for i in $(seq 1 1000); do echo $i >> ids.txt; done` | Create a numeric sequence wordlist |
| `curl http://admin.academy.htb:PORT/admin/admin.php -X POST -d 'id=key' -H 'Content-Type: application/x-www-form-urlencoded'` | POST with curl |

**Extensions / recursion / proxy / reports**
| Command | Description |
|---|---|
| `ffuf -c -w 9-big.txt -u http://easy.box/FUZZ -e .git,.txt,.json,.php,.html,.bak,.old,.sql,.zip,.conf,.cfg,.asp,.aspx,.cs` | Root + many extensions |
| `ffuf -c -w common.txt -u http://vulnnet.htb/FUZZ -e .txt,.json,.php,.html,.bak,.old,.sql,.zip,.zz -fc 403` | Extensions, filter 403 |
| `ffuf -c -v -w directory-list-2.3-small.txt -u http://<ip>/FUZZ -e .php -recursion -recursion-depth 1` | Recursive with ext |
| `ffuf -c -w webfuzz_less.txt -u http://box:8080/FUZZ -x http://box:31337` | Fuzz via upstream proxy |
| `ffuf -c -w webfuzz_less.txt -u http://box:8080/FUZZ -replay-proxy http://127.0.0.1:8080` | Replay matches via Burp |
| `ffuf -c -w common.txt -u http://oscp.sec:8080/FUZZ -o ffuf_report.html -of html` | HTML report |
| `ffuf -c -w common.txt -u http://<ip>:8080/FUZZ -o ffuf_report.html -of html && firefox ffuf_report.html` | Report + open |

**Subdomain / vhost loops**
| Command | Description |
|---|---|
| `ffuf -c -ic -w subdomains-top1million-5000.txt -u http://FUZZ.academy.htb:12345/ -fc 403` | Subdomain fuzz |
| `ffuf -c -w ./vhosts -u http://192.168.10.10 -H "HOST: FUZZ.randomtarget.com" -fs 612` | VHost fuzz |
| `ffuf -u http://trick.htb -c -w list.txt -H 'Host: preprod-FUZZ.trick.htb' -fw 1697` | VHost w/ prefix, filter words |
| `for sub in archive test faculty; do ffuf -c -ic -w web-extensions.txt:FUZZ -u http://$sub.academy.htb:57089/indexFUZZ; done` | Ext fuzz across subdomains |
| `for sub in archive test faculty; do ffuf -w raft-large-words-lowercase.txt:FUZZ -u http://$sub.academy.htb:57089/FUZZ -recursion -recursion-depth 1 -e .php,.phps,.php7 -v -t 200 -fs 287 -ic; done` | Multi-subdomain recursive |

**API / parameters / username-enum / creds**
| Command | Description |
|---|---|
| `ffuf -u 'http://target/weather/forecast?city=\'FUZZ-- -c -w special-chars.txt -mc 200,500 -fw 9` | Fuzz for injection via special chars |
| `ffuf -w burp-parameter-names.txt:FUZZ -u http://admin.academy.htb:54542/admin/admin.php?FUZZ=key -fs xxx` | GET parameter fuzz |
| `ffuf -w burp-parameter-names.txt:PARAM -w values.txt:VAL -u http://flasky.offsec/add?PARAM=VAL -mr "VAL"` | Param+value (match reflected) |
| `ffuf -w burp-parameter-names.txt:FUZZ -u .../admin.php -X POST -d 'FUZZ=key' -H 'Content-Type: application/x-www-form-urlencoded' -fs xxx` | POST parameter fuzz |
| `ffuf -u http://prd.m.rendering-api.interface.htb/FUZZ -c -w raft-small-words.txt -mc all -fs 0` | API endpoint discovery |
| `ffuf -u .../api/FUZZ -c -w raft-small-words.txt -mc all -fs 50 -d 'x=x'` | API discovery with body |
| `ffuf -request api.txt -request-proto http -c -w api-seen-in-wild.txt -mc all -fs 36` | Fuzz from a raw request file |
| `ffuf -c -w names.txt -X POST -d "username=FUZZ&email=x&password=x&cpassword=x" -H "Content-Type: application/x-www-form-urlencoded" -u http://<ip>/customers/signup -mr "An account with this username already exists"` | Username enumeration |
| `ffuf -c -w valid_usernames.txt:W1,10-million-password-list-top-100.txt:W2 -X POST -d "username=W1&password=W2" -H "Content-Type: application/x-www-form-urlencoded" -u http://<ip>/customers/login -fc 200` | Credential brute (two lists) |

**Key flags** (`ffuf` v2.0.0-dev)
| Flag | Meaning |
|---|---|
| `-H` / `-X` / `-b` / `-d` | Header / method / cookie / POST data |
| `-r` / `-recursion` / `-recursion-depth` / `-recursion-strategy` | Follow redirects / recurse / depth / default\|greedy |
| `-replay-proxy` / `-x` | Replay matches via proxy / route all via proxy |
| `-t` / `-rate` / `-p` / `-maxtime` | Threads / req-per-sec / delay / max runtime |
| `-ac` / `-acc` / `-ach` / `-acs` | Auto-calibrate / custom / per-host / basic\|advanced |
| `-c` / `-v` / `-s` / `-ic` | Colorize / verbose / silent / ignore comments |
| `-mc` / `-ml` / `-mr` / `-ms` / `-mw` / `-mt` | Match code / lines / regex / size / words / time |
| `-fc` / `-fl` / `-fr` / `-fs` / `-fw` / `-ft` | Filter code / lines / regex / size / words / time |
| `-mode` | clusterbomb / pitchfork / sniper |
| `-e` / `-w` / `-request` / `-request-proto` | Extensions / wordlist:KEYWORD / raw request / proto |
| `-o` / `-of` / `-od` / `-or` | Output file / format (json,html,md,csv,all) / dir / only-if-results |
