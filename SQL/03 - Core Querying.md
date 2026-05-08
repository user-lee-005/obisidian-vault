---
title: Core Querying
tags:
  - sql
  - select
  - where
  - filtering
date: 2026-05-08
status: complete
---

# Core Querying

> [!tip] Mental Model
> Every SQL query is a **pipeline of transformations** on sets of rows. Understanding the [[02 - SQL Execution Model]] is essential — your `SELECT` runs *after* `WHERE`, `GROUP BY`, and `HAVING`. This page covers every fundamental clause you'll use to filter, shape, and present data.

**Sample tables used throughout:**

```sql
-- employees (id, name, department_id, salary, hire_date, manager_id, is_active)
-- departments (id, name, location)
-- orders (id, customer_id, order_date, status, total_amount)
-- order_items (id, order_id, product_id, quantity, unit_price)
-- products (id, name, category, price, stock_quantity)
-- customers (id, name, email, city, created_at)
-- shipments (id, order_id, carrier, tracking_number, shipped_date, delivered_date, status)
```

---

## SELECT

### Concept

`SELECT` defines **which columns** (or expressions) appear in your result set. Despite being the first clause you write, it is evaluated **near the end** of the execution pipeline (see [[02 - SQL Execution Model]]).

### Selecting Columns

```sql
-- Select specific columns
SELECT name, salary, hire_date
FROM employees;

-- All columns (avoid in production!)
SELECT *
FROM employees;
```

> [!warning] Avoid `SELECT *` in Production
> - Fetches columns you don't need → more I/O
> - Breaks application code when columns are added/removed/reordered
> - Prevents covering-index optimisation
> - Makes queries harder to reason about
>
> **Use `SELECT *` only** for ad-hoc exploration in a SQL client.

### Expressions in SELECT

You can compute values inline:

```sql
-- Arithmetic
SELECT name,
       salary,
       salary * 12          AS annual_salary,
       salary * 12 * 0.30   AS estimated_tax
FROM employees;

-- String concatenation (standard SQL)
SELECT id,
       name || ' (' || city || ')' AS customer_label
FROM customers;

-- MySQL style
SELECT id,
       CONCAT(name, ' (', city, ')') AS customer_label
FROM customers;
```

### Column Aliases (AS)

```sql
SELECT name AS employee_name,
       salary AS monthly_salary,
       salary * 12 AS annual_salary
FROM employees;
```

> [!warning] When Are Aliases Available?
> Because of the [[02 - SQL Execution Model]], an alias defined in `SELECT` is **not** visible in `WHERE`, `HAVING`, or `GROUP BY` (in standard SQL). It **is** visible in `ORDER BY`.
>
> ```sql
> -- ❌ FAILS in most databases
> SELECT salary * 12 AS annual_salary
> FROM employees
> WHERE annual_salary > 100000;
>
> -- ✅ Use the expression again
> SELECT salary * 12 AS annual_salary
> FROM employees
> WHERE salary * 12 > 100000;
>
> -- ✅ Works in ORDER BY
> SELECT salary * 12 AS annual_salary
> FROM employees
> ORDER BY annual_salary DESC;
> ```
>
> MySQL is an exception — it allows alias references in `HAVING` and `GROUP BY`.

---

## DISTINCT

### Concept

`DISTINCT` eliminates duplicate rows from the result set. It operates **after** `SELECT` evaluates, meaning it de-duplicates the *entire selected row*, not just one column.

### Single Column DISTINCT

```sql
-- Unique departments that have employees
SELECT DISTINCT department_id
FROM employees;
```

### Multi-Column DISTINCT

The combination of all selected columns defines uniqueness:

```sql
-- Unique (department, city) pairs
SELECT DISTINCT d.name AS department, d.location
FROM departments d
JOIN employees e ON e.department_id = d.id;
```

### DISTINCT vs GROUP BY

| Feature | `DISTINCT` | `GROUP BY` |
|---|---|---|
| Purpose | Remove duplicate rows | Group rows for aggregation |
| Aggregates allowed? | No | Yes |
| Performance | Typically same plan | Same or better (depends on engine) |
| Readability | Better for simple de-dup | Better when aggregating |

```sql
-- These produce the same result
SELECT DISTINCT department_id FROM employees;
SELECT department_id FROM employees GROUP BY department_id;
```

### Performance Cost

> [!warning] DISTINCT Is Not Free
> DISTINCT requires a **sort or hash** operation to find duplicates. On large result sets this can be expensive.

### DISTINCT as a Code Smell

