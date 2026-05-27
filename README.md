# Data Warehouse | SQL Server

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

<img width="1172" height="629" alt="Data Architecture" src="https://github.com/user-attachments/assets/def86645-f99e-44f6-9928-9e3e37ff48b8" />

## Data Flow

The data flow begins with source files from CRM and ERP systems. Each source table is loaded into Bronze, cleaned into Silver, and integrated into Gold dimensions and facts.

<img width="961" height="553" alt="Data Flow Diagram" src="https://github.com/user-attachments/assets/7a3b86af-9771-4035-af90-f82562770f97" />

## Data Integration Model

The integration model shows how CRM and ERP entities relate to one another before they are combined into the final analytical model.

<img width="698" height="434" alt="Data Integration" src="https://github.com/user-attachments/assets/8e5e792d-6c4b-4d78-90ed-33f9d320abdb" />

## Data Mart Model

The Gold layer is modeled as a Star Schema with two dimensions and one fact view:

- `gold.dim_customers`
- `gold.dim_products`
- `gold.fact_sales`

<img width="761" height="424" alt="Data Model" src="https://github.com/user-attachments/assets/6be5b4d3-8dd0-49be-b5b1-7188cd7d2fb9" />

## Repository Structure

```text
Data_Warehouse_SQL/
|-- docs/
|   |-- data_catalog.md
|   |-- data_architecture.png
|   |-- data_flow_diagram.png
|   |-- data_integration.png
|   |-- data_model.png
|-- scripts/
|   |-- bronze/
|   |   |-- ddl_bronze.sql
|   |   |-- proc_load_bronze.sql
|   |-- silver/
|   |   |-- ddl_silver.sql
|   |   |-- proc_doc_silver.sql
|   |-- gold/
|   |   |-- ddl_gold.sql
|-- tests/
|   |-- quality_checks_silver.sql
|   |-- quality_checks_gold.sql
|-- README.md
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

**Results**

<img width="1388" height="826" alt="image" src="https://github.com/user-attachments/assets/8dd32f23-aa31-42b7-bf08-ec522dbb3dc9" />

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

**Results**

<img width="1470" height="828" alt="Screenshot 2026-05-27 152253" src="https://github.com/user-attachments/assets/4106fb28-d684-4a2e-ae87-f2b666900680" />

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

**Results**

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
8. Run the SQL files in `tests/` to validate Silver and Gold data quality.

## Documentation

- [Data Catalog](docs/Data%20Catalog.md): Business and technical descriptions for Gold Layer objects.
- [Architecture Diagram](docs/Data%20Architecture.png): High-level warehouse architecture.
- [Data Flow Diagram](docs/Data%20Flow%20Diagram.png): End-to-end data movement across layers.
- [Integration Model](docs/Data%20Integration.png): CRM and ERP relationship mapping.
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

## Project Outcome

The final warehouse provides a clean and integrated analytical model that can support BI dashboards, ad-hoc SQL analysis, reporting, and future machine learning use cases.
