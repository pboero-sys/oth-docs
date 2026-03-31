---
name: OTH P&L Dashboard
description: Technical reference for the OTH Capital trailing-12 P&L dashboard (oth-pnl-v5.html) — data format, sign conventions, KEY_MAP, KPIs, and known gaps
type: reference
---

# OTH P&L Dashboard — oth-pnl-v5.html

**Location:** `~/Downloads/oth-pnl-v5.html`

## DATA Structure

The `const DATA` JS object is keyed by property_id. Each property has:
- `name` — display name
- `units` — unit count (null for CTH)
- `real` — always true (legacy flag from when some properties had estimated data)
- `periods` — array of 12 strings, oldest to newest: `["Mar-25", ..., "Feb-26"]`
- `pnl` — array of rows: `[short_key, label, sort_order, section, ...12 values]`
- `kpi` — currently empty array (KPIs computed client-side)

## KEY_MAP: BigQuery line_item -> JS short key

```
gross_potential_rent    -> gpr    Gross Potential Rent
loss_to_lease           -> ltl    (-) Loss to Lease
vacancy_loss            -> vac    (-) Vacancy Loss
concessions             -> conc   (-) Concessions
non_revenue_units       -> nr     (-) Non Revenue Units
bad_debt                -> bad    (-) Bad Debt
effective_rental_income -> eri    Effective Rental Income       (subtotal)
physical_occupancy_pct  -> phys   Physical Occupancy            (metric, 0-1)
economic_occupancy_pct  -> econ   Economic Occupancy            (metric, 0-1)
effective_rent_pu       -> epu    Eff Rent / Unit               (metric, dollar)
rubs / rubs_*           -> rubs   RUBS                          (aggregated)
other_income            -> oi     Other Income
total_effective_income  -> tei    Total Effective Income        (subtotal)
utilities_water         -> utilw  Utilities - Water
utilities_electric      -> utile  Utilities - Electric
utilities_gas           -> utilg  Utilities - Gas
utilities_trash         -> utilt  Utilities - Trash
utilities_general       -> utilx  Utilities - General
utilities               -> util   Utilities
marketing               -> mkt    Marketing
admin                   -> adm    Admin
contract_services       -> cont   Contract Services
maintenance             -> mtc    Maintenance
turnover_expense        -> tox    Turnover Expense
payroll                 -> pay    Payroll
management_fee          -> pm     Management Fee
controllable_expenses   -> ctrl   Controllable Expenses         (subtotal)
property_taxes          -> tax    Property Taxes
insurance               -> ins    Insurance
uncontrollable_expenses -> uctrl  Uncontrollable Expenses       (subtotal)
unlevered_noi           -> noi    Unlevered NOI                 (subtotal)
interest_payment        -> ipmt   Interest Payment
principal_payment       -> ppmt   Principal Payment
debt_service            -> ds     Debt Service
levered_noi             -> lnoi   Levered NOI                   (subtotal)
am_fee                  -> amfee  AM Fee
non_operating_income    -> nonop  Non Operating Income
depreciation            -> dep    Depreciation
net_income              -> ni     Net Income                    (subtotal)
debt_balance            -> debt   Debt Balance
equity_balance          -> eq     Equity Balance
total_capital_basis     -> basis  Total Capital Basis
```

## Sign Convention

Expenses come from Sage as positive numbers but must **display as negative** in the dashboard.

**Always negative (expense/deduction lines):**
ltl, vac, conc, nr, bad, utilw, utile, utilg, utilt, utilx, util, mkt, adm, cont, mtc, tox, pay, pm, ctrl, tax, ins, uctrl, ipmt, ppmt, ds, amfee, dep

**Can be positive or negative (net lines):**
nonop, ni, lnoi

**Always positive (revenue/metric lines):**
gpr, rubs, oi, eri, tei, noi, phys, econ, epu

## KPI Cards

Top-of-dashboard KPI cards use these keys:
- `gpr` — Gross Potential Rent
- `eri` — Effective Rental Income
- `noi` — Unlevered NOI
- `lnoi` — Levered NOI
- `phys` — Physical Occupancy

## Rolling Window

- t1 = most recent completed month, t12 = 12 months ago
- All columns shift back by one when new month data is loaded
- Anchored to `DATE_TRUNC(CURRENT_DATE(), MONTH)` in the BQ view
- Values array in JS is ordered oldest (t12) to newest (t1)

## RUBS Aggregation

Multiple Sage line items (rubs, rubs_water, rubs_trash, rubs_general, rubs_electric, rubs_gas) are summed into a single `rubs` row during the BQ -> JS injection.

## Occupancy Metrics

- `phys` and `econ` are stored as decimals (0-1), displayed as percentages
- `epu` (Eff Rent / Unit) is stored as a dollar amount

## Known Gaps

- **CTH (Cottages at Terrell Hills):** AppFolio property, not connected to Sage. Will be loaded via CSV import. Currently shows partial/placeholder data.
- **SS (Silver Springs):** Missing historical months prior to Jan-26. Only Jan-26 and Feb-26 populated.

## Injection Script

The dashboard DATA block is injected by a Python script that:
1. Queries `oth_gold.v_property_pnl` for all 12 properties
2. Maps line_items via KEY_MAP to short keys
3. Aggregates RUBS sub-lines
4. Replaces `const DATA = {...}` block up to `const SUBS=` marker
