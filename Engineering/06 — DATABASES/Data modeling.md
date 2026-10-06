---
tags: [data-modeling, schema-design]
status: not-started
---

# Data Modeling

> **What this is:** deciding what tables you have, what's in them, and how they relate — *before* you write pipeline code.
> **Why you care:** schema decisions are the hardest thing to change later. Every table, every query, every dashboard depends on them. Get this right and everything downstream is easy.

---

## The idea in plain English

You have one enormous table of sales. Every row repeats the customer's country, the product's description, the product's category, the date's day-of-week.

Three problems with that:

1. **Waste.** "United Kingdom" stored 495,000 times.
2. **Inconsistency.** Someone fixes a product name in some rows and not others. Now you have two products.
3. **You can't record things that didn't happen.** A customer who never ordered doesn't exist in a sales table at all.

**Dimensional modelling** splits it into:

- **Facts** — the things that happened. Numbers you add up. (A sale.)
- **Dimensions** — the things that describe them. Words you filter and group by. (A customer, a product, a date.)

The test: **if you'd `SUM()` it, it's a fact. If you'd `GROUP BY` it, it's a dimension.**

---

## 1. The star schema

```
                  ┌──────────────┐
                  │  dim_date    │
                  │──────────────│
                  │ date_key (PK)│
                  │ date         │
                  │ day_of_week  │
                  │ month        │
                  │ is_weekend   │
                  └──────┬───────┘
                         │
┌──────────────┐   ┌─────▼──────────────┐   ┌──────────────────┐
│ dim_customer │   │ fact_transactions  │   │  dim_product     │
│──────────────│   │────────────────────│   │──────────────────│
│ customer_key │◄──┤ customer_key  (FK) │   │ product_key (PK) │
│   (PK)       │   │ product_key   (FK) ├──►│ product_id       │
│ customer_id  │   │ date_key      (FK) │   │ description      │
│ country      │   │ invoice_no    (DD) │   │ category         │
│ segment      │   │────────────────────│   │ unit_cost        │
└──────────────┘   │ quantity           │   └──────────────────┘
                   │ unit_price         │
                   │ revenue            │  ← the measures
                   └────────────────────┘
```

It's called a **star** because the fact table sits in the middle with dimensions radiating out.

**Why this shape:**

- **One join to any descriptive attribute.** Country, category, day-of-week — all one hop from the fact.
- **Analysts can navigate it.** "Revenue by category by month" is obviously two joins. No tribal knowledge required.
- **Query engines are built for it.** Spark and every warehouse recognise star schemas and optimise for them (small dimensions get broadcast — see [[PySpark core]]).

### Star vs. snowflake

A **snowflake** schema normalises dimensions further — `dim_product` points to `dim_category`, which points to `dim_department`.

More "correct" in a textbook sense. Worse in practice: more joins per query, more complexity for analysts, and the storage saved is trivial.

> **Use a star.** Keep dimensions flat and denormalised, even when it repeats "Kitchen" 400 times. Storage is cheap; analyst confusion and query complexity are not. Snowflake only when a dimension is genuinely enormous.

---

## 2. Grain — decide this first

**The grain is what one row of the fact table means.** One sentence. Write it down before anything else.

For the capstone:

> **One row = one product on one invoice.**

Everything follows from that:
- The natural key is `(invoice_no, product_id)`
- `quantity` and `revenue` are additive at this grain
- Invoice-level facts (like a delivery charge) **don't belong here** — they'd be repeated per line and double-count when summed

> **Getting the grain wrong is the most expensive modelling mistake.** Mix grains in one table — some rows per line item, some per invoice — and every `SUM` is wrong, forever, and the wrongness is subtle enough that nobody notices for months. Decide the grain, write it in a comment at the top of the table definition, and enforce it.

**If you need two grains, build two fact tables.** `fact_transactions` (per line) and `fact_invoices` (per invoice). That's correct, not a compromise.

---

## 3. Keys

### Natural vs. surrogate

