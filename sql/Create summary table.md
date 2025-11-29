### Goal: Combine all 9 tables into a single summary table with derived columns.

- Create summary table with customer, order, and product details

```sql
-- Step 1: Create the summary table structure
CREATE TABLE IF NOT EXISTS summary (
    customer_unique_id TEXT,
    customer_city TEXT,
    customer_state TEXT,
    order_id TEXT,
    order_status TEXT,
    order_purchase_timestamp TIMESTAMP,
    order_approved_at TIMESTAMP,
    order_delivered_carrier_date TIMESTAMP,
    order_delivered_customer_date TIMESTAMP,
    order_estimated_delivery_date TIMESTAMP,
    shipping_limit_date TIMESTAMP,
    seller_id TEXT,
    seller_city TEXT,
    seller_state TEXT,
    product_id TEXT,
    product_category TEXT,
    price NUMERIC,
    freight_value NUMERIC,
    review_score NUMERIC,
    delivery_time_days NUMERIC,
    is_late BOOLEAN,
    is_first_time_buyer BOOLEAN,
    is_repeat_customer BOOLEAN,
    order_value NUMERIC,
    cohort_month TEXT,
    days_since_last_order NUMERIC,
    orders_in_last_90d INT,
    orders_in_last_180d INT
);
```

```sql
-- Step 2: Insert all enriched data
INSERT INTO summary
SELECT
    c.customer_unique_id,
    c.customer_city,
    c.customer_state,
    o.order_id,
    o.order_status,
    o.order_purchase_timestamp,
    o.order_approved_at,
    o.order_delivered_carrier_date,
    o.order_delivered_customer_date,
    o.order_estimated_delivery_date,
    o.shipping_limit_date,
    s.seller_id,
    s.seller_city,
    s.seller_state,
    p.product_id,
    p.product_category,
    p.price,
    o.freight_value,
    r.review_score,
    (o.order_delivered_customer_date::date - o.order_purchase_timestamp::date) AS delivery_time_days,
    CASE
        WHEN o.order_delivered_customer_date IS NOT NULL
             AND o.order_estimated_delivery_date IS NOT NULL
             AND o.order_delivered_customer_date::date > o.order_estimated_delivery_date::date
        THEN true
        ELSE false
    END AS is_late,
    CASE
        WHEN ROW_NUMBER() OVER(PARTITION BY c.customer_unique_id ORDER BY o.order_purchase_timestamp) = 1
        THEN true
        ELSE false
    END AS is_first_time_buyer,
    CASE
        WHEN COUNT(o.order_id) OVER(PARTITION BY c.customer_unique_id) > 1 THEN true
        ELSE false
    END AS is_repeat_customer,
    (p.price + o.freight_value) AS order_value,
    TO_CHAR(MIN(o.order_purchase_timestamp) OVER(PARTITION BY c.customer_unique_id)::date, 'YYYY-MM') AS cohort_month,
    EXTRACT(EPOCH FROM (MAX(o.order_purchase_timestamp) OVER(PARTITION BY c.customer_unique_id) - (SELECT MAX(order_purchase_timestamp) FROM orders)))/86400.0 * -1 AS days_since_last_order,
    SUM(CASE WHEN o.order_purchase_timestamp >= ((SELECT MAX(order_purchase_timestamp) FROM orders) - INTERVAL '90 days') THEN 1 ELSE 0 END) OVER(PARTITION BY c.customer_unique_id) AS orders_in_last_90d,
    SUM(CASE WHEN o.order_purchase_timestamp >= ((SELECT MAX(order_purchase_timestamp) FROM orders) - INTERVAL '180 days') THEN 1 ELSE 0 END) OVER(PARTITION BY c.customer_unique_id) AS orders_in_last_180d
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN sellers s ON oi.seller_id = s.seller_id
LEFT JOIN order_reviews r ON o.order_id = r.order_id;
```


