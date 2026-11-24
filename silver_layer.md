
---

# 🧱 Silver Layer Documentation

This document outlines the creation of **Silver Layer** dimension tables from the `tb_raw_source` Bronze table for downstream analytics and modeling.

---

## ⚙️ Overview

* **Source**: `punit_dbx.bronze.tb_raw_source`
* **Target Schema**: `punit_dbx.silver`
* **Purpose**: Normalize raw data into clean, deduplicated, hashed dimension tables
* **Hashing**: Using `md5(lower(trim(col)))` and `hash(lower(trim(col)))` for consistent surrogate keys

---

## 📋 Dimension Tables

### 1. `dim_country`

```sql
CREATE OR REPLACE TABLE punit_dbx.silver.dim_country AS   
SELECT 
    md5(lower(trim(country))) AS country_id,
    country
FROM (
    SELECT DISTINCT country
    FROM punit_dbx.bronze.tb_raw_source
) AS distinct_country;
```

---

### 2. `dim_state`

```sql
CREATE OR REPLACE TABLE punit_dbx.silver.dim_state AS   
SELECT 
    md5(lower(trim(concat_ws('|', country, state)))) AS state_id,
    md5(lower(trim(country))) AS country_id,
    state,
    country
FROM (
    SELECT DISTINCT state, country
    FROM punit_dbx.bronze.tb_raw_source
) AS distinct_state;
```

---

### 3. `dim_city`

```sql
CREATE OR REPLACE TABLE punit_dbx.silver.dim_city AS  
SELECT 
    md5(lower(trim(concat_ws('|', country, state, city)))) AS city_id,
    md5(lower(trim(concat_ws('|', country, state)))) AS state_id,
    md5(lower(trim(country))) AS country_id,
    city,
    state,
    country
FROM (
    SELECT DISTINCT city, state, country
    FROM punit_dbx.bronze.tb_raw_source
) AS distinct_city;
```

---

### 4. `dim_location`

```sql
CREATE OR REPLACE TEMP VIEW tmp_vw_location AS 
SELECT DISTINCT
    address,
    zip_code,
    city,
    state,
    country,
    md5(lower(trim(concat_ws('|', country, state, city)))) AS city_id,
    md5(lower(trim(concat_ws('|', country, state)))) AS state_id,
    md5(lower(trim(country))) AS country_id
FROM punit_dbx.bronze.tb_raw_source;

CREATE OR REPLACE TABLE punit_dbx.silver.dim_location AS 
SELECT DISTINCT
    md5(lower(trim(concat_ws('|', address, zip_code, city, state, country)))) AS location_id,
    address,
    zip_code,
    city_id,
    state_id,
    country_id
FROM tmp_vw_location;
```

---

### 5. `dim_order_status`

```sql
CREATE OR REPLACE TABLE punit_dbx.silver.dim_order_status AS   
SELECT 
    md5(lower(trim(order_status))) AS order_status_id,
    order_status
FROM (
    SELECT DISTINCT order_status
    FROM punit_dbx.bronze.tb_raw_source
) AS distinct_status;
```

---

### 6. `dim_payment_method`

```sql
CREATE OR REPLACE TABLE punit_dbx.silver.dim_payment_method AS   
SELECT 
    md5(lower(trim(payment_method))) AS payment_method_id,
    payment_method
FROM (
    SELECT DISTINCT payment_method
    FROM punit_dbx.bronze.tb_raw_source
);
```

---

### 7. `dim_date`

```sql
CREATE OR REPLACE TABLE punit_dbx.silver.dim_date (
    date_id       BIGINT,
    date          DATE,
    month         INT,
    year          INT,
    day_of_year   INT,
    day_of_month  INT,
    day_of_week   INT,
    day_name      STRING,
    quarter       INT
);

INSERT INTO punit_dbx.silver.dim_date
SELECT
    YEAR(d) * 10000 + DAYOFYEAR(d) AS date_id,
    d AS date,
    MONTH(d) AS month,
    YEAR(d) AS year,
    DAYOFYEAR(d) AS day_of_year,
    DAY(d) AS day_of_month,
    CASE WHEN DAYOFWEEK(d) = 1 THEN 7 ELSE DAYOFWEEK(d) - 1 END AS day_of_week,
    DATE_FORMAT(d, 'EEEE') AS day_name,
    QUARTER(d) AS quarter
FROM (
    SELECT EXPLODE(SEQUENCE(DATE('2024-01-01'), DATE('2026-12-31'), INTERVAL 1 DAY)) AS d
);
```