| | Natural key | Surrogate key |
|---|---|---|
| What | The real-world identifier: `customer_id = 17850` | A meaningless integer you generate: `customer_key = 1` |
| Comes from | The source system | You |
| Problems | Can change; can be reused; can be a long string; can collide across systems | None, but needs a lookup |

**Use surrogate keys for dimension primary keys.** Keep the natural key as an attribute.

```sql
CREATE TABLE dim_customer (
    customer_key   BIGINT       NOT NULL,     -- surrogate: the PK
    customer_id    NVARCHAR(20) NOT NULL,     -- natural: from the source
    country        NVARCHAR(100),
    ...
);
```

**Two reasons this matters, and the second is the real one:**

1. Source IDs change or collide when you integrate a second system.
2. **You cannot do SCD Type 2 without them** — you need multiple rows for the same customer, which means the natural key can't be unique. (Section 4.)

### `dim_date` — build it, always

Yes, a whole table of dates. Every warehouse has one.

```sql
CREATE TABLE dim_date (
    date_key      INT PRIMARY KEY,     -- 20111201 — sortable and readable
    full_date     DATE,
    day_of_week   NVARCHAR(10),        -- 'Thursday'
    day_of_month  INT,
    week_of_year  INT,
    month_number  INT,
    month_name    NVARCHAR(10),
    quarter       INT,
    year          INT,
    is_weekend    BIT,
    is_holiday    BIT,                 -- ← the one you can't compute
    fiscal_year   INT,                 -- ← nor this
    fiscal_period INT
);
```

**Why not just use the date column and compute?** Because:

- `is_holiday` and `fiscal_year` **cannot be computed** — they're business facts that have to be stored.
- Filtering `WHERE d.is_weekend = 1` uses an index. `WHERE DATEPART(dw, date) IN (1,7)` wraps the column in a function and kills it ([[SQL fundamentals]]).
- Every analyst spells date logic differently, and some of them get it wrong. One table, one definition.
- Rows for dates with **no sales** still exist, so a "revenue by day" report shows zeros instead of missing days.

Build it once with a recursive CTE (the one in [[SQL fundamentals]]) and never think about it again.

---

## 4. Slowly Changing Dimensions

### The problem

Customer 17850 moves from France to Germany.

They spent £5,000 while in France. Now what does "revenue by country" say?

- Overwrite the country → all £5,000 retroactively becomes German revenue. **Your history just changed.**
- Keep the old value → new orders are wrongly attributed to France.

Neither is right, which is why SCD types exist.

### The types

| Type | What it does | Use when |
|---|---|---|
| **Type 0** | Never changes | Date of birth, original signup date |
| **Type 1** | Overwrite. History lost. | Correcting a typo — the old value was simply *wrong* |
| **Type 2** | New row per change, with validity dates. Full history. | The attribute genuinely changed, and history matters |
| Type 3 | Add a `previous_value` column | You only ever need one step back. Rare. |

**Type 1 vs Type 2, the deciding question:** *was the old value wrong, or was it right at the time?*

- Misspelled name → **Type 1**. It was always wrong. Overwrite it.
- Customer moved country → **Type 2**. It was right then, it's different now. Keep both.

### Type 2 in practice

```sql
CREATE TABLE dim_customer (
    customer_key   BIGINT NOT NULL,       -- surrogate PK — DIFFERENT per version
    customer_id    NVARCHAR(20) NOT NULL, -- natural key — SAME across versions
    country        NVARCHAR(100),
    segment        NVARCHAR(50),
    valid_from     DATE NOT NULL,
    valid_to       DATE NOT NULL,         -- '9999-12-31' for current
    is_current     BIT  NOT NULL
);
```

| customer_key | customer_id | country | valid_from | valid_to | is_current |
|---|---|---|---|---|---|
| 1 | 17850 | France | 2010-11-15 | 2011-06-30 | 0 |
| 2 | 17850 | Germany | 2011-07-01 | 9999-12-31 | 1 |

