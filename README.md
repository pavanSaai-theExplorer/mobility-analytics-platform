# 🚖 Mobility Analytics Platform (Go Cabs)
### Scalable Lakehouse Architecture with Spark Declarative Pipelines

![Status](https://img.shields.io/badge/Status-Complete-success)
![Platform](https://img.shields.io/badge/Databricks-Runtime_13.3-FF3621?logo=databricks)
![Spark](https://img.shields.io/badge/Apache_Spark-3.5-E25A1C?logo=apachespark)
![AWS](https://img.shields.io/badge/AWS-S3-232F3E?logo=amazon-aws)

## 📖 Executive Summary
This project implements a production-grade **Lakehouse Data Engineering solution** for "Go Cabs," a transportation service. The objective was to modernize legacy batch processing into a **Declarative Streaming Pipeline** capable of handling real-time data ingestion, schema evolution, and complex upserts.

Leveraging **Databricks** and **Delta Lake**, the pipeline processes raw trip data from AWS S3, applies CDC (Change Data Capture) logic, and serves governed, region-specific datasets for analytics.

---

**Business Impact:**
* **Reduced Data Latency:** Migrated from daily batch loads to continuous streaming.
* **Data Integrity:** Automated schema evolution prevents pipeline failures from upstream changes.
* **Governance:** Implemented fine-grained access control for regional managers.

---

## 🏗️ System Architecture & Data Flow

The solution adheres to the **Medallion Architecture**, ensuring a clear separation of concerns between raw ingestion, refinement, and business aggregation:

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/1cad5b99-da58-4d79-b580-aa8c22005814" />

**1. Ingestion (S3 → Bronze):**
* **Source:** Raw CSV files stored in AWS S3 (partitioned by City and Trips).
* **Mechanism:** Spark **Auto Loader (`cloudFiles`)** for incremental streaming ingestion.
* **Feature:** Automatic **Schema Evolution** to handle upstream data changes without breaking the pipeline.

**2. Refinement (Bronze → Silver):**
* **Transformation:** Data cleaning, type casting, and standardizing formats.
* **CDC Logic:** Handling "Late Arriving Updates" using **Declarative CDC (`apply_changes`)**. This manages trip status changes (e.g., `STARTED` → `COMPLETED`) by Upserting records based on timestamp ordering.

**3. Aggregation (Silver → Gold):**
* **Modeling:** Star Schema creation by joining Fact (Trips) and Dimension (City, Calendar) tables.
* **Governance:** Implementation of **Row-Level Security** via Unity Catalog, creating specific views for regional managers (e.g., Vadodara vs. Chandigarh).

---

## 🧠 Key Engineering Decisions (Trade-offs)

### 1. Declarative vs. Imperative Paradigms
* **Decision:** Adopted a **Declarative** approach (defining *what* the result should be) rather than Imperative (defining *how* to compute it).
* **Impact:** Reduced codebase size by **~60%**, minimized technical debt, and allowed the Catalyst Optimizer to determine the most efficient execution plan automatically.

### 2. Solving the "Files vs. Streams" Problem
* **Challenge:** The source system dumps files irregularly. Using `COPY INTO` or standard batch reads causes latency and requires manual state management.
* **Solution:** Implemented **Auto Loader**. It uses file notification events to detect new data instantly and maintains a scalable checkpointing mechanism for exactly-once guarantees.

### 3. Handling Mutable Data (CDC)
* **Challenge:** Trip statuses change over time. A standard `APPEND` strategy would create duplicate records for the same trip.
* **Solution:** Utilized **SCD Type 1/2** logic via the `apply_changes()` API. Instead of writing complex `MERGE` statements, the pipeline automatically resolves conflicts by prioritizing the record with the latest timestamp.

---

## 💻 Repository Structure
```bash
├── transportation_pipeline/
│   └── transformations/
│       ├── bronze/          # Auto Loader logic & Ingestion
│       ├── silver/          # CDC Logic & Transformation
│       └── gold/            # Aggregations, Joins & Views
├── project_setup.ipynb      # Environment setup & Data generation
└── README.md                # Project documentation
```

---

## 🚀 How to Reproduce

### Prerequisites

- **Databricks Workspace**
  - Community Edition or Premium
- **Cloud Storage**
  - AWS S3 bucket (recommended)  
  - OR Databricks File System (DBFS) for testing
- **Data**
  - Raw CSV files for:
    - `city`
    - `trips`

---

## Step 1: Environment Configuration

### Clone Repository
- Import this GitHub repository into your **Databricks Workspace** using **Repos**.

### Run Setup Script
1. Open `project_setup.ipynb`.
2. Attach a compute cluster.
3. Run all cells to initialize the environment and mount storage paths.

> **Note:**  
> Ensure you update the **S3 / DBFS paths** in this notebook to match your actual storage location.

## Step 2: Running the Pipeline

This project utilizes **Delta Live Tables (DLT)** for orchestration.

### Create the DLT Pipeline
1. Navigate to **Workflows → Delta Live Tables**.
2. Click **Create Pipeline**.
3. Configure the following settings:
   - **Pipeline Name:** `go_cabs_pipeline`
   - **Product Edition:** Core
   - **Pipeline Mode:** Triggered (Batch)
   - **Source Code:**  
     - Click the folder icon  
     - Select the `transportation_pipeline/transformations` directory from this repository
4. Click **Create**, then **Start** to trigger the initial run.

## Step 3: Verifying Results

Once the pipeline finishes, use the **SQL Editor** to query the Gold layer.

### Sample Query
```sql
SELECT * 
FROM go_cabs_gold.analytics_view 
LIMIT 10;
```

---

## 🔮 Future Roadmap

### Data Quality
- Integrate **Delta Live Tables Expectations** (e.g., `@dlt.expect_or_drop`) to automatically quarantine records with:
  - Invalid fare amounts
  - Missing or malformed timestamps

### CI/CD
- Implement **Databricks Asset Bundles (DABs)** to automate deployments using **GitHub Actions**.

### Observability
- Connect **Delta Live Tables system logs** to a monitoring dashboard to track:
  - Data latency
  - Pipeline throughput
    



