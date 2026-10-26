# SWEN466 — Assignment 1: Data Quality Analysis and ERD Design

Use this notebook in the course environment where the five CSV files have already been loaded into the DuckDB `raw` schema. The SQL below is a working query set based on the assignment brief. Run the cells and record the actual results; the data files were not included with the assignment PDF supplied here, so the observed counts and cardinalities cannot be pre-filled reliably.

## Exercise 1

```sql
-- Exercise 1: Inspect the five source tables (schema and sample rows).
-- Run each DESCRIBE and SELECT separately in DuckDB because DESCRIBE returns
-- table-specific schema output and SELECT returns sample records.
DESCRIBE raw.customers;
SELECT * FROM raw.customers LIMIT 5;

DESCRIBE raw.orders;
SELECT * FROM raw.orders LIMIT 5;

DESCRIBE raw.order_items;
SELECT * FROM raw.order_items LIMIT 5;

DESCRIBE raw.products;
SELECT * FROM raw.products LIMIT 5;

DESCRIBE raw.sellers;
SELECT * FROM raw.sellers LIMIT 5;
```

## Exercise 2

```sql
-- Exercise 2: Validate candidate primary keys.
-- A valid PK must have no NULLs and no duplicate key values.
WITH checks AS (
    SELECT 'customers' AS table_name, 'customer_id' AS candidate_key,
           COUNT(*) AS row_count, COUNT(customer_id) AS non_null_count,
           COUNT(DISTINCT customer_id) AS distinct_key_count
    FROM raw.customers
    UNION ALL
    SELECT 'orders', 'order_id', COUNT(*), COUNT(order_id), COUNT(DISTINCT order_id)
    FROM raw.orders
    UNION ALL
    SELECT 'products', 'product_id', COUNT(*), COUNT(product_id), COUNT(DISTINCT product_id)
    FROM raw.products
    UNION ALL
    SELECT 'sellers', 'seller_id', COUNT(*), COUNT(seller_id), COUNT(DISTINCT seller_id)
    FROM raw.sellers
    UNION ALL
    SELECT 'order_items', 'order_id + order_item_id', COUNT(*),
           COUNT(*) FILTER (WHERE order_id IS NOT NULL AND order_item_id IS NOT NULL),
           COUNT(DISTINCT (order_id, order_item_id))
    FROM raw.order_items
)
SELECT *, (row_count = non_null_count AND row_count = distinct_key_count) AS passes_pk_check
FROM checks
ORDER BY table_name;
```

## Exercise 3

```sql
-- Exercise 3: Check foreign keys with anti-joins.
-- Zero returned rows for a relationship means no child rows have a missing parent.
SELECT 'orders.customer_id -> customers.customer_id' AS relationship,
       COUNT(*) AS orphan_count
FROM raw.orders c
ANTI JOIN raw.customers p ON c.customer_id = p.customer_id
UNION ALL
SELECT 'order_items.order_id -> orders.order_id', COUNT(*)
FROM raw.order_items c
ANTI JOIN raw.orders p ON c.order_id = p.order_id
UNION ALL
SELECT 'order_items.product_id -> products.product_id', COUNT(*)
FROM raw.order_items c
ANTI JOIN raw.products p ON c.product_id = p.product_id
UNION ALL
SELECT 'order_items.seller_id -> sellers.seller_id', COUNT(*)
FROM raw.order_items c
ANTI JOIN raw.sellers p ON c.seller_id = p.seller_id;
```

## Exercise 4

```sql
-- Exercise 4: Measure child-row cardinality for each parent.
-- The min/max counts include parents with zero children by LEFT JOIN.
WITH child_counts AS (
    SELECT 'customers -> orders' AS relationship, p.customer_id AS parent_key,
           COUNT(c.order_id) AS child_count
    FROM raw.customers p
    LEFT JOIN raw.orders c ON c.customer_id = p.customer_id
    GROUP BY p.customer_id
    UNION ALL
    SELECT 'orders -> order_items', p.order_id, COUNT(c.order_id)
    FROM raw.orders p
    LEFT JOIN raw.order_items c ON c.order_id = p.order_id
    GROUP BY p.order_id
    UNION ALL
    SELECT 'products -> order_items', p.product_id, COUNT(c.product_id)
    FROM raw.products p
    LEFT JOIN raw.order_items c ON c.product_id = p.product_id
    GROUP BY p.product_id
    UNION ALL
    SELECT 'sellers -> order_items', p.seller_id, COUNT(c.seller_id)
    FROM raw.sellers p
    LEFT JOIN raw.order_items c ON c.seller_id = p.seller_id
    GROUP BY p.seller_id
)
SELECT relationship, MIN(child_count) AS min_children_per_parent,
       MAX(child_count) AS max_children_per_parent,
       COUNT(*) FILTER (WHERE child_count = 0) AS parents_with_zero_children,
       COUNT(*) AS parent_count
FROM child_counts
GROUP BY relationship
ORDER BY relationship;
```

## Exercise 5

```sql
-- Exercise 5: Check customer identifiers and junction-table grain.
-- Compare customer_id (customer record/relationship key) with customer_unique_id
-- (identifier intended to group the same customer across multiple records).
SELECT
    COUNT(*) AS customer_rows,
    COUNT(DISTINCT customer_id) AS distinct_customer_id,
    COUNT(DISTINCT customer_unique_id) AS distinct_customer_unique_id,
    COUNT(*) FILTER (WHERE customer_id IS NULL) AS null_customer_id,
    COUNT(*) FILTER (WHERE customer_unique_id IS NULL) AS null_customer_unique_id
FROM raw.customers;

-- Confirm order_items composite-key uniqueness.
SELECT COUNT(*) AS row_count,
       COUNT(DISTINCT (order_id, order_item_id)) AS distinct_composite_keys,
       COUNT(*) FILTER (WHERE order_id IS NULL OR order_item_id IS NULL) AS rows_with_null_key_part
FROM raw.order_items;
```