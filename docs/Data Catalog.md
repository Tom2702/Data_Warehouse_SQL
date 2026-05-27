# Data Catalog for Gold Layer

## Overview

The Gold Layer is the business-ready data representation used for analytics and reporting. It is modeled as a star schema with customer and product dimensions connected to a sales fact view.

## Object Summary

| Object | Type | Grain | Key Fields | Source |
| --- | --- | --- | --- | --- |
| `gold.dim_customers` | Dimension view | One row per customer | `customer_key`, `customer_id`, `customer_number` | CRM customer data enriched with ERP demographic and location data |
| `gold.dim_products` | Dimension view | One row per current product | `product_key`, `product_id`, `product_number` | CRM product data enriched with ERP category data |
| `gold.fact_sales` | Fact view | One row per sales order line | `order_number`, `product_key`, `customer_key` | CRM sales details joined to the Gold dimensions |

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

## Relationships

| Relationship | Type | Description |
| --- | --- | --- |
| `gold.fact_sales.customer_key` -> `gold.dim_customers.customer_key` | Many-to-one | Connects each sales line to the customer who placed the order. |
| `gold.fact_sales.product_key` -> `gold.dim_products.product_key` | Many-to-one | Connects each sales line to the current product dimension record. |