> [!danger] If You Need DISTINCT, Ask Why
> Reaching for `DISTINCT` to "fix" duplicate rows is a red flag. The duplicates usually mean:
> 1. A **join is wrong** — you joined on the wrong key or missed a condition → duplicates bleed in
> 2. The **data model has duplicates** — fix the data, not the query
> 3. You **joined a one-to-many** relationship and need `GROUP BY` with aggregation instead
>
> Before adding `DISTINCT`, always ask: *"Why are there duplicates in the first place?"*

---

## WHERE

### Concept

`WHERE` filters rows **before** grouping and aggregation. It operates on the raw table data (or joined result). Only rows where the condition evaluates to `TRUE` are kept — rows where it evaluates to `FALSE` or `UNKNOWN` are discarded.

### Comparison Operators

```sql
SELECT * FROM employees WHERE salary = 5000;
SELECT * FROM employees WHERE salary != 5000;    -- same as <>
SELECT * FROM employees WHERE salary <> 5000;
SELECT * FROM employees WHERE salary > 5000;
SELECT * FROM employees WHERE salary < 5000;
SELECT * FROM employees WHERE salary >= 5000;
SELECT * FROM employees WHERE salary <= 5000;
```

### Logical Operators (AND, OR, NOT)

```sql
-- AND: both must be true
SELECT * FROM employees
WHERE department_id = 3 AND salary > 5000;

-- OR: at least one must be true
SELECT * FROM employees
WHERE department_id = 3 OR department_id = 5;

-- NOT: negates the condition
SELECT * FROM employees
WHERE NOT is_active;
```

### Operator Precedence

> [!danger] AND Binds Tighter Than OR
> This is the single most common logical bug in SQL queries.

```sql
-- ❌ Bug! AND evaluates first, so this means:
-- (department_id = 3 AND salary > 5000) OR department_id = 5
SELECT * FROM employees
WHERE department_id = 3 OR department_id = 5 AND salary > 5000;

-- ✅ Use parentheses to make intent explicit
SELECT * FROM employees
WHERE (department_id = 3 OR department_id = 5) AND salary > 5000;
```

**Operator precedence table (highest to lowest):**

| Priority | Operator |
|----------|----------|
| 1 (highest) | `()` parentheses |
| 2 | `NOT` |
| 3 | `AND` |
| 4 (lowest) | `OR` |

> [!tip] Rule of Thumb
> **Always use parentheses** when mixing `AND` and `OR`. It costs nothing and prevents bugs.

---

## ORDER BY

### Concept

`ORDER BY` sorts the result set. It is the **last** clause evaluated (before `LIMIT`). Without `ORDER BY`, SQL makes **no guarantee** about row order — even if results *look* ordered, that's an implementation detail, not a contract.

### ASC vs DESC

```sql
-- Ascending (default)
SELECT name, salary FROM employees ORDER BY salary ASC;

-- Descending
SELECT name, salary FROM employees ORDER BY salary DESC;
```

### Multi-Column Ordering

```sql
-- Sort by department first, then by salary descending within each department
SELECT name, department_id, salary
FROM employees
ORDER BY department_id ASC, salary DESC;
```

### Ordering by Expression

```sql
-- Order by computed annual salary
SELECT name, salary
FROM employees
ORDER BY salary * 12 DESC;
```

### Ordering by Column Position

```sql
-- Refers to the 2nd column in SELECT
SELECT name, salary FROM employees ORDER BY 2 DESC;
```

> [!warning] Avoid Positional ORDER BY
> If someone adds or reorders columns in the `SELECT`, the sort breaks silently. Always use column names or aliases.

### NULL Ordering Behavior

```sql
-- PostgreSQL: control NULL position
SELECT name, manager_id
FROM employees
ORDER BY manager_id ASC NULLS LAST;

-- Default varies by database:
-- PostgreSQL: NULLS LAST for ASC, NULLS FIRST for DESC
-- MySQL/SQL Server: NULLS treated as smallest value (appear first in ASC)
-- Oracle: NULLS LAST for ASC by default
```

| Database | ASC Default | DESC Default |
|----------|-------------|--------------|
| PostgreSQL | NULLS LAST | NULLS FIRST |
| MySQL | NULLS FIRST | NULLS LAST |
| SQL Server | NULLS FIRST | NULLS LAST |
| Oracle | NULLS LAST | NULLS FIRST |

### Performance Implications

