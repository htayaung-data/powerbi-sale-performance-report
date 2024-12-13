Total Qty = SUM ( Sale[Quantity] )


Revenue =
SUMX (
    Sale,
    VAR Qty = Sale[Quantity]
    VAR UnitPrice =
        RELATED ( 'Product'[Unit Price] )
    VAR Commission =
        CALCULATE (
            VALUES ( Commission[Percent] ),
            FILTER (
                Commission,
                Commission[Min] <= Sale[Quantity]
                    && Commission[Max] >= Sale[Quantity]
                    && Commission[Brand Name] = RELATED ( 'Product'[Brand Name] )
            )
        )
    RETURN
        Qty * UnitPrice * ( 1 - Commission )
)


Profit =
SUMX (
    Sale,
    VAR Qty = Sale[Quantity]
    VAR UnitPrice =
        RELATED ( 'Product'[Unit Price] )
    VAR UnitCost =
        RELATED ( 'Product'[Unit Cost] )
    VAR Commission =
        CALCULATE (
            VALUES ( Commission[Percent] ),
            FILTER (
                Commission,
                Commission[Min] <= Sale[Quantity]
                    && Commission[Max] >= Sale[Quantity]
                    && Commission[Brand Name] = RELATED ( 'Product'[Brand Name] )
            )
        )
    RETURN
        Qty * ( UnitPrice - UnitCost ) * ( 1 - Commission )
)


LY_Profit =
CALCULATE ( [Profit], SAMEPERIODLASTYEAR ( 'Date'[Date] ) )


Comparison with LY =
VAR CY = [Profit]
VAR PY =
    CALCULATE ( [Profit], SAMEPERIODLASTYEAR ( 'Date'[Date] ) )
VAR Pct_df = ( CY - PY ) / PY
RETURN
    Pct_df


ABC_Classification =
VAR ThresholdA = 0.7 -- Top 70%
VAR ThresholdB = 0.85 -- Next 15%
VAR BaseTable =
    CALCULATETABLE (
        ADDCOLUMNS ( SUMMARIZE ( Sale, Product[Product Code] ), "@Profit", [Profit] ),
        ALLSELECTED ( 'Product' )
    )
VAR CurrentProfit = [Profit]
VAR TotalProfit =
    CALCULATE ( [Profit], ALLSELECTED ( 'Product' ) )
VAR CumulativeTable =
    FILTER ( BaseTable, [@Profit] >= CurrentProfit )
VAR CumulativeProfit =
    SUMX ( CumulativeTable, [@Profit] )
VAR CumulativePct =
    DIVIDE ( CumulativeProfit, TotalProfit, 0 ) -- Avoid divide-by-zero errors
VAR ABC =
    SWITCH (
        TRUE (),
        CumulativePct <= ThresholdA, "A",
        CumulativePct <= ThresholdB, "B",
        "C"
    )
RETURN
    IF ( ISINSCOPE ( 'Product'[Product Name] ), ABC, BLANK () )