---

### 8. `dim_product_brand`

```sql
CREATE OR REPLACE TABLE punit_dbx.silver.dim_product_brand AS   
SELECT 
    md5(lower(trim(product_brand))) AS product_brand_id,
    product_brand
FROM (
    SELECT DISTINCT product_brand
    FROM punit_dbx.bronze.tb_raw_source
);
```

---

### 9. `dim_product_category`

```sql
CREATE OR REPLACE TABLE punit_dbx.silver.dim_product_category AS   
SELECT 
    md5(lower(trim(product_category))) AS product_category_id,
    product_category
FROM (
    SELECT DISTINCT product_category
    FROM punit_dbx.bronze.tb_raw_source
);
```

---

### 10. `dim_customer`

Handles inconsistencies in `gender` per `customer_id`.

```sql
-- Customers with multiple genders (resolve by majority)
WITH multi_gender_customers AS (
    SELECT customer_id
    FROM punit_dbx.bronze.tb_raw_source
    GROUP BY customer_id
    HAVING COUNT(DISTINCT gender) > 1
),
resolved_gender AS (
    SELECT customer_id, FIRST(gender) AS gender
    FROM (
        SELECT customer_id, gender, COUNT(*) AS cnt
        FROM punit_dbx.bronze.tb_raw_source
        WHERE customer_id IN (SELECT customer_id FROM multi_gender_customers)
        GROUP BY customer_id, gender
        ORDER BY cnt DESC
    ) GROUP BY customer_id
),
final_customer_data AS (
    SELECT DISTINCT customer_id, gender, email, birth_date
    FROM punit_dbx.bronze.tb_raw_source
    WHERE customer_id NOT IN (SELECT customer_id FROM multi_gender_customers)
    UNION ALL
    SELECT customer_id, gender, email, birth_date
    FROM resolved_gender
    JOIN punit_dbx.bronze.tb_raw_source USING (customer_id, gender)
)
CREATE OR REPLACE TABLE punit_dbx.silver.dim_customer AS 
SELECT DISTINCT * FROM final_customer_data;
```

---

### 11. `dim_customer_segment` (SCD Type 2)

#### Multi-Segment Customers

```sql
CREATE OR REPLACE TEMP VIEW customer_multi_segment_scd AS
WITH distinct_data AS (
    SELECT DISTINCT customer_id, customer_segment, datetimestamp
    FROM punit_dbx.bronze.tb_raw_source
),
multi_seg AS (
    SELECT customer_id FROM distinct_data GROUP BY customer_id HAVING COUNT(DISTINCT customer_segment) > 1
),
seg_with_bounds AS (
    SELECT *,
        LAG(customer_segment) OVER (PARTITION BY customer_id ORDER BY datetimestamp) AS prev_seg,
        LEAD(datetimestamp) OVER (PARTITION BY customer_id ORDER BY datetimestamp) AS next_ts
    FROM distinct_data
    WHERE customer_id IN (SELECT customer_id FROM multi_seg)
),
start_periods AS (
    SELECT customer_id, customer_segment, datetimestamp AS start_date,
           LEAD(datetimestamp) OVER (PARTITION BY customer_id ORDER BY datetimestamp) AS end_date
    FROM seg_with_bounds
    WHERE prev_seg IS NULL OR customer_segment != prev_seg
)
SELECT 
    customer_id,
    customer_segment,
    start_date,
    end_date,
    CASE WHEN end_date IS NULL THEN TRUE ELSE FALSE END AS is_active
FROM start_periods;
```

#### Single-Segment Customers

```sql
CREATE OR REPLACE TEMP VIEW customer_single_segment_scd AS
WITH multi_seg AS (
    SELECT customer_id FROM punit_dbx.bronze.tb_raw_source GROUP BY customer_id HAVING COUNT(DISTINCT customer_segment) > 1
),
ranked AS (
    SELECT DISTINCT customer_id, customer_segment, datetimestamp,
           RANK() OVER (PARTITION BY customer_id ORDER BY datetimestamp) AS rnk
    FROM punit_dbx.bronze.tb_raw_source
    WHERE customer_id NOT IN (SELECT customer_id FROM multi_seg)
)
SELECT 
    customer_id,
    customer_segment,
    datetimestamp AS start_date,
    NULL AS end_date,
    TRUE AS is_active
FROM ranked
WHERE rnk = 1;
```