> [!warning] Sorts Are Expensive
> `ORDER BY` requires either:
> 1. An **index** that already stores data in the desired order (fast)
> 2. A **sort operation** in memory or on disk (slow for large datasets)
>
> If you're paginating, consider whether you *really* need a global sort, or if keyset pagination with an indexed column is sufficient.

---

## LIMIT and OFFSET

### Basic Pagination

```sql
-- First 10 rows
SELECT * FROM orders ORDER BY order_date DESC LIMIT 10;

-- Skip 20 rows, take 10 (page 3)
SELECT * FROM orders ORDER BY order_date DESC LIMIT 10 OFFSET 20;

-- SQL Server syntax
SELECT TOP 10 * FROM orders ORDER BY order_date DESC;

-- SQL Server with OFFSET (requires ORDER BY)
SELECT * FROM orders
ORDER BY order_date DESC
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
```

### LIMIT Without ORDER BY Is Non-Deterministic

> [!danger] Always Pair LIMIT With ORDER BY
> Without `ORDER BY`, the database can return *any* rows in *any* order. Running the same query twice may return different results.
>
> ```sql
> -- ❌ Which 10 rows? Nobody knows.
> SELECT * FROM orders LIMIT 10;
>
> -- ✅ Deterministic
> SELECT * FROM orders ORDER BY id LIMIT 10;
> ```

### Why OFFSET Is Slow

```sql
-- To get page 1000 (rows 9991–10000), the DB must:
-- 1. Sort ALL rows
-- 2. Skip 9990 rows
-- 3. Return 10
SELECT * FROM orders ORDER BY order_date DESC LIMIT 10 OFFSET 9990;
-- This gets slower linearly as OFFSET grows!
```

> [!tip] Keyset Pagination (Better Alternative)
> Instead of OFFSET, use the last seen value as a cursor:
>
> ```sql
> -- First page
> SELECT * FROM orders
> ORDER BY order_date DESC, id DESC
> LIMIT 10;
>
> -- Next page: use the last row's values as the cursor
> SELECT * FROM orders
> WHERE (order_date, id) < ('2025-12-01', 9950)
> ORDER BY order_date DESC, id DESC
> LIMIT 10;
> ```
> This uses the index and runs in constant time regardless of page number.

---

## IN Operator

### IN with Value Lists

```sql
SELECT * FROM employees
WHERE department_id IN (1, 3, 5);

-- Equivalent to:
SELECT * FROM employees
WHERE department_id = 1 OR department_id = 3 OR department_id = 5;
```

### IN with Subqueries

```sql
-- Employees in departments located in 'Chennai'
SELECT * FROM employees
WHERE department_id IN (
    SELECT id FROM departments WHERE location = 'Chennai'
);
```

### NOT IN Pitfalls with NULL

> [!danger] NOT IN with NULL Is a Silent Data Killer
> This is one of the most dangerous traps in SQL. If the subquery (or value list) contains **any NULL**, `NOT IN` returns **zero rows**.
>
> **Why?** Because `x NOT IN (1, 2, NULL)` expands to:
> ```
> x != 1 AND x != 2 AND x != NULL
> ```
> `x != NULL` evaluates to `UNKNOWN`. `TRUE AND UNKNOWN = UNKNOWN`. The entire expression is `UNKNOWN`, so the row is excluded.
>
> ```sql
> -- ❌ Returns ZERO rows if any manager_id is NULL!
> SELECT * FROM employees
> WHERE id NOT IN (SELECT manager_id FROM employees);
>
> -- ✅ Fix: filter out NULLs
> SELECT * FROM employees
> WHERE id NOT IN (
>     SELECT manager_id FROM employees WHERE manager_id IS NOT NULL
> );
>
> -- ✅ Better: use NOT EXISTS instead
> SELECT e.*
> FROM employees e
> WHERE NOT EXISTS (
>     SELECT 1 FROM employees m WHERE m.manager_id = e.id
> );
> ```

### IN vs EXISTS Comparison

| Feature | `IN` | `EXISTS` |
|---------|------|----------|
| NULL-safe? | ❌ `NOT IN` breaks with NULLs | ✅ Always safe |
| Short-circuits? | No (evaluates full list) | Yes (stops at first match) |
| Correlated subquery? | Typically not | Yes |
| Best for | Small, known value lists | Large subqueries, NULL-safe anti-joins |
| Readability | Simpler for value lists | Better for correlated checks |

> [!tip] Rule of Thumb
> Use `IN` for small value lists. Use `EXISTS` / `NOT EXISTS` for subqueries, especially when NULLs could appear. See [[05 - EXISTS and NOT EXISTS]].

---

## BETWEEN

