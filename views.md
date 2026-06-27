# SQL Views — Revision Notes

## 1. What is a View?

A **view** is a virtual table based on the result of a SQL query.

- It has **no physical storage** of its own.
- It stores the **query**, not the data.
- Every time you query a view, the database re-runs the underlying query behind the scenes.

```sql
CREATE VIEW top_students AS
SELECT name, score
FROM students
WHERE score > 80;

SELECT * FROM top_students;
```

This silently becomes:
```sql
SELECT name, score FROM students WHERE score > 80;
```

---

## 2. Why "Virtual Table"?

It's called virtual because it **looks and behaves** like a table (you can `SELECT`, filter, `JOIN` it) but doesn't **store** anything itself. The data physically lives in the base table — the view is just a saved lookup pointing at it.

---

## 3. Behavior When the Base Table Changes

| Action on base table | Effect on view |
|---|---|
| `UPDATE` rows | View reflects changes instantly |
| `DELETE` rows | View reflects changes instantly |
| `DROP TABLE` (delete whole table) | View becomes broken/invalid (dangling) until the table is recreated or the view is dropped |

> View definitions aren't auto-deleted when the base table is dropped — they just start erroring out (`relation does not exist`).

---

## 4. Why Use Views? (Real-World Use Cases)

1. **Hiding complexity** — wrap multi-join, multi-filter logic into one reusable name.
2. **Security** — expose only safe columns (e.g. hide `salary` from a view on `employees`); grant access to the view, not the table.
3. **Backward compatibility** — act as a translation layer when underlying schema changes, so dependent apps don't break.
4. **Simplifying BI/reporting** — tools like Tableau/Looker query clean, pre-shaped views instead of raw messy tables.

**Important caveat:** a plain view does **not** improve performance. It re-executes the full query every time — it is not a cache.

---

## 5. Updatable Views

You *can* sometimes `INSERT`/`UPDATE` through a view, but only if it is based on:
- A **single table** (no joins, no `UNION`)
- No `GROUP BY`, `DISTINCT`, aggregate functions, or `HAVING`
- All `NOT NULL` columns without defaults are included (for `INSERT` to succeed)

If a view joins multiple tables or aggregates data → it's generally **read-only**, since the database can't determine which row/table an update should target.

---

## 6. `WITH CHECK OPTION`

```sql
CREATE VIEW high_earners AS
SELECT * FROM employees WHERE salary > 100000
WITH CHECK OPTION;
```

**The rule:** blocks any `INSERT`/`UPDATE` made *through the view* if the resulting row would no longer satisfy the view's `WHERE` clause.

It does **not** care about the direction of change (higher/lower) — only whether the final row still passes the filter.

| Update via view | Passes `salary > 100000`? | Result |
|---|---|---|
| `salary = 50000` | No | **Blocked** |
| `salary = 2000000` | Yes | **Allowed** |
| `salary = 100000` (exact) | No (`>` is strict) | **Blocked** — classic boundary trap |

### Why does this matter if the table itself isn't broken?

Without `CHECK OPTION`, a row can vanish from the view as a side effect of an edit made *through that same view*. This matters when the view is used as an **access boundary** (e.g. a payroll clerk only permitted to touch `high_earners`). Letting an edit silently push a row outside the view's own restriction defeats the purpose of using the view to control what that user/tool can touch — and the change can go unnoticed since the user may not have access to the full base table to see what really happened.

If the view is just a convenience filter (not a security boundary), this doesn't matter much — letting rows leave the view naturally is fine.

---

## 7. View vs Materialized View

| | View | Materialized View |
|---|---|---|
| Storage | None (virtual) | Stores actual data |
| Speed | Slower (recomputes every time) | Faster (precomputed) |
| Freshness | Always live/current | Stale until refreshed |
| Indexable | No (no physical storage) | Yes |
| Use case | Simplicity, security, abstraction | Performance, heavy reporting |

---

## 8. Interview One-Liners

- "A view doesn't store data, a table does."
- "Views are for convenience and security, not performance."
- "Materialized views trade freshness for speed."
- "A view is updatable only if it maps unambiguously back to one base table row."
- "`WITH CHECK OPTION` protects the view's own filter from being violated by edits made through it."

---

## 9. Practice Queries

Sample schema used for all examples below:

```sql
CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    department VARCHAR(50),
    salary INT,
    country VARCHAR(50)
);

CREATE TABLE departments (
    id INT PRIMARY KEY,
    department_name VARCHAR(50),
    manager VARCHAR(50)
);
```

