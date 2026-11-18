# Cloud Data Analytics & Machine Learning Portfolio

**Niharika Rai** · Purdue University · MGMT 467: AI-Assisted Big Data Analytics in the Cloud

This repository holds 14 hands-on projects built on **Google Cloud Platform**. They cover batch and streaming data pipelines, SQL analytics on BigQuery, machine learning in BigQuery ML, and dashboards. The capstone is an end-to-end **real-time cryptocurrency price-direction prediction system**. It combines 37,000+ rows of historical market data with a live API stream and uses feature engineering to raise model recall from **9.4% to 43.3%**.

All work was done in Python and SQL in Google Colab, against real cloud resources (Cloud Storage buckets, BigQuery datasets, Pub/Sub topics, and trained BQML models). It was not run on local toy data.

---

## At a Glance

| | |
|---|---|
| **Cloud platform** | Google Cloud Platform: BigQuery, BigQuery ML, Cloud Storage (GCS), Pub/Sub, Vertex AI / Gemini, Colab Enterprise |
| **Languages** | Python, SQL (BigQuery Standard SQL), Bash (`gcloud`, `gsutil`, `bq`) |
| **Python libraries** | `google-cloud-bigquery`, `google-cloud-pubsub`, `google-cloud-storage`, pandas, Plotly, Matplotlib, Seaborn, requests, kagglehub |
| **ML techniques** | Logistic regression, linear regression, boosted trees, k-means clustering, feature engineering, `TRANSFORM` preprocessing, class-weighting, threshold tuning, sequential (time-based) train/eval splits |
| **Data engineering** | Batch ETL, streaming ingestion, idempotent infrastructure setup, schema mapping, deduplication, outlier winsorization, missingness analysis, data-quality checks |
| **BI & communication** | Looker Studio dashboards, interactive Plotly charts, KPI design, executive summaries, the DIVE (Discover, Investigate, Validate, Extend) analytics framework |
| **AI-assisted development** | Prompt engineering with Gemini / Vertex AI to generate, debug, and validate SQL and Python, with every AI output checked against the data |
| **Datasets worked with** | Kaggle Cryptocurrency Price History, CoinLore live API, OpenSky Network live flight API, U.S. BTS On-Time Performance (1.66M flights), Kaggle U.S. Flight Delays (5.8M flights), Netflix user behavior (210K records), TheLook eCommerce, Global Superstore |

---

## Featured Project: Real-Time Crypto Price-Direction Prediction

📂 [`Final_Project/Final_NiharikaRai_Analysis.ipynb`](Final_Project/Final_NiharikaRai_Analysis.ipynb) · [Reproducibility steps](Final_Project/README.md)

**Business question:** Can historical market data and live market signals be combined to predict whether a cryptocurrency's price will rise the next day?

### Architecture

```
 BATCH PATH                                         STREAMING PATH
 ──────────                                         ──────────────
 Kaggle API (23 coins, daily OHLCV)                 CoinLore REST API (top-100 live tickers)
        │  download, unzip, clean (Python)                 │  Python producer: normalize to JSON,
        ▼                                                  │  add UTC ingestion timestamp
 Cloud Storage bucket  (raw data lake)                     ▼
        │  BigQuery load job (wildcard URI)          Pub/Sub topic ──► Pub/Sub subscription
        ▼                                                  │  subscriber: streaming inserts
 BigQuery: crypto_raw  (37,082 rows)                       ▼
        │  SQL view: LEAD() target label             BigQuery: realtime_crypto_data
        ▼                                                  │
 crypto_cleaned ───────────────┬───────────────────────────┘
                               ▼   LEFT JOIN on symbol + date, ROW_NUMBER() dedupe of latest snapshot
                    crypto_combined_features
                               ▼   LAG(), ratio and volatility features
                    crypto_feature_engineered_view
                               ▼
         BigQuery ML: baseline vs. feature-engineered logistic regression
                               ▼
         Plotly KPI charts  ·  Looker Studio dashboard (team)  ·  DIVE analysis
```

### What I built

