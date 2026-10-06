---
tags: [foundations, sql]
status: not-started
---

# SQL Fundamentals

> **What this is:** the language for asking questions of data that lives in tables.
> **Why you care:** Azure SQL, Spark SQL, Databricks, and every dashboard you build all speak it. If SQL is something you look up instead of something you *know*, everything downstream is slow.

---

## The idea in plain English

Imagine a giant spreadsheet. Actually, imagine several giant spreadsheets that know about each other.

A **table** is one spreadsheet. It has **columns** (the headings across the top: `customer_id`, `country`, `price`) and **rows** (one line per thing: one sale, one customer, one product).

SQL is how you say things like *"show me only the rows where the country is France"* or *"add up the price column, but grouped by month"* — without opening the spreadsheet and scrolling.

The single most important mental shift: **you describe what you want, not how to get it.** In Python you'd write a loop. In SQL you write a description, and the database figures out the loop itself. That's why a database can answer a question over 500 million rows in two seconds and your `for` loop can't.

---

## The running example

Everything below uses these three tables. They're the shape of the capstone dataset, so you'll see them again.

**`sales`** — one row per item sold

| invoice_no | product_id | customer_id | quantity | unit_price | invoice_date |
|---|---|---|---|---|---|
| 536365 | P100 | 17850 | 6 | 2.55 | 2010-12-01 |
| 536365 | P200 | 17850 | 2 | 3.39 | 2010-12-01 |
| 536366 | P100 | 17851 | 1 | 2.55 | 2010-12-02 |
| C536367 | P100 | 17850 | -6 | 2.55 | 2010-12-03 |

**`customers`** — one row per customer

| customer_id | country | signup_date |
|---|---|---|
| 17850 | United Kingdom | 2010-11-15 |
| 17851 | France | 2010-11-20 |

**`products`** — one row per product

| product_id | description | category |
|---|---|---|
| P100 | White Mug | Kitchen |
| P200 | Red Lamp | Lighting |

---

## 1. The anatomy of a query

Every query is built from the same blocks, always written in this order:

```sql
SELECT   customer_id, SUM(quantity * unit_price) AS total_spent
FROM     sales
WHERE    invoice_date >= '2010-12-01'
GROUP BY customer_id
HAVING   SUM(quantity * unit_price) > 100
ORDER BY total_spent DESC
LIMIT    10;
```

Read it out loud, block by block:

| Block | Plain English |
|---|---|
| `SELECT` | which columns I want back |
| `FROM` | which table they come from |
| `WHERE` | throw away rows I don't want (**before** any grouping) |
| `GROUP BY` | squash rows into buckets |
| `HAVING` | throw away whole *buckets* I don't want (**after** grouping) |
| `ORDER BY` | sort the result |
| `LIMIT` | only give me the first N |

### The gotcha: SQL doesn't run in the order you wrote it

This trips up literally everyone. You *write* `SELECT` first, but the database *runs* it almost last:

```
FROM  →  WHERE  →  GROUP BY  →  HAVING  →  SELECT  →  ORDER BY  →  LIMIT
```

Two consequences you will hit today:

**You can't use a `SELECT` alias in `WHERE`.**

```sql
-- ✗ BREAKS: `total` doesn't exist yet when WHERE runs
SELECT quantity * unit_price AS total
FROM sales
WHERE total > 50;

-- ✓ Works: repeat the expression
SELECT quantity * unit_price AS total
FROM sales
WHERE quantity * unit_price > 50;
```

**But you *can* use it in `ORDER BY`**, because `ORDER BY` runs after `SELECT`:

```sql
SELECT quantity * unit_price AS total
FROM sales
ORDER BY total DESC;   -- ✓ fine
```

> **Rule of thumb:** if the error says *"invalid column name"* or *"column does not exist"* on something you clearly just defined — you've hit execution order. Repeat the expression, or wrap the whole thing in a CTE (section 3).

### `WHERE` vs `HAVING`, settled forever

- `WHERE` filters **rows**, before grouping. "Only December sales."
- `HAVING` filters **groups**, after grouping. "Only customers who spent over £100 *in total*."

