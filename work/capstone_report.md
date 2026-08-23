# Content Archetype Clustering: Grouping Search Content by Behavioral Pattern

**Author:** [apna naam likho]
**Lane:** Structured Content Archetype Clustering
**Repo:** https://github.com/Ariba86/flyrank-ml-internship
**Date:** [aaj ki date likho]

## 0. Abstract

Which content pages need editorial attention, and why? This study clusters roughly 101,000 content pages from a 60-day window of pseudonymized search performance data (impressions, clicks, ranking position, and query-diversity signals) across 47 clients. Using K-means clustering (k=5, selected via the elbow method) on eight standardized behavioral features, five distinct performance archetypes emerged — Elite Grower, High-Traffic Decliner, One-Query Dependent, Steady Middle, and Buried/Weak — validated by a moderate silhouette score (0.295) and consistent presence across 85% of clients. The headline finding is that only 0.15% of pages showed genuine month-over-month growth, while traffic volume alone (a naive baseline) could not distinguish growing pages from declining ones within the same traffic bracket. The resulting archetype-action mapping is intended as a triage tool for prioritizing editorial review, not a predictive or causal model.

## 1. Problem Framing

**Unit of analysis:** A content page (`content_hash_id`), aggregated over a 60-day window of search performance data (impressions, clicks, average ranking position, and query-level signals).

**Output:** Each content page is assigned one of five performance archetypes (clusters) — Elite Grower, High-Traffic Decliner, One-Query Dependent, Steady Middle, or Buried/Weak — along with a recommended action (protect, improve, monitor, or prune).

**Action a human takes:** A content strategist or SEO practitioner uses the archetype label to prioritize their review queue — for example, protecting pages in the Elite Grower cluster from unnecessary edits, prioritizing High-Traffic Decliners for urgent review, and deprioritizing or pruning Buried/Weak pages that consume review time without meaningful return.

**Cost of a wrong call:** A misclassified page could send limited editorial time toward a low-impact page while a genuinely declining high-traffic page goes unnoticed — the cost is wasted effort and missed early-intervention opportunities on pages that matter.

**Why ML helps here:** At scale — hundreds of thousands of pages, each with multiple behavioral signals — no team can manually inspect every page's pattern. Clustering surfaces natural groupings that would be invisible through manual review alone, turning an unmanageable volume of data into a small set of actionable categories.

## 2. Data Safety

**Data used:** Aggregated content-level search performance from the FlyRank internship warehouse — specifically `fact_content_daily_performance` (a 60-day window of daily impressions, clicks, and average ranking position) and `fact_content_query_90d` (query-level diversity and concentration signals). All identifiers are pre-pseudonymized hashed IDs (`client_hash_id`, `content_hash_id`); no raw client names, URLs, or query text were accessed.

**Columns deliberately excluded:** No client-identifying fields (client names, domains) were used — only hashed IDs, used solely for grouping, never as model features. Raw query text was excluded entirely; only aggregated query-level statistics (count, share, concentration) were used.

**Leakage risks considered:** Because this is an unsupervised clustering task (not a predictive model with a forward-looking label), there is no `trend_direction` or `trend_pct` label to leak from the future. `imp_last30` and `imp_prev30` were computed as strictly non-overlapping 30-day windows within the same 60-day panel, avoiding double-counting the same days across features.

**Confirmation:** No client-identifying details appear anywhere in `work/` — all identifiers are salted hashes provided by the pre-pseudonymized warehouse release.

## 3. Baseline

**Baseline used:** A simple three-bucket rule based solely on traffic volume (`imp_last30`): High Traffic (≥10,000 impressions), Medium Traffic (500–9,999), and Low Traffic (<500). This is a transparent, single-signal comparison — no clustering, no query or ranking signals involved.

**Why it's a fair comparison:** Traffic volume is the most intuitive, commonly used way to triage content pages, making it a realistic stand-in for how a team might prioritize without a modeling approach.

**Baseline vs. K-means result:** Within the "High Traffic" bucket alone, 151 pages were growing (Elite Grower archetype) while 2,810 pages in the same bucket were actively declining (High-Traffic Decliner archetype) — a distinction invisible to a traffic-only view. Within "Low Traffic," 17,459 pages were flagged by K-means as One-Query Dependent (a specific risk pattern) rather than simply "weak."

## 4. Model / Analysis