The fact table stores `customer_key`, so **a sale is permanently linked to the version of the customer that was true when it happened.** Historical reports stay correct forever, and new sales attribute to Germany. That's the whole point.

**Querying it:**

```sql
-- current state only
SELECT * FROM dim_customer WHERE is_current = 1;

-- as it was on a given date
SELECT * FROM dim_customer WHERE '2011-03-15' BETWEEN valid_from AND valid_to;
```

> `'9999-12-31'` instead of `NULL` for the open end. `BETWEEN` works, no `IS NULL` special-casing, and every query gets simpler. Universal convention.

### Implementing it with `MERGE`

```sql
-- 1. close the current row where something changed
MERGE INTO dim_customer AS t
USING staging_customers AS s
  ON t.customer_id = s.customer_id AND t.is_current = 1
WHEN MATCHED AND (t.country <> s.country OR t.segment <> s.segment) THEN
  UPDATE SET t.valid_to = DATEADD(day, -1, s.effective_date), t.is_current = 0;

-- 2. insert the new version (and any brand-new customers)
INSERT INTO dim_customer (customer_key, customer_id, country, segment, valid_from, valid_to, is_current)
SELECT next_key(), s.customer_id, s.country, s.segment, s.effective_date, '9999-12-31', 1
FROM staging_customers s
LEFT JOIN dim_customer t
  ON t.customer_id = s.customer_id AND t.is_current = 1
WHERE t.customer_id IS NULL
   OR t.country <> s.country
   OR t.segment <> s.segment;
```

Two statements: close the old, open the new. Delta Lake's `MERGE` does this well ([[Databricks and Delta Lake]]).

> **Don't make everything Type 2.** Each Type 2 dimension multiplies rows and complicates every join. **Decide per column.** In the capstone, `country` and `segment` are worth tracking; `description` probably isn't.

---

## 5. Fact table design

### Types of fact

| Type | Meaning | Example |
|---|---|---|
| **Transaction** | One row per event. The most common. | One sale line |
| **Periodic snapshot** | One row per entity per period | Customer RFM as of each month-end |
| **Accumulating snapshot** | One row per process, updated as it progresses | An order: placed → picked → shipped → delivered |

The capstone uses two: `fact_transactions` (transaction grain) and `customer_rfm` (periodic snapshot — hence the `as_of_date` column).

### Additivity — the trap

| Kind | Can you sum it? | Example |
|---|---|---|
| **Additive** | Across every dimension | `revenue`, `quantity` |
| **Semi-additive** | Across some, not time | An account balance — summing across months is meaningless |
| **Non-additive** | Never | Ratios, percentages, averages |

> **The classic bug:** storing `profit_margin_pct` in a fact table. Someone sums it across 400 products and gets 12,000%. **Store the numerator and denominator** (`revenue`, `cost`) and let the ratio be computed at query time. Then it's correct at every level of aggregation, automatically.

### Degenerate dimensions

`invoice_no` sits in the fact table with no `dim_invoice`. It's an identifier with no attributes worth storing separately — a **degenerate dimension**. Perfectly normal; you need it to group line items back into orders.

### Null foreign keys

The retail dataset has sales with no `customer_id`. Options:

- ❌ `NULL` in the fact table — every join needs `LEFT JOIN` and analysts will get counts wrong
- ✅ A special dimension row: `customer_key = -1`, `customer_id = 'UNKNOWN'`

> **Use the unknown-member row.** `INNER JOIN` always works, counts are right, and the unknowns are explicitly visible instead of silently disappearing. Same for `-2` = 'Not Applicable' if you need it.

---

## 6. Naming conventions

Boring, and it's the thing that makes a 40-table warehouse usable.

```
dim_customer          fact_transactions        agg_daily_sales
dim_product           fact_invoices
dim_date
```

