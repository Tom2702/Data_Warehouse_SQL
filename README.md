# Data Warehouse | SQL

![SQL Server](https://img.shields.io/badge/SQL%20Server-Data%20Warehouse-CC2927?style=for-the-badge&logo=microsoftsqlserver&logoColor=white)
![ETL](https://img.shields.io/badge/ETL-Bronze%20%7C%20Silver%20%7C%20Gold-2F6FED?style=for-the-badge)
![Data Modeling](https://img.shields.io/badge/Data%20Modeling-Star%20Schema-F2C94C?style=for-the-badge)

## Overview

This project demonstrates the design and implementation of a modern SQL Server data warehouse using the Medallion Architecture: Bronze, Silver, and Gold layers.

The warehouse consolidates CRM and ERP source data, applies cleansing and standardization rules, integrates related business entities, and exposes a business-ready Star Schema for reporting, analytics, and decision-making.

## Objective

Develop a modern data warehouse using SQL Server to consolidate sales data, enabling analytical reporting and informed decision-making.

The final analytical model helps business users and analysts answer questions about:

- Customer behavior and demographics
- Product performance and product categories
- Sales trends, quantities, pricing, and revenue
- Relationships between customers, products, and sales transactions

## High Level Architecture

The solution follows a layered warehouse architecture in SQL Server. Raw CSV data is loaded into the Bronze layer, transformed and standardized in the Silver layer, then modeled into analytical views in the Gold layer.

<p align="center">
 <img width="1172" height="629" alt="image" src="https://github.com/user-attachments/assets/8cdf18b5-f41f-4e9a-8243-9543c8e4d8cd" />
</p>

## Data Flow

The data flow begins with source files from CRM and ERP systems. Each source table is loaded into Bronze, cleaned into Silver, and integrated into Gold dimensions and facts.

<p align="center">
  <img width="961" height="553" alt="image" src="https://github.com/user-attachments/assets/04ed3f17-8320-4b68-8691-41bbd5188c52" />
</p>

## Data Integration Model

The integration model shows how CRM and ERP entities relate to one another before they are combined into the final analytical model.

<p align="center">
  <img width="698" height="434" alt="image" src="https://github.com/user-attachments/assets/a1c22f4b-983d-4c71-a6e1-6e85a751cdc3" />
</p>

## Data Mart Model

The Gold layer is modeled as a Star Schema with two dimensions and one fact view:

- `gold.dim_customers`
- `gold.dim_products`
- `gold.fact_sales`

<p align="center">
  <img width="761" height="424" alt="image" src="https://github.com/user-attachments/assets/7c0865b6-3ed9-4612-9bbd-4f9cede04d4c" />
</p>

## Repository Structure

```text
Data_Warehouse_SQL/
|-- docs/
|   |-- data_catalog.md
|   |-- data_architecture.png
|   |-- data_flow_diagram.png
|   |-- data_integration.png
|   `-- data_model.png
|-- scripts/
|   |-- analytics.sql
|   |-- bronze/
|   |   |-- ddl_bronze.sql
|   |   `-- proc_load_bronze.sql
|   |-- silver/
|   |   |-- ddl_silver.sql
|   |   `-- proc_doc_silver.sql
|   |-- gold/
|   |   `-- ddl_gold.sql
|   `-- init_database.sql
|-- tests/
|   |-- quality_checks_silver.sql
|   `-- quality_checks_gold.sql
`-- README.md
```

## Implementation Workflow

### 1. Bronze Layer: Raw Data Ingestion

The Bronze layer stores source data as-is from CRM and ERP files. This layer is designed to preserve the original structure of the source systems and provide a traceable foundation for downstream transformations.

**Object type:** Tables  
**Load pattern:** Full load with truncate and insert  
**Transformation level:** None, raw ingestion only

**Main activities**

- Create raw tables in the `bronze` schema.
- Load CRM source files into Bronze tables.
- Load ERP source files into Bronze tables.
- Preserve original source column names and data formats.
- Use Bronze as the audit-friendly landing layer for the warehouse.

**Bronze tables**

| Table | Description |
| --- | --- |
| `bronze.crm_cust_info` | Raw CRM customer master data. |
| `bronze.crm_prd_info` | Raw CRM product information. |
| `bronze.crm_sales_details` | Raw CRM sales transaction details. |
| `bronze.erp_cust_az12` | Raw ERP customer demographic data. |
| `bronze.erp_loc_a101` | Raw ERP customer location data. |
| `bronze.erp_px_cat_g1v2` | Raw ERP product category data. |

**Scripts**

- `scripts/bronze/ddl_bronze.sql`
- `scripts/bronze/proc_load_bronze.sql`

**Result**

<img width="567" height="681" alt="image" src="https://github.com/user-attachments/assets/795c9f0a-6bc5-4c01-acfa-f9354de88d87" />

<!--
<p align="center">
  <img src="docs/images/results/bronze_result.png" alt="Bronze Layer Result" width="900">
</p>
-->

### 2. Silver Layer: Cleansing and Standardization

The Silver layer transforms raw Bronze data into clean, standardized, and analysis-ready tables. This layer resolves data quality issues and prepares entities for integration.

**Object type:** Tables  
**Load pattern:** Full load with truncate and insert  
**Transformation level:** Data cleaning, standardization, normalization, and derived columns

**Main activities**

- Remove duplicate customer records and keep the latest valid record.
- Trim customer names and normalize text fields.
- Standardize gender and marital status values.
- Convert invalid or malformed date values into clean `DATE` fields.
- Split product keys into category and product identifiers.
- Standardize product line values into readable business labels.
- Derive product end dates using window functions.
- Recalculate invalid sales amounts and prices where needed.
- Normalize ERP customer IDs and country values.
- Add `dwh_create_date` metadata to track warehouse load time.

**Silver tables**

| Table | Description |
| --- | --- |
| `silver.crm_cust_info` | Cleaned CRM customer data. |
| `silver.crm_prd_info` | Cleaned CRM product data with derived category and product keys. |
| `silver.crm_sales_details` | Cleaned CRM sales details with valid dates, sales, quantity, and price. |
| `silver.erp_cust_az12` | Cleaned ERP customer demographic data. |
| `silver.erp_loc_a101` | Cleaned ERP customer location data. |
| `silver.erp_px_cat_g1v2` | Cleaned ERP product category reference data. |

**Scripts**

- `scripts/silver/ddl_silver.sql`
- `scripts/silver/proc_doc_silver.sql`
- `tests/quality_checks_silver.sql`

**Result**

<img width="551" height="675" alt="image" src="https://github.com/user-attachments/assets/52ca72a9-0d8a-46aa-88a2-8345f871c5ac" />

<!--
<p align="center">
  <img src="docs/images/results/silver_result.png" alt="Silver Layer Result" width="900">
</p>
-->

### 3. Gold Layer: Business-Ready Star Schema

The Gold layer exposes business-ready views for reporting and analytics. It integrates cleaned CRM and ERP data into dimensions and a fact view.

**Object type:** Views  
**Load pattern:** No physical load; views are generated from Silver and Gold objects  
**Transformation level:** Data integration, business logic, and dimensional modeling

**Main activities**

- Create customer dimension with demographic and location enrichment.
- Create product dimension with category and maintenance attributes.
- Keep only current products in the product dimension.
- Create sales fact view connected to customer and product dimensions.
- Generate surrogate keys using `ROW_NUMBER()`.
- Validate uniqueness and relationship integrity.

**Gold views**

| View | Type | Description |
| --- | --- | --- |
| `gold.dim_customers` | Dimension | Customer profile enriched with ERP birthdate, gender fallback, and country. |
| `gold.dim_products` | Dimension | Current product catalog enriched with ERP category details. |
| `gold.fact_sales` | Fact | Sales transactions linked to customer and product dimensions. |

**Scripts**

- `scripts/gold/ddl_gold.sql`
- `tests/quality_checks_gold.sql`

**Result**

<p align="center">
  <img width="1463" height="826" alt="gold.dim_customers result" src="https://github.com/user-attachments/assets/488b7aec-a2c0-4e94-a2eb-9ac42907e107">
  <br>
  <strong>gold.dim_customers</strong>
</p>

<p align="center">
  <img width="1472" height="822" alt="gold.dim_products result" src="https://github.com/user-attachments/assets/94ca86dd-b41d-4740-8493-8faacebdfcb3">
  <br>
  <strong>gold.dim_products</strong>
</p>

<p align="center">
  <img width="1471" height="828" alt="gold.fact_sales result" src="https://github.com/user-attachments/assets/08ebe47b-ac0c-4088-aa8d-514c7b18ab30">
  <br>
  <strong>gold.fact_sales</strong>
</p>

## Analytics

The project includes an analytics layer built on top of the Gold views. 

| No. | Analysis Area | Business Question | Main Output |
| --- | --- | --- | --- |
| 01 | Sales Performance Over Time | How do sales, customer count, and quantity sold change by month? | Monthly sales performance |
| 02 | Monthly Sales Trend, Running Total, and Moving Average Price | What is the monthly sales trend, cumulative sales growth, and average price movement over time? | Running sales trend and moving average price |
| 03 | Yearly Product Performance | Which products perform above or below their average sales, and how do they change year over year? | Product-level yearly performance |
| 04 | Category Contribution to Overall Sales | Which product categories contribute the most to total revenue? | Category sales share |
| 05 | Product Segmentation by Cost | How many products fall into each cost range? | Product cost segments |
| 06 | Customer Segmentation by Spending Behavior | How many customers are VIP, Regular, or New based on lifespan and spending? | Customer behavior segments |
| 07 | Customer Report View | What customer-level KPIs are needed for BI and customer analytics? | `gold.report_customers` |
| 08 | Product Report View | What product-level KPIs are needed for BI and product performance analysis? | `gold.report_products` |

**Key Findings**

- Bikes contribute 96.46% of total sales, making them the dominant revenue driver.
- New customers are the largest customer segment with 14,631 customers.
- VIP customers represent a smaller but valuable customer group with longer history and higher spending.
- Product sales are concentrated in the Bikes category, especially Mountain Bikes and Road Bikes.
- The analytics views provide reusable datasets for dashboards, ad-hoc analysis, and business reporting.

<details>
<summary><strong>01. Sales Performance Over Time</strong></summary>

**Business question:** How do sales, customer count, and quantity sold change by month?

**SQL query:**

<p align="center">
  <img width="571" height="343" alt="image" src="https://github.com/user-attachments/assets/939b4d09-d4e5-42d5-ae59-a3aa53ba602a" />
</p>

**Result:**

<p align="center">
  <img width="468" height="344" alt="image" src="https://github.com/user-attachments/assets/63072565-bd1f-4b27-b7b2-98e0949cf2f7" />
</p>

</details>
 
<details>
<summary><strong>02. Monthly Sales Trend, Running Total, and Moving Average Price</strong></summary>

**Business question:** What is the monthly sales trend, cumulative sales growth, and average price movement over time?

**SQL query:**

<p align="center">
  <img width="946" height="475" alt="image" src="https://github.com/user-attachments/assets/342a2e84-1d60-4fd5-a22a-abe9f043ca0e" />
</p>

**Result:**

<p align="center">
 <img width="458" height="349" alt="image" src="https://github.com/user-attachments/assets/e539639b-f6d2-4230-89f7-0fc3ad66f675" />
</p>

</details>

<details>
<summary><strong>03. Yearly Product Performance</strong></summary>

**Business question:** Which products perform above or below their average sales, and how do they change year over year?

**SQL query:**

<p align="center">
  <img width="1227" height="963" alt="image" src="https://github.com/user-attachments/assets/bf988d39-b444-4ae9-8875-1cf4cdaf111a" />
</p>

**Result:**

<p align="center">
  <img width="755" height="349" alt="image" src="https://github.com/user-attachments/assets/7d435775-cecc-4c2f-960e-fe1aa8c0153a" />
</p>

</details>

<details>
<summary><strong>04. Category Contribution to Overall Sales</strong></summary>

**Business question:** Which product categories contribute the most to total revenue?

**SQL query:**

<p align="center">
  <img width="1093" height="545" alt="image" src="https://github.com/user-attachments/assets/26815e5b-6907-4adb-8e96-3fa554bc28f9" />
</p>

**Result:**

<p align="center">
  <img width="415" height="110" alt="image" src="https://github.com/user-attachments/assets/a86bfa06-530b-4f11-a268-c2d761cf5df5" />
</p>

</details>

<details>
<summary><strong>05. Product Segmentation by Cost</strong></summary>

**Business question:** How many products fall into each cost range?

**SQL query:**

<p align="center">
  <img width="561" height="627" alt="image" src="https://github.com/user-attachments/assets/89fcf8a6-fff0-4ab4-8844-a7b756f69c0e" />
</p>

**Result:**

<p align="center">
  <img width="218" height="110" alt="image" src="https://github.com/user-attachments/assets/274d3ca6-34fd-4813-a691-93738b9781e0" />
</p>

</details>

<details>
<summary><strong>06. Customer Segmentation by Spending Behavior</strong></summary>

**Business question:** How many customers are VIP, Regular, or New based on lifespan and spending?

**SQL query:**

<p align="center">
 <img width="750" height="874" alt="image" src="https://github.com/user-attachments/assets/91641bcb-1e74-46f3-965f-f82745d3e65c" />
</p>

**Result:**

<p align="center">
  <img width="266" height="120" alt="image" src="https://github.com/user-attachments/assets/bbe04bd1-2605-483d-bb6d-5e7984bd7084" />
</p>

</details>

<details>
<summary><strong>07. Customer Report View</strong></summary>

**Business question:** What customer-level KPIs are needed for BI and customer analytics?

**SQL query:**

<p align="center">
  <img width="710" height="1176" alt="image" src="https://github.com/user-attachments/assets/22c292b9-b0a8-4ab4-a1e6-c461d60edb0d" />
  <img width="705" height="891" alt="image" src="https://github.com/user-attachments/assets/3b4721fb-e4ff-43b3-9eac-28984571910f" />
</p>

**Result:**

<p align="center">
  <img width="1498" height="250" alt="image" src="https://github.com/user-attachments/assets/ede3a7f6-9c7b-4dd1-9c7c-445526d01850" />
</p>

</details>

<details>
<summary><strong>08. Product Report View</strong></summary>

**Business question:** What product-level KPIs are needed for BI and product performance analysis?

**SQL query:**

<p align="center">
  <img width="932" height="1318" alt="image" src="https://github.com/user-attachments/assets/a2d9464b-0536-46c3-ae11-da974cef26d3" />
  <img width="678" height="837" alt="image" src="https://github.com/user-attachments/assets/d4a973f8-28e6-414f-bb89-0d8b8ff800b9" />
</p>

**Result:**

<p align="center">
  <img width="1591" height="248" alt="image" src="https://github.com/user-attachments/assets/a1281898-229d-419c-906d-03caf50f7add" />
</p>

</details>

## Data Quality Checks

The project includes quality checks to validate the warehouse before analytical use.

| Layer | Checks |
| --- | --- |
| Silver | Null keys, duplicate records, invalid dates, inconsistent values, incorrect calculations. |
| Gold | Duplicate surrogate keys, missing dimension relationships, orphaned fact records. |

## How to Run the Project

1. Open SQL Server Management Studio or Azure Data Studio.
2. Run `scripts/init_database.sql` to initialize the database and schemas.
3. Run `scripts/bronze/ddl_bronze.sql` to create Bronze tables.
4. Run `scripts/bronze/proc_load_bronze.sql` to create the Bronze load procedure, then execute `EXEC bronze.load_bronze;`.
5. Run `scripts/silver/ddl_silver.sql` to create Silver tables.
6. Run `scripts/silver/proc_doc_silver.sql` to create the Silver transformation procedure, then execute `EXEC silver.load_silver;`.
7. Run `scripts/gold/ddl_gold.sql` to create Gold analytical views.
8. Run `scripts/analytics/analytics_queries.sql` to execute analytical queries and create reporting views.
9. Run the SQL files in `tests/` to validate Silver and Gold data quality.

## Documentation

- [Data Catalog](docs/Data%20Catalog.md): Business and technical descriptions for Gold Layer objects.
- [Architecture Diagram](docs/Data%20Architecture.png): High-level warehouse architecture.
- [Data Flow Diagram](docs/Data%20Flow%20Diagram.png): End-to-end data movement across layers.
- [Integration Model](docs/Integration%20Model.png): CRM and ERP relationship mapping.
- [Data Model](docs/Data%20Model.png): Gold Layer Star Schema.

## Tech Stack

- SQL Server
- T-SQL
- Stored Procedures
- Views
- Window Functions
- Data Warehousing
- ETL
- Star Schema Modeling

## Outcome

The final warehouse provides a clean and integrated analytical model that can support BI dashboards, ad-hoc SQL analysis, reporting, and future machine learning use cases.
