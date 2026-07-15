# DAX Measures

Documentation for the KPI measures defined in the CNG_Analytics semantic model. All four live in the `KPIs` display folder on their respective fact tables.

## Net Revenue

**Table:** `fact_sales`

```dax
Net Revenue = SUM(fact_sales[NetAmount_USD])
```

Total net sales revenue. No separate return adjustment is needed because `NetAmount_USD` is already signed — it flips negative whenever `is_return` is `TRUE`, so summing it nets returns out automatically.

## Returns Rate

**Table:** `fact_sales`

```dax
Returns Rate =
DIVIDE(
    CALCULATE(COUNTROWS(fact_sales), fact_sales[is_return] = TRUE),
    COUNTROWS(fact_sales)
)
```

Share of sales order lines flagged as returns, out of all sales order lines. A rising rate can point to product quality problems or order-entry mistakes.

## Supplier On-Time Rate

**Table:** `fact_purchases`

```dax
Supplier On-Time Rate = 1 - AVERAGE(fact_purchases[is_late_delivery])
```

Percentage of purchase orders delivered on or before the expected date. `is_late_delivery` (0/1) is precomputed in Gold from `expected_lead_time_days` vs. `actual_lead_time_days`, so this measure just inverts its average. Answers "can I trust my suppliers to deliver on time?"

## Inventory Value at Risk

**Table:** `fact_inventory`

```dax
Inventory Value at Risk = SUM(fact_inventory[ValueBelowReorder_USD])
```

Dollar value of inventory lots currently at or below their reorder level. `ValueBelowReorder_USD` is precomputed in Gold from `BelowReorder`, `StockQty`, and `UnitCost_USD`, so this measure just sums it. Flags how much stock value is close to running out.

## Notes

- **Gross margin was intentionally left out.** Sales and Purchase orders have no shared key (an SO's product wasn't necessarily bought via a specific PO), so `Revenue − COGS` can't be computed reliably without a proxy. Revisit only if a cost basis that doesn't need cross-table matching (e.g. product-level standard cost) becomes acceptable for the business question being asked.
- All four measures are built from columns that are either already netted/flagged in the Gold layer (`NetAmount_USD`, `is_return`, `is_late_delivery`, `ValueBelowReorder_USD`) or a direct aggregation of one — none require assumptions the data doesn't support.
