# Core logic (sanitized)

This document captures the core methodology without proprietary identifiers or connectors.

---

## Attribution redistribution (coverage correction)

When tracking undercounts conversions at adset level:
- Estimate untracked/unattributed conversions at **channel** level.
- Compute a per-channel uplift factor (coverage correction).
- Apply it to adset IHC:
  - redistributed_ihc = ihc * channel_uplift_factor

Assumption: missing conversions are roughly proportional to observed conversions within a channel.


## A) Pre-filters

**Spend gate (last 3 weeks):**
- Drop adsets with spend < $100 in the last 3 weeks.
- If an adset passes, keep its full history for downstream steps.

**Engagement gate (last 8 weeks):**
- Drop adsets with clicks < 100 OR IHC < 1 in the last 8 weeks.

(IHC = attributed orders; unattributed orders may be redistributed to adsets at channel level.)

---

## B) Dynamic lookback window (3–8 weeks)

For each adset, iterate over window sizes 3 → 8 using the most recent weeks:

1) Reindex to include missing weeks (fill 0s) and collapse duplicates by (adset_id, week_start).
2) Compute cumulative totals within the window:
   - total_clicks = sum(clicks)
   - total_ihc = sum(redistributed_ihc)
3) Select the smallest window where:
   - total_clicks ≥ 300 OR total_ihc ≥ 10
4) If selected (LB path):
   - CAC = total_spend / total_ihc
   - selection_reason = "LB"
   - weeks_used = window_size

---

## C) Binomial optimistic CAC fallback (low-stat adsets)

If thresholds are not met by 8 weeks:

1) Estimate CVR upper bound using a beta / Clopper-Pearson style interval:
   - CVR_ub = proportion_confint(count=trials, nobs=clicks, method="beta", alpha=0.2).upper
   - trials = IHC (attributed orders)

2) Convert to conversion upper bound:
   - trials_ub = clicks * CVR_ub

3) Compute optimistic CAC:
   - CAC_fallback = spend / trials_ub

4) Make rollups consistent via ihc_final:
   - if LB: ihc_final = redistributed_ihc_LB
   - if Binomial: ihc_final = trials_ub
   - selection_reason = "Binomial optimistic"

---

## D) Scoring (distance from break-even pLTV = CAC)

Let x = estimated_cac, y = avg predicted LTV (pLTV) for users attributed to the adset.

Signed perpendicular distance from y = x:
- d = (y - x) / sqrt(2)

The report includes:
- signed_distance_from_y_eq_x = d
- scaled_distance ∈ [-1, +1] (normalization step)
- score_bucket ∈ {-1, 0, +1} for quick actions
