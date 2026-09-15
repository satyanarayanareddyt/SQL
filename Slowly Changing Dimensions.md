# Slowly Changing Dimensions (SCD)

Slowly Changing Dimensions (SCDs) describe how a data warehouse handles changes to
dimensional attributes (like a customer's address) over time. The example below tracks
customer **C001 – John Miller** as his address changes across **London → Manchester → Lincoln**.

## Summary

| Type | Name | Description | When to Use |
|------|------|-------------|-------------|
| **Type 0** | Retain original value | The attribute never changes after the record is first created. New values are ignored. | Fixed/immutable attributes such as original registration date, date of birth, or original signup location. |
| **Type 1** | Overwrite value | The old value is overwritten with the new value. No history is kept. | When history is not needed and only the latest value matters (e.g., correcting a typo or a value where the past is irrelevant). |
| **Type 2** | Add new row | A new row is inserted for each change, with start/end dates and a current flag. Full history is preserved. | When complete historical tracking is required and point-in-time analysis matters (most common for auditing/reporting). |
| **Type 3** | Previous value column | Adds a separate column to store the previous value alongside the current one. Limited history. | When you only need the immediately prior value (e.g., current vs. previous address/region). |

---

## Type 0 — Retain Original Value

The value captured when the record is first created is kept forever. Even if the real-world
attribute changes, the stored value is **not** updated.

**Example:** John registers in **London**. Months later he moves to **Manchester**, but the
warehouse still stores his address as **London** because it reflects the value *at registration*.

| Customer_ID | Name | Address |
|-------------|------|---------|
| C001 | John Miller | London |

### How to implement
- Load the attribute only during the initial `INSERT`.
- Exclude it from all update logic in the ETL/merge process.

```sql
-- Insert only; never update Address for existing customers
INSERT INTO dim_customer (customer_id, name, address)
VALUES ('C001', 'John Miller', 'London');
```

---

## Type 1 — Overwrite Value

The existing value is simply replaced with the new value. There is **no history** — once
overwritten, the previous value is gone.

**Example:** John's address starts as **London**, then he moves to **Manchester**. The row is
updated in place and now shows **Manchester** only.

| Customer_ID | Name | Address |
|-------------|------|---------|
| C001 | John Miller | Manchester |

### How to implement
- On a change, run an `UPDATE` that overwrites the current value.

```sql
UPDATE dim_customer
SET address = 'Manchester'
WHERE customer_id = 'C001';
```

```sql
-- Or via MERGE (upsert)
MERGE INTO dim_customer AS tgt
USING staging_customer AS src
  ON tgt.customer_id = src.customer_id
WHEN MATCHED THEN
  UPDATE SET tgt.address = src.address
WHEN NOT MATCHED THEN
  INSERT (customer_id, name, address)
  VALUES (src.customer_id, src.name, src.address);
```

---

## Type 2 — Add New Row

Each change creates a **new row** while old rows are retained. Effective-dating columns
(`Start_Date`, `End_Date`) and a `Current` flag distinguish the active record from historical
ones. This preserves the **full history** of changes.

**Example:** John moves London → Manchester → Lincoln, producing three rows:

| Cust_Key | Cust_ID | Name | Address | Start_Date | End_Date | Current |
|----------|---------|------|---------|------------|-----------|---------|
| 101 | C001 | John Miller | London | 01-OCT-2025 | 15-JAN-2026 | No |
| 102 | C001 | John Miller | Manchester | 16-JAN-2026 | 24-MAR-2026 | No |
| 103 | C001 | John Miller | Lincoln | 25-MAR-2026 | 31-DEC-9999 | Yes |

- `Cust_Key` is a **surrogate key** (unique per row/version).
- `Cust_ID` is the natural/business key (same across versions).
- `End_Date = 31-DEC-9999` marks the open-ended, current record.

### How to implement
1. Expire the current row (set `End_Date` and `Current = 'No'`).
2. Insert a new row with the new value, a new surrogate key, `Current = 'Yes'`, and open `End_Date`.

```sql
-- Step 1: Expire the existing current row
UPDATE dim_customer
SET end_date = '2026-03-24',
    current_flag = 'No'
WHERE cust_id = 'C001' AND current_flag = 'Yes';

-- Step 2: Insert the new version
INSERT INTO dim_customer
  (cust_key, cust_id, name, address, start_date, end_date, current_flag)
VALUES
  (103, 'C001', 'John Miller', 'Lincoln', '2026-03-25', '9999-12-31', 'Yes');
```

---

## Type 3 — Previous Value Column

A dedicated column stores the **previous** value next to the current one. This keeps a
**limited history** — typically only one step back.

**Example:** When John moves to Lincoln, `Current_Address` becomes **Lincoln** and
`Previous_Address` holds **Manchester** (the value that was current before).

| Customer_ID | Name | Current_Address | Previous_Address |
|-------------|------|-----------------|-------------------|
| C001 | John Miller | Lincoln | Manchester |

### How to implement
- On a change, copy the current value into the previous column, then set the new current value.

```sql
UPDATE dim_customer
SET previous_address = current_address,   -- shift current -> previous
    current_address  = 'Lincoln'          -- set new current
WHERE customer_id = 'C001';
```

---

## Choosing the Right Type

| Requirement | Recommended Type |
|-------------|------------------|
| Value must never change | Type 0 |
| Only latest value matters, no history | Type 1 |
| Full audit trail / point-in-time reporting | Type 2 |
| Only need current + immediately prior value | Type 3 |
