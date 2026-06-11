
# Zomato SQL Data Analysis Project
![ERD](https://github.com/smritiD24/Zomato_SQL_Project/blob/main/Zomato_logo.png)
## About This Project

I picked Zomato as my first serious SQL project because it's a company I actually use, and the data model maps really well to real-world business questions — riders, restaurants, customers, orders, deliveries all interacting with each other. The goal wasn't just to run queries, but to think about what the business would actually want to know.

I used **PostgreSQL** for this. The dataset is synthetic (AI-generated sample data) and is used purely for learning and portfolio purposes.

---

## What's Inside

```
zomato-sql-analysis/
│
├── README.md                        -- you're reading it
├── schema_setup.sql                 -- database + table creation
├── 20_Business_Problems_solution.sql   -- core 20 business problems
└── additional_problems.sql          -- 5 extra problems I added on my own
```

---

## Database Schema

Five tables, all connected through foreign keys:

![ERD]([https://github.com/najirh/zomato_sqlp3/blob/main/erd.png](https://github.com/smritiD24/Zomato_SQL_Project/blob/main/erd.png))

| Table | What it stores |
|---|---|
| `customers` | Customer ID, name, registration date |
| `restaurants` | Restaurant ID, name, city, opening hours |
| `riders` | Rider ID, name, sign-up date |
| `orders` | Order details — item, date, time, status, amount |
| `deliveries` | Delivery status, time, and which rider handled it |

```sql
CREATE DATABASE zomato_db;

DROP TABLE IF EXISTS deliveries;
DROP TABLE IF EXISTS Orders;
DROP TABLE IF EXISTS customers;
DROP TABLE IF EXISTS restaurants;
DROP TABLE IF EXISTS riders;

CREATE TABLE restaurants (
    restaurant_id SERIAL PRIMARY KEY,
    restaurant_name VARCHAR(100) NOT NULL,
    city VARCHAR(50),
    opening_hours VARCHAR(50)
);

CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    reg_date DATE
);

CREATE TABLE riders (
    rider_id SERIAL PRIMARY KEY,
    rider_name VARCHAR(100) NOT NULL,
    sign_up DATE
);

CREATE TABLE Orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT,
    restaurant_id INT,
    order_item VARCHAR(255),
    order_date DATE NOT NULL,
    order_time TIME NOT NULL,
    order_status VARCHAR(20) DEFAULT 'Pending',
    total_amount DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (restaurant_id) REFERENCES restaurants(restaurant_id)
);

CREATE TABLE deliveries (
    delivery_id SERIAL PRIMARY KEY,
    order_id INT,
    delivery_status VARCHAR(20) DEFAULT 'Pending',
    delivery_time TIME,
    rider_id INT,
    FOREIGN KEY (order_id) REFERENCES Orders(order_id),
    FOREIGN KEY (rider_id) REFERENCES riders(rider_id)
);
```

---

## Data Cleaning

Before jumping into analysis, I checked all tables for NULLs in critical columns and removed bad rows from `orders` (rows missing item, date, time, status, or amount are useless for any analysis):

```sql
-- Check for NULLs across tables
SELECT COUNT(*) FROM customers WHERE customer_name IS NULL OR reg_date IS NULL;
SELECT COUNT(*) FROM restaurants WHERE restaurant_name IS NULL OR city IS NULL OR opening_hours IS NULL;

-- Remove incomplete order records
DELETE FROM orders
WHERE 
    order_item IS NULL OR order_date IS NULL OR order_time IS NULL
    OR order_status IS NULL OR total_amount IS NULL;
```

> **Note:** I chose DELETE over UPDATE/COALESCE here because rows missing these fields can't contribute to any meaningful analysis — keeping them would skew aggregations.

---

## 20 Business Problems

### Q1. Top 5 dishes ordered by a specific customer in the last 1 year

The challenge here is combining a filter on customer name + date range, then ranking by count. I used `DENSE_RANK()` instead of `LIMIT 5` because ties should be included.

```sql
SELECT customer_name, dishes, total_orders
FROM (
    SELECT 
        c.customer_id,
        c.customer_name,
        o.order_item AS dishes,
        COUNT(*) AS total_orders,
        DENSE_RANK() OVER(ORDER BY COUNT(*) DESC) AS rank
    FROM orders AS o
    JOIN customers AS c ON c.customer_id = o.customer_id
    WHERE 
        o.order_date >= CURRENT_DATE - INTERVAL '1 Year'
        AND c.customer_name = 'Arjun Mehta'
    GROUP BY 1, 2, 3
) AS t1
WHERE rank <= 5;
```

---

### Q2. Peak Order Time Slots (2-hour intervals)

Two approaches for this one — I actually prefer Approach 2 (the math-based one) because it works for any hour without writing 12 WHEN conditions. Less code, same result.

**Approach 1 — Readable CASE statement:**
```sql
SELECT
    CASE
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 0 AND 1 THEN '00:00 - 02:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 2 AND 3 THEN '02:00 - 04:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 4 AND 5 THEN '04:00 - 06:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 6 AND 7 THEN '06:00 - 08:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 8 AND 9 THEN '08:00 - 10:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 10 AND 11 THEN '10:00 - 12:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 12 AND 13 THEN '12:00 - 14:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 14 AND 15 THEN '14:00 - 16:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 16 AND 17 THEN '16:00 - 18:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 18 AND 19 THEN '18:00 - 20:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 20 AND 21 THEN '20:00 - 22:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 22 AND 23 THEN '22:00 - 00:00'
    END AS time_slot,
    COUNT(order_id) AS order_count
FROM Orders
GROUP BY time_slot
ORDER BY order_count DESC;
```

**Approach 2 — Cleaner math-based bucketing (my preferred way):**
```sql
-- FLOOR(hour/2)*2 groups any hour into its 2-hour bucket start
-- e.g. hour 13 -> FLOOR(13/2)*2 = 12, so slot is 12:00-14:00
SELECT 
    FLOOR(EXTRACT(HOUR FROM order_time)/2)*2 AS start_time,
    FLOOR(EXTRACT(HOUR FROM order_time)/2)*2 + 2 AS end_time,
    COUNT(*) AS total_orders
FROM orders
GROUP BY 1, 2
ORDER BY 3 DESC;
```

---

### Q3. Average Order Value — Customers with 750+ orders

Simple GROUP BY + HAVING. The HAVING filters after grouping, unlike WHERE which filters rows before grouping — that distinction matters here.

```sql
SELECT 
    c.customer_name,
    AVG(o.total_amount) AS avg_order_value
FROM orders AS o
JOIN customers AS c ON c.customer_id = o.customer_id
GROUP BY 1
HAVING COUNT(order_id) > 750;
```

---

### Q4. High-Value Customers (spent over ₹1,00,000)

```sql
SELECT 
    c.customer_name,
    SUM(o.total_amount) AS total_spent
FROM orders AS o
JOIN customers AS c ON c.customer_id = o.customer_id
GROUP BY 1
HAVING SUM(o.total_amount) > 100000;
```

---

### Q5. Orders Placed but Never Delivered

Two ways — LEFT JOIN approach and NOT IN subquery. LEFT JOIN is generally faster on large datasets; NOT IN can behave unexpectedly if the subquery returns any NULLs.

**Approach 1 — LEFT JOIN (recommended):**
```sql
SELECT 
    r.restaurant_name,
    COUNT(o.order_id) AS undelivered_orders
FROM orders AS o
LEFT JOIN restaurants AS r ON r.restaurant_id = o.restaurant_id
LEFT JOIN deliveries AS d ON d.order_id = o.order_id
WHERE d.delivery_id IS NULL
GROUP BY 1
ORDER BY 2 DESC;
```

**Approach 2 — NOT IN subquery:**
```sql
SELECT 
    r.restaurant_name,
    COUNT(*) AS undelivered_orders
FROM orders AS o
LEFT JOIN restaurants AS r ON r.restaurant_id = o.restaurant_id
WHERE o.order_id NOT IN (SELECT order_id FROM deliveries)
GROUP BY 1
ORDER BY 2 DESC;
```

---

### Q6. Top Restaurant by Revenue in Each City (Last 1 Year)

Used a CTE + RANK() partitioned by city. The WHERE rank = 1 in the outer query gives us only the top earner per city.

```sql
WITH ranking_table AS (
    SELECT 
        r.city,
        r.restaurant_name,
        SUM(o.total_amount) AS revenue,
        RANK() OVER(PARTITION BY r.city ORDER BY SUM(o.total_amount) DESC) AS rank
    FROM orders AS o
    JOIN restaurants AS r ON r.restaurant_id = o.restaurant_id
    WHERE o.order_date >= CURRENT_DATE - INTERVAL '1 year'
    GROUP BY 1, 2
)
SELECT * FROM ranking_table WHERE rank = 1;
```

---

### Q7. Most Popular Dish in Each City

Same RANK() + PARTITION BY pattern as Q6, but applied to dishes instead of restaurants.

```sql
SELECT * FROM (
    SELECT 
        r.city,
        o.order_item AS dish,
        COUNT(order_id) AS total_orders,
        RANK() OVER(PARTITION BY r.city ORDER BY COUNT(order_id) DESC) AS rank
    FROM orders AS o
    JOIN restaurants AS r ON r.restaurant_id = o.restaurant_id
    GROUP BY 1, 2
) AS t1
WHERE rank = 1;
```

---

### Q8. Customer Churn — Active in 2023, Gone in 2024

A classic churn definition: ordered last year, not this year. Used NOT IN with a subquery.

```sql
-- Customers who ordered in 2023 but placed zero orders in 2024
SELECT DISTINCT customer_id 
FROM orders
WHERE 
    EXTRACT(YEAR FROM order_date) = 2023
    AND customer_id NOT IN (
        SELECT DISTINCT customer_id FROM orders
        WHERE EXTRACT(YEAR FROM order_date) = 2024
    );
```

> Business insight: These are re-engagement targets. A marketing team could run a "We miss you" campaign specifically for this segment.

---

### Q9. Cancellation Rate per Restaurant — 2023 vs 2024

Four CTEs to calculate and then compare. Cancellation here = order placed but no matching delivery record.

```sql
WITH cancel_ratio_23 AS (
    SELECT 
        o.restaurant_id,
        COUNT(o.order_id) AS total_orders,
        COUNT(CASE WHEN d.delivery_id IS NULL THEN 1 END) AS not_delivered
    FROM orders AS o
    LEFT JOIN deliveries AS d ON o.order_id = d.order_id
    WHERE EXTRACT(YEAR FROM o.order_date) = 2023
    GROUP BY o.restaurant_id
),
cancel_ratio_24 AS (
    SELECT 
        o.restaurant_id,
        COUNT(o.order_id) AS total_orders,
        COUNT(CASE WHEN d.delivery_id IS NULL THEN 1 END) AS not_delivered
    FROM orders AS o
    LEFT JOIN deliveries AS d ON o.order_id = d.order_id
    WHERE EXTRACT(YEAR FROM o.order_date) = 2024
    GROUP BY o.restaurant_id
),
last_year AS (
    SELECT restaurant_id, ROUND((not_delivered::numeric / total_orders::numeric) * 100, 2) AS cancel_ratio
    FROM cancel_ratio_23
),
current_year AS (
    SELECT restaurant_id, ROUND((not_delivered::numeric / total_orders::numeric) * 100, 2) AS cancel_ratio
    FROM cancel_ratio_24
)
SELECT 
    c.restaurant_id,
    c.cancel_ratio AS cancellation_2024,
    l.cancel_ratio AS cancellation_2023
FROM current_year AS c
JOIN last_year AS l ON c.restaurant_id = l.restaurant_id;
```

---

### Q10. Average Delivery Time per Rider

The tricky part: delivery can go past midnight, so if `delivery_time < order_time`, we add 1 day before calculating difference. EPOCH converts the interval to seconds, divide by 60 for minutes.

```sql
SELECT 
    o.order_id,
    o.order_time,
    d.delivery_time,
    d.rider_id,
    EXTRACT(EPOCH FROM (
        d.delivery_time - o.order_time + 
        CASE WHEN d.delivery_time < o.order_time THEN INTERVAL '1 day' ELSE INTERVAL '0 day' END
    ))/60 AS delivery_time_minutes
FROM orders AS o
JOIN deliveries AS d ON o.order_id = d.order_id
WHERE d.delivery_status = 'Delivered';
```

---

### Q11. Monthly Growth Rate per Restaurant

LAG() pulls the previous month's count so we can calculate month-over-month growth. The formula is: `(current - previous) / previous * 100`.

```sql
WITH monthly_orders AS (
    SELECT 
        o.restaurant_id,
        EXTRACT(YEAR FROM o.order_date) AS year,
        EXTRACT(MONTH FROM o.order_date) AS month,
        COUNT(o.order_id) AS current_month_orders,
        LAG(COUNT(o.order_id), 1) OVER(
            PARTITION BY o.restaurant_id 
            ORDER BY EXTRACT(YEAR FROM o.order_date), EXTRACT(MONTH FROM o.order_date)
        ) AS prev_month_orders
    FROM orders AS o
    JOIN deliveries AS d ON o.order_id = d.order_id
    WHERE d.delivery_status = 'Delivered'
    GROUP BY 1, 2, 3
)
SELECT
    restaurant_id,
    year,
    month,
    prev_month_orders,
    current_month_orders,
    ROUND(
        (current_month_orders::numeric - prev_month_orders::numeric) / prev_month_orders::numeric * 100
    , 2) AS growth_pct
FROM monthly_orders;
```

---

### Q12. Customer Segmentation — Gold vs Silver

Compare each customer's total spend to the overall AOV. A subquery inside the CASE handles the AOV calculation.

```sql
-- AOV across all orders: ~₹322 (from SELECT AVG(total_amount) FROM orders)
SELECT 
    cx_category,
    SUM(total_orders) AS total_orders,
    SUM(total_spent) AS total_revenue
FROM (
    SELECT 
        customer_id,
        SUM(total_amount) AS total_spent,
        COUNT(order_id) AS total_orders,
        CASE 
            WHEN SUM(total_amount) > (SELECT AVG(total_amount) FROM orders) THEN 'Gold'
            ELSE 'Silver'
        END AS cx_category
    FROM orders
    GROUP BY 1
) AS t1
GROUP BY 1;
```

---

### Q13. Rider Monthly Earnings (8% commission model)

```sql
SELECT 
    d.rider_id,
    TO_CHAR(o.order_date, 'MM-YY') AS month,
    SUM(total_amount) AS total_order_value,
    ROUND(SUM(total_amount) * 0.08, 2) AS rider_earnings
FROM orders AS o
JOIN deliveries AS d ON o.order_id = d.order_id
GROUP BY 1, 2
ORDER BY 1, 2;
```

---

### Q14. Rider Star Ratings Based on Delivery Speed

Delivery time is calculated the same way as Q10, then bucketed into star ratings.

```sql
SELECT 
    rider_id,
    stars,
    COUNT(*) AS total_ratings
FROM (
    SELECT
        rider_id,
        delivery_took_time,
        CASE 
            WHEN delivery_took_time < 15 THEN '5 star'
            WHEN delivery_took_time BETWEEN 15 AND 20 THEN '4 star'
            ELSE '3 star'
        END AS stars
    FROM (
        SELECT 
            o.order_id,
            o.order_time,
            d.delivery_time,
            EXTRACT(EPOCH FROM (
                d.delivery_time - o.order_time + 
                CASE WHEN d.delivery_time < o.order_time THEN INTERVAL '1 day' ELSE INTERVAL '0 day' END
            ))/60 AS delivery_took_time,
            d.rider_id
        FROM orders AS o
        JOIN deliveries AS d ON o.order_id = d.order_id
        WHERE delivery_status = 'Delivered'
    ) AS t1
) AS t2
GROUP BY 1, 2
ORDER BY 1, 3 DESC;
```

---

### Q15. Peak Day per Restaurant

TO_CHAR with 'Day' format returns the day name as a string (e.g. 'Monday   '). RANK() partitioned by restaurant finds the busiest day for each.

```sql
SELECT * FROM (
    SELECT 
        r.restaurant_name,
        TO_CHAR(o.order_date, 'Day') AS day_of_week,
        COUNT(o.order_id) AS total_orders,
        RANK() OVER(PARTITION BY r.restaurant_name ORDER BY COUNT(o.order_id) DESC) AS rank
    FROM orders AS o
    JOIN restaurants AS r ON o.restaurant_id = r.restaurant_id
    GROUP BY 1, 2
) AS t1
WHERE rank = 1;
```

---

### Q16. Customer Lifetime Value (CLV)

Total revenue per customer across their entire order history.

```sql
SELECT 
    o.customer_id,
    c.customer_name,
    SUM(o.total_amount) AS lifetime_value
FROM orders AS o
JOIN customers AS c ON o.customer_id = c.customer_id
GROUP BY 1, 2
ORDER BY 3 DESC;
```

---

### Q17. Month-over-Month Sales Trend

LAG() gives the previous month's total so we can compare side by side.

```sql
SELECT 
    EXTRACT(YEAR FROM order_date) AS year,
    EXTRACT(MONTH FROM order_date) AS month,
    SUM(total_amount) AS monthly_sales,
    LAG(SUM(total_amount), 1) OVER(
        ORDER BY EXTRACT(YEAR FROM order_date), EXTRACT(MONTH FROM order_date)
    ) AS prev_month_sales
FROM orders
GROUP BY 1, 2;
```

---

### Q18. Rider Efficiency — Best and Worst Average Delivery Times

Two CTEs: one to compute per-rider averages, then MIN/MAX across all riders.

```sql
WITH delivery_times AS (
    SELECT 
        d.rider_id,
        EXTRACT(EPOCH FROM (
            d.delivery_time - o.order_time + 
            CASE WHEN d.delivery_time < o.order_time THEN INTERVAL '1 day' ELSE INTERVAL '0 day' END
        ))/60 AS delivery_minutes
    FROM orders AS o
    JOIN deliveries AS d ON o.order_id = d.order_id
    WHERE d.delivery_status = 'Delivered'
),
rider_avg AS (
    SELECT rider_id, AVG(delivery_minutes) AS avg_time
    FROM delivery_times
    GROUP BY 1
)
SELECT 
    MIN(avg_time) AS fastest_rider_avg,
    MAX(avg_time) AS slowest_rider_avg
FROM rider_avg;
```

---

### Q19. Seasonal Item Popularity

I mapped months to seasons relevant to India (Summer = April–June, Monsoon = July–September, Winter = rest). The original used Spring/Summer which doesn't quite match Indian context — I adjusted mine in the additional questions.

```sql
SELECT 
    order_item,
    seasons,
    COUNT(order_id) AS total_orders
FROM (
    SELECT 
        *,
        CASE 
            WHEN EXTRACT(MONTH FROM order_date) BETWEEN 4 AND 6 THEN 'Spring'
            WHEN EXTRACT(MONTH FROM order_date) BETWEEN 7 AND 8 THEN 'Summer'
            ELSE 'Winter'
        END AS seasons
    FROM orders
) AS t1
GROUP BY 1, 2
ORDER BY 1, 3 DESC;
```

---

### Q20. City Revenue Ranking — 2023

```sql
SELECT 
    r.city,
    SUM(total_amount) AS total_revenue,
    RANK() OVER(ORDER BY SUM(total_amount) DESC) AS city_rank
FROM orders AS o
JOIN restaurants AS r ON o.restaurant_id = r.restaurant_id
WHERE EXTRACT(YEAR FROM o.order_date) = 2023
GROUP BY 1;
```

---

## Additional Problems (Self-Added)

These 5 questions I wrote myself after finishing the core 20 — areas I felt weren't covered and would be interesting from a business angle. Solutions are in `additional_problems.sql`.

### Q21. Identify New vs Returning Customers per Month
Find out each month, how many customers were ordering for the first time vs coming back. Useful for tracking acquisition vs retention.

### Q22. Restaurants with Consistently High Cancellation Rates
Find restaurants whose cancellation rate has been above 20% for 3 or more consecutive months. These are red flags operationally.

### Q23. Rider Workload Distribution
How evenly is the delivery work distributed? Find how many deliveries each rider handled per month, and flag riders who handled more than 1.5x the average (overloaded).

### Q24. Average Time Between a Customer's First and Second Order
A key retention metric — if the gap is too long, re-engagement campaigns are needed early. Calculate this for every customer who has placed at least 2 orders.

### Q25. Revenue Contribution by Top 10% of Customers
In most businesses, a small % of customers drive most revenue. Verify if that holds here by calculating the revenue share of the top 10% of customers by spend.

---

## Key Learnings from This Project

- **Window functions** (RANK, DENSE_RANK, LAG) are essential for comparative and ranking problems — I used them in at least 8 of the 20 questions.
- The **midnight edge case** in delivery time calculation (Q10, Q14, Q18) is a real gotcha — adding a conditional INTERVAL '1 day' handles it cleanly.
- **CTEs** make complex multi-step logic much more readable than nested subqueries. I started preferring CTEs from Q9 onward.
- **NOT IN vs LEFT JOIN** for "find rows that don't exist" — LEFT JOIN is safer when NULLs might be present in the subquery result.

---

## Tools Used

- PostgreSQL 15
- pgAdmin 4
- Dataset: Synthetic data generated for educational use (no real Zomato data)

---

*This project was inspired by ZeroAnalyst (Najir H.) on YouTube. The original 20 problem statements are from his course. All SQL solutions, comments, notes, and additional questions (Q21–Q25) are written by me.*
