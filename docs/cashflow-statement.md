# OTH Cashflow Statement — Structure & Construction Guide

_Based on the 1800 Broadway and OTH GP Fund reporting models_

---

## Overview

The Cashflow tab in each property reporting model contains two sections:
1. **Balance Sheet** (rows 6-58) — monthly snapshot of assets, liabilities, and equity
2. **Cash Flow Statement** (rows 63-113) — derived from balance sheet deltas + net income + D&A

The cashflow statement is constructed entirely from the Trial Balance data imported from Sage. No additional data sources are needed beyond what feeds the P&L and Balance Sheet.

---

## Balance Sheet Structure

### Assets

**Current Assets:**

| Tag | Line Item | Source |
|-----|-----------|--------|
| CASH | Cash (Clearing + Operating) | Sage TB — bank accounts |
| SDCASH | SD Cash | Security deposit bank account |
| UDEP | Utility Deposits | Utility company deposits held |
| AR | Accounts Receivable | Tenant AR (net of payments) |
| ADA | Allowance for Doubtful Accounts | Reserve against AR |
| AROTH | AR Other | Intercompany and other receivables |
| LINS | Lender Insurance Escrow | Monthly insurance escrow deposits |
| LOXX | Lender Other Escrow | Other lender-held reserves |
| LTAX | Lender Tax Escrow | Monthly tax escrow deposits |
| LCX | Lender Capex Escrow | Capital expenditure reserves |
| PPINS | Prepaid Insurance | Unamortized insurance premiums |
| PPOTH | Prepaid Other Expenses | Other prepaid items |

**Fixed Assets:**

| Tag | Line Item | Source |
|-----|-----------|--------|
| LAND | Land | Original purchase price allocation |
| BUILD | Building | Original building cost basis |
| CX | Capital Improvements | Cumulative capex since acquisition |
| CDC | Capitalized Deal Costs | Acquisition closing costs |
| CLC | Capitalized Loan Costs | Loan origination costs |
| ACCDEP | Accumulated Depreciation | Cumulative D&A (negative, grows each month) |

### Liabilities

**Current Liabilities:**

| Tag | Line Item | Source |
|-----|-----------|--------|
| AP | Accounts Payable | Vendor invoices owed |
| APOTH | Other Payables | Accrued expenses, intercompany payables |
| PPR | Prepaid Rent | Tenant rent received in advance |
| PTP | Property Taxes Payable | Accrued property tax liability |
| SDP | Security Deposits Payable | Tenant security deposits held |

**Long-term Liabilities:**

| Tag | Line Item | Source |
|-----|-----------|--------|
| MP | Mortgage Payable | Outstanding loan principal |

### Equity

| Tag | Line Item | Source |
|-----|-----------|--------|
| GPE | GP Equity | General partner contributed capital |
| LPE | LP Equity | Limited partner contributed capital |
| GPD | GP Distributions | Cumulative distributions to GP |
| LPD | LP Distributions | Cumulative distributions to LP |
| RE | Retained Earnings | Prior year accumulated net income |
| — | Opening Net Income | YTD net income carried forward from prior periods |
| — | Net Income | Current period net income (from P&L) |

**Check:** Total Assets = Total Liabilities + Total Equity (row 58 should be zero)

---

## Cash Flow Statement Construction

The cash flow statement uses the **indirect method**: start with Net Income, add back non-cash items, then show changes in working capital and balance sheet items.

### Starting Point
- **Beginning Cash** = prior month's Ending Cash (first month = 0 or opening balance)
- **Net Income** = from the P&L tab (same period)

### Operating Activities

**Adjustments (non-cash items):**
- **D&A (Depreciation & Amortization)** = current month's depreciation expense (always positive — adds back the non-cash expense from Net Income)

**Changes in Current Assets (Δ = current month - prior month):**
- Δ SD Cash
- Δ Utility Deposits
- Δ AR (Accounts Receivable)
- Δ ADA (Allowance for Doubtful Accounts)
- Δ AR Other
- Δ Lender Insurance Escrow
- Δ Lender Other Escrow
- Δ Lender Tax Escrow
- Δ Lender Capex Escrow
- Δ Prepaid Insurance
- Δ Prepaid Other Expenses

**Sign convention for asset changes:** An increase in an asset (e.g., AR goes up) is a cash outflow (negative). A decrease is a cash inflow (positive). Formula: `Δ = -(current_balance - prior_balance)`

**Changes in Current Liabilities (Δ = current month - prior month):**
- Δ Accounts Payable
- Δ Other Payables
- Δ Prepaid Rent
- Δ Property Taxes Payable
- Δ Security Deposits Payable

