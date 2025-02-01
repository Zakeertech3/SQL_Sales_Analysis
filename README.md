# Sales_Data_Analysis

/* Business Requirement Document (BRD)

Project Title: Sales Data Analysis and Reporting

Objective:
The primary objective of this project is to analyze sales data to gain insights into sales performance, customer behavior, product trends, and regional performance. This analysis will help in making informed business decisions, improving sales strategies, and identifying areas for growth.

Scope:

Analyze historical sales data to identify trends.

Evaluate product performance across different regions.

Assess customer purchasing behavior.

Generate actionable insights for sales and marketing strategies.

Provide regular reports and dashboards for stakeholders.*/



CREATE DATABASE sales_db;
USE sales_db;


select * from sales_data;


-- 1. What is the month-over-month sales growth rate?


SELECT 
    YEAR_ID, 
    MONTH_ID, 
    SUM(SALES) AS total_sales,
    LAG(SUM(SALES)) OVER (PARTITION BY YEAR_ID ORDER BY MONTH_ID) AS previous_month_sales,
    ROUND(((SUM(SALES) - LAG(SUM(SALES)) OVER (PARTITION BY YEAR_ID ORDER BY MONTH_ID)) / LAG(SUM(SALES)) OVER (PARTITION BY YEAR_ID ORDER BY MONTH_ID)) * 100, 2) AS growth_rate
FROM sales_data
GROUP BY YEAR_ID, MONTH_ID
ORDER BY YEAR_ID, MONTH_ID;


-- 2. Which products show consistent sales growth over the years?

WITH product_sales AS (
    SELECT 
        PRODUCTCODE, 
        YEAR_ID, 
        SUM(SALES) AS yearly_sales
    FROM sales_data
    GROUP BY PRODUCTCODE, YEAR_ID
)
SELECT 
    PRODUCTCODE,
    CASE 
        WHEN MAX(yearly_sales) = MIN(yearly_sales) THEN 'Stable'
        WHEN MAX(yearly_sales) > MIN(yearly_sales) THEN 'Growing'
        ELSE 'Declining'
    END AS sales_trend
FROM product_sales
GROUP BY PRODUCTCODE
HAVING COUNT(DISTINCT YEAR_ID) > 1;


-- 3. What is the customer retention rate year over year?

WITH yearly_customers AS (
    SELECT 
        CUSTOMERNAME, 
        YEAR_ID
    FROM sales_data
    GROUP BY CUSTOMERNAME, YEAR_ID
),
retention AS (
    SELECT 
        a.YEAR_ID AS current_year, 
        COUNT(DISTINCT a.CUSTOMERNAME) AS total_customers,
        COUNT(DISTINCT b.CUSTOMERNAME) AS retained_customers
    FROM yearly_customers a
    LEFT JOIN yearly_customers b ON a.CUSTOMERNAME = b.CUSTOMERNAME AND a.YEAR_ID = b.YEAR_ID + 1
    GROUP BY a.YEAR_ID
)
SELECT 
    current_year, 
    total_customers, 
    retained_customers,
    ROUND((retained_customers / total_customers) * 100, 2) AS retention_rate
FROM retention
ORDER BY current_year;


-- 4. Identify customers who increased their order size by more than 50% year-over-year.

WITH customer_sales AS (
    SELECT 
        CUSTOMERNAME, 
        YEAR_ID, 
        SUM(SALES) AS total_sales
    FROM sales_data
    GROUP BY CUSTOMERNAME, YEAR_ID
),
customer_growth AS (
    SELECT 
        CUSTOMERNAME,
        YEAR_ID,
        total_sales,
        LAG(total_sales) OVER (PARTITION BY CUSTOMERNAME ORDER BY YEAR_ID) AS previous_year_sales
    FROM customer_sales
)
SELECT 
    CUSTOMERNAME, 
    YEAR_ID,
    total_sales,
    previous_year_sales,
    ROUND(((total_sales - previous_year_sales) / previous_year_sales) * 100, 2) AS growth_percentage
