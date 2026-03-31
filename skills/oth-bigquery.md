---
name: OTH BigQuery Data Warehouse
description: Architecture and key tables for oth-data-warehouse BigQuery project — fact_financials, fact_financials_monthly, v_property_pnl, ETL patterns
type: reference
---

# OTH BigQuery Data Warehouse

**Project:** `oth-data-warehouse`
**Region:** US

## Datasets

| Dataset | Layer | Purpose |
|---------|-------|---------|
| `oth_bronze` | Raw | Raw ingested data, minimal transformation |
| `oth_silver` | Cleaned | Deduplicated, typed, tagged financial data |
| `oth_gold` | Presentation | Views and aggregates for dashboards/reporting |

## Key Tables & Views

### oth_silver.fact_financials
- **Source:** Sage Intacct GL journal entries via API
- **Primary key:** `record_no` (STRING) — Sage's unique entry identifier
- **Load pattern:** MERGE by `record_no` — never DELETE WHERE TRUE
- **ETL script:** `/Users/paoloboero/oth-intacct/intacct_client.py`
- **Schema:**
  - record_no, batch_date, account_no, entity_id, department
  - amount (FLOAT64), description, doc_number, currency
  - transaction_type (STRING: "1" debit, "-1" credit)
  - entry_date, batch_no, batch_title
- **Service account:** `warehouse-admin@oth-data-warehouse.iam.gserviceaccount.com`

### oth_silver.fact_financials_monthly
- **Type:** Materialized table (CTAS, not a view)
- **Source:** Rebuilt from fact_financials via CREATE OR REPLACE TABLE
- **Purpose:** Monthly aggregated financials by property/line_item
- **Schema:** property_id, period_month (DATE), tag, line_item, section, sort_order, net_amount
- **Filters applied during build:**
  - Excludes future months (entry_date < first of current month)
  - Deduplicates Batch Date entries (keeps highest-count batch per entity/month)
  - Excludes Payment batches (re-posted expenses)
  - Excludes Undeposited batches

### oth_gold.v_property_pnl
- **Type:** View (not materialized)
- **Format:** Wide — one row per (property_id, line_item), 12 value columns (t1-t12) + 12 date columns
- **Rolling window:** t1 = most recent completed month, t12 = 12 months ago
- **Anchor:** `DATE_DIFF(DATE_TRUNC(CURRENT_DATE(), MONTH), month_start, MONTH)` — shifts automatically
- **Computed rows:** effective_rental_income, physical_occupancy_pct, economic_occupancy_pct, effective_rent_pu, total_effective_income, controllable_expenses, uncontrollable_expenses, unlevered_noi, levered_noi, net_income

### oth_silver.dim_gl_account
- **Source:** Sage Intacct chart of accounts
- **Schema:** gl_account_id, account_name, account_type, normal_balance, status, category

### oth_silver.ref_coa_tags
- **Purpose:** Maps GL account numbers to standardized line_item names, sections, and sort_order
- **Schema:** account_no, tag, line_item, section, sort_order

### oth_silver.dim_property
- **Purpose:** Property reference data (unit counts, etc.)
- **Schema:** property_id, unit_count

## ETL Pipeline

```
Sage Intacct API
    |
    v  (intacct_client.py --start YYYY-MM-DD --end YYYY-MM-DD)
fact_financials  [MERGE by record_no, append-only]
    |
    v  (CTAS rebuild)
fact_financials_monthly  [aggregated by property/month/line_item]
    |
    v  (view)
v_property_pnl  [wide format, rolling T-12]
    |
    v  (Python injection script)
oth-pnl-v5.html dashboard
```

## Critical Rules

1. **NEVER run `DELETE FROM fact_financials WHERE TRUE`** — this wipes all historical data. The MERGE pattern in intacct_client.py makes this unnecessary.
2. **fact_financials_monthly must be rebuilt** after any changes to fact_financials (it's a CTAS, not a live view).
3. **v_property_pnl is a live view** — it automatically reflects changes in fact_financials_monthly.
4. **Rolling window shifts automatically** — when new month data is loaded and fact_financials_monthly is rebuilt, t1 moves forward and t12 drops the oldest month.

## Credentials

- Sage Intacct credentials in `/Users/paoloboero/oth-intacct/.env`
- BigQuery auth via `gcloud auth application-default login` (ADC)
- Do NOT use GOOGLE_APPLICATION_CREDENTIALS env var (removed from .zshrc)
