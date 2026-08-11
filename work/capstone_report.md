# Capstone Report — CTR Opportunity Scoring for Content Review

- **Author:** Sham Kottish
- **Lane:** CTR / Engagement Opportunity Scoring
- **Repo:** https://github.com/ShamKottish/FlyRankML
- **Date:** August 11, 2026

## 0. Abstract

FlyRank content teams may manage many pages while having limited SEO and editorial review time, creating a practical need to decide which pages deserve attention first. This study tests whether observed search-performance signals can help rank content pages for CTR-focused human review using pseudonymized FlyRank warehouse data, with March 2026 used as the feature period and April 2026 as a later evaluation period. I compare a transparent position-adjusted CTR baseline with a Random Forest model using March impressions, clicks, CTR, average search position, position volatility, and active search days, evaluated on clients completely held out from training. Both the Random Forest and transparent baseline achieved **0.880 Precision@50**, substantially above the held-out opportunity base rate of **0.215**, while the Random Forest showed stronger broader ranking performance with **Average Precision of 0.610 versus 0.555** and **ROC-AUC of 0.864 versus 0.795**. The results support using search-performance signals as directional decision-support for deciding which content pages a human reviewer should inspect first, while showing that the simpler transparent baseline remains highly competitive for the operational top-50 queue.

---

## 1. Problem framing

### The content problem

A content team may have hundreds or thousands of pages but limited time available for SEO review. Reviewing every page equally is inefficient, while using one universal CTR threshold can also be misleading because CTR depends strongly on ranking position, search visibility, search intent, SERP conditions, and evidence volume.

The practical question is therefore not simply:

> Which pages have low CTR?

The question is:

> **Which content pages should a FlyRank SEO specialist or content editor inspect first when review capacity is limited?**

### Research question

Among content pages with enough observed search exposure, can March 2026 search-performance signals be used to rank pages associated with a later position-adjusted CTR-opportunity proxy measured in April 2026?

### Unit of analysis

The unit of analysis is **one pseudonymized content item belonging to one pseudonymized client**.

### Output

The system produces a **continuous opportunity score and ranked human-review queue**.

### Intended action

A FlyRank editor or SEO specialist can inspect the highest-ranked pages first and review factors such as:

- title and meta description;
- search-result presentation;
- likely search-intent alignment;
- ranking movement;
- ranking volatility;
- and whether SERP or seasonal changes could explain the observed CTR.

The score itself does not make the editing decision.

### Cost of a wrong recommendation

A **false positive** wastes review time and could encourage an unnecessary change to a page that did not need intervention.

A **false negative** pushes a potentially useful review opportunity too far down the queue.

Because review capacity is limited, ranking quality at the top of the queue matters more than simply maximizing overall classification accuracy.

### Why data and ML can help

CTR opportunity depends on several interacting signals. A page may have low CTR because of poor search-result presentation, but it may also simply rank in a position where lower CTR is normal.

Data allows pages to be compared with other pages in similar search-position contexts. Machine learning can additionally test whether combinations of impressions, clicks, CTR, search position, position volatility, and activity contain ranking information beyond a transparent rule.

---

## 2. Data safety

### Data source

The project uses the approved **pseudonymized FlyRank ML Internship warehouse release**.

The main source is:

`fact_content_daily_performance`

The full warehouse contains tens of millions of daily observations. Heavy aggregation is performed in DuckDB so only the smaller per-content feature table is brought into pandas for modeling.

### Time windows

Two non-overlapping periods are used:

- **Feature window:** March 1–31, 2026
- **Outcome window:** April 1–30, 2026

March measurements represent information available before the later April evaluation period.

### Eligibility

Pages must have:

- at least **500 March impressions**;
- a measurable March search position;
- at least **500 April impressions** for retrospective evaluation;
- and a measurable April search position.

The minimum-impression threshold reduces instability caused by CTR values based on very small numbers of impressions.

### Features allowed

Only observed or safely engineered March measurements are used as predictive features.

The final feature set contains:

- `log_impressions`
- `log_clicks`
- `feature_ctr`
- `feature_avg_position`
- `feature_position_std`
- `feature_active_days`

### Deliberately excluded fields

The following are not predictive features:

- `client_hash_id`
- `content_hash_id`
- April impressions
- April clicks
- April CTR
- April average position
- April position band
- April peer CTR
- `outcome_ctr_gap_pp`
- `opportunity_proxy`
- existing FlyRank product scores or decision flags
- raw client names
- domains
- URLs
- page titles
- keywords
- private search queries

Pseudonymous IDs are used only for joining, grouping, and grouped validation.

### Leakage considerations

The biggest leakage risk is allowing information from the April outcome period into the March feature vector.

To prevent this:

1. all predictive features end on March 31;
2. the outcome begins on April 1;
3. April-derived fields are prohibited from `FEATURES`;
4. the proxy itself is never used as an input;
5. product decision flags are excluded;
6. pseudonymous IDs are not model features.

