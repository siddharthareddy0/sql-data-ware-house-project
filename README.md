# SQL Data Warehouse & ETL Pipeline (SQL Server)

An end-to-end data warehouse built on **SQL Server** that integrates two source systems (CRM and ERP) into a single star schema for sales analytics. Raw CSV files are loaded, cleaned, and modeled using a **Bronze → Silver → Gold** (Medallion) architecture, with data quality checks at every layer.

**Tech stack:** SQL Server, T-SQL (stored procedures, views, window functions), SSMS, Draw.io, Git/GitHub

---

## Architecture

```
 CRM (3 CSV files)  ┐
                    ├──►  BRONZE  ──►  SILVER  ──►  GOLD
 ERP (3 CSV files)  ┘     raw load     cleaning,     star schema
                                       standardizing  (views)
```

| Layer | Purpose | How it is built |
|-------|---------|-----------------|
| **Bronze** | Raw data, stored as-is from the source files | Stored procedure `bronze.load_bronze` truncates each table and loads the CSVs with `BULK INSERT` |
| **Silver** | Cleaned, standardized, deduplicated data | Stored procedure `silver.load_silver` applies the transformations below |
| **Gold** | Business-ready star schema for reporting | Three views: `gold.dim_customers`, `gold.dim_products`, `gold.fact_sales` |

Architecture, data flow, and data model diagrams are in the [`docs/`](docs/) and [`Architecture/`](Architecture/) folders.

---

## Source Data

Six CSV files from two systems, about **100K rows** in total.

| System | File | Content | Approx. rows |
|--------|------|---------|--------------|
| CRM | `cust_info.csv` | Customer details | 18.5K |
| CRM | `prd_info.csv` | Product details and history | 400 |
| CRM | `sales_details.csv` | Sales transactions | 60.4K |
| ERP | `CUST_AZ12.csv` | Customer birthdate and gender | 18.5K |
| ERP | `LOC_A101.csv` | Customer country | 18.5K |
| ERP | `PX_CAT_G1V2.csv` | Product categories | 36 |

---

## Data Cleaning & Transformation (Silver Layer)

| Area | What the pipeline does |
|------|------------------------|
| **Duplicates** | Keeps the latest record per customer using `ROW_NUMBER() OVER (PARTITION BY cst_id ORDER BY cst_create_date DESC)` |
| **Text cleanup** | Trims unwanted spaces from names and categories |
| **Standardization** | Converts coded values to readable ones (`M` → Married/Male, `F` → Female, `DE` → Germany, `M`/`R`/`T` → Mountain/Road/Touring) |
| **Missing values** | Replaces NULL product costs with 0 and unknown values with `n/a` |
| **Dates** | Converts integer dates (`YYYYMMDD`) to `DATE`, and sets invalid dates to NULL |
| **Product history** | Derives each product's end date with `LEAD()` so date ranges do not overlap |
| **Sales validation** | Recalculates sales as `quantity × price` when the amount is missing, zero, or inconsistent |
| **Key cleanup** | Removes prefixes and dashes so ERP and CRM customer IDs match |

Every Silver table also has a `dwh_create_date` audit column.

---

## Data Model (Gold Layer)

A **star schema** with one fact table and two dimensions:

```
   dim_customers                    dim_products
   (customer_key)                   (product_key)
         │                                │
         └──────────►  fact_sales  ◄──────┘
```

- `dim_customers`: CRM and ERP customer data combined (name, country, gender, marital status, birthdate)
- `dim_products`: CRM products joined with ERP categories and subcategories
- `fact_sales`: order number, dates, sales amount, quantity, and price, linked by surrogate keys

Surrogate keys are generated with `ROW_NUMBER()`. Column-level documentation is in [`docs/data_catalog.md`](docs/data_catalog.md).

---

## Data Quality & Reliability

**Quality checks** (in [`tests/`](tests/)):

- **Silver** (`quality_checks_silver.sql`) covers all six tables:
  - NULL or duplicate primary keys
  - Unwanted spaces
  - Invalid date ranges and date order (start > end, order > ship)
  - Sales = quantity × price consistency
  - Standardized value checks
- **Gold** (`quality_checks_gold.sql`):
  - Uniqueness of surrogate keys
  - Fact-to-dimension connectivity (no orphan records)

**Pipeline reliability:**

- `TRY...CATCH` error handling in the load procedures
- Load duration tracked per table and per batch with `DATEDIFF`

---

## Project Structure

```
├── datasets/            # Source CSV files (CRM and ERP)
├── scripts/
│   ├── init_database.sql    # Creates DataWarehouse database and 3 schemas
│   ├── bronze/              # Table DDL + load_bronze procedure
│   ├── silver/              # Table DDL + load_silver procedure
│   └── gold/                # Star schema views
├── tests/               # Data quality checks (Silver and Gold)
├── docs/                # Diagrams, data catalog, naming conventions
├── Architecture/        # Draw.io architecture file
└── README.md
```

---

## How to Run

**Requirements:** SQL Server (Express edition is enough) and SQL Server Management Studio (SSMS).

1. Clone the repository and copy the `datasets/` folder to a local path.
2. In `scripts/bronze/proc_load_bronze.sql`, update the `BULK INSERT` file paths to match your local folder.
3. Run the scripts in this order:

```sql
-- 1. Create database and schemas  (WARNING: drops DataWarehouse if it already exists)
scripts/init_database.sql

-- 2. Create tables
scripts/bronze/ddl_bronze.sql
scripts/silver/ddl_silver.sql

-- 3. Create the load procedures
scripts/bronze/proc_load_bronze.sql
scripts/silver/proc_load_silver.sql

-- 4. Load data
EXEC bronze.load_bronze;
EXEC silver.load_silver;

-- 5. Create the Gold views
scripts/gold/ddl_gold.sql
```

4. Validate the results by running the scripts in `tests/`. Each check should return **no rows**.

---

## Sample Analysis

```sql
-- Total sales by country
SELECT c.country, SUM(f.sales_amount) AS total_sales
FROM gold.fact_sales f
JOIN gold.dim_customers c ON f.customer_key = c.customer_key
GROUP BY c.country
ORDER BY total_sales DESC;
```

---

## Acknowledgements

The dataset, folder structure, and overall approach follow the SQL Data Warehouse project by **Baraa Khatib Salkini ([Data With Baraa](https://www.youtube.com/@DataWithBaraa))**. I built the pipeline by following that project, and I have documented it here in my own words.

---

## Author

**Narra Siddhartha Reddy**
Junior Data Engineer | Hyderabad, India
[GitHub](https://github.com/siddharthareddy0) · [LinkedIn](https://linkedin.com/in/siddhartha-reddy-narra)
