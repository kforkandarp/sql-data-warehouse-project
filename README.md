# 🏢 SQL Data Warehouse & Analytics Project

> An end-to-end **T-SQL data warehouse** built on Microsoft SQL Server, transforming
> raw CRM and ERP exports into a reconciled **star schema for analytics**.

### ⚡ Project Highlights

- 🔄 **Re-runnable ETL** using stored procedures
- 🥉 **Bronze → Silver → Gold** architecture
- 🔀 **CRM + ERP reconciliation**
- ⭐ **Star schema** for analytics
- 🧹 **Data quality & validation checks**
- 🛡️ **TRY/CATCH error handling**


## 📌 At a Glance

| | |
|---|---|
| Database | Microsoft SQL Server |
| Language | T-SQL |
| Architecture | Bronze → Silver → Gold |
| Modeling | Star Schema |
| Loading | `BULK INSERT` + Stored Procedures |
| Data Quality | 11 checks across Silver & Gold |
| IDE | SQL Server Management Studio (SSMS) |

## 🏗️ Architecture

The warehouse follows a **Bronze → Silver → Gold** flow.

### 🥉 Bronze — Raw Layer

- Loads CRM and ERP CSV exports using `BULK INSERT`
- Truncates and reloads on every run
- Keeps the raw source structure intact
- Makes the warehouse reproducible from the source files

### 🥈 Silver — Cleansed & Reconciled

- Handles nulls and duplicates
- Normalizes inconsistent formats
- Applies source-specific cleansing rules
- Resolves data quality issues across CRM and ERP

### 🥇 Gold — Analytics Layer

- `dim_customers`
- `dim_products`
- `fact_sales`
- Uses surrogate keys generated with `ROW_NUMBER()`

### 🛡️ Error Handling

Both `bronze.load_bronze` and `silver.load_silver` use `TRY/CATCH` blocks to:

- Report per-table load duration
- Surface SQL error message, number and state
- Prevent failures from being silently ignored

## 🧠 Key Engineering Decisions

### 💰 Sales Validation & Repair

- Invalid or missing `sls_sales` is recalculated using:
  `quantity × ABS(price)`
- Invalid or missing `sls_price` is derived from:
  `sales / quantity`
- Both directions are handled because either source field can be incorrect

### 📅 Product Versioning

- Product versions are inferred using `LEAD()`
- Each version ends one day before the next version begins
- `dim_products` keeps only the current version

### 👥 Customer Deduplication

- Multiple records per `cst_id` are possible
- The most recent record is retained using `ROW_NUMBER()`
- The same pattern is used to resolve duplicate/stale records

### 🔀 CRM + ERP Reconciliation

- CRM is the primary source for gender
- ERP is used only when CRM contains `'n/a'`
- Conflict resolution happens at the Gold layer

### 🕒 Load Traceability

- Every Silver table carries `dwh_create_date`
- The timestamp records when a row entered the warehouse
- Bronze remains a disposable reload-on-every-run staging layer

### 🧹 Value Normalization

- Encoded values such as `M` / `F` are expanded
- Numeric dates such as `20250101` are validated before casting
- Malformed dates become `NULL`
- Stray `NAS` prefixes are removed from customer IDs

## 🔍 Data Quality

**11 automated checks** across Silver and Gold.

> Every check should return **zero rows** when the data is valid.
> Any returned row is a failure signal.

| Layer | Checks |
|---|---|
| Silver | Null/duplicate primary keys, unwanted whitespace, negative/null costs, invalid date ranges (`end_date < start_date`), implausible numeric dates, `order_date` after `ship_date`/`due_date`, `sales ≠ quantity × price`, out-of-range birthdates, distinct-value scans for standardization drift |
| Gold | Surrogate key uniqueness in both dimensions, referential integrity between `fact_sales` and each dimension (left join, expect no unmatched rows) |




## 🔄 Data Flow

```
CRM CSVs ──┐
           ├──> 🥉 Bronze ──> 🥈 Silver ──> 🥇 Gold ──> 📊 Analytics
ERP CSVs ──┘
```


## 📁 Repository Structure

```
datasets/          Source CSVs (CRM and ERP exports)
scripts/
  bronze/          DDL + load procedure (BULK INSERT, truncate-and-reload)
  silver/          DDL + load procedure (cleansing, reconciliation, dedup)
  gold/            Star schema views (dimensions + fact)
tests/             Data quality checks for silver and gold
docs/              Data catalog describing every column across all layers
```

## 🛠️ Tech Stack

- 🗄️ **Database:** Microsoft SQL Server
- 💻 **Language:** T-SQL
- ⚙️ **ETL:** Stored Procedures + `BULK INSERT`
- 🖥️ **IDE:** SQL Server Management Studio (SSMS)
- ⭐ **Modeling:** Star Schema
- 🧪 **Validation:** SQL-based data quality checks

### ▶️ Running the Pipeline

```sql
EXEC bronze.load_bronze;
EXEC silver.load_silver;
```
Gold views then compute live when queried.