-- Analyze sales perfomance over time
SELECT 
	YEAR(order_date) AS order_year,
	MONTH(order_date) AS order_month,
	SUM(sales_amount) AS total_sales,
	COUNT(DISTINCT customer_key) AS total_customers,
	SUM(quantity) AS total_quantity
FROM gold.fact_sales
WHERE order_date IS NOT NULL
GROUP BY YEAR(order_date), MONTH(order_date)
ORDER BY YEAR(order_date), MONTH(order_date)

-- The total sales per month and the running total and moving average price of sales over time
SELECT 
	order_date,
	total_sales,
	SUM(total_sales) OVER (PARTITION BY order_date ORDER BY order_date) AS running_total_sales,
	SUM(avg_price) OVER (PARTITION BY order_date ORDER BY order_date) AS moving_average_price
FROM
(
SELECT 
	DATETRUNC(MONTH, order_date) AS order_date,
	SUM(sales_amount) AS total_sales,
	AVG(price) AS avg_price
FROM gold.fact_sales
WHERE order_date IS NOT NULL 
GROUP BY DATETRUNC(MONTH, order_date)
) t

-- Analyze the yearly perfomance of products  
WITH yearly_product_sales AS (
SELECT	
	YEAR(f.order_date) AS order_year,
	p.product_name,
	SUM(f.sales_amount) AS current_sales
FROM gold.fact_sales f
LEFT JOIN gold.dim_products p
ON f.product_key = p.product_key
WHERE order_date IS NOT NULL
GROUP BY YEAR(f.order_date), p.product_name
)

SELECT 
	order_year,
	product_name,
	current_sales,
	AVG(current_sales) OVER (PARTITION BY product_name) avg_sales,
	current_sales - AVG(current_sales) OVER (PARTITION BY product_name) AS diff_avg,
	CASE WHEN current_sales - AVG(current_sales) OVER (PARTITION BY product_name) > 0 THEN 'Above Avg'
		 WHEN current_sales - AVG(current_sales) OVER (PARTITION BY product_name) < 0 THEN 'Below Avg'
		 ELSE 'Avg'
	END AS avg_change,
	-- Year-over-year Analysis
	LAG(current_sales) OVER (PARTITION BY product_name ORDER BY order_year) AS py_sales,
	current_sales - LAG(current_sales) OVER (PARTITION BY product_name ORDER BY order_year) AS diff_py,
	CASE WHEN current_sales - LAG(current_sales) OVER (PARTITION BY product_name ORDER BY order_year) > 0 THEN 'Increase'
		 WHEN current_sales - LAG(current_sales) OVER (PARTITION BY product_name ORDER BY order_year) < 0 THEN 'Decrease'
		 ELSE 'No Change'
	END AS py_change
FROM yearly_product_sales
ORDER BY product_name, order_year

-- Which categories contribute the most to overall sales?
WITH category_sales AS (
SELECT 
	category,
	SUM(sales_amount) total_sales
FROM gold.fact_sales f
LEFT JOIN gold.dim_products p
ON p.product_key = f.product_key
GROUP BY category
)

SELECT 
	category,
	total_sales,
	SUM(total_sales) OVER () overall_sales, 
	CONCAT(ROUND((CAST(total_sales AS FLOAT) / SUM(total_sales) OVER ())*100, 2), '%') AS percentage_of_total
FROM category_sales
ORDER BY total_sales DESC

/* Segment products into cost ranges and 
	count how many products fall into each segment */
WITH product_segments AS (
SELECT 
	product_key,
	product_name,
	cost,
	CASE WHEN cost < 100 THEN 'Below 100'
		 WHEN cost BETWEEN 100 AND 50 THEN '100-500'
		 WHEN cost BETWEEN 500 AND 1000 THEN '500-1000'
		 ELSE 'Above 1000'
	END cost_range
FROM gold.dim_products 
)

SELECT 
	cost_range,
	COUNT(product_key) AS total_products
FROM product_segments
GROUP BY cost_range
ORDER BY total_products DESC

/* Group customers into three segments based on their spending behavior:
	- VIP: at least 12 months of history and spending more than 5000.
	- Regular: at least 12 months of history but spending 5000 or less.
	- New: a lifespan less than 12 months.
*/
WITH customer_spending AS (
SELECT 
	c.customer_key,
	SUM(f.sales_amount) AS total_spending,
	DATEDIFF(MONTH, MIN(order_date), MAX(order_date)) AS lifespan
FROM gold.fact_sales f
LEFT JOIN gold.dim_customers c
ON f.customer_key = c.customer_key 
GROUP BY c.customer_key
)
SELECT 
	customer_segment,
	COUNT(customer_key) AS total_customer
FROM (
	SELECT
		customer_key,
		CASE WHEN lifespan >= 12 AND total_spending > 5000 THEN 'VIP'
			 WHEN lifespan >= 12 AND total_spending <= 5000 THEN 'Regular'
			 ELSE 'New'
		END customer_segment
	FROM customer_spending
)t
GROUP BY customer_segment
ORDER BY total_customer DESC