**Sign convention for liability changes:** An increase in a liability (e.g., AP goes up) is a cash inflow (positive — you owe more but haven't paid yet). A decrease is a cash outflow (negative). Formula: `Δ = current_balance - prior_balance`

**Cashflow from Operating Activities** = Net Income + D&A + sum of all current asset deltas + sum of all current liability deltas

### Investing Activities

Changes in fixed assets:
- Δ Land
- Δ Building
- Δ Capital Improvements (capex spend — typically negative)
- Δ Capitalized Deal Costs
- Δ Capitalized Loan Costs

**Note:** Accumulated Depreciation is NOT included here — it's already captured via the D&A add-back in Operating Activities.

**Sign convention:** An increase in a fixed asset is a cash outflow (negative). Formula: `Δ = -(current_balance - prior_balance)`

**Cashflow from Property Investments** = sum of all fixed asset deltas

### Financing Activities

- Δ Mortgage Payable (draws increase cash, paydowns decrease cash)
- Δ GP Fund Equity (new contributions = positive)
- Δ LP Equity (new contributions = positive)
- Δ GP Distributions (distributions paid = negative)
- Δ LP Distributions (distributions paid = negative)

**Sign convention:** Increases in liabilities/equity = cash inflow (positive). Distributions = cash outflow (negative). Formula: `Δ = current_balance - prior_balance`

**Cashflow from Financing** = sum of all financing deltas

### Summary

- **Change in Cash** = Operating + Investing + Financing
- **Ending Cash** = Beginning Cash + Change in Cash
- **Check:** Ending Cash should equal the Cash line on the Balance Sheet

### Distributable Cash Calculation

Below the main cashflow, the model calculates distributable cash:
- **Ending Cash**
- **(-) Current Liabilities** — must have enough to cover obligations
- **(+) Tax Escrow** — these are lender-held, not available for operations but offset the tax payable in current liabilities
- **(-) Operating Reserve** — minimum cash reserve (e.g., $750/unit = $172,500 for 230 units)
- **= Net Hold Back** — if positive, cash is available for distribution
- **Distributable Cash** = MAX(0, Ending Cash + Net Hold Back)

---

## Data Source

All balance sheet line items come from the **Sage Intacct Trial Balance** — the same source that feeds the P&L. Each GL account maps to a balance sheet tag (CASH, AR, AP, MP, etc.) via the COA mapping tab.

For the BQ pipeline, balance sheet accounts can be pulled from `GLACCOUNTBALANCE` using the same `--tb` flag. Balance sheet accounts (1xxxx, 2xxxx, 3xxxx ranges) will have `begin_balance` and `ending_balance` fields that correspond to the Balance Forward and Ending Balance columns in the Excel Trial Balance.

Income statement accounts (4xxxx, 5xxxx, 9xxxx ranges) use `total_debit` and `total_credit` for period activity (as used in the P&L). Balance sheet accounts use `ending_balance` for the point-in-time snapshot.

---

## Key Formulas

```
Operating CF = Net Income
             + D&A
             + Σ(prior_asset - current_asset)     [for each current asset except Cash]
             + Σ(current_liability - prior_liability)  [for each current liability]

Investing CF = Σ(prior_fixed_asset - current_fixed_asset)  [excluding Accum Dep]

Financing CF = (current_debt - prior_debt)
             + (current_equity - prior_equity)
             + (current_distributions - prior_distributions)

Change in Cash = Operating CF + Investing CF + Financing CF
Ending Cash = Beginning Cash + Change in Cash
```

---

## GP Fund Level Cashflow

At the GP Fund level (OTH GP Fund I), the balance sheet consolidates:
- **Investment Assets** — carrying value of each property (equity invested)
- **Committed Capital Receivable** — LP capital not yet called
- **Fund-level cash** — GP Fund bank accounts (Keystone Bank)
- **Fund-level AR** — AM Fee Receivable, intercompany receivables

The fund-level cashflow follows the same indirect method but the "investing" section reflects capital deployed into properties rather than property-level capex.

---

## Common Issues

1. **Check row not zero:** Usually means a GL account isn't mapped to any balance sheet tag. Check the COA tab for unmapped accounts.
2. **Cash doesn't reconcile:** Verify that all bank accounts are included in CASH and that no escrow accounts are double-counted.
3. **Large AR Other swings:** Often caused by intercompany transactions (property-to-fund or fund-to-property) that may need elimination in consolidated reporting.
4. **Negative prepaid rent:** Can happen when tenants have credit balances — legitimate, not a data error.
5. **Accumulated Depreciation mismatch:** Should equal the cumulative sum of monthly depreciation from the P&L. If it doesn't, check for manual depreciation adjustments.
