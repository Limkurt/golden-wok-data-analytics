# Data Quality & Limitations Notes — Golden Wok Analytics

**Location in repo:** `docs/data_quality_notes.md`
**Last updated:** Phase 2 (dimension modelling)
**Purpose:** Document known data-generation quirks, derived fields, and analytical decisions so another person (or future you) can reproduce the model and understand why certain source columns are not used at face value.

---

## 1. Dataset generation context

This dataset is **synthetically generated** (confirmed via `DATA_DICTIONARY.md`'s "Data Generation" sections). Most columns were produced independently via `faker` or `numpy` random distributions, rather than simulated from real underlying business logic. Known independent (uncorrelated-by-design) fields include:

- `actual_delivery_min` — drawn from its own normal distribution (mean=52, std=18), not derived from `promised_delivery_min` + traffic/distance/weather effects.
- `order_profit_ngn`, `delivery_cost_ngn`, `customer_rating`, `food_temp_on_arrival_c` — each independently random, not formulaically linked to delay, distance, or traffic.
- FK columns in `fact_orders` (`kitchen_id`, `zone_id`, `rider_id`, `time_slot_id`) — assigned via `faker`, not weighted by any real relationship to outcomes.

**Implication:** weak or statistically insignificant correlations between operational conditions and outcomes (e.g. traffic vs. delay, weather vs. rating) may be a true reflection of the data, not a modelling error. Per project Rule 4, findings are reported as observed data, statistical association, or plausible explanation — never presented as causation the generation process doesn't support.

---

## 2. `dim_time_slot` — internal inconsistency

`dim_time_slot` (48 rows) is the most affected table. Its columns were each generated independently and **do not agree with each other** for the same `time_slot_id` row. Specifically observed:

| Field | Issue |
|---|---|
| `hour_of_day` | Text values contain invalid minute components (e.g. `02:95`) and, once the hour portion is extracted, only ever range **0–10** — far short of a real 24-hour day. Cannot represent the challenge brief's PM rush window (16:00–19:30) at all. |
| `slot_label` | Well-formed hour-block text spanning the full day (e.g. `07:00-08:00` through `23:00-00:00`), but **does not align** with `hour_of_day`, `meal_period`, `is_rush_hour`, or `day_of_week` for the same row. |
| `meal_period` | Does not correspond to `hour_of_day`, `hour_bucket`, or `slot_label` (e.g. a row with an evening `slot_label` may show `meal_period = "Afternoon"`). |
| `is_weekend` | Contradicts `day_of_week` — some Saturday/Sunday rows show `False`, some weekday rows show `True`. |
| `is_rush_hour` | Original flag does not correspond to any hour field; not usable as-is. |

**Decision:** rather than reconciling these (which would fabricate consistency the data doesn't have), specific fields were designated trustworthy ground truth and the rest were retired from analytical use.

---

## 3. Derived fields (added in Power Query)

| Derived column | Formula (M) | Source of truth | Purpose |
|---|---|---|---|
| `hour_bucket` | `Number.FromText(Text.BeforeDelimiter([hour_of_day], ":"))` | `hour_of_day` | Initial hour extraction — **superseded**, see below. |
| `slot_start_hour` | `Number.FromText(Text.BeforeDelimiter([slot_label], ":"))` | `slot_label` | Full-range (0–23) hour value; the actual field used for any hour-of-day / temporal analysis. |
| `is_weekend_calc` | `if List.Contains({"Saturday","Sunday"}, [day_of_week]) then "Yes" else "No"` | `day_of_week` | Consistent weekend flag. |
| `is_rush_hour_calc` | `if [slot_start_hour] >= 7 and [slot_start_hour] < 9 then "Yes" else if [slot_start_hour] >= 16 and [slot_start_hour] < 20 then "Yes" else "No"` | `slot_start_hour` | Matches the challenge brief's stated rush-hour windows (07:00–09:00, 16:30–19:30). |

**Known approximation:** the brief's rush-hour boundaries include a half-hour mark (16:30, 19:30). Neither `hour_of_day` nor `slot_label` carries sub-hour precision, and `fact_orders` has no per-order timestamp finer than `order_date` + `time_slot_id`. The 16:30–19:30 window is therefore approximated to the nearest full hour blocks (16:00–20:00). This is a genuine precision limit of the source data, not a modelling choice that could be tightened with more careful logic.

**Superseded field note:** `hour_bucket` was the first attempt at extracting a usable hour value, built from `hour_of_day`. It was abandoned in favor of `slot_start_hour` once it became clear `hour_of_day` only spans 0–10 and cannot represent the PM rush window. `hour_bucket` remains in the model but is not used in any visual or measure.

---

## 4. Fields retired from analytical use

These columns remain in the model (not deleted — that isn't a call to make silently on source data) but are **not used** in any visual, slicer, or DAX measure:

- `dim_time_slot[slot_label]` — superseded by `slot_start_hour` for hour analysis; label itself doesn't correspond to other fields.
- `dim_time_slot[hour_of_day]` and `dim_time_slot[hour_bucket]` — capped at 0–10, unsuitable for full-day analysis.
- `dim_time_slot[is_weekend]` — contradicts `day_of_week`; use `is_weekend_calc` instead.
- `dim_time_slot[is_rush_hour]` — doesn't correspond to any hour field; use `is_rush_hour_calc` instead.
- `dim_time_slot[meal_period]` — not referenced by any of the 5 locked business questions; left unused rather than reconciled.

## Trusted fields going forward (`dim_time_slot`)

`time_slot_id` (PK), `weather_condition`, `day_of_week`, `slot_start_hour`, `is_weekend_calc`, `is_rush_hour_calc`.

---

## 5. Other known caveats (from `DATA_DICTIONARY.md` / Phase 1 profiling)

- **Weather granularity:** there is no standalone `dim_weather` table. `weather_condition` lives in `dim_time_slot`, meaning it represents "typical weather associated with this time slot," not the actual weather on a specific order's date. Any weather-related finding (e.g. rain vs. delay) should carry this caveat.
- **Sparse order dates:** `fact_orders.order_date` covers only 146 distinct dates out of the nominal 2023-01-01 to 2024-12-31 range. `dim_date` intentionally contains all 731 calendar dates; dates with no orders are expected to show zero activity, not a data-quality gap.
- **Rider–kitchen snowflake edge:** `dim_rider.assigned_kitchen_id` does not have to match `fact_orders.kitchen_id` for a given order — to be checked explicitly during relationship validation (Phase 2, Model view) and reflected in how the Q2 rider-hotspot narrative is framed.

---

## 6. Why this matters for the final report

Per `product_spec.md` Rule 4 (distinguish observed data / statistical association / plausible explanation) and the Non-Functional Requirements in `system_design.md` (reproducibility), this document exists so that:

1. Any weak or absent correlation found in Phase 3 analysis can be correctly attributed to the data's generation process rather than treated as a modelling failure.
2. Someone else opening this `.pbix` file understands why `slot_start_hour`, `is_weekend_calc`, and `is_rush_hour_calc` exist alongside their raw source counterparts, instead of assuming duplicate/redundant columns.
