# Cielo Azul — KPI Master Power Query Setup
**Complete Instructions | 2201 Polk St, Hollywood FL**

---

## Before You Start

### 1. Create the OneDrive folder structure

Create these folders exactly as shown:

```
/Sunexus Group/
├── 01 - Operations/
│   └── Cielo Azul - Hollywood/
│       ├── Airbnb Exports/              ← drop Airbnb CSV files here
│       └── DoorLoop Exports/
│           ├── RentRoll/                ← drop Rent Roll exports here
│           ├── ProfitLoss/              ← drop P&L exports here
│           ├── OwnerStatement/          ← drop Owner Statement exports here
│           └── BalanceSheet/            ← drop Balance Sheet exports here
└── 04 - Data and Reporting/
    └── Master Datasets/
        └── KPI_Master.xlsx              ← create this blank file now
```

### 2. Name every exported file consistently

Use this naming convention every month — Power Query uses the filename to extract the report month:

```
Airbnb:          2026-03_CieloAzul_Airbnb.csv
Rent Roll:       2026-05_PollkSt_RentRoll.xlsx
Profit & Loss:   2026-05_PollkSt_ProfitLoss.xlsx
Owner Statement: 2026-03_PollkSt_OwnerStatement.xlsx
Balance Sheet:   2026-05_PollkSt_BalanceSheet.xlsx
```

### 3. Confirm listing-to-unit mapping with Carolina

Before running the Airbnb query, confirm this mapping. The listing names come from the raw CSV. The unit numbers come from the Rent Roll:

| Airbnb Listing Name | Unit | Notes |
|---|---|---|
| Modern Hollywood Escape | 204 | **Confirm with Carolina** |
| Modern Hollywood Retreat | 205 | **Confirm with Carolina** |
| Modern Hollywood Downtown Retreat | 202 | **Confirm with Carolina** |
| Hollywood Downtown Escape | 203 | **Confirm with Carolina** |
| Hollywood Oasis | 206 | **Confirm with Carolina** |
| 3 Bedroom PH Spacious Modern Hollywood Stay | 401 | **Confirm with Carolina** |

> Note: "Modern Hollywood Downtwon Retreat" is a typo in Airbnb for "Modern Hollywood Downtown Retreat". Fix the listing name in Airbnb directly to avoid future issues.

---

## Open KPI_Master.xlsx

Open the blank `KPI_Master.xlsx` file you created. All five queries below get built inside this one file.

---

## Query 1 — Airbnb STR Reservations

**Step 1** — Go to `Data → Get Data → From File → From Folder`

**Step 2** — Point at your `/Airbnb Exports/` folder and click **Transform Data**

**Step 3** — Click **Advanced Editor** and replace everything with:

