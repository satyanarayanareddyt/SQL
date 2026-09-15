<style>
a {
    text-decoration: none;
    color: #464feb;
}
tr th, tr td {
    border: 1px solid #e6e6e6;
}
tr th {
    background-color: #f5f5f5;
}
</style>

# Data Modeling — Interview Prep

A concise reference for the core data modeling concepts every data engineer should be able to
explain confidently. Each section covers **what it is**, **why it matters**, and a **quick example**.

## Summary

| # | Concept | One-Line Definition | When / Why It Matters |
|---|---------|---------------------|------------------------|
| 1 | Fact vs Dimension Table | Facts store measurable events; dimensions store descriptive context. | Foundation of every dimensional model. |
| 2 | Star vs Snowflake Schema | Star = denormalized dimensions; Snowflake = normalized dimensions. | Trade-off between query speed and storage/maintenance. |
| 3 | Surrogate vs Natural Key | Surrogate = system-generated key; Natural = business key. | Stability, performance, and SCD support. |
| 4 | Factless Fact Table | A fact table with no measures, only keys. | Modeling events or coverage/many-to-many relationships. |
| 5 | Degenerate Dimension | A dimension attribute stored in the fact table itself. | Transaction identifiers like invoice/order numbers. |
| 6 | Conformed Dimension | A dimension shared consistently across multiple facts/marts. | Enterprise consistency and cross-process analysis. |
| 7 | Slowly Changing Dimensions | Strategies for handling dimension changes over time. | Preserving (or overwriting) historical attribute values. |
| 8 | Grain Definition | The exact level of detail of one fact row. | The single most important modeling decision. |
| 9 | Partitioning Strategy | Splitting large tables into manageable segments. | Query performance, pruning, and maintenance. |
| 10 | CDC vs Incremental Load | Change Data Capture vs delta-based loading. | Efficient, low-latency data ingestion. |
| 11 | Late Arriving Dimensions | Fact arrives before its dimension record exists. | Handling out-of-order data without losing facts. |

---

## 1. Data Modeling — Fact Table vs Dimension Table

**Fact Table** — Stores quantitative, measurable business events (metrics) and foreign keys to
dimensions. Rows are typically numerous and grow over time.
- Contains: **measures** (e.g., `sales_amount`, `quantity`) + **foreign keys**.
- Example: A `Sales_Fact` row = "On this date, this customer bought this product for $50."

**Dimension Table** — Stores descriptive, textual context that gives meaning to facts. Rows are
relatively few and change slowly.
- Contains: **attributes** (e.g., `product_name`, `category`, `customer_city`).
- Example: `Dim_Product` describes each product's name, brand, and category.

| Aspect | Fact Table | Dimension Table |
|--------|-----------|-----------------|
| Content | Measures + FKs | Descriptive attributes |
| Size | Large (many rows) | Smaller (fewer rows) |
| Growth | Grows rapidly | Grows slowly |
| Example | `Sales_Fact` | `Dim_Customer`, `Dim_Product` |

---

## 2. Star Schema vs Snowflake Schema

**Star Schema** — A central fact table connected directly to **denormalized** dimension tables.
Shaped like a star.
- ✅ Simpler queries, fewer joins, faster reads.
- ❌ Some data redundancy.

**Snowflake Schema** — Dimensions are **normalized** into multiple related sub-tables.
- ✅ Less redundancy, saves storage, cleaner data integrity.
- ❌ More joins → more complex, potentially slower queries.

```
Star Schema:                     Snowflake Schema:
                                 
   Dim_Date                         Dim_Date
      |                                |
Dim_Prod — FACT — Dim_Cust      Dim_Prod — FACT — Dim_Cust
      |                             |                 |
   Dim_Store                    Dim_Category      Dim_Region
```

| Aspect | Star | Snowflake |
|--------|------|-----------|
| Dimensions | Denormalized | Normalized |
| Joins | Fewer | More |
| Query speed | Faster | Slower |
| Storage | More | Less |

---

## 3. Surrogate Key vs Natural Key

**Natural (Business) Key** — A key that comes from the source/business data
(e.g., `email`, `SSN`, `product_code`).
- ❌ Can change, may be reused, sometimes large/composite.

**Surrogate Key** — A system-generated, meaningless integer/identity key (e.g., `customer_key = 101`).
- ✅ Stable, compact, immutable, and **essential for Type 2 SCDs** (same business key, multiple versions).

| Aspect | Natural Key | Surrogate Key |
|--------|-------------|---------------|
| Source | Business data | System-generated |
| Stability | Can change | Never changes |
| SCD support | Poor | Required for Type 2 |
| Example | `C001` (Cust_ID) | `101` (Cust_Key) |

**Best practice:** Use surrogate keys as primary keys in dimensions and store the natural key as
an attribute for lineage.

---

## 4. Factless Fact Table

A fact table that contains **only foreign keys** and **no numeric measures**. It records that an
event *happened* or that a relationship *exists*.

**Two common uses:**
1. **Event tracking** — e.g., student attendance (student, class, date — no measure). Counting rows
   gives the metric ("how many attended").
2. **Coverage / many-to-many** — e.g., which products were *eligible* for a promotion (even if not sold).

```sql
-- Attendance: no measures, just the event
CREATE TABLE Attendance_Fact (
  student_key INT,
  class_key   INT,
  date_key    INT
);
-- "How many students attended each class?" = COUNT(*)
```