**Method:** K-means clustering (unsupervised learning), applied to standardized (z-scored) features via `StandardScaler`. The number of clusters (k=5) was selected using the elbow method on inertia across k=2 to k=10, where the rate of improvement visibly flattened around k=5–6.

**Why this fits the lane:** Structured Content Archetype Clustering requires grouping pages into interpretable performance categories without a predefined "correct" label — K-means discovers natural groupings directly from behavioral similarity.

**Feature list (8 features):**
- `imp_last30`, `imp_prev30` — impressions in the most recent 30 days vs. the prior 30 days
- `clk_last30` — clicks in the most recent 30 days
- `pos_last30` — average search ranking position (lower = better)
- `visible_queries` — count of distinct queries driving impressions
- `rare_share`, `anon_share` — share of impressions from rare/long-tail and anonymized queries
- `top_query_share` — share of impressions concentrated in the single top query

**Deliberately excluded:** Raw query text, client/content hash IDs (grouping only), and GA4-derived fields (not all clients have GA4 coverage).

**Target/proxy definition:** There is no predictive target — the "output" is an unsupervised grouping (cluster assignment) interpreted post-hoc as a performance archetype.

## 5. Evaluation

**Split/validation approach:** Since this is unsupervised clustering, there is no train/test split with ground-truth labels. Cluster quality was evaluated using an internal cohesion metric and a cross-client generalization check.

**Silhouette Score:** Computed on a random 10,000-row sample, the clustering achieved a silhouette score of **0.295** — moderate range, indicating real, non-random structure with some overlap between adjacent archetypes.

**Cross-client generalization check:** Of 47 clients, 40 (85%) showed 3 or more distinct archetypes among their pages, with only 1 client showing a single archetype — indicating the five archetypes recur consistently across the client base.

**Error analysis:** Because there is no ground truth, "errors" manifest as boundary ambiguity rather than misclassification. Elite Grower and Buried/Weak are well-isolated, while the moderate silhouette score suggests some pages near the Steady Middle / One-Query Dependent boundary could plausibly sit in either group.

## 6. Interpretation

- **Elite Grower** (152 pages, 0.15%) — high traffic, best ranking (~7), broad query coverage (~596 queries), the only group with month-over-month growth.
- **High-Traffic Decliner** (3,482 pages, 3.4%) — strong traffic and ranking, but trending downward.
- **One-Query Dependent** (22,771 pages, 22.4%) — moderate performance, ~76% of traffic from a single query.
- **Steady Middle** (57,053 pages, 56.2%) — the majority archetype; average performance, mild decline.
- **Buried/Weak** (18,162 pages, 17.9%) — low traffic, worst average ranking (~49), narrow query coverage.

![Pages per Archetype and Traffic vs Ranking](images/cluster_analysis.png)

**Surprises / negative results:** Only 0.15% of pages (Elite Growers) were actually growing — every other archetype showed flat-to-declining impressions month-over-month. Traffic volume alone could not distinguish Elite Growers from High-Traffic Decliners, confirming that direction of change and query concentration matter more than raw volume.

## 7. Recommendation

1. **Investigate High-Traffic Decliners first** (3,482 pages) — highest-leverage review target.
2. **Study Elite Growers** to understand what's working (152 pages).
3. **Diversify One-Query Dependent pages** (22,771 pages) — reduce single-query concentration risk.
4. **Monitor the Steady Middle** on a periodic cycle (57,053 pages).
5. **Review Buried/Weak pages** for pruning or consolidation (18,162 pages).

**Confidence and limits:** These recommendations are directional and decision-support in nature, not causal or predictive claims. Boundary cases should be spot-checked by a human before action.

## 8. Reproducibility

**To re-run from a fresh clone:**
1. Clone the repository: `git clone https://github.com/Ariba86/flyrank-ml-internship.git`
2. Open `work/notebooks/capstone_lane3_clustering.ipynb` in Google Colab.
3. Create a Hugging Face read token and store it as a Colab Secret named `HF_TOKEN`.
4. Request access to `FlyRank/internship-warehouse` on Hugging Face.
5. Run all cells top to bottom.

**Random seeds:** `random_state=42` used for `KMeans` (k=5, n_init=10) and silhouette sampling.

**Environment:** Google Colab (free-tier CPU). Packages: `duckdb`, `huggingface_hub`, `scikit-learn`, `pandas`, `matplotlib`.

## 9. Acknowledgments & Data Credit

Built on the FlyRank ML Internship dataset — [https://flyrank.ai](https://flyrank.ai)
