# Footprinting

> Status: ✅ Complete
> Date started: 2026-10-26
> Date finished: 2026-10-26

## 1. Goal
Per-service footprinting/enumeration. (Service-command reference also in `cheatsheets/service-enumeration.md`.)

## Infrastructure / external recon
```bash
curl -s "https://crt.sh/?q=<domain>&output=json" | jq .   # cert transparency -> subdomains/hosts
for i in $(cat ips.txt); do shodan host $i; done          # what each IP exposes, per Shodan
```

## FTP (21)
```bash
ftp <HOST>                                  # interact (also: nc -nv <HOST> 21 / telnet <HOST> 21)
openssl s_client -connect <HOST>:21 -starttls ftp   # encrypted control channel
wget -m --no-passive ftp://anonymous:anonymous@<HOST>   # mirror everything (anon login)
```
Check anonymous login, readable/writable dirs, config files left in the root.

## SMB (139/445)
```bash
smbclient -N -L //<HOST>                    # list shares, null session
smbclient //<HOST>/<share>                  # connect to a share (then get/put)
rpcclient -U "" <HOST>                       # null RPC -> enumdomusers, querydominfo, etc.
samrdump.py <HOST>                           # impacket user enumeration
smbmap -H <HOST>                             # shares + permissions
crackmapexec smb <HOST> --shares -u '' -p '' # share enum via null session
enum4linux-ng.py <HOST> -A                   # all-in-one (users, shares, policy, OS)
```

## NFS (111/2049)
```bash
showmount -e <HOST>                          # exported shares
mount -t nfs <HOST>:/<share> ./mnt -o nolock # mount it
umount ./mnt                                 # clean up
```
Watch for root_squash misconfig and world-readable exports.

## DNS (53)
```bash
dig ns <domain> @<ns>                         # nameservers
dig any <domain> @<ns>                         # grab everything the server will give
dig axfr <domain> @<ns>                        # zone transfer (jackpot if allowed)
dnsenum --dnsserver <ns> --enum -p 0 -s 0 -o subs.txt -f ~/subdomains.list <domain>
```

## SMTP (25) / IMAP (143/993) / POP3 (110/995)
```bash
telnet <HOST> 25                              # banner; VRFY/EXPN/RCPT user enum
openssl s_client -connect <HOST>:imaps        # IMAPS (also :pop3s)
curl -k 'imaps://<HOST>' --user <user>:<pass> # authenticated IMAPS
```

## SNMP (161/udp)
```bash
snmpwalk -v2c -c <community> <HOST>           # walk OIDs (try community 'public')
onesixtyone -c community-strings.list <HOST>  # brute community strings
braa <community>@<HOST>:.1.*                   # brute/enumerate OIDs fast
```

## Databases — MySQL (3306) / MSSQL (1433) / Oracle TNS (1521)
```bash
mysql -u <user> -p<pass> -h <HOST>                      # MySQL login
mssqlclient.py <user>@<HOST> -windows-auth              # MSSQL (impacket), Windows auth
./odat.py all -s <HOST>                                  # Oracle: scan DB services/SIDs
sqlplus <user>/<pass>@<HOST>/<db>                        # Oracle login
./odat.py utlfile -s <HOST> -d <db> -U <user> -P <pass> --sysdba --putFile C:\\path file.txt ./file.txt  # Oracle file upload
```

## IPMI (623/udp)
```bash
msf6 auxiliary(scanner/ipmi/ipmi_version)     # version detect
msf6 auxiliary(scanner/ipmi/ipmi_dumphashes)  # dump BMC password hashes -> crack offline
```

## Remote management — SSH (22) / RDP (3389) / WinRM (5985/5986)
```bash
ssh-audit.py <HOST>                            # SSH algo/security audit
ssh <user>@<HOST>                              # password login
ssh -i private.key <user>@<HOST>               # key login
ssh <user>@<HOST> -o PreferredAuthentications=password   # force password auth
rdp-sec-check.pl <HOST>                        # RDP security settings
xfreerdp /u:<user> /p:"<pass>" /v:<HOST>       # RDP from Linux
evil-winrm -i <HOST> -u <user> -p <pass>       # WinRM shell
wmiexec.py <user>:"<pass>"@<HOST> "whoami"     # command exec over WMI
```

> Workflow: nmap tells you which ports are open → footprint each service above → pull banners, versions, anonymous/null access, shares, users, and misconfigs. Null/anonymous access (SMB, FTP, SNMP `public`, LDAP anon bind) and DNS AXFR are the quickest early wins.

## LDAP (389/636) — enumeration & injection (reference)
LDAP = directory service (users, groups, devices) in a tree (DIT) of DNs with attributes (cn, uid, mail). Ports **389** (plain) / **636** (LDAPS). Implementations: OpenLDAP, Microsoft AD (AD uses LDAP).

**Enumeration**
```bash
nmap -p389,636 -sV -sC <HOST>
# anonymous bind (some servers allow read):
ldapsearch -x -H ldap://<HOST> -b "<base_dn>" "(objectClass=*)"
# authenticated bind:
ldapsearch -x -H ldap://<HOST> -D "<bind_dn>" -w "<pass>" -b "<base_dn>" "(objectClass=*)"
enum4linux -a <HOST>            # broader Windows/AD enum
```
- Try common base DNs (`dc=example,dc=com`, `ou=users,...`), look for `sAMAccountName`, enumerate groups.
- GUI browsers: JXplorer, Apache Directory Studio, LdapAdmin.

**LDAP injection** (app builds an LDAP filter from user input — like SQLi for LDAP). Special chars: `*` `(` `)` `|` `&` `!` `\`.
- Auth bypass example: filter `(&(uid=$u)(userPassword=$p))`, enter username `*)(uid=*` to break the logic.
- Enumerate users: `*)(|(cn=*)(mail=*))` · privilege probe: `admin*)(|(objectClass=*))`
- **Fix:** sanitize/escape LDAP special chars, parameterize, least-privilege bind account, LDAPS, WAF.
