# SQL

> **In one line —** you describe the result you want, and the database works out how to get it — which is why the same question can run in 3 milliseconds or 30 seconds.

| | |
|---|---|
| **Full name** | Structured Query Language |
| **Category** | Query Language |
| **Architectural Layer** | Data |
| **Related notes** | [Database Fundamentals](Database%20Fundamentals.md) · [Indexes](Indexes.md) · [PostgreSQL](PostgreSQL.md) · [ORM](ORM.md) · [Database Optimization](Database%20Optimization.md) |

---

## 1. Short Definition

*What is it?*

SQL is a **declarative** language for relational data. You state what you want; the query planner decides how to obtain it — which index to use, which join order, whether to sort or hash.

---

## 2. Purpose

*What is its main purpose?*

To let people ask questions of data without knowing how it is stored, while giving the database freedom to answer them efficiently.

---

## 3. Problem

*What engineering problem does it solve?*

Before SQL, retrieving data meant writing procedural code that walked file structures. Every change of storage layout broke every program. SQL separated the **question** from the **execution strategy**, and that separation has survived fifty years.

---

## 4. The four kinds of statement

```text
DQL   SELECT                              ask questions
DML   INSERT · UPDATE · DELETE            change data
DDL   CREATE · ALTER · DROP               change structure
DCL   GRANT · REVOKE                      change permissions
```

---

## 5. The logical order of a SELECT

Written order and execution order are different, and knowing this explains most confusion.

```text
WRITTEN                EXECUTED
SELECT                 1. FROM      / JOIN
FROM                   2. WHERE     ← filters ROWS
JOIN                   3. GROUP BY
WHERE                  4. HAVING    ← filters GROUPS
GROUP BY               5. SELECT
HAVING                 6. ORDER BY
ORDER BY               7. LIMIT
LIMIT
```

> [!TIP]
> This is why you cannot use a `SELECT` alias in `WHERE` — the alias does not exist yet. And why `WHERE` filters rows while `HAVING` filters aggregated groups.

---

## 6. Joins

```text
INNER JOIN       only rows matching in both tables
LEFT JOIN        all rows from the left, NULLs where the right has none
RIGHT JOIN       the mirror image; rarely used, rewrite as LEFT
FULL OUTER JOIN  everything from both sides
CROSS JOIN       every combination — usually a mistake
```

```text
users        orders
─────        ──────
1 Ana        1 → user 1
2 Ivan       2 → user 1
3 Maja       3 → user 2

INNER JOIN → Ana×2, Ivan×1        (Maja disappears)
LEFT  JOIN → Ana×2, Ivan×1, Maja with NULLs
```

> [!CAUTION]
> A `LEFT JOIN` with a condition on the right table in `WHERE` silently becomes an `INNER JOIN` — the NULL rows are filtered out. Put that condition in the `ON` clause instead. This is one of the most common SQL bugs there is.

---

## 7. Aggregation and window functions

```sql
-- Aggregate: collapses rows into groups
SELECT user_id, COUNT(*), SUM(total)
FROM orders GROUP BY user_id;

-- Window: keeps every row, adds a computed column
SELECT id, total,
       SUM(total) OVER (PARTITION BY user_id ORDER BY created) AS running_total
FROM orders;
```

> [!TIP]
> Window functions are the most underused feature in SQL. Running totals, rankings and "compare each row to its group average" are one line here and a nested mess in application code.

---

## 8. Reading a query plan

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42;
```

```text
Seq Scan on orders  (cost=0.00..18584.00 rows=1 width=64)
   ↑ full table scan — reads every row. Add an index.

Index Scan using idx_orders_user on orders
   ↑ used the index — reads a handful of pages
