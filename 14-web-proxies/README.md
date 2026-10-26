# Using Web Proxies

> Status: 🟨 In progress (sections 1–4 of 15 filled)
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

> Workflow: start the proxy → open the pre-configured browser (or point Firefox via FoxyProxy + import the CA cert) → browse the target so requests populate the proxy → intercept, edit the raw request, forward → observe the response. Front-end validation is irrelevant once you're editing on the wire. (Sections 5–15 — Repeater, Intruder, Decoder/Comparer, scanner, extensions — to be added.)