### Inclusive Range

```sql
-- Salary between 3000 and 7000 (inclusive of both endpoints)
SELECT name, salary FROM employees
WHERE salary BETWEEN 3000 AND 7000;

-- Equivalent to:
SELECT name, salary FROM employees
WHERE salary >= 3000 AND salary <= 7000;
```

### BETWEEN with Dates

> [!danger] BETWEEN and Timestamps — A Common Trap
> `BETWEEN '2025-01-01' AND '2025-01-31'` includes `'2025-01-31 00:00:00'` but **excludes** `'2025-01-31 15:30:00'` because `15:30 > 00:00`.
>
> ```sql
> -- ❌ Misses orders placed on Jan 31 after midnight
> SELECT * FROM orders
> WHERE order_date BETWEEN '2025-01-01' AND '2025-01-31';
>
> -- ✅ Use half-open range for timestamps
> SELECT * FROM orders
> WHERE order_date >= '2025-01-01'
>   AND order_date < '2025-02-01';
> ```
> **Always use `>= start AND < next_period`** for timestamp ranges.

### NOT BETWEEN

```sql
-- Employees outside the salary range
SELECT name, salary FROM employees
WHERE salary NOT BETWEEN 3000 AND 7000;
```

---

## LIKE

### Pattern Matching

| Wildcard | Meaning | Example |
|----------|---------|---------|
| `%` | Any sequence of characters (including empty) | `'A%'` matches `'Alice'`, `'A'` |
| `_` | Exactly one character | `'A_e'` matches `'Ace'`, `'Axe'` |

```sql
-- Names starting with 'S'
SELECT * FROM customers WHERE name LIKE 'S%';

-- Names ending with 'son'
SELECT * FROM customers WHERE name LIKE '%son';

-- Names containing 'kumar'
SELECT * FROM customers WHERE name LIKE '%kumar%';

-- Exactly 5-character names
SELECT * FROM customers WHERE name LIKE '_____';
```

### Escape Characters

```sql
-- Find products with literal '%' in the name
SELECT * FROM products WHERE name LIKE '%\%%' ESCAPE '\';

-- Find products with literal '_' in the name
SELECT * FROM products WHERE name LIKE '%\_%' ESCAPE '\';
```

### Performance: Leading Wildcards Kill Indexes

> [!danger] `LIKE '%something'` Cannot Use an Index
> An index on `name` stores values in sorted order (like a phone book). A leading `%` means "starts with anything" — the database must **scan every row**.
>
> ```sql
> -- ❌ Full table scan (leading wildcard)
> SELECT * FROM customers WHERE name LIKE '%kumar%';
>
> -- ✅ Can use index (prefix search)
> SELECT * FROM customers WHERE name LIKE 'Kumar%';
> ```
> For substring/full-text search, use **full-text indexes** (PostgreSQL `tsvector`, MySQL `FULLTEXT`, Elasticsearch).

### Case-Insensitive Matching

```sql
-- PostgreSQL: ILIKE
SELECT * FROM customers WHERE name ILIKE '%kumar%';

-- MySQL: LIKE is case-insensitive by default (depends on collation)

-- Portable approach (but can't use index)
SELECT * FROM customers WHERE LOWER(name) LIKE '%kumar%';
```

---

## NULL Handling

> [!danger] This Is the Most Important Section in This Note
> NULL-related bugs are responsible for more production incidents than almost any other SQL mistake. Master this section thoroughly.

### What Is NULL?

NULL is **not** a value. It is the **absence of information** — it means "unknown" or "not applicable."

- NULL ≠ 0
- NULL ≠ empty string `''`
- NULL ≠ FALSE
- NULL = "I don't know"

### Three-Valued Logic

Standard logic has two values: TRUE and FALSE. SQL has **three**: TRUE, FALSE, and UNKNOWN. Any comparison involving NULL produces UNKNOWN.

```sql
-- All of these evaluate to UNKNOWN, not TRUE or FALSE:
SELECT 5 = NULL;      -- UNKNOWN
SELECT 5 != NULL;     -- UNKNOWN
SELECT NULL = NULL;   -- UNKNOWN (!)
SELECT NULL != NULL;  -- UNKNOWN
SELECT NULL > 0;      -- UNKNOWN
```

### IS NULL vs = NULL

```sql
-- ❌ NEVER WORKS — evaluates to UNKNOWN, returns no rows
SELECT * FROM employees WHERE manager_id = NULL;

-- ✅ Correct way to check for NULL
SELECT * FROM employees WHERE manager_id IS NULL;

-- ❌ NEVER WORKS
SELECT * FROM employees WHERE manager_id != NULL;

-- ✅ Correct
SELECT * FROM employees WHERE manager_id IS NOT NULL;
```

