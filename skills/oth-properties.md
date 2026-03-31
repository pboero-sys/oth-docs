---
name: OTH Property Portfolio
description: Full property list with entity IDs, unit counts, PM platform, fund membership, and offboarded properties
type: reference
---

# OTH Capital — Property Portfolio

## Active Properties

| Entity ID | Property Name | Units | PM Platform | Fund |
|-----------|--------------|-------|-------------|------|
| 18B | 1800 Broadway | 230 | Fortress | GP Fund II |
| ML | Muir Lake | 332 | Fortress | GP Fund I |
| AAH | Avita Alamo Heights | 312 | Fortress | GP Fund I |
| HOH | Heron on Hausman | 106 | Fortress | GP Fund I |
| SRP | Summit at Rivery Park | 228 | Fortress | GP Fund I |
| SS | Silver Springs | 360 | Fortress | GP Fund II |
| FRP | Forest Park | 228 | Fortress | OTH Impact Fund (JV with Redstone) |
| WSP | Windsong Place | 200 | Fortress | (check) |
| LPL | Lakeline Parmer Lane | 312 | Fortress | (check) |
| A36 | Apartments 36 | 38 | Fortress | (check) |
| 616 | 616 W MLK | 6 | Fortress | (check) |
| CTH | Cottages at Terrell Hills | N/A | AppFolio | (check) |

## Fund Structure

- **GP Fund I:** AAH, Muir Lake, Rivery Park (SRP), Heron on Hausman (HOH)
- **GP Fund II:** River House (1800 Broadway / 18B), Silver Springs (SS)
- **OTH Impact Fund:** Forest Park (FRP) / Redstone JV

## PM Software

- **Fortress:** Primary PM platform for all properties except CTH
- **AppFolio:** CTH (Cottages at Terrell Hills) only — not yet connected to BQ warehouse, will be loaded via CSV import
- **Sage Intacct:** Accounting system for all Fortress properties. GL data pulled via API into BigQuery.
- **Yardi:** Used for some properties (split with Fortress)

## Offboarded / Sold Properties

| Entity ID | Property Name | Status | Notes |
|-----------|--------------|--------|-------|
| JOH | (Johnson) | Offboarded | No longer in active portfolio |
| CPZ | (Copper Zone) | Offboarded | No longer in active portfolio |
| ELT | (Elevation) | Offboarded | No longer in active portfolio |

## Data Notes

- **SS (Silver Springs):** Only has Jan-26 and Feb-26 data in BQ warehouse. Historical months missing — may need Sage backfill.
- **CTH (Cottages at Terrell Hills):** P&L managed in AppFolio. Has partial/stale data in BQ from earlier loads. Full data pending CSV import.
- **Unit counts** are stored in `oth_silver.dim_property` and also hardcoded in the dashboard injection script as the UNITS dict.
