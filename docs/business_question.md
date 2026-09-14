# Business Questions — Golden Wok Food Delivery Analytics

These 5 questions define the scope of the MVP. A question counts as answered only when the
final report contains a clear finding, supporting evidence, an appropriate visualization/KPI,
and a practical recommendation.

## 1. What drives delivery delay?
What operational factors — traffic friction, distance, and time-of-day — are the strongest
drivers of delivery delay, and how do they interact?

**Relevant fields:** `traffic_friction_score`, `delivery_distance_km`, `time_slot_id`,
`promised_delivery_min`, `actual_delivery_min`

## 2. Where are delays concentrated?
Which zones, kitchens, riders, or time slots show chronically poor SLA performance?

**Relevant fields:** `zone_id`, `kitchen_id`, `rider_id`, `time_slot_id`, delay derived from
`promised_delivery_min` vs `actual_delivery_min`

## 3. What conditions hurt profitability?
Which operating conditions are associated with the biggest losses in profitability?

**Relevant fields:** `order_profit_ngn`, `delivery_cost_ngn`, `order_value_ngn`, cross-referenced
against zone, kitchen, traffic, and delay

## 4. How does delivery performance relate to customer satisfaction?
How does delivery performance — delay and food temperature on arrival — relate to customer
satisfaction?

**Relevant fields:** `customer_rating`, `food_temp_on_arrival_c`, delay

## 5. What should Golden Wok prioritize?
Based on findings from questions 1–4, what should Golden Wok prioritize operationally, and in
what order?

**Relevant fields:** synthesis of all above — no new fields, this question is answered from the
evidence gathered for Q1–Q4

---

## Scope note
Temporal/seasonal analysis is intentionally folded into Q1 and Q2 via `time_slot_id` rather than
treated as a separate question — with only 146 distinct order dates in this dataset, deep
seasonality analysis would stretch the evidence thinner than it can support. Rider-level hotspot
analysis is included in Q2 since `rider_id` was available in the source data at no extra
modeling cost.