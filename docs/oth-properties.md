# OTH Capital -- Property Portfolio

## Active Properties

| Entity ID | Property Name | Units | PM Platform | Fund |
|-----------|--------------|-------|-------------|------|
| 18B | 1800 Broadway | 230 | Fortress / Sage | GP Fund II |
| ML | Muir Lake | 332 | Fortress / Sage | GP Fund I |
| AAH | Avita Alamo Heights | 312 | Fortress / Sage | GP Fund I |
| HOH | Heron on Hausman | 106 | Fortress / Sage | GP Fund I |
| SRP | Summit at Rivery Park | 228 | Fortress / Sage | GP Fund I |
| SS | Silver Springs | 360 | Fortress / Sage | GP Fund II |
| FRP | Forest Park | 228 | Fortress / Sage | OTH Impact Fund (JV w/ Redstone) |
| WSP | Windsong Place | 200 | Fortress / Sage | -- |
| LPL | Lakeline Parmer Lane | 312 | Fortress / Sage | -- |
| A36 | Apartments 36 | 38 | Fortress / Sage | -- |
| 616 | 616 W MLK | 6 | Fortress / Sage | -- |
| CTH | Cottages at Terrell Hills | -- | AppFolio | -- |

## Fund Structure

- **GP Fund I:** AAH (Avita Alamo Heights), ML (Muir Lake), SRP (Summit at Rivery Park), HOH (Heron on Hausman)
- **GP Fund II:** 18B (1800 Broadway), SS (Silver Springs)
- **OTH Impact Fund:** FRP (Forest Park) -- JV with Redstone

## PM Software

- **Fortress:** Primary property management platform for all properties except CTH
- **Sage Intacct:** Accounting system; GL data pulled via API into BigQuery
- **AppFolio:** CTH only -- not connected to BQ warehouse, will be loaded via CSV import
- **Yardi:** Used for some properties (split with Fortress)

## Offboarded / Corporate-Tracking-Only

| Entity ID | Property Name | Status | Notes |
|-----------|--------------|--------|-------|
| JOH | Johnson | Offboarded | No longer in active portfolio |
| CPZ | Copper Zone | Offboarded | No longer in active portfolio |
| ELT | Elevation | Offboarded | No longer in active portfolio |

These entities may still appear in Sage GL data but should be excluded from property-level reporting.

## Data Notes

- **CTH:** P&L managed in AppFolio. Has partial/stale data in BQ from earlier manual loads. Full data pending CSV import. Unit count TBD.
- **SS:** Only has Jan-26 and Feb-26 data in the warehouse. Historical months prior to that are missing.
- **Unit counts** are stored in `oth_silver.dim_property` and also hardcoded in the dashboard injection script as the UNITS dict.
