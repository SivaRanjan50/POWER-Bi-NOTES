# Power BI Concepts & DAX Functions
## 1. TREATAS Function
```
Creates a virtual relationship between tables without physical relationships.
What it does: It treats the values from one table as if they were values from another table, creating temporary filter context.
Use when:
    Tables have no direct physical relationship
    Working with disconnected tables
    Implementing many-to-many relationships
    Dynamic filtering between unconnected tables
    Role-playing dimensions without duplicating tables
```

## 2. ERROR-HANDLING: IFERROR
```
Catches errors and returns alternative values.
DAX
IFERROR([Expression], [AlternativeValue])
```

## 3. MEASURES vs CALCULATED COLUMNS
```
Measures:
    Dynamic calculations calculated on-demand when visuals render
    Used in visuals as values
    Respond to filters/slicers
    Perform aggregations (SUM, AVERAGE, COUNT)
    No physical storage in the model
    Used for:
        Totals (e.g., SUM(Sales[SalesAmount]))
        Ratios and percentages
        Time intelligence calculations
        Dynamic calculations based on user filters

Calculated Columns:
    Static values added to a table
    Calculated row-by-row during data refresh
    Physically stored in the data model (increases model size)
    Use row context
    Used for row-level calculations
        Profit = Sales[SalesAmount] - Sales[Cost]
    Key difference: Measures use filter context, Calculated Columns use row context
```

## 4. TABLE FUNCTIONS
### VALUES()
```
Returns a one-column table of distinct values
Includes blank rows for unmatched relationships
Visible in filter context
    ex: VALUES(Sales[Category])
```

### DISTINCT()
```
Returns a one-column table of distinct values
Excludes blank rows for unrelated values
    DISTINCT(Sales[SalesPeople])
```
### COUNTROWS()
```
Returns a single scalar value (count of rows)
Use with table expressions
```

### SUMMARIZE()
```
Creates a summary table grouped by specified columns
Returns multiple columns
    SUMMARIZE(Sales, Sales[Category], Sales[Region])
```

### SUMMARIZECOLUMNS() (Recommended)
```
Modern, optimized version of SUMMARIZE()
Designed for reporting/visualization
Automatically handles incompatible filters

    SUMMARIZECOLUMNS(
        Product[Category],
        "Sales", SUM(Sales[SalesAmount]),
        "Customers", DISTINCTCOUNT(Sales[CustomerID])
    )
```

### ADDCOLUMNS()
```
Adds calculated columns to an existing table
Creates virtual tables for filtering/analysis

    ADDCOLUMNS(
        SUMMARIZE(Sales, Sales[Category]),
        "Total Sales", SUM(Sales[SalesAmount]),
        "Average Sales", AVERAGE(Sales[SalesAmount]),
        "Profit", SUM(Sales[SalesAmount]) - SUM(Sales[Cost])
    )
```

### SELECTCOLUMNS()
```
Returns a new table with selected columns
Can rename columns
    
    SELECTCOLUMNS(
        Sales,
        "OrderID", Sales[OrderID],
        "Product Category", Sales[Category],
        "SalesAmount", Sales[SalesAmount],
        "Region", Sales[Region]
    )
```

## 5. FILTER CONTEXT FUNCTIONS
### ALL()
```
Removes all filters (visual-level + slicers)
Returns entire unfiltered dataset

TotalAll = CALCULATE(SUM(Policies[PremiumAmount]), ALL())
```

### ALLEXCEPT()
```
Removes filters from all columns except specified ones

TotalByState = CALCULATE(
    SUM(Policies[PremiumAmount]),
    ALLEXCEPT(Policies, Policies[State])
)
```

### ALLSELECTED()
```
Respects slicers/page filters
Ignores visual-level filters only

SelectedTotal = CALCULATE(SUM(Policies[PremiumAmount]), ALLSELECTED())
```
```
Comparison:
    Aspect	                    ALL()	          ALLSELECTED()
    Visual-level filters	    Ignored	          Ignored
    Slicers/Page filters	    Ignored	          Respected
    Result	                    Entire table	  Filtered by user selection
```


## 6. RANKING FUNCTIONS
### RANKX()
```
Returns the rank of an element within a set.

RANKX(
    ALL(Products[ProductName]),  -- Table to rank
    [Total Sales],               -- Expression to rank by
    ,                            -- Value (optional)
    DESC,                        -- Order: DESC/ASC
    Dense                       -- Tie handling: Dense/Skip
)
```

### TOPN()
```
Returns the top N rows sorted by an expression.

TOPN(
    5,
    SUMMARIZE(Products, Products[ProductName], "Sales", [Total Sales]),
    [Sales], DESC
)
```

