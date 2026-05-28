# Data Catalog for Gold Layer

## Overview

The Gold Layer is the business-ready data representation used for analytics and reporting. It is modeled as a star schema with customer and product dimensions connected to a sales fact view.

## Object Summary

| Object | Type | Grain | Key Fields | Source |
| --- | --- | --- | --- | --- |
| `gold.dim_customers` | Dimension view | One row per customer | `customer_key`, `customer_id`, `customer_number` | CRM customer data enriched with ERP demographic and location data |
| `gold.dim_products` | Dimension view | One row per current product | `product_key`, `product_id`, `product_number` | CRM product data enriched with ERP category data |
| `gold.fact_sales` | Fact view | One row per sales order line | `order_number`, `product_key`, `customer_key` | CRM sales details joined to the Gold dimensions |
| `gold.report_customers` | Analytics view | One row per customer | `customer_key`, `customer_number` | Customer-level KPIs aggregated from Gold sales and customer data |
| `gold.report_products` | Analytics view | One row per product | `product_key`, `product_name` | Product-level KPIs aggregated from Gold sales and product data |

---

### 1. `gold.dim_customers`

- **Purpose:** Stores customer details enriched with demographic and geographic data for customer-level analytics.
- **Grain:** One row per unique customer.
- **Source Tables:** `silver.crm_cust_info`, `silver.erp_cust_az12`, `silver.erp_loc_a101`

| Column Name | Data Type | Description |
| --- | --- | --- |
| `customer_key` | BIGINT | Surrogate key generated with `ROW_NUMBER()` to uniquely identify each customer record in the dimension. |
| `customer_id` | INT | Numeric customer identifier from the CRM source system. |
| `customer_number` | NVARCHAR(50) | Customer business key from CRM, used for joins and tracking across sources. |
| `first_name` | NVARCHAR(50) | Customer first name after trimming and cleansing in the Silver Layer. |
| `last_name` | NVARCHAR(50) | Customer last name after trimming and cleansing in the Silver Layer. |
| `country` | NVARCHAR(50) | Customer country from ERP location data, standardized in the Silver Layer. |
| `marital_status` | NVARCHAR(50) | Standardized marital status, such as `Single`, `Married`, or `n/a`. |
| `gender` | NVARCHAR(50) | Standardized gender. CRM is the primary source; ERP gender is used as a fallback when CRM gender is `n/a`. |
| `birthdate` | DATE | Customer birth date from ERP demographic data. Future birth dates are handled as invalid in the Silver Layer. |
| `create_date` | DATE | Date when the customer record was created in the CRM source system. |

---

### 2. `gold.dim_products`

- **Purpose:** Provides the current product catalog with descriptive product, category, and maintenance attributes.
- **Grain:** One row per current product. Historical product records are excluded where `prd_end_dt` is not null.
- **Source Tables:** `silver.crm_prd_info`, `silver.erp_px_cat_g1v2`

| Column Name | Data Type | Description |
| --- | --- | --- |
| `product_key` | BIGINT | Surrogate key generated with `ROW_NUMBER()` to uniquely identify each current product record in the dimension. |
| `product_id` | INT | Numeric product identifier from the CRM source system. |
| `product_number` | NVARCHAR(50) | Product business key extracted from the CRM product key and used to join sales records to products. |
| `product_name` | NVARCHAR(50) | Descriptive product name from the CRM source system. |
| `category_id` | NVARCHAR(50) | Product category identifier derived from the CRM product key and standardized for ERP category joins. |
| `category` | NVARCHAR(50) | High-level product category from ERP category reference data. |
| `subcategory` | NVARCHAR(50) | More detailed product classification within the category. |
| `maintenance` | NVARCHAR(50) | Maintenance indicator or requirement from ERP category reference data. |
| `cost` | INT | Product cost from CRM. Missing costs are defaulted to `0` in the Silver Layer. |
| `product_line` | NVARCHAR(50) | Product line description, standardized from CRM product line codes such as `Mountain`, `Road`, `Touring`, or `n/a`. |
| `start_date` | DATE | Date when the current product record became active. |

---

### 3. `gold.fact_sales`

- **Purpose:** Stores cleansed sales transactions for reporting on revenue, quantities, customers, and products.
- **Grain:** One row per sales order line item.
- **Source Tables:** `silver.crm_sales_details`, `gold.dim_products`, `gold.dim_customers`