A deliberate leakage stress test was also performed. Adding an April-derived target-related field caused performance to become suspiciously perfect, reaching **1.000 Precision@50 and 1.000 Average Precision**. This deliberately leaky result was used only to confirm that future/label information can artificially inflate performance and is not reported as valid model performance.

No raw client-identifying information appears in the public analysis.

---

## 3. Baseline

Before training a machine-learning model, I created a transparent position-adjusted rule baseline.

### Baseline idea

A universal CTR threshold is inappropriate because expected CTR changes substantially with search position.

The baseline therefore compares each page's March CTR with the median March CTR of pages in the same broad search-position band.

Broad position groups include:

- positions 1–3;
- positions 4–10;
- positions 11–20;
- positions 21–50;
- positions deeper than 50.

The page receives a positive CTR-gap signal when its CTR is below the peer median.

The transparent baseline score is:

`positive peer CTR gap × log(1 + March impressions)`

This means pages receive greater priority when:

1. their CTR is below comparable pages at similar search positions; and
2. they have meaningful search visibility.

### Why this is a fair baseline

The baseline uses the **same March information**, **same held-out clients**, **same April proxy**, and **same evaluation metrics** as the learned model.

It therefore represents a realistic alternative to ML rather than an intentionally weak comparison.

### Baseline results

On completely held-out clients, the transparent baseline achieved:

| Metric | Transparent baseline |
|---|---:|
| Precision@20 | 0.850 |
| Precision@50 | 0.880 |
| Average Precision | 0.555 |
| ROC-AUC | 0.795 |

The held-out April opportunity base rate was **0.215 (21.5%)**.

The strong top-of-queue result demonstrates that a simple position-adjusted rule already provides substantial prioritization value.

---

## 4. Model / analysis

### Assumptions

The analysis makes several explicit assumptions.

**1. March performance contains useful information about later opportunity.**

I assume that March search-performance patterns contain useful information for ranking pages associated with the April opportunity proxy. This does not mean March performance causes April performance.

**2. Similar search positions provide a useful comparison group.**

CTR depends strongly on search position, so comparing pages within broad position bands is assumed to be more meaningful than applying one universal CTR threshold.

**3. The 500-impression threshold reduces low-volume noise.**

A CTR based on very few impressions can change dramatically because of one or two clicks. The threshold is a practical analysis choice rather than a universal SEO rule.

**4. The April proxy is useful for review prioritization.**

The proxy represents relative CTR underperformance at similar search positions. It is not assumed to be objective content quality or proof that a page requires editing.

**5. Holding out complete clients is a stronger test of generalization.**

Pages belonging to one client can share traffic scale, content structure, subject matter, and search behavior. Holding out clients therefore better evaluates transfer to an unseen client than randomly holding out pages.

**6. The observed features contain useful but incomplete information.**

The model does not observe every factor affecting CTR, including exact SERP composition, competitor changes, brand familiarity, or detailed intent.

**7. The design is observational rather than causal.**

Association with a later CTR-opportunity proxy does not imply that changing a page causes improvement.

### Feature engineering

The final Random Forest uses six March-only features.

#### `log_impressions`

March search impressions transformed using `log1p`.

The log transformation reduces the impact of the strongly right-skewed traffic distribution.

#### `log_clicks`

March clicks transformed using `log1p`.

#### `feature_ctr`

Observed March click-through rate.

#### `feature_avg_position`

Impression-weighted average March search position.

#### `feature_position_std`

Variation in March search position.

This captures whether the page's ranking was relatively stable or volatile during the feature period.

#### `feature_active_days`

Number of March days during which the page received at least one search impression.

### Target / proxy definition

The April outcome is a **position-adjusted CTR-opportunity proxy**.

Each page is compared with the April median CTR of pages in its broad April search-position band.

A page receives:

`opportunity_proxy = 1`

when its April CTR is more than **0.10 percentage points below** the median CTR of pages in that position group.

This is a rule-defined decision-support proxy, not human-verified ground truth.

### Method

The final learned method is a **Random Forest classifier**.

The model's predicted probability is used as a continuous ranking score. It is not used as an automatic yes/no publishing decision.

Random Forest was selected because the relationships among visibility, clicks, CTR, ranking position, position volatility, and activity may be nonlinear and interactive.

The random seed is fixed at:

`42`

for reproducibility.

---

## 5. Evaluation

### Validation design

The final validation uses a **grouped client holdout** with `GroupShuffleSplit`.

Approximately 75% of clients are used for training and 25% are held out for testing.

No client appears in both sets.

This is important because a random page split allows pages from the same client to occur in both training and testing, which can produce overly optimistic performance when client-specific characteristics are shared.

### Primary metric

The primary metric is **Precision@50**.

This directly reflects the operational question:

