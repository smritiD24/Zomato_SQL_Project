-- ============================================================
-- Zomato SQL Project - Additional Problems (Q21 to Q25)
-- Written independently after completing the core 20 questions
-- These cover areas like retention, operational risk, and revenue concentration
-- ============================================================


-- Q.21 New vs Returning Customers per Month
-- -----------------------------------------------------------
-- For each month, find how many customers placed their very first order (New)
-- vs how many had ordered before (Returning).
-- This is useful for tracking whether growth is coming from acquisition or retention.

-- Approach:
-- First, find each customer's very first order date.
-- Then for every order, check if its date matches the customer's first order date.
-- If yes = New customer that month, else = Returning.

WITH first_order AS (
    SELECT 
        customer_id,
        MIN(order_date) AS first_order_date
    FROM orders
    GROUP BY customer_id
),
labeled_orders AS (
    SELECT 
        o.customer_id,
        TO_CHAR(o.order_date, 'YYYY-MM') AS order_month,
        CASE 
            WHEN o.order_date = f.first_order_date THEN 'New'
            ELSE 'Returning'
        END AS customer_type
    FROM orders AS o
    JOIN first_order AS f ON o.customer_id = f.customer_id
)
SELECT 
    order_month,
    customer_type,
    COUNT(DISTINCT customer_id) AS customer_count
FROM labeled_orders
GROUP BY 1, 2
ORDER BY 1, 2;

-- Note: A customer is counted as "New" only in the month of their first ever order.
-- In all subsequent months they appear as "Returning" regardless of how long the gap was.


-- Q.22 Restaurants with Consistently High Cancellation Rates
-- -----------------------------------------------------------
-- Find restaurants that had a cancellation rate above 20% for 3 or more consecutive months.
-- These are operational red flags -- could indicate kitchen issues, rider shortages, or bad listings.

-- Approach:
-- Calculate monthly cancellation rate per restaurant.
-- Use LAG() to look at previous 2 months.
-- Flag when all 3 consecutive months are above 20%.

WITH monthly_cancel AS (
    SELECT 
        o.restaurant_id,
        TO_CHAR(o.order_date, 'YYYY-MM') AS order_month,
        COUNT(o.order_id) AS total_orders,
        COUNT(CASE WHEN d.delivery_id IS NULL THEN 1 END) AS undelivered,
        ROUND(
            COUNT(CASE WHEN d.delivery_id IS NULL THEN 1 END)::numeric / COUNT(o.order_id)::numeric * 100
        , 2) AS cancel_rate
    FROM orders AS o
    LEFT JOIN deliveries AS d ON o.order_id = d.order_id
    GROUP BY 1, 2
),
with_lag AS (
    SELECT 
        restaurant_id,
        order_month,
        cancel_rate,
        LAG(cancel_rate, 1) OVER(PARTITION BY restaurant_id ORDER BY order_month) AS prev_1,
        LAG(cancel_rate, 2) OVER(PARTITION BY restaurant_id ORDER BY order_month) AS prev_2
    FROM monthly_cancel
)
SELECT 
    restaurant_id,
    order_month AS third_consecutive_month,
    prev_2 AS month_1_rate,
    prev_1 AS month_2_rate,
    cancel_rate AS month_3_rate
FROM with_lag
WHERE 
    cancel_rate > 20
    AND prev_1 > 20
    AND prev_2 > 20;

-- These restaurants have had 3 straight bad months -- worth investigating or penalizing.


-- Q.23 Rider Workload Distribution
-- -----------------------------------------------------------
-- Find how many deliveries each rider handled per month.
-- Also flag riders who handled more than 1.5x the monthly average (overloaded riders).
-- Helps operations team balance workload fairly.

WITH monthly_deliveries AS (
    SELECT 
        d.rider_id,
        TO_CHAR(o.order_date, 'YYYY-MM') AS delivery_month,
        COUNT(d.delivery_id) AS deliveries_done
    FROM deliveries AS d
    JOIN orders AS o ON d.order_id = o.order_id
    WHERE d.delivery_status = 'Delivered'
    GROUP BY 1, 2
),
monthly_avg AS (
    SELECT 
        delivery_month,
        AVG(deliveries_done) AS avg_deliveries
    FROM monthly_deliveries
    GROUP BY 1
)
SELECT 
    md.rider_id,
    md.delivery_month,
    md.deliveries_done,
    ROUND(ma.avg_deliveries, 1) AS monthly_avg,
    CASE 
        WHEN md.deliveries_done > ma.avg_deliveries * 1.5 THEN 'Overloaded'
        WHEN md.deliveries_done < ma.avg_deliveries * 0.5 THEN 'Underutilized'
        ELSE 'Normal'
    END AS workload_flag
FROM monthly_deliveries AS md
JOIN monthly_avg AS ma ON md.delivery_month = ma.delivery_month
ORDER BY md.delivery_month, md.deliveries_done DESC;


-- Q.24 Average Time Between First and Second Order per Customer
-- -----------------------------------------------------------
-- For every customer who has placed at least 2 orders, calculate the number of days
-- between their first and second order.
-- This is a key early retention metric -- a shorter gap means higher engagement early on.

-- Approach:
-- Use ROW_NUMBER() to number each customer's orders by date.
-- Pull rows where row_number is 1 or 2, then self-join and calculate the gap.

WITH order_sequence AS (
    SELECT 
        customer_id,
        order_date,
        ROW_NUMBER() OVER(PARTITION BY customer_id ORDER BY order_date) AS order_num
    FROM orders
),
first_two AS (
    SELECT 
        customer_id,
        MAX(CASE WHEN order_num = 1 THEN order_date END) AS first_order,
        MAX(CASE WHEN order_num = 2 THEN order_date END) AS second_order
    FROM order_sequence
    WHERE order_num <= 2
    GROUP BY customer_id
    HAVING COUNT(*) = 2  -- only customers who have at least 2 orders
)
SELECT 
    f.customer_id,
    c.customer_name,
    f.first_order,
    f.second_order,
    (f.second_order - f.first_order) AS days_to_return
FROM first_two AS f
JOIN customers AS c ON f.customer_id = c.customer_id
ORDER BY days_to_return ASC;

-- Customers with very short days_to_return are highly engaged early on.
-- Customers with 30+ day gaps might benefit from a timely promo push after their first order.


-- Q.25 Revenue Contribution by Top 10% of Customers
-- -----------------------------------------------------------
-- In most businesses, a small group of customers drives most of the revenue (Pareto principle).
-- This query checks how much of total revenue comes from the top 10% by spending.

-- Approach:
-- Rank customers by total spend using NTILE(10) -- splits into 10 equal buckets (deciles).
-- Bucket 1 = top 10% spenders. Calculate their revenue share vs total.

WITH customer_spend AS (
    SELECT 
        customer_id,
        SUM(total_amount) AS total_spent
    FROM orders
    GROUP BY customer_id
),
with_decile AS (
    SELECT 
        customer_id,
        total_spent,
        NTILE(10) OVER(ORDER BY total_spent DESC) AS spending_decile
    FROM customer_spend
)
SELECT 
    spending_decile,
    COUNT(customer_id) AS num_customers,
    SUM(total_spent) AS segment_revenue,
    ROUND(
        SUM(total_spent) / SUM(SUM(total_spent)) OVER() * 100
    , 2) AS revenue_share_pct
FROM with_decile
GROUP BY 1
ORDER BY 1;

-- Look at decile 1 (top 10%) -- if they contribute 40-60% of revenue, the Pareto principle holds.
-- This tells the business: protect and prioritize these customers above all else.


-- ============================================================
-- End of Additional Problems
-- ============================================================
