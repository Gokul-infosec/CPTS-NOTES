# Service Enumeration Reference

Per-service enum commands. Compiled reference — replace `IP`/`<TARGET>` with the host, `users.txt`/`pass.txt` with wordlists. Brute-force and credentialed tools are intrusive: in-scope + approved only (watch for fail2ban-style lockouts).

## Scanning
```bash
# TCP quick
sudo nmap -Pn -v -sS -sV -sC -oN tcp-quick.nmap IP
# TCP full (all ports)
sudo nmap -Pn -sS --stats-every 3m --max-retries 1 --max-scan-delay 20 --defeat-rst-ratelimit -T4 -p1-65535 -sV -oN tcp-full.nmap IP
# TCP targeted re-scan of newly found ports
sudo nmap -Pn -v -sS -A -p PORTS -oN tcp-extra.nmap IP
# UDP quick / top 1000
sudo nmap -Pn -v -sU -sV --top-ports=30 -oN udp-quick.nmap IP
sudo nmap -Pn --top-ports 1000 -sU --stats-every 3m --max-retries 1 -T4 -oN udp-1000.nmap IP
# Host discovery
fping -a -g 10.10.10.0/24 2>/dev/null      # ICMP sweep
arp-scan -l                                 # local network ARP
```

## FTP (21)
Checklist: version vulns · anonymous login · read access · writable dirs · webroot/other service roots.

## SSH (22)
```bash
nmap -sV --script=ssh-hostkey -p22 IP
```
Checklist: version vulns · user enum · password auth allowed? · host key seen elsewhere · brute (Hydra/CeWL/Patator) only if needed and in scope.

## Telnet (23) / SMTP (25) / POP3 (110)
```bash
# SMTP: HELO/EHLO <domain> for banner/version; check open relay, user enum
# POP3 via telnet:
#   USER <user> / PASS <pass> / LIST / RETR <n>
```

## DNS (53)
Zone transfer + may indicate a Windows DC. (DNS enum tooling in the footprinting module.)

## Kerberos (88)  — indicates a Domain Controller
```bash
kerbrute userenum --dc IP -d DOMAIN users.txt
GetNPUsers.py DOMAIN/ -usersfile users.txt -no-pass    # AS-REP roasting check
```

## RPC (135) / NetBIOS (139)
```bash
sudo nmap -sS -Pn -sV --script=rpcinfo -p135 IP
rpcclient -U "" -N IP
nmblookup -A IP
nbtscan IP
```

## LDAP (389, 636, 3268, 3269)
```bash
sudo nmap -sS -Pn -sV --script="ldap*" -p389,636,3268,3269 IP
```

## SNMP (161/UDP)
```bash
snmpwalk -v2c -c public IP
snmp-check IP
onesixtyone -c community.txt IP
sudo nmap -sU -sV -p161 --script "snmp*" IP
```

## Oracle TNS (1521)
```bash
nmap -sV --script=oracle-tns-version -p1521 IP
odat tnscmd -s IP --ping
odat sidguesser -s IP
```

## MySQL (3306)
```bash
nmap -sV --script="mysql*" -p3306 IP
mysql -h IP -u root -p
```

## Web (80/443)
```bash
sudo nmap -Pn -sC -p80,443 IP
whatweb http://IP
nikto -host IP
gobuster dir --url http://IP --wordlist /usr/share/seclists/Discovery/Web-Content/big.txt
wpscan --url http://IP --enumerate u          # WordPress
```
Checklist: browse app, source view, robots.txt, sitemap.xml, known software + default creds, SQLi, OS command exec, LFI/RFI, shellshock (CGI), cert hostnames.

## SMB (139/445)
```bash
sudo nmap -Pn --script="smb-os-discovery" -p139,445 IP
sudo nmap -Pn --script="smb-enum*" -p139,445 IP
sudo nmap -Pn --script="smb-vuln*" -p139,445 IP
# null session / shares
smbclient -N -L \\\\IP
smbmap -H IP
enum4linux -a IP
# with creds
smbmap -u USER -p PASS -d DOMAIN -H IP
smbmap -R -H IP                               # recursive listing
# download / upload
smbmap -H IP --download 'SHARE\path\file'
smbmap -H IP --upload test.txt SHARE/test.txt
# mount
sudo mount -t cifs //IP/SHARE ~/mnt
```

## NFS (2049) / TFTP (69/UDP)
```bash
showmount -e IP
mount -t nfs -o vers=3 IP:/export ~/mnt
# TFTP: atftp client; can read system files if misconfigured
```

## Automation & finding exploits
```bash
autorecon IP
./NmapAutomator.sh IP All
searchsploit <product> <version>
searchsploit -x path/to/exploit               # view exploit
# also: Exploit-DB, CVE Details per service, Google the exact banner
```
