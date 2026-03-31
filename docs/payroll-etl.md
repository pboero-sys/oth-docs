# Avita PM Payroll Bill-Back ETL — Cloud Function

Automated pipeline that runs every pay period and eliminates manual JE work.

## Architecture

```
Cloud Scheduler (every 2 weeks, Thursday 9am CT)
        │
        ▼
Cloud Function (avita-payroll-etl)
        │
        ├── Gusto API ──────────────── pull processed payroll
        ├── Supabase ───────────────── load employee→property assignments
        │                              + float allocation rules
        ├── BQ oth_silver.dim_property  resolve sage_entity_id + accounting_system
        │                              (never hardcoded — always live from BQ)
        ├── Allocator ──────────────── calculate per-property cost splits
        ├── SageExporter ───────────── generate import CSVs
        │       ├── sage_import_{date}.csv     (Fortress → Sage manual import)
        │       ├── yardi_je_{date}.csv        (Yardi properties)
        │       └── appfolio_je_{date}.csv     (Appfolio properties)
        ├── BQ oth_gold ─────────────── log run + allocation history
        ├── GCS (oth-payroll-imports) ── store import files (7-day links)
        └── Slack (#avita_accounting) ── notify with summary + import link
```

## Key Design Decisions

**BQ as source of truth for property metadata:**
- `sage_entity_id` (LOCATION_ID) — queried from `oth_silver.dim_property` at runtime
- `accounting_system` (sage/yardi/appfolio) — queried from `oth_silver.dim_property`
- No hardcoded property maps anywhere in the codebase

**Sage API is read-only:**
- `intacct_client.py` (in oth-intacct repo) only extracts data into BQ
- Write path to Sage = CSV import (Sean reviews and imports)
- Future: Sage Web Services write path if/when needed

**Supabase for operational config:**
- Float allocation rules (employee X = 33% property A + 33% B + 33% C)
- Updated by HR when assignments change — not a spreadsheet

## Project Structure

```
payroll-etl-function/
├── main.py                        # Cloud Function entry point
├── requirements.txt
├── deploy.sh                      # One-time deployment script
├── bq_schemas/
│   ├── payroll_bill_back_runs.json
│   └── payroll_bill_back_allocations.json
└── src/
    ├── gusto.py                   # Gusto API client
    ├── assignments.py             # Supabase assignment loader
    ├── allocator.py               # Cost allocation (BQ for system lookup)
    ├── sage_export.py             # CSV generator (BQ for entity IDs)
    ├── bigquery_writer.py         # BQ writer (oth_gold layer)
    ├── gcs_uploader.py            # GCS file uploader
    ├── slack_notifier.py          # Slack notification
    └── secrets.py                 # GCP Secret Manager helper
```

## BQ dim_property — Expected Schema

The function expects `oth_silver.dim_property` to have:
- `property_name` — matches names used in Supabase assignments
- `sage_entity_id` — exact Sage location ID string
- `accounting_system` — "sage", "yardi", or "appfolio"
- `status` — "active" or "inactive"

If `accounting_system` column doesn't exist yet in dim_property,
add it via Claude Code:
```sql
ALTER TABLE `oth-data-warehouse.oth_silver.dim_property`
ADD COLUMN accounting_system STRING;

UPDATE `oth-data-warehouse.oth_silver.dim_property`
SET accounting_system = CASE
  -- Fortress → Sage
  WHEN property_name IN (
    '1800 Broadway', 'Avita Alamo Heights', 'Forest Park',
    'Heron on Hausman', 'Lakeline Parmer Lane', 'Muir Lake',
    'Silver Springs', 'Summit at Rivery Park', 'Windsong Place'
  ) THEN 'sage'
  -- Yardi
  WHEN property_name IN (
    'Eryngo Hills', 'Iconic at Mueller', 'Riverhouse', 'Villas de Santa Fe'
  ) THEN 'yardi'
  -- Appfolio
  WHEN property_name IN ('Cottages') THEN 'appfolio'
  ELSE 'sage'
END
WHERE TRUE;
```

## Secrets Required

| Secret ID             | Value                              |
|-----------------------|------------------------------------|
| `gusto-client-id`     | Gusto OAuth app client ID          |
| `gusto-client-secret` | Gusto OAuth app client secret      |
| `supabase-service-key`| Supabase service role key          |
| `slack-bot-token`     | Slack bot token (xoxb-...)         |

## Deployment

```bash
cd payroll-etl-function
chmod +x deploy.sh
./deploy.sh
```

## Manual Trigger

```bash
# Dry run
curl -X POST $FUNCTION_URL \
  -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
  -H "Content-Type: application/json" \
  -d '{"dry_run": true}'

# Specific pay period
curl -X POST $FUNCTION_URL \
  -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
  -H "Content-Type: application/json" \
  -d '{"pay_period_end": "2026-03-21"}'
```

## Open Items Before First Production Run

1. **Gusto API credentials** — add to Secret Manager once approved
2. **`accounting_system` column in dim_property** — run ALTER/UPDATE above via Claude Code
3. **Jonathan Mata** — confirm 3 properties for float split
4. **Emilly De Oliveira** — confirm property assignment
5. **Supabase employee_property_assignments** — seed from master staff sheet
