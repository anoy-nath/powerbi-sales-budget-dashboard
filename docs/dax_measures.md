# DAX measures

All measures live in the `_Measures` table.

## Core

```dax
Total Sales =
SUMX (
    Fact_Sales,
    Fact_Sales[quantity] * RELATED ( Dim_Products[unit_price] )
)
```
Row-by-row quantity × price, with price fetched from the product dimension via the relationship (`RELATED`) rather than duplicated onto the fact table. Refunds carry negative quantities, so this is automatically **net revenue**.

> **Stated assumption:** this project assumes a single fixed unit price per product across the reporting period, which holds for this synthetic dataset. In a production model with historical price changes, the transaction price would be stored in the fact table at point of sale, or resolved through an effective-dated price table.

```dax
Total Budget = SUM ( Fact_Budget[Budget_Amount] )
```

```dax
Variance = [Total Sales] - [Total Budget]
```

```dax
Variance % = DIVIDE ( [Variance], [Total Budget] )
```
`DIVIDE` returns blank instead of an error on divide-by-zero.

## Time intelligence

All of these depend on `Dim_Date` being a contiguous calendar **marked as a date table**.

```dax
Sales LY = CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( Dim_Date[Date] ) )
```

```dax
Sales YoY % = DIVIDE ( [Total Sales] - [Sales LY], [Sales LY] )
```

```dax
Sales YTD = TOTALYTD ( [Total Sales], Dim_Date[Date] )
```

```dax
Sales PM = CALCULATE ( [Total Sales], PREVIOUSMONTH ( Dim_Date[Date] ) )
```

```dax
Sales MoM % = DIVIDE ( [Total Sales] - [Sales PM], [Sales PM] )
```

```dax
Sales FYTD = TOTALYTD ( [Total Sales], Dim_Date[Date], "30/6" )
```
The third argument sets the year-end to 30 June — **Australian financial year**. FYTD resets every July.

## Like-for-like YoY (used on the KPI card)

A plain `SAMEPERIODLASTYEAR` on a card with no date context produces a meaningless grand-total ratio. This measure anchors to the last date with sales and compares matching Jan–May windows:

```dax
Sales YoY % (YTD) =
VAR LastSaleDate = MAX ( Fact_Sales[Date] )
VAR CurrentYTD =
    CALCULATE (
        [Total Sales],
        DATESBETWEEN ( Dim_Date[Date], DATE ( YEAR ( LastSaleDate ), 1, 1 ), LastSaleDate )
    )
VAR PriorYTD =
    CALCULATE (
        [Total Sales],
        DATESBETWEEN (
            Dim_Date[Date],
            DATE ( YEAR ( LastSaleDate ) - 1, 1, 1 ),
            EDATE ( LastSaleDate, -12 )
        )
    )
RETURN
    DIVIDE ( CurrentYTD - PriorYTD, PriorYTD )
```

`MAX(Fact_Sales[Date])` finds where data actually ends (31 May 2025); `DATESBETWEEN` builds two matching year-to-date windows; `EDATE(..., -12)` shifts back exactly twelve months. Result: **+5.1%** — Jan–May 2025 ($141,606) vs Jan–May 2024 ($134,758).

## Date table

```dax
Dim_Date =
ADDCOLUMNS (
    CALENDAR ( DATE ( 2023, 1, 1 ), DATE ( 2025, 12, 31 ) ),
    "Year", YEAR ( [Date] ),
    "Month Number", MONTH ( [Date] ),
    "Month", FORMAT ( [Date], "mmm" ),
    "Month Year", FORMAT ( [Date], "mmm yyyy" ),
    "Month Year Sort", YEAR ( [Date] ) * 100 + MONTH ( [Date] ),
    "Quarter", "Q" & QUARTER ( [Date] )
)
```
Explicit range rather than `CALENDARAUTO()` (which would scan store `open_date` back to 2018). `Month` sorts by `Month Number`; `Month Year` sorts by `Month Year Sort`.