---

## 5. Degenerate Dimension

A dimension **attribute stored directly in the fact table** — with no separate dimension table —
because it has no other descriptive attributes of its own.

- Typically **transaction identifiers**: `invoice_number`, `order_id`, `receipt_number`.
- Useful for grouping/filtering line items belonging to the same transaction.

```
Sales_Fact
+-----------+-----------+----------+----------------+--------+
| date_key  | prod_key  | cust_key | invoice_number | amount |
+-----------+-----------+----------+----------------+--------+
|   2026091 |   55      |   101    |  INV-0098      |  50.00 |   <- invoice_number is degenerate
```

---

## 6. Conformed Dimension

A dimension that is **shared and consistent across multiple fact tables or data marts**, with the
same structure, keys, and meaning everywhere.

- Example: A single `Dim_Date` or `Dim_Customer` used by `Sales_Fact`, `Returns_Fact`, and
  `Shipments_Fact`.
- ✅ Enables **cross-process analysis** ("compare sales vs returns by customer") and enterprise consistency.

```
        Dim_Date  (conformed — same for both)
        /       \
 Sales_Fact   Returns_Fact
        \       /
      Dim_Customer  (conformed)
```

---

## 7. Slowly Changing Dimensions (SCD)

Strategies for handling changes to dimension attributes over time (e.g., a customer's address).

| Type | Name | Behavior |
|------|------|----------|
| Type 0 | Retain original | Never change the value. |
| Type 1 | Overwrite | Replace old value; no history. |
| Type 2 | Add new row | Insert new version with start/end dates + current flag; full history. |
| Type 3 | Previous value column | Add a column holding the prior value; limited history. |

**Type 2 example (most common):**

| Cust_Key | Cust_ID | Address | Start_Date | End_Date | Current |
|----------|---------|---------|------------|-----------|---------|
| 101 | C001 | London | 01-OCT-2025 | 15-JAN-2026 | No |
| 102 | C001 | Manchester | 16-JAN-2026 | 31-DEC-9999 | Yes |

---

## 8. Grain Definition (Very Important)

The **grain** is the precise level of detail represented by **one row** in a fact table. Defining the
grain is the **first and most critical** step in dimensional modeling — everything else (dimensions,
measures) depends on it.

- Declare grain in a single clear sentence.
- Examples:
  - "One row per **product per transaction**" (line-item grain).
  - "One row per **order**" (order grain).
  - "One row per **day per store** for inventory snapshots."

**Rule:** Never mix grains in one fact table. All measures must be consistent with the declared grain
(additive at that level).

---

## 9. Partitioning Strategy

Dividing a large table into smaller, independently manageable **partitions** to improve performance
and maintenance.

**Common strategies:**
- **Range partitioning** — by date (most common): partition per month/day (`sales_2026_09`).
- **List partitioning** — by discrete values (e.g., region, country).
- **Hash partitioning** — even distribution across N buckets when no natural range exists.

**Benefits:**
- **Partition pruning** — queries scan only relevant partitions → faster.
- Easier archival/deletion (drop old partitions).
- Parallel loads and maintenance.

```sql
-- Range partition by month
CREATE TABLE Sales_Fact (...)
PARTITION BY RANGE (date_key) (
  PARTITION p2026_08 VALUES LESS THAN (20260901),
  PARTITION p2026_09 VALUES LESS THAN (20261001)
);
```

---

## 10. CDC vs Incremental Load

Both aim to move only **changed data** (not full reloads), but differ in mechanism.

**Incremental Load** — Pull rows changed since the last run, usually using a **watermark** column
like `last_modified_date` or an increasing ID.
- ✅ Simple to implement.
- ❌ Misses hard deletes; needs a reliable timestamp column.

**Change Data Capture (CDC)** — Reads the database's **transaction log** to capture every
`INSERT`/`UPDATE`/`DELETE` in near real-time.
- ✅ Captures deletes, low latency, low source load.
- ❌ More complex; requires DB/tooling support (e.g., Debezium, SQL Server CDC).

| Aspect | Incremental Load | CDC |
|--------|------------------|-----|
| Mechanism | Watermark column | Transaction log |
| Captures deletes | Usually no | Yes |
| Latency | Batch | Near real-time |
| Complexity | Low | Higher |

---

## 11. Late Arriving Dimensions

Occurs when a **fact record arrives before its corresponding dimension record** exists
(e.g., a sale for a brand-new customer not yet loaded into `Dim_Customer`).

**Handling strategies:**
1. **Inferred (placeholder) member** — Insert a stub dimension row with the natural key and default
   ("Unknown") attributes, assign a surrogate key, and link the fact immediately. Update the stub
   with real attributes when the dimension data later arrives.
2. **Hold / reprocess** — Park the fact in a staging area and reprocess once the dimension appears
   (riskier — can delay facts).

```sql
-- Inferred member: create stub so the fact isn't lost
INSERT INTO Dim_Customer (cust_key, cust_id, name, inferred_flag)
VALUES (9999, 'C999', 'Unknown', 'Y');
-- Later, when real data arrives, update the stub in place (Type 1).
```

**Best practice:** Use inferred members so facts are never dropped, with an `inferred_flag` to track
and backfill them.
