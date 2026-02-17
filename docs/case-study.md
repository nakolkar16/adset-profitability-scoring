# Case Study — CAC vs pLTV Campaign Scoring (Weekly Budget Steering)

## Overview
A weekly scoring system that helps marketing channel owners **scale, hold, or reduce** adsets by comparing:
- **pLTV** (client-provided predicted LTV for users attributed to an adset)
vs
- **CAC** (estimated acquisition cost per attributed order)

The output is a ranked list of adsets plus campaign/channel rollups delivered via **Google Sheets**, a short summary doc, and a Slack update.

---

## 1) Context & decision
**Decision supported**
- Weekly spend changes (pause/scale/hold)
- Monthly budget allocation by channel

**Users**
- Marketing channel experts (each channel owner used their channel-specific report)

**Cadence**
- Weekly (run every Monday)

**Scale**
- ~500–1000 adsets scored weekly
- Rolled up into ~200 campaigns for review

**Outcome**
- Contributed to ~20% ROI improvement and a steady improvement in pLTV/CAC ratio

---

## 2) Data reality & constraints
**Unit of analysis**
- Weekly data at **adset level**, rolled up to campaign/channel

**Channels**
- TikTok
- Meta (FB)
- Google Ads (Brand, Non-Brand, PMAX)

**Key inputs**
- Spend, clicks, impressions (tracking + ad platform sources)
- pLTV (client-provided)
- **IHC = attributed orders** (attribution chain: initializer → holder → closer)
- Some orders were untracked/unattributed due to tracking limitations

### Handling incomplete attribution (redistribution)
Adset-level conversions were undercounted. At channel level, we estimated untracked/unattributed conversions and derived a **per-channel uplift factor**.  
We then scaled each adset’s attributed orders (IHC) to create:

- `redistributed_ihc = ihc * channel_uplift_factor`

**Assumption:** missing conversions are roughly proportional to observed conversions within the same channel.

---

## 3) Approach summary
The system produces a robust CAC estimate per adset using:
1. Pre-filters to remove no-signal adsets while preserving spend coverage  
2. A dynamic lookback window (3–8 weeks) to balance recency and statistical signal  
3. A binomial “optimistic CAC” fallback for low-stat adsets  
4. Explainable scoring based on distance from the break-even line **pLTV = CAC**

---

## 4) Pre-filters (keep signal, keep coverage)
### Spend gate (recent activity)
Drop adsets with:
- **Spend < $100** in the last **3 weeks**

Design choice: once an adset passes this gate, keep its full history for downstream steps.

### Engagement gate (last 8 weeks)
Drop adsets with:
- **Clicks < 100 OR IHC < 1** in the last **8 weeks**

These gates were chosen to retain **>95% spend coverage** in the report (typically ~98%).

---

## 5) Dynamic lookback window (final: 3–8 weeks)
Weeks are processed in descending order (most recent first).

**Implementation detail**
- Reindex per adset to include missing weeks (fill with zeros)
- Collapse duplicates by `(adset_id, week_start)` before reindexing

### Lookback selection rule
Start from 3 weeks and expand up to 8 weeks. For each window, compute:
- `total_clicks = sum(clicks)`
- `total_ihc = sum(redistributed_ihc)`

Select the smallest window where either condition is met:
- `total_clicks ≥ 300` **OR**
- `total_ihc ≥ 10`

If met (**LB path**):
- `estimated_cac = total_spend / total_ihc` (if `total_ihc > 0`)
- `selection_reason = "LB"`
- `weeks_used = window_size`

---

## 6) Binomial “optimistic CAC” fallback (low-stat adsets)
If the adset does not meet thresholds even with 8 weeks:

1) Estimate CVR uncertainty using a beta / Clopper-Pearson style interval:
- `CVR_ub = proportion_confint(count=trials, nobs=clicks, method="beta", alpha=0.2).upper`
- **trials = IHC (attributed orders)**

2) Convert CVR upper bound into conversion upper bound:
- `trials_ub = clicks * CVR_ub`

3) Compute optimistic CAC:
- `fallback_cac = spend / trials_ub` (if `trials_ub > 0`)

4) Make rollups consistent:
- LB path: `ihc_final = redistributed_ihc_LB`
- Binomial path: `ihc_final = trials_ub`
- `selection_reason = "Binomial optimistic"`

---

## 7) Scoring (distance from break-even)
Let:
- `x = estimated_cac`
- `y = avg_pltv_influ_LB`

Signed perpendicular distance from the diagonal `y = x`:
\[
d = \frac{y - x}{\sqrt{2}}
\]

The report includes:
- `signed_distance_from_y_eq_x` (raw distance)
- `scaled_distance` (normalized to [-1, +1])
- `score_bucket ∈ {-1, 0, +1}` for fast decisions

**Interpretation**
- +1: pLTV >> CAC (good candidate to scale)
- 0: near break-even
- -1: pLTV << CAC (hold/reduce)

---

## 8) Deliverable
### Primary artifact: Google Sheet
**Adset-level fields (representative)**
- IDs: `campaign_name`, `campaign_id`, `adset_name`, `adset_id`
- Lookback: `clicks_LB`, `impressions_LB`, `sessions_LB`, `spend_LB`, `redistributed_ihc_LB`
- Value/conv: `influenced_trials_LB`, `avg_pltv_influ_LB`
- CAC: `estimated_cac`, `Binomial LB CVR`, `Binomial UB CVR`
- Metadata: `weeks_used`, `selection_reason`
- HVC: `hvc_male_influenced_LB`, `hvc_female_influenced_LB`, `share_hvc_male_LB`, `share_hvc_female_LB`
- Score: `signed_distance_from_y_eq_x`, `scaled_distance`

### Spend coverage diagnostic (trust-building)
A chart of **cumulative CAC vs spend coverage** over the last 3 weeks:
- used to verify that filtering did not exclude meaningful budget
- used to define a stable channel benchmark

### Channel benchmark
For some channels, a **Target CAC** benchmark is defined as:
- **CAC at 80% spend coverage** (per channel)

### Distribution
- Google Doc summary (highlights + interpretation)
- Slack message with link + callouts

---

## 9) Automation & engineering
- Orchestrated via **Airflow** (weekly schedule)
- Steps: extract → preprocess → redistribute → filter → lookback CAC → score → rollups → publish to Sheets

Robustness notes:
- Decimal-to-float conversion for serialization safety
- Dtype alignment for week filtering
- Guardrails for empty windows and zero denominators

---

## 10) Improvements (future)
- Make pLTV horizon explicit (if available) and align horizons with CAC windowing
- Promote confidence indicators (LB vs Binomial) in the sheet UI
- Add monitoring for attribution/tracking shifts affecting redistribution
