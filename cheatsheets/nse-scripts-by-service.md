# Nmap NSE Scripts by Service

Reference of useful NSE scripts per service/port. Compiled reference — trim to the ones I've actually used as I go. Replace `<target>` with the host.

## Authentication & identity
| Service | Port | Scripts |
|---|---|---|
| LDAP | 389, 636 | `ldap*`, `ldap-search`, `ldap-rootdse`; `"(ldap*) and not brute"` |
| Kerberos | 88 | `krb5-enum-users` (arg `krb5-enum-users.realm='DOMAIN.LOCAL'`), `krb5-info` |
| SMB | 139, 445 | `smb-enum-shares`, `smb-enum-users`, `smb-os-discovery`, `smb-security-mode`, `smb2-capabilities`, `smb2-security-mode`, `smb-null-session`, `smb-vuln*` |
| RDP | 3389 | `rdp-enum-encryption`, `rdp-ntlm-info`, `rdp-vuln-ms12-020` |
| WinRM | 5985, 5986 | `http-windows-enum`, `winrm-enum-users` |

## Network services
| Service | Port | Scripts |
|---|---|---|
| FTP | 21 | `ftp-anon`, `ftp-bounce`, `ftp-syst`, `ftp-vsftpd-backdoor`, `ftp-proftpd-backdoor`, `ftp-libopie` |
| SSH | 22 | `ssh-hostkey`, `ssh-auth-methods`, `sshv1`, `ssh2-enum-algos` |
| Telnet | 23 | `telnet-encryption`, `telnet-ntlm-info` |
| SMTP | 25, 465, 587 | `smtp-commands`, `smtp-enum-users`, `smtp-open-relay`, `smtp-ntlm-info` |
| DNS | 53 | `dns-zone-transfer`, `dns-nsid`, `dns-service-discovery`, `dns-recursion`, `dns-cache-snoop` |
| TFTP | 69 (UDP) | `tftp-enum` (`-sU`) |
| POP3 | 110, 995 | `pop3-capabilities` |
| IMAP | 143, 993 | `imap-capabilities` |
| SNMP | 161, 162 (UDP) | `snmp-info`, `snmp-interfaces`, `snmp-processes`, `snmp-win32-services`, `snmp-sysdescr` (`-sU`) |
| R-services | 512, 513, 514 | `rpcinfo` |
| IPMI | 623 | `ipmi-version`, `ipmi-cipher-zero` |
| RSync | 873 | `rsync-list-modules` |
| NetBIOS | 137, 138 | `nbstat`, `smb-os-discovery`, `smb-enum-shares` |
| RPCBind | 111 | `rpcinfo` (`-sU -sT`) |
| SIP | 5060 (UDP) | `sip-methods`, `sip-enum-users` |
| NTP | 123 (UDP) | `ntp-info`, `ntp-monlist` |
| VNC | 5900 | `vnc-info`, `vnc-title` |
| X11 | 6000 | `x11-access` |

## Databases
| Service | Port | Scripts |
|---|---|---|
| MSSQL | 1433, 1434 | `ms-sql-info`, `ms-sql-empty-password`, `ms-sql-config` |
| Oracle TNS | 1521 | `oracle-tns-version`, `oracle-sid-brute` |
| MySQL | 3306 | `mysql-info`, `mysql-users`, `mysql-databases`, `mysql-empty-password` |
| PostgreSQL | 5432, 5433 | `pgsql-info`, `pgsql-databases`, `pgsql-users` |
| Redis | 6379 | `redis-info` |
| Memcached | 11211 | `memcached-info` |
| Elasticsearch | 9200 | `http-elasticsearch-head`, `http-title`, `http-methods` |
| CouchDB | 5984 | `http-couchdb-info` — note CVE-2017-12635/12636 (RCE) |
| MongoDB/others | — | check service-specific scripts |

## Web & app servers
| Service | Port | Scripts / notes |
|---|---|---|
| HTTP(S) | 80, 443, 8080, 8443 | `http-title`, `http-methods`, `http-enum`, `http-headers`, `http-server-header`, `http-auth-finder`, `http-vuln*` |
| WebDAV | 80, 443, 8080 | `http-webdav-scan` |
| Tomcat | 8080, 8443 | `http-tomcat-manager`, `http-tomcat-users` |
| Jenkins | 8080 | `http-jenkins-info` — watch anon access / script console |
| AJP | 8009 | `ajp-methods`, `ajp-headers`, `ajp-auth` — Ghostcat CVE-2020-1938 |
| Hadoop | 50070 | `http-hadoop-info` |
| Kubernetes API | 6443 | `http-kubernetes-info` — exposed dashboard/kubelet |
| Docker | 2375 | `docker-version` |
| CMS/stack fingerprint | 80, 443 | `http-php-version`, `http-aspnet-debug`, `http-wordpress-enum`, `http-drupal-enum` |

## Misc / automation
```text
nmap --script=default,safe <target>          # safe default set
nmap -sC -sV <target>                         # default scripts + versions
nmap -Pn -n -sS -p- -T4 <target>              # stealth SYN, all ports, no DNS
nmap -p- --min-rate=10000 -T4 <target>        # fast full port sweep
nmap --script vuln <target>                   # known-CVE checks
nmap -p 80,443 --script=http-vuln* <target>
nmap -p 445 --script=smb-vuln* <target>
```

> `brute`, `*-dump-hashes`, `*-backdoor` scripts are active/intrusive — only inside scope, with client approval. Many are noisy and can lock accounts.