### 7. LOOKUPVALUE() vs RELATED()
```
LOOKUPVALUE()
    Retrieves values from another table without physical relationship
    Can use multiple criteria
    Returns blank if no match (or specified default)

Use when:
    Adding descriptive columns to fact tables
    Working with small/medium datasets
    Displaying values from unrelated tables
    Creating calculated columns with complex criteria
        ProductName = LOOKUPVALUE(
            Products[ProductName],
            Products[ProductID], Sales[ProductID],
            "Unknown Product"  -- Default value
        )
```

```
When to use TREATAS vs LOOKUPVALUE:
TREATAS: Dynamic relationships, many-to-many, advanced filtering

LOOKUPVALUE: Simple column lookups, calculated columns
```

## 8. COMMON BUSINESS CALCULATIONS
### 1. Revenue Calculations:
```
Gross Revenue = SUM(Sales[SalesAmount])
Net Revenue = SUM(Sales[SalesAmount]) - SUM(Sales[Discount])
```

### 2. Profitability:
```
Total Profit = SUM(Sales[Profit])
Profit Margin = DIVIDE(SUM(Sales[Profit]), SUM(Sales[SalesAmount]))
```

### 3. Volume Analysis:
```
Total Units Sold = SUM(Sales[Quantity])
Average Order Size = AVERAGE(Sales[Quantity])
```

### 4. Pricing Analysis:
```
Avg Selling Price = AVERAGE(Sales[UnitPrice])
Product Contribution = [Product Sales] / [Total Sales]
```

### 5. Unit Price
```
The price of one single unit of a product
  Ex: If a laptop costs $150 each → Unit Price = $150
Used for calculating extended amounts when multiplied by quantity
Typically stored in Product or Sales table
    UnitPrice = Sales[UnitPrice]  // Direct column reference
```

### 6. Quantity
```
The number of units sold in a transaction
ex: If a customer buys 10 laptops → Quantity = 10
use: Multiplied with unit price to get sales amount
In Data Model: Found in Sales/Fact table
    Quantity = Sales[Quantity]  // Direct column reference
```

### 7. Sales Amount (Revenue)
```
Definition: Total revenue from the sale before any discount
Calculation: Quantity × Unit Price
Ex:
10 laptops × $150 each = $1,500 Sales Amount
      // If stored as column:
      SalesAmount = Sales[SalesAmount]
      
      // If calculated:
      Calculated Sales Amount = SUMX(
          Sales,
          Sales[Quantity] * Sales[UnitPrice]
      )
```
### 8. Discount
```
The amount deducted from Sales Amount
Types:
Fixed amount (e.g., $50 off)
Percentage (e.g., 10% off)
Example: $50 discount means customer pays $50 less
In Data Model: Can be stored as amount or percentage
    Discount Amount = 
    Sales[SalesAmount] * Sales[DiscountPercent]
```
### 9. Cost
```
The total cost to the business for products sold
Usually from Product table (cost price per unit)
      Quantity × Cost Price
      Ex:
      Cost per laptop = $120
      10 laptops cost = 10 × $120 = $1,200

      Total Cost = 
              SUMX(
                  Sales,
                  Sales[Quantity] * RELATED(Products[CostPrice])
              )
```
### 10. Profit
```
The actual profit made on the sale
      SalesAmount - Discount - Cost
        Sales Amount:    $1,500
        Discount:        $0
        Cost:            $1,200
        Profit:          $1,500 - $0 - $1,200 = $300

// Simple profit calculation
Profit = Sales[SalesAmount] - Sales[Discount] - [Total Cost]

// Alternative with related cost
Profit = 
    SUMX(
        Sales,
        (Sales[Quantity] * Sales[UnitPrice])                    -- Revenue
        - (Sales[Quantity] * Sales[DiscountPerUnit])            -- Discount
        - (Sales[Quantity] * RELATED(Products[CostPrice]))       -- Cost
    )
```


```
// Complete Sales Analysis Measures
Revenue = SUM(Sales[SalesAmount])

Total Discount = SUM(Sales[DiscountAmount])

Net Sales = [Revenue] - [Total Discount]

Total Cost = SUMX(Sales, Sales[Quantity] * RELATED(Products[CostPrice]))

Gross Profit = [Net Sales] - [Total Cost]

Profit Margin = DIVIDE([Gross Profit], [Revenue])

Average Selling Price = AVERAGE(Sales[UnitPrice])

Total Units Sold = SUM(Sales[Quantity])

Average Order Value = DIVIDE([Revenue], DISTINCTCOUNT(Sales[OrderID]))

```