```
let
    // Read all CSV files from the Airbnb Exports folder
    Source = Folder.Files("REPLACE_WITH_YOUR_ONEDRIVE_PATH\Airbnb Exports"),
    
    // Keep CSV files only
    CSVOnly = Table.SelectRows(Source, each [Extension] = ".csv"),
    
    // Extract report month from filename (e.g. "2026-03" from "2026-03_CieloAzul_Airbnb.csv")
    AddMonth = Table.AddColumn(CSVOnly, "ReportMonth",
        each Text.Start([Name], 7), type text),

    // Read each file
    ReadFiles = Table.AddColumn(AddMonth, "FileData",
        each Csv.Document(
            File.Contents([Folder Path] & [Name]),
            [Delimiter=",", Encoding=65001, QuoteStyle=QuoteStyle.None])),

    // Expand all files into one table
    Expanded = Table.ExpandTableColumn(ReadFiles, "FileData",
        List.Transform({1..21}, each "Column" & Text.From(_))),

    // Remove duplicate header rows that appear when combining multiple files
    RemoveDupHeaders = Table.SelectRows(Expanded,
        each [Column1] <> "Date"),

    // Use first row as headers
    Promoted = Table.PromoteHeaders(Expanded, [PromoteAllScalars=true]),

    // Remove any rows where Type column is "Type" (duplicate headers)
    CleanHeaders = Table.SelectRows(Promoted,
        each [Type] <> "Type" and [Date] <> "Date"),

    // Keep needed columns only
    SelectCols = Table.SelectColumns(CleanHeaders, {
        "ReportMonth", "Date", "Type", "Confirmation code",
        "Booking date", "Start date", "End date", "Nights",
        "Guest", "Listing", "Amount", "Service fee",
        "Cleaning fee", "Gross earnings", "Airbnb remitted tax"}),

    // Keep Reservations only — drop Payout, Co-Host payout, Resolution Adjustment
    ResOnly = Table.SelectRows(SelectCols,
        each [Type] = "Reservation"),

    // Fix date types
    FixDates = Table.TransformColumnTypes(ResOnly, {
        {"Date", type date},
        {"Start date", type date},
        {"End date", type date},
        {"Booking date", type date}}),

    // Fix number types
    FixNums = Table.TransformColumnTypes(FixDates, {
        {"Nights", Int64.Type},
        {"Amount", Currency.Type},
        {"Service fee", Currency.Type},
        {"Cleaning fee", Currency.Type},
        {"Gross earnings", Currency.Type},
        {"Airbnb remitted tax", Currency.Type}}),

    // Fix typo in listing name — "Downtwon" → "Downtown"
    FixTypo = Table.ReplaceValue(FixNums,
        "Modern Hollywood Downtwon Retreat",
        "Modern Hollywood Downtown Retreat",
        Replacer.ReplaceText, {"Listing"}),

    // Map listing name to unit number
    // UPDATE THESE MAPPINGS AFTER CONFIRMING WITH CAROLINA
    AddUnit = Table.AddColumn(FixTypo, "Unit", each
        if   [Listing] = "Modern Hollywood Escape"                        then "204"
        else if [Listing] = "Modern Hollywood Retreat"                    then "205"
        else if [Listing] = "Modern Hollywood Downtown Retreat"           then "202"
        else if [Listing] = "Hollywood Downtown Escape"                   then "203"
        else if [Listing] = "Hollywood Oasis"                             then "206"
        else if [Listing] = "3 Bedroom PH Spacious Modern Hollywood Stay" then "401"
        else "Unknown", type text),

    // Net revenue = gross earnings minus Airbnb-remitted tax
    AddNet = Table.AddColumn(AddUnit, "Net Revenue",
        each [Gross earnings] - [Airbnb remitted tax], Currency.Type),

    // Flag rows where unit mapping is unknown — review with Carolina
    AddFlag = Table.AddColumn(AddNet, "Needs Review",
        each [Unit] = "Unknown", type logical),

    // Add property identifier — useful when second building is added
    AddProperty = Table.AddColumn(AddFlag, "Property",
        each "2201 Polk St", type text),

    // Add rental type
    AddType = Table.AddColumn(AddProperty, "Rental Type",
        each "STR", type text)

in AddType
```

**Step 4** — Click **Close and Load To** → Table → name the sheet `STR_Reservations`

---

## Query 2 — Rent Roll

**Step 1** — Go to `Data → Get Data → From File → From Folder`

**Step 2** — Point at `/DoorLoop Exports/RentRoll/` and click **Transform Data**

**Step 3** — Open **Advanced Editor** and paste:

