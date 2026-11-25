# DAX Measures

All measures used in the FinSight report. The signs work as follows: amounts in the fact table are positive, so costs are subtracted in the profit measures.

## Date table

```dax
DimDate =
CALENDAR(
    DATE(YEAR(MIN(FactFinancialTransactions[TransactionDate])), 1, 1),
    DATE(YEAR(MAX(FactFinancialTransactions[TransactionDate])), 12, 31)
)
```

Calculated columns on DimDate:

```dax
Year = "FY" & YEAR(DimDate[Date])
Month = FORMAT(DimDate[Date], "mmm")
Month Number = MONTH(DimDate[Date])
Quarter = "FY" & YEAR(DimDate[Date]) & "Q" & QUARTER(DimDate[Date])
Year-Month = FORMAT(DimDate[Date], "yyyy-mm")
```

`Month` is sorted by `Month Number`. The table is marked as a date table.

## Base measures

```dax
Revenue = 
CALCULATE(SUM(FactFinancialTransactions[AmountNGN]),ChartOfAccounts_PL[AccountType] = "Revenue")

Cost of Sales = 
CALCULATE(SUM(FactFinancialTransactions[AmountNGN]), ChartOfAccounts_PL[AccountType] = "Cost of Sales")

Operating Expenses = 
CALCULATE(SUM(FactFinancialTransactions[AmountNGN]), ChartOfAccounts_PL[AccountType] = "Operating Expense")

Other Income = 
CALCULATE(SUM(FactFinancialTransactions[AmountNGN]), ChartOfAccounts_PL[AccountType] = "Other Income")

Other Expense = 
CALCULATE(SUM(FactFinancialTransactions[AmountNGN]), ChartOfAccounts_PL[AccountType] = "Other Expense")

Total Expenses = 
[Cost of Sales] + [Operating Expenses] + [Other Expense]
```

## Profit and margin

```dax
Gross Profit = [Revenue] - [Cost of Sales]

Operating Profit = [Gross Profit] - [Operating Expenses]

Net Profit = [Operating Profit] + [Other Income] - [Other Expense]

Gross Margin = DIVIDE([Gross Profit], [Revenue])
Operating Margin = DIVIDE([Operating Profit], [Revenue])
Net Margin = DIVIDE([Net Profit], [Revenue])
```

## Time intelligence

Prior-year measures use the same pattern for every base measure:

```dax
Revenue PY =
CALCULATE([Revenue], SAMEPERIODLASTYEAR(DimDate[Date]))

Revenue YoY =
DIVIDE([Revenue] - [Revenue PY], [Revenue PY])

Revenue Prior Month =
CALCULATE([Revenue], DATEADD(DimDate[Date], -1, MONTH))

Revenue MoM % =
DIVIDE([Revenue] - [Revenue Prior Month], [Revenue Prior Month])

Cost of Sales PY = CALCULATE([Cost of Sales], SAMEPERIODLASTYEAR(DimDate[Date]))

Gross Margin PY = CALCULATE([Gross Margin], SAMEPERIODLASTYEAR(DimDate[Date]))

Net Margin PY = CALCULATE([Net Margin], SAMEPERIODLASTYEAR(DimDate[Date]))

Operating expense PY = CALCULATE([Operating Expenses], SAMEPERIODLASTYEAR(DimDate[Date]))

Operating Margin PY = CALCULATE([Operating Margin], SAMEPERIODLASTYEAR(DimDate[Date]))

Other Expense PY = CALCULATE([Other Expense], SAMEPERIODLASTYEAR(DimDate[Date]))

Total expense PY = CALCULATE([Total Expenses], SAMEPERIODLASTYEAR(DimDate[Date]))

Net Profit PY = CALCULATE([Net Profit], SAMEPERIODLASTYEAR(DimDate[Date]))

Revenue PY = CALCULATE([Revenue], SAMEPERIODLASTYEAR(DimDate[Date]))
```

Profit measures divide by the absolute prior-year value, because both years are losses:

Other Income PY = CALCULATE([Other Income], SAMEPERIODLASTYEAR(DimDate[Date]))

Operating profit PY = CALCULATE([Operating Profit], SAMEPERIODLASTYEAR(DimDate[Date]))

```dax
Net Profit YoY % = 
VAR PY = CALCULATE([Net Profit], SAMEPERIODLASTYEAR(DimDate[Date]))
RETURN DIVIDE([Net Profit] - PY, ABS(PY))
```

Margins compare as a difference in percentage points:

```dax

Gross Margin Change = 
IF(ISBLANK([Gross Margin PY]), BLANK(), [Gross Margin] - [Gross Margin PY])
```

The same pair is repeated for operating margin and net margin.

## P&L statement

The detailed matrix uses a signed measure built on `NaturalSignMultiplier` in DimAccount, where revenue is +1 and costs are -1:

```dax
P&L Amount =
SUMX(FactFinancialTransactions, FactFinancialTransactions[AmountNGN] * RELATED(ChartOfAccounts_PL[NaturalSignMultiplier]))
```

The summary statement uses a layout table with one row per line, so profit rows can appear between the account sections:

```dax
PL Lines =
DATATABLE(
    "Line", STRING,
    "Sort", INTEGER,
    {
        {"Revenue", 1},
        {"Cost of Sales", 2},
        {"Gross Profit", 3},
        {"Operating Expenses", 4},
        {"Operating Profit", 5},
        {"Other Income", 6},
        {"Other Expense", 7},
        {"Net Profit Before Tax", 8}
    }
)

PL Statement Value =
SWITCH(
    SELECTEDVALUE('PL Lines'[Line]),
    "Revenue", [Revenue],
    "Cost of Sales", -[Cost of Sales],
    "Gross Profit", [Gross Profit],
    "Operating Expenses", -[Operating Expenses],
    "Operating Profit", [Operating Profit],
    "Other Income", [Other Income],
    "Other Expense", -[Other Expense],
    "Net Profit Before Tax", [Net Profit],
    BLANK()
)
```

## Visualization

```dax
Net Profit Change Color = 
VAR v = [Net Profit Change]
RETURN
    IF(ISBLANK(v), "#9AA0A6", IF(v >= 0, "#2E9E5B", "#D64545"))


P&L Font Color = 
IF(
    ISINSCOPE(ChartOfAccounts_PL[PLLine]),
    "#222222",
    IF([P & L Amount] < 0, "#D64545", "#2E9E5B")
)


Total Expenses Change Color = 
VAR v = [Total Expenses Change]
RETURN
    IF(ISBLANK(v), "#9AA0A6", IF(v <= 0, "#2E9E5B", "#D64545"))


P&L Font Color = 
IF(
    ISINSCOPE(ChartOfAccounts_PL[PLLine]),
    "#222222",
    IF([P & L Amount] < 0, "#D64545", "#2E9E5B")
)

```

Cost measures flip the test (`v <= 0` is green), because a rise in cost is bad news.
