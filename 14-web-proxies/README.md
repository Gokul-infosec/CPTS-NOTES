# Using Web Proxies

> Status: 🟨 In progress (sections 1–8 of 15 filled)
> Date started: 2026-10-26
> Date finished: 

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

> Workflow: start the proxy → open the pre-configured browser (or point Firefox via FoxyProxy + import the CA cert) → browse the target so requests populate the proxy → intercept/edit the raw request (or the response to unlock the UI), or set a match-and-replace rule to automate it → send repeated payloads via Repeater/Request Editor → URL-encode request data and decode/encode cookies and params with Decoder / Encoder-Decoder-Hash. Front-end validation is irrelevant once you're editing on the wire. (Sections 9–15 — Comparer, scanner/crawler, Intruder-style fuzzing, extensions — to be added.)