```
let
    Source = Folder.Files("REPLACE_WITH_YOUR_ONEDRIVE_PATH\DoorLoop Exports\RentRoll"),
    XLSXOnly = Table.SelectRows(Source, each [Extension] = ".xlsx"),

    // Read each Excel file
    ReadFiles = Table.AddColumn(XLSXOnly, "FileData",
        each Excel.Workbook(File.Contents([Folder Path] & [Name]), null, true)),

    // Get Sheet1 from each workbook
    GetSheet = Table.AddColumn(ReadFiles, "Sheet",
        each [FileData]{[Item="Sheet1", Kind="Sheet"]}[Data]),

    // Expand into one table
    Expanded = Table.ExpandTableColumn(GetSheet, "Sheet",
        {"Column1","Column2","Column3","Column4","Column5",
         "Column6","Column7","Column8","Column9","Column10","Column11","Column12"}),

    // Row 2 = title, Row 4 = headers, Row 5 = property header, Rows 6-27 = data
    // Row 28-29 = totals — skip first 4 rows and use row 4 as headers
    RemoveTop = Table.Skip(Expanded, 3),
    Promoted = Table.PromoteHeaders(RemoveTop, [PromoteAllScalars=true]),

    // Remove property header row ("property: 2201 Polk St")
    RemovePropRow = Table.SelectRows(Promoted,
        each not Text.StartsWith(Text.From([Unit] ?? ""), "property:")),

    // Remove summary rows at the bottom
    RemoveTotals = Table.SelectRows(RemovePropRow,
        each [Unit] <> "22 Units" and [Unit] <> "Total" and [Unit] <> null),

    // Remove timestamp footer
    RemoveFooter = Table.SelectRows(RemoveTotals,
        each not Text.StartsWith(Text.From([Unit] ?? ""), "Cash basis")),

    // Keep relevant columns
    SelectCols = Table.SelectColumns(RemoveFooter, {
        "Unit", "Lease", "Start date", "End date",
        "Beds / Baths", "Size (sq. ft.)", "Rent",
        "Deposits", "Balance"}),

    // Fix types
    FixTypes = Table.TransformColumnTypes(SelectCols, {
        {"Unit", type text},
        {"Start date", type date},
        {"End date", type date},
        {"Rent", Currency.Type},
        {"Deposits", Currency.Type},
        {"Balance", Currency.Type}}),

    // Replace nulls in numeric columns with 0
    FillNulls = Table.ReplaceValue(
        Table.ReplaceValue(
            Table.ReplaceValue(FixTypes, null, 0, Replacer.ReplaceValue, {"Rent"}),
        null, 0, Replacer.ReplaceValue, {"Deposits"}),
    null, 0, Replacer.ReplaceValue, {"Balance"}),

    // Classify each unit
    AddUnitType = Table.AddColumn(FillNulls, "Unit Type", each
        if [Lease] = "VACANT"             then "Vacant"
        else if [Lease] = "SHORT TERM RENTALS" then "STR"
        else if [Unit] = "Social Room "   then "Commercial"
        else "LTR", type text),

    // Flag occupied vs vacant for occupancy calculation
    AddOccupied = Table.AddColumn(AddUnitType, "Occupied",
        each if [Unit Type] = "Vacant" then 0 else 1, Int64.Type),

    // Add tenant category
    AddTenant = Table.AddColumn(AddOccupied, "Tenant Category", each
        if   Text.Contains([Lease] ?? "", "Stella Ventures") then "Corporate - Stella Ventures"
        else if Text.Contains([Lease] ?? "", "Edinson Porras") then "Sublease - Edinson Porras"
        else if Text.Contains([Lease] ?? "", "Astrid Living")  then "Sublease - Astrid Living"
        else if Text.Contains([Lease] ?? "", "SHORT TERM")     then "STR Pool"
        else if [Lease] = "VACANT"                             then "Vacant"
        else if [Unit] = "Social Room "                        then "Commercial"
        else "Individual", type text),

    AddProperty = Table.AddColumn(AddTenant, "Property",
        each "2201 Polk St", type text)

in AddProperty
```

**Step 4** — Click **Close and Load To** → Table → sheet named `RentRoll`

---

## Query 3 — Profit and Loss

**Step 1** — Go to `Data → Get Data → From File → From Folder`

**Step 2** — Point at `/DoorLoop Exports/ProfitLoss/` and click **Transform Data**

**Step 3** — Open **Advanced Editor** and paste:

```
let
    Source = Folder.Files("REPLACE_WITH_YOUR_ONEDRIVE_PATH\DoorLoop Exports\ProfitLoss"),
    XLSXOnly = Table.SelectRows(Source, each [Extension] = ".xlsx"),

    // Extract report period from filename
    AddMonth = Table.AddColumn(XLSXOnly, "ReportMonth",
        each Text.Start([Name], 7), type text),

    ReadFiles = Table.AddColumn(AddMonth, "FileData",
        each Excel.Workbook(File.Contents([Folder Path] & [Name]), null, true)),

    GetSheet = Table.AddColumn(ReadFiles, "Sheet",
        each [FileData]{[Item="Sheet1", Kind="Sheet"]}[Data]),

    Expanded = Table.ExpandTableColumn(GetSheet, "Sheet",
        {"Column1","Column2","Column3","Column4","Column5"}),

    // P&L structure: Row 2 = title, Row 4 = headers, Rows 5-18 = data
    // Account is in col 1-4 by indent level, Total in col 5
    // Consolidate account name from first non-null column
    AddAccount = Table.AddColumn(Expanded, "Account", each
        if [Column1] <> null then Text.Trim(Text.From([Column1]))
        else if [Column2] <> null then Text.Trim(Text.From([Column2]))
        else if [Column3] <> null then Text.Trim(Text.From([Column3]))
        else Text.Trim(Text.From([Column4] ?? "")), type text),

    AddTotal = Table.AddColumn(AddAccount, "Total",
        each [Column5], type number),

    SelectCols = Table.SelectColumns(AddTotal,
        {"ReportMonth", "Account", "Total"}),

    // Remove title, header, blank and footer rows
    CleanRows = Table.SelectRows(SelectCols, each
        [Account] <> null
        and [Account] <> ""
        and [Account] <> "Account"
        and not Text.StartsWith([Account], "Profit and loss")
        and not Text.StartsWith([Account], "Cash basis")),

    // Keep only the key rows needed for KPI calculations
    KeyRows = Table.SelectRows(CleanRows, each
        List.Contains({
            "04.01Rent Apartments",
            "04.11 Utilities",
            "04.12 Late Fees",
            "04.19 Taxes",
            "Total Income",
            "Expenses",
            "Net operating income",
            "Net income"
        }, [Account])),

    AddProperty = Table.AddColumn(KeyRows, "Property",
        each "2201 Polk St", type text)

in AddProperty
```

**Step 4** — Click **Close and Load To** → Table → sheet named `ProfitLoss`

---

## Query 4 — Owner Statement

**Step 1** — Go to `Data → Get Data → From File → From Folder`

**Step 2** — Point at `/DoorLoop Exports/OwnerStatement/` and click **Transform Data**

**Step 3** — Open **Advanced Editor** and paste:

```
let
    Source = Folder.Files("REPLACE_WITH_YOUR_ONEDRIVE_PATH\DoorLoop Exports\OwnerStatement"),
    XLSXOnly = Table.SelectRows(Source, each [Extension] = ".xlsx"),

    AddMonth = Table.AddColumn(XLSXOnly, "ReportMonth",
        each Text.Start([Name], 7), type text),

    ReadFiles = Table.AddColumn(AddMonth, "FileData",
        each Excel.Workbook(File.Contents([Folder Path] & [Name]), null, true)),

    GetSheet = Table.AddColumn(ReadFiles, "Sheet",
        each [FileData]{[Item="Sheet1", Kind="Sheet"]}[Data]),

    // Owner Statement has 8 columns: Account spread across cols 1-6, Property col, Total col
    Expanded = Table.ExpandTableColumn(GetSheet, "Sheet",
        {"Column1","Column2","Column3","Column4","Column5","Column6","Column7","Column8"}),

    // Consolidate account name
    AddAccount = Table.AddColumn(Expanded, "Account", each
        if [Column1] <> null then Text.Trim(Text.From([Column1]))
        else if [Column2] <> null then Text.Trim(Text.From([Column2]))
        else if [Column3] <> null then Text.Trim(Text.From([Column3]))
        else if [Column4] <> null then Text.Trim(Text.From([Column4]))
        else if [Column5] <> null then Text.Trim(Text.From([Column5]))
        else Text.Trim(Text.From([Column6] ?? "")), type text),

    // Column7 = property value, Column8 = total
    AddPropertyVal = Table.AddColumn(AddAccount, "PropertyValue",
        each [Column7], type number),

    AddTotal = Table.AddColumn(AddPropertyVal, "Total",
        each [Column8], type number),

    SelectCols = Table.SelectColumns(AddTotal,
        {"ReportMonth", "Account", "PropertyValue", "Total"}),

    CleanRows = Table.SelectRows(SelectCols, each
        [Account] <> null
        and [Account] <> ""
        and [Account] <> "Account"
        and not Text.StartsWith([Account], "Statement summary")
        and not Text.StartsWith([Account], "Cash basis")),

    // Key rows for owner reporting
    KeyRows = Table.SelectRows(CleanRows, each
        List.Contains({
            "Cash at beginning of period",
            "04.01Rent Apartments",
            "04.11 Utilities",
            "Total Income",
            "Total Net income",
            "Net cash increase for period",
            "Cash at end of period",
            "Current liabilities",
            "Undeposited Funds",
            "Cash available"
        }, [Account])),

    AddProperty = Table.AddColumn(KeyRows, "Property",
        each "2201 Polk St", type text)

in AddProperty
```

