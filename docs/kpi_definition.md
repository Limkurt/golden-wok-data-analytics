# KPI Definitions

SLA On-Time % =
DIVIDE(
    CALCULATE(COUNTROWS(fact_orders), fact_orders[actual_delivery_min] <= fact_orders[promised_delivery_min]),
    COUNTROWS(fact_orders)
)

Avg Delivery Time = AVERAGE(fact_orders[actual_delivery_min])

Avg Delay = AVERAGE(fact_orders[delivery_delay])

Total Revenue = SUM(fact_orders[order_value_ngn])

Total Cost = SUM(fact_orders[delivery_cost_ngn])

Profit Margin % = DIVIDE(SUM(fact_orders[order_profit_ngn]), SUM(fact_orders[order_value_ngn]))

Avg Customer Rating = AVERAGE(fact_orders[customer_rating])

Avg Rider Rating = AVERAGE(dim_rider[avg_rider_rating])