# Grain Definition — fact_delivery

**Grain:** One row = one order/delivery event, uniquely identified by `order_id`.

## Profiling Results
- `order_id`: unique across all rows, no duplicates found.
- Foreign keys (`kitchen_id`, `zone_id`, `rider_id`, `time_slot_id`): valid distinct values, consistent with their respective dimension tables.
- `order_date`: 146 distinct dates in the dataset.
- `promised_delivery_min` / `actual_delivery_min`: distinct/unique counts lower than row count as expected for a bounded numeric measure (values naturally repeat across orders) — not a duplication concern.
- All other fact columns: 100% valid, high-cardinality, consistent with continuous financial/distance measures.

## Notes for Phase 2
- `food_temp_on_arrival_c` and `customer_rating` are nullable — KPI measures (e.g. average rating) must exclude nulls rather than treating them as 0.

## Conclusion
Grain confirmed at one-row-per-order-event. No deduplication logic required before building `fact_delivery`.