#### Final Segment Table

```sql
CREATE OR REPLACE TABLE punit_dbx.silver.dim_customer_segment AS
SELECT 
    *,
    md5(lower(trim(customer_segment))) AS customer_segment_id
FROM (
    SELECT * FROM customer_multi_segment_scd
    UNION ALL
    SELECT * FROM customer_single_segment_scd
);
```

--- 

### 12. `dim_product` (SCD for Price Tracking)

Tracks changes in product pricing over time using SCD logic.

```sql
-- Step 1: Track price changes per product
CREATE OR REPLACE TEMP VIEW product_scd AS
WITH cte AS (
  SELECT 
    DISTINCT product_id, amount, datetimestamp, product_brand, product_category,
    LAG(amount) OVER (PARTITION BY product_id ORDER BY datetimestamp) AS lag_amount
  FROM punit_dbx.bronze.tb_raw_source
),
cte2 AS (
  SELECT *,
    CASE WHEN lag_amount IS NULL OR amount != lag_amount THEN datetimestamp ELSE NULL END AS start_date
  FROM cte
),
cte3 AS (
  SELECT *,
    LEAD(start_date) OVER (PARTITION BY product_id ORDER BY start_date) AS end_date
  FROM cte2
  WHERE start_date IS NOT NULL
),
cte4 AS (
  SELECT 
    product_id, amount, start_date, product_brand, product_category,
    end_date,
    CASE WHEN end_date IS NULL THEN TRUE ELSE FALSE END AS is_active
  FROM cte3
)
SELECT * FROM cte4;
```

```sql
-- Step 2: Get most common product name per product_id
CREATE OR REPLACE TEMP VIEW product_list AS
WITH ranked_cte AS (
  SELECT product_id, product_type, COUNT(*) AS cnt,
         ROW_NUMBER() OVER (PARTITION BY product_id ORDER BY COUNT(*) DESC) AS rn
  FROM punit_dbx.bronze.tb_raw_source
  GROUP BY product_id, product_type
)
SELECT product_id, product_type
FROM ranked_cte
WHERE rn = 1;
```

```sql
-- Step 3: Merge name + hash foreign keys
CREATE OR REPLACE TEMP VIEW product_list_df AS
SELECT 
  ps.*, 
  pl.product_type AS product_name,
  hash(lower(ps.product_category)) AS product_category_id,
  hash(lower(ps.product_brand)) AS product_brand_id
FROM product_list pl
JOIN product_scd ps ON pl.product_id = ps.product_id;
```

```sql
-- Step 4: Save to silver layer
CREATE OR REPLACE TABLE punit_dbx.silver.dim_product AS
SELECT 
  product_id, 
  product_name, 
  product_category_id,
  product_brand_id,
  amount, 
  start_date, 
  end_date, 
  is_active
FROM product_list_df;
```

---

### 13. `dim_feedback` (SCD for Feedback + Rating)

Tracks changes in product feedback and ratings per customer.

```sql
-- Step 1: SCD logic for feedback/rating
CREATE OR REPLACE TEMP VIEW feedback_scd AS
WITH cte AS (
  SELECT DISTINCT
    product_id, customer_id, feedback, rating, datetimestamp,
    LAG(feedback) OVER (PARTITION BY product_id ORDER BY datetimestamp) AS lag_feed,
    LAG(rating) OVER (PARTITION BY product_id ORDER BY datetimestamp) AS lag_rating
  FROM punit_dbx.bronze.tb_raw_source
),
cte2 AS (
  SELECT *,
    CASE 
      WHEN lag_feed IS NULL AND lag_rating IS NULL THEN datetimestamp
      WHEN lag_feed != feedback OR lag_rating != rating THEN datetimestamp
      ELSE NULL
    END AS start_date
  FROM cte
),
cte3 AS (
  SELECT *,
    LEAD(start_date) OVER (PARTITION BY product_id ORDER BY start_date) AS end_date
  FROM cte2
),
cte4 AS (
  SELECT *,
    CASE WHEN end_date IS NULL THEN TRUE ELSE FALSE END AS is_active
  FROM cte3
)
SELECT * FROM cte4;
```

```sql
-- Step 2: Generate unique feedback_id
CREATE OR REPLACE TEMP VIEW feedback_scd_unique AS
WITH cte AS (
  SELECT 
    md5(concat(
      coalesce(product_id, ''), coalesce(customer_id, ''),
      coalesce(datetimestamp, ''), coalesce(feedback, ''),
      coalesce(try_cast(rating AS STRING), '')
    )) AS feedback_id,
    product_id, customer_id, feedback, rating, start_date, end_date, is_active
  FROM feedback_scd
)
SELECT * FROM cte;
```