### 9.1 Basic view (single table, simple filter)

```sql
CREATE VIEW us_employees AS
SELECT id, name, department, salary
FROM employees
WHERE country = 'US';
```
```sql
SELECT * FROM us_employees;
```
> Practice: write a view called `high_paid` showing employees with `salary > 80000`.

---

### 9.2 View hiding sensitive columns (security use case)

```sql
CREATE VIEW employee_public AS
SELECT id, name, department
FROM employees;
```
> `salary` and `country` are simply never exposed — not filtered at query time, but absent from the view definition itself.

---

### 9.3 View with a JOIN (read-only)

```sql
CREATE VIEW employee_dept_info AS
SELECT e.name, e.salary, d.department_name, d.manager
FROM employees e
JOIN departments d ON e.department = d.department_name;
```
```sql
SELECT * FROM employee_dept_info WHERE manager = 'Alex';
```
> This view is **not updatable** — it spans two tables, so the engine can't tell which base table a row update should hit.

---

### 9.4 View with aggregation (also read-only)

```sql
CREATE VIEW department_avg_salary AS
SELECT department, AVG(salary) AS avg_salary, COUNT(*) AS headcount
FROM employees
GROUP BY department;
```
```sql
SELECT * FROM department_avg_salary ORDER BY avg_salary DESC;
```
> Practice: modify this to also show `MAX(salary)` per department.

---

### 9.5 Updatable view (single table, no aggregation)

```sql
CREATE VIEW it_employees AS
SELECT id, name, salary
FROM employees
WHERE department = 'IT';
```
```sql
-- This works: single table, no GROUP BY/aggregates
UPDATE it_employees SET salary = salary + 5000 WHERE name = 'Sam';
```
```sql
-- This also works through the view
INSERT INTO it_employees (id, name, salary) VALUES (101, 'Riya', 70000);
```
> Note: `INSERT` here only succeeds if `department` (used in the `WHERE`, but not selected) has a default or is nullable — otherwise the row can't satisfy `NOT NULL` constraints. This is a common gotcha.

---

### 9.6 `WITH CHECK OPTION` in practice

```sql
CREATE VIEW high_earners AS
SELECT id, name, salary
FROM employees
WHERE salary > 100000
WITH CHECK OPTION;
```
```sql
-- Allowed: still satisfies salary > 100000
UPDATE high_earners SET salary = 150000 WHERE name = 'Sam';

-- Blocked: row would no longer match the view's WHERE clause
UPDATE high_earners SET salary = 50000 WHERE name = 'Sam';

-- Blocked: boundary case, salary > 100000 is strict
UPDATE high_earners SET salary = 100000 WHERE name = 'Sam';
```
> Practice: create `low_earners` as `salary < 50000 WITH CHECK OPTION`, and write one allowed and one blocked `UPDATE`.

---

### 9.7 Nested view (view built on another view)

```sql
CREATE VIEW us_high_earners AS
SELECT *
FROM us_employees
WHERE salary > 100000;
```
> Built on top of `us_employees` (9.1). Querying `us_high_earners` resolves through both view definitions down to the real `employees` table.

---

### 9.8 Materialized view (for comparison — syntax varies by database)

```sql
-- PostgreSQL syntax
CREATE MATERIALIZED VIEW department_avg_salary_mat AS
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department;

-- Data is now physically stored. Must be refreshed manually:
REFRESH MATERIALIZED VIEW department_avg_salary_mat;
```
> Practice: explain out loud why `department_avg_salary` (9.4) always shows current data, while `department_avg_salary_mat` can be stale until refreshed.

---

### 9.9 Dropping / replacing a view

```sql
DROP VIEW us_high_earners;

-- Modify a view's definition without dropping it first
CREATE OR REPLACE VIEW us_employees AS
SELECT id, name, department, salary, country
FROM employees
WHERE country = 'US' AND salary > 30000;
```

---

## 10. Quick Self-Test

1. Does a plain view speed up queries? → **No**
2. Can you index a regular view? → **No** (only materialized views)
3. If you `DROP TABLE` on the base table, does the view get deleted too? → **No**, it becomes invalid/dangling
4. Is `salary = 100000` allowed through a `WHERE salary > 100000 ... WITH CHECK OPTION` view? → **No** (boundary case, strict `>`)
5. Can a view based on a `JOIN` of two tables generally be updated? → **No**, typically read-only
