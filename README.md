# SQL Server Data Warehouse (Medallion Architecture)

A data warehouse built in **SQL Server** that consolidates sales data from two source systems (CRM and ERP) into a clean star schema ready for analytics.

> **Credit:** This project follows along with [Data With Baraa](LINK-TO-THE-ORIGINAL-PROJECT)'s SQL Data Warehouse project. I had no prior data warehousing experience and built it to strengthen my SQL skills and step into data engineering. The architecture and datasets come from that project; this repo is my own build-along, with my own notes and what I learned along the way.

## Project Status

| Part | Scope | Status |
|------|-------|--------|
| 1. Data Engineering | Build the warehouse: ingest, clean, integrate, model | ✅ Done |
| 2. BI: Analytics & Reporting | SQL analysis of customer behavior, product performance and sales trends | 🔜 Next |

## Architecture


| Layer | Purpose | Load method | Object type |
|-------|---------|-------------|-------------|
| **Bronze** | Raw copy of the source files, no changes | `BULK INSERT` (truncate and load) | Tables |
| **Silver** | Cleaned, standardized, deduplicated data | Stored procedure, transformations in SQL | Tables |
| **Gold** | Business-ready star schema | Views over silver | Views |

## Data Sources

Two source systems delivered as CSV files:

- **CRM:** `cust_info`, `prd_info`, `sales_details`
- **ERP:** `cust_az12`, `loc_a101`, `px_cat_g1v2`

Scope is the latest data only. Historization (slowly changing dimensions) was out of scope.

## Data Cleaning (Silver Layer)

- **Deduplication:** `ROW_NUMBER()` keeps the most recent record per customer
- **Standardization:** codes expanded to readable values (marital status, gender, product line, country)
- **Whitespace:** `TRIM` applied to text fields
- **Key alignment:** customer IDs reconciled between CRM and ERP (removed `NAS` prefixes and stray dashes)
- **Invalid dates:** malformed integer dates converted to `DATE` or set to `NULL`; future birthdates set to `NULL`
- **Sales logic:** sales, quantity and price recalculated when they didn't agree or were missing or negative
- **Product history:** product end dates derived with `LEAD()` from the next start date
- **Missing values:** `n/a` used for unknown categories

## Gold Layer: Star Schema

**`gold.dim_customers`** combines CRM customer data with ERP demographics and location. Gender comes from the CRM and falls back to the ERP when the CRM value is unknown.

| Column | Description |
|--------|-------------|
| `customer_key` | Surrogate key |
| `customer_id` | Numeric customer ID |
| `customer_name` | Alphanumeric customer code from the source system |
| `first_name`, `last_name` | Customer name |
| `country` | Country of residence |
| `marital_status`, `gender` | Standardized demographics |
| `birthdate` | Date of birth |
| `create_date` | Date the customer record was created |

**`gold.dim_products`** holds current products only, enriched with ERP category data.

| Column | Description |
|--------|-------------|
| `product_key` | Surrogate key |
| `product_id`, `product_number` | Source identifiers |
| `product_name` | Product description |
| `category_id`, `category`, `subcategory` | Product classification |
| `maintenance` | Whether the product requires maintenance |
| `cost` | Base cost |
| `product_line` | Product series (e.g. Road, Mountain) |
| `start_date` | Date the product became available |

**`gold.fact_sales`** has one row per order line.

| Column | Description |
|--------|-------------|
| `order_number` | Sales order ID |
| `product_key`, `customer_key` | Links to the dimensions |
| `order_date`, `shipping_date`, `due_date` | Order dates |
| `sales_amount`, `quantity`, `price` | Line item measures |

## Data Quality Checks

Test scripts validate each layer:

- **Silver:** duplicate or null keys, unwanted spaces, standardized values, valid date ranges and order, and sales = quantity × price
- **Gold:** surrogate key uniqueness and referential integrity between the fact table and both dimensions

## How to Run

1. Create the database and the `bronze`, `silver` and `gold` schemas.
2. Create the bronze and silver tables using the DDL scripts.
3. Update the file paths inside the bronze load procedure to match where your CSVs are stored.
4. Run the pipeline:
   ```sql
   EXEC bronze.load_bronze;
   EXEC silver.load_silver;
   ```
5. Create the gold views using the gold DDL script.
6. Run the quality check scripts and confirm they return the expected results.

## Known Limitations

- Amounts (`sales_amount`, `price`, `cost`) are stored as whole numbers, so cents are not preserved.
- Surrogate keys use `ROW_NUMBER()` in views, so they are not stable across reloads. This is acceptable for a full-refresh design.
- No history tracking: only the latest data is modeled.
- Source file paths in the bronze load are hardcoded.
- Procedures print errors but do not re-throw them.

## What I Learned

- How medallion architecture separates raw, cleaned and business-ready data
- Writing repeatable ETL with stored procedures
- Spotting data quality problems and fixing them in SQL
- Designing a star schema with surrogate keys
- Testing a warehouse with validation queries

## Next Steps

Part 2 will use the gold layer to answer business questions in SQL:

- **Customer behavior:** who buys, from where, and how often
- **Product performance:** best and worst categories, lines and products
- **Sales trends:** performance over time

## Tech

SQL Server, T-SQL, SSMS, Git/GitHub
