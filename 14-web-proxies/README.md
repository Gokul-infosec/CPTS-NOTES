# Using Web Proxies

> Status: ✅ Complete (section 15 is the skills-assessment lab to do myself)
> Date started: 2026-10-26
> Date finished: 2026-10-26

## 1. Goal
Sit a **web proxy** (Burp Suite / OWASP ZAP) between the browser and the back-end server as a man-in-the-middle, so I can **capture, inspect, intercept, modify and replay** every HTTP request/response. This is the core tool of web app pentesting — front-end JS validation means nothing once I can edit the raw request on the wire.

## Web proxies — what & why
- A proxy works on web ports (HTTP/80, HTTPS/443) — unlike Wireshark, which sniffs all local traffic. It shows every request the app makes and every response, and lets me intercept one to change its data and watch how the back-end reacts.
- Uses beyond capture/replay: vuln scanning, web fuzzing, crawling, app mapping, request analysis, config testing, code review.

## Burp vs ZAP
| | Burp Suite | OWASP ZAP |
|---|---|---|
| Cost | Free Community + paid Pro/Enterprise | Fully free & open-source (OWASP) |
| Paid-only | Active scanner, fast Intruder, BApp extensions | — none, no throttling/limits |
| Launch | `burpsuite`, app menu, or `java -jar burpsuite.jar` | `zaproxy`, app menu, or `java -jar` |
| Extra | built-in Chromium browser | **HUD** (control ZAP from inside the browser) |
Both need a JRE (bundled in the installer). Both pre-installed on Pwnbox/Parrot/Kali. Community Burp = temporary projects only (disk-save is Pro); with an edu/business email you can trial Burp Pro. Learn both — switch by situation.

