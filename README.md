# CAC vs pLTV Adset Scoring (Weekly Budget Steering)

Automated a weekly decision system to **pause/scale adsets** across Meta, TikTok, and Google (Brand/Non-Brand/PMAX) by comparing **client-provided predicted LTV (pLTV)** against a statistically robust **CAC** estimate.

**Impact & scale**
- ~**500–1000 adsets** scored weekly → rolled up to ~**200 campaigns**
- Used by marketing experts to steer weekly spend and **monthly budget allocation**
- Contributed to ~**20% ROI improvement** and a steady improvement in the pLTV/CAC ratio
- Maintained **>95% spend coverage** (typically ~98%) after filtering

## What makes this robust
Sparse/noisy attribution at adset level makes naive CAC unstable. This approach balances **recency** and **statistical signal**:

- **Pre-filters:** drop adsets with low recent spend and no measurable engagement signal
- **Dynamic lookback window (3–8 weeks):** choose the smallest recent window meeting signal thresholds
- **Binomial “optimistic CAC” fallback:** if signal remains too small, use a beta/Clopper-Pearson upper bound on CVR
- **Explainable scoring:** rank adsets by signed distance from break-even line **pLTV = CAC**
- **Channel benchmark:** “Target CAC” defined as CAC at **80% spend coverage** (diagnostic + reference)

## Repo contents
- Full write-up: **[`docs/case-study.md`](./docs/case-study.md)**
- Core logic snippets (sanitized): **[`snippets/core-logic.md`](./snippets/core-logic.md)**

## Pipeline Overview
![Pipeline overview](img/pipeline-diagram.png)

