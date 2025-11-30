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

<img width="462" height="114" alt="Screenshot 2025-11-29 at 3 59 35 PM" src="https://github.com/user-attachments/assets/ab844ed6-c17c-47c2-a23a-34f8158f441d" />


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

**This tells you the problem is NOT operational.**

It's not delivery speed. It's not seller quality. It's not product issues. Those things barely move the needle.


## Step 3: Test the Category Hypothesis
If operations don't explain it, maybe it's product mix - people only need these items once.
- Are there ANY categories with 8-10%+ repeat rates?
- When people DO come back, what are they buying?
```sql
-- Repeat rate by first purchase category
SELECT 
    product_category,
    COUNT(DISTINCT customer_unique_id) as customers,
    COUNT(DISTINCT CASE WHEN is_repeat_customer = 1 THEN customer_unique_id END) as repeat_customers,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN is_repeat_customer = 1 THEN customer_unique_id END) / 
          COUNT(DISTINCT customer_unique_id), 2) as repeat_rate_pct
FROM summary
GROUP BY product_category
HAVING COUNT(DISTINCT customer_unique_id) >= 500  -- Only meaningful volume categories
ORDER BY repeat_rate_pct DESC;
```
Here is the output of the query:

<img width="525" height="663" alt="Screenshot 2025-11-30 at 12 18 11 PM" src="https://github.com/user-attachments/assets/31179859-dbb6-4d01-b6d8-ada5d5dba1a1" />



**Key findings:**

- Home Appliances shows **10.95% repeat rate** - triple the average.
- Fashion Bags at 9.12%. These prove that repeat purchases ARE possible on this platform.

**But even "naturally repeatable" categories are terrible:**

- Health Beauty: 4.16% (should be 20-30% for consumables)
- Pet Shop: 4.97% (pet food/supplies should be 25%+)
- Baby: 4.16% (diapers, formula should be high repeat)

**Volume is in the wrong categories:**

The top sellers (Bed Bath Table, Furniture, Sports Leisure) are 5-7% repeat categories, while naturally high-repeat categories like Pet Shop and Baby are tiny.
But here's the real insight: Even in the best categories, retention is catastrophic. Home appliances at 11% is still terrible. In a healthy marketplace, consumables should hit 25-40% repeat rates.


## Step 4: Test the Market Behavior Hypothesis
Something fundamental is broken with Brazilian e-commerce trust or Olist's market position in 2016-2018. Let's check payment methods and order values.

- If high-value customers return more, it suggests quality/trust issues with cheap items
- If certain regions have dramatically different repeat rates, it's a logistics/trust issue specific to those areas
- If everything is flat ~3% regardless of price or location, the problem is systemic platform trust
```sql
-- Do higher-value customers return more often?
SELECT 
    CASE 
        WHEN first_order_value < 50 THEN 'Under $50'
        WHEN first_order_value >= 50 AND first_order_value < 100 THEN '$50-100'
        WHEN first_order_value >= 100 AND first_order_value < 200 THEN '$100-200'
        WHEN first_order_value >= 200 AND first_order_value < 500 THEN '$200-500'
        ELSE '$500+'
    END AS order_value_bracket,
    
    COUNT(DISTINCT customer_unique_id) AS customers,
    COUNT(DISTINCT CASE WHEN is_repeat_customer = 1 THEN customer_unique_id END) AS repeat_customers,
    
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN is_repeat_customer = 1 THEN customer_unique_id END) 
          / COUNT(DISTINCT customer_unique_id), 2) AS repeat_rate_pct,
          
    ROUND(AVG(first_order_value), 2) AS avg_order_value

FROM (
    SELECT DISTINCT ON (customer_unique_id)
        customer_unique_id,
        is_repeat_customer,
        order_value AS first_order_value,
        order_purchase_timestamp
    FROM summary
    WHERE order_status = 'delivered'
    ORDER BY customer_unique_id, order_purchase_timestamp
) first_orders

GROUP BY 
    CASE 
        WHEN first_order_value < 50 THEN 'Under $50'
        WHEN first_order_value >= 50 AND first_order_value < 100 THEN '$50-100'
        WHEN first_order_value >= 100 AND first_order_value < 200 THEN '$100-200'
        WHEN first_order_value >= 200 AND first_order_value < 500 THEN '$200-500'
        ELSE '$500+'
    END

ORDER BY avg_order_value;
```
Here is the output of the query:

<img width="583" height="163" alt="Screenshot 2025-11-30 at 12 41 03 PM" src="https://github.com/user-attachments/assets/f064bffd-7b35-495f-a7a3-b23c008ebd20" />

**Price doesn't matter:** 
- Even customers spending $500+ only return 1.95% of the time.
- If anything, higher spenders are LESS likely to return.


```sql
-- Repeat rate by customer state (top 10 states by volume)
SELECT 
    customer_state,
    COUNT(DISTINCT customer_unique_id) as customers,
    COUNT(DISTINCT CASE WHEN is_repeat_customer = 1 THEN customer_unique_id END) as repeat_customers,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN is_repeat_customer = 1 THEN customer_unique_id END) / 
          COUNT(DISTINCT customer_unique_id), 2) as repeat_rate_pct
FROM (
    SELECT DISTINCT ON (customer_unique_id)
        customer_unique_id,
        customer_state,
        is_repeat_customer,
        order_purchase_timestamp
    FROM summary
    WHERE order_status = 'delivered'
    ORDER BY customer_unique_id, order_purchase_timestamp
) first_orders
GROUP BY customer_state
HAVING COUNT(DISTINCT customer_unique_id) >= 1000
ORDER BY customers DESC;
```
Here is the output of the query:

<img width="457" height="323" alt="Screenshot 2025-11-30 at 12 49 38 PM" src="https://github.com/user-attachments/assets/d649a87a-14e8-4ca2-8967-0aadeeba7a9b" />

**Geography doesn't matter:**
- São Paulo (39k customers, most developed market) has 3.30% repeat rate.
- Rural states like Ceará have 1.83%.
- The range is 1.83% to 3.53% - essentially flat.