| Column Name | Data Type | Description |
| --- | --- | --- |
| `order_number` | NVARCHAR(50) | Unique sales order number from the CRM sales source. |
| `product_key` | BIGINT | Surrogate key linking each sales line to `gold.dim_products`. |
| `customer_key` | BIGINT | Surrogate key linking each sales line to `gold.dim_customers`. |
| `order_date` | DATE | Date when the order was placed. Invalid or malformed source dates are converted to null in the Silver Layer. |
| `shipping_date` | DATE | Date when the order was shipped. Invalid or malformed source dates are converted to null in the Silver Layer. |
| `due_date` | DATE | Date when the order was due. Invalid or malformed source dates are converted to null in the Silver Layer. |
| `sales_amount` | INT | Total sales amount for the order line. Missing or incorrect values are recalculated as `quantity * ABS(price)` in the Silver Layer. |
| `quantity` | INT | Number of units sold for the order line. |
| `price` | INT | Unit price for the order line. Missing or invalid values are derived from `sales_amount / quantity` where possible. |

---

### 4. `gold.report_customers`

- **Purpose:** Provides a BI-ready customer report with customer profile, segmentation, recency, and spending metrics.
- **Grain:** One row per customer.
- **Source Tables:** `gold.fact_sales`, `gold.dim_customers`

| Column Name | Description |
| --- | --- |
| `customer_key` | Surrogate key of the customer. |
| `customer_number` | Customer business key from the CRM source. |
| `customer_name` | Full customer name created from first and last name. |
| `age` | Customer age calculated from birthdate. |
| `age_group` | Age segment such as `20-29`, `30-39`, `40-49`, or `50 and above`. |
| `customer_segment` | Customer value segment: `VIP`, `Regular`, or `New`. |
| `last_order_date` | Most recent order date for the customer. |
| `recency_in_months` | Number of months since the customer's latest order. |
| `total_orders` | Total distinct orders placed by the customer. |
| `total_sales` | Total sales amount generated by the customer. |
| `total_quantity` | Total quantity purchased by the customer. |
| `total_products` | Number of distinct products purchased by the customer. |
| `lifespan` | Number of months between the customer's first and latest order. |
| `avg_order_value` | Average sales amount per order. |
| `average_monthly_spend` | Average monthly customer spend across the customer lifespan. |

---

### 5. `gold.report_products`

- **Purpose:** Provides a BI-ready product report with product attributes, performance segmentation, recency, and revenue metrics.
- **Grain:** One row per product.
- **Source Tables:** `gold.fact_sales`, `gold.dim_products`

| Column Name | Description |
| --- | --- |
| `product_key` | Surrogate key of the product. |
| `product_name` | Product name from the product dimension. |
| `category` | Product category. |
| `subcategory` | Product subcategory. |
| `cost` | Product cost. |
| `last_sale_date` | Most recent sale date for the product. |
| `recency_in_months` | Number of months since the product's latest sale. |
| `product_segment` | Product performance segment: `High-Performer`, `Mid-Range`, or `Low-Performer`. |
| `lifespan` | Number of months between the product's first and latest sale. |
| `total_orders` | Total distinct orders containing the product. |
| `total_sales` | Total revenue generated by the product. |
| `total_quantity` | Total units sold. |
| `total_customers` | Number of distinct customers who purchased the product. |
| `avg_selling_price` | Average selling price per unit. |
| `avg_order_revenue` | Average revenue per order for the product. |
| `avg_monthly_revenue` | Average monthly revenue across the product lifespan. |

## Relationships

| Relationship | Type | Description |
| --- | --- | --- |
| `gold.fact_sales.customer_key` -> `gold.dim_customers.customer_key` | Many-to-one | Connects each sales line to the customer who placed the order. |
| `gold.fact_sales.product_key` -> `gold.dim_products.product_key` | Many-to-one | Connects each sales line to the current product dimension record. |

## Quality Expectations

| Object | Check | Expected Result |
| --- | --- | --- |
| `gold.dim_customers` | Duplicate `customer_key` values | No duplicates |
| `gold.dim_products` | Duplicate `product_key` values | No duplicates |
| `gold.fact_sales` | Sales records without matching customer or product dimension records | No orphaned fact rows |
