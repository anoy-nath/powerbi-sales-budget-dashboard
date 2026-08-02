# Data dictionary

All data is synthetic, generated to simulate a realistic multi-source extract from an Australian café & retail group ("Brew & Co"), Jan 2023 – May 2025.

## fact_sales.csv (~79,500 rows)

Daily point-of-sale transactions. Grain: one row per line item.

| Column | Type (raw) | Notes |
|---|---|---|
| transaction_id | text | Unique per transaction. Prefix `T` = sale, `R` = refund. |
| date | **text** | Mostly `dd/mm/yyyy` (Australian), ~12 rows in ISO `yyyy-mm-dd`. Parsed in Power Query with a `try ... otherwise` fallback. |
| store_id | text | FK → dim_stores |
| product_id | text | FK → dim_products. ~15 rows blank (excluded in ETL). |
| quantity | whole number | Negative for refunds (kept — revenue is net of returns). |
| unit_price | decimal | ~85% blank by design. **Dropped in ETL** — price belongs to the product dimension; revenue is computed via relationship (`SUMX` + `RELATED`). |

**Deliberate quality issues:** mixed date formats · ~25 exact duplicate rows · ~15 blank product references · negative quantities (legitimate refunds, not errors).

## fact_budget.csv (348 rows)

Monthly budget targets. Grain: month × region × category — **coarser than sales**, which is the central modelling challenge.

| Column | Type (raw) | Notes |
|---|---|---|
| month | text | `Mmm-YYYY` (e.g. `Jan-2023`). Converted to first-of-month date in Power Query. |
| region | text | QLD / NSW / VIC — clean here, unlike the store table. |
| category | text | **Different naming scheme than products:** `Drinks`, `Bakery & Snacks`, `Food`, `Merchandise`. Mapped to the canonical set via conditional column. |
| budget_amount | whole number | AUD. |

## dim_stores.csv (8 rows)

| Column | Notes |
|---|---|
| store_id | PK |
| store_name | One value has stray leading/trailing spaces (trimmed in ETL). |
| state | **Seven spellings of three states:** `QLD`, `Qld`, `Queensland`, `NSW`, `N.S.W.`, `VIC`, `Vic`. Standardised to a `Region` column via conditional logic. |
| open_date | ISO date. |

## dim_products.csv (12 rows)

| Column | Notes |
|---|---|
| product_id | PK |
| product_name | — |
| category | Canonical set: `Beverages`, `Bakery`, `Food`, `Retail`. |
| cost_price / unit_price | AUD, fixed decimal (currency) in the model. |

## Tables created in the model (not in source)

| Table | How | Purpose |
|---|---|---|
| Dim_Date | DAX (`CALENDAR` + `ADDCOLUMNS`), marked as date table | Time intelligence incl. Australian FY (Jul–Jun). Month columns carry explicit sort-by columns. |
| Dim_Region | Power Query reference → distinct | Conformed dimension: filters `dim_stores` (→ sales) and `fact_budget` simultaneously. |
| Dim_Category | Power Query reference → distinct | Conformed dimension: filters `dim_products` (→ sales) and `fact_budget` simultaneously. |
| _Measures | Empty table | Home for all DAX measures. |
