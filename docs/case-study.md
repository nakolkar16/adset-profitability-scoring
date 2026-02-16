# Case Study — CAC vs pLTV Campaign Scoring (Weekly Budget Steering)

## 1) Context & decision
**Goal:** Help marketing channel owners decide which adsets/campaigns to **scale, hold, or reduce** based on profitability.

**Users:** A marketing team organized by channel experts (e.g., Meta owner reviewed Meta report; SEM Brand owner reviewed SEM Brand report).

**Cadence:** Weekly reporting; used to steer **monthly marketing budgets**.

**Scale:** ~**500–1000 adsets** scored weekly; rolled up into ~**200 campaigns** for channel-level review.

**Impact:** Contributed to ~**20% improvement in ROI**, a steady improvement in the pLTV/CAC ratio, and earlier detection of underperforming adsets (negative scores).

---

## 2) Data reality & constraints
- **Unit of analysis:** Weekly **adset-level** performance; rolled up to campaign and channel.
- **Channels covered:** TikTok, Meta (FB), Google Ads (Branded, Non-Branded, PMAX).
- **Inputs:**
  - Spend/clicks/impressions from tracking + ad platform sources (incl. Google Ads).
  - **pLTV** provided by the client: *average predicted LTV for users attributed to the adset*.
  - Attribution proxy **IHC** (Initializer → Holder → Closer): treated as **attributed orders**.
  - Handling incomplete attribution (redistribution): Tracking undercounted conversions at adset level. We estimated channel-level untracked conversions and derived a per-channel **uplift factor**, then scaled adset-level attributed orders (IHC) to produce `redistributed_ihc` used downstream for CAC estimation.”
- **Core challenge:** CAC becomes unstable for low-volume adsets if computed naively (tiny denominators), but long windows reduce responsiveness. The solution needed to be **stable, recent, and explainable**.

---

## 3) Approach overview
The pipeline produces a reliable CAC per adset using:
1) **Pre-filters** to remove no-signal adsets while preserving spend coverage  
2) **Dynamic lookback window selection (3–8 weeks)** to balance recency vs statistical signal  
3) **Binomial optimistic fallback** when even 8 weeks lacks sufficient signal  
4) A simple, interpretable **score based on distance from break-even**: pLTV vs CAC  

---

## 4) Pre-filters (high coverage, low noise)
Applied before CAC estimation:

### Spend gate (recent activity)
Drop adsets with **spend < $100** over the last **3 weeks**.  
Design choice: once an adset passes this gate, keep its full history for downstream calculations.

### Engagement gate (last 8 weeks)
Drop adsets with **clicks < 100 OR IHC < 1** over the last **8 weeks**.  
(IHC = attributed orders; `redistributed_ihc` used for totals.)

These gates were chosen to retain **>95% of spend** in the final report (typically **~98%**) while removing adsets with no measurable signal.

---

## 5) Dynamic lookback window (final: 3–8 weeks)
For each adset, weeks are processed in descending order (most recent first).

**Implementation detail:** reindex to include missing weeks (fill with zeros) and collapse duplicate `week_start` rows per adset before reindexing.

Starting from 3 weeks and expanding to 8, compute cumulative totals inside the window:
- `total_clicks = sum(clicks)`
- `total_ihc = sum(redistributed_ihc)`

Choose the *smallest* window where either condition is met:
- `total_clicks ≥ 300` **OR** `total_ihc ≥ 10`

If met (**LB path**):
- `estimated_cac = total_spend / total_ihc` (if `total_ihc > 0`)
- `selection_reason = "LB"`
- `weeks_used = selected window size`

This yields a CAC estimate that is **as recent as possible** while still meeting a minimum evidence threshold.

---

## 6) Fallback for low-stat adsets (binomial optimistic CAC)
If an adset does not reach the LB thresholds even at 8 weeks:

- Use up to the last 8 weeks.
- Estimate conversion-rate uncertainty using a **beta / Clopper-Pearson style interval** on CVR:
  - `CVR_ub = proportion_confint(count=trials, nobs=clicks, method="beta", alpha=0.2).upper`
  - Here, **trials = IHC (attributed orders)**.
- Convert CVR bound into conversion bound:
  - `trials_ub = clicks * CVR_ub`
- Compute “optimistic” CAC:
  - `fallback_cac = spend / trials_ub` (if `trials_ub > 0`)
- Use a consistent downstream conversion estimate (`ihc_final`):
  - LB path: `ihc_final = redistributed_ihc_LB`
  - Binomial path: `ihc_final = trials_ub`
- `selection_reason = "Binomial optimistic"`

This prevents low-signal adsets from being incorrectly labeled as poor solely due to sparse data, while still producing a practical decision tool.

---

## 7) Scoring (explainable, action-oriented)
Each adset is scored using distance from the break-even diagonal **pLTV = CAC**.

Let:
- `y = avg_pltv_influ_LB`
- `x = estimated_cac`

Signed perpendicular distance from the diagonal:
\[
d = \frac{y - x}{\sqrt{2}}
\]

The report includes:
- `signed_distance_from_y_eq_x` (the raw distance)
- `scaled_distance` (normalized to [-1, +1])
- `score_bucket`:
  - **+1** = pLTV >> CAC (good)
  - **0**  = pLTV ≈ CAC (break-even)
  - **-1** = pLTV << CAC (poor)

**Action mapping:** negative-scoring adsets → **hold or reduce spend**; positive-scoring → candidates to scale.

---

## 8) Deliverable (what stakeholders received)
Delivered weekly as:
- **Google Sheet** (primary artifact)
- **Google Doc summary** (context + key takeaways)
- **Slack message** (distribution + callouts)

### Adset-level sheet fields (representative)
- Identifiers: `campaign_name`, `campaign_id`, `adset_name`, `adset_id`
- Lookback stats: `clicks_LB`, `impressions_LB`, `sessions_LB`, `spend_LB`, `redistributed_ihc_LB`
- Conversions and value: `influenced_trials_LB`, `avg_pltv_influ_LB`
- CAC outputs: `estimated_cac`, `Binomial LB CVR`, `Binomial UB CVR`
- Metadata: `weeks_used`, `selection_reason`
- HVC cuts (if used): `hvc_male_influenced_LB`, `hvc_female_influenced_LB`, `share_hvc_male_LB`, `share_hvc_female_LB`
- Scoring: `signed_distance_from_y_eq_x`, `scaled_distance`

### Spend coverage diagnostic (trust-building)
A chart showing **cumulative CAC vs spend coverage** over the last 3 weeks:
- Helps validate that focusing on top-ranked adsets still represents most spend
- Used as a sanity check that filters weren’t excluding meaningful budget

### Channel benchmark
For some channels (e.g., SEM Brand), the report used a **Target CAC benchmark** defined as:
- **CAC at 80% spend coverage**

This provides a stable channel-level reference point to complement adset-level ranking.

---

## 9) Engineering / automation
- Orchestrated via **Airflow**.
- End-to-end pipeline: extract → preprocess/types → filters → dynamic CAC → scoring → rollups → write to GSheet.
- Robustness steps included:
  - decimal-to-float conversion to avoid serialization issues
  - dtype alignment for date filters
  - safe handling of empty windows and zero denominators

---

## 10) What I’d improve next
- Make pLTV horizon explicit (if accessible) and align CAC/valuation windows more tightly.
- Add a “confidence flag” (LB vs Binomial) more prominently in the UI (to steer interpretation).
- Add monitoring for attribution/tracking shifts that affect redistribution behavior.