**Step 4** — Click **Close and Load To** → Table → sheet named `OwnerStatement`

---

## Query 5 — Balance Sheet

**Step 1** — Go to `Data → Get Data → From File → From Folder`

**Step 2** — Point at `/DoorLoop Exports/BalanceSheet/` and click **Transform Data**

**Step 3** — Open **Advanced Editor** and paste:

```
let
    Source = Folder.Files("REPLACE_WITH_YOUR_ONEDRIVE_PATH\DoorLoop Exports\BalanceSheet"),
    XLSXOnly = Table.SelectRows(Source, each [Extension] = ".xlsx"),

    AddMonth = Table.AddColumn(XLSXOnly, "ReportMonth",
        each Text.Start([Name], 7), type text),

    ReadFiles = Table.AddColumn(AddMonth, "FileData",
        each Excel.Workbook(File.Contents([Folder Path] & [Name]), null, true)),

    GetSheet = Table.AddColumn(ReadFiles, "Sheet",
        each [FileData]{[Item="Sheet1", Kind="Sheet"]}[Data]),

    // Balance Sheet has 7 columns: Account spread cols 1-6, Total in col 7
    Expanded = Table.ExpandTableColumn(GetSheet, "Sheet",
        {"Column1","Column2","Column3","Column4","Column5","Column6","Column7"}),

    // Consolidate account name from indented columns
    AddAccount = Table.AddColumn(Expanded, "Account", each
        if [Column1] <> null then Text.Trim(Text.From([Column1]))
        else if [Column2] <> null then Text.Trim(Text.From([Column2]))
        else if [Column3] <> null then Text.Trim(Text.From([Column3]))
        else if [Column4] <> null then Text.Trim(Text.From([Column4]))
        else if [Column5] <> null then Text.Trim(Text.From([Column5]))
        else Text.Trim(Text.From([Column6] ?? "")), type text),

    AddTotal = Table.AddColumn(AddAccount, "Total",
        each [Column7], type number),

    SelectCols = Table.SelectColumns(AddTotal,
        {"ReportMonth", "Account", "Total"}),

    CleanRows = Table.SelectRows(SelectCols, each
        [Account] <> null
        and [Account] <> ""
        and [Account] <> "Account"
        and not Text.StartsWith([Account], "Balance sheet")
        and not Text.StartsWith([Account], "Cash basis")),

    // Key balance sheet metrics only
    KeyRows = Table.SelectRows(CleanRows, each
        List.Contains({
            "Bank",
            "Undeposited Funds",
            "Total Current assets",
            "Total Assets",
            "Security Deposit",
            "Pet Deposit",
            "Last Month's Rent",
            "Total Current liabilities",
            "Total Liabilities",
            "Equity retained earnings",
            "Equity net income",
            "Total Equity",
            "Total Liabilities and equity"
        }, [Account])),

    AddProperty = Table.AddColumn(KeyRows, "Property",
        each "2201 Polk St", type text)

in AddProperty
```

**Step 4** — Click **Close and Load To** → Table → sheet named `BalanceSheet`

---

## Query 6 — KPI Summary

This is the final query that pulls from all five above and produces the numbers Power BI displays.

**Step 1** — Go to `Data → Get Data → From Other Sources → Blank Query`

**Step 2** — Open **Advanced Editor** and paste:

```
let
    // Reference all loaded tables
    STR = Excel.CurrentWorkbook(){[Name="STR_Reservations"]}[Content],
    RR  = Excel.CurrentWorkbook(){[Name="RentRoll"]}[Content],
    PL  = Excel.CurrentWorkbook(){[Name="ProfitLoss"]}[Content],
    OS  = Excel.CurrentWorkbook(){[Name="OwnerStatement"]}[Content],
    BS  = Excel.CurrentWorkbook(){[Name="BalanceSheet"]}[Content],

    // Helper: get a value from a key-value table by account name
    GetVal = (tbl, acct) =>
        let r = Table.SelectRows(tbl, each [Account] = acct)
        in if Table.IsEmpty(r) then 0
           else List.Last(Table.Column(r, "Total")),

    // ── UNIT COUNTS from Rent Roll ────────────────────────────────
    TotalResidential  = 21,
    OccupiedUnits     = List.Sum(Table.SelectRows(RR,
                            each [Unit Type] <> "Commercial")[Occupied]),
    VacantUnits       = TotalResidential - OccupiedUnits,
    STR_Units         = Table.RowCount(Table.SelectRows(RR, each [Unit Type] = "STR")),
    LTR_Units         = Table.RowCount(Table.SelectRows(RR, each [Unit Type] = "LTR"
                            and [Occupied] = 1)),

    // ── RENT ROLL KPIs ────────────────────────────────────────────
    TotalMonthlyRent  = List.Sum(Table.SelectRows(RR,
                            each [Unit Type] = "LTR")[Rent]),
    TotalDeposits     = List.Sum(RR[Deposits]),
    OutstandingBal    = List.Sum(RR[Balance]),
    LTR_Occupancy     = LTR_Units / (TotalResidential - STR_Units),

    // ── STR KPIs from Airbnb ──────────────────────────────────────
    STR_Revenue       = List.Sum(STR[#"Gross earnings"]),
    STR_NetRevenue    = List.Sum(STR[Net Revenue]),
    STR_Stays         = Table.RowCount(STR),
    STR_Nights        = List.Sum(STR[Nights]),
    STR_DaysInPeriod  = 31,
    STR_Occupancy     = STR_Nights / (STR_Units * STR_DaysInPeriod),
    STR_RevPAU        = if STR_Units = 0 then 0 else STR_Revenue / STR_Units,
    STR_RevenuePerStay = if STR_Stays = 0 then 0 else STR_Revenue / STR_Stays,
    STR_AvgNights     = if STR_Stays = 0 then 0 else STR_Nights / STR_Stays,
    STR_Tax           = List.Sum(STR[#"Airbnb remitted tax"]),

    // ── FINANCIAL KPIs from P&L ───────────────────────────────────
    RentIncome_YTD    = GetVal(PL, "04.01Rent Apartments"),
    UtilityIncome_YTD = GetVal(PL, "04.11 Utilities"),
    LateFees_YTD      = GetVal(PL, "04.12 Late Fees"),
    TotalIncome_YTD   = GetVal(PL, "Total Income"),
    NOI_YTD           = GetVal(PL, "Net operating income"),

    // ── OWNER STATEMENT KPIs ──────────────────────────────────────
    CashBegin         = GetVal(OS, "Cash at beginning of period"),
    MonthlyRentIncome = GetVal(OS, "04.01Rent Apartments"),
    MonthlyNetIncome  = GetVal(OS, "Total Net income"),
    CashEnd           = GetVal(OS, "Cash at end of period"),
    CashAvailable     = GetVal(OS, "Cash available"),

    // ── BALANCE SHEET KPIs ────────────────────────────────────────
    CashInBank        = GetVal(BS, "Bank"),
    UndepositedFunds  = GetVal(BS, "Undeposited Funds"),
    TotalAssets       = GetVal(BS, "Total Assets"),
    SecurityDeposits  = GetVal(BS, "Security Deposit"),
    TotalLiabilities  = GetVal(BS, "Total Liabilities"),
    TotalEquity       = GetVal(BS, "Total Equity"),
    NetIncome_YTD     = GetVal(BS, "Equity net income"),

    // ── BUILD SUMMARY TABLE ───────────────────────────────────────
    Summary = #table(
        {"KPI", "Category", "Value", "Format"},
        {
            // Occupancy
            {"LTR Occupancy Rate",       "Occupancy", LTR_Occupancy,    "Percent"},
            {"STR Occupancy Rate",       "Occupancy", STR_Occupancy,    "Percent"},
            {"Total Units",              "Occupancy", TotalResidential, "Number"},
            {"Occupied Units",           "Occupancy", OccupiedUnits,    "Number"},
            {"Vacant Units",             "Occupancy", VacantUnits,      "Number"},
            {"LTR Units Occupied",       "Occupancy", LTR_Units,        "Number"},
            {"STR Units",                "Occupancy", STR_Units,        "Number"},
            // STR Revenue
            {"STR Gross Revenue",        "STR",       STR_Revenue,      "Currency"},
            {"STR Net Revenue",          "STR",       STR_NetRevenue,   "Currency"},
            {"STR Tax Remitted",         "STR",       STR_Tax,          "Currency"},
            {"STR Total Stays",          "STR",       STR_Stays,        "Number"},
            {"STR Total Nights",         "STR",       STR_Nights,       "Number"},
            {"STR Avg Nights per Stay",  "STR",       STR_AvgNights,    "Number"},
            {"STR RevPAU",               "STR",       STR_RevPAU,       "Currency"},
            {"STR Revenue per Stay",     "STR",       STR_RevenuePerStay,"Currency"},
            // LTR Revenue
            {"LTR Monthly Rent Roll",    "LTR",       TotalMonthlyRent, "Currency"},
            {"Outstanding Balances",     "LTR",       OutstandingBal,   "Currency"},
            {"Security Deposits Held",   "LTR",       TotalDeposits,    "Currency"},
            // P&L
            {"Rent Income YTD",          "Finance",   RentIncome_YTD,   "Currency"},
            {"Total Income YTD",         "Finance",   TotalIncome_YTD,  "Currency"},
            {"NOI YTD",                  "Finance",   NOI_YTD,          "Currency"},
            {"Monthly Rent Income",      "Finance",   MonthlyRentIncome,"Currency"},
            {"Monthly Net Income",       "Finance",   MonthlyNetIncome, "Currency"},
            // Balance Sheet
            {"Cash in Bank",             "Balance",   CashInBank,       "Currency"},
            {"Cash Available",           "Balance",   CashAvailable,    "Currency"},
            {"Total Assets",             "Balance",   TotalAssets,      "Currency"},
            {"Total Liabilities",        "Balance",   TotalLiabilities, "Currency"},
            {"Total Equity",             "Balance",   TotalEquity,      "Currency"},
            {"Net Income YTD",           "Balance",   NetIncome_YTD,    "Currency"}
        })

in Summary
```