FROM customer_growth
WHERE previous_year_sales IS NOT NULL AND ((total_sales - previous_year_sales) / previous_year_sales) > 0.5
ORDER BY growth_percentage DESC;


-- 5. What is the revenue contribution of each product line relative to the total sales?

SELECT 
    PRODUCTLINE, 
    SUM(SALES) AS total_sales,
    ROUND((SUM(SALES) / (SELECT SUM(SALES) FROM sales_data)) * 100, 2) AS revenue_contribution_percentage
FROM sales_data
GROUP BY PRODUCTLINE
ORDER BY revenue_contribution_percentage DESC;


-- 6. Which customers have purchased from multiple product lines, and how many lines did they purchase from?

SELECT 
    CUSTOMERNAME,
    COUNT(DISTINCT PRODUCTLINE) AS distinct_product_lines
FROM sales_data
GROUP BY CUSTOMERNAME
HAVING COUNT(DISTINCT PRODUCTLINE) > 1
ORDER BY distinct_product_lines DESC;


-- 7. Identify the top 10% of customers contributing to 80% of the total revenue (Pareto Principle).

WITH customer_revenue AS (
    SELECT 
        CUSTOMERNAME, 
        SUM(SALES) AS total_revenue
    FROM sales_data
    GROUP BY CUSTOMERNAME
),
cumulative_revenue AS (
    SELECT 
        CUSTOMERNAME,
        total_revenue,
        SUM(total_revenue) OVER (ORDER BY total_revenue DESC) AS running_total,
        (SUM(total_revenue) OVER ()) * 0.8 AS eighty_percent_revenue
    FROM customer_revenue
)
SELECT 
    CUSTOMERNAME, 
    total_revenue
FROM cumulative_revenue
WHERE running_total <= eighty_percent_revenue
ORDER BY total_revenue DESC;


-- 8. Analyze the correlation between MSRP and actual sales price (PRICEEACH).

SELECT 
    PRODUCTCODE,
    AVG(MSRP) AS avg_msrp,
    AVG(PRICEEACH) AS avg_price_each,
    ROUND((AVG(PRICEEACH) / AVG(MSRP)) * 100, 2) AS discount_percentage
FROM sales_data
GROUP BY PRODUCTCODE
ORDER BY discount_percentage DESC;


-- 9. Which regions (by country) show the highest growth in sales year over year?

WITH country_sales AS (
    SELECT 
        COUNTRY, 
        YEAR_ID, 
        SUM(SALES) AS total_sales
    FROM sales_data
    GROUP BY COUNTRY, YEAR_ID
),
growth_analysis AS (
    SELECT 
        COUNTRY, 
        YEAR_ID,
        total_sales,
        LAG(total_sales) OVER (PARTITION BY COUNTRY ORDER BY YEAR_ID) AS previous_year_sales
    FROM country_sales
)
SELECT 
    COUNTRY, 
    YEAR_ID,
    total_sales,
    previous_year_sales,
    ROUND(((total_sales - previous_year_sales) / previous_year_sales) * 100, 2) AS growth_percentage
FROM growth_analysis
WHERE previous_year_sales IS NOT NULL
ORDER BY growth_percentage DESC;


-- 10. Which products have the highest variance in monthly sales?

WITH monthly_product_sales AS (
    SELECT 
        PRODUCTCODE, 
        YEAR_ID, 
        MONTH_ID, 
        SUM(SALES) AS monthly_sales
    FROM sales_data
    GROUP BY PRODUCTCODE, YEAR_ID, MONTH_ID
),
variance_calculation AS (
    SELECT 
        PRODUCTCODE, 
        VARIANCE(monthly_sales) AS sales_variance
    FROM monthly_product_sales
    GROUP BY PRODUCTCODE
)
SELECT 
    PRODUCTCODE, 
    sales_variance
FROM variance_calculation
ORDER BY sales_variance DESC
LIMIT 5;