> If a reviewer can inspect only the first 50 recommendations, how many of those recommendations correspond to the later opportunity proxy?

Precision@20 is also reported to examine the very top of the queue.

Average Precision and ROC-AUC provide broader measures of ranking/discrimination across the full held-out population.

### Results

| Method | Precision@20 | Precision@50 | Average Precision | ROC-AUC |
|---|---:|---:|---:|---:|
| Base rate | 0.215 | 0.215 | 0.215 | 0.500 |
| Transparent baseline | 0.850 | 0.880 | 0.555 | 0.795 |
| Random Forest | 0.850 | 0.880 | 0.610 | 0.864 |

### Main finding

The Random Forest and transparent baseline achieved the **same top-of-queue performance**:

- Precision@20 = **0.850**
- Precision@50 = **0.880**

At Precision@50, this means **44 of the first 50 pages** matched the later April opportunity proxy.

Both substantially exceeded the **21.5% base rate**.

However, the Random Forest performed better over the broader ranking:

- Average Precision increased from **0.555 to 0.610**.
- ROC-AUC increased from **0.795 to 0.864**.

### Error analysis

High model scores are not always correct recommendations.

False positives can occur when:

- ranking conditions change between March and April;
- a page experiences unusual position volatility;
- SERP composition changes;
- seasonality affects demand;
- the same search position corresponds to different query intents;
- or the rule-defined opportunity proxy fails to capture the full context.

Missed opportunities may occur when a page appears normal during March but experiences different April search conditions.

The model therefore should not be interpreted as a deterministic editing system.

---

## 6. Interpretation

The most important result is not simply that the Random Forest produced a higher global ranking metric.

The result is that **additional ML complexity did not improve the operational top-20 or top-50 queue over the transparent baseline**.

Both methods achieved:

- **85% Precision@20**
- **88% Precision@50**

This means a relatively simple, position-aware rule already captured much of the strongest actionable signal.

The Random Forest nevertheless showed better discrimination across the broader population, increasing Average Precision from **0.555 to 0.610** and ROC-AUC from **0.795 to 0.864**.

This suggests that the model learned additional combinations of search signals that were useful further down the ranking but did not change the composition of the very highest-priority group enough to improve Precision@50.

### Operational interpretation

For a content team that reviews only a relatively small number of pages, the transparent baseline may be preferable because it provides:

- equal top-50 precision;
- clearer reasoning;
- lower complexity;
- and easier explanation to an editor.

If the system needs to prioritize a much larger inventory, the Random Forest's stronger broader ranking performance may become more useful.

### Negative result as a useful result

The lack of a Precision@50 improvement is not a failed experiment.

It demonstrates that **using machine learning is not automatically better than using a well-designed transparent rule**.

The appropriate method depends on the decision being supported.

---

## 7. Recommendation

The final output should be used as a **human-review prioritization tool**.

Pages are ranked by opportunity score and accompanied by transparent reason codes.

Possible reason codes include:

- `high_visibility`
- `below_position_peer_ctr`
- `strong_position_ctr_gap`
- `position_volatile`
- `model_priority_only`

### Recommended actions

#### Review title, meta, and intent

Use when a page has meaningful visibility, a relatively strong search position, and CTR below its position peers.

A reviewer should inspect:

- title wording;
- meta description;
- SERP presentation;
- and likely search intent.

#### Review snippet and query fit

Use when the primary observed signal is a position-adjusted CTR gap.

The reviewer should determine whether the page's search-result presentation accurately communicates what the searcher will find.

#### Monitor position before editing

Use when search position is highly volatile.

The CTR signal may partly reflect ranking movement rather than a persistent content problem.

#### Manual review only

Use when the model assigns a high opportunity score but transparent evidence does not support a specific editing hypothesis.

### Human review requirement

Before acting, an editor should check:

- current page relevance;
- recent ranking movement;
- search intent;
- SERP features;
- seasonality;
- business importance;
- and possible downside from changing a currently useful page.

### Actions that should never be automatic

The model should not automatically:

- rewrite titles;
- publish content changes;
- delete pages;
- redirect pages;
- deindex pages;
- alter canonical tags;
- make technical SEO changes;
- or claim that a specific change will improve CTR.

### Confidence

Confidence is strongest when:

- the page has substantial impression volume;
- its position is relatively stable;
- and its CTR gap versus similar-position pages is clear.

Recommendations become less reliable when evidence volume is limited or ranking position is unstable.

The queue is therefore **directional decision-support**, not an autonomous SEO system.

---

## 8. Reproducibility

The complete capstone workflow is contained in:

`work/notebooks/capstone.ipynb`

### Environment

Primary tools include:

- Python
- DuckDB
- pandas
- NumPy
- scikit-learn
- matplotlib
- Hugging Face Hub

The notebook installs the required packages with:

```bash
pip install duckdb huggingface_hub scikit-learn