**First launch:** Burp → *temporary project* → *Use Burp defaults* → Start Burp. ZAP → choose *no* (don't persist) for a temporary session. Dark theme: Burp `Settings>User interface>Display>theme=dark`; ZAP `Tools>Options>Display>Look and Feel=Flat Dark`.

## Proxy setup
**Easiest — pre-configured browser** (proxy + CA cert already set):
- Burp: `Proxy > Intercept > Open Browser` → routes all traffic through Burp.
- ZAP: the Firefox icon at the end of the top bar.

**Using your own Firefox** (both proxies default to `127.0.0.1:8080`):
- Change the listener port if needed — Burp `Proxy > Proxy settings > Proxy listeners`, ZAP `Tools > Options > Network > Local Servers/Proxies` (match it in Firefox). A port in use = proxy fails to start.
- Use **FoxyProxy** (Firefox extension, pre-installed on Pwnbox): Options → Add → IP `127.0.0.1`, port `8080`, name "Burp"/"ZAP" → Save → click the FoxyProxy icon and select it.

**Install the CA certificate** (so HTTPS routes without per-request warnings):
- Burp: select Burp in FoxyProxy → browse to `http://burp` → download **CA Certificate**.
- ZAP: `Tools > Options > Network > Server Certificates > Save` (Generate makes a new one).
- Firefox: `about:preferences#privacy` → View Certificates → **Authorities** tab → Import the cert → tick **Trust this CA to identify websites** (+ email) → OK.

## Intercepting & manipulating requests
**Intercept toggle:**
- Burp: `Proxy > Intercept` — **on by default**; `Intercept is on/off` toggles. Buttons: **Forward**, **Drop**, Action, Open Browser. (All Firefox traffic is caught — keep hitting Forward until you reach the request to your target IP.)
- ZAP: interception **off by default** (green = passing); toggle with the top-bar button or **Ctrl+B**. Forward one request with the **step** button (right of the red break).
- ZAP **HUD** (Heads-Up Display): controls ZAP from inside the browser — enable via the top-bar button; turn on intercept with the 2nd button on the left pane. On an intercepted request: **Step** (send it, break the next), **Continue** (let the rest through), **Drop**.

**Why it matters:** the page may only allow numbers in a field via front-end JS, but once intercepted I can put anything in the raw request. If the back-end doesn't validate, I can break/exploit it. This is the delivery mechanism for SQLi, command injection, upload bypass, auth bypass, XSS, XXE, error-handling and deserialization testing (the attacks themselves live in their own modules).

**Example (request manipulation):** a ping form sends `POST /ping` with body `ip=1`. Intercept it and change the value to `;ls;` → forward → the response returns the `ls` output instead of ping output = command injected. That's the whole point of the proxy: edit the wire, watch the back-end react.

## Intercepting responses
Catch the server's response *before* the browser renders it — to un-restrict fields, reveal hidden ones, enable disabled buttons.
- **Burp:** `Proxy > Proxy settings > Response interception rules` → enable **Intercept Response**. Turn request intercept on, `Ctrl+Shift+R` full-refresh, Forward the request → the response appears → edit the HTML (e.g. change `type="number"` → `type="text"`, `maxlength="3"` → `maxlength="100"`) → Forward.
- **ZAP:** click **Step** on an intercepted request → it auto-intercepts the response → edit → Continue.
- **ZAP HUD shortcuts** (no intercept needed): **Show/Enable** button (light-bulb, 3rd on left) un-hides + enables fields instantly; **Comments** button (add via `+`) marks positions with HTML comments (hover to read). Burp equivalent: `Proxy > Proxy settings > Response modification rules` → e.g. **Unhide hidden form fields**.

## Automatic modification (match & replace)
Apply a rule to *every* request/response automatically instead of intercepting each one.
- **Burp:** `Proxy > Proxy settings > HTTP match and replace rules > Add`.
  - Spoof User-Agent: Type **Request header**, Match `^User-Agent.*$`, Replace `User-Agent: HackTheBox Agent 1.0`, **Regex = true**.
  - Persist the field unlock: Type **Response body**, Match `type="number"`, Replace `type="text"`, Regex = false (add another for `maxlength`). Now the injection works on a plain refresh, no intercept.
- **ZAP Replacer** (`Ctrl+R` or Options → Replacer → Add): Description, Match Type (e.g. Request Header — adds if absent), Match String (`User-Agent`), Replacement String, Enable. **Initiators** tab scopes where it applies (default: all HTTP(S) messages).

## Repeating requests (Repeater)
Resend any past request with quick edits — no re-intercept. Essential for iterating payloads (e.g. enumerate with different commands).
- **Proxy history:** Burp `Proxy > HTTP history`; ZAP bottom **History** tab / HUD History pane. Both filter + sort. (Both also keep **WebSockets history**.) Burp can show **Original** vs **Edited Request**; ZAP shows only the sent request.
- **Burp Repeater:** select a request → `Ctrl+R` sends it to Repeater → `Ctrl+Shift+R` jumps there → **Send**. Right-click → **Change Request Method** to flip POST/GET without rewriting.
- **ZAP:** right-click → **Open/Resend with Request Editor** → Send (Method drop-down to switch verb). HUD: click a history item → Request Editor → **Replay in Console** or **Replay in Browser**.

## Encoding / decoding
Request data must be correctly URL-encoded or the server errors. Key chars to encode: **space** (ends data), **`&`** (param delimiter), **`#`** (fragment).
- **URL-encode in Burp Repeater:** select text → right-click → `Convert Selection > URL > URL-encode key characters`, or **`Ctrl+U`**. Right-click to enable *URL-encode as you type*. ZAP URL-encodes request data automatically in the background. (Also Full URL-encoding / Unicode URL-encoding for heavy special chars.)
- **Full encoder:** Burp **Decoder** tab (or the **Inspector** in Proxy/Repeater); ZAP **Encoder/Decoder/Hash** (`Ctrl+E`, auto-decodes; "Add New Tab" for custom encoders). Both do **HTML, Unicode, Base64, ASCII hex**.
- **Cookie-tamper example:** a base64 cookie `eyJ1c2VybmFtZSI6Imd1ZXN0IiwgImlzX2FkbWluIjpmYWxzZX0=` decodes to `{"username":"guest", "is_admin":false}` → change to `admin`/`true` → re-encode base64 → use in Repeater. (Burp: chain encoders by picking a method in the output pane.)

## Proxying other tools (CLI & thick clients)
Point any tool at `http://127.0.0.1:8080` to see its web requests in Burp/ZAP. (Proxying slows tools down — only do it when debugging their traffic.)
- **proxychains** — routes any CLI tool through the proxy. Edit `/etc/proxychains.conf`: comment the final `socks4` line, add `http 127.0.0.1 8080`. Use `-q` (quiet):
  ```bash
  proxychains -q curl http://SERVER_IP:PORT
  ```
- **Metasploit** — set the proxy per module with `set PROXIES`:
  ```bash
  msfconsole
  use auxiliary/scanner/http/robots_txt
  set PROXIES HTTP:127.0.0.1:8080
  set RHOST SERVER_IP ; set RPORT PORT ; run
  ```
  Then check the proxy's HTTP history. Same works for scripts/thick clients — set each tool's proxy.

## Burp Intruder (web fuzzer / brute-forcer)
Fuzz pages, dirs, subdomains, parameters, values; alternative to ffuf/gobuster/wfuzz. **Community is throttled to ~1 req/sec** (use it only for short runs); Pro is unthrottled.
1. **Send to Intruder** — in Proxy history, right-click the request → *Send to Intruder* (`Ctrl+I`); open with `Ctrl+Shift+I`. Target is pre-filled.
2. **Positions** — mark the payload point with `§` (select the word, click **Add §**). For dir fuzzing: `GET /§DIRECTORY§/`. Attack types: **Sniper** (1 position), **Cluster Bomb** (multiple positions, all combos), etc. Leave the two blank lines at the request's end.
3. **Payloads:**
   - *Payload type* — **Simple list** (iterate a wordlist), **Runtime file** (stream a huge list line-by-line to save memory), **Character substitution** (permutations), and more.
   - *Configuration* — **Load** a wordlist (e.g. `/opt/useful/seclists/Discovery/Web-Content/common.txt`), or **Add** items manually; combine lists.
   - *Processing* — rules over the wordlist, e.g. **Skip if matches regex** `^\..*$` to skip lines starting with `.`; or append an extension.
   - *Encoding* — toggle payload URL-encoding (leave on).
4. **Settings** — retries/pause = 0; **Grep - Match** to flag responses: enable, Clear, add `200 OK`, uncheck *Exclude HTTP headers* (the status is in the header). **Grep - Extract** pulls a specific part of long responses. **Resource Pool** caps network use for big attacks.
5. **Start Attack** → sort by the `200 OK` column / Status / Length to find hits (e.g. `/admin/` = 200). Verify manually.

Beyond dirs: password brute-force, parameter fuzzing, and **password spraying** against AD-backed apps (OWA, SSL VPN portals, RDS, Citrix, custom AD-auth web apps). Throttling makes Community impractical for large runs — ZAP's fuzzer (next) has no paid tier.

## ZAP Fuzzer
ZAP's fuzzer — **not throttled** (big edge over free Burp Intruder), though fewer features.
1. Capture a request, in History right-click → **Attack > Fuzz**.
2. **Locations** — select the word to fuzz (e.g. `test`) → **Add** (green marker) = payload position.
3. **Payloads** — **Add** → 8 types incl. **File** (your wordlist), **File Fuzzers** (built-in dbs, e.g. dirbuster `directory-list-1.0.txt`), **Numberzz** (number sequences).
4. **Processors** (per-word transform): Base64 encode/decode, MD5/SHA-1/256/512, Prefix/Postfix String, **URL encode/decode**, Script. (Generate Preview to check.)
5. **Options** — Concurrent Scanning Threads (e.g. 20 = fast), retries, **Depth first** (all words on one position before moving on — e.g. all passwords for one user) vs **Breadth first** (one word across all positions first).
6. **Start Fuzzer** → sort by Response code for `200`; also watch **Size Resp. Body** (different page) and **RTT** (time-based SQLi).

## Burp Scanner (Pro-only)
Crawler builds the site structure; Scanner does passive + active vuln scanning.
- **Scope** (`Target > Scope`): `Target > Site map` → right-click target → **Add to scope** (Burp offers to restrict to in-scope only); **Remove from scope** for dangerous items (e.g. logout); advanced scope = regex include/exclude.
- **Start a scan:** right-click a request → **Scan** (configure) or **Passive/Active Scan** (defaults); or **New Scan** on the Dashboard for a target set.
- **Crawler:** *Crawl* maps links/forms (does **not** brute unreferenced pages — use Intruder/Content Discovery for that). *Crawl and Audit* = crawl then scan. Pick a preset via **Select from library** (e.g. "Crawl strategy - fastest"); add creds / record a login for authenticated coverage.
- **Passive scan** — no new requests; analyses already-visited pages for issues (missing headers, DOM-XSS). Gives severity + **Confidence**. Prioritise **High severity + Certain/Firm confidence**.
- **Active scan** — crawl + fuzz to find pages → passive scan → verify each issue with requests → JS analysis → fuzz insertion points for XSS/CmdInj/SQLi etc. Watch progress in **Tasks**/**Logger**; filter Issue activity by **High + Certain**. (Example hit: OS command injection on the `ip` parameter, High/Firm.)
- **Reporting:** `Target > Site map` → right-click target → **Issue > Report issues for this host** → pick format/content → open in a browser. (Appendix data only — never the final client deliverable.)

## ZAP Scanner
- **Spider** (≈ Burp Crawler): History right-click → **Attack > Spider**, or HUD **Spider Start** (2nd right-pane button); add to scope when prompted. Results in **Sites** tree. **Ajax Spider** (3rd button) also follows JS/AJAX-requested links — run after the normal spider for extra coverage.
- **Passive scan** runs automatically as the Spider makes requests — **Alerts** populate (left = current page, right = whole app); see the **Alerts** tab.
- **Active scan** — **Active Scan** button (right pane); runs attacks against all pages/params (slower). High alerts are the ones that compromise the app/server (e.g. Remote OS Command Injection, attack `127.0.0.1&cat /etc/passwd&`, evidence `root:x:0:0`). Replay via HUD / Request Editor.
- **Reporting:** `Report > Generate HTML Report` (also XML / Markdown).

## Extensions
- **Burp BApp Store** (`Extensions > BApp Store`): sort by Popularity; some are Pro-only; some need Jython/Jruby. e.g. **Decoder Improved** (more encoders/hashers). Worth checking: Active Scan++, J2EEScan, Java Deserialization Scanner, Retire.JS, Autorize, CSRF Scanner, JS Link Finder, Software Vulnerability Scanner, CSP Auditor, Headers Analyzer, Error Message Checks.
- **ZAP Marketplace** (`Manage Add-ons > Marketplace`): Release (stable) vs Beta/Alpha builds. e.g. **FuzzDB Files/Offensive** add wordlists — then in File Fuzzers use `fuzzdb > attack > os-cmd-execution` (`command_execution-unix.txt`) for command-injection fuzzing, handy against WAF-protected apps.

> Workflow: start the proxy → open the pre-configured browser (or point Firefox via FoxyProxy + import the CA cert) → browse the target so requests populate the proxy → intercept/edit the raw request (or the response to unlock the UI), or set a match-and-replace rule to automate it → send repeated payloads via Repeater/Request Editor → URL-encode request data and decode/encode cookies and params with Decoder / Encoder-Decoder-Hash → fuzz with Intruder / ZAP Fuzzer, map + scan with the Crawler/Spider + passive/active scanners, extend with BApp Store / Marketplace → proxy CLI tools (proxychains / MSF `set PROXIES`) to inspect their traffic. Front-end validation is irrelevant once you're editing on the wire; scanner reports are appendix data, never the final deliverable.
