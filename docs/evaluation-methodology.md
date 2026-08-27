# Evaluation Methodology & Benchmark Specification — AskOFF Canada

This document details the evaluation harnesses, relevance grading formulas, test query suites, and empirical results used to assess the AskOFF search engine.

---

## 1. Evaluation Philosophy & Rigor

Evaluating search engines for grocery products presents unique challenges compared to general web search:
1. **Multi-Attribute Intent**: A query like `"zero sugar chocolate"` is not merely a request for documents containing the word "sugar"; it represents a strict nutritional constraint ($0.0\text{g} \text{ sugar}$).
2. **Recipe Decoupling**: Queries like `"500 mL frozen blueberries"` must isolate the ingredient without filtering out packages that do not mention "500 mL".
3. **Crowd-Sourced Noise**: Upstream food catalogs contain multilingual titles, missing nutrient rows, and differing packaging units.

### Critical Distinction: Programmatic Ground Truth vs. Human Relevance
> [!IMPORTANT]
> The benchmarks presented in this report utilize **programmatic, rule-based ground truth** implemented via deterministic grading harnesses (`backend/evaluation/grading.py` and `evaluate.py`). While programmatic grading enables reproducible, automated regression testing across thousands of records, **it is not identical to human editorial judgment**. A programmatic grader verifies lexical overlap, dietary flags, and numeric bounds; it does not capture subjective human satisfaction or culinary affinity.

---

## 2. Mathematical Metric Formulations

The evaluation harness computes four standard Information Retrieval (IR) metrics across query result sets of depth $k = 10$:

### 1. Precision@$k$ (P@5 and P@10)
Measures the proportion of top-$k$ retrieved products that are relevant ($\text{score} \ge 2$):
$$\text{P@}k = \frac{\sum_{i=1}^{k} \mathbb{I}(\text{relevance}_i \ge 2)}{k}$$

### 2. Discounted Cumulative Gain (DCG@$k$) & NDCG@10
Measures ranking quality with logarithmic rank discounting, rewarding engines that place highly relevant items at rank 1:
$$\text{DCG@}k = \sum_{i=1}^{k} \frac{2^{\text{relevance}_i} - 1}{\log_2(i + 1)}$$

$$\text{NDCG@}10 = \frac{\text{DCG@}10}{\text{IDCG@}10}$$
Where $\text{IDCG@}10$ is the Ideal DCG achieved by sorting all retrieved results in descending order of true relevance.

### 3. Mean Reciprocal Rank (MRR)
Measures how quickly the user encounters the first acceptable product ($\text{relevance} \ge 2$):
$$\text{MRR} = \frac{1}{\text{rank of first relevant document}}$$
If no relevant document appears in the top 10, $\text{MRR} = 0.0$.

---

## 3. Four-Tier Relevance Grading System (`grading.py`)

Each retrieved product receives an integer grade from 0 to 3 based on structured condition matching:

| Grade | Classification | Definition & Criteria |
|---|---|---|
| **3** | **Highly Relevant** | Hard conditions pass (dietary flags, numeric thresholds, brand if required) AND the primary query terms are satisfied directly within the `product_name`. |
| **2** | **Relevant** | Hard conditions pass AND query terms/nutrients are satisfied across general fields (`categories`, `ingredients`, or `search_text`). |
| **1** | **Partially Relevant** | Hard conditions pass, but query keywords are only partially matched or secondary ingredients match. |
| **0** | **Irrelevant** | Hard condition failure: product contains a disallowed term, fails a mandatory dietary flag, violates a numeric nutrient bound, or misses a mandatory brand. |

---

## 4. Benchmark Suite 1: 35-Query Structured Benchmark

The 35-query benchmark (`backend/evaluation/benchmark_queries.json`) tests 11 distinct query archetypes against the 124,145 Canadian product catalog.

### Verified Results (Executed via `evaluate.py`)