-- Customer Report
CREATE VIEW gold.report_customers AS
-- Base Query: Retrieves core columns from table
WITH base_query AS (
SELECT 
	f.order_number,
	f.product_key,
	f.order_date,
	f.sales_amount,
	f.quantity,
	c.customer_key,
	c.customer_number,
	CONCAT(c.first_name,' ', c.last_name) AS customer_name,
	DATEDIFF(YEAR, c.birthdate, GETDATE()) age
FROM gold.fact_sales f
LEFT JOIN gold.dim_customers c
ON c.customer_key = f.customer_key 
WHERE order_date IS NOT NULL
),
-- Customer Aggregations: Summarizes key metrics at the customer level
customer_aggregation AS (
	SELECT 
		customer_key,
		customer_number,
		customer_name,
		age,
		COUNT(DISTINCT order_number) AS total_orders,
		SUM(sales_amount) AS total_sales,
		SUM(quantity) AS total_quantity,
		COUNT(DISTINCT product_key) AS total_products,
		MAX(order_date) AS last_order_date,
		DATEDIFF(MONTH, MIN(order_date), MAX(order_date)) AS lifespan
	FROM base_query
	GROUP BY 
		customer_key,
		customer_number,
		customer_name,
		age
)
-- Final Query: Combines all customer results into one output
SELECT 
	customer_key,
	customer_number,
	customer_name,
	age,
	CASE WHEN age < 20 THEN 'Under 20'
		 WHEN age BETWEEN 20 AND 29 THEN '20-29'
		 WHEN age BETWEEN 30 AND 39 THEN '30-39'
		 WHEN age BETWEEN 40 AND 49 THEN '40-49'
		 ELSE '50 and above'
	END AS age_group,
	CASE WHEN lifespan >= 12 AND total_sales > 5000 THEN 'VIP'
		 WHEN lifespan >= 12 AND total_sales <= 5000 THEN 'Regular'
		 ELSE 'New'
	END AS customer_segment,
	last_order_date,
	DATEDIFF(MONTH, last_order_date, GETDATE()) AS recency_in_months,
	total_orders,
	total_sales,
	total_quantity,
	total_products,
	lifespan,
	CASE WHEN total_orders = 0 THEN 0 -- Average Order Value
		ELSE total_sales / total_orders 
	END AS avg_order_value,
	CASE WHEN lifespan = 0 THEN 0 -- Average Monthly Spend
		 ELSE total_sales / lifespan
	END AS average_monthly_spend
FROM customer_aggregation

-- Product Report
CREATE VIEW gold.report_products AS 
-- Retrieves core columns from fact_sales and dim_products
WITH base_query AS (
SELECT 
	f.order_number,
	f.customer_key,
	f.product_key,
	f.order_date,
	f.sales_amount,
	f.quantity,
	f.price,
	p.product_name,
	p.category,
	p.subcategory,
	p.cost
FROM gold.fact_sales f
LEFT JOIN gold.dim_products p
ON f.product_key = p.product_key
WHERE order_date IS NOT NULL -- Consider only valid sales dates
),
-- Product Aggregations: Summarizes key metrics at the product level
product_aggregation AS (
	SELECT 
		product_key,
		product_name,
		category,
		subcategory,
		cost,
		DATEDIFF(MONTH, MIN(order_date), MAX(order_date)) AS lifespan,
		MAX(order_date) AS last_sale_date,
		COUNT(DISTINCT order_number) AS total_orders,
		COUNT(DISTINCT customer_key) AS total_customers,
		SUM(sales_amount) AS total_sales,
		SUM(quantity) AS total_quantity,
		ROUND(AVG(CAST(sales_amount AS FLOAT) / NULLIF(quantity, 0)), 1) AS avg_selling_price
	FROM base_query
	GROUP BY 
		product_key,
		product_name,
		category,
		subcategory,
		cost
)
-- Final Query: Combines all customer results into one output
SELECT 
	product_key,
	product_name,
	category,
	subcategory,
	cost,
	last_sale_date,
	DATEDIFF(MONTH, last_sale_date, GETDATE()) AS recency_in_months,
	CASE
		WHEN total_sales > 50000 THEN 'High-Performer'
		WHEN total_sales >= 10000 THEN 'Mid-Range'
		ELSE 'Low-Performer'
	END AS product_segment,
	lifespan,
	total_orders,
	total_sales, 
	total_quantity, 
	total_customers, 
	avg_selling_price,
	CASE -- Average Order Revenue
		WHEN total_orders = 0 THEN 0
		ELSE total_sales / total_orders
	END AS avg_order_revenue,
	CASE WHEN lifespan = 0 THEN 0 -- Average Monthly Revenue
		 ELSE total_sales / lifespan
	END AS avg_monthly_revenue
FROM product_aggregation
