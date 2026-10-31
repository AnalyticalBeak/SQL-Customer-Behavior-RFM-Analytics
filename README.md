# 📊Customer Behaviour & RFM Analysis (PostgreSQL)

![Database](https://img.shields.io/badge/Database-PostgreSQL-blue?style=flat&logo=postgresql) 
![Tool](https://img.shields.io/badge/IDE-DBeaver-orange?style=flat&logo=dbeaver) 
![Status](https://img.shields.io/badge/Project-Completed-brightgreen)

## 📌Executive Summary
This project analyses customer buying behaviour, profitability by channel and Recency Frequency Monetary (RFM) Segmentation. This database uses complex queries, Window functions (DENSE_RANK) and CTE's to show insights such as Customer Retention, Ranking of High Value Customers and Revenue Performance.
---

## 🏗️ Relational Data Model (DDL)
There are 4 tables within this relational database:

- **`customers`**: Customer demographics and join date by province in South Africa.
- **`products`**: Products, Category, Price per unit.
- **`orders`**: Orders placed, date and store channel.
- **`order_items`**: Quantity of item purchased, discounts.

- 
---
## 🔍 Key Analytics & Queries
### 1) Executive Revenue & Profitability by Channel
Business Objective: To understand how much revenue is being made by store channels vs Online. Including Gross Revenue, Total Discounts and Net Revenue.
```sql
SELECT
o.store_channel,
COUNT(DISTINCT o.order_id) AS total_orders,
SUM(p.unit_price * oi.quantity) AS gross_revenue,
SUM(oi.discount_amount) AS total_discounts,
SUM((p.unit_price * oi.quantity) - oi.discount_amount) AS net_revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
GROUP BY o.store_channel
ORDER BY net_revenue DESC;
```

### 2) Customer Spend Ranking (Window Functions)
Business Objective: Rank customer spending behavior using DENSE_RANK() without gaps in ranking sequence.
```sql
WITH customer_spend AS (
    SELECT 
        c.customer_id,
        CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
        c.city,
        SUM((p.unit_price * oi.quantity) - oi.discount_amount) AS total_net_spend
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    GROUP BY c.customer_id, c.first_name, c.last_name, c.city
)
SELECT 
    customer_id,
    customer_name,
    city,
    total_net_spend,
    DENSE_RANK() OVER (ORDER BY total_net_spend DESC) AS spend_rank
FROM customer_spend;
```

### 3) RFM Customer Segmentation
Business Objective: Segment customers into actionable tiers (VIP / Champions, Loyal Customer, At Risk / Churning) using date arithmetic and conditional CASE statements.
```sql
WITH rfm_base AS (
    SELECT 
        c.customer_id,
        CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
        MAX(o.order_date) AS last_order_date,
        COUNT(DISTINCT o.order_id) AS order_frequency,
        SUM((p.unit_price * oi.quantity) - oi.discount_amount) AS monetary_value
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id
    LEFT JOIN order_items oi ON o.order_id = oi.order_id
    LEFT JOIN products p ON oi.product_id = p.product_id
    GROUP BY c.customer_id, c.first_name, c.last_name
)
SELECT 
    customer_id,
    customer_name,
    last_order_date,
    ('2025-07-01'::DATE - last_order_date) AS recency_days,
    order_frequency,
    COALESCE(monetary_value, 0.00) AS total_monetary_val,
    CASE 
        WHEN COALESCE(monetary_value, 0) > 5000 AND order_frequency >= 2 THEN 'VIP / Champions'
        WHEN order_frequency >= 2 THEN 'Loyal Customer'
        WHEN ('2025-07-01'::DATE - last_order_date) > 90 THEN 'At Risk / Churning'
        ELSE 'New / Potential'
    END AS rfm_segment
FROM rfm_base
ORDER BY total_monetary_val DESC;
```