You can't write `WHERE SUM(...) > 100` because at `WHERE` time nothing has been summed yet.

---

## 2. JOINs — combining tables

Your sale rows only have a `customer_id`. To find out which *country* that customer is in, you have to reach into the `customers` table. That's a join.

```sql
SELECT s.invoice_no, s.quantity, c.country
FROM sales s
JOIN customers c ON s.customer_id = c.customer_id;
```

`ON s.customer_id = c.customer_id` is the *matching rule*: glue a sales row to a customers row wherever those two values are equal.

### The four types, as a picture

Think of two overlapping circles: table A on the left, table B on the right.

| Type | Keeps | Use when |
|---|---|---|
| `INNER JOIN` | only rows that match in **both** | the normal case — you need data from both sides |
| `LEFT JOIN` | **all** of A, plus B where it matches (nulls where it doesn't) | "every customer, and their orders *if any*" |
| `RIGHT JOIN` | all of B, plus A where it matches | rare — just flip the tables and use `LEFT` |
| `FULL OUTER JOIN` | everything from both sides, nulls wherever there's no match | reconciling two lists that should agree but don't |

```sql
-- Every customer, even the ones who never bought anything.
-- Those get NULL in the sales columns.
SELECT c.customer_id, c.country, s.invoice_no
FROM customers c
LEFT JOIN sales s ON c.customer_id = s.customer_id;
```

### A `SELF JOIN` is a table joined to itself

Useful when rows relate to *other rows in the same table* — an employee and their manager, or a product and its replacement.

```sql
-- pairs of different products bought on the same invoice
SELECT a.product_id, b.product_id
FROM sales a
JOIN sales b
  ON a.invoice_no = b.invoice_no
 AND a.product_id < b.product_id;   -- `<` stops duplicates and self-pairs
```

### ⚠️ The row-count trap — the #1 SQL bug in real jobs

A join does **not** always keep your row count the same. If one row on the left matches **three** rows on the right, you get **three** rows out.

`sales` has 4 rows. If a customer somehow appeared twice in `customers` (a duplicate row — this happens constantly in real data), joining would silently double some of your sales. Then you `SUM` the revenue and your number is wrong, and nothing errors.

**Always sanity check:**

```sql
SELECT COUNT(*) FROM sales;                     -- before: 541909
SELECT COUNT(*) FROM sales s JOIN customers c   -- after:  541909?
       ON s.customer_id = c.customer_id;
```

If the number went **up**, your right-hand table has duplicates. If it went **down**, some rows didn't match and an `INNER JOIN` quietly threw them away — you may have wanted a `LEFT JOIN`.

> **This is the single highest-value habit in this note.** Count before, count after, explain the difference.

---

## 3. CTEs — naming your steps

A **CTE** (Common Table Expression) is a temporary named result you build with `WITH`. It's the SQL equivalent of assigning to a variable instead of writing one enormous nested expression.

Compare. Nested subquery — hard to read, hard to debug:

```sql
SELECT country, AVG(total)
FROM (
  SELECT c.country, s.invoice_no, SUM(s.quantity * s.unit_price) AS total
  FROM sales s JOIN customers c ON s.customer_id = c.customer_id
  GROUP BY c.country, s.invoice_no
) AS invoice_totals
GROUP BY country;
```

Same thing with a CTE — reads top to bottom, like a recipe:

```sql
WITH invoice_totals AS (
    SELECT c.country,
           s.invoice_no,
           SUM(s.quantity * s.unit_price) AS total
    FROM sales s
    JOIN customers c ON s.customer_id = c.customer_id
    GROUP BY c.country, s.invoice_no
)
SELECT country, AVG(total) AS avg_basket
FROM invoice_totals
GROUP BY country;
```

You can chain as many as you like, and each can use the ones above it:

```sql
WITH cleaned AS (
    SELECT * FROM sales WHERE quantity > 0        -- step 1: drop refunds
),
with_revenue AS (
    SELECT *, quantity * unit_price AS revenue    -- step 2: add a column
    FROM cleaned
)
SELECT product_id, SUM(revenue) AS total_revenue  -- step 3: aggregate
FROM with_revenue
GROUP BY product_id;
```

> **Debugging trick:** when a big CTE query gives a wrong answer, comment out everything after one CTE and `SELECT * FROM that_cte LIMIT 20`. You'll find the broken step in about thirty seconds. This is why you write CTEs instead of nested subqueries.

### Recursive CTEs (rarer — know they exist)

For data that points at itself: org charts, folder trees, category hierarchies. A recursive CTE has a starting row and a rule for finding the next level, and repeats until nothing new comes back.

```sql
WITH RECURSIVE date_series AS (
    SELECT CAST('2010-12-01' AS DATE) AS d        -- the anchor: where we start
    UNION ALL
    SELECT DATEADD(day, 1, d)                     -- the rule: one day later
    FROM date_series
    WHERE d < '2011-12-09'                        -- the stop condition — NEVER forget this
)
SELECT * FROM date_series;
```

That generates every date in a range — genuinely useful for building a `dim_date` table (you'll need one in [[Data modeling]]).

> Miss the stop condition and you get an infinite loop. Most databases cap it (SQL Server at 100 iterations by default) and throw an error rather than hanging forever.

---

## 4. Window functions — the superpower

This is the concept that separates people who *use* SQL from people who *know* SQL. Learn it properly; it comes up in every data interview.

### The problem it solves

`GROUP BY` **collapses** rows. Ten sales by one customer become one row. But sometimes you want to keep all ten rows *and* attach a summary to each one — "here's this sale, and here's what % of that customer's total spending it represents."

A window function does exactly that: it computes across a group of related rows, **without collapsing them**.

```sql
SELECT
    invoice_no,
    customer_id,
    quantity * unit_price AS revenue,
    SUM(quantity * unit_price) OVER (PARTITION BY customer_id) AS customer_total
FROM sales;
```

Every row survives. Each one now carries its customer's grand total alongside it.

### The anatomy of `OVER (...)`

```sql
FUNCTION() OVER (
    PARTITION BY some_column      -- split rows into groups (like GROUP BY, but non-destructive)
    ORDER BY     another_column   -- order within each group (needed for ranking / running totals)
)
```

- **No `PARTITION BY`** → the window is the whole table.
- **`PARTITION BY customer_id`** → a separate window per customer; calculations restart for each one.

### The functions worth memorising

**Ranking** — "which is the biggest, second biggest…"

```sql
SELECT
    customer_id,
    invoice_no,
    revenue,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY revenue DESC) AS rn,
    RANK()       OVER (PARTITION BY customer_id ORDER BY revenue DESC) AS rnk,
    DENSE_RANK() OVER (PARTITION BY customer_id ORDER BY revenue DESC) AS dense_rnk
FROM invoice_revenue;
```

The difference, with three tied scores of 100, 100, 90:

| revenue | `ROW_NUMBER` | `RANK` | `DENSE_RANK` |
|---|---|---|---|
| 100 | 1 | 1 | 1 |
| 100 | 2 | 1 | 1 |
| 90 | 3 | **3** | **2** |

- `ROW_NUMBER` — always 1,2,3. Ties broken arbitrarily. Use it to **pick exactly one row per group**.
- `RANK` — ties share a number, then it *skips*. Like Olympic medals: two golds, no silver.
- `DENSE_RANK` — ties share a number, no skipping.

**Offset** — "compare this row to the one before it"

```sql
SELECT
    sale_date,
    daily_revenue,
    LAG(daily_revenue, 1)  OVER (ORDER BY sale_date) AS yesterday,
    LEAD(daily_revenue, 1) OVER (ORDER BY sale_date) AS tomorrow,
    daily_revenue - LAG(daily_revenue, 1) OVER (ORDER BY sale_date) AS day_over_day_change
FROM daily_sales;
```

`LAG` looks backwards, `LEAD` looks forwards. This is how every "vs. last month" number on every dashboard is built. You'll use it directly to make lag features for the forecasting model in [[Capstone build guide]].

**Running totals** — add `ORDER BY` to an aggregate and it becomes cumulative:

```sql
SELECT
    sale_date,
    daily_revenue,
    SUM(daily_revenue) OVER (ORDER BY sale_date) AS revenue_to_date,
    AVG(daily_revenue) OVER (ORDER BY sale_date
                             ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS rolling_7day_avg
FROM daily_sales;
```

That `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` is a **frame** — it narrows the window to "this row and the six before it". That's a 7-day moving average, in one line.

### The pattern you'll reuse most: "top N per group"

Get each customer's single biggest order. You **can't** do this with `GROUP BY` alone (you'd get the max revenue but lose the invoice number). Window function + CTE:

```sql
WITH ranked AS (
    SELECT
        customer_id,
        invoice_no,
        revenue,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY revenue DESC) AS rn
    FROM invoice_revenue
)
SELECT customer_id, invoice_no, revenue
FROM ranked
WHERE rn = 1;
```

> **Why the CTE is mandatory:** you can't put a window function in a `WHERE` clause — remember the execution order, `WHERE` runs long before `SELECT`. Compute the ranking in one step, filter it in the next. Memorise this shape; you will type it hundreds of times.

Same trick deduplicates a table — rank by whatever makes a row "the good one" and keep `rn = 1`.

---

## 5. Indexes — why some queries are instant and some take a minute

### The analogy

A textbook with no index: to find every mention of "photosynthesis" you read all 900 pages. That's a **full table scan**.

A textbook *with* an index: flip to the back, look up the word, get page numbers, jump straight there. That's an **index seek**.

A database index is exactly that — a pre-sorted lookup structure (a B-tree) pointing at where rows live.

```sql
CREATE INDEX idx_sales_customer ON sales (customer_id);
```

Now `WHERE customer_id = 17850` is near-instant instead of reading all 541,909 rows.

### The cost

Indexes aren't free, and this is the part people forget:

- **Writes get slower.** Every `INSERT`/`UPDATE`/`DELETE` has to update every index on that table too. Ten indexes = ten extra bits of bookkeeping per write.
- **They take disk space.**
- **They go stale** if statistics aren't maintained, and the database starts making bad plans.

> **Rule of thumb:** index the columns you filter on (`WHERE`), join on (`ON`), and sort by (`ORDER BY`). Don't index everything "just in case" — that's the classic over-engineering move, and it makes your writes crawl.

### What stops an index from being used

If you wrap the indexed column in a function, the index becomes useless — the database can't look up something it hasn't got a sorted list of:

```sql
-- ✗ index on invoice_date can't be used: the function hides the raw value
WHERE YEAR(invoice_date) = 2011

-- ✓ index works fine: raw column compared to constants
WHERE invoice_date >= '2011-01-01' AND invoice_date < '2012-01-01'
```

This one change has taken real queries from 40 seconds to 0.2 seconds. It's called making the predicate **sargable** — worth knowing the word, because that's what people will call it.

---

## 6. Query plans — reading the database's homework

`EXPLAIN` (or `EXPLAIN ANALYZE` in Postgres, "Display Estimated Execution Plan" in SQL Server, `.explain()` in Spark) shows the *strategy* the database picked before it runs.

```sql
EXPLAIN
SELECT c.country, SUM(s.quantity * s.unit_price)
FROM sales s JOIN customers c ON s.customer_id = c.customer_id
GROUP BY c.country;
```

You don't need to understand every line. Look for four things:

| What you see | What it means | Worry? |
|---|---|---|
| **Seq Scan / Table Scan** | reading every row | Only if the table is big and you filtered it. Means a missing or unusable index. |
| **Index Seek / Index Scan** | using an index | Good. |
| **Hash Join / Merge Join** | standard join strategies for big tables | Fine. |
| **Nested Loop** over a big table | for each row on the left, scan the right | **Red flag** on big tables — usually a missing index on the join column. |
| **rows=** estimates wildly off from reality | statistics are stale | Run `ANALYZE` / `UPDATE STATISTICS`. |

> The plan is read **inside-out and bottom-up** — the most indented lines run first.

---

## Cheat sheet

```sql
-- filter rows
SELECT * FROM sales WHERE quantity > 0 AND country IN ('France','Germany');

-- text pattern / null / range
WHERE description LIKE '%MUG%'
WHERE customer_id IS NULL              -- never `= NULL`, that's always false
WHERE invoice_date BETWEEN '2011-01-01' AND '2011-01-31'

-- aggregate
SELECT product_id, COUNT(*), SUM(quantity), AVG(unit_price), MIN(x), MAX(x)
FROM sales GROUP BY product_id;

-- count distinct things
SELECT COUNT(DISTINCT customer_id) FROM sales;

-- handle nulls
SELECT COALESCE(customer_id, -1) AS customer_id FROM sales;   -- first non-null wins

-- if/else
SELECT CASE WHEN revenue > 100 THEN 'big'
            WHEN revenue > 10  THEN 'medium'
            ELSE 'small' END AS bucket
FROM invoice_revenue;

-- stack two results (UNION removes duplicates, UNION ALL is faster and keeps them)
SELECT product_id FROM sales_2010
UNION ALL
SELECT product_id FROM sales_2011;

-- "does a matching row exist?" — usually faster than a join if you need no columns from B
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM sales s WHERE s.customer_id = c.customer_id);
```

**Dialect differences that will bite you** (Azure SQL is T-SQL; Spark/Postgres are closer to standard):

| Task | T-SQL (Azure SQL) | Postgres / Spark SQL |
|---|---|---|
| First 10 rows | `SELECT TOP 10 ...` | `... LIMIT 10` |
| Join strings | `'a' + 'b'` | `'a' \|\| 'b'` or `CONCAT()` |
| Current time | `GETDATE()` | `NOW()` / `current_timestamp()` |
| Null fallback | `ISNULL(x, 0)` | `COALESCE(x, 0)` (works in both — prefer it) |

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| "Invalid column name" on an alias you just made | Execution order — `WHERE` runs before `SELECT` | Repeat the expression, or wrap in a CTE |
| Row count exploded after a join | Duplicates on the right-hand table | `SELECT key, COUNT(*) FROM b GROUP BY key HAVING COUNT(*) > 1` |
| Row count shrank after a join | `INNER JOIN` dropped non-matching rows | Switch to `LEFT JOIN` if you wanted to keep them |
| Totals are too high | Same thing — a fan-out join double-counted | Aggregate *before* joining, in a CTE |
| `WHERE col = NULL` returns nothing | `NULL` isn't equal to anything, not even itself | Use `IS NULL` / `IS NOT NULL` |
| Query is suddenly slow | Missing index, or a function wrapping the filtered column | `EXPLAIN` it; look for a scan where you expected a seek |
| "Column must appear in the GROUP BY clause" | You selected a column that's neither grouped nor aggregated | Add it to `GROUP BY`, or wrap it in `MIN()`/`MAX()`, or use a window function |
| Integer division gives 0 | `5 / 2 = 2` with integer columns | Cast one side: `CAST(a AS FLOAT) / b` or `a * 1.0 / b` |

---

## Practice checklist

- [ ] JOIN types: inner, left, right, full, self, and when each changes row counts
- [ ] CTEs (`WITH`) for readable multi-step queries, including recursive CTEs
- [ ] Window functions: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG`/`LEAD`, `PARTITION BY`
- [ ] Aggregations with `GROUP BY` / `HAVING`, and the order SQL actually evaluates clauses in
- [ ] Indexes: what they speed up, what they cost on writes
- [ ] Query plans: read an `EXPLAIN` output and spot a full table scan

## Hands-on

- [ ] Take a public dataset (e.g. the UCI Online Retail II dataset) and write 10 queries covering every concept above
- [ ] Write the "top N per group" query from memory, no looking
- [ ] Rewrite one query three ways (subquery, CTE, window function) and compare the query plans
- [ ] Deliberately cause a fan-out join, watch the row count explode, then fix it by aggregating first

## Resources

- [Mode SQL Tutorial](https://mode.com/sql-tutorial/) — free, interactive
- [Use The Index, Luke](https://use-the-index-luke.com/) — how indexes actually work
- [PostgreSQL: Window Functions tutorial](https://www.postgresql.org/docs/current/tutorial-window.html) — the clearest official explanation of windows anywhere

## Next

[[Dev environment - Git, Docker, CLI]]
