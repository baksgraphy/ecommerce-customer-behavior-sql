# 🛍️ E-Commerce Customer Behavior & Revenue Analysis

## 📌 Executive Summary
This project analyzes customer ordering habits, demographic revenue drivers, top-performing product categories, and monthly sales trends using **TheLook E-Commerce** dataset in **Google BigQuery (SQL)**. The goal is to translate raw transactional data into actionable business recommendations for inventory management and targeted marketing.

---

## 🛠️ Tools & Dataset
* **SQL Engine:** Google BigQuery
* **Dataset:** `bigquery-public-data.thelook_ecommerce` (`order_items`, `users`, `products`)

---

## 🔍 Key Business Questions & SQL Insights

### 1. Overall High-Level Metrics
* **Total Completed Orders:** 31,304
* **Total Revenue:** $2,707,394.77
* **Average Item Price:** $59.66

---

### 2. Customer Demographics (`JOIN` Analysis)
To identify which age demographics drive the most revenue, we joined order items with user demographic data:

```sql
SELECT 
  CASE 
    WHEN u.age < 20 THEN 'Under 20'
    WHEN u.age BETWEEN 20 AND 34 THEN '20-34 (Young Adults)'
    WHEN u.age BETWEEN 35 AND 54 THEN '35-54 (Adults)'
    ELSE '55+ (Seniors)'
  END AS age_group,
  COUNT(DISTINCT oi.order_id) AS total_orders,
  ROUND(SUM(oi.sale_price), 2) AS total_revenue,
  ROUND(AVG(oi.sale_price), 2) AS avg_item_price
FROM `bigquery-public-data.thelook_ecommerce.order_items` AS oi
JOIN `bigquery-public-data.thelook_ecommerce.users` AS u
  ON oi.user_id = u.id
WHERE oi.status = 'Complete'
GROUP BY age_group
ORDER BY total_revenue DESC;
```

### 3. Top Products per Category (`DENSE_RANK()`)
Using window functions, we ranked the top revenue-generating items within each product category:

```sql
WITH CategorySales AS (
  SELECT 
    p.category,
    p.name AS product_name,
    ROUND(SUM(oi.sale_price), 2) AS total_revenue,
    DENSE_RANK() OVER(PARTITION BY p.category ORDER BY SUM(oi.sale_price) DESC) AS rank_in_category
  FROM `bigquery-public-data.thelook_ecommerce.order_items` AS oi
  JOIN `bigquery-public-data.thelook_ecommerce.products` AS p
    ON oi.product_id = p.id
  WHERE oi.status = 'Complete'
  GROUP BY p.category, p.name
)
SELECT 
  category, 
  rank_in_category, 
  product_name, 
  total_revenue
FROM CategorySales
WHERE rank_in_category <= 3
ORDER BY category, rank_in_category;
```

* **Insight:** Key hero products include *Oakley Men's Racing Jacket Oval Sunglasses* (Accessories) and *adidas Women's adiFIT Slim Pants* (Activewear).

### 4. Monthly Revenue Trends & Seasonality
To track monthly business growth and seasonality, we truncated timestamps to monthly aggregates:

```sql
SELECT 
  TIMESTAMP_TRUNC(created_at, MONTH) AS sales_month,
  COUNT(DISTINCT order_id) AS total_orders,
  ROUND(SUM(sale_price), 2) AS monthly_revenue
FROM `bigquery-public-data.thelook_ecommerce.order_items`
WHERE status = 'Complete'
GROUP BY sales_month
ORDER BY sales_month DESC
LIMIT 12;
```

* **Insight:** August and September consistently show peak order volume ($180k+ revenue in Sept), indicating strong late-summer demand before Q4.

💡 Strategic Business Recommendations
Targeted Demographics: Focus promotional ad spend on 35+ demographics, as mature customers represent the largest volume of high-value purchases.

Stock Optimization: Increase inventory threshold for top-performing SKUs (e.g., Oakley Sunglasses) to avoid stockouts during demand spikes.

Logistics & Capacity: Scale warehouse operations and staffing ahead of August/September peak periods to handle seasonal volume surges efficiently.
