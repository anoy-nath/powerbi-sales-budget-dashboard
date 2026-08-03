# Brew & Co — Sales & Budget Performance Dashboard

An end-to-end Power BI project: messy multi-source data → refreshable Power Query ETL → dimensional model with two fact tables at different grains, connected through conformed Date, Region and Category dimensions → DAX time intelligence → executive dashboard.

**[▶ View the live interactive Power BI report](https://app.fabric.microsoft.com/view?r=eyJrIjoiNzViYzNhY2EtZmIzZS00ODhhLTljNjgtY2YwMTcyNzhhNjMzIiwidCI6ImJkZTczODM5LWY2OTQtNDI3MC05ZmExLTQxNTNkNzQ2OTdkZCJ9)**

![Executive summary dashboard](images/executive_summary.png)

## Project at a glance

| Area | Detail |
|---|---|
| Dataset | ~79,500 synthetic transaction rows |
| Period | January 2023 – May 2025 |
| Geography | 8 stores across QLD, NSW and VIC |
| Fact tables | Daily sales (store × product) and monthly budgets (region × category) |
| Tools | Power BI, Power Query (M), DAX |
| Key result | 97.2% budget attainment; VIC drove the entire group shortfall |

**Explore:** [Live dashboard](YOUR-PUBLISH-TO-WEB-LINK-HERE) · [DAX measures](docs/dax_measures.md) · [Build notes](docs/build_notes.md) · [Data dictionary](docs/data_dictionary.md) · [Download the PBIX](BrewAndCo.pbix)

---

## The business question

A fictional Australian café & retail group (8 stores across QLD, NSW and VIC) tracks actual sales against monthly budget. Finance was reconciling the two manually in Excel each month. This project turns that reconciliation into a self-service analytics product.

## Key findings

The group finished at **97.2% of budget** — $776,128 actual vs $798,200 target (**−$22,072 / −2.8%**).

| Region | Actual | Budget | Variance | Attainment |
|---|---|---|---|---|
| QLD | $375,377 | $365,000 | +$10,377 | **103%** |
| NSW | $214,867 | $220,300 | −$5,433 | 98% |
| VIC | $185,884 | $212,900 | −$27,016 | **87%** |

**Excluding VIC, the group beat budget by $4,944.** One region accounted for more than the entire shortfall — while like-for-like sales grew **+5.1% YoY** (Jan–May 2025 vs 2024). The result shifts management attention toward VIC and raises a testable question: does the gap reflect local demand, operational execution, product mix, or an unrealistic budget baseline? Each of those has a different remedy — and the dashboard's job is to make sure the right question gets asked.

## The data problem

The source extracts were deliberately messy, mirroring real operational systems:

- **~79,500 transaction rows** over 2.5 years with dates stored as text in *mixed formats* (`dd/mm/yyyy` and ISO `yyyy-mm-dd` in the same column)
- Duplicate rows, blank product references, refunds recorded as negative quantities
- Store regions spelled seven ways: `QLD`, `Qld`, `Queensland`, `NSW`, `N.S.W.`, `VIC`, `Vic`
- Budget held at a **different grain** than sales (monthly × region vs daily × store) with **different category names** (`Drinks` vs `Beverages`, `Merchandise` vs `Retail`)

## The solution

![Data model — dimensional model with conformed dimensions](images/model_view.png)

**Power Query (M):** locale-aware date parsing with `try ... otherwise` fallback for mixed formats, deduplication after validating the row-level grain, text standardisation, conditional-column mapping to a canonical category set. Fully refreshable — zero manual steps.

**Data model:** a star-schema-style dimensional model with two fact tables (`Fact_Sales`, `Fact_Budget`) at different grains, integrated through **conformed dimensions** (`Dim_Region`, `Dim_Category`) so a single slicer filters both facts correctly, with deliberate snowflaking on the sales side (Region → Stores → Sales; Category → Products → Sales). Dedicated marked `Dim_Date` table.

**DAX:** measure layer covering Variance / Variance %, `SAMEPERIODLASTYEAR` YoY, MoM, calendar YTD, **Australian fiscal YTD (July–June)**, and a like-for-like YTD-vs-prior-YTD comparison using variables. All measures in a dedicated `_Measures` table. See [docs/dax_measures.md](docs/dax_measures.md).

**Report:** executive page with KPI cards (rules-based conditional colour), actual-vs-budget trend, diverging variance chart, bookmark-driven pop-out slicer panels, and edit-interactions tuned so the regional comparison stays in context when filtered.

## Repository contents

| Path | Contents |
|---|---|
| `data/` | Source CSVs (synthetic, generated for this project) |
| `docs/data_dictionary.md` | Every table and column, with the deliberate data-quality issues catalogued |
| `docs/dax_measures.md` | All DAX measures with explanations and stated assumptions |
| `docs/build_notes.md` | How the model was built, step by step, and the design decisions |
| `theme/BrewAndCo_Theme.json` | Custom Power BI theme (semantic colour system) |
| `images/` | Dashboard and model screenshots |
| `BrewAndCo.pbix` | The Power BI Desktop file |

## Tools

Power BI Desktop · Power Query (M) · DAX · Publish to web

---

*All data in this project is synthetic. No real business data is used. Report published via Publish to web, which is appropriate here precisely because the data is synthetic — real business data would require secure sharing with row-level security instead.*
