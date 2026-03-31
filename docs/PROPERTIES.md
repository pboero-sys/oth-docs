# OTH Capital / Avita PM — Property Reference

_Last updated: March 30, 2026_

---

## Active Managed Properties — Full P&L Reporting

These properties have full T-12 P&L reporting via `oth_gold.v_property_pnl`.
All are on Fortress (Sage integration) unless noted.

| Sage Entity ID | Property Name | Units | Platform |
|---|---|---|---|
| `18B` | 1800 Broadway | 230 | Fortress → Sage |
| `ML` | Muir Lake | 332 | Fortress → Sage |
| `AAH` | Avita Alamo Heights | 312 | Fortress → Sage |
| `HOH` | Heron on Hausman | 106 | Fortress → Sage |
| `SRP` | Summit at Rivery Park | 228 | Fortress → Sage |
| `SS` | Silver Springs | 360 | Fortress → Sage |
| `FRP` | Forest Park | 228 | Fortress → Sage |
| `WSP` | Windsong Place | 200 | Fortress → Sage |
| `LPL` | Lakeline Parmer Lane | 312 | Fortress → Sage |
| `A36` | Apartments 36 | 38 | Fortress → Sage |
| `616` | 616 W MLK | 6 | Fortress → Sage |
| `CTH` | Cottages at Terrell Hills (Brix) | — | Appfolio (Regency) |

BigQuery filter: `WHERE managed = TRUE` on `dim_property`

---

## Corporate Tracking Only — Historical PM Fees + Open Receivables

These properties are **not managed by Avita** and have **no property-level P&L reporting**.
They appear in Avita PM's Sage books only for:
- Historical property management fees received
- Potential open receivables (account 11014 — Avita PM Receivable)

| Sage Entity ID | Property Name | Offboard Date | Notes |
|---|---|---|---|
| `JOH` | The Johnny | ~April 23, 2025 | Part of ORIX portfolio departure |
| `CPZ` | Casa Paz | April 23, 2025 | Owner in financial distress at departure — $160K in arrears, collection risk on open AR |
| `ELT` | The Ellington | ~September 2025 | Part of ORIX portfolio — Pedro confirmed Sept 2025 departure |

**Context:** All three were part of the ORIX owner relationship. Karthik announced Casa Paz/Johnny resignation April 7, 2025 due to owner's loan distress. ORIX operated Ellington + Casa Paz + Celebrations + The Row — all departed together per Pedro's Sept 22, 2025 update.

**Open AR risk:** Casa Paz is highest priority — owner had active collections at offboard. Check account 11014 (Avita PM Receivable) for balances dated before each offboard date.

Reporting surface: `oth_gold.v_avita_receivables` (to be built)
BigQuery filter: `WHERE managed = FALSE` on `dim_property`

---

## Fortress Double-Posting Issue

**Background:** Fortress (the property management software) posts GL entries to Sage
Intacct **twice per month** for each property. This causes inflated revenue figures
(~4-6x) if both batches are included in P&L reporting.

**How it works:**

1. **1st of month** — Fortress posts ~200 individual unit-level AR sub-ledger charges
   (one line per unit — rent, pet fees, RUBS, etc.). These hit revenue accounts AND
   Accounts Receivable. The `batch_title` looks like: `01/01/2026 – Batch Date`

2. **~25th of month** — Fortress posts the same unit-level detail lines again with
   **offsetting debits that cancel them out**, PLUS a single summary entry with the
   correct monthly total. This is the P&L-accurate batch.

**Why the Excel models were correct:** The trial balance import feeding the P&L tab
uses Fortress's "Monthly Summary Transactions" export — which only ever sees the
summary entry, never the unit-level sub-ledger detail.

**The fix:** Filter `fact_financials` to exclude the 1st-of-month AR batches using:

```sql
NOT REGEXP_CONTAINS(batch_title, r'^\d{2}/\d{2}/\d{4}\s*[–-]\s*Batch Date')
```

This is applied in `oth_silver.fact_financials_monthly` as a WHERE clause.

**Which fields to use:**
- `batch_no` — the batch number in Sage
- `batch_title` — human-readable batch description (the distinguishing field)
- Both are now included in every GLENTRY pull from `intacct_client.py` (added March 2026)

---

## Sage Intacct API Notes

- **Object:** `GLENTRY` (not `GLJOURNALENTRYLINESDETAIL`)
- **No JOURNAL field** on GLENTRY — use `BATCHTITLE` / `BATCHNO` instead
- **Batch title patterns by type:**
  - Daily AR batches (exclude): `MM/DD/YYYY – Batch Date`
  - Accruals (keep): `MM/YYYY Accruals`, `MM/YYYY Interest Accrual`
  - Bills (keep): `Bills – 18B: YYYY/MM/DD Batch Summary Entry`
  - Payments (keep): `Payments(Bank-...) – 18B: ...`
  - Management fees (keep): `MM/YYYY Management Fee`
  - Adjustments (keep): `GPR Adjustment`, `Monthly Depreciation`, `Mortgage Payment`
  - Reversals (keep): `Reversed – MM/YYYY Accruals`

---

## BigQuery Dataset Structure

```
oth-data-warehouse
├── oth_bronze
│   └── raw_ingestion          (currently unused — ETL loads directly to silver)
├── oth_silver
│   ├── fact_financials         (3.36M rows, Jan 2022 – Dec 2025, 59 entities)
│   ├── fact_financials_monthly (aggregated monthly, batch_title filtered)
│   ├── dim_gl_account          (chart of accounts, incomestatement + balancesheet)
│   ├── dim_property            (property master, managed flag, unit_count)
│   └── ref_pnl_line_items      (account → P&L line item mapping, 280 accounts)
└── oth_gold
    └── v_property_pnl          (T-12 trailing P&L view, 12 active properties)
```

---

## Data Freshness

- `fact_financials`: pulled via `intacct_client.py --start YYYY-MM-DD --end YYYY-MM-DD`
- Current coverage: Jan 2022 – Dec 2025 (backfill completed March 2026)
- Closed period gate: entries before current month only (open month excluded)
- Scheduled refresh: TBD (Cloud Function to be built)