1. **Batch ETL pipeline.** I wrote reusable Python functions that authenticate to GCP, create a globally unique GCS bucket only if it doesn't already exist, download the Kaggle archive, strip malformed characters from 23 CSVs, stage them in Cloud Storage, and load them into BigQuery with a single wildcard load job (**37,082 rows**).
2. **Target engineering in SQL.** I built the binary label `is_price_up_next_day` with `LEAD(Close) OVER (PARTITION BY Symbol ORDER BY Date)`. It is partitioned per coin so one coin's prices never leak into another's label.
3. **Data quality checks.** I confirmed there are no duplicate `(Date, Symbol)` records, profiled timestamp granularity (every record is a daily close at 23:59 UTC), and documented the transformation logic.
4. **Streaming pipeline.** I built a Pub/Sub publisher that pulls live CoinLore tickers, normalizes them to a fixed JSON schema, and publishes them. A subscriber streams the messages into BigQuery and acknowledges each one. Infrastructure (topic, subscription, dataset, table) is created idempotently, so the notebook is safe to re-run. I validated the stream with timestamped queries showing near real-time rows.
5. **Batch and stream fusion.** I joined the historical and live data into one feature view. `ROW_NUMBER()` selects the latest streaming snapshot per coin per day, and `COALESCE` handles missing values.
6. **Modeling and evaluation.** I trained a baseline and a feature-engineered **BigQuery ML logistic regression** with `auto_class_weights` to handle class imbalance. I evaluated both with `ML.EVALUATE`, inspected predictions with `ML.EXPLAIN_PREDICT`, and inspected features with `ML.FEATURE_INFO`.

### Results: feature engineering more than tripled F1

| Metric | Baseline | Feature-Engineered | Change |
|---|---|---|---|
| **Recall** | 0.094 | **0.433** | **+33.9 pts (4.6×)** |
| **F1 score** | 0.159 | **0.486** | **+32.7 pts (3.1×)** |
| Precision | 0.518 | 0.552 | +3.4 pts |
| Accuracy | 0.507 | 0.542 | +3.5 pts |
| ROC AUC | 0.513 | 0.546 | +3.4 pts |

**Engineered features:** `daily_return` (momentum), `high_low_spread` (intraday volatility), `prev_day_close_lag` (temporal dependency), `avg_price` (smoothed price), and `volume_per_marketcap` (liquidity / trading intensity).

**Key insight from the DIVE analysis:** `ML.FEATURE_INFO` showed that the streaming features had zero variance in training. The live API snapshot (Dec 2025) does not overlap in time with the historical Kaggle window, so the join filled those features with 0. All of the performance gain came from the engineered batch features. I documented this as a limitation and proposed fixes: accumulate streaming history over time, align it on time windows, and add rolling averages, RSI/MACD, and interaction terms. I also proposed trying tree-based models.

**Team context:** The team split the analysis by coin. I owned **XMR, BNB, AAVE, XLM, XEM, and USDC** and contributed to the shared Looker Studio dashboard, architecture README, live demo, and governance note.

---

## Additional Projects

### Assignments

| Project | What I did | Highlights |
|---|---|---|
| **[E-commerce growth analytics (DIVE)](Assignments/Unit1_NiharikaRai_DIVE.ipynb)**<br/>TheLook eCommerce (7 tables) | Defined and computed growth KPIs. Used CTEs and `LAG()` window functions for day-over-day, MoM, and YoY growth. Drilled into product categories, traffic sources, and regions. Validated AI-generated insights with counter-queries. Built Plotly charts and a Looker Studio executive dashboard. | Repeat-purchase rate **37.5%** (29,966 of 80,010 users); AOV **$86.27**; Search drives **~$7.6M** in revenue. Caught a misleading assumption (sale price ≠ retail price): the validation showed **0 of 181K** items were discounted. Recommended a one-time-buyer retention program and channel-level CLTV analysis. |
| **[Flight diversion classifier](Assignments/Unit2_NiharikaRai_BQML.ipynb)**<br/>5.8M U.S. flights (Kaggle) | Sourced a replacement dataset after the public BigQuery tables were retired. Mapped it to a canonical schema, made an 80/20 split, and trained a baseline vs. a `TRANSFORM`-engineered logistic regression (route, day-of-week, delay buckets). Built confusion matrices at 0.5 and a custom threshold. | ROC AUC improved from **0.714 to 0.766** on a highly imbalanced target (~0.27% diverted). Wrote an operations rationale for the 0.75 threshold, weighing the cost of false alarms against missed diversions. |

### Labs: Data Engineering & Streaming

