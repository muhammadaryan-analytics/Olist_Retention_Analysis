## Step 1: Validate the Core Problem
- Before diagnose anything, confirm the retention problem is real and worth solving.
```sql
-- Cohort analysis by first purchase year
SELECT 
    EXTRACT(YEAR FROM cohort_month) as cohort_year,
    COUNT(DISTINCT customer_unique_id) as total_customers,
    COUNT(DISTINCT CASE WHEN is_repeat_customer = 1 THEN customer_unique_id END) as repeat_customers,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN is_repeat_customer = 1 THEN customer_unique_id END) / 
          COUNT(DISTINCT customer_unique_id), 2) as repeat_rate_pct
FROM summary
GROUP BY EXTRACT(YEAR FROM cohort_month)
ORDER BY cohort_year;
```
Here is the output of the query:

<img width="550" height="150" alt="Screenshot 2025-11-29 at 3 59 35 PM" src="https://github.com/user-attachments/assets/91c17497-d8d7-4d2c-b22e-fd0b1ddf05db" />

- **93,102** out of **96,099** customers **(96.9%)** made exactly one purchase. That's the core problem.
- **3.1% repeat rate.**


## Step 2: Test the Delivery Hypothesis
The brief assumes delivery performance is the main driver.
- Do repeat customers have better first-order delivery experiences?
- Do customers with late first deliveries return at lower rates?
- Do customers with bad first-order reviews return at lower rates?




```sql
-- For first-time orders only, does late delivery predict whether they return?
SELECT 
    CASE WHEN is_late = 1 THEN 'Late Delivery' ELSE 'On-Time Delivery' END as delivery_performance,
    COUNT(DISTINCT customer_unique_id) as customers,
    COUNT(DISTINCT CASE WHEN is_repeat_customer = 1 THEN customer_unique_id END) as customers_who_returned,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN is_repeat_customer = 1 THEN customer_unique_id END) / 
          COUNT(DISTINCT customer_unique_id), 2) as return_rate_pct
FROM (
    SELECT DISTINCT ON (customer_unique_id)
        customer_unique_id,
        is_late,
        is_repeat_customer,
        order_purchase_timestamp
    FROM summary
    WHERE order_status = 'delivered'
    ORDER BY customer_unique_id, order_purchase_timestamp
) first_orders
GROUP BY CASE WHEN is_late = 1 THEN 'Late Delivery' ELSE 'On-Time Delivery' END;
```
Here is the output of the query:

<img width="517" height="94" alt="Screenshot 2025-11-30 at 11 47 39 AM" src="https://github.com/user-attachments/assets/c5260957-918e-4cd8-b191-b5b9f1a22039" />

**Delivery performance barely matters:**

- Late delivery: 2.57% return rate
- On-time delivery: 3.24% return rate
- Difference: 0.67 percentage points

```sql
-- Does review score on first order predict return?
SELECT 
    CASE 
        WHEN review_score >= 4 THEN 'Positive (4-5 stars)'
        WHEN review_score = 3 THEN 'Neutral (3 stars)'
        ELSE 'Negative (1-2 stars)'
    END as review_category,
    COUNT(DISTINCT customer_unique_id) as customers,
    COUNT(DISTINCT CASE WHEN is_repeat_customer = 1 THEN customer_unique_id END) as customers_who_returned,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN is_repeat_customer = 1 THEN customer_unique_id END) / 
          COUNT(DISTINCT customer_unique_id), 2) as return_rate_pct
FROM (
    SELECT DISTINCT ON (customer_unique_id)
        customer_unique_id,
        review_score,
        is_repeat_customer,
        order_purchase_timestamp
    FROM summary
    WHERE order_status = 'delivered' AND review_score IS NOT NULL
    ORDER BY customer_unique_id, order_purchase_timestamp
) first_orders
GROUP BY CASE 
    WHEN review_score >= 4 THEN 'Positive (4-5 stars)'
    WHEN review_score = 3 THEN 'Neutral (3 stars)'
    ELSE 'Negative (1-2 stars)'
END;
```
Here is the output of the query:

<img width="489" height="121" alt="Screenshot 2025-11-30 at 11 50 11 AM" src="https://github.com/user-attachments/assets/f66f5508-aed9-4c96-af40-0c36d02df66b" />


**Review scores don't matter at all:**

- Negative (1-2 stars): 3.10% return
- Neutral (3 stars): 3.18% return
- Positive (4-5 stars): 3.18% return

Even customers who had perfect experiences (on-time delivery, 5-star reviews) still only come back 3.2% of the time. That's the same rate as customers who had terrible experiences.
This tells you the problem is NOT operational.
It's not delivery speed. It's not seller quality. It's not product issues. Those things barely move the needle.











