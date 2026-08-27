# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Ozair Malik
- **Lane:** Lane 2 — Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/ozair247/flyrank-ml-internship
- **Date:** 2026-08-28

## 0. Abstract

We ask whether a learned model can prioritize content refresh candidates better than a hand‑written rule using only search performance and content metadata from a single month. The analysis uses 9.8 million daily search rows from March 2026, aggregated to 176,000 content‑item records with features such as age, impressions, CTR, average position, and engagement signals. A Random Forest model trained on these features and evaluated on a client‑grouped held‑out split achieves Precision@50 of 0.56, compared with 0.04 for the Week‑4 baseline rule — a 52‑percentage‑point improvement. The model output is converted into a ranked action queue with reason codes, intended to help a content strategist decide which pages to review first. All findings are directional and decision‑support only; no production claims are made.

## 1. Problem framing

**Decision supported:** Which pages should a content strategist review for refresh first, given limited editorial time and resources.

**Unit of analysis:** One content item per client, aggregated over a calendar month.

**Output:** A ranked list (score) with action labels and reason codes.

**Human action:** Start at rank 1 and review the page for possible content refresh, metadata updates, or consolidation. Do not automate actions; every recommendation requires human judgment.

**Cost of a wrong call:** A false positive wastes editorial time on a page that does not need a refresh; a false negative lets a declining page go unnoticed, risking further traffic loss.

**Why data/ML helps:** The signal space (age, impressions, position, CTR, engagement) is small and interpretable, but a simple rule cannot capture non‑linear interactions (e.g., staleness matters most when a page is still visible). A learned model can combine these signals more flexibly and produce a calibrated risk score.

## 2. Data safety

**Data used:**
- `fact_content_daily_performance` (partitioned, month=2026-03)
- `dim_content` (content metadata)
- `dim_clients` (history start dates, used only for availability checks)

**Columns deliberately excluded:**
- `impr_last15` and `impr_prev15` — they are the label’s own ingredients; using them as features would be leakage.
- `gsc_sum_position` — redundant with `gsc_avg_position`, adds no independent signal.
- All raw URL, keyword, and client identifiers — kept hashed; never used as features.
- `health_score`, `priority_score`, `action_type` — product decision outputs that must not be learned from.

**Leakage risks considered:**
- Future‑window columns (`impr_last15`, `impr_prev15`) were never included in the feature set.
- Pseudonymous IDs (`client_hash_id`, `content_hash_id`, etc.) were used only for grouping and joining, never as model features.
- No client‑identifying information appears anywhere under `work/`.

## 3. Baseline

**Transparent rule:**  
`score = (content_age_days ≥ 180) × (impressions_month ≥ 500) × impressions_month`

Why it is a fair comparison:  
- It uses only two signals that are available before the decision moment.  
- It produces a ranking in the same format as the model (higher score = higher review priority).  
- It was evaluated on the **same** client‑grouped test split as the model.

**Baseline numbers (held‑out test set, 15 clients):**
- Precision@20 = 0.00  
- Precision@50 = 0.04  
- Test base rate = 0.269

The baseline’s near‑zero Precision@20 is not a bug; it reflects the rule’s tendency to push very old, high‑impression pages to the top, regardless of actual decline.

## 4. Model / analysis

**Method:** Random Forest classifier (300 trees, max depth 6, min samples leaf 20, random state 42).  
Why it fits the lane:  
- The label is binary (`is_declining_proxy`), and we need a probability to rank.  
- Random Forest can express non‑linear interactions (e.g., staleness matters only at certain traffic levels).  
- It is more interpretable than gradient boosting and easier to explain to stakeholders.

**Feature list:**
- `content_age_days` — days since content creation.  
- `impressions_month` — total Google Search Console impressions in March.  
- `ctr_month` — clicks / impressions × 100.  
- `avg_position_month` — average GSC position (0‑values removed).  
- `engagement_rate_month` — GA4 engaged sessions / sessions × 100 (0 where no sessions).  
- `has_ga4_month` — 1 if any GA4 data available; 0 otherwise.  
- `word_count` — number of words in content (missing filled with 0).  
- `has_word_count` — 1 if word count is present; 0 otherwise.

**Left out on purpose:**
- `clicks_month` — highly correlated with impressions; not needed for ranking.  
- `sessions_month`, `engaged_sessions_month` — raw counts are already represented by engagement rate.  
- `impr_last15`, `impr_prev15` — label ingredients (leakage).  
- Content type, intent, keyword metrics — not used to keep the model simple and focused on performance signals.