### Truth Tables for AND / OR with NULL

**AND Truth Table:**

| A | B | A AND B |
|---|---|---------|
| TRUE | TRUE | TRUE |
| TRUE | FALSE | FALSE |
| TRUE | UNKNOWN | UNKNOWN |
| FALSE | TRUE | FALSE |
| FALSE | FALSE | FALSE |
| FALSE | UNKNOWN | **FALSE** |
| UNKNOWN | TRUE | UNKNOWN |
| UNKNOWN | FALSE | **FALSE** |
| UNKNOWN | UNKNOWN | UNKNOWN |

> [!tip] Key insight: `FALSE AND UNKNOWN = FALSE` (FALSE "dominates" AND)

**OR Truth Table:**

| A | B | A OR B |
|---|---|--------|
| TRUE | TRUE | TRUE |
| TRUE | FALSE | TRUE |
| TRUE | UNKNOWN | **TRUE** |
| FALSE | TRUE | TRUE |
| FALSE | FALSE | FALSE |
| FALSE | UNKNOWN | UNKNOWN |
| UNKNOWN | TRUE | **TRUE** |
| UNKNOWN | FALSE | UNKNOWN |
| UNKNOWN | UNKNOWN | UNKNOWN |

> [!tip] Key insight: `TRUE OR UNKNOWN = TRUE` (TRUE "dominates" OR)

### NULL Behavior Summary Table

| Context | NULL Behavior |
|---------|---------------|
| `WHERE x = NULL` | Never matches (UNKNOWN) |
| `WHERE x IS NULL` | Matches NULL rows ✅ |
| `COUNT(*)` | Counts all rows including NULLs |
| `COUNT(column)` | **Skips** NULL values |
| `SUM(column)` | Ignores NULLs (returns NULL if all NULL) |
| `AVG(column)` | Ignores NULLs (different from treating as 0!) |
| `MIN/MAX(column)` | Ignores NULLs |
| `DISTINCT` | Treats all NULLs as one group |
| `GROUP BY` | Treats all NULLs as one group |
| `ORDER BY ASC` | Position depends on database (see above) |
| `UNION` | Treats NULLs as equal for deduplication |
| `IN (NULL)` | `x IN (NULL)` → UNKNOWN |
| `NOT IN (…, NULL)` | **Always UNKNOWN** → returns no rows! |
| `CASE WHEN NULL` | Falls through to ELSE |
| `JOIN ON a = b` | NULL = NULL → does not match |

### COALESCE

Returns the first non-NULL argument:

```sql
-- Replace NULL manager_id with 'No Manager'
SELECT name,
       COALESCE(manager_id, 0) AS effective_manager_id
FROM employees;

-- Chain multiple fallbacks
SELECT COALESCE(phone, mobile, email, 'No Contact') AS contact_info
FROM customers;
```

### NULLIF

Returns NULL if both arguments are equal; otherwise returns the first argument. Useful to prevent division by zero:

```sql
-- Avoid division by zero
SELECT total_amount / NULLIF(quantity, 0) AS unit_price
FROM order_items;

-- Without NULLIF, division by zero raises an error
```

### IFNULL / NVL

```sql
-- MySQL: IFNULL
SELECT name, IFNULL(manager_id, 0) FROM employees;

-- Oracle: NVL
SELECT name, NVL(manager_id, 0) FROM employees;

-- Standard SQL: use COALESCE (portable)
SELECT name, COALESCE(manager_id, 0) FROM employees;
```

### How Beginners Think vs How Strong SQL Engineers Think

> [!question] How Beginners Think About NULL
> - NULL is like zero or an empty string
> - `WHERE x != 5` should return rows where `x` is NULL
> - `NOT IN` works fine with any subquery
> - `COUNT(column)` and `COUNT(*)` are the same thing

> [!tip] How Strong SQL Engineers Think About NULL
> - NULL means "I don't know the answer" — it's the absence of information
> - `WHERE x != 5` does NOT return NULLs — they're UNKNOWN, not FALSE
> - I always use `NOT EXISTS` instead of `NOT IN` with subqueries
> - `COUNT(*)` counts rows; `COUNT(column)` counts non-NULL values — they serve different purposes
> - I always ask: "What happens when this column is NULL?"
> - I use `COALESCE` proactively, not reactively after bugs appear

