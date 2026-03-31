# OTH BigQuery Data Warehouse

**Project:** `oth-data-warehouse`
**Region:** US

## Datasets

| Dataset | Layer | Purpose |
|---------|-------|---------|
| oth_bronze | Raw | Raw ingested data, minimal transformation |
| oth_silver | Cleaned | Deduplicated, typed, tagged financial data |
| oth_gold | Presentation | Views and aggregates for dashboards and reporting |

## Key Tables

### oth_silver.fact_financials

Raw GL journal entries from Sage Intacct.

- **Primary key:** `record_no` (STRING) -- Sage's unique entry identifier
- **Load pattern:** MERGE by `record_no` via staging table. **Never run `DELETE FROM fact_financials WHERE TRUE`** -- this wipes all historical data and has caused data loss.
- **ETL script:** `intacct_client.py --start YYYY-MM-DD --end YYYY-MM-DD`
- **Schema:** record_no, batch_date, account_no, entity_id, department, amount (FLOAT64), description, doc_number, currency, transaction_type ("1" debit / "-1" credit), entry_date, batch_no, batch_title
- **Service account:** `warehouse-admin@oth-data-warehouse.iam.gserviceaccount.com`

### oth_silver.fact_financials_monthly

Monthly aggregated financials by property and line item.

- **Type:** Materialized table (CTAS, not a view)
- **Must be rebuilt** after any changes to fact_financials
- **Schema:** property_id, period_month (DATE), tag, line_item, section, sort_order, net_amount
- **Filters applied during build:**
  - Excludes future months (`entry_date < DATE_TRUNC(CURRENT_DATE(), MONTH)`)
  - Deduplicates Batch Date entries (keeps highest-count batch per entity/month)
  - Excludes Payment and Undeposited batches

### oth_gold.v_property_pnl

Presentation view for the P&L dashboard.

- **Type:** View (live, not materialized)
- **Format:** Wide -- one row per (property_id, line_item), 12 value columns (t1-t12) + 12 date columns
- **Rolling window:** t1 = most recent completed month, t12 = 12 months ago
- **Anchor:** `DATE_DIFF(DATE_TRUNC(CURRENT_DATE(), MONTH), month_start, MONTH)`
- **Computed rows:** effective_rental_income, physical_occupancy_pct, economic_occupancy_pct, effective_rent_pu, total_effective_income, controllable_expenses, uncontrollable_expenses, unlevered_noi, levered_noi, net_income

### oth_silver.dim_gl_account

Chart of accounts from Sage Intacct.

- **Schema:** gl_account_id, account_name, account_type, normal_balance, status, category

### oth_silver.ref_coa_tags

Maps GL account numbers to standardized line_item names, sections, and sort order.

- **Schema:** account_no, tag, line_item, section, sort_order

### oth_silver.dim_property

Property reference data.

- **Schema:** property_id, unit_count

## ETL Pipeline

```
Sage Intacct API
    |
    v  intacct_client.py --start YYYY-MM-DD --end YYYY-MM-DD
fact_financials  [MERGE by record_no -- append only, never delete]
    |
    v  CTAS rebuild
fact_financials_monthly  [aggregated by property/month/line_item]
    |
    v  live view
v_property_pnl  [wide format, rolling T-12]
    |
    v  Python injection script
oth-pnl-v5.html dashboard
```

## Critical Rules

1. **Never `DELETE FROM fact_financials WHERE TRUE`** -- use MERGE by record_no instead. Re-running the sync is safe and idempotent.
2. **Rebuild fact_financials_monthly** after loading new data into fact_financials (it is a CTAS, not a live view).
3. **v_property_pnl is a live view** -- it reflects fact_financials_monthly automatically.
4. **Rolling window shifts automatically** -- t1 moves forward when new month data lands and fact_financials_monthly is rebuilt.