```

> [!IMPORTANT]
> `EXPLAIN ANALYZE` is the single most valuable SQL skill. It tells you what the database *actually did*, rather than what you assumed. Most "the database is slow" reports are one missing index, visible immediately in the plan.

---

## 9. Real World Example

- **Analytics and reporting** — SQL is still the best tool for aggregating millions of rows.
- **ORMs generate SQL** — and reading the generated queries is how you find N+1 problems.
- **Data warehouses** (BigQuery, Snowflake, Redshift) all chose SQL as their interface, decades after it was declared obsolete.

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Use SQL directly for reporting, complex aggregations, bulk operations and anything where an [ORM](ORM.md) produces poor queries. Write it by hand when the query is the point.

> [!CAUTION]
> Avoid raw SQL for simple CRUD in an application — an ORM is safer and shorter. And **never build SQL by string concatenation with user input**; that is SQL injection, discussed below.

---

## 11. Advantages and Disadvantages

**Advantages**
- Declarative — the optimiser improves without your code changing
- Extremely expressive for set-based operations
- Portable in concept across databases
- Fifty years of tooling, documentation and expertise

**Disadvantages**
- Small differences between dialects break portability in practice
- Easy to write correct-looking queries that scan entire tables
- Poor composability — building queries dynamically gets ugly
- NULL semantics surprise people constantly

---

## 12. NULL — the recurring trap

```sql
NULL = NULL          → NULL (not true!)
WHERE x = NULL       → returns nothing; use IS NULL
COUNT(column)        → skips NULLs
COUNT(*)             → counts rows
NOT IN (1, 2, NULL)  → returns nothing at all
```

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Indexes** | The difference between microseconds and seconds |
| **`SELECT *`** | Transfers columns you do not need; blocks index-only scans |
| **N+1** | Hundreds of round trips instead of one query |
| **Unbounded results** | Memory exhaustion as the table grows |
| **Functions on indexed columns** | `WHERE lower(email) = ...` cannot use a plain index |

---

## 14. Security Considerations

> [!CAUTION]
> **SQL injection remains one of the most damaging vulnerability classes**, and it has exactly one reliable fix.

```python
# ✗ VULNERABLE — the input becomes part of the query
cursor.execute(f"SELECT * FROM users WHERE email = '{email}'")

# ✓ SAFE — the input is data, never code
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
```

- **Parameterised queries always.** Escaping by hand is not a substitute, and neither is validation
- **ORMs parameterise by default** — the danger is `raw()` and string-built fragments
- **Least privilege** — the application user should not own the schema or hold `DROP` rights
- **Never expose database errors** to clients; they reveal table and column names
- **Table and column names cannot be parameterised** — if they must be dynamic, validate against an allow-list

---

## 15. Mental Model

> [!NOTE]
> **SQL is ordering from a menu; procedural code is going into the kitchen.**
>
> You say what you want, not how to cook it. The kitchen (the query planner) may change technique entirely — a new index, a different join strategy — and your order is unchanged. That is also why a badly phrased order can take an hour without you realising why.

---

## 16. Mini Architecture Diagram

```text
SQL text
    ↓
Parser
    ↓
Query planner   ← chooses indexes, join order, scan type
    ↓
Executor
    ↓
Buffer pool (RAM) → disk if needed
    ↓
Rows
```

---

## 17. Complete Request Flow

```text
SELECT o.id, u.name FROM orders o JOIN users u ON u.id = o.user_id
WHERE o.status = 'pending' ORDER BY o.created DESC LIMIT 20;
    ↓
Parsed and validated
    ↓
Planner: index on (status, created)? → index scan, no sort needed
         no index?                   → sequential scan + sort → slow
    ↓
Executor reads pages: buffer pool hit → RAM; miss → disk
    ↓
Join performed (nested loop / hash / merge, chosen by the planner)
    ↓
20 rows returned
    ↓
EXPLAIN ANALYZE shows exactly which of these paths was taken
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> SQL describes the result, not the method — so performance is decided by indexes and the query plan, and `EXPLAIN ANALYZE` is how you see what actually happened.

---

## 19. Common Mistakes

- **String-concatenated queries** — SQL injection
- **`SELECT *`** in application code
- **Conditions on the right table of a `LEFT JOIN` placed in `WHERE`**
- **Missing indexes** on foreign keys and filter columns
- **`NOT IN` with a subquery that can contain NULL**
- **No `LIMIT`** on queries against growing tables
- **Never reading the query plan** before optimising

---

## 20. Open Source Technologies

- **PostgreSQL**, **MySQL**, **SQLite** — the databases
- **DBeaver**, **pgAdmin**, **psql** — clients
- **pg_stat_statements** — find your actual slowest queries
- **sqlfluff** — lint and format SQL
- **DuckDB** — SQL over local files, excellent for analysis

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Run `EXPLAIN ANALYZE` on your slowest query and identify whether it uses an index.
- [ ] Rewrite one piece of application-side aggregation as a window function.
- [ ] Search your codebase for any query built with string formatting.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Application → SQL → parser → planner → executor → buffer pool → disk
```

## 2. Request Flow

```text
Input       a declarative query
    ↓
Processing  parsed, planned against available indexes, executed
    ↓
Output      a result set, produced by a strategy you did not specify
```

## 3. Real-World Usage

**Every modern data warehouse** — BigQuery, Snowflake, Databricks — chose SQL as its interface. A language repeatedly declared obsolete became the standard for systems that did not exist when it was designed.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A declarative language for querying relational data |
| **Why does it exist?** | To separate the question from the execution strategy |
| **Where does it belong?** | Between your application and the database engine |
| **When should I use it?** | For anything relational — always parameterised, never concatenated |
