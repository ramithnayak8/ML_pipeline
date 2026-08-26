# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Ramith Nayak
- **Lane:** Lane 2 — Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/ramithnayak8/ML_pipeline
- **Date:** 2026-08-26

## 0. Abstract

Which content page should an editor review first, out of thousands, when only a handful can be
checked each week? Using the 30,000-row FlyRank starter dataset (32 pseudonymized clients,
trailing-90-day search and engagement signals), we compared a transparent hand-written baseline
rule against a logistic regression and a random forest, all evaluated on an honest
client-grouped holdout split. Both models beat the baseline by a wide margin — Precision@50 rose
from 0.42 (rule) to 0.72 (logistic regression) and 0.64 (random forest), against a 51.1% test
base rate — but a companion validation audit showed a naive random split would have overstated
that result by 0.146 AUC, and that a single leaked column (`trend_pct`) can push AUC to a
suspicious 1.000 even under honest validation. The output is a ranked, reason-coded action
queue with disclosed confidence levels (only 10% "high"), meant to prioritize a limited weekly
review, not to declare any single page a certain decline.

## 1. Problem framing

**Decision:** the order a capacity-limited weekly content-review queue gets worked in — not a
one-off true/false verdict on any single page. **Unit of analysis:** one content page
(`content_id`), summarized over its trailing 90 days. **Output:** a ranked list with a score,
reason codes, a suggested action (refresh / expand and refresh / refresh and review CTR /
monitor), and a confidence label. **Action a human takes:** a content strategist works down the
list each cycle, using the reason codes — not the bare score — to decide what to actually do to
a page. **Cost of a wrong call:** a false positive wastes a scarce editor-hour reviewing a page
that did not need it; a false negative lets a real decliner keep losing visibility unnoticed for
another cycle, a cost that compounds the longer it is missed. **Why ML helps:** the underlying
signal is spread thin across many weakly informative, interacting fields — the model's own
top-10 feature importances top out at 0.16 and taper off gradually, with no single dominant
field a hand-written rule could cleanly encode.

## 2. Data safety

**Source:** `data/raw/content_refresh_anonymized.csv`, the starter dataset shipped in this repo
— 30,000 pseudonymized pages, 32 clients, all metrics aggregated over a trailing 90-day window
ending at export time. This capstone uses ONLY this starter slice, not the ~79M-row Hugging Face
warehouse release — a disclosed scope limitation (Section 8), not an oversight.

**Excluded, and why:**
- `trend_direction` / `trend_pct` — the label and the exact percentage it is bucketed from.
- `impressions_last_30d`, `clicks_last_30d`, `sessions_last_30d`, `impressions_prev_30d`,
  `clicks_prev_30d`, `sessions_prev_30d` — the six columns `trend_pct` is computed from; using
  them as features would hand the model the label's own ingredients.
- `provider_used`, `model_used` — content-generation metadata, not a performance signal.
- `content_id`, `client_id` — pseudonyms; grouping and splitting only, never model inputs.

**Confirmed:** none of the above appear in the model's real feature list (checked
programmatically, not just asserted). Nothing client-identifying, no raw URLs or queries, appear
anywhere in this work.

## 3. Baseline

A transparent rule built from three observable, human-readable conditions: `stale_visible`
(not updated in 180+ days, still drawing 500+ impressions), `low_ctr_visible` (visible,
well-positioned, but CTR under 0.5% — confirmed as a real signal in the signal audit, a +12.6
point decline-rate gap on n=9,759), and `thin_visible` (under 1,200 words, still drawing 250+
impressions). The rule's score is simply the count of matching reasons plus a visibility
tiebreaker — no fitted weights, and the label never enters the formula. On the same honest
client-holdout split used for the model (Section 5), the baseline reaches Precision@20 = 0.40
and Precision@50 = 0.42, against a 51.1% base rate — better than chance, but a rule alone leaves
real headroom.

## 4. Model / analysis

