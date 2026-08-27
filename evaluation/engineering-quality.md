# Engineering Quality & Security Audit — AskOFF Canada

This document compiles the automated test results, static analysis outputs, type checking reports, security reviews, and code hygiene assessments for the AskOFF platform across both backend and frontend repositories.

---

## 1. Automated Test Suite Summary (172 Passing Tests)

The platform enforces hermetic test automation across all layers. Both test suites were executed and verified locally.

```
+-----------------------------------------------------------------------------+
|                      OVERALL TEST SUITE RESULTS                             |
|  Total Automated Tests: 172  |  Passing: 172 (100%)  |  Failures: 0 (0.0%)  |
+-----------------------------------------------------------------------------+
|  Backend (Pytest):   148 passed in 29.09s                                   |
|  Frontend (Vitest):   24 passed in 51.41s                                   |
+-----------------------------------------------------------------------------+
```

---

## 2. Backend Test Suite Breakdown (`pytest backend/tests/`)

Executed on Python 3.11.9 with pytest 8.2.2. All 148 tests passed cleanly in 29.09 seconds.

| Test File | Tests Passed | Subsystem Under Test | Key Assertions & Scenarios |
|---|---|---|---|
| `test_api.py` | **27** | FastAPI REST Gateway | Input validation bounds, query length guards, pagination limits, `/compare` error handling, OpenAPI schema compliance. |
| `test_evaluation_metrics.py` | **3** | IR Metrics Engine | Mathematical correctness of P@k, DCG/NDCG formulation, MRR rank discounting, and unjudged result safety. |
| `test_index_lifecycle.py` | **6** | OpenSearch Lifecycle | Timestamped index creation, bulk document streaming, mapping registration, and atomic alias swapping. |
| `test_nlp_semantics.py` | **21** | NLP Pipeline | Entity extraction, brand detection (Compliments, PC), intent classification, and query interpretation. |
| `test_normalizers.py` | **9** | Query Normalization | Spelling typo correction (`protien` $\to$ `protein`), operator spacing, and symbol stripping. |
| `test_nutrition.py` | **16** | Nutrition Parsing | Structuring raw nutrition payloads, per-100g normalization, and physical limit validation. |
| `test_nutrition_ranking.py` | **6** | Directional Ranking | Sorting by directional nutritional criteria (lowest sugar, highest protein) with BM25 tie-breaking. |
| `test_pipeline.py` | **2** | Ingestion Engine | Batch streaming via DuckDB cursor, Pydantic `RawProduct` instantiation, and NaN sanitation. |
| `test_query_engine.py` | **17** | Query Engine | Recipe quantity extraction (`250 g`, `500 mL`), numeric threshold bounds, and dietary flag extraction. |
| `test_ranking.py` | **8** | BM25 Retrieval | Tiered weights: phrase match (10.0 boost), AND match (5.0 boost), and fuzzy match (0.5 boost). |
| `test_retrieval_quality.py` | **4** | End-to-End Retrieval | Precision and recall bounds across sample product documents. |
| `test_search.py` | **4** | Search Router | Parameter routing, filter clause construction, and pagination math. |
| `test_search_engine.py` | **9** | Repository Layer | DuckDB and OpenSearch repository abstraction interfaces and error handling. |
| `test_settings.py` | **4** | Configuration Guards | Pydantic validation: rejection of wildcard CORS in production, mandatory TLS, and credential checks. |
| `test_synonyms.py` | **12** | Canadian Synonyms | Canadian retail synonym mapping (`soya` $\to$ `soy`, `yogourt` $\to$ `yogurt`, `kraft dinner` $\to$ `macaroni and cheese`). |
| **TOTAL** | **148** | — | **100% Passed (0 Failures, 0 Regressions)** |

---

## 3. Frontend Test Suite Breakdown (`vitest run`)

Executed using Vitest 4.1.11 and JSDOM. All 24 tests across 5 test suites passed.

| Test File | Tests Passed | Subsystem Under Test | Key Assertions & Scenarios |
|---|---|---|---|
| `src/tests/badges.test.ts` | **2** | Visual Badges | Nutri-Score (A to E) badge generation, color mapping, and NOVA group formatting. |
| `src/tests/productCard.test.ts` | **6** | Product Card UI | Card rendering, macro-nutrient pills, image fallback handling, and bookmark actions. |
| `src/tests/client.test.ts` | **5** | API Client | REST client request building, timeout handling, error serialization, and CDN URL normalization. |
| `src/tests/assistantService.test.ts`| **4** | OffBot Assistant | Deterministic question evaluation, Nutri-Score explanations, and ODbL citation construction. |
| `src/tests/components.test.tsx` | **7** | Presentation UI | `ErrorBoundary` recovery, `SearchBar` debouncing, `Pagination` boundary conditions, and `EmptyState` rendering. |
| **TOTAL** | **24** | — | **100% Passed (0 Failures, 0 Regressions)** |

---

## 4. Static Analysis & Type Checking

### Python Backend Linting (`ruff check backend`)
- **Tool**: Ruff 0.5+
- **Result**: `All checks passed!` (0 lint errors, 0 style warnings).
- **Enforced Standards**: PEP 8 formatting, unused import elimination, strict type annotation preservation, and exception chaining hygiene.

### Frontend Type Safety (`tsc -b`)
- **Tool**: TypeScript Compiler (`tsc` 6.0+)
- **Configuration**: Strict mode enabled (`noImplicitAny: true`, `strictNullChecks: true`).
- **Result**: 0 type errors. The frontend TypeScript interfaces strictly mirror backend Pydantic models.

### Frontend Production Bundling (`vite build`)
- **Tool**: Vite 8.1+ / Rollup
- **Result**: 1,863 modules transformed cleanly in 4.45 seconds. All production chunks are gzipped and code-split by route.

---

## 5. Security & Operational Hardening

### Input Validation & API Defense
- **Query Length Restriction**: Query strings longer than 500 characters are rejected with HTTP 422 to protect regex engines from ReDoS attacks.
- **Pagination Boundary Guards**: Page size is capped at 100 items; offsets cannot exceed 10,000 to prevent deep-pagination memory exhaustion.
- **Comparison Size Limits**: The `/compare` endpoint accepts a maximum of 50 product barcodes per request.

### CORS & Secret Handling
- **CORS Restriction**: Wildcard origins (`*`) are disallowed when credentials are enabled. In production, explicit trusted origins must be declared in `ASKOFF_CORS_ORIGINS`.
- **Zero Hardcoded Secrets**: All OpenSearch credentials, hosts, and paths are injected via environment variables (`.env`).
- **Sanitized Errors**: Exception stack traces and internal database error payloads are stripped before error responses are sent to the client.

### Container Security
- **Unprivileged Execution**: The backend container runs under `appuser` (UID 10001), preventing root privilege escalation vulnerabilities.
- **Healthcheck Probes**: Native container health checks verify service liveness via `GET /health` every 30 seconds.