| Rule | Example |
|---|---|
| Prefix by type | `dim_`, `fact_`, `agg_`, `stg_` |
| `snake_case`, lowercase, always | `product_category`, never `ProductCategory` |
| Singular dimension names | `dim_customer`, not `dim_customers` |
| Suffix keys | `customer_key` (surrogate), `customer_id` (natural) |
| Suffix by type | `_date`, `_at` (timestamp), `_flag`/`is_` (boolean), `_amt`, `_qty` |
| Same concept, same name everywhere | `customer_id` in all 12 tables — never `cust_id` in one |
| No reserved words | not `date`, `order`, `user`, `group` |
| Spell it out | `quantity`, not `qty`; `description`, not `desc` (also a keyword) |

> **The one that matters most: the same concept has the same name in every table.** Then joins are obvious, `USING (customer_id)` works, and nobody has to check whether this table calls it `cust_id`. This single rule saves more time than all the others combined.

---

## 7. The capstone schema

Design this before writing pipeline code ([[Capstone build guide]]).

**Grain statements — write these first:**
- `fact_transactions`: one row per product per invoice
- `customer_rfm`: one row per customer per snapshot date

```sql
-- ============ DIMENSIONS ============

CREATE TABLE dim_date (
    date_key     INT PRIMARY KEY,            -- 20111201
    full_date    DATE NOT NULL,
    day_of_week  NVARCHAR(10),
    week_of_year INT,
    month_number INT,
    month_name   NVARCHAR(10),
    quarter      INT,
    year         INT,
    is_weekend   BIT
);

CREATE TABLE dim_product (
    product_key  BIGINT PRIMARY KEY,         -- surrogate
    product_id   NVARCHAR(20) NOT NULL,      -- natural (StockCode)
    description  NVARCHAR(200),
    category     NVARCHAR(100),              -- derived from description
    is_current   BIT NOT NULL
);

CREATE TABLE dim_customer (                  -- SCD Type 2
    customer_key BIGINT PRIMARY KEY,
    customer_id  NVARCHAR(20) NOT NULL,
    country      NVARCHAR(100),
    segment      NVARCHAR(50),
    valid_from   DATE NOT NULL,
    valid_to     DATE NOT NULL,              -- '9999-12-31' = current
    is_current   BIT  NOT NULL
);
-- plus the unknown member:
-- (-1, 'UNKNOWN', 'Unknown', 'Unknown', '1900-01-01', '9999-12-31', 1)

-- ============ FACT ============
-- GRAIN: one row per product per invoice

CREATE TABLE fact_transactions (
    transaction_key BIGINT IDENTITY(1,1) PRIMARY KEY,
    date_key        INT    NOT NULL REFERENCES dim_date(date_key),
    customer_key    BIGINT NOT NULL REFERENCES dim_customer(customer_key),
    product_key     BIGINT NOT NULL REFERENCES dim_product(product_key),
    invoice_no      NVARCHAR(20) NOT NULL,    -- degenerate dimension
    quantity        INT           NOT NULL,
    unit_price      DECIMAL(10,2) NOT NULL,   -- DECIMAL, never FLOAT
    revenue         DECIMAL(12,2) NOT NULL    -- pre-computed for query speed
);

CREATE INDEX ix_fact_date     ON fact_transactions (date_key);
CREATE INDEX ix_fact_customer ON fact_transactions (customer_key);
CREATE INDEX ix_fact_product  ON fact_transactions (product_key);

-- ============ GOLD SERVING TABLES ============
-- Denormalised, small, and what the API actually queries

CREATE TABLE daily_sales (                   -- GRAIN: one row per date per category
    date             DATE          NOT NULL,
    product_category NVARCHAR(100) NOT NULL,
    revenue          DECIMAL(12,2) NOT NULL,
    units_sold       INT           NOT NULL,
    PRIMARY KEY (date, product_category)
);

CREATE TABLE customer_rfm (                  -- GRAIN: one row per customer per snapshot
    customer_id    NVARCHAR(20)  NOT NULL,
    as_of_date     DATE          NOT NULL,
    recency_days   INT           NOT NULL,
    frequency      INT           NOT NULL,
    monetary_value DECIMAL(12,2) NOT NULL,
    segment        NVARCHAR(50),
    PRIMARY KEY (customer_id, as_of_date)
);
```