---

## CASE Expressions

### Concept

`CASE` is SQL's conditional expression — the equivalent of `if/else` in programming languages. It returns a **value**, so it can be used anywhere an expression is valid: `SELECT`, `WHERE`, `ORDER BY`, `GROUP BY`, inside aggregate functions.

### Simple CASE

Compares one expression against multiple values:

```sql
SELECT name,
       CASE department_id
           WHEN 1 THEN 'Engineering'
           WHEN 2 THEN 'Sales'
           WHEN 3 THEN 'Operations'
           ELSE 'Other'
       END AS department_name
FROM employees;
```

### Searched CASE

Evaluates arbitrary boolean conditions (more flexible):

```sql
SELECT name, salary,
       CASE
           WHEN salary >= 10000 THEN 'Senior'
           WHEN salary >= 6000  THEN 'Mid-Level'
           WHEN salary >= 3000  THEN 'Junior'
           ELSE 'Intern'
       END AS salary_band
FROM employees;
```

> [!warning] CASE Evaluates Top-Down
> The first matching `WHEN` wins. Order matters! If an employee has salary 12000, only 'Senior' is returned — it doesn't fall through to 'Mid-Level'.

### CASE in SELECT

```sql
-- Map order status codes to labels
SELECT id,
       order_date,
       CASE status
           WHEN 'P' THEN 'Pending'
           WHEN 'S' THEN 'Shipped'
           WHEN 'D' THEN 'Delivered'
           WHEN 'C' THEN 'Cancelled'
           ELSE 'Unknown'
       END AS status_label,
       total_amount
FROM orders;
```

### CASE in WHERE

```sql
-- Different filter logic based on a parameter
SELECT * FROM orders
WHERE CASE
          WHEN @include_cancelled = 1 THEN 1
          ELSE CASE WHEN status != 'C' THEN 1 ELSE 0 END
      END = 1;

-- Simpler: use CASE to handle NULL comparisons
SELECT * FROM employees
WHERE CASE WHEN manager_id IS NULL THEN 'CEO' ELSE 'Staff' END = 'CEO';
```

### CASE in ORDER BY

```sql
-- Custom sort order: Active first, then Inactive, then Terminated
SELECT name, status
FROM employees
ORDER BY CASE status
             WHEN 'Active' THEN 1
             WHEN 'Inactive' THEN 2
             WHEN 'Terminated' THEN 3
             ELSE 4
         END;
```

### CASE in Aggregations (Conditional Aggregation)

This is a powerful technique — preview for [[06 - GROUP BY and Aggregation]]:

```sql
-- Count orders by status in a single query (pivot-style)
SELECT
    COUNT(CASE WHEN status = 'P' THEN 1 END) AS pending_count,
    COUNT(CASE WHEN status = 'S' THEN 1 END) AS shipped_count,
    COUNT(CASE WHEN status = 'D' THEN 1 END) AS delivered_count,
    COUNT(CASE WHEN status = 'C' THEN 1 END) AS cancelled_count
FROM orders;

-- SUM with conditions
SELECT department_id,
       SUM(CASE WHEN is_active THEN salary ELSE 0 END) AS active_payroll,
       SUM(CASE WHEN NOT is_active THEN salary ELSE 0 END) AS inactive_payroll
FROM employees
GROUP BY department_id;
```

### CASE vs IF Functions

| Feature | `CASE` (Standard SQL) | `IF()` (MySQL) | `IIF()` (SQL Server) |
|---------|----------------------|----------------|----------------------|
| Portable? | ✅ Yes | ❌ MySQL only | ❌ SQL Server only |
| Multiple branches? | ✅ Yes | ❌ Only 2 (true/false) | ❌ Only 2 |
| Use in all clauses? | ✅ Yes | ✅ Yes | ✅ Yes |

> [!tip] Always prefer `CASE` — it's standard SQL and works everywhere.

### Nested CASE

```sql
SELECT name, department_id, salary,
       CASE
           WHEN department_id = 1 THEN
               CASE
                   WHEN salary > 8000 THEN 'Senior Engineer'
                   ELSE 'Junior Engineer'
               END
           WHEN department_id = 2 THEN
               CASE
                   WHEN salary > 7000 THEN 'Senior Sales'
                   ELSE 'Junior Sales'
               END
           ELSE 'Other'
       END AS role_title
FROM employees;
```

> [!warning] If you have deeply nested CASE statements, consider:
> - A lookup/mapping table instead
> - Application-layer logic
> - A view that encapsulates the mapping

### Common Patterns

