# Core logic (sanitized)

This file captures the methodology without proprietary identifiers, credentials, or client data.

---

## 1) Attribution redistribution (coverage correction)
Tracking undercounts conversions at adset level. We estimate untracked/unattributed conversions at **channel** level and derive a per-channel uplift factor:

- `redistributed_ihc = ihc * channel_uplift_factor`

Assumption: missing conversions are roughly proportional to observed conversions within a channel.

---

## 2) Pre-filters
**Spend gate (last 3 weeks)**
- Drop adsets with spend < $100 in the last 3 weeks.
- If an adset passes, keep its full history for downstream steps.

**Engagement gate (last 8 weeks)**
- Drop adsets with clicks < 100 OR IHC < 1 in the last 8 weeks.

(IHC = attributed orders.)

---

## 3) Dynamic lookback window (3–8 weeks)
For each adset:
1) Ensure missing weeks exist (reindex + fill zeros)
2) Collapse duplicate `(adset_id, week_start)` rows (required before reindexing)
3) Scan windows from 3 → 8 weeks using most recent weeks

Within a window, compute:
- `total_clicks = sum(clicks)`
- `total_ihc = sum(redistributed_ihc)`

Select the smallest window where:
- `total_clicks ≥ 300` OR `total_ihc ≥ 10`

If selected:
- `CAC = total_spend / total_ihc`
- `selection_reason = "LB"`
- `weeks_used = window_size`

---

## 4) Binomial optimistic CAC fallback
If thresholds are not met by 8 weeks:

1) Compute CVR upper bound (beta / Clopper-Pearson style):
- `CVR_ub = proportion_confint(count=trials, nobs=clicks, method="beta", alpha=0.2).upper`
- `trials = IHC` (attributed orders)

2) Convert to conversion bound:
- `trials_ub = clicks * CVR_ub`

3) Optimistic CAC:
- `CAC_fallback = spend / trials_ub`

4) Keep rollups consistent:
- LB path: `ihc_final = redistributed_ihc_LB`
- Binomial path: `ihc_final = trials_ub`
- `selection_reason = "Binomial optimistic"`

---

## 5) Scoring (distance from break-even pLTV = CAC)
Let:
- `x = estimated_cac`
- `y = avg predicted LTV (pLTV)`

Signed perpendicular distance from `y = x`:
- `d = (y - x) / sqrt(2)`

Report fields:
- `signed_distance_from_y_eq_x`
- `scaled_distance` in [-1, +1]
- `score_bucket` in {-1, 0, +1}