```sql
-- Step 3: Save to silver layer
CREATE OR REPLACE TABLE punit_dbx.silver.dim_feedback AS 
SELECT DISTINCT * FROM feedback_scd_unique;
```

---

### 14. `dim_shipping`

```sql
CREATE OR REPLACE TABLE punit_dbx.silver.dim_shipping AS   
SELECT 
   hash(lower(shipping_method)) AS shipping_method_id,
   shipping_method
FROM (
    SELECT DISTINCT shipping_method
    FROM punit_dbx.bronze.tb_raw_source
);
```

---

## 📦 Fact Table

### 15. `fact_orders_details`

Combines keys and measures for analytics.

```sql
-- Step 1: Create clean base view
CREATE OR REPLACE VIEW tmp_raw_source AS 
SELECT DISTINCT * FROM punit_dbx.bronze.tb_raw_source;
```

```sql
-- Step 2: Construct fact with surrogate keys
CREATE OR REPLACE TEMP VIEW tmp_orders_details AS
SELECT  
  transaction_id,
  customer_id,
  product_id,
  datetimestamp,
  phone,
  quantity,
  birth_date,
  hash(lower(product_brand)) AS product_brand_id,
  hash(lower(product_category)) AS product_category_id,
  hash(lower(shipping_method)) AS shipping_method_id,
  md5(lower(trim(concat_ws('|', address, zip_code, city, state, country)))) AS location_id,
  hash(lower(payment_method)) AS payment_method_id,
  hash(lower(order_status)) AS order_status_id,
  hash(lower(customer_segment)) AS customer_segment_id,
  md5(concat(
    coalesce(product_id, ''),
    coalesce(customer_id, ''),
    coalesce(datetimestamp, ''),
    coalesce(feedback, ''),
    coalesce(try_cast(rating AS STRING), '')
  )) AS feedback_id,
  YEAR(datetimestamp) * 10000 + DAYOFYEAR(datetimestamp) AS date_id
FROM tmp_raw_source;
```

```sql
-- Step 3: Save to Silver Layer
CREATE OR REPLACE TABLE punit_dbx.silver.fact_orders_details AS
SELECT * FROM tmp_orders_details;
```

---

## ✅ Summary

| Table Name             | Description                                       | SCD Type   | Notes                                                                 |
| ---------------------- | ------------------------------------------------- | ---------- | --------------------------------------------------------------------- |
| `dim_country`          | Unique list of countries                          | -          | Uses `hash(lower(country))` as surrogate key                          |
| `dim_state`            | Unique list of states with country linkage        | -          | Includes `state_id` and `country_id` from hashed lowercased values    |
| `dim_city`             | Unique list of cities linked to state & country   | -          | `city_id`, `state_id`, and `country_id` from hash values              |
| `dim_location`         | Unique addresses with generated `location_id`     | -          | Combines address components and hashes with `MD5()`                   |
| `dim_order_status`     | Distinct order statuses                           | -          | Uses `hash(lower(order_status))`                                      |
| `dim_payment_method`   | Distinct payment methods                          | -          | Uses `hash(lower(payment_method))`                                    |
| `dim_date`             | Calendar table with day/month/year/quarter fields | -          | Covers 2024–2026 using `explode(sequence(...))`                       |
| `dim_product_brand`    | Distinct product brands                           | -          | Uses `hash(lower(product_brand))`                                     |
| `dim_product_category` | Distinct product categories                       | -          | Uses `hash(lower(product_category))`                                  |
| `dim_customer`         | Cleaned customer details                          | -          | Resolves gender conflicts and deduplicates using temporary views      |
| `dim_customer_segment` | Customer segment history tracking                 | SCD Type 2 | Tracks historical changes in customer segments                        |
| `dim_product`          | Product info with price versioning                | SCD Type 2 | Tracks changes in product price, brand, category                      |
| `dim_feedback`         | Feedback/rating history                           | SCD Type 2 | Tracks customer feedback/rating changes over time                     |
| `dim_shipping`         | Shipping method reference                         | -          | Extracts distinct shipping methods with hash keys                     |
| `fact_orders_details`  | Central fact table for orders                     | -          | Combines all dimension keys and measures into one detailed fact table |

---
