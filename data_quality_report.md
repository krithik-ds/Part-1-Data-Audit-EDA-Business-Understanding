# Data Quality Report — D2C Churn Capstone

**Dataset snapshot date:** 2025-09-30  
**Report scope:** All 7 raw files in the capstone dataset package

---

## 1. Missing Values

| File | Column | Missing Count | Missing % | Notes |
|---|---|---|---|---|
| `customers.csv` | `loyalty_tier` | 1,386 | 57.8% | Intentional — null means "not enrolled in loyalty programme". Impute as a separate category `"Not Enrolled"`, do not drop. |
| `customers.csv` | `skin_type` | 401 | 16.7% | Self-reported and optional. Can be imputed with mode or kept as `"Unknown"` category. |
| `orders.csv` | `rating` | 80 | ~0.8% | Customers who didn't rate their order. Use mean imputation or a separate flag column before averaging. |

**Other files:** No missing values found in `support_tickets.csv`, `web_events_snapshot.csv`, `churn_labels.csv`, or `intervention_history.csv`.

**Treatment recommendation:**  
- `loyalty_tier` → fill NaN with `"Not Enrolled"` (this is meaningful business state, not random missingness)
- `skin_type` → fill NaN with `"Unknown"` if used as a feature
- `rating` → when computing `avg_rating`, use `.mean()` which already skips NaN, or flag with a boolean `has_rated` column

---

## 2. Duplicate-Like Records

| File | Issue | Count | Detail |
|---|---|---|---|
| `orders.csv` | Order IDs ending in `_DUP` | 12 | Intentional duplicates simulating real-world deduplication problems. These represent re-submitted or copy-pasted order records. |

**Treatment recommendation:**  
Remove all rows where `order_id.str.endswith("_DUP")` before any analysis or feature engineering. These are not real transactions.

---

## 3. Outliers

| File | Column | Issue | Count | Detail |
|---|---|---|---|---|
| `orders.csv` | `gross_amount` | Extremely high values | 101 (above 99th pct) | Max value is ₹24,789 vs. median ~₹700. These could be bulk/B2B orders or data entry errors. |

**Treatment recommendation:**  
- Investigate whether the high-value orders belong to specific customers or categories
- For aggregated features (total spend, avg order value), these are valid and should be kept — they reflect real customer LTV
- For outlier-sensitive models, consider capping `gross_amount` at the 99th percentile (₹~5,000) or applying a log transform

---

## 4. Post-Snapshot Orders (Leakage Risk — Critical)

| File | Column | Issue | Count |
|---|---|---|---|
| `orders.csv` | `order_date` | Rows dated after 2025-09-30 | 1,872 |

**This is the most important data quality issue for modeling.**  
Orders after `2025-09-30` were used to construct the churn label (`churn_next_60d`). Using any feature derived from these rows as a model input would be **data leakage** — the model would be trained on information it cannot possibly have at prediction time.

**Treatment recommendation:**  
Always filter: `orders = orders[orders["order_date"] <= "2025-09-30"]` before any feature engineering. The `rfm_modeling_snapshot.csv` already enforces this.

---

## 5. Join Coverage

| Relationship | Expected Rows | Actual Distinct Customer IDs | Gap |
|---|---|---|---|
| `customers` → `orders` (pre-snapshot) | 2,400 | ~2,350+ | A small number of customers have no pre-snapshot orders — they may be very new signups |
| `customers` → `support_tickets` | 2,400 | 1,247 | 1,153 customers have zero tickets — this is **expected and meaningful**, not an error |
| `customers` → `web_events_snapshot` | 2,400 | 2,400 | Full coverage |
| `customers` → `churn_labels` | 2,400 | 2,400 | Full coverage |
| `customers` → `intervention_history` | 2,400 | 2,400 | Full coverage |

**Treatment recommendation:**  
Always use **left joins from `customers`** as the base. Customers with no tickets should receive `ticket_count = 0`, not be dropped.

---

## 6. Date Consistency

- All `snapshot_date` columns in `web_events_snapshot`, `churn_labels`, `rfm_modeling_snapshot`, and `intervention_history` are consistently `2025-09-30`
- `orders.order_date` ranges from `2024-01-09` to `2025-11-29` (spanning both pre and post snapshot)
- `support_tickets.ticket_date` ranges from `2024-01-13` to `2025-09-30` (all pre-snapshot — no leakage risk)
- `customers.signup_date` ranges from `2024-01-01` to `2025-09-15` (all customers signed up before or on snapshot date)

No inconsistencies found in date formats.

---

## 7. Columns That Could Cause Leakage

| Column | File | Risk | Notes |
|---|---|---|---|
| `order_date > 2025-09-30` in `orders.csv` | `orders.csv` | **HIGH** | Must filter out before feature engineering |
| `churn_next_60d` | `rfm_modeling_snapshot.csv` | **HIGH** | This is the target variable — never use as a feature |
| `split` | `churn_labels.csv`, `rfm_modeling_snapshot.csv` | Low | Not a leakage risk, but do not use as a feature |

---

## Summary

| Category | Count | Action Required |
|---|---|---|
| Columns with missing values | 3 | Impute with meaningful defaults |
| Duplicate-like records | 12 | Drop `_DUP` rows |
| Outlier order values | 101 | Keep for LTV features; cap/log-transform for sensitive models |
| Post-snapshot rows (leakage) | 1,872 | Filter out before any feature creation |
| Join gaps (no tickets) | 1,153 customers | Fill `ticket_count = 0` — expected, not an error |