| Lab | What I did | Highlights |
|---|---|---|
| **[Lab 4: Kaggle → GCS → BigQuery + Data Quality](Labs/Lab4_Unit2_Lab1_PromptPlusExamples_Colab_Kaggle_GCS_BQ_DQ.ipynb)** | Built a reproducible pipeline: Kaggle CLI (with credentials secured at `0600` permissions), GCS staging, and BigQuery loads of 6 Netflix tables. Profiled missingness (MCAR/MAR/MNAR), duplicates, and outliers. | Removed **5,000 duplicate** watch events with a deterministic `ROW_NUMBER()` policy (105K to 100K rows). Flagged **3.46%** IQR outliers and winsorized at P01/P99 (max dropped from 799 to 204 min). Added business anomaly flags. |
| **[Lab 7: BTS flights batch ETL](Labs/Lab7_Unit3_bts_batch_etl.ipynb)** | Wrote a pipeline that downloads monthly U.S. DOT on-time files, cleans them, and loads them through GCS into BigQuery. Trained linear regression, logistic regression, and k-means models in BQML. | **1.66M flight records** loaded in one job. Trained regression, classification, and clustering models on the same curated view. |
| **[Lab 8: OpenSky API → GCS → BigQuery → BQML](Labs/Lab8_unit3_2_opensky_bq_ml.ipynb)** + [screenshots](Labs/Lab8_Screenshot1.png) | Ingested live aircraft telemetry, landed it as JSONL in Cloud Storage, and loaded it into BigQuery. Trained a velocity regressor and an on-ground classifier. | **Root-caused a failed model:** a `WHERE … IS NOT NULL` filter silently removed every `on_ground = TRUE` row, leaving only one label class. Fixed it with `COALESCE`. |
| **[Lab 9: Real-time streaming + live ML scoring](Labs/Lab9_Unit3_3_opensky_ps_bqml_with_prompts.ipynb)** | Built an event-driven architecture: a producer publishes live LA-airspace flights to Pub/Sub, and a consumer streams them into BigQuery and runs `ML.PREDICT` on each new record. | Real-time inference on live data, flight-path tracking by callsign, and classification of the latest 100 records. |

### Labs: Machine Learning in BigQuery

| Lab | What I did | Highlights |
|---|---|---|
| **[Lab 5: Churn prediction & feature engineering](Labs/Lab5_Unit2_Lab2_PromptStudio_Tasks0to4.ipynb)** ([part 2](Labs/Lab5_Unit2_Lab2_PromptStudio_Tasks5onwards.ipynb)) | Joined users with watch history and defined a 90-day-inactivity churn label. Engineered buckets, binary flags, interaction terms, and missingness indicators. Retrained the model and compared weights with `ML.WEIGHTS`. | Found that low-watch-time users churn at **46%** vs. **9.7%** for heavy users, and that a missing rating nearly **doubles** churn (19.3% vs. 10.2%). AUC moved from 0.638 to 0.642. |
| **[Lab 6: Leakage-free churn pipeline](Labs/Lab6_Logistic_Regression_BQML.ipynb)** | Built a 9-step auditable pipeline: month calendar, user × month grid with zero-fill, rolling 3-month features, and a `LEAD()`-based next-month churn label. Used a sequential (time-based) train/eval split to prevent look-ahead bias. Used Gemini to compress the pipeline into one CTE query. | **Equivalence check passed with 0 differences** in row count, class balance, and checksum hash across 236,900 rows. Diagnosed a majority-class collapse (AUC ≈ 0.50) with confusion matrices and benchmarked a boosted-tree alternative. |

### Labs: SQL, AI-Assisted Analytics & Visualization

| Lab | What I did | Highlights |
|---|---|---|
| **[Lab 1: AI-assisted SQL foundations](Labs/Lab1_AI_Assisted_SQL.ipynb)** | Turned business questions into SQL with Gemini (filters, aggregations, rankings) on Superstore data. | Identified top-profit customers and segment reach. |
| **[Lab 2: Advanced SQL via prompt engineering](Labs/Lab2_VertexAI_BigQuery_PromptsOnly.ipynb)** | Wrote CTEs, `RANK` / `ROW_NUMBER` / `LAG`, moving averages, joins, and `HAVING` filters. Built a programmatic view that standardizes column names. Used the model to debug errors and optimize query cost. Wrote counter-queries to test conclusions. | Found that "Tables" loses **$17.7K** consistently across regions, years, and ship modes, and tied the loss to heavy discounting. Wrote a 4-sentence executive summary. |
| **[Lab 3: Data visualization & dashboarding](Labs/Lab3_DataViz_Dashboarding.ipynb)** ([Plotly companion](Labs/Lab3_Viz_Plotly_and_Looker.ipynb)) | Designed airline KPIs (on-time rate, load factor, CASM, yield) and built Matplotlib/Seaborn and interactive Plotly visuals. Wireframed a Looker Studio dashboard and defined its calculated fields. | Covers exploratory vs. explanatory design, chart selection, and pitfalls. |

