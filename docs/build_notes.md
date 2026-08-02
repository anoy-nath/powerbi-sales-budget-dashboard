# Build notes & design decisions

A record of how the model was built and *why* — the decisions are the portfolio.

## 1. ETL (Power Query)

**Mixed date formats.** The sales `date` column was text, mostly `dd/mm/yyyy` with a few ISO rows. Power BI's automatic "Changed Type" step guesses with a US locale and silently corrupts d/m/y data, so it was deleted and replaced with an explicit parse:

```m
try Date.FromText([date], [Format="dd/MM/yyyy", Culture="en-AU"])
otherwise Date.FromText([date], [Format="yyyy-MM-dd", Culture="en-AU"])
```

**Deduplication** on `transaction_id` (a transaction ID is unique by definition — removing duplicates on it cannot delete legitimate rows).

**Blank product IDs** filtered out (unmappable to any product; ~15 rows).

**Refunds kept.** Negative quantities are real business events, not errors. Revenue is therefore net of returns. Knowing the difference between a data error (remove) and a legitimate negative (keep) matters.

**`unit_price` dropped from the fact table.** The Excel instinct is to VLOOKUP/merge price onto every sales row. Resisted: price belongs to the product dimension, and the model fetches it per-row via `RELATED` at measure time. Facts stay lean; price is maintained in one place.

**Region standardisation.** Seven spellings of three states collapsed with a three-rule conditional column (`contains "Q"` → QLD, etc.) after inspecting distinct values — minimal rules beat seven hard-coded branches.

**Budget month** (`Jan-2023` text) converted by prefixing `01-` and parsing `dd-MMM-yyyy`.

**Budget categories** mapped to the canonical product set (`Drinks`→`Beverages`, `Bakery & Snacks`→`Bakery`, `Merchandise`→`Retail`) via conditional column — reconciling naming *before* modelling.

## 2. Model

**Star schema, two facts.** `Fact_Sales` (daily × store × product) and `Fact_Budget` (monthly × region × category) cannot be joined directly — different grains, ambiguous relationship, double-counting risk.

**Conformed dimensions** solve it:
- `Dim_Region` → filters `Dim_Stores` (which filters sales) *and* `Fact_Budget` directly
- `Dim_Category` → filters `Dim_Products` (which filters sales) *and* `Fact_Budget` directly
- `Dim_Date` → filters both facts (daily for sales, first-of-month for budget)

One slicer therefore filters both facts coherently, which is what makes the variance comparison valid. The one-hop-removed shape (Region → Stores → Sales) is a deliberate snowflake.

**All relationships one-to-many, single direction** (dimension filters fact, never the reverse). No bidirectional filtering anywhere — it wasn't needed, and avoiding it prevents ambiguity.

**Dedicated Date table**, marked, with explicit sort columns for month labels.

**`_Measures` table** as the single home for all measures.

## 3. Report

- **Semantic colour system** (custom theme): one hero colour for actuals, muted reference colour for budget, red/green reserved exclusively for variance. Colour encodes meaning, never decoration.
- **Answer-first layout:** KPI cards deliver the verdict in seconds; charts below explain why; the matrix provides detail.
- **Rules-based conditional formatting** on variance cards and matrix (fx → Rules → <0 red, ≥0 green).
- **Bookmark-driven slicer panels:** grouped panel (shape + slicer + close button) toggled by paired bookmarks with **Data unticked** — so opening the panel never resets the user's selection.
- **Edit interactions:** the Variance-by-Region chart is excluded from the Region slicer so it always shows all three regions for context while the rest of the page filters.
- **Page-level date filter** (≤ 31 May 2025) trims empty future months from the calendar.

## 4. Publishing

Published to the Power BI Service (My Workspace) and shared via **Publish to web** — appropriate only because the data is synthetic. Real business data would require secure sharing with row-level security; publish-to-web has neither authentication nor RLS.

## 5. What I'd do next

- Row-level security so each regional manager sees only their stores
- Source data in SQL with incremental refresh rather than CSVs
- Forecast measures — shift the question from "did we hit budget" to "will we"
- Drill-through to store level, so clicking VIC reveals which stores drive the miss
