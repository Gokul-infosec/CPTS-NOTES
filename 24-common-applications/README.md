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

### My notes / lab output
Filled in once I run this on a WordPress target.

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