| Category | Query Count | Mean P@5 | Mean P@10 | Mean NDCG@10 | Mean MRR |
|---|---|---|---|---|---|
| **Recipe Ingredients** | 4 | **1.00** (100%) | **0.97** (97%) | **1.00** | **1.00** |
| **Numeric Nutrition Queries** | 2 | **1.00** (100%) | **1.00** (100%) | **1.00** | **1.00** |
| **Product Modifier** | 1 | **1.00** (100%) | **1.00** (100%) | **1.00** | **1.00** |
| **Ingredient Lookup** | 1 | **1.00** (100%) | **1.00** (100%) | **1.00** | **1.00** |
| **Product with Number** | 1 | **1.00** (100%) | **1.00** (100%) | **0.99** | **1.00** |
| **Brand Lookup** | 1 | **1.00** (100%) | **0.90** (90%) | **1.00** | **1.00** |
| **Dietary Restrictions** | 4 | **0.85** (85%) | **0.80** (80%) | **1.00** | **1.00** |
| **Store Brand Products** | 5 | **0.68** (68%) | **0.54** (54%) | **0.99** | **1.00** |
| **General Product Search** | 12 | **0.43** (43%) | **0.48** (48%) | **0.81** | **0.54** |
| **Category Queries** | 2 | **0.00** (0%) | **0.00** (0%) | **0.63** | **0.00** |
| **Broad Nutrition Phrases**| 2 | **0.00** (0%) | **0.00** (0%) | **0.20** | **0.00** |
| **OVERALL COMPOSITE** | **35** | **62.86%** | **61.43%** | **86.59%** | **0.726** |

### Analysis of Benchmark Strengths & Gaps
- **Exceptional on Structured & Constrained Queries**: Queries with explicit quantities (`"500 mL frozen blueberries"`), numeric bounds (`"products with at least 20g protein"`), and dietary constraints (`"gluten free bread"`) achieved near-perfect P@5 (85%–100%) and MRR of 1.00.
- **Challenges on Broad Category & Nutrition Terms**: Unqualified queries like `"breakfast cereal"` or `"frozen vegetables"` produced lower P@5 scores because the catalog contains thousands of loose matches that matched partial tokens without satisfying strict conjunction definitions.
- **Top Hit Alignment**: The high overall Mean NDCG@10 (**86.59%**) and MRR (**0.726**) demonstrate that even when lower-ranked results diverge, the top-ranked products are overwhelmingly relevant.

---

## 5. Benchmark Suite 2: 69-Query Black-Box Live Audit

A comprehensive black-box audit (`scratch/run_comprehensive_audit.py` and `scratch/full_audit_evaluation.json`) inspected 69 live queries through the live OpenSearch search pipeline, examining 560 individual returned products.

### Verified Audit Results

| Audit Suite | Queries Evaluated | Inspected Products | Pass Count | Weak Matches | Irrelevant Matches | Relevance Rate |
|---|---|---|---|---|---|---|
| **Basic Single/Multi-Term** | 34 | 340 | 338 | 2 | 0 | **99.41%** |
| **Intent & Compound Queries**| 17 | 170 | 169 | 1 | 0 | **99.41%** |
| **Numeric Constraints** | 5 (10 items each) | 50 | 45 | 0 | 5 (violations) | **90.00%** compliance (10% violation) |

### Investigation of Numeric Violations
Of 50 inspected results for queries specifying numeric thresholds (e.g., `"chocolate under 10g sugar"`, `"snacks with at least 20g protein"`), 5 items (10%) exhibited violations. Investigation revealed:
1. **Missing Nutrient Records**: Products where the specific nutrient field was null in upstream Open Food Facts data were occasionally not filtered if the query term matched general product metadata.
2. **Serving Size vs. 100g Mismatch**: Certain products declared nutrients per-serving rather than per-100g, causing threshold boundary evaluation mismatches.

This finding was documented as an engineering priority for ongoing ingestion hardening.