### Why the gold tables aren't a star

`daily_sales` and `customer_rfm` are flat, denormalised, and don't join to anything.

That's deliberate. They exist to be read by an API at millisecond latency ([[Azure SQL Database]]). No joins means no join cost. The star schema is the *modelling* layer; these are the *serving* layer. Different jobs, different shapes.

### RFM, since it's the segmentation input

| Letter | Means | Computed as |
|---|---|---|
| **R**ecency | Days since their last purchase | `DATEDIFF(day, MAX(invoice_date), as_of_date)` |
| **F**requency | How many distinct orders | `COUNT(DISTINCT invoice_no)` |
| **M**onetary | Total spent | `SUM(revenue)` |

Low recency + high frequency + high monetary = your best customer. It's a 1990s marketing technique and it's still the standard baseline for retail segmentation — largely because it works and it's explainable.

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| Totals are double what they should be | Mixed grain, or a fan-out join | Write the grain down; check row counts around joins |
| A percentage sums to 4,000% | Non-additive measure stored in a fact | Store numerator and denominator separately |
| Historical report changed after an update | Type 1 overwrite where Type 2 was needed | Convert to Type 2; you can't recover lost history |
| Rows disappear when joining a dimension | `NULL` foreign keys + `INNER JOIN` | Use the `-1` unknown member |
| Days with no sales missing from a report | Joining from the fact table | Join from `dim_date` with a `LEFT JOIN` |
| Money totals off by pennies | `FLOAT` instead of `DECIMAL` | `DECIMAL(12,2)` |
| Same field named differently per table | No naming convention | One name per concept, enforced in review |
| Duplicate customers appear | Natural key isn't unique (Type 2 working as designed) | Filter `is_current = 1`, or join on the surrogate key |
| Date filters are slow | Function wrapping the column | Use `dim_date` flags, or a range predicate |
| Query needs six joins for one attribute | Snowflaked too far | Flatten the dimension |
| Can't reconcile a number with the source | No audit trail | This is why bronze is immutable ([[ADLS Gen2]]) |

---

## Practice checklist

- [ ] Facts vs. dimensions — the `SUM` vs `GROUP BY` test
- [ ] Star schema, and why not to snowflake
- [ ] **Grain** — writing it in one sentence before anything else
- [ ] Natural vs. surrogate keys, and why SCD Type 2 requires surrogates
- [ ] `dim_date` — why it exists when you already have a date column
- [ ] Slowly changing dimensions: Type 1 vs. Type 2, and the "was it wrong, or was it right at the time?" test
- [ ] Implementing Type 2 with `valid_from` / `valid_to` / `is_current` and `MERGE`
- [ ] Fact types: transaction, periodic snapshot, accumulating snapshot
- [ ] Additive / semi-additive / non-additive measures — and never storing a ratio
- [ ] Degenerate dimensions
- [ ] The unknown-member row instead of null foreign keys
- [ ] Normalization vs. denormalization, and why the serving layer is flat
- [ ] Naming conventions that scale past 20 tables

## Hands-on

- [ ] Write the grain statement for every table in the capstone, in one sentence each
- [ ] Design the full star schema before writing a line of pipeline code
- [ ] Build `dim_date` with a recursive CTE, covering 2009–2012
- [ ] Decide, explicitly, which dimensions need SCD Type 2 and why — write the reason down
- [ ] Implement the Type 2 `MERGE` for `dim_customer` and test it with a country change
- [ ] Add the `-1` unknown member and confirm the null-customer sales still join
- [ ] Deliberately store a percentage in a fact table, sum it, and see the nonsense

## Resources

- [Kimball Group: Dimensional Modeling Techniques](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/) — the canonical reference, free
- [Kimball: Slowly Changing Dimensions](https://www.kimballgroup.com/2013/02/design-tip-152-slowly-changing-dimension-types-0-4-5-6-7/)

## Next

[[Adjacent tools you will meet]]
