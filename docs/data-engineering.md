# Data Engineering — Open Food Facts Canada Dataset

This document details the data engineering lifecycle, schema architecture, transformation logic, and reproducibility steps used to produce the canonical Canadian food dataset for the AskOFF search engine.

---

## 1. Dataset Overview & Verified Metrics

The AskOFF project indexes a specialized, Canada-focused subset derived directly from the global Open Food Facts database.

| Metric / Property | Verified Value | Evidence Source |
|---|---|---|
| **Dataset Source** | Hugging Face: `openfoodfacts/product-database` (split: `food`) | `OFF_Canada_Data_Code.ipynb` |
| **Storage Format** | Apache Parquet (ZSTD compression) | `metadata.json`, filesystem audit |
| **Total Rows** | **124,145** | `verify_dataset_stats.py` |
| **Unique Barcodes (`code`)** | **124,145** | `verify_dataset_stats.py` |
| **Duplicate Barcodes** | **0** | `verify_dataset_stats.py` |
| **Total Columns** | **25** | Parquet schema inspection |
| **File Size (Compressed)** | **21.8 MB** (21,832,251 bytes) | File system stat |
| **Barcodes with Leading Zeroes** | **113,135** (91.13%) | `metadata.json` |
| **Products with Nutriments Struct** | **113,459** (91.39%) | `verify_dataset_stats.py` |
| **Products with Image Metadata** | **124,145** (100.00%) | `verify_dataset_stats.py` |
| **Direct `front_image_url` Count** | **28,608** (23.044%) | `verify_dataset_stats.py` |
| **Missing Direct Front Image URLs**| **95,537** (76.956%) | `metadata.json` |

---

## 2. Source Data Extraction & DuckDB Pipeline

The global Open Food Facts database contains millions of crowd-sourced entries across more than 150 countries. Processing the entire multi-gigabyte export on local developer machines or standard cloud instances presents memory bottlenecks when using traditional tools such as Pandas.

### Why DuckDB?
DuckDB was selected as the analytical query engine due to its:
- **Zero-Copy Streaming**: Ability to read multi-gigabyte remote or local Parquet files directly from disk without exhausting RAM.
- **Nested Parquet Support**: Native understanding of complex nested structs, lists, and maps (e.g., `images`, `nutriments`, `categories_tags`).
- **Columnar Projection**: High-speed projection selecting only the 25 necessary columns, reducing disk I/O and intermediate serialization costs.

### Extraction Logic (`OFF_Canada_Data_Code.ipynb`)
The extraction pipeline executes in three distinct steps:

#### Step A: Filtering Canadian Products
Records are filtered using DuckDB's native list function:
```sql
SELECT COUNT(*)
FROM read_parquet('/content/off_food.parquet')
WHERE list_contains(countries_tags, 'en:canada');
```
This isolates 124,145 records whose country distribution tags include Canada.

#### Step B: Column Selection & Normalization
24 core attributes are extracted and compressed into an intermediate Parquet artifact:
```sql
COPY (
    SELECT
        code,
        product_name,
        generic_name,
        brands,
        categories,
        categories_tags,
        ingredients_text,
        ingredients_tags,
        ingredients_analysis_tags,
        allergens_tags,
        labels_tags,
        nutriments,
        nutriscore_grade,
        nutriscore_score,
        nova_group,
        environmental_score_grade,
        quantity,
        product_quantity,
        product_quantity_unit,
        serving_size,
        countries_tags,
        images,
        completeness,
        last_modified_t
    FROM read_parquet('/content/off_food.parquet')
    WHERE list_contains(countries_tags, 'en:canada')
)
TO '/content/off_canada.parquet'
(FORMAT PARQUET, COMPRESSION ZSTD);
```

#### Step C: CDN Image URL Derivation
While raw image identifiers reside in the `images` struct, resolving them dynamically at search time introduces client-side complexity. The pipeline unrolls the `front_en` image metadata and constructs the canonical Open Food Facts CDN URL:
```sql
COPY (
    SELECT
        *,
        CASE
            WHEN EXISTS (
                SELECT 1
                FROM UNNEST(images) AS t(img)
                WHERE img.key = 'front_en'
                  AND img.imgid IS NOT NULL
                  AND img.rev IS NOT NULL
            )
            THEN
                'https://images.openfoodfacts.org/images/products/'
                || LPAD(CAST(code AS VARCHAR), 13, '0')
                || '/'
                || (
                    SELECT CAST(img.imgid AS VARCHAR)
                    FROM UNNEST(images) AS t(img)
                    WHERE img.key = 'front_en'
                      AND img.imgid IS NOT NULL
                      AND img.rev IS NOT NULL
                    LIMIT 1
                )
                || '.jpg'
            ELSE NULL
        END AS front_image_url
    FROM read_parquet('/content/off_canada.parquet')
)
TO '/content/off_canada_with_images.parquet'
(FORMAT PARQUET, COMPRESSION ZSTD);
```

---

## 3. Dataset Schema (25 Columns)

The canonical schema output in `openfoodfacts_canada.parquet` is defined below:

| Column Name | Data Type | Description |
|---|---|---|
| `code` | `VARCHAR` | Unique product barcode identifier (string preserved). |
| `product_name` | `STRUCT(lang VARCHAR, text VARCHAR)[]` | Multilingual product titles. |
| `generic_name` | `STRUCT(lang VARCHAR, text VARCHAR)[]` | Multilingual generic food descriptions. |
| `brands` | `VARCHAR` | Comma-separated brand names. |
| `categories` | `VARCHAR` | Free-text categorization. |
| `categories_tags` | `VARCHAR[]` | Hierarchical taxonomy tags (e.g., `en:plant-based-foods`). |
| `ingredients_text` | `STRUCT(lang VARCHAR, text VARCHAR)[]` | Raw ingredient declarations in French and English. |
| `ingredients_tags` | `VARCHAR[]` | Normalized ingredient taxonomy tags. |
| `ingredients_analysis_tags` | `VARCHAR[]` | Automated tags (e.g., `en:palm-oil-free`, `en:vegan`). |
| `allergens_tags` | `VARCHAR[]` | Declared allergen tags (e.g., `en:peanuts`, `en:milk`). |
| `labels_tags` | `VARCHAR[]` | Certifications and claims (e.g., `en:organic`, `en:non-gmo`). |
| `nutriments` | `STRUCT(...)[]` | Nested array containing per-100g and per-serving macro/micronutrients. |
| `nutriscore_grade` | `VARCHAR` | Official Nutri-Score grade (`a`, `b`, `c`, `d`, `e`). |
| `nutriscore_score` | `INTEGER` | Discrete Nutri-Score calculation integer. |
| `nova_group` | `INTEGER` | Food processing classification group (1 to 4). |
| `environmental_score_grade`| `VARCHAR` | Eco-Score classification grade when available. |
| `quantity` | `VARCHAR` | Declared packaging quantity (e.g., `500 g`, `1 L`). |
| `product_quantity` | `VARCHAR` | Parsed numeric packaging quantity. |
| `product_quantity_unit`| `VARCHAR` | Unit associated with parsed packaging quantity. |
| `serving_size` | `VARCHAR` | Standardized manufacturer serving size declaration. |
| `countries_tags` | `VARCHAR[]` | Array of country distribution tags. |
| `images` | `STRUCT(...)[]` | Comprehensive image metadata (revisions, sizes, uploaders). |
| `completeness` | `FLOAT` | Open Food Facts record completeness ratio ($0.0 \to 1.0$). |
| `last_modified_t` | `BIGINT` | Unix epoch timestamp of the latest source edit. |
| `front_image_url` | `VARCHAR` | Precomputed direct front image URL. |

---

## 4. Critical Engineering Realities & Data Nuances

### 1. Barcode Integrity: Leading Zeroes
In barcode standards (UPC-A, EAN-13), barcodes frequently start with leading zeroes (e.g., `0060383708825`). 
- Storing barcodes as integers converts `0060383708825` to `60383708825`, permanently breaking product lookup and external CDN path resolution.
- **Verified Reality**: Exactly **113,135 of the 124,145 barcodes (91.13%)** in this dataset begin with zero. The ingestion pipeline strictly enforces string typing (`VARCHAR`) throughout DuckDB, Pydantic, and OpenSearch.

### 2. Geographic Scope: Canada-Focused vs. Canada-Only
- **100%** of records contain `en:canada` in `countries_tags`.
- However, **6,583 records** also contain tags for other regions (e.g., `en:united-states`, `en:france`, `en:world`).
- **Engineering Precision**: This dataset is accurately described as **Canada-focused**, reflecting products distributed within Canadian retail markets, rather than strictly Canada-exclusive.

### 3. Missing Data & Sparsity
Crowd-sourced open data inevitably contains missing or sparse fields:
- **Direct Front Images**: Direct `front_image_url` exists for **28,608 records (23.044%)**. The remaining 95,537 records do not have direct `front_en` URLs. Downstream consumers must handle fallback resolution or render category-specific SVG placeholders.
- **Nutritional Values**: While 113,459 records (91.39%) contain a `nutriments` struct, specific sub-fields (such as dietary fiber, sodium, or trans fats) can be null or zero.
- **NaN Sanitation**: Floating point `NaN` and `Inf` values common in scientific datasets are explicitly sanitized by the adapter before transmission to OpenSearch, preventing JSON serialization failures.

---

## 5. Dataset Reproduction & Official Publications

The complete dataset can be reproduced from scratch using the documented notebook, and is published on public platforms:

| Resource | Official Location | Description |
|---|---|---|
| **Hugging Face Dataset** | [`offCanada/openfoodfacts-canada`](https://huggingface.co/datasets/offCanada/openfoodfacts-canada) | Official transformed Canadian Parquet dataset. |
| **Generation Notebook** | [`OFF_Canada_Data_Code.ipynb`](https://huggingface.co/datasets/offCanada/openfoodfacts-canada/blob/main/OFF_Canada_Data_Code.ipynb) | Google Colab / DuckDB reproduction code. |
| **Kaggle Publication** | [`saitejakommi/open-food-facts-canada-dataset`](https://www.kaggle.com/datasets/saitejakommi/open-food-facts-canada-dataset) | Public community dataset on Kaggle. |
| **Upstream Source** | [`openfoodfacts/product-database` (split: `food`)](https://huggingface.co/datasets/openfoodfacts/product-database) | Raw global Open Food Facts dataset. |

### Reproduction Steps:
1. **Open Google Colab** (standard CPU/T4 runtime).
2. **Install Dependencies**:
   ```bash
   pip install -q datasets duckdb
   ```
3. **Execute `OFF_Canada_Data_Code.ipynb`**:
   - Downloads `off_food.parquet` directly from the Hugging Face hub.
   - Executes DuckDB SQL transformations.
   - Generates `off_canada_with_images.parquet` (local repository name: `openfoodfacts_canada.parquet`).
4. **Local Verification**:
   Run the verification script from the backend repository root:
   ```bash
   python scratch/verify_dataset_stats.py
   ```
   This confirms 124,145 rows, 0 duplicate barcodes, and verified column counts.

