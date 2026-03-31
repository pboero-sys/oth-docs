# OTH P&L Dashboard

**File:** `~/Downloads/oth-pnl-v5.html`

## KEY_MAP: BigQuery line_item to JS Short Key

| BigQuery line_item | JS Key | Display Label |
|---|---|---|
| gross_potential_rent | gpr | Gross Potential Rent |
| loss_to_lease | ltl | (-) Loss to Lease |
| vacancy_loss | vac | (-) Vacancy Loss |
| concessions | conc | (-) Concessions |
| non_revenue_units | nr | (-) Non Revenue Units |
| bad_debt | bad | (-) Bad Debt |
| effective_rental_income | eri | Effective Rental Income |
| physical_occupancy_pct | phys | Physical Occupancy |
| economic_occupancy_pct | econ | Economic Occupancy |
| effective_rent_pu | epu | Eff Rent / Unit |
| rubs / rubs_water / rubs_trash / rubs_general / rubs_electric / rubs_gas | rubs | RUBS |
| other_income | oi | Other Income |
| total_effective_income | tei | Total Effective Income |
| utilities_water | utilw | Utilities - Water |
| utilities_electric | utile | Utilities - Electric |
| utilities_gas | utilg | Utilities - Gas |
| utilities_trash | utilt | Utilities - Trash |
| utilities_general | utilx | Utilities - General |
| utilities | util | Utilities |
| marketing | mkt | Marketing |
| admin | adm | Admin |
| contract_services | cont | Contract Services |
| maintenance | mtc | Maintenance |
| turnover_expense | tox | Turnover Expense |
| payroll | pay | Payroll |
| management_fee | pm | Management Fee |
| controllable_expenses | ctrl | Controllable Expenses |
| property_taxes | tax | Property Taxes |
| insurance | ins | Insurance |
| uncontrollable_expenses | uctrl | Uncontrollable Expenses |
| unlevered_noi | noi | Unlevered NOI |
| interest_payment | ipmt | Interest Payment |
| principal_payment | ppmt | Principal Payment |
| debt_service | ds | Debt Service |
| levered_noi | lnoi | Levered NOI |
| am_fee | amfee | AM Fee |
| non_operating / non_operating_income | nonop | Non Operating Income |
| depreciation | dep | Depreciation |
| net_income | ni | Net Income |
| debt_balance | debt | Debt Balance |
| equity_balance | eq | Equity Balance |
| total_capital_basis | basis | Total Capital Basis |

## Sign Convention

Expenses come from Sage Intacct as **positive numbers**. The dashboard must negate them for display.

**Keys that must be multiplied by -1 before display (NEGATIVE_KEYS):**

```
ltl, vac, conc, nr, bad,
utilw, utile, utilg, utilt, utilx, util,
mkt, adm, cont, mtc, tox, pay, pm, ctrl,
tax, ins, uctrl,
ipmt, ppmt, ds,
amfee, dep
```

**Keys that can be positive or negative (no forced sign):** nonop, ni, lnoi

**Always positive (revenue/metrics):** gpr, rubs, oi, eri, tei, noi, phys, econ, epu

## Rolling T-12 Window

- `t1` = most recent completed month, `t12` = 12 months ago
- All columns shift back by one when new month data is loaded
- Anchored to `DATE_TRUNC(CURRENT_DATE(), MONTH)` in the BigQuery view
- Values array in JS is ordered oldest (t12) to newest (t1)

## RUBS Aggregation

Multiple Sage line items (`rubs`, `rubs_water`, `rubs_trash`, `rubs_general`, `rubs_electric`, `rubs_gas`) are summed into a single `rubs` row during the BigQuery-to-JS injection script.

## Occupancy Metrics

- `phys` and `econ` stored as decimals (0-1), displayed as percentages
- `epu` (Eff Rent / Unit) stored as a dollar amount

## KPI Cards

Top-of-dashboard cards use: gpr, eri, noi, lnoi, phys

## Known Data Gaps

- **CTH (Cottages at Terrell Hills):** AppFolio property, not connected to Sage. Will be loaded via CSV import.
- **SS (Silver Springs):** Missing historical months prior to Jan-26.