**Step 4** — Click **Close and Load To** → Table → sheet named `KPI_Summary`

---

## Connect Power BI

Open Power BI Desktop and do the following:

```
Home → Get Data → Excel Workbook
→ Navigate to KPI_Master.xlsx in OneDrive
→ Select these tables: KPI_Summary, STR_Reservations, RentRoll
→ Click Load
```

**Suggested visuals:**

| Visual | Fields | Sheet |
|---|---|---|
| KPI cards | LTR Occupancy Rate, STR Occupancy Rate, NOI YTD, Cash in Bank | KPI_Summary |
| Bar chart | STR Revenue by ReportMonth | STR_Reservations |
| Table | Unit, Lease, Unit Type, Rent, Balance | RentRoll |
| Donut chart | Occupied vs Vacant units | RentRoll |
| Line chart | Monthly Net Income over time | OwnerStatement |
| KPI card | Outstanding Balances (Unit 308) | KPI_Summary |

**Set refresh schedule:**
```
Home → Transform Data → Data Source Settings
→ Set refresh to match your monthly export cadence
```

---

## Carolina's Monthly Workflow

Once everything is built, her entire process is:

```
1. Export Rent Roll from DoorLoop    → drop in RentRoll folder
2. Export P&L from DoorLoop          → drop in ProfitLoss folder
3. Export Owner Statement            → drop in OwnerStatement folder
4. Export Balance Sheet              → drop in BalanceSheet folder
5. Drop Airbnb CSV in Airbnb Exports folder
6. Open KPI_Master.xlsx
7. Press Ctrl + Alt + F5  (Refresh All)
8. Power BI refreshes automatically
```

**Total manual time:** Under 5 minutes per month.

---

## Known Issues to Resolve

| Issue | Impact | Action |
|---|---|---|
| Units 202-205 show $0 rent in DoorLoop despite active leases | LTR rent roll total is understated | Confirm with Carolina why rent is not entered |
| Unit 206 is VACANT in DoorLoop but appears in Airbnb | STR occupancy calculation may be wrong | Confirm with Carolina if 206 is STR or truly vacant |
| Listing-to-unit mapping unconfirmed | STR unit-level reporting is unreliable | Sit with Carolina and confirm all 6 mappings |
| Expenses show $0 in P&L | NOI equals gross income — cannot be trusted | Confirm where operating expenses are tracked |
| Airbnb listing "Downtwon" typo | Will create Unknown unit mapping | Fix listing name in Airbnb directly |

---

*Cielo Azul — KPI Master Power Query Setup | Prepared by Henrique Rio | May 2026*