**Status Mapping:**
```sql
SELECT id,
       CASE status WHEN 'A' THEN 'Active' WHEN 'I' THEN 'Inactive' END AS status_text
FROM employees;
```

**Bucketing / Binning:**
```sql
SELECT CASE
           WHEN total_amount < 100 THEN 'Small'
           WHEN total_amount < 1000 THEN 'Medium'
           ELSE 'Large'
       END AS order_size,
       COUNT(*) AS order_count
FROM orders
GROUP BY CASE
             WHEN total_amount < 100 THEN 'Small'
             WHEN total_amount < 1000 THEN 'Medium'
             ELSE 'Large'
         END;
```

**Pivoting (Cross-tab):**
```sql
SELECT customer_id,
       SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 1 THEN total_amount ELSE 0 END) AS jan_total,
       SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 2 THEN total_amount ELSE 0 END) AS feb_total,
       SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 3 THEN total_amount ELSE 0 END) AS mar_total
FROM orders
GROUP BY customer_id;
```

---

## Comprehensive Practice Section

### Exercise 1 — Basic SELECT
Write a query to list every employee's name and annual salary (salary × 12), aliased as `annual_salary`, sorted by annual salary descending.

```sql
SELECT name, salary * 12 AS annual_salary
FROM employees
ORDER BY annual_salary DESC;
```

### Exercise 2 — Filtering with WHERE
Find all orders placed in 2025 with a total amount over 500 that are not cancelled.

```sql
SELECT *
FROM orders
WHERE order_date >= '2025-01-01'
  AND order_date < '2026-01-01'
  AND total_amount > 500
  AND status != 'C';
```

### Exercise 3 — DISTINCT
Find the distinct cities where customers are located.

```sql
SELECT DISTINCT city
FROM customers
ORDER BY city;
```

### Exercise 4 — IN and NOT IN
Find all products in the 'Electronics' or 'Furniture' categories, excluding product IDs 101, 102.

```sql
SELECT *
FROM products
WHERE category IN ('Electronics', 'Furniture')
  AND id NOT IN (101, 102);
```

### Exercise 5 — NULL Handling
List all employees who do **not** have a manager. Then list all employees who **do** have a manager.

```sql
-- No manager (top-level / CEO)
SELECT * FROM employees WHERE manager_id IS NULL;

-- Has a manager
SELECT * FROM employees WHERE manager_id IS NOT NULL;
```

### Exercise 6 — BETWEEN with Dates
Find all shipments that were shipped in March 2025. Handle the timestamp issue correctly.

```sql
-- ✅ Half-open range (correct for timestamps)
SELECT *
FROM shipments
WHERE shipped_date >= '2025-03-01'
  AND shipped_date < '2025-04-01';
```

### Exercise 7 — LIKE Pattern Matching
Find all customers whose email is from a Gmail domain.

```sql
SELECT * FROM customers
WHERE email LIKE '%@gmail.com';
```

### Exercise 8 — CASE Expression
Write a query that categorises products by stock level: 'Out of Stock' (0), 'Low' (1–10), 'Medium' (11–50), 'High' (51+).

```sql
SELECT name, stock_quantity,
       CASE
           WHEN stock_quantity = 0 THEN 'Out of Stock'
           WHEN stock_quantity <= 10 THEN 'Low'
           WHEN stock_quantity <= 50 THEN 'Medium'
           ELSE 'High'
       END AS stock_level
FROM products
ORDER BY stock_quantity;
```

### Exercise 9 — COALESCE and NULLIF
Calculate revenue per unit for each order item. If quantity is 0, return NULL instead of an error.

```sql
SELECT id,
       order_id,
       product_id,
       quantity,
       unit_price,
       (unit_price * quantity) AS line_total,
       unit_price * quantity / NULLIF(quantity, 0) AS revenue_per_unit
FROM order_items;
```

### Exercise 10 — Conditional Aggregation
Write a single query that returns the count of orders in each status (Pending, Shipped, Delivered, Cancelled) as separate columns.

```sql
SELECT
    COUNT(CASE WHEN status = 'P' THEN 1 END) AS pending,
    COUNT(CASE WHEN status = 'S' THEN 1 END) AS shipped,
    COUNT(CASE WHEN status = 'D' THEN 1 END) AS delivered,
    COUNT(CASE WHEN status = 'C' THEN 1 END) AS cancelled,
    COUNT(*) AS total
FROM orders;
```

### Debug This Query — Exercise 11
What's wrong with this query? Fix it.