**Target (proxy) definition:**  
`is_declining_proxy = 1` if `impr_prev15 > 0` and `impr_last15 < 0.8 × impr_prev15`; else 0.

## 5. Evaluation

**Split:** Grouped by `client_hash_id` (70% train, 30% test) using `GroupShuffleSplit` with `random_state=42`.  
Why: Pages from the same client share a template, vertical, and team. A random split would leak information about a client from train to test and inflate scores.

**Metrics (same split for baseline and model):**

| Model | Precision@20 | Precision@50 | AUROC |
|---|---:|---:|---:|
| Baseline rule (Week 4) | 0.00 | 0.04 | 0.500 |
| Random Forest | 0.55 | 0.56 | 0.649 |

Test base rate = 0.269

**Error analysis:**  
Among the model’s top 50 predictions, 22 are false positives. They cluster in two age buckets: 5 under 90 days and 17 between 180–365 days. The top three false positives have `impressions_month = 1` and `ctr_month = 0` — pages with no prior traffic baseline. The proxy label cannot mark them as declining because there is nothing to decline from; the model incorrectly treats staleness as decline risk when traffic is near zero.

## 6. Interpretation

**Top features by permutation importance:**
1. `content_age_days` (0.066) — older pages are more likely to be flagged as declining; matches the baseline’s staleness idea.
2. `impressions_month` (0.036) — visible pages are more likely to show decline signal.
3. `avg_position_month` (0.021) — lower positions (worse ranking) correlate with decline risk.

**Surprises / negative results:**
- Very old pages (365+ days) actually have a **lower** decline rate than middle‑aged pages (90–180 days). Age alone is not monotonic; staleness matters, but only up to a point.
- Low CTR among top‑ranking pages is **not** associated with higher decline risk. This was a counterintuitive finding and means the original “CTR‑fix” flag may be misaligned.

**Key insight:** The model’s value comes from balancing staleness with visibility and recent ranking signals — exactly what a simple rule cannot do.

## 7. Recommendation

**Ranked actions:** The model outputs a queue with action labels:
- `review_for_refresh` — top decile risk + ≥500 impressions.
- `monitor` — high risk but low traffic, or moderate risk with visible traffic.
- `no_action` — low risk or invisible.

**How a FlyRank editor would use it tomorrow:**  
Start at rank 1, review the page (is it live? is the decline seasonal? is an update already scheduled?), and only then decide whether to refresh content, update metadata, or leave as is. Every item must pass a human check; the model is decision‑support, not an automated system.

**Confidence and limits:**  
On the held‑out client split, Precision@50 is 0.56, but this is based on a single month and a proxy label. I would not recommend using the model on a new client or a different time period without retraining. False positives on near‑zero‑traffic pages are the biggest risk; a simple pre‑filter (`impr_prev15 ≥ 10`) would likely improve precision further.

## 8. Reproducibility

**Environment:** Colab with DuckDB, scikit‑learn 1.3+, pandas 2.0+, numpy 1.24+, matplotlib 3.7+.

**Random seeds:** `random_state=42` for train/test split, logistic regression, and Random Forest.

**Commands to re‑run everything from a fresh clone:**
1. `git clone https://github.com/ozair247/flyrank-ml-internship`
2. Open `work/notebooks/w07_action_playbook.ipynb` in Colab (requires Hugging Face token).
3. Run all cells. The notebook regenerates:
   - `work/outputs/w07_action_queue.csv`
   - `work/figures/action_distribution.png`
   - `work/figures/importance.png`

**Sealed / holdout evaluation note:**  
All model evaluation in this report was performed on a grouped split using `GroupShuffleSplit` with `random_state=42`. The exact split code is in `w05_model.ipynb` and `w07_action_playbook.ipynb`. The metrics JSON (`w05_model_metrics.json`) is committed and contains the same numbers printed above.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset — [https://flyrank.ai](https://flyrank.ai)

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> **Metrics vs. base rate:** Test base rate = 0.269; model Precision@50 = 0.56 (lift = +0.291); model AUROC = 0.649 (baseline = 0.500).
> No causal claims; no “predicted Google’s algorithm”; no client‑identifying details; numbers match a fresh re‑run.