---

## Skills Demonstrated

**Data engineering**
- End-to-end **batch** (API/Kaggle → GCS → BigQuery) and **streaming** (API → Pub/Sub → BigQuery) pipelines
- **Idempotent**, re-runnable infrastructure code with structured error handling
- Choosing between schema autodetect and explicit schema mapping; view-based transformation layers
- Secure credential handling (no secrets in code; `0600` token permissions)

**Analytics & SQL**
- Window functions (`LEAD`, `LAG`, `ROW_NUMBER`, `RANK`, rolling `AVG/SUM … OVER`), CTEs, joins, `GENERATE_DATE_ARRAY`, `APPROX_QUANTILES`
- KPI design: growth rates, AOV, repeat-purchase rate, load factor, CASM, volatility measures
- Cost-awareness: column pruning, partition filters, `LIMIT` during exploration

**Machine learning**
- BigQuery ML: `LOGISTIC_REG`, `LINEAR_REG`, `BOOSTED_TREE_CLASSIFIER`, `KMEANS`, `TRANSFORM`
- Evaluation beyond accuracy: precision, recall, F1, ROC AUC, log loss, confusion matrices, ROC curves, threshold selection
- Handling **class imbalance** and **data leakage**; time-based validation splits
- Model debugging: diagnosing majority-class collapse, single-label training failures, and zero-variance features

**Data quality & governance**
- Missingness analysis (MCAR/MAR/MNAR), deduplication policies, IQR outlier detection, winsorization, anomaly flags
- Reconciliation checks (row counts, class distributions, checksum hashes)

**Communication**
- Executive summaries, strategist-pattern recommendations, Looker Studio dashboards, and documented DIVE reasoning (Discover → Investigate → Validate → Extend)
- **Responsible AI-assisted development:** I use Gemini to speed up code and SQL, and I validate every output against the data. Several notebooks document cases where the first AI answer was wrong and how I caught and corrected it.

---

## Repository Structure

```
mgmt467-analytics-portfolio/
├── Final_Project/
│   ├── Final_NiharikaRai_Analysis.ipynb     # Capstone: real-time crypto prediction (batch + streaming + BQML)
│   └── README.md                             # Step-by-step reproducibility guide
├── Assignments/
│   ├── Unit1_NiharikaRai_DIVE.ipynb          # E-commerce KPI analysis + Looker Studio dashboard
│   └── Unit2_NiharikaRai_BQML.ipynb          # Flight-diversion classifier with TRANSFORM + threshold tuning
└── Labs/
    ├── Lab1_AI_Assisted_SQL.ipynb
    ├── Lab2_VertexAI_BigQuery_PromptsOnly.ipynb
    ├── Lab3_DataViz_Dashboarding.ipynb
    ├── Lab3_Viz_Plotly_and_Looker.ipynb
    ├── Lab4_Unit2_Lab1_PromptPlusExamples_Colab_Kaggle_GCS_BQ_DQ.ipynb
    ├── Lab5_Unit2_Lab2_PromptStudio_Tasks0to4.ipynb
    ├── Lab5_Unit2_Lab2_PromptStudio_Tasks5onwards.ipynb
    ├── Lab6_Logistic_Regression_BQML.ipynb
    ├── Lab7_Unit3_bts_batch_etl.ipynb
    ├── Lab8_unit3_2_opensky_bq_ml.ipynb
    ├── Lab8_Screenshot{1,2,3}.png             # BigQuery table, schema, and GCS bucket evidence
    └── Lab9_Unit3_3_opensky_ps_bqml_with_prompts.ipynb
```

---

## Running the Notebooks

Every notebook includes its executed outputs, so you can review results directly on GitHub without running anything. To reproduce a notebook:

1. Open it in **Google Colab** or **Colab Enterprise**.
2. Have a GCP project with the **BigQuery**, **Cloud Storage**, and **Pub/Sub** APIs enabled (the streaming labs need Pub/Sub).
3. Run the setup cell to authenticate, then set your own `PROJECT_ID` (and bucket name where prompted).
4. For Kaggle downloads, upload your own `kaggle.json` token when prompted. Never commit it.
5. Run the cells top to bottom. The infrastructure cells skip any resource that already exists, so they are safe to re-run.

The full capstone walkthrough is in [`Final_Project/README.md`](Final_Project/README.md).

---