```sql
-- ❌ Buggy
SELECT name, salary * 12 AS annual_salary
FROM employees
WHERE annual_salary > 100000
ORDER BY annual_salary;
```

**Bug:** `annual_salary` alias is used in `WHERE`, but aliases aren't available there (see [[02 - SQL Execution Model]]).

```sql
-- ✅ Fixed
SELECT name, salary * 12 AS annual_salary
FROM employees
WHERE salary * 12 > 100000
ORDER BY annual_salary;
```

### Debug This Query — Exercise 12
Why does this query return zero rows even though there are employees without a manager?

```sql
-- ❌ Returns 0 rows
SELECT * FROM employees
WHERE manager_id = NULL;
```

**Bug:** `= NULL` always evaluates to UNKNOWN. Use `IS NULL`.

```sql
-- ✅ Fixed
SELECT * FROM employees
WHERE manager_id IS NULL;
```

### Rewrite This Better — Exercise 13
Simplify this query:

```sql
-- ❌ Verbose
SELECT * FROM employees
WHERE department_id = 1
   OR department_id = 3
   OR department_id = 5
   OR department_id = 7;
```

```sql
-- ✅ Better
SELECT * FROM employees
WHERE department_id IN (1, 3, 5, 7);
```

### Rewrite This Better — Exercise 14
This query is slow. Why? Rewrite it.

```sql
-- ❌ Slow — leading wildcard prevents index use
SELECT * FROM customers
WHERE name LIKE '%Technologies%';
```

```sql
-- ✅ If prefix search is acceptable:
SELECT * FROM customers
WHERE name LIKE 'Technologies%';

-- ✅ If substring search is required, use full-text search:
-- PostgreSQL:
SELECT * FROM customers
WHERE to_tsvector('english', name) @@ to_tsquery('Technologies');
```

---

## Interview Questions

### Q1: What's the difference between `WHERE` and `HAVING`?
**Answer:** `WHERE` filters rows **before** grouping; `HAVING` filters groups **after** aggregation. `WHERE` cannot reference aggregate functions. See [[06 - GROUP BY and Aggregation]].

### Q2: What does `SELECT * FROM employees WHERE department_id NOT IN (SELECT manager_id FROM employees)` return if some `manager_id` values are NULL?
**Answer:** **Zero rows.** `NOT IN` with any NULL in the subquery result makes the entire condition UNKNOWN for every row. Fix: add `WHERE manager_id IS NOT NULL` in the subquery, or use `NOT EXISTS`.

### Q3: What is the difference between `COUNT(*)` and `COUNT(column_name)`?
**Answer:** `COUNT(*)` counts all rows. `COUNT(column_name)` counts only rows where `column_name` is not NULL.

### Q4: Can you use a column alias in the WHERE clause?
**Answer:** No, in standard SQL. `WHERE` is evaluated before `SELECT`, so the alias doesn't exist yet. MySQL is an exception for `HAVING` but not `WHERE`.

### Q5: What's wrong with `ORDER BY 1, 2`?
**Answer:** Positional ordering is fragile — if `SELECT` columns change, the sort silently breaks. Use explicit column names or aliases instead.

### Q6: Explain the difference between `BETWEEN '2025-01-01' AND '2025-01-31'` and `>= '2025-01-01' AND < '2025-02-01'` for a `TIMESTAMP` column.
**Answer:** `BETWEEN` is inclusive on both ends. For timestamps, `BETWEEN ... AND '2025-01-31'` only matches `'2025-01-31 00:00:00.000'`, missing everything later on Jan 31. The half-open range (`>= AND <`) captures the entire last day.

### Q7: What does `NULL = NULL` evaluate to?
**Answer:** `UNKNOWN` (not TRUE, not FALSE). This is why `WHERE a = b` never matches when both are NULL, and why joins on nullable columns skip NULL pairs.

### Q8: Write a query to find employees who earn more than the average salary without using a subquery in FROM.
**Answer:**
```sql
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

### Q9: How can DISTINCT indicate a code problem?
**Answer:** DISTINCT often masks a bad join or missing `GROUP BY`. If you're getting duplicates, the root cause is usually incorrect join cardinality. See [[04 - Joins]].

---

> [!example] Next Steps
> - [[04 - Joins]] — Combining data from multiple tables
> - [[05 - EXISTS and NOT EXISTS]] — NULL-safe alternatives to IN/NOT IN
> - [[06 - GROUP BY and Aggregation]] — Grouping and summarising data
> - [[02 - SQL Execution Model]] — Understanding clause evaluation order
