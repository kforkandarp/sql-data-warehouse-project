# SQL Data Warehouse & Analytics Project

An end-to-end T-SQL data warehouse that ingests raw CRM and ERP CSV exports into
Microsoft SQL Server, reconciles them across sources, and models them into a
star schema for analytics — built as automated, re-runnable stored procedures,
not one-off scripts.

## Architecture

**Bronze (raw):** CRM (`cust_info`, `prd_info`, `sales_details`) and ERP
(`loc_a101`, `cust_az12`, `px_cat_g1v2`) tables are truncated and reloaded via
`BULK INSERT` on every run, so the warehouse is always reproducible from the
source CSVs rather than accumulating drift over repeated loads.

**Silver (cleansed, reconciled):** Each table gets its own transformation logic —
not a generic cleaning pass. Nulls, duplicates, formatting inconsistencies and
cross-source conflicts are resolved per-field, based on what each source
actually gets wrong (see Data Quality below).

**Gold (business, star schema):** Two dimension views (`dim_customers`,
`dim_products`) and one fact view (`fact_sales`), built directly on top of
silver with surrogate keys generated via `ROW_NUMBER()`.

Both load procedures (`bronze.load_bronze`, `silver.load_silver`) run inside
`TRY/CATCH` blocks, print per-table load duration, and surface the SQL error
message, number, and state on failure rather than failing silently.

## Key data quality and transformation decisions

- **Sales figures are repaired, not just flagged.** Where `sls_sales` is missing,
  zero, or doesn't equal `quantity × ABS(price)`, it's recalculated from quantity
  and price rather than discarded. Symmetrically, where `sls_price` is missing or
  invalid, it's derived back from sales and quantity. Both directions are handled
  because either field can be the one that's wrong in the source data.
- **Product versioning via window functions.** `crm_prd_info` has no explicit
  end date per product version — it's inferred with
  `LEAD(prd_start_dt) OVER (PARTITION BY prd_key ORDER BY prd_start_dt) - 1`,
  so each version's end date is one day before the next version's start date.
  `dim_products` then filters to `prd_end_dt IS NULL` to keep only the current
  version of each product.
- **Most-recent-record deduplication.** `crm_cust_info` can contain multiple
  rows per `cst_id`; only the latest by `cst_create_date` is kept, using
  `ROW_NUMBER() OVER (PARTITION BY cst_id ORDER BY cst_create_date DESC)` —
  the same pattern used later to resolve duplicate/stale records in general.
- **Cross-source conflict resolution, resolved at the gold layer.** Gender
  exists in both CRM and ERP data and the two don't always agree. CRM is
  treated as the primary source; ERP's value is only used as a fallback when
  CRM's is `'n/a'` — an explicit, intentional precedence rule rather than
  picking whichever source loaded last. This reconciliation lives in the
  `gold.dim_customers` view itself, not in silver — silver cleans each source
  independently, and gold is where the two are actually merged and conflicts
  resolved.
- **Load traceability.** Every silver table carries a `dwh_create_date`
  column (defaulted to `GETDATE()`), so each row records when it entered the
  warehouse — absent from bronze, since bronze is a pure, disposable
  reload-on-every-run staging area.
- **Encoded and malformed values are normalized, not dropped.** Single-letter
  codes (`M`/`F`, `S`/`M`, `M`/`R`/`S`/`T`) are expanded to readable values;
  numeric dates stored as integers (e.g. `20250101`) are validated for length
  and plausible range before casting to `DATE`, with anything malformed set to
  `NULL` rather than causing a cast error; a stray `'NAS'` prefix on some
  customer IDs is stripped so they join correctly against CRM.

## Data quality checks

11 checks across silver and gold, each written to return **zero rows** when
the data is clean — so any result is itself the failure signal:

| Layer | Checks |
|---|---|
| Silver | Null/duplicate primary keys, unwanted whitespace, negative/null costs, invalid date ranges (`end_date < start_date`), implausible numeric dates, `order_date` after `ship_date`/`due_date`, `sales ≠ quantity × price`, out-of-range birthdates, distinct-value scans for standardization drift |
| Gold | Surrogate key uniqueness in both dimensions, referential integrity between `fact_sales` and each dimension (left join, expect no unmatched rows) |

## Repository structure

```
datasets/          Source CSVs (CRM and ERP exports)
scripts/
  bronze/          DDL + load procedure (BULK INSERT, truncate-and-reload)
  silver/          DDL + load procedure (cleansing, reconciliation, dedup)
  gold/            Star schema views (dimensions + fact)
tests/             Data quality checks for silver and gold
docs/              Data catalog describing every column across all layers
```

## Tech stack

Microsoft SQL Server, T-SQL stored procedures, SSMS. No external orchestration —
`EXEC bronze.load_bronze; EXEC silver.load_silver;` runs the full pipeline, then
gold views compute live on query.