Logistic regression and a random forest, chosen per the "readable first, stronger only if it
earns it" rule for a yes/no observed label. Features: 18 numeric + 8 categorical raw fields
(engineered into ~46 columns after one-hot encoding), including two flags this work adds beyond
the reference pipeline — `has_keyword_data` / `has_word_count` — so missingness (which follows
`content_type`, not chance) stays explicit instead of silently folding into a `fillna(0)`.
**Target/proxy, in one sentence:** `is_declining_label = (trend_direction == "down")`, a bucket
over the CURRENT trailing-90-day window, not a future outcome — a proxy, not a forecast.
Deliberately left out: any product-decision flag (none ship in this dataset), and the six
window columns that would leak the label.

## 5. Evaluation

**Split:** client-grouped 80/20 (`GroupShuffleSplit` on `client_id`, seed 42; 25 train / 7 test
clients, zero client overlap) — pages from one client never appear on both sides, since a random
split would let a model partly memorize a client instead of learning a general pattern.

| Method | Precision@20 | Precision@50 | ROC-AUC |
|---|---:|---:|---:|
| Baseline rule | 0.40 | 0.42 | 0.53 |
| Logistic regression | 0.70 | 0.72 | 0.62 |
| Random forest | 0.55 | 0.64 | 0.61 |

(Test base rate: 51.1%.) An unexpected, honestly reported result: logistic regression beat
random forest here, the opposite of this repo's own reference pipeline — plausibly because with
only 32 clients total, which 7 land in the test fold can matter as much as the algorithm choice.
**Error analysis:** the random forest's actual wrong top-50 picks cluster around pages with weak
position (20–40) and a flat zero CTR that are `stable` or `up`, not `down` — the model reads
"weak position and CTR" as general risk, but that combination does not reliably signal
*direction* of movement, a real limit of these features for this label.

## 6. Interpretation

The model's top features are `days_with_impressions`, `log_impressions_90d`, and
`avg_position` — all plausible (a page needs a track record of visibility and a real position
before its trend means much), and none suspiciously dominant, which is itself a leakage
sanity-check. A deliberate negative-result check: adding `trend_pct` (the label's own
ingredient) as a feature collapses ROC-AUC from 0.605 to 1.000 even on the honest, out-of-fold
test set — proof the leak is real information leakage, not an in-sample artifact, and proof the
test harness itself works (a leaked feature SHOULD look "too good"). A second audit compared a
naive random split (ignoring `client_id`) against the honest one: the naive version overstated
AUC by +0.146 — a concrete measure of how much of a dishonest score would have been client
memorization, not skill.

## 7. Recommendation

The final ranked queue blends the validated logistic regression (60%, percentile-ranked, refit
on all rows once validation confirmed the method) with the baseline rule (40%, also
percentile-ranked). Reason codes carry the three baseline flags plus `model_flagged_high_risk`
(model probability ≥ 0.65), and confidence is "high" only for pages that clear a volume floor,
carry a real reason code, and land in the top decile of the blended score — 10.0% of all rows
(2,996 of 30,000). The top 10 by this score are 90% actually declining pages, and 10 distinct
clients appear in the top 50, a spread checked explicitly so the queue is not secretly a
single-client list. A FlyRank editor would work this list top-down each cycle, using the
attached reason code (not the bare score) to decide the action, and would stop trusting a pick
without a real reason code as anything more than a low-confidence nudge to `monitor`.

## 8. Reproducibility

Re-run from a fresh clone: `pip install -r requirements.txt`, then open and run
`work/notebooks/capstone.ipynb` top to bottom (it rebuilds the feature vector, baseline, split,
models, and ranked queue from `data/raw/content_refresh_anonymized.csv` alone — no external
data or credentials required). Random seed 42 is fixed everywhere a split or model is created.
Full supporting work lives in `work/notebooks/w01`–`w07`, each independently runnable.
**Disclosed scope limitation:** this capstone does not query the Hugging Face warehouse release
— extending it there (a real future-window label, multi-month history) is the natural next step
and is named explicitly rather than implied to already be done.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset. Data credit and program info:
[https://flyrank.ai](https://flyrank.ai).

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run of `work/notebooks/capstone.ipynb`.
