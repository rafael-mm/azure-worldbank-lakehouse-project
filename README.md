# Azure Data Engineering Project: World Bank Lakehouse

End-to-End Data Engineering pipeline built using **Azure** and **Databricks**, following modern cloud architecture principles (**Medallion Architecture**) and rigorous cost optimization strategies (*FinOps*).

---

## 🏗️ 1. Architecture & Core Components

This project simulates a corporate analytical ingestion and processing platform utilizing the public **World Bank API (v2)**.

* **Orchestration:** Azure Data Factory (ADF) V2 (metadata-driven dynamic pagination flows).
* **Storage (Data Lake):** Azure Data Lake Storage (ADLS) Gen2 (`Standard / LRS`) with *Hierarchical Namespace* enabled, organized into `bronze`, `silver`, and `gold` containers.
* **Processing & Transformation:** Databricks (Serverless Compute / Unity Catalog) running PySpark and Delta Lake.
* **Security & Governance:** Passwordless authentication between ADF and ADLS Gen2 via **System-Assigned Managed Identity** (RBAC `Storage Blob Data Contributor`), and secret-free notebook execution via environment variables / Databricks Widgets.

> *[Placeholder: Architecture Diagram (`assets/architecture-diagram.png`)]*

---

## 📊 2. Pipeline Strategy (Medallion Architecture)

### 📥 Bronze Layer (Raw)
Fidelity storage for raw JSON payloads collected directly from the REST API, applying distinct storage patterns based on data volatility to prevent the *Small Files Problem*:
1. **Fact / Time-Series Data (`population`):** Organized under **Hive-style partitioning** (`bronze/world-bank/.../year=YYYY/month=MM/day=DD/`) to enable query *Partition Pruning*.
2. **Dimension Data (`countries`):** Organized under a **Landing Folder** pattern (`bronze/world-bank/.../YYYY-MM-DD/page_N.json`), delegating file consolidation and schema parsing to Spark.

### 🧹 Silver Layer (Refined)
* Multi-file JSON/NDJSON ingestion, struct flattening, strong typing, and null filtering.
* Business rule enforcement: filtering out macro-economic aggregates (`region_name != "Aggregates"`) in the country dimension to prevent *double-counting* in downstream aggregations.
* Persistence as ACID-compliant **Delta Lake** tables (`silver.worldbank_countries` and `silver.worldbank_population`).

### 🏆 Gold Layer (Analytic / Serving)
* Relational join between Fact and Dimension Delta tables.
* Analytical enrichment using PySpark **Window Functions** (`lag`) to compute Year-over-Year (`yoy_growth_percentage`) population growth.
* Serving curated analytical datasets (`gold.population_analytics`) persisted in Delta and Parquet formats for BI consumption.

---

## 💡 3. Technical Highlights & Engineering Decisions

* **Dynamic REST API Pagination:** To retrieve the full historical series, upfront ADF *Web Activity* dynamically extracts total pages to drive a parameterized *ForEach* loop.
* **Serverless Compatibility:** Bypassing legacy Hadoop cluster properties restricted in Serverless environments by using Azure Blob REST APIs with short-lived, least-privilege SAS tokens injected at runtime.
* **ANSI Mode Resilience:** Safe PySpark casting preventing pipeline failures on empty coordinate strings.
* **FinOps by Design:** Centralized provisioning under a single Resource Group using `Standard/LRS` storage and serverless on-demand compute to maintain near-zero cloud spend.